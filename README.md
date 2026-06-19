# claude-box

One command to boot an isolated devcontainer and attach an interactive Claude session inside it.

```sh
claude-box <task> [repo-source]
```

The repo's code lives in a Docker volume — never on your host filesystem. Your Claude subscription credentials are copied in; nothing else of your host auth comes along. Secrets are injected through devcontainer's native `--secrets-file`, so they never appear on `argv`, in `ps`, or in shell history.

## Why

Running an agent with `--dangerously-skip-permissions` on your real filesystem and your real credentials is risky. `claude-box` gives each task its own sandbox: an isolated copy of the repo, a clean container, and only the auth it needs.

## Usage

```sh
claude-box <task> [repo-source]
```

- `task` — a run label. Names the Docker volume, the dtach session, and the container.
- `repo-source` — a local path or a git URL. Defaults to the current directory.

The command is idempotent:

- **Box already up** → attaches to the running dtach session.
- **Box down** → creates the volume, clones the repo, boots the devcontainer, sets up Claude, then attaches.

Detach from the session with the dtach binding (`Ctrl-\`). Re-run the same command to reattach.

### Examples

```sh
# Sandbox the current directory under the label "refactor"
claude-box refactor

# Clone a remote repo into a fresh box
claude-box fix-bug https://github.com/you/project.git

# Sandbox a local repo by path
claude-box review ~/code/some-project
```

## How it works

1. **Volume** — a Docker volume `claude-box-<task>` is created and the repo is cloned into it via a throwaway `alpine/git` container, so the host filesystem is never touched. A setup script is dropped into the volume.
2. **Config** — the repo's own `.devcontainer` (or `.devcontainer.json`) is reused if present; otherwise a default universal image with the GitHub CLI feature is used. The config is patched to mount the volume as the workspace.
3. **Boot** — `@devcontainers/cli` brings the container up. Your Claude credentials are bind-mounted in at boot; secrets are passed via `--secrets-file`.
4. **Setup** — inside the container, the setup script installs `dtach` and the `claude` CLI (if missing), copies in your credentials, optionally logs in `gh` with `GH_TOKEN`, and starts Claude under dtach with `--dangerously-skip-permissions`.
5. **Attach** — `docker exec -it ... dtach -a` drops you into the live Claude session.

## Requirements

- Docker
- Node.js / `npx` (the script uses `npx -y @devcontainers/cli`)
- `jq`
- A Claude install on the host — run `claude` once so that `~/.claude/.credentials.json` exists.

## Configuration

| Variable / path | Purpose |
| --- | --- |
| `CLAUDE_BOX_HOME` | Base directory for run state. Defaults to `~/.claude-box`. |
| `~/.claude-box/.env` | Your secrets, one `KEY=value` per line. Injected into the container via `--secrets-file`. Keep it `chmod 600` and gitignored. |
| `GH_TOKEN` (in `.env`) | If set, used to authenticate `gh` inside the box and to clone private git URLs. |
| `~/.claude/.credentials.json` | Host Claude credentials, copied into the container. |

## Installation

Put the `claude-box` script somewhere on your `PATH`:

```sh
cp claude-box ~/.local/bin/
chmod +x ~/.local/bin/claude-box
```
