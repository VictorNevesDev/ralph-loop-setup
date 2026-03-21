# Ralph Loop Setup

A ready-to-use template for running [Ralph Loop](https://github.com/anthropics/claude-code/tree/main/ralph) — the autonomous AI coding agent that takes PRDs and implements them story by story.

This template gives you a pre-configured Dev Container with Claude Code installed, `--dangerously-skip-permissions` already wired up, and a network firewall so the agent can only talk to approved services (GitHub, npm, Anthropic API). Clone it, open in VS Code, and start running Ralph.

## What's Included

- **Dev Container** — Dockerfile and `devcontainer.json` ready to build, with Claude Code pre-installed globally
- **Network firewall** — `iptables` rules that block all outbound traffic except GitHub, npm, Anthropic API, Sentry, and VS Code marketplace
- **Ralph script** — `ralph.sh` pre-configured with `--dangerously-skip-permissions` for fully autonomous operation
- **Agent instructions** — `CLAUDE.md` with the iteration loop logic (read PRD, pick story, implement, commit, repeat)
- **PRD skill** — Claude Code skill to convert your PRDs into Ralph's `prd.json` format

## Prerequisites

- [Docker](https://www.docker.com/) installed and running
- [VS Code](https://code.visualstudio.com/) with the [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)
- An [Anthropic API key](https://console.anthropic.com/) (for Claude Code) or Amp credentials
- [GitHub CLI (`gh`)](https://cli.github.com/) authenticated for repo access

## Quick Start

### 1. Clone and open in Dev Container

```bash
git clone <repo-url>
cd ralph_loop
```

Open in VS Code and reopen in the Dev Container when prompted (or run `Dev Containers: Reopen in Container` from the command palette).

The container automatically:
- Installs Claude Code globally
- Configures a firewall that only allows traffic to approved services
- Sets up zsh with git and fzf integrations

### 2. Clone your project inside the container

Once inside the Dev Container, clone the repo you want Ralph to work on:

```bash
git clone https://github.com/your-org/your-project.git
cd your-project
```

### 3. Create a PRD

Write a PRD for your feature (markdown format), then convert it to Ralph's `prd.json` format:

```bash
# Using the /ralph skill in Claude Code
claude
# Then type: /ralph
# Paste or reference your PRD
```

Or manually create `scripts/ralph/prd.json`:

```json
{
  "project": "MyProject",
  "branchName": "ralph/feature-name",
  "description": "Feature description",
  "userStories": [
    {
      "id": "US-001",
      "title": "Story title",
      "description": "As a user, I want X so that Y",
      "acceptanceCriteria": [
        "Criterion 1",
        "Typecheck passes"
      ],
      "priority": 1,
      "passes": false,
      "notes": ""
    }
  ]
}
```

### 4. Run Ralph

```bash
cd scripts/ralph
./ralph.sh [--tool amp|claude] [max_iterations]
```

**Examples:**

```bash
# Run with Amp (default), 10 iterations max
./ralph.sh

# Run with Claude Code, 20 iterations max
./ralph.sh --tool claude 20

# Run with Amp, 5 iterations max
./ralph.sh --tool amp 5
```

## How It Works

This template wraps [Ralph Loop](https://github.com/anthropics/claude-code/tree/main/ralph) with a sandboxed container environment. The loop itself:

1. Reads the PRD and progress log
2. Checks out the correct branch
3. Picks the highest-priority incomplete story
4. Spawns a fresh Claude/Amp instance to implement it
5. Commits changes and updates progress
6. Repeats until all stories pass or max iterations reached

Each iteration is a fresh agent instance with no memory of previous runs. Context is passed through `progress.txt` and `CLAUDE.md`.

## Project Structure

| File | Purpose |
|------|---------|
| `.devcontainer/` | Dev Container config, Dockerfile, and firewall script |
| `scripts/ralph/ralph.sh` | Main loop script (calls Claude with `--dangerously-skip-permissions`) |
| `scripts/ralph/CLAUDE.md` | Agent instructions read each iteration |
| `scripts/ralph/prd.json` | Your PRD with user stories (you create this) |
| `scripts/ralph/progress.txt` | Cumulative progress log (auto-generated) |

## Firewall

The firewall (`init-firewall.sh`) runs on container start and uses `iptables` + `ipset` to block all outbound traffic except:

- GitHub (API, web, git — IPs fetched dynamically from GitHub's `/meta` endpoint)
- npm registry
- Anthropic API
- Sentry
- VS Code marketplace

This prevents the AI agent from making unintended network requests while running autonomously.

## Tips for Writing PRDs

- **Keep stories small** — each must be completable in a single iteration (one context window)
- **Order by dependency** — schema changes first, then backend, then UI
- **Use verifiable acceptance criteria** — e.g., "Typecheck passes", "Filter shows: All, Active, Done"
- **Always include** `"Typecheck passes"` as a criterion on every story

## Stop Condition

Ralph stops when:
- All user stories have `"passes": true` → outputs `COMPLETE`
- Max iterations reached → exits with error
