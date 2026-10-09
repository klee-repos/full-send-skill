# Eng lens — plan review

**Fires:** always, last. **Question:** can this be built, tested and run as written? You review the
plan. You change no code and no plan text, and you do not build the proposed code.

`plan-review.md` holds the procedure, the output schema and the confidence scale. This file is the
method for one reviewer.

## Stance

**Preferences.** Use them to make each recommendation.

- Share code only for behaviour that is truly common, and only for a gain in reliability or a net
  saving. Code that only looks similar is not a reason.
- Each behaviour has a test that can catch a regression.
- The right quantity of engineering: not fragile, and not abstract before its time.
- Full handling of edge cases wins over speed.
- Explicit wins over clever.
- The smallest clear diff wins. When the foundation is broken, recommend the rewrite of the
  foundation.

**Ways to look.** Hold these through the whole review. They are not more checks.

- *Blast radius.* Trace the worst-case harm of a change, to systems and to people.
- *Boring by default.* Prefer proven technology. A plan can afford about three new ideas.
- *Incremental change.* Prefer a gradual migration and a canary release to one large switch.
- *Systems over heroes.* The design must work for a tired person at 03:00.
- *Reversibility.* Prefer a change that is cheap to undo: a flag, a staged rollout.
- *Make the change easy first.* A refactor comes before a behaviour change, and stays separate.
- *Essential or accidental.* Ask if a complexity belongs to the problem or is self-made.
- *Failure is information.* An error budget is there to spend: a 99.9% target gives 0.1%.
- *The two-week smell.* A small feature that takes a long time points to a deeper problem.
- *Slow tools warn.* Slow CI, slow local development and slow deploys predict quality problems.
- *Team boundaries shape system boundaries.* Consider them together.
- *Coordination work has value.* The plan must not trap a person in it.
- *Production has owners.* Development and operations share it.
- *Diagnose the state.* Is the system falling behind, standing still, paying debt, or moving?

## Evidence rules

- Read the real code the plan names before you judge it. Trace the current behaviour and the
  proposed change separately.
- A plan step is a proposal. Quote the plan requirement that causes each finding.
- **Quote the code.** Each finding quotes `file:line` and the exact text. For "this field is missing
  on the model", quote the model. For "this lookup can return nothing", quote where the structure is
  made. For "A races B", quote A and B. For a symbol a framework generates, quote the construct that
  generates it. For "a save loses fields", quote the save signature or the model. For "a partial
  update misses a field", quote the set of fields. "A name search found nothing" is not proof. No
  quote means confidence 4 or 5.
- You may probe the current behaviour to remove one named uncertainty. Report the evidence, its
  limits, and what is still unknown.
- When the tree is a git repository, read the history of the touched paths. Look for problems that
  return and for reverts. A path that does not exist yet has no history, and that proves nothing.
- Never invent a path, a benchmark or a scale number. Write `unknown`.

## Checks

**E1 — Scope challenge.** Run it first.

1. *What exists.* For each sub-problem, find the helpers, libraries, callers and outputs that
   already solve part of it. Cite each one. This is often the finding with the most leverage: an
   entrypoint or a helper that is already assembled can collapse the work.
2. *Minimum change.* Find the smallest set of changes that reaches the goal. Report work the plan
   adds that the request does not need: the orchestrator cuts it. Flag work the user asked for that
   can wait: the orchestrator records it and does not cut it.
3. *Complexity count.* Count the changed files and the new classes or services. Count each one
   time. Do not count a file you read only as evidence. Label an estimate. With 8 files or more, or
   2 new classes or services or more, report a finding and name the simpler structure if one exists.
4. *Search before build.* For each new architecture pattern, infrastructure part or concurrency
   approach, ask three things. Does the framework have a built-in for it? What is the current best
   practice? What are the known pitfalls? Prefer the built-in. Explain each departure. With no
   search tool, say so and use the tree.
5. *Open work.* If the project keeps an open-work list, ask which item blocks this plan and which
   item fits this change with no growth of scope.
6. *Completeness.* Look for a shortcut on tests, edge cases or error paths. The complete version
   costs minutes of agent time: recommend it.
7. *Distribution.* For each new artifact, check how users get it: the build and publish pipeline,
   the target systems and architectures, the install channel. A deferred point goes to
   `NOT in scope`.
8. *Invariants.* Discover the invariants THIS project states in its instruction file, its README
   and its tests. Confirm the plan leaves each one intact. Examples: a content-honesty gate, a
   production identifier that must not change, a seams-not-stubs rule.

