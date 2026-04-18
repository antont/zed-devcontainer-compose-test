# VERIFY: fixture for zed-industries/zed issue on compose-project-name divergence

Each numbered step is copy-pasteable. `$PWD` refers to this fixture's folder.
Every "expected output" block was captured from a real run on 2026-04-18.

## 0. Prerequisites

- Docker daemon running.
- `@devcontainers/cli` installed (`devcontainer` on `PATH`).
- A Zed build with Rust-impl dev containers (stable >= v0.232.2, or a local dev build).

## 1. Clean slate

```sh
docker ps -a --filter "label=devcontainer.local_folder=$PWD" -q | xargs -r docker rm -f
```

Expect: no output (or the list of container IDs being removed).

## 2. Claim: Zed derives compose project from `safe_id_lower(devcontainer.json's name)`

Steps:

1. Open this folder in Zed: `zed --dev-container $PWD`.
2. After the container comes up, inspect:

```sh
docker ps -a \
  --filter "label=devcontainer.local_folder=$PWD" \
  --filter "label=devcontainer.config_file=$PWD/.devcontainer/devcontainer.json" \
  --format '{{.ID}}  {{.Names}}  project={{.Label "com.docker.compose.project"}}'
```

Expected: one container whose project label is `compose_duplicate_repro` — i.e.
`safe_id_lower("Compose Duplicate Repro")` where `name` is taken from
`.devcontainer/devcontainer.json`.

Real output from 2026-04-18:

```
104a7750c825  compose_duplicate_repro-app-1  project=compose_duplicate_repro
```

## 3. Claim: `@devcontainers/cli` derives project from `${folderBasename}_devcontainer`

**Important**: do not quit Zed first. Run the CLI while Zed's container is still
running, so both tools are active on this folder simultaneously.

```sh
devcontainer up --workspace-folder $PWD
```

Then:

```sh
docker ps -a \
  --filter "label=devcontainer.local_folder=$PWD" \
  --format '{{.ID}}  {{.Names}}  project={{.Label "com.docker.compose.project"}}'
```

Expected: two containers, one for each tool, with different `project` values.

Real output from 2026-04-18:

```
d11220c9e1cb  devcontainer-compose-test_devcontainer-app-1  project=devcontainer-compose-test_devcontainer
104a7750c825  compose_duplicate_repro-app-1                  project=compose_duplicate_repro
```

`devcontainer-compose-test_devcontainer` is the folder's basename +
`_devcontainer`, matching the CLI's `getDefaultName` derivation.

## 4. Claim: both containers stamp identical identifying labels

```sh
for id in $(docker ps -a --filter "label=devcontainer.local_folder=$PWD" -q); do
  echo "=== $id ==="
  docker inspect "$id" --format 'project={{index .Config.Labels "com.docker.compose.project"}}
local_folder={{index .Config.Labels "devcontainer.local_folder"}}
config_file={{index .Config.Labels "devcontainer.config_file"}}'
done
```

Expected: different `project`, identical `local_folder`, identical `config_file`.

Real output from 2026-04-18:

```
=== d11220c9e1cb ===
project=devcontainer-compose-test_devcontainer
local_folder=/Users/antont/src/devcontainer-compose-test
config_file=/Users/antont/src/devcontainer-compose-test/.devcontainer/devcontainer.json
=== 104a7750c825 ===
project=compose_duplicate_repro
local_folder=/Users/antont/src/devcontainer-compose-test
config_file=/Users/antont/src/devcontainer-compose-test/.devcontainer/devcontainer.json
```

## 5. Claim: spec-style dual-label query returns >1 row

```sh
docker ps -a \
  --filter "label=devcontainer.local_folder=$PWD" \
  --filter "label=devcontainer.config_file=$PWD/.devcontainer/devcontainer.json" \
  --format '{{.ID}}  {{.Names}}  project={{.Label "com.docker.compose.project"}}'
```

Expected: two rows. This is the query Zed's `find_process_by_filters` issues;
the duplicate-container error surfaced by zed-industries/zed#54068 fires on
exactly this output.

Real output from 2026-04-18:

```
d11220c9e1cb  devcontainer-compose-test_devcontainer-app-1  project=devcontainer-compose-test_devcontainer
104a7750c825  compose_duplicate_repro-app-1                  project=compose_duplicate_repro
```

## 6. Claim: order matters — CLI first, then Zed, does NOT duplicate

Reset first:

```sh
docker ps -a --filter "label=devcontainer.local_folder=$PWD" -q | xargs -r docker rm -f
```

Then:

1. `devcontainer up --workspace-folder $PWD` → creates
   `devcontainer-compose-test_devcontainer-app-1`.
2. Open in Zed: `zed --dev-container $PWD`.
3. Observe:

```sh
docker ps -a \
  --filter "label=devcontainer.local_folder=$PWD" \
  --format '{{.ID}}  {{.Names}}  project={{.Label "com.docker.compose.project"}}'
```

Expected: one row (the CLI's container). Zed's
`check_for_existing_container` correctly finds the CLI container by its
`devcontainer.*` labels and reuses it rather than creating its own.

Real output from 2026-04-18:

```
9b2b92c8de6c  devcontainer-compose-test_devcontainer-app-1  project=devcontainer-compose-test_devcontainer
```

`9b2b92c8de6c` is the CLI-created container ID; after Zed attached, this
row is unchanged and no new compose project appeared. The Zed log for
the same session confirms the attach via `SshRemoteClient`:

```
2026-04-18T16:29:31 INFO  [remote::transport::docker] Remote platform discovered: Some(RemotePlatform { os: Linux, arch: Aarch64 })
2026-04-18T16:30:38 INFO  [remote_server] (remote server) accepting new connections
```

This asymmetry is what makes the issue latent: users who always open Zed
before running the CLI see duplicates; users who start with the CLI do not.

## Cleanup

```sh
docker ps -a --filter "label=devcontainer.local_folder=$PWD" -q | xargs -r docker rm -f
```
