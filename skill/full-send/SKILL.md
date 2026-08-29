---
name: full-send
license: PolyForm-Noncommercial-1.0.0
metadata:
  version: 1.3.0
description: >-
  Run the full delivery pipeline end-to-end and autonomously, in order: ground-truth investigation
  (formal RCA from real data for bugs), a scratchpad plan, plan review via /autoplan or
  /plan-eng-review (plus /plan-design-review and Mobbin research for UI work), ADVERSARIAL REVIEW
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
implement → agentic end-to-end validation → loop until green-with-proof.** The user usually states
this pipeline (or part of it) verbatim in the opening message. Your job is to run it faithfully and
autonomously — and to *improve* on it (e.g. add a self-verification pass), never to water it down.

The point of the whole thing: **no claim of "done" is trusted unless it is proven against real,
observed state through a deterministic verdict the LLM did not author.** Every phase exists to close
a specific way a change can look done but not be done.

## The autonomy contract (read first)

- **Proceed on sensible defaults.** Do not stop to ask permission at each phase. Run the whole loop.
- **Escalate rarely and only for genuine forks.** Use `AskUserQuestion` ONLY when a *scope or
  architecture* decision is genuinely the user's to make and you cannot resolve it from the request,
  the code, or the sessions — always with a recommended default. Judge by the *decision*, never by
  the fact that some sub-skill popped a question. Never relay a sub-skill's housekeeping prompts
  (onboarding, telemetry opt-in, its own scope-challenge menus, mockup ceremony) to the user —
  self-answer those.
- **Never batch clarifying questions.** One sharp question beats five. The user actively dislikes
  question batches and will override you.
- **Respect blunt overrides.** "Spike then full-send, don't ask me" means exactly that.
- **Loop is convergence, not a counter.** Terminate on stable green-with-proof re-run against
  current state — never on a fixed number of tries, never on the first green.
- **Scale to the task.** The full ceremony is for a stated feature/bug. A *correction to what you
  just built* takes the fast path (edit → tests → screenshot), not a fresh pipeline. See Step 9.
- **A subagent inherits none of this.** Spawned attackers, verifiers and mappers start with empty
  context: every rule that must govern one — default-REFUTED, cite `file:line`, don't invent handled
  problems, the sanctioned-seam rule — has to be restated **inside its prompt**. That is
  transmission across a context boundary, not duplication. The run ledger below is how the *shared*
  half of that transmission scales past a couple of rules: one file every agent reads first.
- **Where a project `CLAUDE.md` conflicts,** its conventions win on file placement and house style.
  Two things it does not override: this skill's proof obligations, and the writing standard below. A
  project that states its own prose standard is a conflict you **name in the ledger's `Style` entry**
  and keep writing through — never a reason to drop the standard silently.

## Writing standard (every artifact this pipeline emits)

Every human-readable string this pipeline writes is in **ASD-STE100 Simplified Technical English** —
drafted that way, not cleaned up at the end. The artifacts are how a long autonomous run stays
legible: to the user watching it converge, and to the next agent that reads the ledger cold.

**Scope — all of it.** The ledger, the RCA, the plan, agent findings, the proof record, your replies,
**code comments and docstrings**, log and error strings, commit messages, PR title and body,
task-board entries.

**The rules that carry the weight:**

- **Short sentences.** ≤20 words for an instruction, ≤25 for a description. One instruction per sentence.
- **Active voice, actor named.** "The verifier refutes the finding" — not "the finding is refuted."
- **One word, one meaning.** Never swap in a synonym for variety. A second word for one concept reads
  as a second concept.
- **Common words.** No idiom, no metaphor. Keep the articles — no telegraphic style.
- **Noun clusters of 3 words or fewer.** A new paragraph per topic.

**Exact terms are exempt, and they outrank the style.** Identifiers, API and file names, error
strings, CLI invocations, `file:line` cites, and every observed value you quote stay verbatim.
Simplify the sentence around the term; never simplify the term. Never paraphrase a raw observed value
to make a sentence read easier — that breaks Step 7's proof rules, and the proof rules win.

