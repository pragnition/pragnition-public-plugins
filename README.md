# Pragnition Labs Plugin Marketplace

Official Claude Code plugin marketplace by [Pragnition Labs](https://github.com/pragnition).

## Setup

Add this marketplace to Claude Code:

```
/plugin marketplace add pragnition/pragnition-public-plugins
```

## Available Plugins

See the full [Plugin Catalog](PLUGINS.md) for details and documentation links.

| Plugin | Description | Version |
|--------|-------------|---------|
| [rapid](documentation/rapid.md) | Team-based parallel development with isolated worktrees, interface contracts, adversarial review, and intelligent merge | 6.0.0 |

### Install a plugin

```
/plugin install <plugin-name>@pragnition-plugins
```

For example:

```
/plugin install rapid@pragnition-plugins
```

## For Teams

Add to your project's `.claude/settings.json` to make the marketplace available to all team members:

```json
{
  "extraKnownMarketplaces": {
    "pragnition-plugins": {
      "source": {
        "source": "github",
        "repo": "pragnition/pragnition-public-plugins"
      }
    }
  }
}
```

## Contributing

To add a plugin to this marketplace, add an entry to `.claude-plugin/marketplace.json` with:
- `name` -- plugin identifier
- `source` -- GitHub repo reference (`{ "source": "github", "repo": "owner/repo" }`)
- `description` -- what the plugin does
- `version` -- semver version string
- `tags` -- searchable keywords

The plugin repo must contain a `.claude-plugin/plugin.json` manifest, a `DOCS.md` file
at its root, and the standard plugin directory structure (`skills/`, `agents/`, etc.).

See `CLAUDE.md` for detailed maintenance instructions.
