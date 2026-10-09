---
name: full-send
license: MIT
metadata:
  version: 1.5.0
description: >-
  Run the full delivery pipeline end-to-end and autonomously, in order: ground-truth investigation
  (formal RCA from real data for bugs), a scratchpad plan, a built-in multi-lens PLAN REVIEW (scope
  and engineering always; design plus Mobbin research for UI work; DX for developer tools),
  ADVERSARIAL REVIEW
  (bespoke attack, then verify-default-REFUTED, then synthesize), direct implementation, then
  AGENTIC END-TO-END validation on the REAL app with real user clicks and deterministic verdicts,
  looping until green-with-proof and no regressions. FIRES when the user wants that arc, however
  few phases they name and however phrased; you supply the rest. DOES NOT FIRE for one
  review/test/QA command's output in isolation (just run the tests), nor on quality adjectives
  alone with no phase named ("make it solid"). Examples: "plan, adversarial review, build, e2e,
  loop until verified"; "RCA this bug then fix it with proof"; "loop until verified with no
  regressions"; "validate with agentic tests like a real user".
---

# full-send

Codifies a battle-tested delivery loop: **ground truth → plan → plan-review → adversarial review →
implement → agentic end-to-end validation → loop until green-with-proof.** Your job is to run it
faithfully and autonomously — and to *improve* on it (e.g. add a self-verification pass), never to
water it down.

The point of the whole thing: **no claim of "done" is trusted unless it is proven against real,
observed state through a deterministic verdict the LLM did not author.**

## The autonomy contract (read first)

- **Proceed on sensible defaults, and keep the turn.** Do not stop to ask permission at each
  phase. Four stops break the pipeline: the run asks a question; the run announces the
  next action and stops; the run writes a phase report; the run yields while its own agents still
  run. Only seven reasons make a yield legal: the pipeline reached its terminal proof; an
  `AskUserQuestion` scope fork is open; a named external actor must act; the ledger names a Hold
  that this phase reaches; a phase defines a bounded non-convergence stop and the run reached it —
  the Step 8 three-round stop-and-report is one; the ledger cost ceiling cannot cover a
  lens whose fire condition this change meets, or a subsystem the change touches; or the next action reaches outside this
  run's workspace where another person or system observes it. That last reason is a named class,
  not a
  judgement about undo: a push or merge to a shared remote, a release or deploy, a message or mail
  sent, a schema or data change on a shared store, a delete, and any spend on a metered resource.
  Ask once, name the action, and go on when the user answers. For every other case, do the next
  action now.
- **Escalate rarely and only for genuine forks.** Use `AskUserQuestion` ONLY when a *scope or
  architecture* decision is genuinely the user's to make and you cannot resolve it from the request,
  the code, or the sessions — always with a recommended default. Judge by the *decision*, never by
  the fact that some tool or skill popped a question. Never relay another skill's housekeeping
  prompts (onboarding, telemetry opt-in, its own menus) to the user — self-answer those.
- **Never batch clarifying questions.** One sharp question beats five.
- **Respect blunt overrides.** "Spike then full-send, don't ask me" means exactly that.
- **Observe before you answer.** A question about observable behaviour needs an observation, not a
  code read. Operate the surface before you answer, and cite what you saw — but only a surface this
  run owns: a local build, or the disposable store or test account the `Workspace` row names. Never
  operate a production system, another person's data, or anything that emits outward, to answer a
  question. When no owned surface reaches it, label the answer as read from source, and name the one
  action that settles it.
- **Loop is convergence, not a counter.** Terminate on stable green-with-proof re-run against
  current state — never on a fixed number of tries, never on the first green.
- **Scale to the task.** The full ceremony is for a stated feature/bug. A *correction to what you
  just built* takes the fast path (edit → tests → screenshot), not a fresh pipeline. See Step 9.
- **An explicit invocation outranks your own scale judgement.** When the user names this pipeline,
  run it. Do not rule the task too small.
- **A subagent inherits none of this.** Spawned attackers, verifiers and mappers start with empty
  context: every rule that must govern one — default-REFUTED, cite `file:line`, don't invent handled
  problems, the sanctioned-seam rule — has to be restated **inside its prompt**. The run ledger
  below carries the *shared* half: one file every agent reads first.
