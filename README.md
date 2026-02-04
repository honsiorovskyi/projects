# projects

A set of bash scripts for managing isolated project workspaces with Git repositories.

## Why?

This tool is designed primarily for working with **AI coding agents** (like Claude, Cursor, etc.) across multiple repositories. Instead of running an agent on each repo individually, you can:

1. Group related repos into a single project workspace
2. Give the AI agent access to the entire project
3. Let it understand and modify code across repo boundaries

This is especially useful for microservices, monorepo-adjacent setups, or any work that spans multiple repositories.

## Concept

Each project lives in `$PROJECTS_HOME/<project-name>/` (default: `~/projects/`). Inside a project folder, you clone repositories that are relevant to that project. Each repo gets a branch named after the project, keeping your work isolated.

```
~/projects/
├── feature-xyz-jira-123/
│   ├── api-gateway/       # branch: feature-xyz-jira-123
│   ├── user-service/      # branch: feature-xyz-jira-123
│   └── shared-lib/        # branch: feature-xyz-jira-123
└── bugfix-abc-jira-456/
    ├── backend/           # branch: bugfix-abc-jira-456
    └── frontend/          # branch: bugfix-abc-jira-456
```

## Requirements

- `bash`
- `git`
- `fzf`
- `curl` and `jq` (for `psync`)

## How It Works

The tool uses two separate caches:

### Repo List (`PROJECTS_CACHE_REPOS_LIST`)

A lightweight text file containing repository names from your organization. This powers the fzf picker in `padd`.

- **Location**: `~/.cache/p.repos.list` (a few KB)
- **Updated by**: `psync` (fetches from git provider API)
- **Auto-sync**: If you source the prompt helpers, `projects::maybe_sync` refreshes this in the background when it's older than `PROJECTS_SYNC_MAX_AGE` (default: 1 week)

### Git Cache (`PROJECTS_CACHE_REPOS`)

Local bare repositories used to speed up cloning. When you clone a repo for the first time, it's cached here. Subsequent clones of the same repo (in other projects) copy from the local cache instead of fetching everything from the remote.

- **Location**: `~/.cache/p.repos/` (grows as you use more repos)
- **Updated by**: `pclone` (on-demand, when you clone)
- **Content**: Shallow bare repos (single branch, depth 1)

```
~/.cache/
├── p.repos.list          # repo list (lightweight, auto-synced)
└── p.repos/              # git cache (grows on demand)
    ├── api-gateway.git
    ├── user-service.git
    └── shared-lib.git
```

## Environment Variables

### Core

| Variable | Description | Default |
|----------|-------------|---------|
| `PROJECTS_HOME` | Base directory for projects | `~/projects` |

### Git Provider

| Variable | Description | Default |
|----------|-------------|---------|
| `GITHUB_ORG` | GitHub organization name | - |
| `GITHUB_TOKEN` | GitHub API token (for private repos) | - |
| `PROJECTS_GIT_BASE` | Git URL prefix for cloning | `git@github.com:$GITHUB_ORG` |
| `PROJECTS_SYNC_COMMAND` | Custom command to list repos (one per line) | `github-repos` |

### Cache

| Variable | Description | Default |
|----------|-------------|---------|
| `PROJECTS_CACHE_REPOS_LIST` | Repo list file (lightweight, auto-synced) | `~/.cache/p.repos.list` |
| `PROJECTS_CACHE_REPOS` | Git bare repo cache (grows on demand) | `~/.cache/p.repos` |
| `PROJECTS_SYNC_MAX_AGE` | Auto-sync repo list when older than N seconds | `604800` (1 week) |

### Optional

| Variable | Description | Default |
|----------|-------------|---------|
| `PROJECTS_TICKET_ENABLED` | Enable ticket ID integration in `pnew` | (disabled) |

## Commands

### `pnew [name] [ticket] [repo_filter]`

Create a new project directory.

- Prompts for missing `name` argument
- If `PROJECTS_TICKET_ENABLED=1`, also prompts for ticket ID
- Creates `$PROJECTS_HOME/<name>[-<ticket>]/`
- Opens `padd` for adding repos

```bash
# Without ticket integration (default)
pnew "feature xyz" api
# Creates ~/projects/feature-xyz/ and opens repo selector filtered by "api"

# With PROJECTS_TICKET_ENABLED=1
pnew "feature xyz" JIRA-123 api
# Creates ~/projects/feature-xyz-jira-123/ and opens repo selector filtered by "api"
```

### `padd [--sync] [query]`

Interactively add repositories to the current project.

- Must be run from inside a project directory
- Uses fzf to select repos (shows ✓ for already-added repos)
- `--sync` refreshes the repo cache first
- `query` pre-fills the fzf search

```bash
padd api         # filter repos by "api"
padd --sync      # refresh cache, then select
```

### `pclone <repo1> [repo2] ...`

Clone repositories into the current project.

- Uses a local bare repo cache for faster subsequent clones
- Creates a branch named after the project in each repo
- Shallow clones by default

