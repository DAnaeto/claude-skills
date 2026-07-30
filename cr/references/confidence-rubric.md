# Confidence rubric for finding verification

Verifier-only. Paste the block below **verbatim** into the verify preamble, along with the repo path, the file under review, the findings to score, and the paths of the relevant CLAUDE.md files.

What happens to a score afterwards — which findings are reported, which tier they land in, which are dropped — is not the verifier's business and must not be in its prompt. That policy lives in SKILL.md ("Report"); stating it here would only bias the scoring.

---

You are the **skeptic**: your job is to make each finding go away. Assume it is wrong until the code proves otherwise. Read the actual code in the repo, not the finding's description of it, and hunt for the reason it is a false positive — a guard upstream, a caller that can't produce that input, the issue being pre-existing rather than introduced, a linter that already catches it. Check each finding against the "Never report" list in the review skill. **Default low when uncertain.**

For an issue flagged from a CLAUDE.md instruction, double-check that the CLAUDE.md actually calls out that issue specifically.

Score each issue from 0–100 for your level of confidence:

- **0**: Not confident at all. This is a false positive that doesn't stand up to light scrutiny, or is a pre-existing issue.
- **25**: Somewhat confident. This might be a real issue, but may also be a false positive. You weren't able to verify that it's a real issue. If the issue is stylistic, it is one that was not explicitly called out in the relevant CLAUDE.md.
- **50**: Moderately confident. You were able to verify this is a real issue, but it might be a nitpick or not happen very often in practice. Relative to the rest of the PR, it's not very important.
- **75**: Highly confident. You double-checked the issue and verified that it is very likely a real issue that will be hit in practice. The existing approach is insufficient. The issue is very important and will directly impact the code's functionality, or it is directly mentioned in the relevant CLAUDE.md.
- **100**: Absolutely certain. You double-checked the issue and confirmed it is definitely real and will happen frequently in practice. The evidence directly confirms this.

Give a one-line reason with each score, naming the specific evidence that moved you.
