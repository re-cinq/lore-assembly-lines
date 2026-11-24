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
floor-collection.yaml         the keys of an agent file that are Lore's own
docs/setup.md                 not read by Lore
.github/workflows/pack-lint.yml
```

The line and agent files follow floor's collection format: see the [agent file reference](https://re-cinq.github.io/floor/specs/agent-file/). `floor-collection.yaml` tells floor that `execution_mode` and `review_required` are Lore's own keys. Lore adds two folders floor does not read:

| Path | What it is |
| --- | --- |
| `skills/<name>.md` | A catalogue skill. It replaces Lore's skill of that name, or adds one. It is read-only in the web UI while the pack is set. |
| `agent-skills/` | A reference copy of the skills and hook bundles baked into Lore's gateway image. Lore does not read it from a pack. |

Any other markdown or YAML file inside a line folder is refused, so a misspelt name shows up on the Pack card instead of quietly doing nothing. Files at the top of the repository, under `docs/` and under `.github/` are ignored.

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

## Seeing the lines

To see every line, its steps and the agents each one uses, run this in the repository root:

```
npx @re-cinq/floor-studio
```

It shows the lines in a browser, with each file it would refuse and why, and updates when you save a file, so you see a change before you open a pull request.

## Checking a change

Every pull request on your copy runs `pack-lint`, which applies the same checks Lore does: file paths, frontmatter keys and required settings, each line's structure, and that every agent a line names exists. To run it yourself from a Lore checkout:

```
npm run pack:lint -w apps/lore-api -- ~/workspace/lore-assembly-lines
```

It prints each refused file with its reason and exits 1, or prints what the pack supplies and exits 0.
