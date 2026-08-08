# Agentic end-to-end validation

The demand this satisfies: *"validate it completely using e2e agentic tests, visually and logically,
just like a user — loop until verified with no regressions."* The core idea is one sentence:

> **The verdict is a deterministic assertion against the REAL running artifact, authored by code —
> the LLM perceives and navigates, but never decides pass/fail.**

## Contents
- [The generalized verdict (any surface)](#the-generalized-verdict-any-surface)
- [Before the first assertion](#before-the-first-assertion)
- [UI specialization (mandatory when a UI exists)](#ui-specialization-mandatory-when-a-ui-exists)
- [Harness discovery](#harness-discovery)
- [Tier selection](#tier-selection)
- [Driver fallbacks](#driver-fallbacks)
- [Proof integrity](#proof-integrity)
- [The proof record](#the-proof-record)

## The generalized verdict (any surface)

The change might not have a UI. The principle holds regardless — exercise the **real** artifact and
assert observable output deterministically:

| Surface | Drive it by | Deterministic verdict |
|---|---|---|
| CLI | spawn the built binary, feed real input/flags | assert stdout / stderr / exit code |
| Library / module | call the public API in a throwaway harness | assert return values / emitted events / side effects |
| HTTP service | hit the real running endpoint (a live-http spike) | assert status / body shape / persisted row |
| UI | agent taps the real screens like a user | assert `testID`/DOM/state after each step |

In every row, an LLM may help *drive* or *perceive*, but the verdict is a deterministic check —
code that passes or fails on its own, committed to **before** the run and not rewritten to match
what happened. You may author that check (the ladder's last rungs require it); what you may not do
is *be* it, or edit it once you have seen the output. If a surface has genuinely no drivable path,
**say so** — do not substitute "the unit tests pass" for "it works like a user."

**Two corollaries for every row.** *The write path may not be its own witness* — if the claim
includes a durable effect, observe it through a channel the write did not own (a fresh
read/connection, a reload, a cold client, a direct query at the store), and tag fixtures with a
per-run unique marker so the state you read back provably came from THIS run. And *collect the
artifact's error channels for the duration of the run* — uncaught errors, non-success requests the
run triggered, the process log — and gate on them alongside your assertions: an optimistic UI
renders success over a write that failed, and a DOM-only assertion passes it. Exclude pre-existing
noise by name in the proof record, never silently.

## Before the first assertion

Four things are decided *before* you drive, not discovered after:

| Pin down | The rule | Skip it and |
|---|---|---|
| **Artifact** | Rebuild / reinstall / redeploy from the working tree, then prove it took — a build or version marker, or a negative baseline you observed (the new surface absent *before*, present *after*). Step 7 runs on an **uncommitted** tree, so nothing long-lived contains your change by default. Reach it from outside: a harness that calls the app object in-process, or imports `src/`, is the unit tier in e2e clothing. | a 200 from last week's build |
| **Assert channel** | Name where the deterministic value will be read from, before you tap. First that works: a queryable element/state tree the driver exposes; a real out-of-band side effect (persisted row, written file, request actually sent); readback you add (a debug-gated state dump, one machine-readable log line). An act-only driver authorizes no verdict — **pixels corroborate, they never decide**. | narrating a screenshot and calling it deterministic |
| **Entry state** | Use a mechanism the project already ships — seed script, documented test account, dev-only impersonation route, disposable store. Never fabricate the door: a forged session or a patched auth guard is a mock shell at the boundary that matters most. Re-establish it **every** run; residue (granted permissions, first-run flags, caches, seeded rows) makes run N skip the branch under test and still pass. | a green that quietly stopped testing anything |
| **Driver** | Emit it as **one re-runnable command** — a scratchpad script is enough: start, wait on a readiness *condition* (never a sleep), act, assert, exit non-zero, tear down on every path. Every wait gets a hard timeout that **exits non-zero**, never an unbounded one: a driver blocked forever on a selector that will never appear is the single most common way this step hangs, and it looks exactly like a slow test. Bind a port you chose; a listener you did not start is serving someone else's code. | red/green and the staleness re-run have nothing to run |

## UI specialization (mandatory when a UI exists)

Two rules, enforced in code, treated as sacred:

1. **Anti-hallucination.** A planned action may only reference a control actually present on the
   current screen (perceived from the live `testID`/accessibility tree). Inventing a selector is a
   hard error, not a guess.
2. **No pass/fail authority.** The agent only *navigates* toward a natural-language goal. Every value
   that matters is pinned by a deterministic `testID`/DOM/state assertion afterward (e.g.
   `data-state`, a band attribute, `audio.paused`, a chip count). The planner can be scripted in CI
   (deterministic, no key) or an LLM in dev reading the same perceived tree — either way the
   assertions are the verdict.

Model real-user constraints faithfully: browser autoplay policy (media blocked until a real gesture),
off-screen/occluded controls excluded from perception, human-paced settle between taps. A test that
bypasses the constraints a user faces is theater.

## Harness discovery

This is a global skill — discover the project's harness instead of assuming one:

```sh
# 1. Named test/e2e/judge entrypoints (non-JS: a Makefile target, pytest/conftest.py,
#    go test, cargo test, a Gradle/xcodebuild test task):
jq -r '.scripts | to_entries[] | "\(.key): \(.value)"' package.json 2>/dev/null \
  | grep -iE 'e2e|agentic|test|judge|maestro|playwright|cypress'
# 2. How the project itself says to run them:
grep -rniE 'e2e|agentic|maestro|playwright|deterministic|testID|real screens' \
  CLAUDE.md README.md 2>/dev/null
# 3. Harness on disk (find, not globs — monorepos nest these several levels deep):
find . \( -name node_modules -o -name .git -o -name dist -o -name build \) -prune -o \
  \( -name 'playwright.config.*' -o -name 'cypress.config.*' -o -name '*maestro*' \
     -o -name '.maestro' -o -name 'e2e' -o -name 'conftest.py' \) -print 2>/dev/null
```

Prefer the project's own agentic/real-screen harness over anything you stand up ad hoc. (Glob/Grep
work too, and dodge shell-quoting entirely.)

## Tier selection

Choose tier(s) by what the change actually exercises, and keep the proofs **separate** — one tier's
green does not cover another's claim:

- **Web/agentic drive** — render, navigation, interaction for anything the web/RN-web (or the app's
  own agentic harness) can reach. The default for UI changes.
- **Content-correctness (LLM-judge) harness** — when output *quality/correctness* is the claim (e.g.
  "the description always names what it is"). Run over a real-input matrix with **deterministic
  hard-gates** that gate the exit code; any LLM quality score is **report-only, never gating**. This
  is a DIFFERENT proof from the render/nav proof — never conflate them.
- **Native tier** (e.g. Maestro driving the real compiled binary on a simulator/emulator) — gestures,
  permission dialogs, navigation, and anything the browser can't reach. Native accessibility tree
  only (resource-id, leaf elements). The simulator is itself a lossy tier: name what it provably
  cannot exercise (no camera or mic, no biometrics, no real network/GPU) and split the claim rather
  than letting that residue ride inside the validated part.
- **Physical device / `xcrun devicectl`** — native-only behavior (platform glass/blur, screen
  transitions, audio frameworks) that web/CLI E2E provably cannot exercise. Flag this honestly and
  either the user drives the device or the agent does via `devicectl`.
- **Screenshot + multimodal read** — literal "visually, just like a user" confirmation, layered on
  top of the deterministic assertions (never instead of them).

## Driver fallbacks

Use the first available; at least one MUST be stood up before claiming UI validation:

1. the project's own `e2e:*` / agentic harness (best — it already encodes the two rules);
2. a browser- or simulator-driving tool already in your tools — `claude-in-chrome`, a Playwright
   MCP, a simulator/emulator control MCP. Scan the loaded **and deferred** lists by *class*, not
   brand, and batch-load candidates in ONE `ToolSearch` call;
3. drive the real running artifact yourself and assert on its DOM / state / exit code.

No external QA skill sits on this ladder on purpose: a skill that autonomously *fixes* what it finds
takes ownership of a diff that is yours, and one that demands a clean git tree cannot run at all
here — Step 7 is reached with the whole implementation uncommitted.

**A driving agent is a supervised agent.** Whatever rung you land on, an agent driving a UI is the
likeliest thing in this pipeline to loop without progressing — it re-perceives the same screen,
re-plans the same tap, and emits plausible narration the whole time. Bound it in the prompt: a step
budget, a heartbeat line per step appended to `<run>/agents/e2e-<flow>.progress`, and an explicit
"if the same screen state repeats twice, stop and report `status:"blocked"` with what you saw" —
a repeated state is a stall, and continuing past it just manufactures a longer transcript. Kill and
re-run narrower rather than waiting; if two attempts stall, drive that flow yourself. The verdict is
the deterministic assertion either way, so a stalled driver is never a fail *or* a pass — it is an
unproven claim, and it gets named as one.

If no rung is already available, **author one**: a suite in the platform's own first-party UI/e2e
framework, or a driver script in the scratchpad. Install any tooling **out-of-tree** (one-shot
runner, scratchpad-local, container) — never add a dependency entry, lockfile change, or CI job as a
side effect of proving one change. Only when that fails too do you **flag the gap** rather than
downgrading silently.

## Proof integrity

- **Red/green every new test.** Revert the fix → the test must fail → restore → it passes. A test
  that can't go red proves nothing.
- **Red/green must traverse the build path.** Reverting source proves nothing until the artifact is
  rebuilt from it: revert → rebuild/redeploy → red → restore → rebuild → green, and record both
  commands. "Revert" here is an in-place edit you undo — **never** `git stash` / `git checkout --` /
  `git restore`, which take the whole uncommitted implementation and any concurrent WIP with them.
  Disable the one seam the claim rests on (the write call, the guard, the persistence hop), not the
  feature: a feature-shaped revert reds because the screen no longer exists, proving only that the
  driver ran.
- **Demand the red you predicted.** Read the failure. A red from a build error, a dead port, a
  missing fixture, or a selector timeout says nothing about the assertion — repair the harness and
  re-red. And if the revert stays *green*, suspect a stale artifact before you suspect the test.
- **Seam legitimacy (no fake green).** Red/green proves the test *fires*, not that its seam is
  honest. Assert against a sanctioned mechanism (recorded/replayed tape or a live call — never a stub
  that returns success) and pin real observable state, not a mock or an internal. Do not weaken an
  assertion to force green.
- **Targeted first, then the full gate.** Re-run the single suite after each fix for speed; then run
  the full regression gate (unit + typecheck + lint/selectors + the project's other suites), and
  re-run it after ANY concurrent change so "no regressions" is current, not stale.

### The proof record

"Green-with-proof" is a **record, not an adjective** — an unreported proof is a claim. Per proven
claim, write to `<run>/proof/` and summarize in the reply:

- the exact re-runnable invocation and its exit code;
- what was driven — build/version marker, target (URL/port/device/path), entry state;
- the raw observed value behind each assertion, quoted, never paraphrased;
- both halves of red/green, and what was reverted to force the red;
- error-channel output, and anything excluded as pre-existing, by name;
- screenshot / log / trace paths — layered on the assertions, never in place of them;
- the tiers **not** covered, named — including any flow left unproven because its driver stalled out.

Write the record's prose to the run's writing standard, but **the standard stops at the evidence**.
Quoted values, invocations, exit codes, error text and `file:line` cites are copied exactly. A
simplified observed value is not a proof, it is a paraphrase of one.
