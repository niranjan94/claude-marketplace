# personal-claude-marketplace

A personal plugin marketplace for Claude. Serves as a registry of plugins and skills for use with AI agents.

## Installation

```
/plugin marketplace add niranjan94/claude-marketplace
```

## Structure

```
.claude-plugin/
  marketplace.json    # Marketplace manifest listing available plugins
```

## Plugins

| Plugin | Source | Description |
|--------|--------|-------------|
| agent-skills | [niranjan94/agent-skills](https://github.com/niranjan94/agent-skills) | Personal collection of skills for use with AI Agents |

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
            "source": "niranjan94/agent-skills",
            "description": "My personal collection of skills I use with AI Agents"
        }
    ]
}
```

To add a new plugin, append an entry to the `plugins` array with a `name`, `source` (GitHub `owner/repo`), and `description`.

## License

MIT
