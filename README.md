# examples-github-action-dynamic-subtree-sync-agent-skills

A GitHub Action that pulls a **specific skill subdirectory** from a source
registry repo (**RepoA**) into a target path in the repo where the workflow runs
(**RepoB**), using `git subtree`.

Standard `git subtree pull` grafts the *root* of a remote. To extract one nested
subdirectory and nothing else, this action does a three-step orchestration:

1. **Fetch** the source branch into RepoB's `.git` (no working-tree merge).
2. **Split** — `git subtree split` replays only the commits touching
   `source_path` into a synthetic temporary branch containing just that
   directory's history.
3. **Inject** — `git subtree add` (first import) or `git subtree merge`
   (subsequent syncs) grafts that branch under `target_path`, then pushes.

## Files

| File | Lives in | Purpose |
|------|----------|---------|
| [`.github/workflows/sync-skill-subtree.yml`](.github/workflows/sync-skill-subtree.yml) | **RepoB** (this repo) | The sync action. |
| [`examples/source-repo-workflow/notify-downstream.yml`](examples/source-repo-workflow/notify-downstream.yml) | **RepoA** (source) | Optional: fires a webhook so RepoB auto-syncs on change. |

## Running it

**Manually** — Actions tab → *Sync Agent Skill (subtree)* → *Run workflow*, then fill in:

- `source_path` — path in RepoA, e.g. `registry/core/communication`
- `target_path` — where it lands in RepoB, e.g. `agents/skills/communication`
- `source_branch` — branch of RepoA to pull (default `main`)
- `source_repo` — optional override of the default source (`SOURCE_REPO_URL`)

**Automatically** — install `notify-downstream.yml` in RepoA. On a push that
touches `**/*.md`, it sends a `repository_dispatch` (`skills_updated`) whose
`client_payload` carries the paths/branch, and RepoB syncs without manual input.

## Setup (required)

1. **`SOURCE_REPO_URL`** — edit the `env:` block in `sync-skill-subtree.yml` to
   point at your registry, e.g. `github.com/your-org/skill-registry.git`. (Or
   pass `source_repo` per run.)

2. **`SKILL_SYNC_PAT` secret** — `GITHUB_TOKEN` only has access to the repo it
   runs in, so cross-repo reads need a PAT. Create one with:
   - **Fine-grained:** *Read* contents on RepoA + *Read/Write* contents on RepoB.
   - **Classic:** `repo` scope.

   Add it as a repository secret named `SKILL_SYNC_PAT` in **RepoB** (and in
   **RepoA** too if you use the notifier, which needs dispatch access to RepoB).

3. **`fetch-depth: 0`** is mandatory and already set. `git subtree` needs full
   history to find merge bases; a shallow clone fails with merge-base errors.

## Why `merge`/`add` instead of `pull`

`git subtree pull` requires `<repository> <ref>` arguments — it can't consume a
bare local branch. Because step 2 produces a *local* split branch, the inject
step uses `git subtree add <branch>` (first time) and `git subtree merge <branch>`
(updates). Both accept a bare commit-ish; `--squash` keeps RepoB's history flat.

## Notes / customization

- The workflow **pushes directly** to the current branch. If RepoB has branch
  protection, swap the final push for a PR (e.g. `peter-evans/create-pull-request`).
- Source-path validation uses `git cat-file -e`/`-t` so it works for any nesting
  depth and confirms the target is a directory (tree), not a file.
- The push step is a no-op when a squash-merge produced no new commit, so reruns
  with no upstream changes are safe.
