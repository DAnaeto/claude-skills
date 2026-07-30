---
name: cr
description: Unified code review — one command for GitHub PRs and local diffs. Use whenever the user asks to review code, a PR, a pull request, a branch, a diff, or their changes ("review PR 27", "review my changes", "look over this branch before I push", "/cr"). Replaces /review, /code-review, and code-review:code-review for this user: prefer this skill over all three whenever a code review is requested. Auto-detects target (PR number/URL → that PR; no args → current branch vs main), scales depth by effort level, reports verified findings in chat, and only posts to GitHub with an explicit --comment.
---

# Unified Code Review

One review command with one mental model: **figure out the target, gather the diff, review at the requested depth, verify before reporting, report in chat** — and touch GitHub only when explicitly asked.

## Parse the arguments

Arguments may arrive in any order; parse loosely:

- **A PR number or GitHub URL** → PR mode.
- **Nothing target-like** → local mode: the current branch's work (committed since the merge-base with the default branch, plus staged/unstaged changes).
- **Effort word** (`low`, `medium`, `high`, `xhigh`, `max`, or natural language like "thorough", "quick") → sets depth. Default: `medium`.
- **`--as <persona>`** → review in that voice: `perfectionist` (default), `adversary`, `operator`, `maintainer`. See Voice.
- **`--comment`** → after reviewing, also post the findings to the PR on GitHub (PR mode only).
- **`--fix`** → after reporting, apply the confirmed findings to the working tree, then re-report each finding with its outcome (`fixed` / `skipped` / `no_change_needed`).

## Reference files — read only when the gate is met

None is unconditional. Skip the ones the diff and the depth don't call for.

| File | Read when |
|---|---|
| `references/personas.md` | `--as` was passed, **or** depth ≥ high (per-lens assignment). Not for a default-voice low/medium review. |
| `references/silent-failure-lens.md` | the diff has error-handling surface: try/catch/except, fallback-on-failure, retries, log-and-continue, null-coalescing over failable calls. At high+, hand the file to that lens agent instead of reading it. |
| `references/test-coverage-lens.md` | the diff changes testable behavior or touches test files. Same hand-off at high+. |
| `references/confidence-rubric.md` | **verifier-only, depth ≥ high** — it exists to be pasted into the verify preamble. Never at low/medium: self-verification there is re-reading the code, not scoring it. |
| `references/review-workflow.md` | depth ≥ high. |
| `references/github-comment.md` | PR mode **and** `--comment`. |

## Voice (persona)

Default: a **brutal, detail-oriented perfectionist senior engineer** — unsparing about real defects, precise about the failure scenario, no hedging and no praise padding.

The persona sets tone and where attention lands first. **It never moves the evidence bar.** "Never report" below outranks every persona, always: brutal means *unsparing about real defects*, not *finds more things*. A persona that drifts into nitpicking has failed twice — the review is wrong, and it cost more to be wrong.

`--as adversary|operator|maintainer` swaps the voice; at high+ the parallel lenses take different personas so the fan-out is perspective-diverse. Both live in `references/personas.md`, gated above.

## Gather the diff (the only review scope)

**PR mode:**
```bash
gh pr view <N> --json title,body,author,baseRefName,headRefName,state,additions,deletions,changedFiles,labels
gh pr diff <N>
```
The PR's diff is the entire review scope — local working-tree changes are out of scope. When you need surrounding code, Read files from this checkout if it's on the PR's branch; otherwise fetch contents via `gh`.

**Local mode:**
```bash
git diff $(git merge-base origin/HEAD HEAD 2>/dev/null || git merge-base main HEAD)  # branch work
git diff HEAD    # uncommitted on top
git status --short
```
If the repo has no main/default branch to diff against, review the uncommitted changes only and say so.

Large diffs: save to a file and read in chunks, or delegate file-by-file summarization to gatherer agents. Skip pure deletions of generated/vendored/doc artifacts after confirming that's what they are.

