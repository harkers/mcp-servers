# GitHub MCP Server — Self-Hosted Configuration

## Overview

This repository wraps the official Model Context Protocol (MCP) GitHub server for local use with Goose. The server runs as a `stdio` process — no network server, no OAuth flows, no browser prompts.

## Requirements

- **Node.js** ≥ 18 (tested with v22)
- **Goose** ≥ 1.40
- A **GitHub Personal Access Token (PAT)** with appropriate scopes

## Installation

### 1. Install the server package

```bash
npm install -g @modelcontextprotocol/server-github
```

Verify it works:

```bash
npx @modelcontextprotocol/server-github
# Should print: "GitHub MCP Server running on stdio"
```

### 2. Generate a GitHub PAT

Go to **GitHub → Settings → Developer settings → Personal access tokens → Fine-grained tokens**

Required scopes for common use cases:

| Use Case | Required Scopes |
|---|---|
| Basic repo read | `repo`, `read:org` |
| Read-only (recommended) | `repo`, `read:org`, `read:projects` |
| Full CRUD | `repo`, `read:org`, `read:projects`, `write:org` |
| Issues only | `repo` (repo scope includes issues) |

**Recommendation:** Create a **fine-grained** token with minimal needed scopes only.

### 3. Store the token securely

Add the token to Goose's secrets file (`~/.config/goose/secrets.yaml`):

```yaml
github_mcp_token: ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Then reference it in config via env var key (if your Goose version supports env_keys), or embed directly in the config (less secure).

## Goose Configuration

Add this block to `~/.config/goose/config.yaml` under `extensions:`:

```yaml
  githubmcp:
    enabled: true
    type: stdio
    name: Github MCP
    description: Self-hosted GitHub MCP server for repo, issue, and PR operations
    cmd: npx
    args:
      - -y
      - '@modelcontextprotocol/server-github'
    envs:
      GITHUB_TOKEN: ghp_your_token_here
    env_keys: []
    timeout: 300
    cwd: null
    bundled: null
```

### Key differences from the broken `streamable_http` config

| Aspect | Broken (remote) | Fixed (self-hosted) |
|---|---|---|
| **Type** | `streamable_http` | `stdio` |
| **Auth** | OAuth flow (doesn't work) | Direct PAT via `GITHUB_TOKEN` |
| **Endpoint** | `api.githubcopilot.com/mcp/` | N/A — runs locally on stdio |
| **Browser needed?** | Yes (device flow) | No |

## Available Tools

The GitHub MCP server exposes these tools:

### Repository Management
- **create_or_update_file** — Create or update a file in a repository
- **create_repository** — Create a new repository
- **delete_file** — Delete a file from a repository
- **fork_repository** — Fork a repository to your account

### Issues
- **create_issue** — Create a new issue
- **get_issue** — Get issue details by number
- **list_issues** — List and filter issues
- **add_issue_comment** — Add a comment to an issue
- **update_issue** — Update issue fields

### Pull Requests
- **create_pull_request** — Create a new PR
- **get_pull_request** — Get PR details
- **list_pull_requests** — List and filter PRs
- **get_pull_request_files** — Get changed files in a PR
- **get_pull_request_status** — Get PR commit statuses
- **merge_pull_request** — Merge a PR

### Code Search
- **search_code** — Search code across repositories
- **search_issues** — Search issues and PRs
- **search_repositories** — Search repositories

### Reviews
- **get_pull_request_reviews** — List review requests
- **get_pull_request_review_thread** — Get review comments

### Project/Workflow
- **get_file_contents** — Read file from repository
- **get_branch** — Get branch information
- **create_branch** — Create a new branch
- **list_branches** — List repository branches
- **search_code**, **search_issues**, **search_repositories**

## Security Notes

1. **Never commit your PAT** to version control. Store it in `secrets.yaml` or use an environment variable.
2. **Use fine-grained tokens** with the minimum scopes needed.
3. **Rotate tokens** periodically (GitHub allows 1-year expiry on fine-grained tokens).
4. The server runs locally and **never sends your token to a third party** — it passes directly to `api.github.com`.

## Troubleshooting

### "Token does not have sufficient privileges"
- Your PAT lacks one or more required scopes. Regenerate with the needed scopes.
- For repos in organizations, you need `read:org` scope.

### "GitHub MCP Server running on stdio" but Goose fails to connect
- Ensure `npx` is available and can resolve `@modelcontextprotocol/server-github`.
- Check Node.js version: `node --version` (need ≥ 18).

### "Repository not found"
- The token may not have access to the target repository.
- For private repos, the token must have `repo` scope.
- For organization repos, ensure the token's user has access.

## Reference

- Official server: https://github.com/modelcontextprotocol/servers/tree/main/src/github
- MCP spec: https://spec.modelcontextprotocol.io/
- Goose docs: https://github.com/gooseworkspace/goose
