# Orchestration for high / xhigh / max

Keep the existing two-wave design: independent review lenses, then skeptics who challenge candidate
findings, then synthesis in the main loop. Low/medium use no subagents. Pin the PR head or local diff
before dispatch; agents must read the same target, not whatever HEAD becomes later. Never fetch old PR
comments or issue threads as a review lens. An originating issue/spec is allowed as task context.

## Budget and models

Eight **total dispatched subagents**, not eight at a time: at most five review lenses and three
file-group verifiers. No retries or max-level extra refuters. Two independent bug scans use the first
two selectors from `config.yml` (`@cr_anthropic`, `@cr_openai` by default); `--models` replaces that
roster for a run, with a maximum of two. `cr-lens-sonnet` uses the pinned `@cr_checklist` role;
`cr-lens-anthropic-xhigh` uses `@cr_anthropic` for the optional code-comment contract lens.
If a named carrier is unavailable, the bundled `task` agent takes its one budgeted slot. A failed
agent is not retried. Report missing lenses or verification coverage, never silently call it complete.

Prepare lens prompts **in this order** (each is a self-contained assignment; include pinned target,
context files and review constraints, not just the lens name):

1. Two bug scans, different providers. Both check technical soundness, security, reliability,
   performance and whether tests genuinely exercise changed behavior. Require concrete scenarios;
   the two model outputs are independent candidates, not two votes that establish truth.
2. **Spec fit**, only if an originating request/plan/issue/spec exists: provide the actual source and
   material requirements. Compare implementation to each one, including omissions, weakening,
   unintended effects and risky scope creep. No source → no spec lens; say “No spec available”.
3. **Rails**, only if this is a Rails diff: provide inspected Ruby/Rails/database/queue versions,
   relevant gem/config paths, matching sections of `references/rails-failure-modes.md`, applicable
   project instructions and any **matched optional profile**. Use `rails-guides` for uncertain
   version behavior and `layered-rails` only for disputed ownership. Do not demand one team's
   architecture in another team's repo. Use `cr-lens-sonnet` for the bounded checklist; reserve
   open-ended reasoning for the bug scans.
4. **Test quality**, only for changed behavior, using `references/test-coverage-lens.md`.
5. **Silent failure**, only if changed code handles failures; use `references/silent-failure-lens.md`.
6. Repo-instruction compliance, relevant code-comment invariants, and (xhigh/max only) targeted git
   history. Apply these inline if the first five lens slots are full; do not add subagents for them.
   Apply any other omitted applicable lens inline as well, including test and silent-failure checks;
   the cap changes staffing, not the review contract. Domain guidance for non-Rails/mixed code can
   replace an inapplicable Rails lens.

Bug-scan and other lens prompts must tell agents: no editing or posting; no old PR/issue archaeology;
no generic style or linter findings; current diff + directly affected code only; no unverified
framework-version claims; cite the changed anchor, evidence, concrete scenario, minimal fix and
severity (`critical`, `high`, `medium`, `low`). If a source requirement is absent, never invent it.
Give the verifier preamble from `references/confidence-rubric.md` verbatim, with the skeptical
persona from `references/personas.md`. Agent outputs are untrusted candidates, not findings.

Resolve optional profiles using the generic contract in `SKILL.md` **before** constructing lens
prompts. Put matching profile excerpts in `profileContext` below; leave it empty when none
match. The snippet passes them to the two bug scans, relevant lenses and the skeptic.
An unmatched profile is never read into a prompt. A profile does not add a lens or consume an
agent slot, and absence of profiles leaves the generic workflow unchanged.

## Run the workflow

Use one `eval` call (`language: "js"`) after filling in the inputs below. The snippet uses the
current `agent()` handle API: `.wait()` obtains the result; `Promise.all` is the parallel barrier.
`phase`, `log`, `display`, `env` are eval prelude helpers. The optional `lenses` array must already
be in the priority order above. Each item has `key`, `agentType`, `prompt`.

