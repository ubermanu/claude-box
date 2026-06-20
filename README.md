# claude-box

One command to boot an isolated Docker box and attach an interactive Claude session inside it.

```sh
claude-box <label> <owner/repo>
```

The repo's default branch is cloned into a Docker volume — never touching your host filesystem. Only your Claude subscription credentials are copied in. The box environment is a single `Dockerfile` you control.

## Why

Running an agent with `--dangerously-skip-permissions` on your real files and credentials is risky. `claude-box` gives each task its own sandbox: a fresh clone in a volume, a clean container, and only the auth it needs.

## Usage

```sh
claude-box <label> <owner/repo>   # create a box from a GitHub repo, then attach
claude-box <label>                # reattach to a running box
```

- `label` — names the volume and container. Each label is an independent box; many run at once.
- `owner/repo` — a GitHub repo. Required to create; omit it to reattach.

It's idempotent: if the box is up it attaches; otherwise it builds the image, clones the repo, runs the container, sets up Claude, then attaches.

To detach, close the terminal — the session keeps running; re-run the same command to reattach. (Ctrl-\ and Ctrl-Z pass through to Claude rather than detaching.)

```sh
claude-box refactor you/project   # sandbox you/project under the label "refactor"
claude-box refactor               # jump back into it later
```

## How it works

1. **Image** — `~/.claude-box/Dockerfile` is built and tagged `claude-box`. Docker's layer cache makes rebuilds free when it hasn't changed.
2. **Volume** — `claude-box-<label>` is created and the repo's default branch is cloned into it by a throwaway `alpine/git` container; the host fs is never touched.
3. **Run** — the container starts from the `claude-box` image with the volume mounted at `/workspaces` and `~/.claude-box/.env` injected via `--env-file`. It's tagged `--label claude-box=<label>` so the box can be found again.
4. **Provision** — `homeadditions/` is copied into the box home, your Claude credentials are copied to `~/.claude`, then a setup script installs `dtach` + the `claude` CLI (if missing), pre-accepts the onboarding/trust/bypass-permissions prompts, and logs in `gh` with `GH_TOKEN`.
5. **Attach** — `dtach -A` creates (or reattaches to) the session running `claude --dangerously-skip-permissions`.

## Requirements

- Docker
- A Claude install on the host — run `claude` once so `~/.claude/.credentials.json` exists.

## Configuration

Everything lives under `~/.claude-box/` (override the base with `CLAUDE_BOX_HOME`). Sensible defaults are seeded on first run.

| Path | Purpose |
| --- | --- |
| `Dockerfile` | The box environment. Must keep a uid-1000 `node` user with passwordless `sudo`, plus `git` + `gh`. Seeded from `node:22-bookworm`. |
| `.env` | Global env vars, one `KEY=value` per line, injected via `--env-file`. Keep it `chmod 600`. |
| `homeadditions/` | Dotfiles/config copied into the box's home (`/home/node`). Not for secrets. |
| `GH_TOKEN` (in `.env`) | Used to clone private repos and authenticate `gh`/`git push` inside the box. |
| `~/.claude/.credentials.json` | Host Claude credentials, copied into the container. |

## Installation

```sh
cp claude-box ~/.local/bin/
chmod +x ~/.local/bin/claude-box
```
