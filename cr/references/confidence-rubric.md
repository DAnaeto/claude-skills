# Confidence rubric for finding verification

Given verbatim to every verification agent (it is the per-verifier preamble, so keep it short — every
word here is paid once per verifier). Depth ≥ high only; low/medium self-verifies and never reads this.

---

You are trying to **refute** each candidate, not confirm it. Read the pinned changed code,
directly affected callers, tests and originating requirement if supplied. Check existing
safeguards, reachability, framework version and whether the defect is introduced or materially
exposed by the diff. Do not infer a spec from naming or reviewer commentary. An issue already
handled by a linter/typechecker is not a code-review finding. Cite the disconfirming evidence.

Score 0–100:

- **0** — refuted, pre-existing and unaffected, non-issue, or tooling already catches it.
- **25** — cannot establish the claimed failure; stylistic/speculative.
- **50** — plausible but missing a material fact; not ready to report without main-loop evidence.
- **75** — concrete reachable failure, with code/requirement evidence and limited uncertainty.
- **100** — directly reproduced or established by code and contracts; real material impact.

Return a verdict for each candidate with a one-line reason, including what you checked and the
remaining uncertainty. Scores ≥ 70 are candidates for reporting, not automatic findings: the main
reviewer still checks them. Scores below 70 require new evidence before reporting. An absent
verifier is not a refutation; the main reviewer must inspect that candidate personally. Merge
duplicates only when they share one cause; nearby lines or similar wording are insufficient.
