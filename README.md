# claude-box

One command to boot an isolated devcontainer and attach an interactive Claude session inside it.

```sh
claude-box <task> [repo-source]
```

The repo lives in a Docker volume — never on your host filesystem. Only your Claude subscription credentials are copied in. Secrets are injected via devcontainer's native `--secrets-file`, so they never hit `argv`, `ps`, or shell history.

## Why

Running an agent with `--dangerously-skip-permissions` on your real files and credentials is risky. `claude-box` gives each task its own sandbox: an isolated repo copy, a clean container, and only the auth it needs.

## Usage

```sh
claude-box <task> [repo-source]
```

- `task` — run label; names the volume, dtach session, and container.
- `repo-source` — local path or git URL. Defaults to the current directory.

It's idempotent: if the box is up it attaches; otherwise it creates the volume, clones the repo, boots the container, sets up Claude, then attaches.

To detach, close the terminal — the session keeps running; re-run the same command to reattach. (Ctrl-\ and Ctrl-Z pass through to Claude rather than detaching.)

```sh
claude-box refactor                                   # sandbox cwd
claude-box fix-bug https://github.com/you/project.git # clone a remote repo
claude-box review ~/code/some-project                 # sandbox a local path
```

## How it works

1. **Volume** — `claude-box-<task>` is created and the repo cloned into it by a throwaway `alpine/git` container; the host fs is never touched. A setup script is dropped in alongside.
2. **Config** — the repo's own `.devcontainer` is reused if present, else a default universal image. It's patched to mount the volume as the workspace.
3. **Boot** — `@devcontainers/cli` brings the container up. Claude credentials are bind-mounted in; secrets are passed via `--secrets-file`.
4. **Setup** — installs `dtach` and the `claude` CLI if missing, copies in your credentials, pre-accepts the onboarding, trust, and bypass-permissions prompts, and optionally logs in `gh` with `GH_TOKEN`.
5. **Attach** — `dtach -A` creates (or reattaches to) the session running `claude --dangerously-skip-permissions`.

## Requirements

- Docker
- Node.js / `npx`
- `jq`
- A Claude install on the host — run `claude` once so `~/.claude/.credentials.json` exists.

## Configuration

| Variable / path | Purpose |
| --- | --- |
| `CLAUDE_BOX_HOME` | Base directory for run state. Defaults to `~/.claude-box`. |
| `~/.claude-box/.env` | Secrets, one `KEY=value` per line. Injected via `--secrets-file`. Keep it `chmod 600` and gitignored. |
| `GH_TOKEN` (in `.env`) | If set, authenticates `gh` in the box and clones private git URLs. |
| `~/.claude/.credentials.json` | Host Claude credentials, copied into the container. |

## Installation

```sh
cp claude-box ~/.local/bin/
chmod +x ~/.local/bin/claude-box
```