- **Where a project `CLAUDE.md` conflicts,** its conventions win on file placement and house style.
  Two things it does not override: this skill's proof obligations, and the writing standard below. A
  project that states its own prose standard is a conflict you **name in the ledger's `Style` entry**
  and keep writing through — never a reason to drop the standard silently.

## Writing standard (every artifact this pipeline emits)

Every human-readable string this pipeline writes is in **ASD-STE100 Simplified Technical English** —
drafted that way, not cleaned up at the end.

**Scope — all of it.** The ledger, the RCA, the plan, agent findings, the proof record, your replies,
**code comments and docstrings**, log and error strings, commit messages, PR title and body,
task-board entries, and the text the product shows a person. Name the audience: a reviewer's work
item and the user's report differ.

**The rules that carry the weight:**

- **Short sentences.** ≤20 words for an instruction, ≤25 for a description. One instruction per sentence.
- **Active voice, actor named.** "The verifier refutes the finding" — not "the finding is refuted."
- **One word, one meaning.** Never swap in a synonym for variety.
- **Common words.** No idiom, no metaphor. Keep the articles — no telegraphic style.
- **Noun clusters of 3 words or fewer.** A new paragraph per topic.

**Exact terms are exempt, and they outrank the style.** Identifiers, API and file names, error
strings, CLI invocations, `file:line` cites, and every observed value you quote stay verbatim.
Simplify the sentence around the term; never simplify the term. Never paraphrase a raw observed value
to make a sentence read easier — that breaks Step 7's proof rules, and the proof rules win.

The standard travels through the ledger, not through your context. Step 0 seeds it; every spawned
agent reads it first. With no writable dir and so no ledger file (see Graceful degradation),
restate the base rule above inside each spawned prompt.

## Orchestration primitive (mechanism-agnostic)

Several phases fan work out to parallel agents. The **structure** is what matters, not the tool:

- **Universal primary — single-turn parallel subagents.** Spawn N `Agent`/`Task` subagents *in one
  turn*, each with a strict JSON output schema. Collect a stage's results before starting the stage
  that depends on them. If `Task`/subagent or MCP tools aren't loaded yet, load them via
  `ToolSearch` (`select:<name>`).
- **Optional accelerator — the `Workflow` tool.** If it is available, prefer it for the
  fan-out/adversarial phases. It adds speed, not correctness: the three-stage adversarial shape
  and real parallelism must hold either way. A `Workflow` run is still a wave you supervise
  (`/workflows`, then `TaskStop` + resume-from-runId on a stalled stage).

Probe once, inline: if `Workflow` is in your tools, use it; else use parallel subagents. Do not
build a decision tree around this.

**Two invariants hold on either mechanism.** Full contract in
`references/run-ledger-and-supervision.md` — read it before your first fan-out.

- **Shared state lives in files, not in your context.** Your context is a cache of those files,
  never the system of record.
- **Every spawned agent is supervised until it is accounted for.**
- **After a compaction,** re-read `LEDGER.md`, the phase contract and this skill's section for the
  phase before your next tool call. Name the phase you resumed. Log any scope cut in `Open`, out
  loud.

---

## Step 0 — Contract & board

Restate the user's pipeline as an explicit phase contract in your first substantive reply, and
capture every **hard constraint** they stated (a design source of truth, a production identifier
that must not change, a hosting preference, "no separate screen", etc.). Create a task board
(`TaskCreate`) mirroring the phases, one `in_progress` at a time.

Then create the **run directory and `LEDGER.md` — now, before any agent is spawned** (layout and
contract in `references/run-ledger-and-supervision.md`): the harness scratchpad if one exists, else
`/tmp/full-send-<slug>/`. Seed the ledger with the phase contract, those hard constraints verbatim,
and the `Style` line that carries the writing standard to every agent. Every later artifact lands
here, and every agent you spawn from here on reads this file first.

**Scale first.** A correction to what you just built takes the fast path in Step 9 — edit, prove,
done — and skips the rest of this step. Everything below is for a stated feature, a bug, or a new
directive. Fill these rows now, before Step 1, each one that applies. **`Workspace`** — the paths this run may write, the paths it must
not, how the work reaches its destination, and how to get back to the start state, or the words
`no undo path`. When nothing else covers the destination, make an undo path: copy the start state
  into `<run>/pre/` and record that. Name the disposable store or test account this run may operate,
  or the word `none`. **`Hold`** — the phase after which you will stop for the user to look, or the words
