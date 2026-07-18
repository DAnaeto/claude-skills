# Workflow orchestration for high / xhigh / max

At `high` and above, run the lens fan-out + verification through the Workflow tool instead of ad-hoc Agent calls. The `/cr` invocation at these levels IS the user's opt-in to multi-agent orchestration. Synthesis (overview, `ReportFindings`, verdict) stays in the main loop — it needs conversation context.

Build the lens prompts first (diff path, repo path, lens instructions, false-positive pre-filter, per SKILL.md), then invoke Workflow with this script, passing `args`:

```json
{
  "repo": "<abs repo path>",
  "lenses": [{"key": "bugs", "prompt": "<full self-contained lens prompt>", "model": null, "effort": null}, ...],
  "verifyPreamble": "<the confidence rubric from references/confidence-rubric.md, verbatim, plus: 'You are trying to REFUTE this finding. Read the actual code in the repo. Score 0-100 per the rubric; default low when uncertain. Finding to verify:'"
}
```

**Model tiering** (set per lens in `args.lenses`): every lens runs **Opus**; effort is the cost knob. `bug-scan` and `comment-contracts` get `model: "opus", effort: "xhigh"` — they do the deep semantic work that converts reasoning depth into caught bugs. Every other lens (`claude-md`, `git-history`, `silent-failure`, `test-coverage`, `prior-prs`, domain lenses) gets `model: "opus", effort: "medium"` — checklist/retrieval-shaped sweeps whose output the adversarial verify stage gates anyway, so they don't need the extra thinking budget. Verifiers are hardcoded to Sonnet medium below (bounded re-investigations; the one stage where the cheaper model is safe because a wrong kill/keep shows up in the score, not silently).

```js
export const meta = {
  name: 'cr-review',
  description: 'Code-review lens fan-out with adversarial verification',
  phases: [
    { title: 'Review', detail: 'one agent per lens' },
    { title: 'Verify', detail: 'score each deduped finding; report all but clear false positives, tiered by score' },
  ],
}

const FINDINGS = {
  type: 'object',
  properties: {
    findings: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          file: { type: 'string' }, line: { type: 'integer' },
          summary: { type: 'string' }, failure_scenario: { type: 'string' },
          category: { type: 'string' }, evidence: { type: 'string' },
        },
        required: ['file', 'summary', 'failure_scenario', 'category'],
      },
    },
    mentions: { type: 'array', items: { type: 'string' } },
  },
  required: ['findings'],
}
const VERDICT = {
  type: 'object',
  properties: { score: { type: 'integer' }, reason: { type: 'string' } },
  required: ['score', 'reason'],
}

phase('Review')
// Barrier (not pipeline) is deliberate: dedup across ALL lenses before paying for verification.
const results = await parallel(
  args.lenses.map(l => () => agent(l.prompt, {
    label: `lens:${l.key}`, phase: 'Review', schema: FINDINGS,
    // Per-lens tiering (see Model tiering above): null/undefined inherits the session model.
    ...(l.model ? { model: l.model } : {}), ...(l.effort ? { effort: l.effort } : {}),
  }))
)
const all = results.filter(Boolean).flatMap(r => r.findings.map(f => ({ ...f })))
const mentions = results.filter(Boolean).flatMap(r => r.mentions || [])

// Dedup: same file + nearby line = same finding; keep the first, note the corroborating lens.
const deduped = []
for (const f of all) {
  const dup = deduped.find(d => d.file === f.file && Math.abs((d.line || 0) - (f.line || 0)) <= 3)
  if (dup) { dup.corroborated = true } else { deduped.push(f) }
}
log(`${all.length} raw findings -> ${deduped.length} after dedup`)

phase('Verify')
const verified = await parallel(
  deduped.map(f => () =>
    // Verifiers run on Sonnet at medium effort: verification is a bounded re-investigation
    // (read the cited code, try to refute, score) — Sonnet keeps the code-reading judgment at a
    // fraction of the cost of inheriting the session model, and this stage is the token hog
    // (one agent per finding). Finder lenses stay on the session model; they need the depth.
    agent(`${args.verifyPreamble}\n${JSON.stringify(f)}\nRepo: ${args.repo}`,
      { label: `verify:${f.file.split('/').pop()}`, phase: 'Verify', schema: VERDICT, model: 'sonnet', effort: 'medium' })
      .then(v => ({ ...f, score: v ? v.score : 0, verify_reason: v ? v.reason : 'verifier died' }))
  )
)
// Verification LABELS, it does not gate: surface every finding that isn't an outright false
// positive (score < 30 = refuted — pre-existing, linter-caught, or no real scenario), ordered
// by score. The score becomes each finding's confidence tier at report time, not a survival cut.
const reported = verified.filter(Boolean).filter(f => f.score >= 30)
  .sort((a, b) => b.score - a.score)
const dropped = verified.filter(Boolean).filter(f => f.score < 30)
  .map(f => ({ summary: f.summary, score: f.score, reason: f.verify_reason }))
log(`${reported.length} reported (>=30, tiered by score), ${dropped.length} dropped as false positives`)
return { reported, dropped, mentions }
```

After the workflow returns: report **every** item in `reported` via `ReportFindings`, ordered by score — mark score ≥ 80 (or corroborated across lenses) `CONFIRMED` and the rest `PLAUSIBLE`, folding each `PLAUSIBLE` item's score + `verify_reason` into its `failure_scenario`/`summary` so the reader can weigh a lower-confidence finding rather than having it silently dropped. Weave `mentions` and the `dropped` clear-false-positives into the prose overview only where they add signal, and give the verdict. If the workflow is interrupted, resume with `{scriptPath, resumeFromRunId}` rather than relaunching.

Depth knobs: `high` = the always-on lenses as workflow lenses; `xhigh` = + git-history and code-comment lenses; `max` = + prior-PRs lens and a second verification vote per finding (majority of 2-of-3 refuters).
