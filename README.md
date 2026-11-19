# Lore assembly lines

This repository is a Lore pack. Your team edits its assembly lines and agent prompts in Git. When a change merges to the configured branch, Lore syncs the pack and Floor runs the new definitions. [Floor workflow engine](https://github.com/re-cinq/floor)

## Define your first line

1. Decide what happens when Product approves a launch, puts it on hold, or hasn't answered yet. Write the rule in YAML and check it against an example. [Define your first assembly line](https://github.com/re-cinq/floor/blob/main/docs/define-a-launch-approval.md)
2. Change the rule so Communications also has to sign off. Trace what happens when the teams disagree. [Add a second sign-off](https://github.com/re-cinq/floor/blob/main/docs/add-a-launch-review.md)

To try the example in this pack, save it as `launch-approval/pipeline.yaml` and open a pull request. After it merges, the Pack card in Lore Settings shows the commit the install applied and any files it refused. For a first install, compare the lines you want to keep here with your Lore release. A full pipeline file replaces the shipped line. [Connect the pack to Lore](docs/setup.md)

## Layout

One folder per assembly line. The folder name is the line's name.

```
code-review/
  pipeline.yaml               the line: its steps and how they connect
  agents/code-review.md       the agent a step runs: settings, then the prompt
implementation-loop/
  pipeline.yaml
  agents/loop-dod.md
  agents/loop-tdd-round.md
  ...
docs/setup.md                 not read by Lore
.github/workflows/pack-lint.yml
```

Lore reads three kinds of path inside a line folder:

| Path | What it is |
| --- | --- |
| `<line>/pipeline.yaml` | The line. Its `line.id` must equal the folder name. |
| `<line>/agents/<agent>.md` | An agent: settings as frontmatter, the prompt as the body. |
| `<line>/agents/<agent>.extra.md` | Text added to the end of the prompt Lore ships for `<agent>`. Only for an agent this pack does not define in full. |

Any other markdown or YAML file inside a line folder is refused, so a misspelt name shows up on the Pack card instead of quietly doing nothing. Files at the top of the repository, under `docs/` and under `.github/` are ignored.

These are the lines and the agents each one uses:

| Line | Agents |
| --- | --- |
| code-review | code-review |
| code-review-recheck | code-review-recheck |
| code-review-reply | code-review-refine |
| daily-digest | digest-refine |
| feature-planning | plan-analyze, plan-sketch, plan-validate, spec-analysis, spec-draft, spec-qa-answer, spec-qa-generate, spec-qa-recheck, spec-rework, spec-write |
| implementation-loop | loop-dod, loop-fix-ci, loop-sync, loop-tdd-round |
| issue-triage | triage-assess-implementation, triage-decompose, triage-diagnose, triage-reproduce, triage-verify |
| onboard | onboard-author, onboard-fix-ci |
| spec-tickets | spec-tickets-decompose |
| spec-upkeep | spec-upkeep-fix-ci, spec-upkeep-update-specs |
| weekly-changelog | changelog-intro |
| auto-merge, merge, lore-run-settled | none, every step is Lore's own code |

An agent is defined once in the whole pack. A line can use an agent defined in another line's folder.

## What runs

Everything in this pack runs as written. Lore does not merge its own newer prompts or lines into yours. When your install moves to a new Lore release, copy that release's files in (see the setup guide) and read the diff before merging.

Deleting a file hands that part back to Lore. Delete an agent file and the agent runs Lore's shipped prompt again. Delete a line's `pipeline.yaml` and Lore's own line goes back on the floor. Delete a folder for a line you added yourself and the line is archived.

An agent the pack defines cannot be edited as the organisation default in the web UI. Edit its file here.

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

A new line is a new folder with a `pipeline.yaml`, in the same format as the others here. A step that runs your own code is a `kind: service` station served by a process you deploy. The [pack layout reference](https://github.com/re-cinq/lore/blob/main/docs/using-lore/pack-layout-reference.md) covers it.

## Checking a change

Every pull request runs `pack-lint`, which applies the same checks Lore does: file paths, frontmatter keys and required settings, each line's structure, and that every agent a line names exists. To run it yourself from a Lore checkout:

```
npm run pack:lint -w apps/lore-api -- ~/workspace/lore-assembly-lines
```

It prints each refused file with its reason and exits 1, or prints what the pack supplies and exits 0.
