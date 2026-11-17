# Setting up and configuring a pack

This guide connects a pack repository to a Lore install and walks through the first change. The file layout and every rule Lore checks are in the [pack layout reference](https://github.com/re-cinq/lore/blob/main/docs/using-lore/pack-layout-reference.md). The pack needs a Lore release that carries packs, v1.0.3 or later.

## 1. Make your own copy

Create a repository from this one with GitHub's "Use this template" button, in the organisation your Lore install works with. The copy starts out changing nothing. Keep it private if your prompts are your own business, because Lore reads it through its GitHub App and needs no public access.

One install reads one pack, so one repository serves the whole organisation.

## 2. Let Lore read it

Lore reads the pack through the GitHub App it already uses. Install that App on the new repository, with read access to its contents. If the App is installed on all of the organisation's repositories, there is nothing to do.

A repository the App cannot see is recorded as an unreachable pack, and Lore keeps whatever it applied last.

## 3. Point Lore at the repository

Set the repository on lore-api and on the stations service. lore-api applies the pack, and the stations service turns a push to the repository into a sync, so both need it. `LORE_PACK_REF` is the branch or tag to read. Leave it empty to read the default branch.

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

and run the same `helm upgrade --install` you installed with. On a cluster deployed from the `lore-platform` umbrella chart directly, the same keys sit under `lore-api.env` and `lore-stations.env`.

## 4. Check the first sync

lore-api applies the pack when it starts. Open Settings in the Lore web UI and find the Pack card. It shows the repository, the commit it applied, what changed and any refused files. A fresh copy shows the head commit, "Applied" and nothing changed.

The same state is at `GET /api/pack` with any read token:

```
curl -s -H "Authorization: Bearer $LORE_TOKEN" https://<lore-api host>/api/pack
```

`configured: false` means lore-api did not get `LORE_PACK_REPO`. A sync with an error means Lore could not read the repository. Check the App installation from step 2.

## 5. Make a first change

Add instructions to the code review agent. Create `code-review/agents/code-review.extra.md`:

```markdown
Flag any new SQL query that is built by string concatenation.
```

Open a pull request. The `pack-lint` workflow checks the file the same way Lore will. Merge it. The push reaches Lore through the GitHub webhook and the pack is applied within a minute. If the webhook is lost, the 15-minute tick applies it anyway, and an admin can press Sync now on the Pack card at any time.

The Pack card then lists `code-review` as changed. From the next review on, the agent's prompt is Lore's shipped prompt followed by your text. The agent's Edit link on the Agents page is gone, because the pack now owns its organisation default. A repository's own override of the agent stays editable.

To undo the change, delete the file and merge. The agent goes back to the shipped prompt.

## 6. Bigger changes

To rewrite an agent instead of adding to it, copy its file from `defaults/prompts/` to `<line>/agents/<agent>.md` and edit it. Lore's later changes to that prompt stop reaching you, so prefer an `.extra.md` when you can.

To add an agent or an assembly line of your own, follow the README and the layout reference. A new assembly line can name Lore's agents and your own. A station that needs your own code runs in a service you deploy, as the reference describes under "A custom stations service".

## When a file is refused

Lore checks each file on its own, applies what passes, and lists what it refused with the reason on the Pack card. The usual causes are an `.extra.md` for an agent Lore does not ship, an agent missing `image` or `timeout_minutes`, the same agent given twice, and a `pipeline.yaml` in a folder named like a shipped assembly line. Fix the file and push again. The `pack-lint` workflow catches all of these before a merge.

## Upgrading Lore

When your install moves to a new Lore release, do three things in the pack:

1. Set `ref` in `.github/workflows/pack-lint.yml` to the new release.
2. Copy `libs/shared/src/agent-defaults/*.md` and `libs/assembly-lines/src/floor-pipelines/*.yaml` from that release of Lore over `defaults/prompts/` and `defaults/pipelines/`, so the copies you read match what runs.
3. Read the diff of `defaults/`. A shipped prompt you extend with an `.extra.md` may now say something your text contradicts, and a prompt you replaced may have gained something you want.
