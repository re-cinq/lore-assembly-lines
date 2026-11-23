# Setting up and configuring a pack

This guide connects a pack repository to a Lore install and walks through a first change. The full set of rules is in the [pack layout reference](https://github.com/re-cinq/lore/blob/main/docs/using-lore/pack-layout-reference.md). Packs need Lore v1.0.3 or later.

## 1. Make your own copy

Create a repository from this one with GitHub's "Use this template" button, in the organisation your Lore install works with. It can be private. Lore reads it through its GitHub App, not over public access.

One install reads one pack, so one repository serves the whole organisation.

The copy holds what the latest Lore sync wrote here. If your install runs an older release, compare the copy with the tag of your Lore release (see [Upgrading Lore](#upgrading-lore)), or connecting it would put newer prompts and lines in front of your install.

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

Open `code-review/agents/code-review-product.md`. Add this instruction before the output format in the product reviewer’s prompt.

```markdown
When a change affects exports, check whether existing customers can still read the exported file using the documented format.
```

Open a pull request. `pack-lint` checks it the same way Lore will. Merge it. The push reaches Lore through the GitHub webhook and the pack is applied within a minute. If the webhook is lost, a 15-minute tick applies it anyway, and an admin can press Sync now on the Pack card.

The product pass uses your prompt on the next review or recheck. The Agents page shows it, without an Edit link, because the pack owns it. A repository's own override of the agent stays editable.

To undo, revert the commit and merge.

## 6. Changing a line

A line's `assembly-line.yaml` lists its steps (stations), the start and end, and which outcome of each step leads where. Edit it the same way: the floor gets your version on the next sync, and runs that already started finish on the version they started with.

Be careful with Lore's own steps (the ones with `kind: service`). Lore's code answers them and expects their names, inputs and outputs as shipped. Changing an agent step's prompt or outcomes is safe ground; renaming or removing a service step can stop the line.

A step with the same name in two lines is one step on the floor. If you change a step that another line also uses, keep the two `assembly-line.yaml` files in agreement.

## When a file is refused

Lore checks each file on its own, applies what passes, and lists what it refused with the reason on the Pack card. A refused file leaves the floor as it was for that agent or line. The usual causes:

- a misspelt path, such as `pipeline.yml` or `agent/` instead of `agents/`;
- an agent missing `image` or `timeout_minutes`, or with a setting Lore does not know;
- the same agent defined in two folders, or as both `.md` and `.extra.md`;
- a line whose `line.id` is not its folder name, or that names an agent nobody defines.

Fix the file and push again. `pack-lint` catches all of these before a merge.

## Upgrading Lore

A pack used as is freezes every agent and skill it holds. When your install moves to a new Lore release:

1. Set `ref` in `.github/workflows/pack-lint.yml` to the new release.
2. Compare your copy with the tag of this repository that matches the release, for example `git diff <your base> v1.4.0` after adding this repository as a remote. The tag holds exactly what that Lore release ships.
3. On a branch, bring over the lines, agents and skills you want. Keep your own edits where the new files would overwrite them. Merge when `pack-lint` passes.
