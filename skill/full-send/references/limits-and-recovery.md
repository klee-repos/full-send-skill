# Limits and recovery

This file closes six gaps in the pipeline. Each one lets a run go wrong with no way back. A landed
change that fails has no rollback step. A plan proved wrong gets patched instead of replanned. An
open-ended request has no stop condition. A request built on a false fact is never challenged. The
run does not bound its own cost. No person gets a package they can review.

## Contents
- [Roll back](#roll-back)
- [When the plan proves wrong](#when-the-plan-proves-wrong)
- [An open-ended request](#an-open-ended-request)
- [Push back](#push-back)
- [Cost ceiling](#cost-ceiling)
- [Handoff pack](#handoff-pack)

## Roll back

A landed change that then failed is a rollback, not a patch. You **roll back** in this order. Stop
the implementation. Write the failure signature to the ledger. Restore the start state the ledger
`Workspace` row recorded. Re-run the failed proof, and the proof that passed before it.

**Restore forward, never by discarding.** The whole-run ban holds here: no command that discards
work a person wrote. Restore by a new change that undoes the old one, by a new release of the
earlier artifact, or by an edit you author. When the `Workspace` row says `no undo path`, do not
attempt a restore: write the failure signature and the present state to the ledger, tell the user
what is broken, and stop the implementation there. That stop is the autonomy contract's
bounded-stop yield.

Check the order of effects before you restore a running system: an earlier artifact put back over a
landed data change can be worse than the failure. The rollback took when the failure signature is
gone and the earlier proof is green. Read both from a channel the rollback did not write. Then
re-enter Step 1. An effect you cannot undo is **not reversible**: name it with its remediation.

## When the plan proves wrong

**The plan proves wrong** when a fact you observe in Step 6 makes a plan step impossible, or puts
the goal out of reach. A hard step is not a wrong plan. Two invalid steps in one round is.

Then stop the implementation. Write the fact and its `file:line` to the ledger. Re-enter Step 3.
Re-run gate A **and gate B** on the changed sections: implementation code waits for the
`SCOPE+ENG[+DESIGN][+DX]+ADVERSARIAL CLEARED` mark, the same as on the first pass. Gate A names
the lens that owns each changed section (`plan-review.md` § A re-run). Patching on is wrong,
because the gates approved the old plan and later proof would cover an unreviewed one.

**The return never yields the turn**, and it is bounded. Return to Step 3 at most twice in one run,
and write each return to the ledger `Cost ceiling` row, which already holds that field, because a
count you keep only in context does not survive a compaction.
On the third, stop and report the fact that invalidated the plan, with its `file:line`, and the
approach you would take instead.

## An open-ended request

An **open-ended** request states a direction and no end, so it holds no stop condition. At Step 0,
draft the concrete changes it covers to the ledger as the stop set, and into your Step 0 contract.
**Freeze the set after Step 1**, when ground truth is in. Step 0 is the moment you know least, so a
set frozen there certifies the narrowest reading of the request. Widen it out loud when Step 1 shows
an item you missed.

The run is done when every item is green-with-proof. This is not a round counter: each item still
ends on a stable re-run. The frozen set bounds NEW work only. A defect on a surface this run touched
is never a later item; it is work inside the current item, and Step 9's rule applies. A later item
that asks for new behaviour goes to Step 9 as out of scope.

## Push back

**Push back** only when Step 1 ground truth shows the request rests on a false fact: the defect is
elsewhere, the behavior already exists, or the change makes a stated constraint false. An opinion
about the approach is not a false fact. Otherwise take the default and proceed.

Say it once: the false fact with its `file:line`, what is true, and the work you do instead. Then
start it. Push-back never buys a skipped proof, and never ends the run.

## Cost ceiling

At Step 0 write a **cost ceiling** to the ledger: a maximum agent count PER WAVE, a maximum wave
count, and a wall-clock limit. Use the user's numbers. Else set it from the change: one surface and
one file, 8 agents per wave, 7 waves, 90 minutes; a stated feature or bug, 12 agents per wave, 12
waves, 4 hours. A **wave** is one set of agents you spawn in one message. The lower figure buys one
mapping wave, gate A in one wave, one gate B round of three waves, and two waves for a retry or a
second eng read. The higher figure buys one mapping wave, gate A in up to three waves, two gate B
rounds, and two retry waves. Raise it only when a
phase you already ran shows you must. The wall-clock limit is a reporting mark, not a stop: when the run crosses it, write the
elapsed time and the remaining items to the ledger `Open` row, and go on.

**The ceiling never ends the run.** It bounds fan-out width, not the loop. Check it before each
fan-out. When the next wave crosses it, cut that wave to the highest-stakes lenses and name the
dropped ones in `Open` as uncovered. Then go on. Two cuts it may never make: a lens the catalog
entry states a fire condition this change meets, and a subsystem the change touches. A ceiling that cuts only optional lenses never yields the turn: report what stays uncovered, name
the ceiling, and go on with the highest-stakes work inside it. A ceiling that cannot cover one of
those two — a lens whose fire condition this change meets, or a subsystem the change touches — is
the thing you report, not a smaller plan, and it is the one case where the ceiling yields the turn.
The autonomy contract lists that yield.

## Handoff pack

At Step 9 write a **Handoff pack** to `<run>/HANDOFF.md`, and name that path in the closing report.
It holds the request as stated, the change list with a reason each, the proof record path and its
verdict line, what stays uncovered, the not-reversible effects with their rollback step, and the
reviewer's open questions. One page, for a reader who did not watch the run.

Gate A supplies the open questions. Copy them from the plan's `## Review record` and its
`NOT in scope` list: each `taste` decision with its rejected option, each deferred item with its
reason, each push-back, and each `direction` change the run did not ask.
