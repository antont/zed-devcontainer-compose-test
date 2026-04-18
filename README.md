# Compose Duplicate Repro

Minimal fixture to reproduce a duplicate-devcontainer state.

Two tools (e.g. the `devcontainer` CLI and Zed) pick different Docker Compose
project names when bringing up the same `devcontainer.json`. Both tools apply
the same `devcontainer.local_folder` / `devcontainer.config_file` labels, so
you end up with two containers that match the identifying filters, in
violation of the spec's intent that those labels be unique per project.

The key ingredient: the `devcontainer.json` sets a `name` field. Zed uses
that (safe-id-lowered) as the compose project name, while the
`devcontainer` CLI / VS Code derive the project name from the folder name.
The two derivations disagree, so each tool creates its own compose project
rather than reusing the other's.

## Reproduce

Requires a Docker daemon and Node's `@devcontainers/cli` (`devcontainer` on
`PATH`).

```sh
# Start from a clean slate
docker ps -a --filter "label=devcontainer.local_folder=$PWD" -q | xargs -r docker rm -f

# 1. Bring up with the reference CLI (same project-name derivation as VS Code)
devcontainer up --workspace-folder .

# 2. Bring up the same folder in Zed
#    (Zed picks a different compose project name because devcontainer.json has `name`)

# 3. Observe the duplicates
docker ps -a \
  --filter "label=devcontainer.local_folder=$PWD" \
  --filter "label=devcontainer.config_file=$PWD/.devcontainer/devcontainer.json" \
  --format '{{.Names}}  project={{.Label "com.docker.compose.project"}}'
```

You should see two `*-app-1` containers with different
`com.docker.compose.project` values but identical identifying labels.

## Clean up

```sh
docker ps -a --filter "label=devcontainer.local_folder=$PWD" -q | xargs -r docker rm -f
```
