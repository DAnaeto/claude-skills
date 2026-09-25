---
name: cr
description: 'Unified code review — one command for GitHub PRs and local diffs. Use whenever the user asks to review code, a PR, a pull request, a branch, a diff, or their changes ("review PR 27", "review my changes", "look over this branch before I push", "/cr"). Replaces /review, /code-review, and code-review:code-review for this user: prefer this skill over all three whenever a code review is requested. Auto-detects target (PR number/URL → that PR; no args → current branch vs main), reviews as a brutal senior engineer (override with `--as`), scales depth by effort level, verifies findings before reporting, and only posts to GitHub with an explicit --comment.'
---

# Unified Code Review

One command, one mental model: **figure out the target, gather the diff, review at the requested depth
in persona, verify before reporting, report in chat** — and touch GitHub only when explicitly asked.

## Parse the arguments

Any order, parse loosely:

- **A PR number or GitHub URL** → PR mode.
- **An explicit branch, commit, or diff target** → review that target, not an inferred PR.
- **Nothing target-like** → local mode: the current branch's work (committed since the merge-base with
  the default branch, plus staged/unstaged changes and relevant untracked source files).
- **Effort word** (`low`, `medium`, `high`, `xhigh`, `max`, or natural language like "thorough",
  "quick") → depth. Default `medium`.
- **`--as <persona>`** → override the reviewer persona (see below).
- **`--comment`** → after reviewing, also post the findings to the PR (PR mode only) as a formal
  GitHub review — never an approval. See the `--comment` section below for which review type.
- **`--fix`** → after reporting, attempt every surviving actionable finding, including `PLAUSIBLE`
  ones (not just score ≥ 80), then report `fixed` / `skipped` / `no_change_needed` per finding.
- **`--paste`** → force the paste-ready block on (default: on in PR mode, off in local mode).
- **`--models <sel1>,<sel2>,...`** → override the bug-scan model roster for this run only (high/xhigh/
  max; see the high+ section below). Each entry is a `modelRoles` alias (`@default`, `@slow`, ...) or a
  raw `provider/model-id`, optionally suffixed `:effort`. Omit to use the two pinned selectors in
  `config.yml` (`@cr_anthropic` and `@cr_openai`); the hard cap ignores additional selectors after two.

## Persona

Default: **brutal, detail-oriented perfectionist senior engineer.**

- Terse. No praise padding, no "great work", no restating what the diff obviously does.
- Writes to a competent peer: name the defect and its consequence, skip the tutorial.
- Won't let a real defect slide because the diff is small, the fix is awkward, or the author clearly
  worked hard on it.
- Holds itself to the same bar: re-read the code before making any claim.

**The persona sets voice and where attention goes — never the evidence bar.** "Brutal" means
uncompromising about real defects, not license to raise nitpicks: the Never-report list below outranks
the persona in every case. Padding the list with style opinions is sloppiness, not rigor.

The alternate personas (`--as adversary|operator|maintainer|perfectionist`) and the per-lens persona
assignment used at high+ live in `references/personas.md` — read it only when `--as` is given or depth
is ≥ high.

## Pin the target and requirements

**PR mode:** read title, body, head/base SHAs and the PR diff. The PR's diff is the review scope;
local working-tree changes are out of scope. Read surrounding code from this checkout only when it is
at the PR's head; otherwise fetch the head version. Fix the head SHA for anchors and recheck before
posting. Start with a diff stat and skip generated, vendored and lockfile contents unless needed to
verify a concrete issue. For `db/structure.sql`, inspect only unexpected drift against migrations.

**Local mode:** compare committed branch work with the merge-base of the default branch, then include
staged and unstaged changes. Inspect relevant untracked source files too: `git diff` omits them.
Explicit branches/commits use their requested range; do not silently switch to the current branch.
Without a usable base, review only the working tree and say so. Large diffs: inspect in chunks.

**Context pass (always):** read the root and affected-directory `AGENTS.md`/`CLAUDE.md`, applicable
repo standards, nearby callers and tests. Review changed behavior and directly affected existing
code, not unrelated legacy issues. An old defect becomes in scope only when the change exposes or
depends on it.

