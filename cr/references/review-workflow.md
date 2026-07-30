# Workflow orchestration for high / xhigh / max

At `high` and above, run the lens fan-out + verification through the Workflow tool instead of ad-hoc Agent calls. The `/cr` invocation at these levels IS the user's opt-in to multi-agent orchestration. Synthesis (overview, `ReportFindings`, verdict) stays in the main loop — it needs conversation context.

Build the lens prompts first (diff path, repo path, lens instructions, assigned persona, false-positive pre-filter, per SKILL.md), then invoke Workflow with the script below, passing `args`:

```json
{
  "repo": "<abs repo path>",
  "lenses": [
    {"key": "bug-scan", "persona": "perfectionist", "prompt": "<full self-contained lens prompt>", "model": "opus", "effort": "xhigh"},
    {"key": "claude-md", "persona": "maintainer", "prompt": "...", "model": "sonnet", "effort": "medium"}
  ],
  "verifyPreamble": "<the block from references/confidence-rubric.md, verbatim>",
  "extraVotes": 0
}
```

`extraVotes` is `0` at high/xhigh and `2` at `max`.

**Model tiering** (set per lens in `args.lenses`). Only the deep semantic lenses are worth Opus:

- `bug-scan` and `comment-contracts` — and the adversary `bug-scan` pass — get `model: "opus", effort: "xhigh"`. Reading changed code for real defects is where reasoning depth converts into caught bugs.
- Every other lens (`claude-md`, `git-history`, `silent-failure`, `test-coverage`, `prior-prs`, domain lenses) gets `model: "sonnet", effort: "medium"`. These sweep against a written rubric — checklist- and retrieval-shaped work — and the verify stage gates their output anyway, so a full-depth Opus pass buys little and costs a lot.

Verifiers are hardcoded below to Sonnet medium, refuters to Sonnet high. Persona assignment per lens is in `references/personas.md`; verifiers and refuters always run as the `skeptic`.

**Verification is batched per file, not per finding.** One agent reads a file once and scores every finding in it. Re-reading the same file once per finding was the dominant verifier cost and bought nothing — the verifier's evidence is the same file either way.