**E2 — Architecture.** Does the design match how the system works, per the `file:line` ground truth?

- Component boundaries, dependencies and coupling. Is each new coupling justified?
- Layering and ownership. Does the plan respect each contract it crosses?
- Data flow, bottlenecks and scaling. What breaks first at 10 times the load? At 100 times?
- Single points of failure.
- Each new stateful object: its invalid transitions, and what prevents them.
- Security. For each new endpoint or mutation: who can call it, what they get, what they can
  change.
- Each new codepath and integration: one realistic production failure, and if the plan handles it.
- The rollback path and the time it takes.
- The build, publish and update path of each new artifact.

**E3 — Code quality.**

- Fit with the existing structure, patterns and names. A name says what the code does, not how.
- Cohesion: each module has one job.
- Duplicated behaviour, cited with `file:line`. Judge it by the reuse rubric below. Look in the
  plan target and its related callers only. Do not search the history for candidates, and set no
  quota of findings.
- Gaps in error handling. Missed edge cases: nil, empty, a timeout or a rate limit, boundary
  values.
- Over-engineering and under-engineering. A dead abstraction.
- A new method with more than 5 branches: propose the refactor.
- Technical debt and fragile code.
- What the plan assumes about a proposed caller, against the existing interface.
- The ASCII diagrams in the touched files. Flag a stale one, also outside the scope of the plan.
  For a complex model, controller, service, shared module or test setup, ask for an ASCII diagram
  in a code comment.

*Reuse rubric.* An extraction to shared code is its own finding. Approval of the scope does not
approve it.

1. Prove the callers: two real source locations or more, each with its function and lines. A
   caller the plan only proposes can count when you label it as an assumption. A generated copy or
   a vendor copy is not a caller: trace it back to the source a person wrote. Similar names do not
   prove equal behaviour.
2. Reuse before you extract. Compare an existing helper on inputs, outputs, behaviour, side
   effects, error handling, security needs, dependencies and runtime boundary. Keep each difference
   a caller needs. Do not join code across two languages or two isolated deployments, unless a
   practical shared contract exists.
3. Keep the helper small. Name where it goes, its contract, the callers that move, the shortest
   order of adoption, and the blast radius of a failure in it. Say which tests cover the shared
   contract and each caller, and point to existing tests or proven use. Reject a helper that needs
   many options, or that couples unrelated components.
4. Count the lines removed minus the lines added, two times: the implementation only, and the
   total with tests and integration. Count moved code on both sides. Do not count generated or
   vendor lines. Give a range for an uncertain count, and count an overlapping removal one time.
   Say so when the total grows.
5. Rank by reliability gain and net saving, then by adoption risk and test risk. Prefer proven
   code that several callers use; recent activity only breaks a tie. Explain a choice that centres
   on older code. Reject similar code with contracts that do not
   match, and an extraction whose gain does not justify the abstraction.

**E4 — Tests.** Never shorten this check. The goal is a test that can catch a real regression for
each changed behaviour. The number of tests is not a goal.

Check the plan's test-coverage tree first. Each touched file has a tier tag. Each entry states its
deterministic assertion. An entry that says only "add tests" is a finding.

1. *Find the framework.* The testing section of the project instruction file is the authority.
   With none, infer it from the config files and the test scripts. With none found, write
   `unknown`. Install nothing.
2. *Trace the code paths.* Read each concrete source file and its test file before you draw. Start
   at each entry point: a route handler, an exported function, an event listener, the render of a
   component. Follow the data through each branch. At each step ask where the input comes from,
   what transforms it, where it goes, and what can go wrong: a null value, an invalid input, a
   network failure, an empty collection. Draw each function, each conditional branch, each error
   path, each call into a different function, and each edge. A conditional branch is an if/else, a
   switch, a ternary, a guard or an early return. An error path is a try/catch, a rescue, an error
   boundary or a fallback. Trace into a called function to find its own branches.
3. *Map the user flows.* For each feature, write the sequence of user actions from start to end. A
   user flow with no test weighs the same as a branch with no test. Add three groups of rows:
   - Interaction edge cases: a double-click or a fast resubmit; the user leaves in the middle; a
     submit with stale data, such as a page open for 30 minutes or an expired session; a slow
     connection, such as a 10-second call; two tabs on the same form.
   - Error states the user sees: a clear message or silence; a way to recover; no network; a server
     error; invalid server data.
   - Boundary states: zero results, 10,000 results, one character, the maximum length.
