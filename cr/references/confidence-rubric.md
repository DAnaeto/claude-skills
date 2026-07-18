# Confidence rubric for finding verification

Give each verification agent: the diff (or relevant hunks), the finding's description, the list of relevant CLAUDE.md paths, and this rubric **verbatim**. The agent scores the finding 0–100. The score **labels** the finding, it does not gate it: report every finding that scores above the false-positive floor, ranked by score. A score at/near **0** means the verifier refuted it (pre-existing, linter-caught, or no real scenario) — drop those. Everything else is reported, with **≥ 80** marked `CONFIRMED` and the rest `PLAUSIBLE` (carry the score + one-line reason so the reader can weigh it). The point of verification is to attach confidence, not to hide anything real that scored below a threshold.

For issues flagged from CLAUDE.md instructions, the agent must double-check that the CLAUDE.md actually calls out that issue specifically.

---

Score each issue on a scale from 0–100, indicating your level of confidence:

- **0**: Not confident at all. This is a false positive that doesn't stand up to light scrutiny, or is a pre-existing issue.
- **25**: Somewhat confident. This might be a real issue, but may also be a false positive. You weren't able to verify that it's a real issue. If the issue is stylistic, it is one that was not explicitly called out in the relevant CLAUDE.md.
- **50**: Moderately confident. You were able to verify this is a real issue, but it might be a nitpick or not happen very often in practice. Relative to the rest of the PR, it's not very important.
- **75**: Highly confident. You double-checked the issue and verified that it is very likely a real issue that will be hit in practice. The existing approach is insufficient. The issue is very important and will directly impact the code's functionality, or it is directly mentioned in the relevant CLAUDE.md.
- **100**: Absolutely certain. You double-checked the issue and confirmed it is definitely real and will happen frequently in practice. The evidence directly confirms this.

---

The verifier is trying to **refute** the finding, not confirm it. Instruct it to check the false-positive list in SKILL.md and to default low when uncertain.
