# Posting a review to GitHub (--comment only)

Posting is outward-facing: it happens only with an explicit `--comment`, and only after the checks
below. It posts a formal GitHub **review**, never a plain issue comment — this makes the finding show
up as a review decision on the PR, not just a comment in the thread.

## Eligibility — check immediately before posting

Skip posting (and say why) if the PR:

- is closed or merged
- is a draft
- doesn't need a review (automated PR, trivial and obviously fine)
- already has a code review from you (don't double-post; offer to update instead)
- has no surviving actionable findings — do not post a formal "no issues" review

If the review took a while, re-run this check right before posting — state can change.

## Choosing the review event

Never `APPROVE` — that's not this skill's job even when the diff is clean.

- PR author is **not** you (`gh pr view <N> --json author` vs. `gh api user --jq .login`):
  confirmed `critical`/`high` finding → `REQUEST_CHANGES`; otherwise `COMMENT`.
- PR author **is** you: always `COMMENT`. GitHub's API rejects `REQUEST_CHANGES` (and `APPROVE`) from
  a PR's own author — trying anyway fails the call, so don't attempt it.

## Posting

Use `gh pr review` (never `gh pr comment`, never a web request):

```
gh pr review <N> --request-changes --body "$BODY"   # confirmed critical/high finding, not own PR
gh pr review <N> --comment --body "$BODY"            # other actionable findings, or own PR
```

## Review body format

Keep it brief, no emojis or fabricated attribution; cite and link each verified finding:

```
### Code review

1. [high] <concrete failure and minimal change>
   <sha-pinned link>

2. [medium] <concrete failure and minimal change>
   <sha-pinned link>
```

Do not claim a review used Claude Code merely because a carrier used an Anthropic model. No
reaction requests or automatic signatures. Include only findings from the final synthesized report.

## Sha-pinned link rules (GitHub won't render previews otherwise)

Format: `https://github.com/<owner>/<repo>/blob/<full-sha>/<path>#L<start>-L<end>`

- **Full** commit sha — resolve it first (`gh pr view <N> --json headRefOid`); command substitution
  inside the comment body will not work, the comment is rendered as literal Markdown
- Repo must be the one under review
- `#` after the file path; range as `L<start>-L<end>`
- Include ≥1 line of context each side (commenting on lines 5–6 → link `L4-L7`)
- When citing a CLAUDE.md rule, link the CLAUDE.md too