**Context pass (always):** locate the root CLAUDE.md and any CLAUDE.md in directories the diff touches — these are the project's own review criteria and outrank generic taste.

## Domain lenses (detect from the diff, don't ask)

Stack-specific expertise lives in dedicated skills/agents — pull them in as extra review criteria when the diff touches their domain, instead of reviewing from generic knowledge:

- **Rails** (`.rb` under app/, Gemfile, migrations) → two complementary sources:
  - `layered-rails:layered-rails-reviewer` agent (or the `layered-rails` skill's review workflow) as an opinionated architecture lens: layer violations, callbacks, god objects, anemic models.
  - The `rails-guides` skill (`~/.claude/skills/rails-guides/`) as ground truth: its SKILL.md maps topics to official Rails Guides. Read only the reference file(s) matching what the diff touches (migrations → migrations guide; params/auth → security guide; queries → querying guide; hooks → callbacks guide) and use them to (a) verify framework-behavior findings before reporting — a claim about callback ordering, transaction semantics, or eager loading should be checked against the guide, not memory — and (b) catch diffs that contradict documented Rails behavior. The security guide is worth consulting for any Rails diff touching params, cookies, SQL, or rendering.
- **Frontend** (React/Next.js components, CSS, UI code) → load `vercel-react-best-practices` for performance patterns; add `web-design-guidelines` when the diff changes user-facing UI/accessibility surface.
- **Data/ETL pipelines** → the project's CLAUDE.md invariants are the domain lens (e.g. money types, streaming, migration drift); weight them heavily.
- Other stacks: if an installed skill/agent clearly covers the domain, use it; otherwise proceed with the general lenses.

A mixed diff gets multiple lenses, each applied only to the files in its domain. At high/max, run each domain lens as its own parallel agent alongside the standard ones. Domain-lens findings go through the same verification and false-positive filters as everything else.

## Review at the requested depth

Every depth hunts the same things, in priority order: **correctness bugs** (concrete inputs/state → wrong behavior), **silent failures**, **violations of project conventions** (CLAUDE.md, invariants stated in nearby code comments), **security issues in the changed code**, **performance regressions**, **test coverage gaps**. The two lens files back the silent-failure and test-coverage items — read them only when the diff has the surface they cover (see the gate table); a diff with no error handling doesn't need the silent-failure lens loaded. At high+ they go to dedicated agents rather than into your own context. Nitpicks a senior engineer wouldn't raise don't make the list at any depth.

### low / medium (default) — single-pass, self-verified

Read the diff carefully with the CLAUDE.md context loaded. For each candidate finding, before keeping it: re-read the actual code (not just the diff hunk), check it against the false-positive list below, and construct the concrete failure scenario. No scenario → not a finding. Medium reads surrounding code where the diff alone is ambiguous; low stays close to the hunks. No confidence scoring here — self-verification is re-reading the code, so mark what survives `CONFIRMED` and don't read the rubric.

### high / xhigh / max — parallel lenses + adversarial verification

Orchestrate via the Workflow tool using the script in `references/review-workflow.md` — the `/cr` invocation at these levels is the user's explicit opt-in to multi-agent orchestration. The workflow fans out the lenses, dedups their findings at a barrier, verifies them **one agent per file** (not per finding), and at `max` spends extra refuters only on the ambiguous band; synthesis stays with you. The lenses:

1. One agent per lens (skip lenses that don't apply, e.g. no git history on a fresh repo), each assigned a persona per `references/personas.md`. **Model tiering**: only the deep semantic lenses (`bug-scan`, `comment-contracts`) are worth Opus xhigh — the rest are checklist-shaped sweeps and run Sonnet medium, since verification gates their output anyway. Per-lens settings are in `references/review-workflow.md`. The lenses:
   - **CLAUDE.md compliance** — audit the diff against every relevant CLAUDE.md (guidance written for code-writing doesn't all apply to review; judge applicability).
   - **Bug scan** — read the changed hunks for real bugs; big issues, not nitpicks.
   - **Git history** — `git blame`/`git log` the modified code; flag changes that break something the history shows was deliberate.
   - **Prior PRs** (PR mode, `max`) — review comments on past PRs touching these files that apply again here.
   - **Code-comment guidance** — invariants and warnings in comments within modified files that the change violates.
   - **Silent failures** — give the agent `references/silent-failure-lens.md`; it sweeps every error-handling site in the diff.
   - **Test coverage** — give the agent `references/test-coverage-lens.md`; it maps new behavior to tests and rates the gaps.
2. Verification is **batched per file**: one Sonnet-medium agent per file reads that file once and scores every finding in it against the rubric in `references/confidence-rubric.md` (given verbatim). Re-reading the same file once per finding was the dominant verifier cost. A verifier that dies or returns no verdict for a finding leaves it **unverified**, not refuted — unverified findings are reported as `PLAUSIBLE`, never silently dropped.
3. At `max` only, extra refuters run on the **ambiguous band** (score 30–79, plus unverified findings). Findings at ≥ 80 are already confirmed and findings under 30 are already refuted; more votes don't move either, so they don't get any.

## Never report (false positives)

- Pre-existing issues on lines the diff didn't touch
- Anything a linter, typechecker, or compiler catches (imports, type errors, formatting) — assume CI runs those; don't run builds yourself
- Pedantic nitpicks a senior engineer wouldn't raise
- General quality complaints (docs, style) unless a CLAUDE.md explicitly requires them. Test coverage is NOT a general complaint here — it has its own lens — but only gaps rated 8+ on that lens's scale become findings; "coverage could be higher" never does
- Designed, documented degradation paths flagged as "silent failures" — the silent-failure lens only fires where errors vanish invisibly or the project's own error contract isn't followed
- Issues explicitly silenced in code (lint-ignore comments)
- Intentional functionality changes that are the point of the diff
- "Looks like a bug" that survives on vibes but has no concrete failure scenario

This list outranks the persona, the domain lenses, and any verifier score.

## Report

Always in chat, in this order:

1. **Overview** — a few sentences: what the change does, its shape, and your overall read. For substantial PRs add brief sections (design/quality observations, test coverage, risks) in prose.
2. **Findings** — call `ReportFindings` once with **every** finding that survived verification, ranked by confidence score, highest first (empty array only if nothing survived). Each finding carries `file`, `line`, `summary`, `failure_scenario`, `category`, and `verdict`. Set `level` to the effort used. Do not duplicate the findings as prose text.
3. **Verdict** — one line: approve / approve with follow-ups / needs changes, plus the 1–2 things you'd actually act on.

**Verification labels, it does not gate** — the single statement of this rule; the rubric and the workflow script both defer here. Nothing real is dropped for merely scoring low; the score sets the tier, not survival:

- **≥ 80**, corroborated across lenses, or self-verified at low/medium → `CONFIRMED`
- **30–79**, or **unverified** (verifier died or returned nothing) → `PLAUSIBLE`; fold the score and the verifier's one-line reason into `summary`/`failure_scenario` so the reader can weigh it. An absent verdict is not a refutation
- **under 30** → actively refuted (pre-existing, linter-caught, or no real scenario). These, and only these, are dropped

**Paste-ready block:** compose the GitHub-comment markdown only in **PR mode** — locally there is no PR to paste it into and `ReportFindings` already renders the findings in chat. In PR mode, compose it when `--comment` was passed (then post it) or when the user asks.

**`--comment`:** post to the PR — follow `references/github-comment.md` (eligibility checks, format, sha-pinned links) exactly. Never post without the flag.

**`--fix`:** apply the confirmed findings to the working tree, then call `ReportFindings` again with `outcome` set per finding.
