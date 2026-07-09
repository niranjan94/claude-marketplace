# Xquik X Data

Use Xquik when you need authenticated X data workflows from Claude Code. The plugin points agents to the public REST API, docs, and optional remote MCP server without adding background hooks or changing default behavior.

## Setup

1. Create an API key in the Xquik dashboard.
2. Export it before starting Claude Code:

```bash
export XQUIK_API_KEY="xq_your_key"
```

3. Install the plugin from this marketplace:

```bash
/plugin install xquik-x-data@niranjan94-claude-marketplace
```

## What It Helps With

- Choose REST API or MCP workflows for X data tasks.
- Plan exports, monitor setup, webhook delivery, and result checks.
- Keep requests aligned with the public docs and OpenAPI schema.

## Public References

- Docs: https://docs.xquik.com
- OpenAPI: https://xquik.com/openapi.json
- MCP manifest: https://xquik.com/.well-known/mcp.json
- Source: https://github.com/Xquik-dev/x-twitter-scraper