**Optional project profiles:** if `~/.agents/cr-profiles/` exists, inspect the YAML frontmatter
of its `*.md` files. Each profile declares `name` and a `repositories` list of exact
`owner/repo` names. In PR mode use the target PR's `owner/repo` (not a local fork's
origin); in local mode resolve the checkout's `origin` remote (accept
`git@github.com:owner/repo.git` and `https://github.com/owner/repo.git`, stripping `.git`).
Do **not** match a directory basename, branch name, or code keyword. Read the body only of
profiles whose repository entry matches; if identity is unavailable or none match, load no
profile. Profiles are optional review criteria, not new agents or instructions to override
the repo's guidance, the diff boundary, or the evidence bar. Keep them outside this skill
directory so `/cr` remains portable. At high+ pass relevant profile excerpts to applicable
lenses and the skeptic; low/medium apply them in the main pass.

To extend `/cr` for another organization, add a profile file with `name` and exact
`repositories` frontmatter plus review rules; removing that file disables it. No edit to `/cr`
is needed.

**Separate two questions:** (1) Is the implementation technically sound? (2) Does it implement the
originating request? Locate user-supplied instructions, PR body, referenced issue, or matching
repo-local plan/spec. Read the original requirement, not a reviewer paraphrase. Compare each material
requirement with changed code and behavior; flag omissions, silent weakening, unintended changes,
and risky scope creep with a citation to the requirement. If no source exists, say “No spec available”
and do not invent one. Do not search old PR comments or historical issues as a substitute. A linked
originating issue is task context, not the prohibited past-PR review lens.

## Technical review (every language)

Trace changed behavior from inputs through state changes and effects. Look for reachable logic and
boundary errors, invalid transitions, partial writes, retry/idempotency failures, resource cleanup,
rollback and backward-compatibility hazards. For security, trace untrusted input to authorization,
ownership, query, deserialization, redirect, URL-fetch, HTML, filesystem and dynamic-dispatch sinks;
describe the attack and impact. For performance, establish a plausible workload before flagging N+1,
unbounded work, materialization, excess round trips or allocations. A pattern alone is not a finding.

Treat tests as changed implementation: can they pass with the behavior broken? Check meaningful
success/failure paths, auth negatives, concurrent/retried effects where relevant, and test doubles
that conceal the path being tested. Existing integration coverage counts. Missing tests need a
specific regression they would catch, not a demand for coverage for its own sake.

## Rails and Ruby (only when the diff touches a Rails app)

First inspect the actual `.ruby-version`/toolchain, `Gemfile.lock` and Rails config, database
adapter/schema, queue adapter/job settings, and relevant gems. Do not assume Rails 8, PostgreSQL,
Solid Queue, RSpec, Minitest, or a particular auth library from the folder name. Verify uncertain
version/adapter behavior against the installed code or the matching `rails-guides` reference.
Do not call valid older idioms a bug solely because a newer API exists.

Read `references/rails-failure-modes.md` for Rails diffs, including low/medium reviews. It adds
concrete failure scenarios, disproof checks and Ruby/AR/job/security pitfalls beyond the general
guides and architecture lens. Read its matching sections rather than applying every probe as a
mandatory finding. Verify adapter/version-dependent claims in the installed app or official docs.

- **Ruby:** only `nil` and `false` are falsey (`0`, `""`, `[]` are truthy). Check symbol/string
  keys, keyword forwarding, `map` versus `each` return values, shared mutable defaults, bang
  methods returning `nil`, mutation, rescue scope and exception propagation. Trace untrusted
  `send`/`constantize`, `Marshal.load`, unsafe YAML and shell interpolation to real sinks.
- **Active Record:** check relation versus array semantics, association cardinality, `includes`/
  `preload`/`eager_load` joins, hidden `default_scope`, enum mapping changes, callback lifecycle,
  and bulk APIs that bypass validation/callbacks/timestamps. Follow callers to confirm a suspected
  N+1 or query change; do not infer a query from syntax alone.
- **Integrity and rollout:** identify the actual invariant and its writers before proposing a
  database constraint. Unique validation alone races; check matching unique index where uniqueness
  is essential. Consider NOT NULL, foreign keys and CHECK constraints when a real invariant needs
  them, without mechanically demanding every constraint. Check migration/backfill ordering,
  lock duration on the actual adapter/table size, deploy compatibility, reruns, and rollback.
