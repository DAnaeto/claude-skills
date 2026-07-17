# Posting a review comment to GitHub (--comment only)

Posting is outward-facing: it happens only with an explicit `--comment`, and only after the checks below.

## Eligibility — check immediately before posting

Skip posting (and say why) if the PR:

- is closed or merged
- is a draft
- doesn't need a review (automated PR, trivial and obviously fine)
- already has a code review comment from you (don't double-post; offer to update instead)

If the review took a while, re-run this check right before posting — state can change.

## Comment format

Use `gh pr comment` (never web requests). Keep it brief, no emojis, cite and link every finding.

With findings:

```
### Code review

Found <N> issues:

1. <brief description> (CLAUDE.md says "<...>")

<sha-pinned link, see below>

2. <brief description> (bug due to <file and snippet>)

<sha-pinned link>

🤖 Generated with [Claude Code](https://claude.ai/code)

<sub>- If this code review was useful, please react with 👍. Otherwise, react with 👎.</sub>
```

With no findings:

```
### Code review

No issues found. Checked for bugs and CLAUDE.md compliance.

🤖 Generated with [Claude Code](https://claude.ai/code)
```

## Sha-pinned link rules (GitHub won't render previews otherwise)

Format: `https://github.com/<owner>/<repo>/blob/<full-sha>/<path>#L<start>-L<end>`

- **Full** commit sha — resolve it first (`gh pr view <N> --json headRefOid`); command substitution inside the comment body will not work, the comment is rendered as literal Markdown
- Repo must be the one under review
- `#` after the file path; range as `L<start>-L<end>`
- Include ≥1 line of context each side (commenting on lines 5–6 → link `L4-L7`)
- When citing a CLAUDE.md rule, link the CLAUDE.md too