`run to the terminal proof`. Any user words that defer the next phase set a Hold: "stop after X",
"let me look", "wait", "don't do Y yet", "check with me before Z". Write them verbatim.
**`Capabilities`**, and the **`Cost ceiling`** row; add a **`Stop set`** row for an open-ended
request — see `references/limits-and-recovery.md`, which also owns rollback, the return path from a
wrong plan, push-back, and the handoff pack.

## Step 1 — Ground truth first (invariant)

Never author a plan from memory. Anchor it in verified reality.

- **Features:** fan out **2–6 read-only mapping subagents**, each pinned to one subsystem with a
  strict JSON schema that **requires `file:line` citations**, running alongside your own direct
  `Read`s. Each writes its full map to `<run>/agents/` and returns a receipt.
  **Supervise the wave to close**. Template + schema in `references/investigation-and-plan.md`.
- **Analysis / document:** the subject is data or prose, not a subsystem. Pull the real inputs and
  freeze a copy in the run directory. Cite a source locator for every claim — `file:line` where the
  source has lines, else the sheet, the row, or the page.
- **Infra / deploy:** probe the live machine, not the config that claims to describe it — real
  running processes, actual deployed state, the real build output.
- **Bugs:** do a **formal RCA from real data, with a proof** — pull the ACTUAL failing artifact
  first (copy the real DB and query persisted rows; read real logs / traces / build output; grep the
  SDK headers) and verify *every step of the trace* BEFORE reading source.
- **Re-verify the 2 riskiest discovery claims yourself** against real source.
- **Promote only what you verified into the ledger's `Ground truth`.** Everything else stays in the
  agent's own file until it earns promotion.

## Step 2 — Design lane (UI/design work only)

Detect whether the change has a **UI/design surface**. If it does, this lane is **not optional**:

- **Inspiration:** use the **Mobbin MCP** (`mcp__mobbin__search_screens` / `search_flows`) with 3–8
  narrow queries for the *specific* pattern. Synthesize a named reference set and one concrete,
  ownable direction.
- **Write it down.** At Step 3, put the reference set, the direction, and the path of each image
  the user gave into the plan's `## Design` section. Gate A's design reviewer starts with empty
  context: it reads that section, and nothing you only hold in yours.
- **Cross-check** against the project's design source of truth (a `design.md`, a token file, brand
  refs). If code and a stale doc disagree, code wins.