- **Transactions and jobs:** database transactions cannot undo external calls. Check `after_commit`
  versus in-transaction effects, enqueue timing for this Rails/queue configuration, repeated job
  delivery, worker crashes after effects, stable idempotency keys, bounded retries and recovery.
  A model callback is not automatically wrong; flag a concrete unexpected effect or failure path.
- **Requests and security:** check action-by-action authorization, ownership/policy scoping before
  record access, `permit!`/`to_unsafe_h`, mass assignment, serialization exposure, SQL fragments/
  `Arel.sql`, redirect targets, XSS and CSRF for cookie-authenticated requests. Rails 8
  `params.expect` is relevant only when available; `require(...).permit(...)` is not inherently
  incorrect. Follow the app's auth stack rather than prescribing Pundit, CanCanCan or another.
- **Performance:** look for association access in iteration, `.to_a`/Ruby filtering before SQL,
  `map` where a narrow query suffices, expensive work inside a request/job loop, unstable
  pagination, missing indexes for new important filters, and cache keys missing tenant/auth state.
  State why this path is hot or can grow before raising it.

**Rails-native design:** prefer the simplest appropriate model, association, scope, concern,
controller, job, validation, constraint, callback or RESTful resource. An extra service, repository,
command, form, factory, DI layer or interface must solve a specific coupling, ownership, reuse,
test-boundary, integration or transaction-orchestration problem. Equally, do not demand “fat model”
design where the repo consistently uses another domain-operation pattern. Use `layered-rails` as a
*question set*, not a rule to extract; use its review workflow only on affected architecture and
verify each claim against local conventions. An idiomatic but different Rails style is not a defect.

The `layered-rails` “specification test” asks whether a class belongs in its layer; it does
not check whether the PR fulfills the originating user/issue specification. Keep both checks
separate and apply the latter whenever a real requirements source exists.

For non-Rails diffs, apply the technical review above and only the installed domain guidance
matching the changed stack (`vercel-react-best-practices`, `web-design-guidelines`, etc.). In a
mixed diff, scope each lens to its domain. Do not create a separate agent per file or domain when
the eight-agent budget would sacrifice requirement fit or verification.

## Review at the requested depth

All depths compare technical soundness with requirement fit where a real source exists. Hunt
correctness and security first, then reliability, meaningful performance, project conventions and
test effectiveness. A bare pattern or hypothetical concern is not a finding.

**Read references only when relevant:** `references/rails-failure-modes.md` for Rails diffs;
`references/silent-failure-lens.md` for changed error paths;
`references/test-coverage-lens.md` for changed behavior; `references/personas.md` for `--as` or high+;
`references/confidence-rubric.md` and `references/review-workflow.md` for high+;
`references/github-comment.md` for `--comment` only.

### low / medium (default) — single pass, self-verified

No subagents. Read the diff and directly affected code/tests, compare with the available spec, then
generate candidate findings. Medium reads further context where needed; low remains near hunks.
Challenge each candidate yourself and discard it if the safeguard, caller, framework behavior or
tests disprove it. Check one last time for a missed requirement or significant affected path.

### high / xhigh / max — parallel lenses + adversarial verification

Use `references/review-workflow.md`: two independent bug scans (`@cr_anthropic` and `@cr_openai` by
default), then targeted spec-fit/Rails/test/failure lenses, followed by file-group skeptic checks.
The cap is eight total subagents (five review lenses, three verifiers), not eight concurrent jobs;
no past-issues agent, extra refuters or retry agents. An originating spec-fit lens takes precedence
over generic checklist sweeps. Rails depth is driven by inspected versions/config, not hardcoded
claims. `xhigh`/`max` may use the optional code-comment and targeted git-history lenses within the
same cap, never silently exceed it. Skipped verification is disclosed and handled by the main reviewer.
Keep synthesis and the final adversarial recheck in the main loop; agent agreement is not proof.

## Self-critique before reporting