```bash
pclone api-gateway user-service
```

### `pgo [--print] [project] [repo]`

Navigate to a project/repo.

- `project` and `repo` are fzf query prefills
- `--print` outputs the path instead of spawning a shell
- `.` entry in repo list goes to project root

```bash
pgo              # select project, then repo
pgo feature      # filter projects by "feature"
pgo feature api  # filter projects by "feature", repos by "api"
```

### `psync`

Sync the list of organization repositories to cache.

- Uses `github-repos` by default (requires `GITHUB_ORG`)
- Or runs `PROJECTS_SYNC_COMMAND` if set
- Saves to `$PROJECTS_CACHE_REPOS_LIST`

```bash
psync    # syncs repos from GITHUB_ORG
```

### `github-repos [days]`

List GitHub org repos pushed within the last N days (default: 180).

- Requires `GITHUB_ORG` and optionally `GITHUB_TOKEN`
- Used by `psync` as the default sync method

```bash
github-repos        # repos pushed in last 180 days
github-repos 30     # repos pushed in last 30 days
```

#### Other Git Providers

Set `PROJECTS_SYNC_COMMAND` to a command that outputs repo names (one per line):

```bash
# GitLab
export PROJECTS_SYNC_COMMAND='curl -s "https://gitlab.com/api/v4/groups/$ORG/projects?per_page=100&simple=true" \
  -H "PRIVATE-TOKEN: $GITLAB_TOKEN" | jq -r ".[].path"'

# Codeberg / Gitea
export PROJECTS_SYNC_COMMAND='curl -s "https://codeberg.org/api/v1/orgs/$ORG/repos?limit=100" \
  -H "Authorization: token $CODEBERG_TOKEN" | jq -r ".[].name"'

# Bitbucket
export PROJECTS_SYNC_COMMAND='curl -s "https://api.bitbucket.org/2.0/repositories/$ORG?pagelen=100" \
  -u "$BITBUCKET_USER:$BITBUCKET_TOKEN" | jq -r ".values[].slug"'

# Static file
export PROJECTS_SYNC_COMMAND='cat ~/.config/my-repos.txt'
```

## Shell Aliases

Add to your shell config:

```bash
source ~/.local/projects/share/projects/aliases
```

This provides:
- `p` - navigate to project/repo (wrapper around `pgo --print`)
- `pn` - alias for `pnew`
- `pa` - alias for `padd`

## Prompt Integration

Source the prompt helpers to show project context in your shell prompt:

```bash
source ~/.local/projects/share/projects/prompt
```

### Available Functions

| Function | Description |
|----------|-------------|
| `projects::is_inside_project` | Returns 0 if in a project directory |
| `projects::is_inside_repo` | Returns 0 if in a repo within a project |
| `projects::get_project` | Outputs current project name |
| `projects::get_repo` | Outputs current repo name |
| `projects::get_context` | Outputs `project:repo` or `project` |
| `projects::branch_matches_project` | Returns 0 if git branch == project name |
| `projects::git_status` | Outputs git status indicators (`*+%`) |
| `projects::maybe_sync` | Run psync in background if cache is stale |
| `projects::prompt_prefix` | Outputs colored prompt prefix |
| `projects::prompt_prefix_plain` | Outputs plain text prompt prefix |

### Example: Custom Prompt

```bash
source ~/.local/projects/share/projects/prompt

my_prompt() {
  projects::maybe_sync  # auto-sync in background if cache is stale
  
  local prefix=$(projects::prompt_prefix)
  if [[ -n "$prefix" ]]; then
    PS1="$prefix \$ "
  else
    PS1="\W \$ "
  fi
}
PROMPT_COMMAND='my_prompt'
```

This shows:
- `feature-xyz/api-gateway $ ` when in a project repo (branch hidden if it matches project)
- `feature-xyz/api-gateway main *+ $ ` when on a different branch with changes
- `~ $ ` when outside projects

## Setup

### GitHub (Quickstart)

1. Clone the repository:
```bash
git clone https://github.com/honsiorovskyi/projects.git ~/.local/projects
```

2. Add to your shell config (e.g., `~/.bashrc`):
```bash
export GITHUB_ORG="your-org"
export GITHUB_TOKEN="ghp_..."  # optional, for private repos
export PATH="$PATH:$HOME/.local/projects/bin"
source ~/.local/projects/share/projects/aliases
```

3. Sync and create your first project:
```bash
psync
pnew
```

### Other Providers

1. Clone the repository:
```bash
git clone https://github.com/honsiorovskyi/projects.git ~/.local/projects
```

2. Add to your shell config:
```bash
export PROJECTS_GIT_BASE="git@gitlab.com:your-org"  # or your provider
export PROJECTS_SYNC_COMMAND='your-sync-command'    # see examples above
export PATH="$PATH:$HOME/.local/projects/bin"
source ~/.local/projects/share/projects/aliases
```

3. Sync and create your first project:
```bash
psync
pnew
```

## License

AGPL-3.0 - see [LICENSE](LICENSE)
