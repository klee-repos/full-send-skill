# Run ledger & agent supervision

Every fan-out phase in this skill depends on two things that are easy to lose and expensive to lose
quietly: **shared state** and **liveness**.

State kept only in the orchestrator's context never reaches the agents at all — each one starts
empty, re-derives ground truth from scratch, drifts to a slightly different picture of reality, and
the round that follows is assembled from N inconsistent worlds. The same state is also the first
thing a compaction takes, so a long run forgets its own constraints mid-pipeline.

Liveness fails the other way. An agent that hangs looks exactly like one that is thinking, and an
agent that died looks exactly like a lens that found nothing. Both emit no signal, so both read as
*clean*. That is this pipeline's quietest failure: a confident report whose coverage is missing
members nobody counted.

Two invariants close them:

1. **State lives in files. Your context is a cache of those files, never the system of record.**
2. **Every spawned agent is supervised until it is accounted for.**

## Contents
- [The run directory](#the-run-directory)
- [LEDGER.md — the shared brief](#ledgermd--the-shared-brief)
- [The five lines every agent prompt carries](#the-five-lines-every-agent-prompt-carries)
- [`status` is a required field, everywhere](#status-is-a-required-field-everywhere)
- [The supervision loop](#the-supervision-loop)
- [Macro-loop anti-stall](#macro-loop-anti-stall)

## The run directory

Created in **Step 0, before the first agent is spawned** — not when the first artifact happens to
need a home. Locate it in this order: the harness-provided scratchpad if one exists; else
`/tmp/full-send-<slug>/`; else any writable dir outside the repo. **Never the repo tree, never
git-committed** — all of it is scratch.

```
<run>/
├── LEDGER.md                     # the canonical shared brief — every agent reads this FIRST
├── <NAME>-RCA.md                 # bugs only
├── <NAME>-PLAN.md
├── <NAME>-adversarial.json       # the synthesizer object, per round
├── proof/                        # proof records, driver scripts, logs, screenshots
└── agents/
    ├── MANIFEST.jsonl            # one line per spawned agent (see below)
    ├── <phase>-<key>.json        # that agent's FULL output — the agent writes it
    └── <phase>-<key>.progress    # that agent's append-only heartbeat — the agent writes it
```

One `MANIFEST.jsonl` line per agent, appended **at spawn time**, not at return:

```json
{"id":"attack-regression","phase":"adversarial-R1","brief":"regression lens","out":"agents/adversarial-R1-regression.json","status":"spawned"}
```

What is not recorded cannot be noticed missing. A wave of six where five return reads as complete
unless something written down says six were expected.

## LEDGER.md — the shared brief

```markdown
# <NAME> — RUN LEDGER
## Run          one-line goal; what is explicitly out of scope
## Constraints  the user's hard constraints, verbatim (design source, prod identifier, "no new screen")
## Style        the writing standard for every prose field — the rules copied out, never a pointer to them
## Phase        current phase; which gates are cleared
## Ground truth VERIFIED facts only, each with file:line
## Decisions    settled forks + why — INCLUDING claims already refuted, so no later wave re-raises them
## Artifacts    paths: RCA, plan, adversarial JSON, proof records, driver command
## Open         what is still unknown or uncovered
```

Rules that make it trustworthy:

- **You are the only writer.** Agents read it and write their own files. Concurrent writers corrupt
  it, and a corrupted shared brief poisons every agent downstream at once.
- **Update it at every phase boundary, and before spawning the next wave** — after folding survivors,
  after re-verifying a claim, after a constraint is discovered. A stale ledger is worse than no
  ledger, because agents will trust it.
- **Only verified facts reach `Ground truth`** — something you `Read` yourself, or an agent claim you
  spot-checked. Unverified agent claims stay in the agent's own file until they earn promotion.
- **It is a brief, not a transcript.** Keep it readable in ~30 seconds; link to files for depth. A
  ledger too long to read stops being read, which is the same as not having one.
- **`Style` is copied out in full, never referenced.** A subagent reads this file, not `SKILL.md` — a
  pointer it will not follow is not a transmitted rule, and "see `SKILL.md`" written into this row is
  the failure, not the rule. Copy the whole of `SKILL.md` § Writing standard into it: the rule list
  **and** the paragraph naming what is exempt. The exemption is the half that protects quoted values
  and `file:line` cites, so a copy that drops it is worse than no copy. This is fixed boilerplate of a
  known size, so it does not compete with the 30-second read the way a growing section would.
- **Refuted claims are load-bearing.** Recording them is what stops round R2's attackers from
  re-litigating exactly what R1 already killed — the most common form of a loop that spins without
  progressing.

## The five lines every agent prompt carries

A subagent inherits none of your context, so these travel *inside* every spawned prompt — mapper,
attacker, verifier, synthesizer, driver:

1. **READ `<run>/LEDGER.md` FIRST.** It is this run's shared ground truth. Do not re-derive what it
   already states, do not propose anything that violates a listed Constraint, and write every prose
   field you emit — claims, findings, notes, `statusNote` — to its `Style` entry.
2. **WRITE your full output to `<run>/agents/<phase>-<key>.json`.** That file is your deliverable.
   (Read-only agents are read-only *on the project* — their own two files under `agents/` are the
   only writes they may make.)
3. **APPEND a heartbeat line to `<run>/agents/<phase>-<key>.progress` at every milestone** — started,
   read <file>, found N candidates, verifying, writing output. This is how you are known to be alive:
   an agent with no new line across two checks is treated as hung and killed.
4. **BUDGET: ~<N> tool calls / <M> files.** On hitting it, write what you have with
   `"status":"partial"`, note what went uncovered, and stop. Do not keep searching for a better
   answer past the budget.
5. **RETURN only** the output path, your `status`, and a ≤3-line summary. The file is the
   deliverable; your reply is a receipt.

The receipt rule is not tidiness. Six agents returning 4KB of JSON each is precisely what pushes the
orchestrator into a compaction mid-round — the event that loses the run. Files survive compaction;
returned blobs are what causes it.

## `status` is a required field, everywhere

Add to every schema this skill defines (attack, verify, synthesize, mapping, e2e report):

```json
"status":     { "type": "string", "enum": ["complete", "partial", "blocked"] },
"statusNote": { "type": "string", "description": "what went uncovered, or what blocked it" }
```

Without it, `{"findings": []}` from an agent that died is indistinguishable from a lens that
genuinely found nothing — and *empty output silently increases confidence*. `complete` means the
work actually ran to its end. **Absent output is never `complete`.**

The accounting rule, which generalizes the verifier rule already in
`references/adversarial-review.md`: **absent, malformed, or blocked output is `unverified` — never a
pass, and never a clean bill of health.** Re-run it once, narrower; if it fails again, do that slice
yourself and say so.

## The supervision loop

After spawning a wave (all in ONE message, so they run concurrently), do **not** drift into
unrelated work and wait to be notified. Supervise it:

1. **Record at spawn.** Append every agent's `MANIFEST.jsonl` line as you spawn it.
2. **Poll on a bounded interval.** Each check reads two cheap things: which `agents/*.json` outputs
   now exist, and the tail of each `.progress`. Mechanism, first available — `TaskList`/`TaskGet` for
   liveness, `TaskOutput` for a partial, `Monitor` for an until-condition, `TaskStop` to kill,
   `SendMessage` to continue one with its context intact; load them via `ToolSearch` if deferred. No
   task tools at all? Poll the progress files directly. The heartbeat files are the mechanism-
   independent signal — they work when nothing else is loaded.
3. **Classify every agent on every check:**
   - **done** — its output file exists and parses.
   - **working** — a new `.progress` line since your last check.
   - **stalled** — alive, but no new progress line across two consecutive checks. That is a loop
     with no progress. Kill it (`TaskStop`). Do not wait one more round, and do not nudge it with the
     same prompt — same input, same loop.
   - **dead** — the task ended and no output file exists.
4. **Handle stalled/dead once, then absorb it.** Re-spawn exactly once with a *narrower* scope, a
   smaller budget, and the specific sub-question (never a wider scope "so it can finish" — unbounded
   scope is what stalls agents). If the retry fails too, do that slice yourself in the main thread and
   record it as covered-by-orchestrator.
5. **Close the wave explicitly.** Every manifest entry must end at `done`,
   `covered-by-orchestrator`, or `uncovered` **named in the phase summary**. Only then update
   `LEDGER.md` and move to the dependent stage. A missing member must never dissolve into a survival
   ratio or a coverage claim.

**Wave-level stop:** if more than half a wave stalls, stop the wave and inspect your own prompt. The
cause is nearly always shared — an unbounded search, a path that doesn't exist, a target the agents
can't reach — and re-spawning six agents just repeats the same mistake six more times.

**Never:**

- Fire-and-forget a wave and infer success from the members that happened to answer.
- Re-prompt a stalled agent with its original prompt.
- Widen scope to help a stalled agent finish.
- Let an absent agent's silence pass as a clean lens.

## Macro-loop anti-stall

The same disease at pipeline scale: re-running Step 8's loop against an unchanged reality and calling
it iteration. A round only counts if it produced **new observed signal** — a different failure
signature, a new observed value, a new `file:line`. Record per round in the ledger: what changed in
the code, what the run observed, and how that differs from the previous round.

- **Two consecutive rounds with the same failure signature = a stall, not progress.** Stop patching.
  Get new signal through a channel you have not read yet (a log, a state dump you add, a smaller
  repro), or escalate architecturally per Step 8.
- **Three rounds with no new signal = stop and report honestly** what remains unproven, quoting the
  exact failure signature and what you tried.

This does not contradict "loop is convergence, not a counter." Convergence still terminates only on a
stable green-with-proof re-run. This bound applies to **non-convergence**, and its outcome is an
honest unproven report — never a claimed pass.
