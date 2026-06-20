# claude-box

One command to boot an isolated Docker container and attach an interactive Claude session inside it.

```sh
claude-box <owner/repo>
```

## Usage

```sh
claude-box <owner/repo>            # create a box from a GitHub repo, then attach
claude-box <owner/repo> -p "..."   # create a box, boot Claude with an initial prompt
cat PROMPT.md | claude-box <owner/repo>   # same, prompt piped on stdin
claude-box <id>                    # reattach to a running box
```

- `owner/repo` — a GitHub repo (anything with a `/`). Creates a fresh box and prints its id.
- `id` — the random 5-char id of an existing box (anything without a `/`). Reattaches to it.
- `-p, --prompt` — an initial prompt fed to Claude on first boot. If omitted and stdin is piped, the piped text is used instead. The prompt only fires on first start; reattaching resumes the existing session.

Each box gets a random id (e.g. `u87ec`) that keys its volume and container; many run at once. On create the id is printed so you can reattach later. It builds the image, clones the repo, runs the container, sets up Claude, then attaches.

To detach, close the terminal, the session keeps running; re-run `claude-box <id>` to reattach.

## Requirements

- Docker
- A Claude subscription
- A Claude install configured on the host

## Configuration

Everything lives under `~/.claude-box/` (override the base with `CLAUDE_BOX_HOME`). Sensible defaults are seeded on first run.

| Path | Purpose |
| --- | --- |
| `Dockerfile` | The box environment. Must keep a uid-1000 `node` user with passwordless `sudo`, plus `git` + `gh`. Seeded from `node:22-bookworm`. |
| `.env` | Global env vars, one `KEY=value` per line, injected via `--env-file`. Keep it `chmod 600`. |
| `homeadditions/` | Dotfiles/config copied into the box's home (`/home/node`). Not for secrets. |
| `~/.claude/.credentials.json` | Host Claude credentials, copied into the container. |