- **MCP may be deferred or absent.** Before first use: if the tool is in the deferred list,
  `ToolSearch`-load it (`select:mcp__mobbin__search_screens,...`); if no `mcp__mobbin__*` tool
  exists after that, take the fallback (the user's provided screenshots / the project design source).
  A deferred/absent-tool error IS the not-connected branch — never a hard failure.

Skip this lane entirely for pure backend/infra work.

## Step 3 — Plan to the scratchpad (invariant)

Write a single `<NAME>-PLAN.md` into the **run directory created in Step 0** — never the repo tree,
never git-committed. Write to the skeleton in `references/investigation-and-plan.md`, and point the
ledger's `Artifacts` section at it.

The plan carries: goal/scope; numbered **fix sections F1/F2…**; a **test-coverage tree** tagging each
touched file with the tier that will prove it and the deterministic assertion that tier makes —
`[UNIT]` / `[E2E-web]` / `[E2E-native]` / `[DEVICE]` / `[ANALYSIS]` / `[BATCH]` / `[DOCUMENT]` /
`[MIGRATION]` / `[JUDGE]` (content-correctness; a separate
proof from render/nav) — substituting the tiers this project actually has; migration/back-compat;
and risks. Use `AskUserQuestion`
here only to lock a genuine scope/architecture fork.

## Step 4 — Review gate A (the plan review)

**The procedure, the prompts, the schema and the decision rules are in `references/plan-review.md`
— read it before you run this gate.** The gate is native: it calls no other skill.

- **Four lenses, one file each.** `scope` and `eng` always run. `design` runs when the plan has a UI
  surface. `dx` runs when the plan has a developer-facing surface. Each reviewer subagent reads its
  own lens file, `references/plan-lens-<lens>.md`. You do not paste the checklist.
- **Two independent reviewers for each lens:** a checklist reviewer and a challenger. Both read one
  frozen snapshot of the plan. Neither reads the other's output.
- **Ordered passes.** Scope, then design and DX, then eng. The eng lens runs last, so it reviews the
  plan that gate B and Step 6 receive. A plan whose fixes touch one file runs every lens in one
  wave.
- **You decide every finding** by the six principles. Class it `mechanical`, `taste` or
  `direction`, and log it in the plan at once. A decision replaces the user's answer. It never
  replaces the analysis.
- **You never decide a `direction` change.** That is a change to what the user stated, which both
  reviewers want. The user's direction stands. Ask it as the `AskUserQuestion` scope fork, after
  the last pass.
- **Each pass closes before the next one opens:** fold, read the whole plan body again, write the
  pass line.

Run it **autonomously.** **Gate A must never halt the pipeline:** it does not stop at a pass
boundary. It can cause two yields only: an open `direction` question, and a ledger `Hold` row that
names the gate. Only when the user explicitly asks to drive the plan review do you ask where the
autonomous run decides. This is "Respect blunt overrides" applied to gate A.

Fold findings back into the plan until the gate closes. Before you fold a `blocker`, a `major` or
a **scope-changing** finding, spot-verify it against real source or plan text. A `minor` finding at
confidence 7 or more may fold directly.

## Step 5 — Review gate B: the adversarial review (the signature move)

**Full template, lens catalog, and schemas in `references/adversarial-review.md` — read it before
running this the first time.** The shape is invariant; the mechanism is not:

- **ATTACK** — N independent agents (N≈3–6, scaled to surface area), each pinned to exactly ONE named
  failure lens, each **required to cite `file:line`** and told *"do not invent problems the
  code/plan already handles."*
- **VERIFY** — a **fresh** agent per raw finding whose **default verdict is REFUTED / false-positive**
  unless it can construct a concrete, reachable failing sequence from the real code. It also
  **rejects a finding's proposed fix if that fix would regress prior shipped behavior.** Keep this
  burden-of-proof asymmetry.
- **SYNTHESIZE** — dedupe/rank survivors, emit an exact plan-edit per survivor, and issue
  **GO / GO-WITH-PATCHES / NO-GO.**

Run it via `Workflow` if available, else as parallel subagents in the same three stages. Every
stage obeys the fan-out invariants: attackers, verifiers **and the synthesizer** read
`<run>/LEDGER.md` first, write to `<run>/agents/`, and are **supervised to close**. Do useful prep
while the wave runs, but keep polling it. The closing steps — spot-verify the top ~2 survivors, fold
them into the plan, mark it `SCOPE+ENG[+DESIGN][+DX]+ADVERSARIAL CLEARED` before any implementation code — are
in `references/adversarial-review.md`. Calibration: ~40–65% of raw findings typically survive
verification; a round where nearly all or nearly none survive is a smell; inspect the verify stage.

## Step 6 — Implement (direct, single-threaded)

Now, and only now, write code — **yourself, in the main thread.** Follow the plan's `Build order`,
and write each entry of its `Test requirements`: a `blocker` entry is not optional. Subagent fan-out is for
investigation and review; it is *not* for writing the implementation. When the deliverable is an
analysis or a document, "implement" means author it and its checks: the frozen inputs, the second
independent derivation, and the claim list its tier asserts against. Each rule below that names
code applies where code exists — the checker is code. Blast radius, the discard ban,
verify-before-spend and the writing standard hold always.

- **Extract pure / dependency-injected helpers first and unit-test them first**.
- **Typecheck / compile after edits** (filter to touched files to dodge pre-existing errors in
  untouched modules if needed).
- **Minimize blast radius.** Touch the smallest set of files. Leave the project's stated invariants
  untouched — discover them from `CLAUDE.md`/README/tests.
- **Verify-before-spend.** Prove risky new code against a real *disposable* dependency (e.g. a
  throwaway Docker Postgres) BEFORE you consume a billable or irreversible
  resource, a shared quota, or a rate limit. Send nothing outside this run's workspace
  before the proof is complete. An external publish cannot be undone.
- Every confirmed defect gets **both a source fix AND a new regression test** pinned to the exact
  repro sequence.
- **Never run a command that discards uncommitted work.** This holds for the whole run, not only
  for red/green. The ban covers work a person wrote and this run did not create. It does not cover
  build output, dependency directories, or your own scratch — delete those freely to rebuild. Get a
  baseline by an in-place edit you undo, or by a copy outside the tree.
- **Comments, docstrings, log lines and error strings follow the writing standard** above. Match the
  surrounding file's comment density — the standard governs how a comment reads, never how many you
  add. Identifiers and API names keep the spelling the code already uses.

## Step 7 — Agentic end-to-end validation (real artifact, deterministic verdict)

Prove the change works through simulated real use. **The surface table, the five preconditions, the
tier table, driver fallbacks, red/green and the proof record are all in
`references/agentic-e2e.md` — read it before you drive anything.** Two rules are sacred on every
surface:

1. **The verdict is a deterministic assertion against the REAL running artifact — the LLM never
   decides pass/fail.** A CLI, a library, a service and a UI each get their own drive-and-assert row
   in that reference. **No obvious surface?** Reinterpret it — drive a real browser to read a real
   build log, or run a physical-device prebuild → install → observe loop for a native asset. There
   is almost always a real artifact to exercise.
2. **Anti-hallucination:** an agent driving a UI may only interact with a selector actually present
   on the current screen; a deterministic `testID`/DOM/state assertion pins every value that matters.

**UI specialization is mandatory when a UI exists.** Discover the project's harness first. Drive the
REAL screens like a user, never a mock shell. Stand up at least one driver from the reference ladder
before you claim UI validation; if none can be stood up, **flag the gap honestly** — never silently
downgrade "validated like a user" to "the unit tests pass." Pick the tier(s) by what the change
actually touches, keep render/nav and content-correctness as **separate** proofs, prove every NEW
test red/green through the build path, and run the targeted suite before the full regression gate.

## Step 8 — Loop to green-with-proof

- **A fact that invalidates the plan takes the return path** in
  `references/limits-and-recovery.md`, not another patch. A landed change that then failed takes the
  rollback there.
- **Debug from real signal; never guess twice.** See the actual failure first — real build log,
  hierarchy dump + screenshot, deterministic repro, live command output — then fix, then re-run.
  Diagnose failures one at a time.
- **Loop the adversarial review** R1→Rn — re-running the same three-stage shape against the
  *implementation diff*, not just the plan — each round stricter about rejecting
  unreachable/theoretical findings, until a round finds nothing on the crash-causing invariants.
- **No cheating in tests (invariant).** A new or edited test must assert against a **sanctioned
  mechanism** — a recorded/replayed tape or a live call, never a stub that returns success — and it
  must pin **real observable state**. Do not weaken an assertion to force green. Seam legitimacy in
  full: `references/agentic-e2e.md`. The adversarial test-integrity lens enforces it.
- **Anti-stall guard: a round only counts if it produced NEW observed signal** — a different failure
  signature, a new observed value, a new `file:line`. Log per round in the ledger what changed in the
  code and what the run observed. The two-round stall rule, the three-round stop-and-report rule and
  their escalation path are in `references/run-ledger-and-supervision.md`. That bound holds for
  non-convergence only; it never converts into a claimed pass.
- **Staleness guard.** Re-run the FULL regression gate after ANY concurrent change, so "no
  regressions" holds against current state. A performance regression is a regression: gate it where
  the project measures performance, else name it uncovered. Real-device feedback from the user
  re-enters this same fix → typecheck → targeted-E2E → full-gate cycle.
- **Allow architectural escalation.** If incremental patches keep failing, prefer a structural change
  that makes the guarantee true *by construction* over another patch.

On infra/deploy work the macro loop runs once; convergence lives in the tight
fail → diagnose-from-real-output → fix → re-trigger micro-loop.

Terminate only on a stable green-with-proof re-run against current state — and emit the **proof
record** (`references/agentic-e2e.md`), never a prose "all green."

## Step 9 — Close out

- Persist a durable fact to a **memory file**, not a committed doc; update `CLAUDE.md` only for
  genuinely shared how-to. On a repeated operational failure you repair the procedure that failed,
  and name the file and line.
- **Write the handoff pack** (`references/limits-and-recovery.md`) and name its path in the closing
  report. It carries gate A's open items: each `taste` decision, each `NOT in scope` item, and each
  `direction` change you could not ask.
- **Scoped `git add`** of only this change's files (never `git add -A`) when concurrent WIP is
  present; be transparent in the commit body about unavoidable shared-file bits. **Commit/push only
  when the user asks.**
- Restart the correct always-on process if you touched the running system; **kill your own orphaned
  background jobs / locks.**
- **Refresh the tracked work item** from the final diff, if one exists. Never remove a field a
  person wrote. A board another person reads is an outward action under the autonomy contract.
- Spawn a background **task chip** only for issues on surfaces this run did not touch. A defect on
  a surface this run touched is work, not a report line: fix it. Sweep the siblings the diff names,
  once.
- **Scale-down boundary.** *Same feature* = the surface built in this session's full-ceremony pass;
  a correction to it takes the fast path: touch only what the user named, then observe the corrected
  output once by the cheapest real path — run the command, call the function, open the page, or read
  the text. A NEW stated directive re-enters the pipeline at proportional scale. When unsure: a
  trivial in-file tweak is a correction (fast
  path); anything introducing a new module/seam/migration is a new directive (full pipeline). Do not
  reframe this dial around diff size or file count.
- **The closing report answers the question the user asked.** Every number comes from a command;
  you re-derive it from disk, not from memory.

---

## Before you declare green

This is the short set whose violation produces a green that is **wrong but looks right**. Re-read
it before done.

1. The adversarial review kept its three-stage shape — single-lens attackers citing `file:line`, a
   **fresh** verifier per finding defaulting to REFUTED, then synthesis.
2. **Every fan-out wave closed with every member accounted for.** A dead agent returns empty, and
   empty reads as clean unless something counted it.
3. You re-verified the highest-stakes review claims against real source before you acted on them.
4. Every verdict came from REAL observed state. You debugged from signal and never guessed twice.
5. The end-to-end verdict is a deterministic assertion **you did not author**, run against a
   rebuilt artifact; on a UI the agent only touched what was on screen.
6. No test cheats — sanctioned seams, real-state assertions, red/green through the build path.
7. Content-correctness and render/navigation are still **separate** proofs.
8. Whatever your tier provably cannot exercise is named as uncovered, not folded into the pass.
9. **Re-check every Constraint the ledger recorded at Step 0.** You used the method the user named.
10. Each `blocker` test requirement of the plan has a test. Each row of the plan's `E2E drive list`
    has a proof line, or is named as uncovered.

## Graceful degradation (this is a global skill — assume nothing)

A named tool, MCP, or skill that is absent, deferred, or halting on interactivity is the **degraded
branch, never a hard failure**: take the fallback stated at that step and keep going. These are not
obvious from any single step:

- **Never run the attackers serially in the foreground.**
- **No task board?** `TaskCreate` may be absent or deferred; track the phases in your replies instead.
- **No task-control tools?** Supervision still runs on the `.progress` heartbeat files alone, which
  need nothing but `Read`/`Bash` (`references/run-ledger-and-supervision.md`). Never degrade
  supervision into waiting.
- **No writable dir outside the repo?** Say so, keep the ledger inline in your replies, and **shrink
  the waves** (2–3 agents, tighter budgets).
- **Everything project-specific is a discovery step or a labeled example**, never a hardcoded command.

## References

Read each file at the step that names it.

- `run-ledger-and-supervision.md` — run directory, `LEDGER.md`, the five lines, `status`,
  supervision, the anti-stall bound.
- `adversarial-review.md` — the Attack→Verify→Synthesize template, the lens catalog, the schemas.
- `agentic-e2e.md` — preconditions, harness discovery, tiers, drivers, red/green, the proof record.
- `investigation-and-plan.md` — the fan-out template and the plan skeleton.
- `plan-review.md` — gate A: the lenses, the passes, the reviewer prompts, the decision rules.
- `plan-lens-scope.md`, `plan-lens-eng.md`, `plan-lens-design.md`, `plan-lens-dx.md` — one gate A
  lens each. A reviewer subagent reads its own file. You read `plan-review.md`, and the tail of a
  lens file when you close its pass.
- `limits-and-recovery.md` — rollback, the return path, the stop set, push-back, the cost ceiling,
  the handoff pack.