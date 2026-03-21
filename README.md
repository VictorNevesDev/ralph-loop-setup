# Ralph Loop

An autonomous AI coding agent loop that takes a PRD (Product Requirements Document), breaks it into user stories, and iteratively implements them using Claude Code or Amp.

Ralph runs in a sandboxed Dev Container with network restrictions, ensuring the AI agent can only access approved services (GitHub, npm, Anthropic API).

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
- Configures a firewall that only allows traffic to GitHub, npm, Anthropic API, and Sentry
- Sets up zsh with git and fzf integrations

### 2. Create a PRD

Write a PRD for your feature (markdown format), then convert it to Ralph's `prd.json` format:

```bash
# Using the /ralph skill in Claude Code
claude
# Then type: /ralph
# Paste or reference your PRD
```

Or manually create `scripts/ralph/prd.json` following this structure:

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

### 3. Run Ralph

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

Ralph will:
1. Read the PRD and progress log
2. Check out the correct branch
3. Pick the highest-priority incomplete story
4. Implement it and run quality checks
5. Commit changes and update progress
6. Repeat until all stories pass or max iterations reached

## How It Works

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│   prd.json  │────▶│  ralph.sh    │────▶│ Claude/Amp  │
│  (stories)  │     │  (loop)      │     │  (agent)    │
└─────────────┘     └──────┬───────┘     └──────┬──────┘
                           │                     │
                           │  ◀── iteration ──▶  │
                           │                     │
                    ┌──────▼───────┐     ┌──────▼──────┐
                    │ progress.txt │     │ git commit  │
                    │  (log)       │     │  (code)     │
                    └──────────────┘     └─────────────┘
```

Each iteration spawns a fresh agent instance with no memory of previous runs. The agent reads `CLAUDE.md` for instructions and `progress.txt` for context from prior iterations.

### Key Files

| File | Purpose |
|------|---------|
| `scripts/ralph/ralph.sh` | Main loop script |
| `scripts/ralph/CLAUDE.md` | Agent instructions (read each iteration) |
| `scripts/ralph/prd.json` | PRD with user stories (you create this) |
| `scripts/ralph/progress.txt` | Cumulative progress log (auto-generated) |
| `scripts/ralph/archive/` | Archived runs from previous features |

## Writing Good PRDs for Ralph

**Keep stories small** - each must be completable in a single iteration (one context window).

**Order by dependency** - schema changes first, then backend, then UI.

**Use verifiable acceptance criteria:**
- "Add `status` column with default `pending`"
- "Filter dropdown shows: All, Active, Done"
- "Typecheck passes"

**Always include** `"Typecheck passes"` as a criterion on every story.

See the full guide in `.claude/skills/ralph/SKILL.md`.

## Dev Container

The Dev Container provides a sandboxed environment with:

- **Node.js 20** runtime
- **Claude Code** pre-installed
- **Network firewall** restricting outbound traffic to:
  - GitHub (API, web, git)
  - npm registry
  - Anthropic API
  - VS Code marketplace
- **Tools**: git, gh, jq, fzf, zsh, nano, vim

### Firewall

The firewall (`init-firewall.sh`) runs on container start and blocks all outbound traffic except to approved services. This prevents the AI agent from making unintended network requests.

## Archiving

When you switch to a new feature (different `branchName` in `prd.json`), Ralph automatically archives the previous run's `prd.json` and `progress.txt` to:

```
scripts/ralph/archive/YYYY-MM-DD-feature-name/
```

## Stop Condition

Ralph stops when:
- All user stories have `"passes": true` (outputs `COMPLETE`)
- Max iterations reached (exits with error)