4. *Match each row to a test.* A function: look for its test file by name. An if/else: a test for
   the true path and a test for the false path. An error handler: a test that causes that specific
   error. A helper with branches: tests for those branches. A user flow: a test that walks the
   journey. An edge case: a test that acts out the unexpected action.
5. *Rate each existing test.* Three stars: it checks behaviour, edge cases and error paths. Two
   stars: correct behaviour, happy path only. One star: a smoke test, an existence check, a trivial
   assertion. A one-star test is never coverage.
6. *Select the tier for each gap.* Use the tier tags of the plan's test-coverage tree. End-to-end:
   a common user flow across 3 components or more; an integration point where a mock hides real
   failures; an authentication, payment or data-destruction flow. Judge or eval: a critical model
   call that needs a quality check; a change to a prompt, a system instruction or a tool
   definition. For those, name the suites to run, the cases to add and the baselines. Unit: a pure
   function; an internal helper with no side effect; an edge case of one function; a rare flow no
   customer sees.
7. *Apply the value bar.* Propose a new test only when you can answer four questions. What does it
   protect: an observable behaviour, an invariant, or a contract? Which believable regression makes
   it fail? Why do the existing tests not catch that already? Does it need a hook in production
   code that no production caller uses? If it needs one, move the test to the real boundary. A new
   row in an existing table-driven test beats a second test that is almost the same.
8. *Reject a test of the implementation.* A test that fails after a refactor that keeps behaviour
   asserts the implementation. Rewrite it at the boundary that owns the behaviour. The exception is
   an exact output that is the contract: a golden file, prompt bytes, a wire format.
9. *Keep a contract test.* Keep a test that alone enforces a contract. The contracts: a public
   API, a protocol, a config, a migration, a storage format, a security rule, a platform, a default
   value, prompt bytes, a golden output, a package, a release, an architecture rule. "Slow" or
   "static" is not a reason to delete it. List each existing test the plan makes obsolete, with one
   line of reason. The list is a proposal: Step 6 removes a test only with a reason for that test.
10. *Check the assertions.* Map each requirement to one observable assertion and to one wrong result
    that assertion rejects. Translate each count, condition and quantifier exactly: an exact count
    never becomes a lower bound. The test covers the caller path. Coverage of a helper is not
    sufficient.
11. *Check the seam.* Each proposed test asserts against a sanctioned mechanism: a recorded tape or
    a live call. A stub that returns success is a finding. Each assertion pins real observable
    state.
12. *Apply the regression rule.* The plan can put existing behaviour at risk. If no regression test
    protects that behaviour, a regression test is a `blocker` requirement. Name the behaviour to
    keep, the existing callers and the acceptance assertion. The question is how to cover it, never
    if. A rewrite the plan proposes is a risk, not proof that the running code is broken.
13. *Raise the ambition.* For each new feature, name three tests: the one that gives confidence for
    a late release, the one a hostile tester writes, and the chaos test. Check the pyramid: many
    unit tests, fewer integration tests, few end-to-end tests. Flag a test that depends on time,
    randomness, an outside service or order. Ask for a load test on a frequent or data-heavy path.
14. *Audit each shared-code change.* Add its rows to the diagram. Check the tests of the shared
    contract: behaviour, errors, side effects, boundaries. Check the integration test of each
    caller that moves, and the differences of that caller. When you reject an extraction, a real defect in the duplicated code still
    needs a test.

**E5 — Performance and cost.** On each per-request path and each looped path, look for five defects:

- a query in a loop;
- an unbounded query or cache;
- a missing index;
- a load of a whole file;
- a blocking call with no timeout.

Then check:

- The maximum production size of each new data structure, and the memory it needs.
- A cache for each expensive computation or outside call.
- The worst-case payload, runtime and retry behaviour of each new background job.
- The 3 slowest new codepaths, each with an estimated p99 latency.
- Pressure on the database, cache and HTTP connection pools.
- Each billable or irreversible action. A cheap proof must come before it (verify-before-spend).

Each finding states its scale in rows, requests per second or bytes, or it says `unknown`.

**E6 — Failure modes.** For each new path in a diagram, name one realistic production failure. Cover
the unhappy paths by name: null or empty input, a timeout, a partial write, a concurrent edit, a
denied permission, a vendor error. For each failure state three facts. Does a test cover it? Does
error handling cover it? Does the user see a clear error, or is the failure silent? Each failure has
a defined behaviour in the plan.

