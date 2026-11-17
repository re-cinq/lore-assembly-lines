# Lore assembly lines

This repository is a Lore pack. Lore reads it and applies it on top of the agents and assembly lines it ships, so you can change what an agent is told, or add agents and assembly lines of your own, without building a modified Lore image.

Right now the pack changes nothing. Every agent and assembly line runs exactly as Lore ships it, and keeps getting Lore's improvements as new versions come out.

## What is in here

```
defaults/
  prompts/<agent>.md        the prompt and settings of every agent Lore ships
  pipelines/<line>.yaml     every assembly line Lore ships
.github/workflows/pack-lint.yml
```

The `defaults` folder is a copy of Lore v1.0.2, kept for reading. Lore ignores it. Look up an agent's current prompt there before you decide how to change it. It does not update itself, so copy it again from `libs/shared/src/agent-defaults/` and `libs/assembly-lines/src/floor-pipelines/` in the Lore repository when you move to a new release.

Lore only reads files at these paths:

```
<line>/pipeline.yaml              a new assembly line
<line>/agents/<agent>.md          a full agent definition
<line>/agents/<agent>.extra.md    text added to the end of a shipped agent's prompt
```

Anything else, this README included, is ignored. Do not create a top-level folder whose second level is `agents/` unless you mean it, because Lore applies every `<folder>/agents/*.md` it finds.

## Changing a shipped agent

There are two ways, and they behave differently when Lore updates.

To add instructions, write an `.extra.md` file. Lore appends its text to the shipped prompt after a blank line. The shipped prompt below your text still updates with each Lore release. This is the right choice almost every time.

```
code-review/agents/code-review.extra.md
```

```markdown
Flag any new SQL query that is built by string concatenation.
```

To take over an agent completely, write `<agent>.md` with the same name as the shipped agent. Start from the copy in `defaults/prompts/`. From then on you own that prompt, and Lore's later changes to it no longer reach you.

The folder an agent file sits in only groups it. An agent can be given once in the whole pack, either as `.md` or as `.extra.md`, not both.

These are the agents each shipped assembly line uses:

| Assembly line | Agents |
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

`auto-merge`, `merge` and `lore-run-settled` run no agents.

## Adding an agent or an assembly line

A new agent is an `<agent>.md` file with a name Lore does not ship. Its frontmatter holds the settings and its body is the prompt. `image`, `timeout_minutes` and the prompt are required.

```markdown
---
image: ghcr.io/acme/agent:2
timeout_minutes: 10
model: claude-sonnet-4-6
---
Work the ticket. Follow the Acme naming rules.
```

A setting may name an environment variable of the Lore install as `${NAME}`. Lore fills it in when it applies the pack. It never fills variables into a prompt.

A new assembly line is `<line>/pipeline.yaml`, in the same format as the files in `defaults/pipelines/`. Its `line.id` must equal the folder name, and every agent its stations name must be shipped by Lore or defined in this pack. You cannot replace a shipped assembly line. A folder with the same name as one is only a place to put agent files.

## Connecting the pack to Lore

Set the repository on both lore-api and the stations service in the umbrella chart's values:

```yaml
lore-api:
  env:
    LORE_PACK_REPO: <owner>/lore-assembly-lines
    LORE_PACK_REF: main
lore-stations:
  env:
    LORE_PACK_REPO: <owner>/lore-assembly-lines
    LORE_PACK_REF: main
```

The Lore GitHub App must be installed on this repository with read access to its contents.

Lore applies the pack when lore-api starts, when a push reaches the configured branch, every 15 minutes, and when an admin presses Sync now on the Pack card of the Settings page. Removing a file undoes it. A removed `.extra.md` or replacement sends the agent back to its shipped prompt, and a removed assembly line is archived on the floor.

An agent the pack supplies cannot be edited as the organisation default in the web UI. Edit its file here instead.

## Checking a change

Lore refuses a bad file on its own and applies the rest. The refusals show on the Pack card. To catch them before merging, run the linter from a Lore checkout:

```
npm run pack:lint -w apps/lore-api -- ~/workspace/lore-assembly-lines
```

It exits 1 and prints each refused file with its reason. The `pack-lint` workflow runs the same check on every pull request. Its `ref` names the Lore release whose checks it runs. Set it to the release your install runs.

The full layout reference is `docs/using-lore/pack-layout-reference.md` in the Lore repository.
