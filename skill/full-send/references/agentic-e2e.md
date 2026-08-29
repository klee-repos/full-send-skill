# Agentic end-to-end validation

The demand this satisfies: *"validate it completely using e2e agentic tests, visually and logically,
just like a user — loop until verified with no regressions."*

> **The verdict is a deterministic assertion against the REAL running artifact, authored by code —
> the LLM perceives and navigates, but never decides pass/fail.**

## Contents
- [The generalized verdict](#the-generalized-verdict-any-surface)
- [Before the first assertion](#before-the-first-assertion)
- [UI specialization](#ui-specialization-mandatory-when-a-ui-exists)
- [Harness discovery](#harness-discovery)
- [Tier selection](#tier-selection)
- [Driver fallbacks](#driver-fallbacks)
- [Proof integrity](#proof-integrity)
- [The proof record](#the-proof-record)

## The generalized verdict (any surface)

The change might not have a UI. The principle holds — exercise the **real** artifact and assert
observable output deterministically:

| Surface | Drive it by | Deterministic verdict |
|---|---|---|
| CLI | spawn the built binary, feed real input/flags | assert stdout / stderr / exit code |
| Library / module | call the public API in a throwaway harness | assert return values / emitted events / side effects |
| HTTP service | hit the real running endpoint (a live-http spike) | assert status / body shape / persisted row |
| UI | agent taps the real screens like a user | assert `testID`/DOM/state after each step |
| Analysis / report | re-run the derivation from the frozen input by a second, independent script | assert the two results match, and that a mutated input flips the result |
| Batch / pipeline | run the job over a real input set you can count | assert per-item coverage and the output rows, not only the exit code |
| Document / prose | extract each checkable claim with its source locator | a script re-reads each source and asserts the claim; non-zero on the first miss |
| Data migration | run the migration on a copy of the real data | assert the after-state against the before-state, row by row, before the real run |

In every row, an LLM may *drive* or *perceive*, but the verdict is a deterministic check: code that
passes or fails on its own, written **before** the run and not rewritten to match what happened.
You may author that check — the last rungs of the driver ladder below require it. You may not *be*
it, or edit it once you have seen the output. If a surface has no drivable path, **say so** — do
not substitute "the unit tests pass" for "it works like a user."

**Two corollaries for every row.** *The write path may not be its own witness* — if the claim
includes a durable effect, observe it through a channel the write did not own (a fresh
connection, a reload, a cold client, a direct query at the store), and tag fixtures with a
per-run unique marker. Scope every assertion to an identifier this run created. Do not assert
"at least one", and do not assert a count, over a collection earlier runs share. And *collect the
artifact's error channels during the run* — uncaught errors, non-success requests the run made, the
process log — and gate on them with your assertions. Exclude pre-existing noise by name in the
proof record, never silently.

## Before the first assertion

Five things are decided *before* you drive, not discovered after:

| Pin down | The rule | Skip it and |
|---|---|---|
| **Artifact** | Rebuild / reinstall / redeploy from the working tree, then prove it took — a build or version marker, or the negative baseline the last row demands. Step 7 runs on an **uncommitted** tree. Reach it from outside: a harness that calls the app object in-process, or imports `src/`, is the unit tier in e2e clothing. | a 200 from last week's build |
| **Assert channel** | Name where you will read the deterministic value, before you tap. First that works: a queryable element/state tree the driver exposes; a real out-of-band side effect (persisted row, written file, request sent); readback you add (a debug-gated state dump, one machine-readable log line). An act-only driver authorizes no verdict — **pixels corroborate, they never decide**. | narrating a screenshot and calling it deterministic |
| **Entry state** | Use a mechanism the project ships — seed script, documented test account, dev-only impersonation route, disposable store. Never fabricate the door — no forged session, no patched auth guard. Re-establish it **every** run; residue (granted permissions, first-run flags, caches, seeded rows) makes run N skip the branch under test and still pass. | a green that quietly stopped testing anything |
| **Driver** | Emit it as **one re-runnable command** — a scratchpad script is enough: start, wait on a readiness *condition* (never a sleep), act, assert, exit non-zero, tear down on every path. Every wait gets a hard timeout that **exits non-zero**: a driver blocked forever on a selector that never appears hangs the step, and looks like a slow test. Bind a port you chose; a listener you did not start is serving someone else's code. The same holds for every shared resource the developer is using: a device, a simulator, a database, a cache directory, a cloud project. Use your own instance. Restore what you changed. | red/green and the staleness re-run have nothing to run |
| **Assertion baseline** | Run the assertion against the entry state BEFORE you act. It must fail. Act, then run it again. Record both results. This is not the red/green rule. An assertion that the entry state already satisfies stays *green* after the revert. The staleness rule then blames the artifact and not the assertion. | an assertion that already passed |

## UI specialization (mandatory when a UI exists)

Two rules, enforced in code:

1. **Anti-hallucination.** A planned action may only reference a control present on the current
   screen (perceived from the live `testID`/accessibility tree). Inventing a selector is a hard
   error, not a guess.
2. **No pass/fail authority.** The agent only *navigates* toward a natural-language goal. Every value
   that matters is pinned by a deterministic `testID`/DOM/state assertion afterward (e.g.
   `data-state`, `audio.paused`, a chip count). The planner may be a script in CI or an LLM in dev,
   reading the same perceived tree.

Model real-user constraints: browser autoplay policy (media blocked until a real gesture),
off-screen/occluded controls excluded from perception, human-paced settle between taps.

## Harness discovery

This is a global skill — discover the project's harness, do not assume one:

```sh
# 1. Named test/e2e/judge entrypoints (non-JS: a Makefile target, pytest/conftest.py,
#    go test, cargo test, a Gradle/xcodebuild test task):
jq -r '.scripts | to_entries[] | "\(.key): \(.value)"' package.json 2>/dev/null \
  | grep -iE 'e2e|agentic|test|judge|maestro|playwright|cypress'
# 2. How the project says to run them:
grep -rniE 'e2e|agentic|maestro|playwright|deterministic|testID|real screens' \
  CLAUDE.md README.md 2>/dev/null
# 3. Harness on disk (find, not globs — monorepos nest these deep):
find . \( -name node_modules -o -name .git -o -name dist -o -name build \) -prune -o \
  \( -name '*playwright*' -o -name '*cypress*' -o -name '*maestro*' -o -name 'e2e' \
     -o -name 'conftest.py' \) -print 2>/dev/null
```

Prefer the project's own harness over anything you stand up ad hoc. (Glob/Grep work too.)

## Tier selection

Choose tiers by what the change exercises, and keep the proofs **separate** — one tier's green does
not cover another's claim:

- **Web/agentic drive** — render, navigation and interaction for anything the web (or the app's own
  agentic harness) can reach. The default for UI changes.
- **Content-correctness (LLM-judge) harness** — when output *quality/correctness* is the claim.
  Run over a real-input matrix with **deterministic
  hard-gates** that gate the exit code; any LLM quality score is **report-only, never gating**. This
  is a DIFFERENT proof from the render/nav proof — never conflate them.
- **Native tier** (a driver running the real compiled binary on a simulator/emulator) — gestures,
  permission dialogs, navigation, and anything the browser can't reach. Native accessibility tree
  only (resource-id, leaf elements). The simulator is a lossy tier: name what it cannot exercise
  (no camera or mic, no biometrics, no real network/GPU) and split the claim.
- **Physical device / `xcrun devicectl`** — native-only behavior (platform glass/blur, screen
  transitions, audio frameworks) that web/CLI E2E cannot exercise. Flag it: either the user
  drives the device or the agent does via `devicectl`.
- **Screenshot + multimodal read** — literal "visually, just like a user" confirmation, layered on
  top of the deterministic assertions (never instead of them).
- `[ANALYSIS]` — freeze the input, re-derive the number by an independent path **that is a script,
  not a reading**, and compare the two derivations by exit code. Then mutate the
  input to prove the gate is not vacuous.
- `[BATCH]` — assert coverage of the input, not only the exit code.
- `[DOCUMENT]` — extract each checkable claim to a machine-readable list, one line each with its
  source locator, then let a script re-read every source and assert the value. The script exits
  non-zero on the first mismatch.
- `[MIGRATION]` — prove the data after the move against the data before it, on a copy, before the
  real run.

A tier changes WHAT the assertion reads, never WHETHER an assertion decides.

## Driver fallbacks

Use the first available; at least one MUST be stood up before claiming UI validation:

1. the project's own `e2e:*` / agentic harness (best — it already encodes the two rules);
2. a browser- or simulator-driving tool already in your tools. Scan the loaded **and deferred**
  lists by *class*, not
   brand, and batch-load candidates in ONE `ToolSearch` call;
3. drive the real running artifact yourself and assert on its DOM / state / exit code.

No external QA skill sits on this ladder: one that autonomously *fixes* what it finds takes
ownership of your diff, and one that demands a clean tree cannot run here.

**A driving agent is a supervised agent.** A UI driver is the likeliest agent here to loop without
progressing: it re-perceives the same screen and re-plans the same tap. Bound it in the prompt — a
step budget, a heartbeat line per step appended to `<run>/agents/e2e-<flow>.progress`, and "if the
same screen state repeats twice, stop and report `status:"blocked"` with what you saw". A repeated
state is a stall. Kill and re-run narrower rather than waiting; if two attempts stall, drive that
flow yourself. A stalled driver is never a fail *or* a pass. It is an unproven claim, and the run
names it as one.

**Record which rungs you rejected**, and why, before you author one: a self-authored harness is the
weakest rung, and no run reaches it by default. If no rung is already available, **author one**: a
suite in the platform's own e2e framework, or a
driver script in the scratchpad. Install any tooling **out-of-tree** — never add a dependency
entry, lockfile change, or CI job as a
side effect of proving one change. Only when that fails too do you **flag the gap**, not downgrade silently.

## Proof integrity

- **Red/green every new test.** Revert the fix → the test must fail → restore → it passes.
- **Mutation-red the check.** Break the input on purpose and prove the check goes red. Red/green
  proves the check fires against YOUR change; mutation-red proves it discriminates at all.
- **Red/green must traverse the build path.** Reverting source proves nothing until you rebuild the
  artifact: revert → rebuild/redeploy → red → restore → rebuild → green, and record both
  commands. "Revert" here is an in-place edit you undo — **never** `git stash` / `git checkout --` /
  `git restore`, which take the whole uncommitted implementation with them.
  Disable the one seam the claim rests on (the write call, the guard, the persistence hop), not the
  feature: a feature-shaped revert reds because the screen is gone.
- **Demand the red you predicted.** Read the failure. A red from a build error, a dead port, a
  missing fixture, or a selector timeout says nothing about the assertion — repair the harness and
  re-red. And if the revert stays *green*, suspect a stale artifact or a weak assertion before you
  suspect the test. Keep the text you replaced, and put it back by hand: the undo must work with no
  version control.
- **Seam legitimacy (no fake green).** Red/green proves the test *fires*, not that its seam is
  honest. Assert against a sanctioned mechanism (a recorded tape or a live call, never a stub that
  returns success) and pin real observable state, not a mock or an internal. Do not weaken an
  assertion to force green.
- **Targeted first, then the full gate.** Re-run the single suite after each fix; then run the
  full regression gate (unit + typecheck + lint/selectors + the project's other suites), and
  re-run it after ANY concurrent change so "no regressions" is current, not stale.

### The proof record

"Green-with-proof" is a **record, not an adjective** — an unreported proof is a claim. Per proven
claim, write to `<run>/proof/` and summarize in the reply:

- the exact re-runnable invocation and its exit code;
- each stated requirement, mapped to its own proof line; where the requirement is about what a
  person perceives, that line records an observation of the real output, in addition to the
  deterministic assertion and never in place of it — not a check that a locator resolved;
- what was driven — build/version marker, target (URL/port/device/path), entry state;
- the raw observed value behind each assertion, quoted, never paraphrased;
- both halves of red/green, and what was reverted to force the red;
- error-channel output, and anything excluded as pre-existing, by name;
- screenshot / log / trace paths — layered on the assertions, never in place of them;
- the tiers **not** covered, named — including any flow left unproven because its driver stalled.

Write the record's prose to the run's writing standard, but **the standard stops at the evidence**.
Quoted values, invocations, exit codes, error text and `file:line` cites are copied exactly.
