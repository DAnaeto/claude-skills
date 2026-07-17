# Lens: test coverage quality and completeness

Behavioral coverage, not line coverage. The question is never "what % is covered" — it's "which regression would sail through CI unnoticed?"

## Hunt for

- Untested error-handling paths (pairs with the silent-failure lens: an untested catch block is where silent failures live)
- Missing boundary/edge cases for new logic
- Uncovered critical business-logic branches
- Absent negative cases for validation logic
- Missing concurrency/async coverage where the change is concurrent/async

## Judge test quality, not just presence

- Tests should pin behavior and contracts, not implementation details — flag tests that would break on a reasonable refactor, or that overfit (asserting exact strings/ordering that isn't part of the contract)
- A test that can't fail when the behavior regresses is not coverage
- Some paths are already covered by existing integration tests — check before flagging

## Rate every gap 1–10 before reporting

- **9–10**: breaking it means data loss, security failure, or system outage
- **7–8**: breaking it causes user-facing errors
- **5–6**: edge cases causing confusion or minor issues
- **1–4**: completeness for its own sake

**Report 8+ as findings** (category `test-coverage`, with the concrete regression the missing test would catch as the failure scenario). Mention 5–7 in the prose overview at high/max only. Never report below 5 — no trivial getters/setters, no coverage-metric pedantry. Follow the project's own testing standards from CLAUDE.md (e.g. a repo that keeps 100% coverage has said coverage IS a requirement — hold it to that).
