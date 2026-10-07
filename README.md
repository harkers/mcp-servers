# mcp-servers — Self-Hosted GitHub MCP Server

Local collection of MCP (Model Context Protocol) servers for use with AI agents.

## Servers

### GitHub MCP Server

Wraps the official [`@modelcontextprotocol/server-github`](https://github.com/modelcontextprotocol/servers/tree/main/src/github) server for use with Goose and other MCP clients.

#### Quick Start

```bash
# Install Node.js dependencies
npm install -g @modelcontextprotocol/server-github

# Run with your PAT
GITHUB_TOKEN=ghp_your_token npx @modelcontextprotocol/server-github
```

#### Goose Configuration

Add to `~/.config/goose/config.yaml`:

```yaml
githubmcp:
  enabled: true
  type: stdio
  name: Github MCP
  cmd: npx
  args:
    - -y
    - '@modelcontextprotocol/server-github'
  envs:
    GITHUB_TOKEN: <your-pat>
```

See [SERVER_GITHUB.md](SERVER_GITHUB.md) for full documentation.