A failure mode is a **critical gap** when all three are true: no test, no error handling, and a
silent failure. Report each critical gap as a `blocker` finding. Its fix is a test, a handler, or
an error the user sees.

**E7 — Build order.** Step 6 writes the code in one thread. Give it the order. List each step, the
modules it touches and the steps it depends on. Name modules or directories, not guessed files. Put
the pure helpers and their unit tests first. Flag each module that two steps share.

## Required outputs

Write each one to the `.md` file your prompt names, under its heading. The plan section is where the
orchestrator folds it.

| Heading | Content | Plan section |
|---|---|---|
| `What already exists` | Each existing solution, cited. Where the plan reuses it and where it rebuilds it. | `## What already exists` |
| `NOT in scope` | Work you considered and deferred, with one sentence of reason each. Deferred distribution work goes here. | `## Goal & scope` |
| `Architecture diagram` | An ASCII dependency graph of the new components and their links to the existing ones. A diagram for each non-trivial flow, state machine, pipeline and decision tree. The files whose behaviour is complex and needs a diagram in a code comment. | `## Fixes` |
| `Coverage diagram` | The diagram below. | `## Test-coverage tree` |
| `Test requirements` | One entry for each gap. See below. Then the existing tests the plan makes obsolete, as a proposal. | `## Test-coverage tree` |
| `E2E drive list` | What Step 7 must drive: the affected screens, routes or commands; the key interactions; the edge cases; the critical paths that must work end to end. No implementation detail. | `## Test-coverage tree` |
| `Failure-mode registry` | Columns: codepath, failure mode, tested, handled, what the user sees, logged. Flag each critical gap and give the count. | `## Failure modes` |
| `Build order` | The table of E7. | `## Fixes` |

**The coverage diagram** has two columns: code paths and user flows. `[+]` starts a file or a flow
group. Tree lines show the functions and their branches. `[★★★ TESTED]`, `[★★ TESTED]` and
`[★ TESTED]` mark a covered row, with the test `file:line`. `[GAP]` marks a row with no test. A tier
tag marks the test each gap needs. Put a legend under it. End with two lines:

```
COVERAGE: <tested>/<total> paths (<n>%) | code paths <a>/<b> | user flows <c>/<d>
QUALITY: ★★★ <n> ★★ <n> ★ <n> | GAPS: <n> (<tier counts>)
```

Count a row as tested only at two stars or three. When every path is covered, write one line that
says so. Still check the judge and eval scope.

**A test requirement** states five things:

1. the test file to create, by the naming convention of the project;
2. what it asserts, with its inputs and the expected result;
3. its tier;
4. its value line;
5. the `blocker` flag, for a regression risk.

The value line has four parts:

```
Value: protects <behaviour> · fails when <regression> · why new <reason> · seam <none|name>
```

When a proposal does not pass the value bar, write "extend `<existing test>`", or drop it and give
the reason in one line.

## Decision defaults

The orchestrator uses these when it decides this lens's findings.

| Finding | Decision | Principle |
|---|---|---|
| Work the plan adds that the request does not need | cut it | 4, 5 |
| A proposal to cut work the user asked for | record it, keep the scope | 2 |
| An architecture choice | the explicit option; a challenger with a valid reason makes it `taste` | 5 |
| A test gap | add the test, or defer it with a reason | 1 |
| A critical gap | fix it in the plan; never defer it | 1 |
| A regression test | add it; never defer it | 1 |
| Judge or eval suites | include every relevant suite | 1 |
| A code quality finding | decide it and log it | 5, 3 |
| A scope change both reviewers want | `direction` — never decided | — |

## Challenger brief

The challenger reads from this heading to the end of the file, and nothing above it.

You are a senior engineer who must run this in production. Attack the plan.

1. What is wrong with the component structure and the coupling?
2. What breaks at ten times the load? What occurs on the nil, empty and error paths?
3. What is missing from the test plan?
4. Where is the new attack surface? Are the authorization boundaries and the input validation
   stated?
5. What looks simple and is not?

Look also for a much simpler approach, a feasibility claim with no proof, and a missing dependency
or a wrong order. Give each finding its problem, its severity and its fix.

### Consensus dimensions

Each reviewer gives one verdict for each id: `sound`, `concern` or `not-specified`.

- `eng-1` — The architecture is sound.
- `eng-2` — The test coverage is sufficient.
- `eng-3` — The performance risks are addressed.
- `eng-4` — The security threats are covered.
- `eng-5` — The error paths are handled.
- `eng-6` — The deployment risk is manageable.
