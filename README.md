# claude-box

One command to boot an isolated Docker box and attach an interactive Claude session inside it.

```sh
claude-box <owner/repo>
```

The repo's default branch is cloned into a Docker volume — never touching your host filesystem. Only your Claude subscription credentials are copied in. The box environment is a single `Dockerfile` you control.

## Why

Running an agent with `--dangerously-skip-permissions` on your real files and credentials is risky. `claude-box` gives each task its own sandbox: a fresh clone in a volume, a clean container, and only the auth it needs.

## Usage

```sh
claude-box <owner/repo>   # create a box from a GitHub repo, then attach
claude-box <id>           # reattach to a running box
```

- `owner/repo` — a GitHub repo (anything with a `/`). Creates a fresh box and prints its id.
- `id` — the random 5-char id of an existing box (anything without a `/`). Reattaches to it.

Each box gets a random id (e.g. `u87ec`) that keys its volume and container; many run at once. On create the id is printed so you can reattach later. It builds the image, clones the repo, runs the container, sets up Claude, then attaches.

To detach, close the terminal — the session keeps running; re-run `claude-box <id>` to reattach. (Ctrl-\ and Ctrl-Z pass through to Claude rather than detaching.)

```sh
claude-box you/project   # sandbox you/project in a new box -> prints id, e.g. u87ec
claude-box u87ec         # jump back into it later
```

## How it works

1. **Image** — `~/.claude-box/Dockerfile` is built and tagged `claude-box`. Docker's layer cache makes rebuilds free when it hasn't changed.
2. **Volume** — `claude-box-<id>` is created and the repo's default branch is cloned into it by a throwaway `alpine/git` container; the host fs is never touched.
3. **Run** — the container starts from the `claude-box` image with the volume mounted at `/workspaces` and `~/.claude-box/.env` injected via `--env-file`. It's tagged `--label claude-box=<id>` so the box can be found again.
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
