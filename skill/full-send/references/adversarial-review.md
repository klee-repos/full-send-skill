# Adversarial review — reusable template

This is the highest-value, hardest-to-reinvent piece of the pipeline. It is **always the same
three-stage shape**; only the mechanism (a `Workflow` script vs. parallel subagents) and the lens
set vary by task.

## Contents
- [Why it works](#why-it-works)
- [The invariant shape](#the-invariant-shape)
- [Failure-lens catalog](#failure-lens-catalog)
- [JSON schemas](#json-schemas)
- [Form A — Workflow script (accelerator)](#form-a--workflow-script-accelerator)
- [Form B — parallel subagents (universal)](#form-b--parallel-subagents-universal)
- [After it returns](#after-it-returns)

## Why it works

A single-pass review rubber-stamps: one agent reads a plan, nods, and the load-bearing defect ships.
This design defeats that with **asymmetric skepticism**. Attackers are cheap and many; each is
forced to ground objections in real code. Then *every* raw finding faces a fresh skeptic whose job
is to **refute it by default** — a finding only survives if that skeptic can build a concrete,
reachable failing sequence. This burden-of-proof-on-the-attacker is what separates real defects from
plausible-sounding noise.

Note the asymmetry is easiest to lose on **absence** claims ("X is missing"), which are far harder
to refute than presence claims ("X is broken"). If a round is auditing for gaps, expect the
survival rate to run high and discount it accordingly.

## The invariant shape

1. **ATTACK** — N independent agents (N≈3–6, scaled to surface area), each pinned to **exactly one**
   named failure lens. Hard rules for every attacker:
   - Cite `file:line` (or the exact plan section) for every objection — no ungrounded claims.
   - "Do not invent problems the code/plan already handles." Re-raising a risk the plan already
     mitigates adequately is noise; an *inadequate* mitigation is a valid finding — say why.
   - Prefer a few load-bearing findings over many nitpicks.
2. **VERIFY** — one **fresh** agent per raw finding. Default verdict **REFUTED**. It flips to
   CONFIRMED/PARTIAL only by constructing a concrete, reachable failing sequence from the real code
   (running a throwaway repro probe when useful). It **also assesses the proposed fix** and marks it
   `reject-regressive` if applying it would break something the plan/code already gets right,
   supplying a better `revisedFix`. If a verifier dies or returns unusable output (schema-retry cap,
   subagent failure), its finding is **unverified, not auto-confirmed** — re-run that one or
   spot-check it yourself.
3. **SYNTHESIZE** — one agent dedupes and ranks survivors most-severe-first, emits an **exact edit**
   per survivor, separates architectural findings from detail, and issues **GO / GO-WITH-PATCHES /
   NO-GO**.

Run in the **background** while you do useful prep, if the mechanism supports it. That's a speed win,
not a correctness requirement.

## Failure-lens catalog

Pick the 3–6 lenses that fit the change; give each attacker exactly one. Recurring lenses:

- **regression / prior-shipped-behavior** — does this break something that currently works?
- **security & blast-radius** — secret exposure, auth/ACL, over-broad scope, data loss.
- **e2e-contract / determinism** — does the validation actually pin the claim? flaky/racey? does the
  test assert real state or a mock?
- **grounding-reality** — are the plan's factual premises true against real code/data? miscitations?
- **scope-coherence** — internal contradictions; does the plan do what the user asked and no more?
- **native-timing / lifecycle** — mount/unmount, animation, permission dialogs, autoplay gates.
- **cost / metering / irreversibility** — billable or one-way actions taken before proof.
- **test-integrity / no-fake-green** *(universal — fire on ANY change that adds or edits tests)* —
  attack every new/edited test and its seam. Assert the sink is a **sanctioned mechanism** (a
  recorded/replayed tape or a live call — never a stub that returns success) and the assertion pins
  **real observable state**, not a mock or an app internal. This is DISTINCT from any product-level
  "honesty gate" lens: a product honesty gate (e.g. "the narrator must hedge unless grounded") is
  project-specific and may be absent; the test-integrity lens is universal.

The catalog above came out of mobile + AI-output product work — **reinterpret the lenses for the
project in front of you** (its invariants come from its `CLAUDE.md`/README/tests).

## JSON schemas

Real JSON Schema, not illustrations. On Form A pass them directly as the `schema:` argument. On
Form B a subagent has no schema parameter — paste the shape into the prompt and end with
`Output ONLY this JSON, no prose:`, then parse and re-ask once on malformed output. The `enum`s are
the interface either way: they are what constrain a verifier to a verdict rather than an essay.

Every schema carries `status` + `statusNote` (see `references/run-ledger-and-supervision.md`). It is
what makes a dead agent's `{findings: []}` distinguishable from a lens that genuinely found nothing —
without it, the failure mode of this phase is a *quieter, cleaner-looking* report.

```js
const RUN_STATUS = {   // spread into every schema below
  status:     { type: 'string', enum: ['complete', 'partial', 'blocked'] },
  statusNote: { type: 'string', description: 'what went uncovered, or what blocked it' },
}

const ATTACK_SCHEMA = {
  type: 'object', additionalProperties: false, required: ['findings', 'status', 'statusNote'],
  properties: { ...RUN_STATUS, findings: { type: 'array', items: {
    type: 'object', additionalProperties: false,
    required: ['id','title','severity','evidence','breakScenario','proposedFix'],
    properties: {
      id: { type: 'string' }, title: { type: 'string' },
      severity: { type: 'string', enum: ['blocker','major','minor'] },
      evidence: { type: 'string', description: 'file:line or plan-section quote grounding it' },
      breakScenario: { type: 'string', description: 'the concrete way this bites in practice' },
      proposedFix: { type: 'string', description: 'a specific, minimal change' },
    } } } },
}

const VERIFY_SCHEMA = {
  type: 'object', additionalProperties: false,
  required: ['verdict','whyConcrete','fixAssessment','revisedFix','status','statusNote'],
  properties: {
    ...RUN_STATUS,
    verdict: { type: 'string', enum: ['CONFIRMED','PARTIAL','REFUTED'] },
    whyConcrete: { type: 'string', description: 'the reachable failing sequence, OR why it is a false positive / already handled' },
    fixAssessment: { type: 'string', enum: ['keep','revise','reject-regressive'] },
    revisedFix: { type: 'string', description: 'the fix to actually apply; empty if REFUTED' },
  },
}

const SYNTH_SCHEMA = {
  type: 'object', additionalProperties: false,
  required: ['decision','survivorsRanked','rawCount','survivorCount','droppedNote','summaryForUser',
             'status','statusNote'],
  properties: {
    ...RUN_STATUS,
    decision: { type: 'string', enum: ['GO','GO-WITH-PATCHES','NO-GO'] },
    survivorsRanked: { type: 'array', items: {
      type: 'object', additionalProperties: false,
      required: ['title','severity','lens','exactPlanEdit','isArchitectural'],
      properties: {
        title: { type: 'string' },
        severity: { type: 'string', enum: ['blocker','major','minor'] },
        lens: { type: 'string' },
        exactPlanEdit: { type: 'string' },
        isArchitectural: { type: 'boolean' },
      } } },
    rawCount: { type: 'number' }, survivorCount: { type: 'number' },
    droppedNote: { type: 'string' },
    summaryForUser: { type: 'string', description: '3-6 sentence verdict + must-fix edits' },
  },
}
```

`rawCount`/`survivorCount` are in the synthesizer schema so the survival ratio is surfaced
structurally instead of eyeballed — and because this object is what gets persisted. The synthesizer
only ever sees **survivors**, so both counts must be stated in its prompt for it to echo: a required
field it cannot derive gets fabricated, and a fabricated ratio defeats the smell test that reads it.
Same for `droppedNote`.

## Form A — Workflow script (accelerator)

Use when the `Workflow` tool is available. This is the canonical pipeline shape (each finding
verifies as soon as its lens returns; no barrier until synthesis):

```js
export const meta = {
  name: 'adversarial-review',
  description: 'Attack -> verify(default REFUTED) -> synthesize over a plan or a diff',
  phases: [{ title: 'Attack' }, { title: 'Verify' }, { title: 'Synthesize' }],
}
const RUN    = '<abs path to the run directory>'   // holds LEDGER.md and agents/
const TARGET = '<abs path to the plan file OR a diff/dir to attack>'
const GROUND = '<abs path(s) to ground-truth the attackers must read: real source, the RCA, specs>'
const ROUND  = 'R1'
const LENSES = [ /* {key, brief} — one failure lens each, per the catalog */ ]
// ATTACK_SCHEMA / VERIFY_SCHEMA / SYNTH_SCHEMA: paste the three objects from "JSON schemas" above.
// Shared preamble — a subagent inherits nothing, so every prompt carries it (see run-ledger doc).
// The synthesize call below carries a trimmed variant, for the reason stated at its call site.
const HEAD = (out) =>
  `READ ${RUN}/LEDGER.md FIRST — it is this run's shared ground truth and constraints; do not `+
  `re-derive what it states, do not violate a Constraint, and do not re-raise a claim listed as `+
  `already refuted. Append a heartbeat line to ${out}.progress at each milestone. Write your full `+
  `output to ${out}. Budget ~25 tool calls; on hitting it emit status:"partial" and stop.\n`

phase('Attack')
const results = await pipeline(
  LENSES,
  (lens) => agent(
    HEAD(`${RUN}/agents/${ROUND}-attack-${lens.key}.json`) +
    `Hostile expert reviewer. Attack ${TARGET} through ONLY this lens: ${lens.brief}. `+
    `READ ${GROUND} first. Cite file:line for every objection. Do NOT invent problems already handled. `+
    `A few load-bearing findings beat many nitpicks.`,
    { label: `attack:${lens.key}`, phase: 'Attack', schema: ATTACK_SCHEMA, effort: 'high' }
  ).then(r => ({ lens, ranToEnd: r?.status === 'complete',
                 findings: (r?.findings||[]).map(f => ({ ...f, lens: lens.key })) }),
         () => ({ lens, ranToEnd: false, findings: [] })),
  (attacked) => parallel(attacked.findings.map(f => () =>
    agent(
      HEAD(`${RUN}/agents/${ROUND}-verify-${f.id}.json`) +
      `Re-litigate ONE finding. DEFAULT verdict REFUTED unless you can build a concrete reachable `+
      `failure from real code (read ${GROUND}). Also reject the fix if it would regress prior behavior.\n`+
      `FINDING(${f.lens}/${f.severity}): ${f.title}\nEvidence: ${f.evidence}\nBreak: ${f.breakScenario}\nFix: ${f.proposedFix}`,
      { label: `verify:${f.id}`, phase: 'Verify', schema: VERIFY_SCHEMA, effort: 'high' }
    ).then(v => (v?.verdict ? { ...f, ...v } : { ...f, verdict: null }), () => ({ ...f, verdict: null }))
  )).then(vs => ({ lens: attacked.lens, ranToEnd: attacked.ranToEnd, verified: vs }))
)
const lensRuns = results.filter(Boolean)
const ranOk = new Set(lensRuns.filter(r => r.ranToEnd).map(r => r.lens.key))
// A lens that did not run to its end — or that vanished from the pipeline entirely — is a HOLE in
// coverage, not a clean lens. Derive from LENSES, not from what came back: a lens whose whole chain
// threw leaves no result to notice missing. Re-run it narrower or cover it yourself.
const deadLenses = LENSES.map(l => l.key).filter(k => !ranOk.has(k))
const raw = lensRuns.flatMap(r => r.verified).filter(Boolean)
const DECIDED = new Set(['CONFIRMED', 'PARTIAL', 'REFUTED'])
// A verifier that died or returned unusable output leaves its finding UNVERIFIED — never a free pass.
const survivors  = raw.filter(f => f.verdict === 'CONFIRMED' || f.verdict === 'PARTIAL')
const refuted    = raw.filter(f => f.verdict === 'REFUTED')
const unverified = raw.filter(f => !DECIDED.has(f.verdict))

phase('Synthesize')
const verdict = await agent(
  // Deliberately NOT HEAD: this agent's deliverable is the returned object, not a file, so the
  // output-path/heartbeat/budget lines do not apply. It still reads the ledger — `summaryForUser`
  // is the prose the user actually reads, and it is the last prose written before the gate closes.
  `READ ${RUN}/LEDGER.md FIRST. Write summaryForUser, droppedNote and every exactPlanEdit to its `+
  `Style entry; copy quoted values and file:line cites verbatim.\n`+
  `Dedupe+rank survivors most-severe-first; emit an exact edit per survivor; separate architectural `+
  `from detail; issue GO/GO-WITH-PATCHES/NO-GO.\n`+
  `Echo these counts verbatim, do NOT recompute: rawCount=${raw.length}, survivorCount=${survivors.length}. `+
  `${refuted.length} findings were REFUTED and dropped before you saw them. `+
  `${unverified.length} had a dead or unusable verifier and are UNVERIFIED — neither survivor nor `+
  `refuted; state that count in droppedNote. These lenses never ran to completion and are UNCOVERED, `+
  `not clean — name them in droppedNote too: ${JSON.stringify(deadLenses)}.\n`+
  `SURVIVORS:\n${JSON.stringify(survivors, null, 2)}`,
  { label: 'synthesize', phase: 'Synthesize', effort: 'high', schema: SYNTH_SCHEMA }
)
return { survivors, unverified, deadLenses, verdict }
```

Notes: pass a per-finding **fresh** verifier (a new `agent()` call = fresh context). Give attackers +
verifiers `effort: 'high'` — this is the phase where reasoning depth pays. Every `schema:` argument
must be a real schema object; `{}` enforces nothing, and the synthesize stage is the one that emits
the GO/NO-GO.

`!== 'REFUTED'` is **not** the survivor test. A verifier that dies spreads no verdict, so that filter
silently promotes an unadjudicated attacker claim straight into the plan — the exact inversion of
this phase's thesis. Use the CONFIRMED/PARTIAL allowlist, and spot-check anything `unverified`
yourself before folding it in. Likewise a dead *attacker* returns `{findings: []}`, which is
indistinguishable from a lens that found nothing — which is why `status` is required and
`ranToEnd`/`deadLenses` exist: a lens that did not complete is **uncovered**, and must be re-run
narrower or covered by you before the survival ratio means anything.

**Workflow does not exempt you from supervision.** A backgrounded script is still a wave: watch it
with `/workflows`, and if a stage sits with no progress, kill the run (`TaskStop`) and resume it
(`Workflow({scriptPath, resumeFromRunId})` — completed `agent()` calls return from cache, so only the
stalled one re-runs). Read `<transcriptDir>/journal.jsonl` before concluding a stage returned nothing;
a cached empty result and a real empty result look identical in the return value alone.

## Form B — parallel subagents (universal)

Use when there is no `Workflow` tool (the common case in most sessions). Same shape, three turns:

Every prompt in all three turns opens with the five lines from
`references/run-ledger-and-supervision.md` — read `LEDGER.md` first, write your output file, append
heartbeats, respect the budget, return a receipt — and every turn is a **supervised wave**: manifest
at spawn, poll, kill no-progress agents, close with every member accounted for.

1. **Attack turn:** spawn the N attacker subagents (one `Agent`/`Task` call each, all in ONE
   message so they run concurrently), each with its single lens, the ground-truth paths to read, the
   `file:line` mandate, and the "don't invent handled problems" rule. Each writes its `findings[]`
   JSON to `<run>/agents/<round>-attack-<lens>.json`. Read the files, not the replies. A lens whose
   file is missing or whose `status` isn't `complete` is **uncovered** — re-spawn it once narrower,
   then run that lens yourself.
2. **Verify turn:** for each raw finding, spawn a **fresh** subagent (again, all in one message),
   default-REFUTED, fix-regression-check included. Collect verdicts; keep only CONFIRMED/PARTIAL.
   A verifier that died, stalled, or returned unparseable output leaves its finding **unverified** —
   re-run or spot-check that one yourself, never fold it in. This is the stage where a stall is most
   expensive: 20 findings means 20 agents, and the ones that quietly never return are exactly the
   ones nobody misses.
3. **Synthesize turn:** one subagent (or do it yourself) dedupes/ranks survivors, emits exact edits,
   issues GO/GO-WITH-PATCHES/NO-GO. If you spawn a subagent, hand it the raw-finding count, the
   survivor count, and the uncovered-lens list explicitly — it sees only survivors and would
   otherwise invent `rawCount` and report full coverage it never had.

This path runs anywhere and preserves real parallelism within each stage — never degrade to running
the attackers serially in the foreground. On the do-it-yourself synthesize branch, still produce the
`SYNTH_SCHEMA` object rather than prose: it is what the next section persists.

## After it returns

- **Independently spot-verify the top ~2 survivors** against real source. Reviewer agents can
  miscite; the highest-stakes edits deserve your own eyes.
- **Record refuted claims in the ledger's `Decisions`**, not just the survivors. R2's attackers read
  that file; without it they re-raise what R1 already killed, and the review loops without
  converging — the single most common way this phase burns rounds on no progress.
- **Write the synthesizer's object verbatim to `<NAME>-adversarial.json`** beside the plan — on
  either form, including the do-it-yourself branch. This gate is the most expensive phase in the
  pipeline and otherwise persists nothing: the file is what survives a compaction or a fresh
  session. Then fold each `survivorsRanked` entry into the plan as an edit.
- Fold survivors into the plan via edits; mark it `ENG[+DESIGN]+ADVERSARIAL CLEARED`.
- On `NO-GO`, revise and **loop** — re-run the review, each round stricter about rejecting
  unreachable/theoretical findings, until a round lands nothing on the crash-causing invariants.
- This same template applies standalone to any risky diff or bugfix, not just a plan — it's a general
  habit, not a plan-only step.