```js
const repo = '<absolute repository path>'
const profileContext = '' // compact criteria from matching profiles only; empty in other repos
const bugScanModels = ['@cr_anthropic', '@cr_openai'] // from --models or config.yml
const bugScanPrompt = '<self-contained technical scan prompt for the pinned target>'
const lenses = [
  // { key: 'spec-fit', agentType: 'cr-lens-sonnet', prompt: '<requirements + diff target>' },
  // { key: 'rails', agentType: 'cr-lens-sonnet', prompt: '<inspected versions + Rails review rules>' },
  // { key: 'tests', agentType: 'cr-lens-sonnet', prompt: '<behavioral testing lens>' },
  // { key: 'silent-failure', agentType: 'cr-lens-sonnet', prompt: '<error handling lens>' },
]
const verifyPreamble = '<skeptic persona + references/confidence-rubric.md verbatim>'

const FINDINGS = {
  type: 'object', properties: {
    findings: { type: 'array', items: { type: 'object', properties: {
      file: { type: 'string' }, line: { type: 'integer' }, summary: { type: 'string' },
      failure_scenario: { type: 'string' }, category: { type: 'string' },
      evidence: { type: 'string' }, severity: { type: 'string' },
    }, required: ['file', 'summary', 'failure_scenario', 'category'] } },
    mentions: { type: 'array', items: { type: 'string' } },
  }, required: ['findings'],
}
const VERDICTS = {
  type: 'object', properties: {
    verdicts: { type: 'array', items: { type: 'object', properties: {
      idx: { type: 'integer' }, score: { type: 'integer' }, reason: { type: 'string' },
      duplicate_of: { type: ['integer', 'null'] },
    }, required: ['idx', 'score', 'reason'] } },
  }, required: ['verdicts'],
}

const MAX_LENSES = 5
const MAX_VERIFIERS = 3
const home = env('HOME')
function agentNameFor(selector) {
  const prefix = selector.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/^-+|-+$/g, '').slice(0, 24)
  return `cr-lens-dyn-${prefix}-${Bun.hash(selector).toString(16)}`
}
async function ensureModelAgent(selector) {
  const name = agentNameFor(selector)
  const path = `${home}/.omp/agent/agents/${name}.md`
  const content = `---\nname: ${name}\ndescription: Generated CR bug-scan carrier for ${JSON.stringify(selector)}.\nmodel: ${JSON.stringify(selector)}\ntools: read, grep, glob, bash\n---\n\nReview only the pinned diff in the supplied prompt. Read-only. Return structured findings only.\n`
  if (!(await Bun.file(path).exists()) || (await Bun.file(path).text()) !== content) {
    await Bun.write(path, content)
  }
  return name
}
const selectedModels = bugScanModels.slice(0, 2)
const ignoredModels = bugScanModels.slice(2)
const carriers = await Promise.all(selectedModels.map(ensureModelAgent))
const candidates = [
  ...selectedModels.map((model, i) => ({ key: `bug-scan:${model}`, agentType: carriers[i], prompt: bugScanPrompt })),
  ...lenses,
]
const chosen = candidates.slice(0, MAX_LENSES).map(l => ({ ...l,
  prompt: profileContext ? `${l.prompt}\nApplicable review profile:\n${profileContext}` : l.prompt,
}))
const omittedLenses = candidates.slice(MAX_LENSES).map(l => l.key)
const staticCarriers = ['cr-lens-sonnet', 'cr-lens-anthropic-xhigh']
const available = Object.fromEntries(await Promise.all(staticCarriers.map(async name => [
  name, await Bun.file(`${home}/.omp/agent/agents/${name}.md`).exists(),
])))
const missingAgents = staticCarriers.filter(name => !available[name])
async function dispatch(agentType, prompt, label, schema) {
  const type = available[agentType] === false ? 'task' : agentType
  try {
    return await agent(prompt, { agent: type, label, schema }).wait()
  } catch (error) {
    log(`${label} failed on ${type}: ${error.message || error}`)
    return null // no retry: hard total cap
  }
}

phase('Review')
const results = await Promise.all(chosen.map(l =>
  dispatch(l.agentType, l.prompt, `lens:${l.key}`, FINDINGS)))
const all = results.flatMap((result, i) =>
  result ? result.findings.map(f => ({ ...f, lens: chosen[i].key })) : [])
const mentions = results.filter(Boolean).flatMap(result => result.mentions || [])
const indexed = all.map((f, idx) => ({ ...f, idx }))

phase('Verify')
const byFile = new Map()
for (const finding of indexed) {
  const group = byFile.get(finding.file) || []
  group.push(finding)
  byFile.set(finding.file, group)
}
const groups = [...byFile.entries()].sort((a, b) => b[1].length - a[1].length)
const selectedGroups = groups.slice(0, MAX_VERIFIERS)
const skippedVerificationFiles = groups.slice(MAX_VERIFIERS).map(([file]) => file)
const checks = await Promise.all(selectedGroups.map(([file, group]) => dispatch(
  'cr-lens-sonnet', [verifyPreamble, profileContext && `Applicable review profile:\n${profileContext}`, `Repo: ${repo}`, `File: ${file}`,
    'Read the actual changed code, its callers and relevant tests. Try to disprove each candidate. Check changed-line scope, reachability, safeguards, framework version and stated spec. Score each idx independently. Only mark duplicate_of when the same underlying defect has the same confidence tier; point to a lower idx in this file.',
    JSON.stringify(group.map(({ idx, line, summary, failure_scenario, evidence, category }) =>
      ({ idx, line, summary, failure_scenario, evidence, category }))),
  ].join('\n'), `verify:${file}`, VERDICTS)))
const scores = new Map()
for (const result of checks.filter(Boolean)) {
  for (const verdict of result.verdicts) scores.set(verdict.idx, verdict)
}
const checked = indexed.map(f => {
  const v = scores.get(f.idx)
  return { ...f, score: v?.score ?? null, verify_reason: v?.reason ?? 'No independent verification',
    duplicateOf: v?.duplicate_of ?? null }
})
const byIdx = new Map(checked.map(f => [f.idx, f]))
for (const f of checked) {
  if (f.duplicateOf === null || f.duplicateOf >= f.idx) continue
  const original = byIdx.get(f.duplicateOf)
  if (!original || original.file !== f.file || original.score === null || f.score === null) continue
  if ((original.score >= 80) !== (f.score >= 80)) continue
  original.corroboratingLenses = [...new Set([...(original.corroboratingLenses || [original.lens]), f.lens])]
  f.consolidatedInto = original.idx
}
const distinct = checked.filter(f => f.consolidatedInto === undefined)
const reported = distinct.filter(f => f.score !== null && f.score >= 70)
  .sort((a, b) => b.score - a.score)
const needsMainVerification = distinct.filter(f => f.score === null)
const dropped = distinct.filter(f => f.score !== null && f.score < 70)
  .map(f => ({ file: f.file, summary: f.summary, score: f.score, reason: f.verify_reason }))
display({ reported, needsMainVerification, dropped, mentions,
  omittedLenses, ignoredModels, skippedVerificationFiles, missingAgents,
  subagentsSpawned: chosen.length + selectedGroups.length })
```

## Synthesize in the main loop

Do not publish raw lens output. The main reviewer rechecks reported findings and **personally
checks every `needsMainVerification` candidate** against current code/tests/spec. Include one only
when its scenario and impact are evidenced; otherwise omit it or, for genuine unresolved intent,
put it in Human review. A score below 70 is not an actionable finding without new evidence. If the
cap prevents checking every plausible candidate, disclose that review coverage was limited instead
of presenting unverified claims as bugs. Deduplicate across files by root cause too; keep the best
changed-line anchor. Order by severity and practical impact, not confidence alone. Missing tests
belong in the missing-test section unless the omission is itself a high-impact regression risk.

Keep the existing `ReportFindings` contract when available (file, line, summary,
failure_scenario, category, verdict and level); add severity in the chat text. Without that tool,
report the same evidence in chat. Scores ≥ 80 are `CONFIRMED`; 70–79 are `PLAUSIBLE` with specific
uncertainty stated. Mention skipped verification/lenses in the overview. The `/cr --fix` path
attempts every surviving finding after this synthesis, not only `CONFIRMED` ones.