For **each** candidate, inspect the actual changed line, surrounding implementation, call sites,
relevant tests and explicit requirement (if any). Try to find an existing safeguard. Verify
version-specific claims using the installed framework or appropriate official guide. State the
reachable input/state, observed wrong behavior, impact and smallest reasonable fix. Discard if
pre-existing and unaffected, speculative, stylistic, already mitigated, or caught by the normal
linter/compiler/typechecker. A targeted reproduction or short scenario can resolve ambiguity;
don't run project-wide checks or invent facts. Deduplicate by underlying cause, including across
files; anchor to a changed line. No fixed finding quota. No material problem → say so.

Severity describes **impact, not reviewer confidence**:
- `critical`: likely exploit, corruption/data loss, major outage or severe invariant break.
- `high`: important user-facing or operational failure likely to occur.
- `medium`: concrete bug or maintainability problem with meaningful consequences.
- `low`: small but actionable issue with limited impact; no cosmetic style nits.

Confidence is separate: report verified findings, with `CONFIRMED` for strong direct evidence and
`PLAUSIBLE` only when a concrete scenario remains but one assumption cannot be settled. Do not
promote an unverified agent hypothesis into a finding; check it yourself or put a *material intent
question* in Human review. Neither a low score nor an `ignore` comment proves a real defect safe.

Tests missing for important behavior belong in a short **Missing test scenarios** section with the
exact regression they would catch. Only elevate a missing test to a finding for a material blind spot
(the existing test-coverage lens uses 8+/10), after checking existing integration coverage. Product
ambiguity, rollout choices and architectural tradeoffs whose answer changes the outcome belong in
**Human review**, not as pretend bugs or generic warnings. No spec available → state that in the
summary, not as a speculative finding.

## Report

Keep the existing chat-first findings and paste-ready PR output, but show only synthesized,
actionable results. No raw agent brainstorm, score dump, praise padding or generic checklist.

1. **Findings**, ordered by severity and practical impact. For each: `[severity] path:line`,
   problem, concrete failure scenario, smallest recommended change, and `CONFIRMED` or
   `PLAUSIBLE` with its remaining uncertainty. Cite a requirement when the defect is spec fit.
   If none: **No material issues found.** Call `ReportFindings` once with the surviving findings
   when that tool exists; retain `file`, `line`, `summary`, `failure_scenario`, `category`, `verdict`
   and `level` fields. Otherwise give the same content in chat; never assume the tool exists.
2. **Missing test scenarios**, only important untested regressions not already covered elsewhere.
   Skip the section when empty. Do not duplicate a reported test-coverage finding.
3. **Human review**, only genuine product/rollout/architecture decisions whose answer matters.
   Say what evidence is missing and the alternatives; never present an unverified technical claim
   as a decision for a human. Skip the section when empty.
4. **Summary**, one or two sentences: scope, verdict (approve / approve with follow-ups / needs
   changes), any limited review coverage, and “No spec available” when applicable.

**Paste-ready findings:** on by default for PR mode, off for local mode (`--paste` forces on).
After the summary, include one concise quote addressed to the author per finding, anchored to a
changed line on the PR head. Include a GitHub `suggestion` only when the replacement is exact and
stands alone; do not fabricate a compilable patch or spec snippet. No finding → no paste-ready block.
Check PR-head contents once per file with findings; local checkout line numbers are not authoritative
unless it is at the PR head. In local mode, use current on-disk line numbers.

**`--comment`:** post a formal review only with this flag, following `references/github-comment.md`
(eligibility, event type, format, sha-pinned links). Never post an approval or post when no findings
survived. `--paste` is the manual alternative; it does not itself write to GitHub.

Review event type: on another author's PR, `REQUEST_CHANGES` for a confirmed critical/high issue;
otherwise `COMMENT`. On your own PR, always `COMMENT` (GitHub disallows self-request-changes).

**`--fix`:** attempt *all* surviving findings, including plausible ones after checking their concrete
scenario; never apply refuted or unsupported hypotheses. Report outcome and reason for each.
If reviewing a PR from a checkout not at that PR's head, first use a dedicated PR worktree or
skip edits and explain why; never patch a different branch by accident. Verify changed behavior
after fixes and re-report outcomes. No commits or pushes unless explicitly requested.
