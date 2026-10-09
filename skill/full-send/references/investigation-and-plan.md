# Ground-truth investigation + the scratchpad plan

Two artifacts feed the review gates: a **verified** picture of reality, and a plan built on it. Both
must be real, not remembered.

## Ground-truth fan-out (features)

Spawn 2–6 read-only mapping subagents in ONE turn (concurrent), each pinned to a single subsystem.
The point of the strict schema is to force `file:line` grounding so the plan can't drift into fiction.

Per-agent instruction shape (the five lines below are the shared preamble from
`references/run-ledger-and-supervision.md` — a subagent inherits none of your context):

> READ `<run>/LEDGER.md` FIRST — this run's goal, constraints, and already-verified facts. Do not
> re-derive what it states.
> WRITE your full output to `<run>/agents/map-<subsystem>.json`; APPEND a heartbeat line to
> `<run>/agents/map-<subsystem>.progress` at each milestone (started / read <file> / N claims /
> writing). BUDGET ~25 tool calls — on hitting it, emit `"status":"partial"`, open `statusNote`
> with `budget:`, name what you did not reach, and stop. RETURN only the path, `status`, and a ≤3-line summary.
>
> You are mapping ONE subsystem for a change. READ-ONLY — do not edit anything outside your own two
> files above. Trace `<subsystem>` and report exactly how it works today, with `file:line` for every
> claim. Be exhaustive and factual; cite, do not summarize from memory. Flag anything that
> contradicts the stated goal.

Output schema:
```json
{ "subsystem": "...",
  "status": "complete | partial | blocked",
  "statusNote": "what went uncovered, or what blocked it",
  "howItWorksToday": [ { "claim": "...", "cite": "path/to/file.ts:120-134" } ],
  "entryPoints": [ "file:line — every path into the changed behavior; flag each parallel-path left on the old logic" ],
  "invariantsObserved": [ "a product rule, a deployment identifier, a testing convention — each with a file:line cite" ],
  "risksForThisChange": [ "..." ],
  "openQuestions": [ "..." ] }
```

**Supervise the wave to close.** Manifest each mapper at spawn; poll the heartbeats; kill and
re-spawn narrower any mapper with no new progress across two checks; map the last stubborn subsystem
yourself. A subsystem that silently went unmapped is a hole in the plan's foundation, and it will
surface as a "surprise" during implementation rather than as a missing map.

Then **re-verify the 2 riskiest claims yourself** with a direct `Read` before trusting them —
reader agents can miscite, and a plan built on a miscitation is worse than none. **Promote only what
you verified into the ledger's `Ground truth`**; everything else stays in the mapper's file. Read
the decision record as well as the code: issue
history, change descriptions, design notes, a decision log, earlier sessions. It does not replace
the code. Write "no decision record" in the ledger if none exists. Do not re-open a settled
decision.

## Formal RCA (bugs)

Do NOT read source first. Pull the **real failing artifact** and prove the trace end to end:

- Copy the real database and query the actual persisted rows (don't infer what "should" be stored).
- Read real logs / traces (grab a `trace_id` off a failing line and pull every correlated line) /
  the real build output / a hierarchy dump + screenshot.
- Grep the actual SDK/vendor headers for the real error semantics.

Write the RCA to the scratchpad as `<NAME>-RCA.md`, with each step of the trace marked verified. The
RCA doubles as the plan's foundation. Only after the root cause is proven do you author the fix
plan. A source fix does not repair the data the
defect already wrote. Repair that data, or say why it needs no repair.

**Pin the repro before you fix.** The RCA must end with the exact sequence that reproduces the
failure and the observed wrong value — run it and record the output *while the bug is still live*.
That sequence is what Step 6's regression test asserts against and what Step 7 reverts to force red;
neither has an antecedent without it. If the failure cannot be reproduced on demand, say so
explicitly — an unreproducible bug cannot be proven fixed, only unobserved.

## The scratchpad plan skeleton

Write `<NAME>-PLAN.md` into the run directory created in Step 0 (scratchpad → else
`/tmp/full-send-<slug>/`). **Never the repo tree, never git-committed.**

```markdown
# <NAME> — PLAN
## Goal & scope   (what, and explicitly what's out of scope: `NOT in scope`, a reason per item)
## Constraints    (hard user constraints: design source, prod id, hosting, "no separate screen", …)
## Ground truth   (the verified facts this rests on, with file:line)
## What already exists   (each sub-problem, the code that solves part of it, reused or rebuilt)
## Fixes
### F1 — <title>   (files touched, exact change, why)
### F2 — <title>
## Design                 (UI work only: the Step 2 reference set and direction; then gate A's design lens)
## Developer experience   (developer-facing work only: gate A's DX lens fills it)
## Failure modes          (each failure, tested, handled, what the user sees; critical gaps flagged)
## Test-coverage tree
- path/to/fileA     [UNIT]        — <what it proves>
- path/to/screen    [E2E-web]     — <the deterministic assertion>
- path/to/native    [E2E-native]  — <gesture/permission fidelity>
- path/to/generator [JUDGE]       — <the deterministic hard-gate; quality score report-only>
- <native-only>     [DEVICE]      — <flagged: un-provable by web/CLI E2E>
## Migration / back-compat
## Risks & open forks   (AskUserQuestion only for genuine scope/architecture forks)
## Review status   (gate A: CLEAR / CLEAR-WITH-GAPS / NEEDS-EDITS · gate B verdict → `<NAME>-adversarial.json`)
## Review record   (ALWAYS LAST: which lenses fired, the pass lines, the decision log, the themes)
```

The tier tags are a default vocabulary — **use the tiers this project actually has**. The
`Review status` line is what Steps 4–5 clear before any implementation code is written.

Everything above `## Review record` is the **plan body**. Gate A's reviewers and Step 6 read the
plan body only, so a requirement must stand in the body. Gate A fills the sections it owns and
writes its analysis below the line (`plan-review.md`). Leave a section empty until its lens runs,
with one exception: at Step 3, write the Step 2 reference set, the direction, and the path of each
image the user gave into `## Design`. The design reviewer has no other way to read them.