This is a *global* constraint on a run, so it travels the way every other global constraint travels:
through the ledger, not through your context. Step 0 seeds it; every spawned agent reads it first.
When there is no writable dir and so no ledger file (see Graceful degradation), it falls back to the
base rule above — restate it inside each spawned prompt. The channel degrades; the standard does not.

## Orchestration primitive (mechanism-agnostic)

Several phases fan work out to parallel agents. The **structure** is what matters, not the tool:

- **Universal primary — single-turn parallel subagents.** Spawn N `Agent`/`Task` subagents *in one
  turn* (they run concurrently), each with a strict JSON output schema. Collect a stage's results
  before starting the stage that depends on them. This works in every Claude Code session. If
  `Task`/subagent or MCP tools aren't loaded yet, load them via `ToolSearch` (`select:<name>`).
- **Optional accelerator — the `Workflow` tool.** If it's available (some sessions enable it),
  prefer it for the fan-out/adversarial phases: it runs in the background with auto-notify and
  supports loop-until-done scripts. Background execution + auto-notify is a *performance*
  optimization, not a correctness requirement — the three-stage adversarial shape and real
  parallelism must hold either way. Backgrounded is not unwatched: a `Workflow` run is still a wave
  you supervise (`/workflows`, then `TaskStop` + resume-from-runId on a stalled stage).

Probe once, inline, without ceremony: if `Workflow` is in your tools, use it; else use parallel
subagents. Do not build a decision tree around this.

**Two invariants hold on either mechanism.** Full contract in
`references/run-ledger-and-supervision.md` — read it before your first fan-out.

- **Shared state lives in files, not in your context.** Before the first agent is spawned, create the
  run directory and its `LEDGER.md`. Every agent **reads the ledger first** and **writes its full
  output to its own file** under `<run>/agents/`, returning only a path + `status` + a ≤3-line
  receipt. Your context is a cache of those files, never the system of record — it is exactly what a
  compaction takes, and what a subagent never had. This is what keeps N fanned-out agents working
  from one picture of reality instead of N drifting ones.
- **Every spawned agent is supervised until it is accounted for.** Record it in the manifest at
  spawn; poll its heartbeat; **kill a no-progress agent rather than waiting on it**; re-spawn once
  narrower, then cover the slice yourself. A wave is closed only when every member is `done`,
  `covered-by-orchestrator`, or named `uncovered`. Dead and stalled agents are the quietest failure
  in this pipeline: they return empty, and empty reads as clean.

---

## Step 0 — Contract & board

Restate the user's pipeline as an explicit phase contract in your first substantive reply (so they
can correct it), and capture every **hard constraint** they stated (a design source of truth, a
production identifier that must not change, a hosting preference, "no separate screen", etc.).
Create a task board (`TaskCreate`) mirroring the phases, one `in_progress` at a time. The board is
how you and the user watch the loop converge.

Then create the **run directory and `LEDGER.md` — now, before any agent is spawned** (layout and
contract in `references/run-ledger-and-supervision.md`): the harness scratchpad if one exists, else
`/tmp/full-send-<slug>/`. Seed the ledger with the phase contract, those hard constraints verbatim,
and the `Style` line that carries the writing standard to every agent. Every later artifact — RCA,
plan, agent outputs, adversarial JSON, proof records — lands here, and every agent you spawn from
here on reads this file first.

## Step 1 — Ground truth first (invariant)

Never author a plan from memory. Anchor it in verified reality.

- **Features:** fan out **2–6 read-only mapping subagents** (parallel, per the orchestration
  primitive above), each pinned to one subsystem with a strict JSON schema that **requires
  `file:line` citations**, running alongside your own direct `Read`s. Each writes its full map to
  `<run>/agents/` and returns a receipt; **supervise the wave to close** — an unmapped subsystem is
  a hole in the plan's foundation, not a smaller plan. Template + schema in
  `references/investigation-and-plan.md`.
- **Infra / deploy:** probe the live machine, not the config that claims to describe it — real
  running processes, actual deployed state, the real build output.
- **Bugs:** do a **formal RCA from real data, with a proof** — pull the ACTUAL failing artifact
  first (copy the real DB and query persisted rows; read real logs / traces / build output; grep the
  SDK headers) and verify *every step of the trace* BEFORE reading source. Reason from signal, not
  from source alone.
