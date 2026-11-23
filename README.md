# Lore assembly lines

This repository helps start other Lore installs with default flows of their own. It holds the lines, agent prompts and skills that [Lore](https://github.com/re-cinq/lore) ships by default.

Its content is updated in Lore and written here on every merge to Lore's main. A release of Lore tags this repository with the same version. A change is proposed in Lore, never here: a pull request on this repository is overwritten by the next sync.

To use it, create your own repository from this template, edit that, and point your install at it with `LORE_PACK_REPO`. [Connect the pack to Lore](docs/setup.md)

Your team then edits its assembly lines and agent prompts in Git. When a change merges to the configured branch, Lore syncs the pack and the [Floor workflow engine](https://github.com/re-cinq/floor) runs the new definitions.

## Define your first line

1. Decide what happens when Product approves a launch, puts it on hold, or hasn't answered yet. Write the rule in YAML and check it against an example. [Define your first assembly line](https://github.com/re-cinq/floor/blob/main/docs/define-a-launch-approval.md)
2. Change the rule so Communications also has to sign off. Trace what happens when the teams disagree. [Add a second sign-off](https://github.com/re-cinq/floor/blob/main/docs/add-a-launch-review.md)

To try the example in your copy, save it as `launch-approval/assembly-line.yaml` and open a pull request. After it merges, the Pack card in Lore Settings shows the commit the install applied and any files it refused. A full pipeline file replaces the shipped line.

## Layout

One folder per assembly line. The folder name is the line's name.

```
code-review/
  assembly-line.yaml          the line: its steps and how they connect
  agents/code-review-product.md       the agent a step runs: settings, then the prompt
implementation-loop/
  assembly-line.yaml
  agents/loop-dod.md
  ...
skills/tdd-start.md           a catalogue skill
agent-skills/                 a reference copy, not read by Lore from a pack
docs/setup.md                 not read by Lore
.github/workflows/pack-lint.yml
```

Lore reads these paths:

| Path | What it is |
| --- | --- |
| `<line>/assembly-line.yaml` | The line. Its `line.id` must equal the folder name. Lore still reads the old name `pipeline.yaml` in a pack written before, but a folder may not hold both. |
| `<line>/agents/<agent>.md` | An agent: settings as frontmatter, the prompt as the body. |
| `<line>/agents/<agent>.extra.md` | Text added to the end of the prompt Lore ships for `<agent>`. Only for an agent this pack does not define in full. |
| `skills/<name>.md` | A catalogue skill. It replaces Lore's skill of that name, or adds one. It is read-only in the web UI while the pack is set. |
| `agent-skills/` | A reference copy of the skills and hook bundles baked into Lore's gateway image. Lore does not read it from a pack. |

Any other markdown or YAML file inside a line folder is refused, so a misspelt name shows up on the Pack card instead of quietly doing nothing. Files at the top of the repository, under `docs/` and under `.github/` are ignored.

These are the lines and the agents each one uses:

| Line | Agents |
| --- | --- |
| auto-merge | none, every step is Lore's own code |
| code-review | review-story, review-implementation, review-issue, review-docs, code-review |
| code-review-recheck | review-story, review-implementation, review-issue, review-docs, code-review-recheck |
| code-review-reply | code-review-refine |
| daily-digest | digest-refine |
| feature-planning | plan-analyze, plan-sketch, plan-validate, spec-analysis, spec-write, spec-rework, spec-draft, spec-qa-generate, spec-qa-answer, spec-qa-recheck |
| implementation-loop | loop-sync, loop-dod, loop-tdd-round, loop-test-review, loop-fix-ci |
| issue-triage | triage-assess-implementation, triage-verify, triage-reproduce, triage-diagnose, triage-decompose |
| lore-run-settled | none, every step is Lore's own code |
| merge | none, every step is Lore's own code |
| onboard | onboard-author, onboard-fix-ci |
| release | release-notes |
| spec-tickets | spec-tickets-decompose |
| spec-upkeep | spec-upkeep-update-specs, spec-upkeep-fix-ci |
| weekly-changelog | changelog-intro |

An agent is defined once in the whole pack, in the folder of the first line that names it. A line can use an agent defined in another line's folder.

## What runs

Everything in a pack runs as written. Lore does not merge its own newer prompts, skills or lines into yours. A pack used as is freezes every agent and skill it holds: newer prompts from a Lore release do not flow in until you copy them. The tag of this repository that matches your Lore release is the place to copy from.

Deleting a file hands that part back to Lore. Delete an agent file and the agent runs Lore's shipped prompt again. Delete a line's `assembly-line.yaml` and Lore's own line goes back on the floor. Delete a folder for a line you added yourself and the line is archived.

An agent the pack defines cannot be edited as the organisation default in the web UI. Edit its file in your copy.

## Adding your own

A new agent is an `agents/<name>.md` with a name Lore does not ship. `image`, `timeout_minutes` and the prompt are required:

```markdown
---
image: ghcr.io/acme/agent:2
timeout_minutes: 10
model: claude-sonnet-4-6
---
Work the ticket. Follow the Acme naming rules.
```

A setting may name an environment variable of the install as `${NAME}`, which Lore fills in when it applies the pack. The shipped agents use `${LORE_AGENT_IMAGE}` this way. Variables are never filled into a prompt.

A new line is a new folder with an `assembly-line.yaml`, in the same format as the others here. A step that runs your own code is a `kind: service` station served by a process you deploy. The [pack layout reference](https://github.com/re-cinq/lore/blob/main/docs/using-lore/pack-layout-reference.md) covers it.

## Checking a change

Every pull request on your copy runs `pack-lint`, which applies the same checks Lore does: file paths, frontmatter keys and required settings, each line's structure, and that every agent a line names exists. To run it yourself from a Lore checkout:

```
npm run pack:lint -w apps/lore-api -- ~/workspace/lore-assembly-lines
```

It prints each refused file with its reason and exits 1, or prints what the pack supplies and exits 0.
