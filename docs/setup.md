# Setting up and configuring a pack

This guide connects a pack repository to a Lore install and walks through a first change. The full set of rules is in the [pack layout reference](https://github.com/re-cinq/lore/blob/main/docs/using-lore/pack-layout-reference.md). Packs need Lore v1.0.3 or later.

## 1. Make your own copy

Create a repository from this one with GitHub's "Use this template" button, in the organisation your Lore install works with. It can be private. Lore reads it through its GitHub App, not over public access.

One install reads one pack, so one repository serves the whole organisation.

The copy holds what the Lore release in its README ships. If your install runs a newer release, refresh the copy first (see [Upgrading Lore](#upgrading-lore)), or connecting it would put the older prompts and lines back.

## 2. Let Lore read it

Install the Lore GitHub App on the new repository, with read access to its contents. If the App is installed on all of the organisation's repositories, there is nothing to do.

A repository the App cannot see is recorded as an unreachable pack, and the floor keeps whatever it held.

## 3. Point Lore at the repository

Set the repository on lore-api and on the stations service. lore-api applies the pack, and the stations service turns a push to the repository into a sync, so both need it. `LORE_PACK_REF` is the branch or tag to read; leave it empty to read the default branch.

On an install made with the `lore-install` chart, add to your values file:

```yaml
lore-platform:
  lore-api:
    env:
      LORE_PACK_REPO: acme/lore-assembly-lines
      LORE_PACK_REF: main
  lore-stations:
    env:
      LORE_PACK_REPO: acme/lore-assembly-lines
      LORE_PACK_REF: main
```

and run the same `helm upgrade --install` you installed with. On a cluster deployed from the `lore-platform` umbrella chart directly, the same keys go under `lore-api.env` and `lore-stations.env`.

## 4. Check the first sync

lore-api applies the pack when it starts. Open Settings in the Lore web UI and find the Pack card. It shows the repository, the commit it applied, what changed and any refused files. A fresh copy of the same release shows "Applied" and lists the agents and lines, with nothing refused.

The same state is at `GET /api/pack` with any read token:

```
curl -s -H "Authorization: Bearer $LORE_TOKEN" https://<lore-api host>/api/pack
```

`configured: false` means lore-api did not get `LORE_PACK_REPO`. A sync with an error means Lore could not read the repository; check the App installation from step 2.

## 5. Make a first change

Open `code-review/agents/code-review.md` and add a line to the end of the prompt:

```markdown
Flag any new SQL query that is built by string concatenation.
```

Open a pull request. `pack-lint` checks it the same way Lore will. Merge it. The push reaches Lore through the GitHub webhook and the pack is applied within a minute. If the webhook is lost, a 15-minute tick applies it anyway, and an admin can press Sync now on the Pack card.

From the next review on, the code review agent runs your prompt. The Agents page shows it, without an Edit link, because the pack owns it. A repository's own override of the agent stays editable.

To undo, revert the commit and merge.

## 6. Changing a line

A line's `pipeline.yaml` lists its steps (stations), the start and end, and which outcome of each step leads where. Edit it the same way: the floor gets your version on the next sync, and runs that already started finish on the version they started with.

Be careful with Lore's own steps (the ones with `kind: service`). Lore's code answers them and expects their names, inputs and outputs as shipped. Changing an agent step's prompt or outcomes is safe ground; renaming or removing a service step can stop the line.

A step with the same name in two lines is one step on the floor. If you change a step that another line also uses, keep the two `pipeline.yaml` files in agreement.

## When a file is refused

Lore checks each file on its own, applies what passes, and lists what it refused with the reason on the Pack card. A refused file leaves the floor as it was for that agent or line. The usual causes:

- a misspelt path, such as `pipeline.yml` or `agent/` instead of `agents/`;
- an agent missing `image` or `timeout_minutes`, or with a setting Lore does not know;
- the same agent defined in two folders, or as both `.md` and `.extra.md`;
- a line whose `line.id` is not its folder name, or that names an agent nobody defines.

Fix the file and push again. `pack-lint` catches all of these before a merge.

## Upgrading Lore

When your install moves to a new Lore release:

1. Set `ref` in `.github/workflows/pack-lint.yml` to the new release.
2. On a branch, copy each `libs/assembly-lines/src/floor-pipelines/<line>.yaml` of that release to `<line>/pipeline.yaml`, and each agent file from `libs/shared/src/agent-defaults/` to the folder of the line that uses it.
3. Read the diff. Keep the changes Lore made and put your own edits back where the copy overwrote them. Merge when `pack-lint` passes.
