# full-send

A [Claude Code](https://claude.com/claude-code) skill that runs a rigorous delivery pipeline end-to-end and autonomously:

**ground-truth investigation → plan → plan review → adversarial review → implement → agentic end-to-end validation → loop until green-with-proof.**

The point of the whole thing: **no claim of "done" is trusted unless it is proven against real, observed state through a deterministic verdict the LLM did not author.** Every phase exists to close a specific way a change can *look* done but not be done.

## What it does

Given a feature or bug, `full-send` runs (and loops) this pipeline without stopping to ask permission at each phase:

0. **Contract & board** — restate the pipeline, capture every hard constraint verbatim, and create the run directory and `LEDGER.md` before the first agent is spawned.
1. **Ground truth** — for bugs, a formal RCA from real data (real DB rows, real logs, real traces) verified before reading source; for features, parallel read-only mapping agents that cite `file:line`.
2. **Design lane** — for UI work only: reference research, then a cross-check against the project's design source of truth.
3. **Plan** — written to a scratchpad (never the repo tree, never committed), with numbered fix sections and a test-coverage tree.
4. **Review gate A: the plan review** — built in, with no other skill. Up to four lenses review the plan in order: scope (always), design (UI work), developer experience (developer-facing work), then engineering (always, last). Each lens gets two independent reviewers that read one frozen copy of the plan. The run decides every finding by six fixed principles and logs each decision. It never decides a change to the direction the user gave: that one goes to the user.
5. **Review gate B: the adversarial review** (the signature move) — N single-lens attackers cite `file:line` → per-finding fresh verifiers that **default to REFUTED** → synthesize survivors into plan edits and a GO / GO-WITH-PATCHES / NO-GO verdict. Asymmetric skepticism is the anti-rubber-stamp mechanism.
6. **Implement** — direct, single-threaded, in the main thread. Fan-out is for investigation and review, not for writing code.
7. **Agentic end-to-end validation** — a deterministic assertion against the REAL running artifact (CLI binary, live endpoint, real user clicks on real screens). The LLM never decides pass/fail.
8. **Loop** — debug from real signal, re-run the adversarial pass, prove red/green on every new test, until stable green-with-proof against current state.
9. **Close out** — a handoff pack for the human reviewer, the tracked work item refreshed from the final diff, and every number in the closing report re-derived from disk.

Two invariants hold across every phase that fans work out to agents:

- **Shared state lives in files, not in context.** A run directory with a `LEDGER.md` is created before the first agent is spawned. Every agent reads the ledger first and writes its full output to its own file, returning only a path + `status` + a short receipt. Context is a cache of those files, never the system of record — it's what a compaction takes, and what a subagent never had. This is what keeps N fanned-out agents working from one picture of reality instead of N drifting ones.
- **Every spawned agent is supervised until it's accounted for.** Agents are recorded in a manifest at spawn, heartbeat to a `.progress` file, and are *killed* — not waited on — when they stop progressing, then re-spawned once narrower before the orchestrator covers the slice itself. A wave closes only when every member is `done`, covered, or named `uncovered`. Dead and stalled agents are the quietest failure mode here: they return empty, and empty reads as clean. The same guard applies to the macro loop — a round that produces no new observed signal is a stall, not an iteration, and it ends in an honest unproven report rather than another round.

## Writing standard

Every human-readable string the pipeline emits is written in [ASD-STE100](https://www.asd-ste100.org/) Simplified Technical English: the ledger, the RCA, the plan, agent findings, the proof record, **code comments and docstrings**, log and error strings, commit messages, and PR bodies.

Short sentences, active voice with the actor named, one word for one meaning, no idiom or metaphor. A long autonomous run emits a lot of prose, and prose nobody can read quickly is prose nobody reads.

Exact terms are exempt, and they outrank the style — identifiers, API and file names, error strings, `file:line` cites, and every quoted observed value stay verbatim. Simplifying an observed value would turn a proof into a paraphrase of one, so the proof rules win. The standard reaches subagents through the ledger's `Style` entry, because a subagent inherits no context.

## Install

This repo packages the skill under `skill/full-send/`. **Recommended: install via symlink** so the live skill stays in sync with the repo — edit in the repo, `git push`, and your installed copy updates automatically (no re-copy step, no drift):

```bash
# clone + symlink (recommended — single source of truth)
git clone https://github.com/klee-repos/full-send-skill   # skip if already cloned
mkdir -p ~/.claude/skills
rm -rf ~/.claude/skills/full-send    # replace ANY prior install: symlink OR copied directory
ln -s "$(pwd)/full-send-skill/skill/full-send" ~/.claude/skills/full-send
test -f ~/.claude/skills/full-send/SKILL.md \
  && echo "full-send installed" \
  || echo "BROKEN LINK — re-run from the directory that CONTAINS full-send-skill/"
```

The `rm -rf` matters: `ln -sf` alone cannot unlink a *directory*, so over a prior copy-install it
exits 0, nests a link inside it, and leaves you loading the stale skill forever with no symptom. The
`test` line catches the other silent failure — `ln -s` also exits 0 against a wrong source path,
leaving a dangling link. (`rm -rf` on a symlink removes only the link; your clone is untouched.)

If you prefer a static copy (no auto-sync), copy instead:

```bash
git clone https://github.com/klee-repos/full-send-skill
mkdir -p ~/.claude/skills
rm -rf ~/.claude/skills/full-send   # replace any prior install, don't nest inside it
cp -R full-send-skill/skill/full-send ~/.claude/skills/full-send
```

> **Note on the symlink approach:** the live skill `~/.claude/skills/full-send` becomes a pointer to your local clone. To update it, just `git pull` in the clone — the installed skill sees the new files immediately. If you ever move the clone, recreate the link: `rm -rf ~/.claude/skills/full-send && ln -s <newpath>/skill/full-send ~/.claude/skills/full-send`.

Then invoke it from a Claude Code session: ask for the pipeline by name (e.g. *"full-send this bugfix"*, *"plan → adversarial review → build → e2e, loop until verified"*).

## Dependencies

`full-send` needs no other skill. Every phase, the plan review included, is in this repository.

One integration is optional:

- **Mobbin MCP** (`mcp__mobbin__search_screens` / `search_flows`) — design references for UI work. When it is absent, the skill uses the screenshots you give it and the design source of the project.

**No QA skill sits on the E2E driver ladder, by design.** A QA skill that autonomously *fixes* what it finds takes ownership of a diff that is yours, and one that requires a clean git tree cannot run at Step 7 at all — Step 7 is reached with the whole implementation uncommitted. `full-send` drives the real artifact itself, or authors a harness out-of-tree.

See the "Graceful degradation" section of `skill/full-send/SKILL.md` for the fallback rules.

## Repo layout

```
skill/full-send/
├── SKILL.md                              # the skill (frontmatter + pipeline)
└── references/
    ├── adversarial-review.md             # Attack→Verify→Synthesize template, lens catalog, schemas
    ├── agentic-e2e.md                    # deterministic-verdict principle, harness discovery, tier table
    ├── investigation-and-plan.md         # read-only ground-truth fan-out template + plan skeleton
    ├── limits-and-recovery.md            # rollback, plan invalidation, open-ended scope, push-back, cost ceiling, handoff
    ├── plan-review.md                    # Gate A: lenses, passes, reviewer prompts, decision rules
    ├── plan-lens-scope.md                # Gate A lens: premise, alternatives, scope, failure and rollout
    ├── plan-lens-eng.md                  # Gate A lens: architecture, code quality, tests, performance
    ├── plan-lens-design.md               # Gate A lens: hierarchy, states, journey, generic-design checks
    ├── plan-lens-dx.md                   # Gate A lens: time to a first result, interface, errors, docs
    └── run-ledger-and-supervision.md     # file-backed shared state + agent liveness/stall handling
```

## License

MIT. See [LICENSE](LICENSE).

[NOTICE](NOTICE) names the prior work that the plan review builds on.
