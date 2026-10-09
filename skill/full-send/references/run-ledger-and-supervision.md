# Run ledger & agent supervision

Every fan-out phase loses two things quietly. State kept only in your context never reaches the
agents, and a compaction takes it first. A hung agent and a dead agent both emit no signal, so both
read as *clean*. Two invariants close them:

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

Created in **Step 0, before the first agent is spawned**. Locate it in this order: the
harness-provided scratchpad if one exists; else `/tmp/full-send-<slug>/`; else any writable dir
outside the repo. Add a unique suffix to that fallback path. **Never the repo tree, never
git-committed** — all of it is scratch.

```
<run>/
├── LEDGER.md                     # the canonical shared brief — every agent reads this FIRST
├── <NAME>-RCA.md                 # bugs only
├── <NAME>-PLAN.md
├── <NAME>-adversarial.json       # the synthesizer object, per round
├── pre/                          # the start state: the plan before gate A, and any undo copy
├── proof/                        # proof records, driver scripts, logs, screenshots
└── agents/
    ├── MANIFEST.jsonl            # one line per spawned agent (see below)
    ├── gateA-<pass>-input.md     # gate A: the frozen plan body one pass reviews — you write it
    ├── gateA-<pass>-<lens>-<a|b>.json  # gate A: one reviewer; a later spawn adds -r2, never overwrites
    ├── <phase>-<key>.json        # that agent's FULL output — the agent writes it
    ├── <phase>-<key>.md          # gate A checklist reviewers only: plan-ready text of their outputs
    └── <phase>-<key>.progress    # that agent's append-only heartbeat — the agent writes it
```

One `MANIFEST.jsonl` line per agent, appended **at spawn time**, not at return:

```json
{"id":"attack-regression","phase":"adversarial-R1","brief":"regression lens","status":"spawned",
 "out":"agents/adversarial-R1-regression.json"}
```

## LEDGER.md — the shared brief

```markdown
# <NAME> — RUN LEDGER
## Run          one-line goal; what is explicitly out of scope
## Constraints  the user's hard constraints, verbatim (design source, prod identifier, "no new screen")
## Style        the writing standard for every prose field — the rules copied out, never a pointer to them
## Phase        current phase; which gates are cleared
## Ground truth VERIFIED facts only, each with file:line
## Decisions    settled forks + why — INCLUDING refuted claims, so no later wave re-raises them
## Artifacts    paths: RCA, plan, adversarial JSON, proof records, driver command
## Workspace    the paths this run may write; the paths it must not; how the work reaches its destination; how to get back to the start state, or the words `no undo path`; the disposable store or test account this run may operate, or the word `none`
## Items        per-item state when the request names several items; the whole record with no tracker
## Capabilities one line per orchestration primitive: available, or blocked and by what
## Hold         the phase after which the user wants to look; the run stops there
## Cost ceiling agents per wave, waves, wall clock; the returns to Step 3 you have used; the gate A re-runs the user caused
## Stop set     for an open-ended request: the concrete changes it covers, frozen after Step 1
## Open         what is still unknown or uncovered; and any scope you narrowed, written down
```

Rules that make it trustworthy:

- **You are the only writer.** Agents read it and write their own files.
- **Update it at every phase boundary, and before spawning the next wave.**
- **Only verified facts reach `Ground truth`** — something you `Read` yourself, or an agent claim
  you spot-checked. An unverified claim stays in the agent's own file until it earns promotion.
- **It is a brief, not a transcript.** Keep it readable in ~30 seconds; link to files for depth.
- **`Style` is copied out in full, never referenced.** A subagent reads this file, not `SKILL.md`.
  Copy the whole of `SKILL.md` § Writing standard: the rule list **and** the paragraph naming what
  is exempt. The exemption protects quoted values and `file:line` cites.
- **Refuted claims are load-bearing.**
- **A row is at most 5 lines, and you create a row only when it applies.**
- **`Workspace` records the fence; it never shrinks the fix.** When ground truth shows the change
  needs a path the row does not list, add the path, say so in the ledger and in your next reply,
  then go on.

## The five lines every agent prompt carries

A subagent inherits none of your context, so these travel *inside* every spawned prompt — mapper,
attacker, verifier, synthesizer, driver:

1. **READ `<run>/LEDGER.md` FIRST.** It is this run's shared ground truth. Do not re-derive what it
   states. Do not violate a listed Constraint. Write every prose field you emit — claims, findings,
   notes, `statusNote` — to its `Style` entry.
2. **WRITE your output to `<run>/agents/<phase>-<key>.json` AS YOU GO.** Create it at the first
   milestone with `"status":"partial"`. Rewrite it after every new finding. Set
   `"status":"complete"` on the last write. A long final emit is how these agents die. Most agent
   deaths land on a transport error, and many arrive before the agent's first tool call. Read-only
   agents are read-only *on the project*; their two files under `agents/` are their only writes.
   A gate A checklist reviewer also writes the one `.md` file its prompt names.
   **Write no scratch file** there — the orchestrator collects that directory by glob.
3. **APPEND a heartbeat line to `<run>/agents/<phase>-<key>.progress` at every milestone** — started,
   read <file>, found N candidates, verifying, writing output. An agent with no new line across two
   checks is treated as hung and killed. Each line carries the finding itself, not only a step name.
4. **BUDGET: ~<N> tool calls / <M> files.** On hitting it, write what you have with
   `"status":"partial"`, open `statusNote` with `budget:`, name what went uncovered, and stop. That
   prefix is what tells the orchestrator you finished on your budget and did not die. Do not keep
   searching for a better answer past the budget.
