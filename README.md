# neynar-deploy

Deploy static sites, Vite apps, and Next.js apps to a live URL with a single API call. Built-in versioning with instant rollback -- no git required.

**API:** `https://api.host.neynar.app`
**Deployed sites:** `https://{project-name}.host.neynar.app`

## Install as an Agent Skill

### Claude Code

```bash
claude /install-skill https://github.com/neynarxyz/neynar-deploy
```

### Manual (any MCP-compatible agent)

Copy the `SKILL.md` file from this repo into your project's `.claude/skills/` directory, or add it to your agent's skill search path.

### MCP Server

If your agent supports MCP, connect directly:

```json
{
  "mcpServers": {
    "neynar-deploy": {
      "command": "npx",
      "args": ["-y", "@neynar/service.agent-deploy"]
    }
  }
}
```

## Quick Start

```bash
# Create and archive a site
mkdir -p /tmp/my-site
echo '<html><body><h1>Hello World</h1></body></html>' > /tmp/my-site/index.html
tar czf /tmp/my-site.tar.gz -C /tmp/my-site .

# Deploy (no API key needed -- one is auto-generated)
curl -X POST https://api.host.neynar.app/v1/deploy \
  -F "files=@/tmp/my-site.tar.gz" \
  -F "projectName=my-site" \
  -F "framework=static"
```

The response includes a live URL and an API key for subsequent deploys.

## What You Get

- **One-call deploy** -- tar.gz your files, POST, get a URL
- **Built-in versioning** -- every deploy auto-increments a version number
- **Instant rollback** -- revert to any previous version
- **Project context** -- structured deploy history and analytics in one API call
- **Custom domains** -- `{project-name}.host.neynar.app` automatically
- **Analytics** -- pageviews, unique visitors, top pages, referrers
- **Free tier** -- 3 projects, 10 deploys/hour, 50MB max upload

## Full API Reference

See the complete [API documentation](SKILL.md#api-reference) in the SKILL.md file.

## License

MIT
