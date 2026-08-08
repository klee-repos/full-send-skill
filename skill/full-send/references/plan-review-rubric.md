# Plan-review rubric (autonomous fallback for gate A)

Use this when `/autoplan` isn't installed, when an invoked review skill reports `BLOCKED —
AskUserQuestion unavailable`, or when you otherwise need the review substance without an interactive
skill. Run it via a subagent (or inline): apply the **eng-lens** checklist to every plan, and the
**design-lens** checklist additionally for UI/design work. Produce findings in the same shape the
adversarial stage uses (`title / severity / evidence(file:line) / breakScenario / proposedFix`), fold
them back into the plan until clean, and spot-verify any high-severity finding against real source
before folding.

You are reading this because gate A already fell through to the fallback branch — `/autoplan` was
absent, or a review skill halted. Do not route back into `/plan-eng-review` or `/plan-design-review`
from here: they are interactive by design and will halt the pipeline again. Run the checklists
below yourself, via a subagent.

## Eng lens (always)

- **Step-0 scope challenge (do this first).** What already exists that does this? What's the
  *minimum* change? Build-vs-reuse — are we rebuilding something already in the tree? Kill scope the
  plan doesn't need. (The single highest-leverage move is often discovering an entrypoint/helper is
  already assembled, collapsing the work.)
- **Architecture & data flow.** Does the design match how the system actually works (per the
  file:line ground truth)? Are boundaries/contracts respected? Any layering or ownership violation?
- **Code quality.** Cohesion, naming that matches surrounding code, no dead abstraction, error
  handling present where it matters.
- **Test-coverage tree.** Every touched file tagged with the tier that will prove it; each new
  behavior has a test; the plan says what the deterministic assertion is (not "add tests").
- **Failure modes.** Enumerate what happens on the unhappy paths — null/empty, timeout, partial
  write, concurrent edit, permission denied, vendor error. Each should have a defined behavior.
- **Performance / cost.** Hot paths, N+1s, and any billable/irreversible action — is it gated behind
  a cheap proof (verify-before-spend)?
- **Invariants preserved.** Discover THIS project's stated invariants (from `CLAUDE.md`/README/tests)
  — e.g. a content-honesty gate, a production identifier that must not change, a seams-not-stubs rule
  — and confirm the plan leaves them intact.

## Design lens (UI/design work)

- **Information architecture.** Is the hierarchy/flow right? Does the change fit the app's navigation
  model, or bolt on a foreign pattern?
- **States.** Empty, loading, error, and success states all designed — empty state as an invitation,
  not an error-shaped dead end. Icon/first-run/permission states covered.
- **AI-slop blacklist.** Kill generic marks and clichés (default sparkles/orbs, centered-everything,
  purple gradients, lorem cards). The direction should be ownable, grounded in the Mobbin reference
  set and the project design source.
- **Motion.** Real tokens (durations/easings), a `prefers-reduced-motion` path, no gratuitous
  animation. Transitions serve comprehension.
- **Accessibility.** Contrast meets AA against the actual background, hit targets ≥ the platform
  minimum, labels/roles present, dynamic-type/scaling survives.
- **Consistency with the design source.** Color, type, spacing, and shape tokens match the project's
  `design.md`/token file. If code and a stale doc disagree, code wins — flag the doc.

## Output

Return findings ranked most-severe-first, each with a concrete `proposedFix`, plus a one-line verdict
(`CLEAR` / `NEEDS-EDITS`). Fold edits into the plan; re-run the lens if anything material changed.
