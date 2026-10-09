<div align="center">

```text
█▀▀ █ █ █   █       ▄▀▀ █▀▀ █▄ █ █▀▄
█▀  █ █ █   █   ▀▀▀  ▀▄ █▀  █ ▀█ █ █
▀   ▀▀▀ ▀▀▀ ▀▀▀     ▀▀  ▀▀▀ ▀  ▀ ▀▀ 
```

<kbd><b>version</b> 1.5.0</kbd> [<kbd><b>license</b> MIT</kbd>](LICENSE) [<kbd><b>agent</b> Claude Code</kbd>](https://claude.com/claude-code)

</div>

**full-send** is a Claude Code skill that runs a delivery pipeline from investigation to proof, and it does not stop for permission at each phase.

A change can look done and not be done. So full-send trusts no claim of "done" until a deterministic check proves it against real, observed state. The model does not write that verdict.

## Install

full-send needs [Claude Code](https://claude.com/claude-code) and no other skill. Run these commands from the folder that holds your clones.

```bash
# link (recommended): the installed skill follows your clone
git clone https://github.com/klee-repos/full-send-skill   # skip if already cloned
mkdir -p ~/.claude/skills
rm -rf ~/.claude/skills/full-send    # replace ANY prior install: symlink OR copied directory
ln -s "$(pwd)/full-send-skill/skill/full-send" ~/.claude/skills/full-send
test -f ~/.claude/skills/full-send/SKILL.md \
  && echo "full-send installed" \
  || echo "BROKEN LINK — re-run from the directory that CONTAINS full-send-skill/"

# copy: a static copy that does not follow your clone
git clone https://github.com/klee-repos/full-send-skill
mkdir -p ~/.claude/skills
rm -rf ~/.claude/skills/full-send   # replace any prior install, don't nest inside it
cp -R full-send-skill/skill/full-send ~/.claude/skills/full-send
```

> **Keep the `rm -rf` line.** `ln -sf` cannot replace a directory. Over an old copy, it nests a link inside the copy, and you keep the old skill with no error.

With the link, run `git pull` in the clone to update the skill. If you move the clone, make the link again.

## Use

Ask for the pipeline in a Claude Code session. Name it, or name the phases you want.

```text
full-send this bugfix                                       # the whole pipeline, by name
plan, adversarial review, build, e2e, loop until verified   # the same pipeline, by its phases
RCA this bug then fix it with proof                         # a bug: the root cause first
validate with agentic tests like a real user                # the end-to-end proof
```

The skill first restates your request as a phase contract and records each hard constraint in your words. It creates a run directory with a `LEDGER.md` before it starts an agent. Then it runs every phase. It asks you about a real fork of scope or architecture. It also asks before an action that other people can see, such as a push or a deploy.

## What it does

- **Ground truth**: for a bug, the skill does a formal RCA from real data before it reads source. For a feature, read-only agents map the code and cite `file:line`.
- **Plan**: the plan goes to a scratchpad, never to the repo. It has numbered fixes and a test-coverage tree.
- **Plan review**: up to four lenses review the plan: scope, design, developer experience and engineering. Each lens gets two independent reviewers that read one frozen copy of the plan.
- **Adversarial review**: single-lens attackers cite `file:line`. A fresh verifier for each finding starts at REFUTED. A synthesis gives GO, GO-WITH-PATCHES or NO-GO.
- **Implement**: the main thread writes the code. Agents investigate and review, and they do not write the implementation.
- **End-to-end proof**: a deterministic assertion runs against the real artifact: a CLI binary, a live endpoint, or real clicks on real screens.
- **Loop**: the skill debugs from real signal and runs the adversarial review again, until a stable green with proof.
- **Close out**: you get a handoff pack, and every number in the closing report comes from disk.

Two rules hold in every phase that uses agents. Shared state lives in files: each agent reads the ledger first and writes its full output to its own file. The run supervises each agent until it can account for that agent, because a dead agent returns nothing, and nothing reads as clean.

The plan review needs no other skill. The run decides every finding by six fixed principles and logs each decision. It never decides a change to the direction you gave, so that question goes to you.

Every string the pipeline writes follows [ASD-STE100](https://www.asd-ste100.org/) Simplified Technical English. Exact terms stay verbatim: identifiers, file names, error strings and each observed value.

For UI work, the skill uses the Mobbin MCP for design references when you connect it. Without it, the skill uses the screenshots you give it and the design source of the project.

## Docs

`skill/full-send/SKILL.md` holds the pipeline. Each file in `references` holds one phase in detail.

```text
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

MIT. See [LICENSE](LICENSE). [NOTICE](NOTICE) names the prior work that the plan review builds on.