```js
export const meta = {
  name: 'cr-review',
  description: 'Code-review lens fan-out with per-file batched adversarial verification',
  phases: [
    { title: 'Review', detail: 'one agent per lens, each in its assigned persona' },
    { title: 'Verify', detail: 'one agent per file scores every finding in that file' },
    { title: 'Refute', detail: 'extra votes on the ambiguous band only (max)' },
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
// One verifier per FILE returns one verdict per finding id it was handed.
const VERDICTS = {
  type: 'object',
  properties: {
    verdicts: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          id: { type: 'string' }, score: { type: 'integer' }, reason: { type: 'string' },
        },
        required: ['id', 'score', 'reason'],
      },
    },
  },
  required: ['verdicts'],
}

const payload = f => ({
  id: f.id, file: f.file, line: f.line, summary: f.summary,
  failure_scenario: f.failure_scenario, category: f.category, evidence: f.evidence,
})

phase('Review')
// Barrier (not pipeline) is deliberate: dedup across ALL lenses before paying for verification.
const results = await parallel(
  args.lenses.map(l => () => agent(l.prompt, {
    label: `${l.persona || 'default'}:${l.key}`, phase: 'Review', schema: FINDINGS,
    // Per-lens tiering (see Model tiering above): null/undefined inherits the session model.
    ...(l.model ? { model: l.model } : {}), ...(l.effort ? { effort: l.effort } : {}),
  }))
)
const all = []
const mentions = []
results.forEach((r, i) => {
  if (!r) return
  const l = args.lenses[i]
  for (const f of r.findings || []) all.push({ ...f, lens: l.key, persona: l.persona || 'default' })
  for (const m of r.mentions || []) mentions.push(m)
})

// Dedup: same file + nearby line = same finding; keep the first, note the corroborating lens.
// Corroboration across DIFFERENT personas is the strongest signal the fan-out produces.
const deduped = []
for (const f of all) {
  const dup = deduped.find(d => d.file === f.file && Math.abs((d.line || 0) - (f.line || 0)) <= 3)
  if (dup) {
    if (!dup.corroborating_lenses.includes(f.lens)) {
      dup.corroborating_lenses.push(f.lens)
      dup.corroborated = true
    }
  } else {
    deduped.push({ ...f, id: `f${deduped.length}`, corroborated: false, corroborating_lenses: [f.lens] })
  }
}
log(`${all.length} raw findings -> ${deduped.length} after dedup`)

phase('Verify')
// Group by file: one verifier reads each file ONCE and scores every finding in it.
const byFile = []
for (const f of deduped) {
  const g = byFile.find(x => x.file === f.file)
  if (g) g.findings.push(f)
  else byFile.push({ file: f.file, findings: [f] })
}
log(`verifying ${deduped.length} finding(s) in ${byFile.length} per-file batch(es)`)

const batches = await parallel(
  byFile.map(g => () =>
    agent(
      `${args.verifyPreamble}\n\nRepo: ${args.repo}\nFile under review: ${g.file}\n` +
      `Read this file once, then score EVERY finding below independently — a weak finding next to ` +
      `a strong one must not drag it up or down. Return exactly one verdict per id.\n` +
      JSON.stringify(g.findings.map(payload)),
      { label: `verify:${g.file.split('/').pop()}`, phase: 'Verify', schema: VERDICTS, model: 'sonnet', effort: 'medium' }
    )
      .then(v => ({ verdicts: (v && v.verdicts) || [] }))
      .catch(() => ({ verdicts: [] }))
  )
)

// A dead or incomplete verifier leaves a finding UNVERIFIED (score null) — never silently refuted.
// Scoring it 0 here would drop a real finding at the `< 30` cut below with no evidence against it.
const verdictById = new Map()
for (const b of batches.filter(Boolean)) {
  for (const v of b.verdicts) {
    if (v && typeof v.score === 'number') verdictById.set(v.id, v)
  }
}
let scored = deduped.map(f => {
  const v = verdictById.get(f.id)
  return v
    ? { ...f, score: v.score, verify_reason: v.reason }
    : { ...f, score: null, verify_reason: 'unverified — no verdict returned for this finding' }
})
const unverified = scored.filter(f => f.score === null).length
if (unverified) log(`${unverified} finding(s) unverified -> reported as PLAUSIBLE, not dropped`)

const EXTRA = args.extraVotes || 0
if (EXTRA > 0) {
  phase('Refute')
  // Only the ambiguous band pays for extra votes. >= 80 is already CONFIRMED and < 30 is already
  // refuted; more votes move neither. Unverified findings join the band — no score is maximally
  // ambiguous. Per-finding (not per-file) here: independent votes are the point, and the band is
  // small enough that the re-read cost is worth the independence.
  const ambiguous = scored.filter(f => f.score === null || (f.score >= 30 && f.score < 80))
  log(`${ambiguous.length}/${scored.length} in the ambiguous band -> ${EXTRA} extra vote(s) each`)
  const revotes = await parallel(
    ambiguous.map(f => () =>
      parallel(Array.from({ length: EXTRA }, (_, i) => () =>
        agent(
          `${args.verifyPreamble}\n\nRepo: ${args.repo}\nIndependent refutation pass ${i + 1}.\n` +
          `Return exactly one verdict, for id ${f.id}.\n${JSON.stringify(payload(f))}`,
          { label: `refute${i + 1}:${f.id}`, phase: 'Refute', schema: VERDICTS, model: 'sonnet', effort: 'high' }
        )
      )).then(vs => ({
        id: f.id,
        extra: vs.filter(Boolean).flatMap(v => v.verdicts || [])
          .filter(v => typeof v.score === 'number').map(v => v.score),
      }))
    )
  )
  const extraById = new Map()
  for (const r of revotes.filter(Boolean)) extraById.set(r.id, r.extra)
  scored = scored.map(f => {
    const extra = extraById.get(f.id)
    if (!extra || !extra.length) return f
    // Median of all votes = the 2-of-3 majority, generalised, and robust to one outlier refuter.
    const votes = (f.score === null ? extra : [f.score, ...extra]).slice().sort((a, b) => a - b)
    const mid = votes.length >> 1
    const median = votes.length % 2 ? votes[mid] : Math.round((votes[mid - 1] + votes[mid]) / 2)
    return { ...f, score: median, verify_reason: `${f.verify_reason} | ${extra.length} extra refuter(s) [${extra.join(', ')}] -> median ${median}` }
  })
}

// Verification LABELS, it does not gate — the rule and the tiers live in SKILL.md ("Report").
// Only outright refutations (score < 30) are dropped. Unverified (null) findings are reported.
const rank = f => (f.score === null ? 55 : f.score)
const reported = scored.filter(f => f.score === null || f.score >= 30).sort((a, b) => rank(b) - rank(a))
const dropped = scored.filter(f => f.score !== null && f.score < 30)
  .map(f => ({ summary: f.summary, score: f.score, reason: f.verify_reason }))
log(`${reported.length} reported (tiered by score), ${dropped.length} dropped as false positives`)
return { reported, dropped, mentions }
```

After the workflow returns: report **every** item in `reported` via `ReportFindings`, ordered by score, applying the tiers from SKILL.md's "Report" section (`score >= 80` or `corroborated` → `CONFIRMED`; 30–79 and `score === null` → `PLAUSIBLE`, folding the score/`verify_reason` into `summary`/`failure_scenario`). Weave `mentions` and the `dropped` false positives into the prose overview only where they add signal, and give the verdict. If the workflow is interrupted, resume with `{scriptPath, resumeFromRunId}` rather than relaunching.

Depth knobs: `high` = the always-on lenses, `extraVotes: 0`; `xhigh` = + git-history and code-comment lenses; `max` = + prior-PRs lens, the adversary `bug-scan` pass, and `extraVotes: 2` on the ambiguous band only.