- **Re-verify the 2 riskiest discovery claims yourself** against real source. Reader agents can
  miscite; a plan built on a miscitation is worse than no plan.
- **Promote only what you verified into the ledger's `Ground truth`.** Everything else stays in the
  agent's own file until it earns promotion — the ledger is the one thing later waves take on faith.

## Step 2 — Design lane (UI/design work only)

Detect whether the change has a **UI/design surface**. If it does, this lane is **not optional**:

- **Inspiration:** use the **Mobbin MCP** (`mcp__mobbin__search_screens` / `search_flows`) with 3–8
  narrow queries for the *specific* pattern (e.g. icon→card morph, an analyzing overlay, empty-state
  as an invitation). Synthesize a named reference set and one concrete, ownable direction.
- **Cross-check** against the project's design source of truth (a `design.md`, a token file, brand
  refs). If code and a stale doc disagree, code wins.
- **MCP may be deferred or absent.** Before first use: if the tool is in the deferred list,
  `ToolSearch`-load it (`select:mcp__mobbin__search_screens,...`); if no `mcp__mobbin__*` tool
  exists after that, take the fallback (the user's provided screenshots / the project design source).
  A deferred/absent-tool error IS the not-connected branch — never a hard failure.

Skip this lane entirely for pure backend/infra work.

## Step 3 — Plan to the scratchpad (invariant)

Write a single `<NAME>-PLAN.md` into the **run directory created in Step 0** — never the repo tree,
never git-committed (plans/RCAs/proof artifacts are scratch, not deliverables). Write to the skeleton
in `references/investigation-and-plan.md` rather than inventing a shape, and point the ledger's
`Artifacts` section at it.

The plan carries: goal/scope; numbered **fix sections F1/F2…**; a **test-coverage tree** tagging each
touched file with the tier that will prove it and the deterministic assertion that tier makes —
`[UNIT]` / `[E2E-web]` / `[E2E-native]` / `[DEVICE]` / `[JUDGE]` (content-correctness; a separate
proof from render/nav) — substituting the tiers this project actually has; migration/back-compat;
and risks. Use `AskUserQuestion`
here only to lock a genuine scope/architecture fork.

## Step 4 — Review gate A (engineering + design lenses)

A cheap single-pass expert review *before* the expensive adversarial pass. Run it **autonomously** —
never hand a sub-skill's own prompts to the user.

Priority ladder (use the first that applies):

1. **PREFERRED — `/autoplan` if installed.** It reads the eng/design (and CEO/DX) review skills from
   disk, runs them at full depth, and auto-decides every intermediate question via its decision
   principles, surfacing only genuine taste/product forks at one final gate. This is the sanctioned
   non-interactive path and faithfully runs the `/plan-eng-review` + `/plan-design-review` substance
   the user asks for.
2. **FALLBACK — the rubric subagent** (`references/plan-review-rubric.md`). Use this whenever:
   `/autoplan` is absent; **or** an invoked review skill reports `BLOCKED — AskUserQuestion
   unavailable`; **or** it otherwise stalls on interactivity it can't resolve. Do not count on a
   sub-skill's own non-interactive mode — those are typically gated on an orchestrator-specific
   env var that is unset in an ordinary session, so the skill takes its interactive path and halts. Run the eng-lens checklist always
   and the design-lens checklist for UI work, via a subagent, never touching the interactive skill.
   Run **both** lenses on UI work — the design lens has repeatedly caught bugs in the *engineering*
   proposal, which is why Step 2's lane is not optional.
3. **Only if the user explicitly asked to drive the plan review themselves** — invoke
   `/plan-eng-review` / `/plan-design-review` directly and let them ask. This is "Respect blunt
   overrides" applied to gate A.

Absent that explicit request, **gate A must never halt the pipeline.**

Fold findings back into the plan until the gate is CLEAR. Review output is a fallible **hypothesis**,
not an oracle: before folding a **high-severity or scope-changing** finding, spot-verify it against
real source/plan text (same discipline as Steps 1 and 5); low-severity findings may fold directly.

## Step 5 — Review gate B: the adversarial review (the signature move)

The phase that gives the loop its teeth. **Full template, lens catalog, and schemas in
`references/adversarial-review.md` — read it before running this the first time.** The shape is
invariant; the mechanism is not:

- **ATTACK** — N independent agents (N≈3–6, scaled to surface area), each pinned to exactly ONE named
  failure lens, each **required to cite `file:line`** and told *"do not invent problems the
  code/plan already handles."*
- **VERIFY** — a **fresh** agent per raw finding whose **default verdict is REFUTED / false-positive**
  unless it can construct a concrete, reachable failing sequence from the real code. It also
  **rejects a finding's proposed fix if that fix would regress prior shipped behavior.** This
  burden-of-proof asymmetry is the honesty mechanism that stops rubber-stamping — keep it.
- **SYNTHESIZE** — dedupe/rank survivors, emit an exact plan-edit per survivor, and issue
  **GO / GO-WITH-PATCHES / NO-GO.**

Run it via `Workflow` (backgrounded) if available, else as parallel subagents in the same three
stages. Every stage obeys the fan-out invariants: attackers, verifiers **and the synthesizer** read
`<run>/LEDGER.md` first (so no round re-litigates what a previous round already refuted, and the
verdict prose the user reads follows the run's `Style` entry), write their findings to
`<run>/agents/`, and are **supervised to close** — a stalled verifier is killed and re-run, and its
finding stays `unverified` rather than sliding into either bucket. Do useful prep while the wave
runs, but keep polling it. When it returns, **independently spot-verify the top ~2
survivors** against source, fold survivors into the plan, and mark the plan
`ENG[+DESIGN]+ADVERSARIAL CLEARED` before writing any implementation code. Calibration: ~40–65% of
raw findings typically survive verification — a round where nearly all or nearly none survive is a
smell; inspect the verify stage.

## Step 6 — Implement (direct, single-threaded)

Now, and only now, write code — **yourself, in the main thread.** Subagent fan-out is for
investigation and review; it is *not* for writing the implementation (parallel writers thrash a
shared tree and dilute your ownership of the diff).

- **Extract pure / dependency-injected helpers first and unit-test them first** — fast feedback
  before the riskier component rewrite.
- **Typecheck / compile after edits** (filter to touched files to dodge pre-existing errors in
  untouched modules if needed).
- **Minimize blast radius.** Touch the smallest set of files. Leave the project's stated invariants
  untouched — discover them from `CLAUDE.md`/README/tests (Step 5 folds them into the attack lenses).
- **Verify-before-spend.** Prove risky new code against a real *disposable* dependency (e.g. a
  throwaway Docker Postgres) BEFORE creating any billable or irreversible resource.
- Every confirmed defect gets **both a source fix AND a new regression test** pinned to the exact
  repro sequence.
- **Comments, docstrings, log lines and error strings follow the writing standard** above. Match the
  surrounding file's comment density — the standard governs how a comment reads, never how many you
  add. Identifiers and API names keep the spelling the code already uses.

## Step 7 — Agentic end-to-end validation (real artifact, deterministic verdict)

Prove the change works through simulated real use. **How-to + a discovery recipe + tier table in
`references/agentic-e2e.md`.** Two rules are sacred and must hold on every surface:

1. **The verdict is a deterministic assertion against the REAL running artifact — the LLM never
   decides pass/fail.** This generalizes beyond UIs:
   - **CLI:** spawn the built binary, feed real input, assert stdout / exit code.
   - **Library:** exercise the public API in a throwaway harness, assert observable outputs.
   - **Service:** hit the real running endpoint (a live-http spike against the deployed URL).
   - **UI:** real user clicks on the real screens (below).
   - **No obvious surface?** Reinterpret it — drive a real browser to read a real build log, or for
     a native asset (an app icon, a splash screen), a physical-device prebuild → install → observe
     loop. There is almost always a real artifact to exercise.
2. **Anti-hallucination:** an agent driving a UI may only interact with a selector actually present
   on the current screen; every value that matters is pinned by a deterministic `testID`/DOM/state
   assertion afterward.

**UI specialization (mandatory when a UI exists).** Discover the project's harness first (its
`package.json` `e2e:*` / `test` / `judge` scripts, `CLAUDE.md`, an `e2e/` dir, a Playwright/Maestro
config). Drive the REAL screens like a user — never a mock shell. Drivers, first available: the
project's own agentic harness; then any browser/simulator-driving MCP in your loaded or deferred
tools; then drive the artifact yourself; then author a harness out-of-tree. **At least one must be
stood up before you claim UI validation.** Only if none can be do you **flag the gap honestly** —
never silently downgrade "validated like a user" to "the unit tests pass." Before driving anything,
pin the four preconditions in `references/agentic-e2e.md` (artifact, assert channel, entry state,
driver) — a green against a stale artifact is the default failure here, not an edge case.

Pick the tier(s) by what the change actually touches, and keep the proofs **separate**: render/nav
proof (agentic drive), content-correctness proof (an LLM-judge harness with *deterministic*
hard-gates, quality scores report-only), native-only behavior (a physical device / `xcrun
devicectl` — honestly flagged as un-provable by web E2E), and a literal screenshot + multimodal read
for visual confirmation. Prove every NEW test **red/green** (revert the fix → it must fail →
restore → green) **and** that its seam is legitimate (see the no-fake-green rule in Step 8). Run the
targeted suite first, then the full regression gate (unit + typecheck + lint + the project's other
suites).

## Step 8 — Loop to green-with-proof

- **Debug from real signal; never guess twice.** See the actual failure first — real build log,
  hierarchy dump + screenshot, deterministic repro, live command output — then fix, then re-run.
  Diagnose failures one at a time.
- **Loop the adversarial review** R1→Rn — re-running the same three-stage shape against the
  *implementation diff*, not just the plan — each round stricter about rejecting
  unreachable/theoretical findings, until a round finds nothing on the crash-causing invariants.
  Bug work usually needs several rounds; feature work often one.
- **No cheating in tests (invariant).** Green is only real if the seam is legitimate. Red/green
  proves a test *fires*; it does not prove its seam is honest. A new/edited test must assert against
  a **sanctioned mechanism** (a recorded/replayed tape or a live call — never a stub that returns
  success) and pin **real observable state** (not a mock, not an app internal). Do not weaken an
  assertion to force green. The adversarial catalog's test-integrity lens enforces this on any
  test-touching change.
- **Anti-stall guard: a round only counts if it produced NEW observed signal** — a different failure
  signature, a new observed value, a new `file:line`. Log per round in the ledger what changed in the
  code, what the run observed, and how that differs from last round. **Two rounds with the same
  failure signature is a stall, not iteration:** stop patching and either open a channel you haven't
  read yet (a log, a state dump you add, a smaller repro) or escalate architecturally. After three
  rounds with no new signal, **stop and report what is unproven**, quoting the failure signature.
  This bounds *non-convergence* only — it never converts into a claimed pass, and convergence itself
  still terminates only on stable green-with-proof.
- **Staleness guard.** Re-run the FULL regression gate after ANY concurrent change so "no
  regressions" is proven against current state, not a stale pass. Real-device feedback from the user
  re-enters this same fix → typecheck → targeted-E2E → full-gate cycle.
- **Allow architectural escalation.** If incremental patches keep failing, prefer a structural change
  that makes the guarantee true *by construction* (e.g. merge two screens so no navigation fires, so
  there is nothing to remount) over another patch.

On infra/deploy work the macro loop runs once; convergence lives in the tight
fail → diagnose-from-real-output → fix → re-trigger micro-loop.

Terminate only on a stable green-with-proof re-run against current state — and emit the **proof
record** (`references/agentic-e2e.md`), never a prose "all green."

## Step 9 — Close out

- Persist the durable takeaway to a **memory file**, not a committed doc; update `CLAUDE.md` only for
  genuinely shared how-to.
- **Scoped `git add`** of only this change's files (never `git add -A`) when concurrent WIP is
  present; be transparent in the commit body about unavoidable shared-file bits. **Commit/push only
  when the user asks.**
- Restart the correct always-on process if you touched the running system; **kill your own orphaned
  background jobs / locks.**
- Spawn a background **task chip** for out-of-scope issues you noticed in passing rather than fixing
  them inline.
- **Scale-down boundary.** *Same feature* = the surface built in this session's full-ceremony pass;
  a correction to it takes the fast path (edit + tests + screenshot). A NEW stated directive
  re-enters the pipeline at proportional scale (the 2–6 investigators and N-scaled adversarial dials
  already right-size trivial surfaces). When unsure: a trivial in-file tweak is a correction (fast
  path); anything introducing a new module/seam/migration is a new directive (full pipeline). Do not
  reframe this dial around diff size or file count.

---

## Before you declare green

Each phase above states its own rules at its point of use. This list is not a summary of them — it
is the short set whose violation produces a green that is **wrong but looks right**, gathered at the
moment you are most tempted to skip them. Re-read it before you claim done.

1. Adversarial review kept its three-stage shape — single-lens attackers citing `file:line` → a
   **fresh** verifier per finding defaulting to REFUTED → synthesize. Batched or agreeable verifiers
   produce a clean report and catch nothing.
2. **Every fan-out wave closed with every member accounted for** — `done`, covered by you, or named
   `uncovered`. No coverage claim rests on an agent that never reported: a dead or stalled agent
   returns empty, and empty is indistinguishable from clean unless something counted it.
3. Review-agent output is a fallible hypothesis, not an oracle — the highest-stakes claims were
   re-verified against real source before you acted on them.
4. Every verdict came from REAL observed state, not reasoning-from-source-alone. You debugged from
   signal and never guessed twice.
5. The end-to-end verdict is a deterministic assertion **you did not author**, run against a
   rebuilt artifact; on a UI the agent only touched what was actually on screen.
6. No test cheats — sanctioned seams, real-state assertions, red/green proven through the build
   path. No assertion was weakened to reach green.
7. Content-correctness and render/navigation are still **separate** proofs for separate claims.
8. Whatever the tier you ran provably cannot exercise (native-only behavior, simulator gaps) is
   named as uncovered rather than folded into the pass.

## Graceful degradation (this is a global skill — assume nothing)

A named tool, MCP, or skill that is absent, deferred, or halting on interactivity is the **degraded
branch, never a hard failure**: take the fallback stated at that step and keep going. Each step owns
its own fallback; these are the ones not obvious from any single step:

- **Never run the attackers serially in the foreground.** Serial attackers see each other's findings,
  which destroys the independence the whole adversarial phase rests on — while still emitting a
  plausible-looking findings list.
- **No task board?** `TaskCreate` may be absent or deferred; track the phases in your replies instead.
  The board is how the loop is watched, not how it works.
- **No task-control tools?** `TaskList`/`TaskGet`/`TaskStop`/`Monitor` may be absent or deferred
  (`ToolSearch`-load them first). Supervision still runs: poll the `.progress` heartbeat files and
  the expected output paths directly. The heartbeat files are deliberately mechanism-independent —
  they are readable with nothing but `Read`/`Bash`. What you must never do is degrade supervision
  into waiting.
- **No writable dir outside the repo?** Say so, keep the ledger inline in your replies, and **shrink
  the waves** (2–3 agents, tighter budgets) — without file-backed state, a large fan-out is exactly
  what loses the run to a compaction.
- **Everything project-specific is a discovery step or a labeled example**, never a hardcoded
  command — any language's test runner, `/tmp` for scratch when no scratchpad exists.

## References

- `references/run-ledger-and-supervision.md` — the run-directory layout, the `LEDGER.md` contract,
  the five lines every spawned prompt carries, the universal `status` field, the
  spawn→poll→classify→close supervision loop, and the macro-loop anti-stall bound. Read before your
  first fan-out.
- `references/adversarial-review.md` — the reusable Attack→Verify→Synthesize template, failure-lens
  catalog (incl. the universal test-integrity / no-fake-green lens), JSON schemas, both Workflow and
  parallel-subagent forms.
- `references/agentic-e2e.md` — the deterministic-verdict principle, the four preconditions to pin
  before driving anything, harness discovery, tier table, driver fallbacks, red/green +
  seam-integrity proof, and the proof record.
- `references/investigation-and-plan.md` — the read-only ground-truth fan-out template (file:line
  JSON schema) and the scratchpad plan skeleton.
- `references/plan-review-rubric.md` — the eng-lens + design-lens review checklists, used as the
  autonomous fallback for gate A.