5. **RETURN only** the output path, your `status`, and a ≤3-line summary. The file is the
   deliverable; your reply is a receipt.

## `status` is a required field, everywhere

Add to every schema this skill defines (attack, verify, synthesize, mapping, e2e report):

```json
"status":     { "type": "string", "enum": ["complete", "partial", "blocked"] },
"statusNote": { "type": "string", "description": "what went uncovered, or what blocked it" }
```

Without it, *empty output silently increases confidence*. `complete` means the work ran to its end.
**Absent output is never `complete`.** **Absent, malformed, or blocked output is `unverified`** —
never a pass, never a clean bill of health. Re-run it once, narrower. If it fails again, do that
slice yourself and say so.

## The supervision loop

After spawning a wave (all in ONE message, so they run concurrently), do **not** drift into
unrelated work and wait to be notified. Supervise it:

1. **Record at spawn.** Append every agent's `MANIFEST.jsonl` line as you spawn it.
2. **Poll on a bounded interval.** Each check reads two cheap things: which `agents/*.json` outputs
   now exist, and the tail of each `.progress`. Use your harness's task tools where it has them —
   liveness, a partial read, an until-condition, a kill, a message to continue one agent — and load
   them if they are deferred. With no task tools, poll the progress files directly. Print one
   liveness line per poll: the live agents, the elapsed time, and each one's last milestone.
3. **Classify every agent on every check:**
   - **done** — its output file exists, it parses, and its status is complete. A partial file
     belongs to a live agent, not a done one.
   - **working** — a new `.progress` line that carries a finding, and no class below fires.
   - **grinding** — 3 or more heartbeat lines that name a step and carry no finding. Kill it and
     narrow it. Do not wait.
   - **at-risk** — your harness reports it near its context limit, or its transcript is far larger
     than its siblings'. Kill it and narrow it before it dies on its final write. When the harness
     reports neither, this class does not fire, and `grinding` is the one you can read.
   - **budget-capped** — the task ended, its `status` is `partial`, and its `statusNote` opens with
     `budget:`. That is a finished agent that stopped where you told it to. Count the gaps the note
     names as uncovered, and close it.
   - **partial-dead** — the task ended and its `status` is not `complete`, with NO `budget:` prefix.
     It died mid-work. Treat it exactly as `dead`: the slice is `unverified`, never a lens that ran.
     Read its findings as leads only; they never enter a survival ratio. When you cannot tell the
     two apart, it is `partial-dead` — an unclaimed budget stop is the cheaper mistake.
   - **stalled** — alive, but no new progress line across two consecutive checks. Kill it
     (`TaskStop`). Do not wait one more round, and do not nudge it with the same prompt.
   - **dead** — the task ended and no output file exists, and it wrote at least one `.progress`
     line, so it started work and then died.
   - **born-dead** — the task ended, you did not kill it, and it wrote NO output file AND NO
     `.progress` line. It died in transport, before its first milestone.
4. **Handle stalled/dead once, then absorb it.** A **born-dead** agent you retry once, unchanged,
   because it died in transport and not in its prompt. The
   retry budget belongs to the slice, not to the agent — one retry total, and a re-spawned agent
   gets none of its own. If more than half a wave is born-dead, stop the wave. The cause is transport,
   not your prompt. Pause once, then re-spawn the born-dead members. That pause is a back-off
   between spawns, never supervision by waiting. Otherwise re-spawn exactly once with a *narrower* scope, a smaller budget, and the
   specific sub-question. Never widen the scope "so it can finish". If the retry fails, do that
   slice yourself and record it as covered-by-orchestrator.
5. **Close the wave explicitly.** Every manifest entry must end at `done`,
   `covered-by-orchestrator`, or `uncovered` **named in the phase summary**. Only then update
   `LEDGER.md` and move to the dependent stage. A missing member must never dissolve into a survival
   ratio or a coverage claim.

**Wave-level stop:** if more than half a wave stalls, stop the wave and inspect your own prompt. The
cause is nearly always shared — an unbounded search, a bad path, an unreachable target.

**Abort:** On a user interrupt or a stop instruction, act before you reply. Stop every live agent and
confirm each stop. Kill your own background jobs. Write an `Aborted` ledger entry that names the
phase, the artifacts on disk, and every outward action you already took.

**Never:**

- Fire-and-forget a wave and infer success from the members that happened to answer.
- Re-prompt a stalled agent with its original prompt. ONE exception: a **born-dead** agent returned
  nothing AND wrote no output file. Retry it once, unchanged. Re-spawn only the born-dead members;
  a whole-wave restart kills healthy siblings and
  truncates the partial files they wrote.
- Widen scope to help a stalled agent finish.
- Let an absent agent's silence pass as a clean lens.

## Macro-loop anti-stall

A round counts only if it produced **new observed signal** — a different failure signature, a new
observed value, a new `file:line`. Record per round what changed and what the run observed.

- **Two consecutive rounds with the same failure signature = a stall, not progress.** Stop patching.
  Get new signal through a channel you have not read yet (a log, a state dump you add, a smaller
  repro), or escalate architecturally per Step 8.
- **Three rounds with no new signal = stop and report honestly** what remains unproven, quoting the
  exact failure signature and what you tried.

This bound applies to **non-convergence** only. Its outcome is an honest unproven report, never a
claimed pass.
