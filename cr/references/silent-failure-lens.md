# Lens: silent failures and error handling

Every swallowed error is hours of future debugging. The lens: find places where something goes wrong and nobody — not the user, not the logs, not the caller — finds out, or finds out with nothing actionable.

## Sweep the diff for error-handling surface

try/catch (try/except, Result types, error callbacks), fallback logic and defaults-on-failure, optional chaining / null-coalescing that skips failable operations, retry loops, and branches that log-and-continue.

## Interrogate each one

- **Swallowing**: empty catch blocks; catching and returning null/default without logging; retries that exhaust silently. Would anyone debugging this in 6 months know it happened?
- **Breadth**: does the catch grab only the expected error types, or would it also hide unrelated bugs (typos, attribute errors, OOM)? List what it could hide.
- **Fallbacks**: is falling back to alternative behavior explicit and justified (documented, spec'd, or deliberately designed — e.g. an optional subsystem that's allowed to be absent), or does it mask the real problem? Production fallbacks to mock/stub implementations are architectural defects.
- **Propagation**: should this bubble up to a handler that has enough context to act, instead of being caught here?
- **Messages**: do user-facing errors say what went wrong AND what to do about it, with enough context (operation, identifiers) to distinguish it from similar failures?

## Severity and reporting

- **CRITICAL** — true silent failure or a broad catch that hides unrelated errors → always a finding (category `silent-failure`, with the specific hidden-error scenario)
- **HIGH** — unjustified fallback, useless/generic error message → finding
- **MEDIUM** — missing context in an otherwise-surfaced error → prose mention at high/max only

Respect the project's own error-handling contract from CLAUDE.md before flagging: some codebases deliberately distinguish "disable the subsystem" errors from "skip this one item" errors, or explicitly isolate per-item failures so one bad input can't abort a batch. A designed, documented degradation path is not a silent failure — flag only where the design isn't being followed or the degradation is invisible.
