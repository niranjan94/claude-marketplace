# niranjan94-claude-marketplace

A personal plugin marketplace for Claude. Serves as a registry of plugins and skills for use with AI agents.

## Installation

```
/plugin marketplace add niranjan94/claude-marketplace
```

(or)

```
claude plugin marketplace add niranjan94/claude-marketplace --scope user
```

And to install any plugin from the marketplace

```
claude plugin install <plugin-name>@niranjan94-claude-marketplace --scope user
```

For example, to install agent-skills:

```
claude plugin install agent-skills@niranjan94-claude-marketplace --scope user
```

## Structure

```
.claude-plugin/
  marketplace.json    # Marketplace manifest listing available plugins
plugins/
  agent-skills/       # Personal collection of skills for use with AI Agents
```

## Plugins

| Plugin | Source | Description |
|--------|--------|-------------|
| agent-skills | [plugins/agent-skills](./plugins/agent-skills) | Personal collection of skills for use with AI Agents |
| xquik-x-data | [plugins/xquik-x-data](./plugins/xquik-x-data) | Xquik REST API and remote MCP workflows for X data, exports, monitoring, and webhooks |

## Configuration

The marketplace is defined in `.claude-plugin/marketplace.json`:

```json
{
    "name": "personal-claude-marketplace",
    "owner": {
        "name": "Niranjan Rajendran"
    },
    "plugins": [
        {
            "name": "agent-skills",
            "source": "plugins/agent-skills",
            "description": "My personal collection of skills I use with AI Agents"
        }
    ]
}
```

To add a new plugin, append an entry to the `plugins` array with a `name`, `source` (path within this repo or GitHub `owner/repo`), and `description`.

## License

MIT
