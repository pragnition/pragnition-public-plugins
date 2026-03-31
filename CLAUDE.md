# pragnition-plugins -- Marketplace Maintenance Guide

This is the central Claude Code plugin marketplace for Pragnition Labs.
It lives at `pragnition/pragnition-public-plugins` on GitHub and serves as the catalog that
users add with `/plugin marketplace add pragnition/pragnition-public-plugins`.

## Repository Structure

```
.claude-plugin/
  marketplace.json        # The marketplace catalog -- THE source of truth
documentation/
  <plugin-name>.md        # Human-readable docs for each plugin
PLUGINS.md                # Public catalog table of all plugins
README.md                 # User-facing setup and install instructions
CLAUDE.md                 # This file -- maintenance instructions
LICENSE
```

Plugins are **not bundled** in this repo. Each plugin lives in its own GitHub repo
(e.g., `pragnition/RAPID`) and is referenced by the marketplace catalog.

## Adding a New Plugin

When asked to add a plugin to this marketplace, follow these steps exactly.

### Step 1: Read the plugin's DOCS.md

Every plugin repo ships a `DOCS.md` file at its root. Fetch it first:

```
gh api repos/pragnition/<plugin-name>/contents/DOCS.md --jq '.content' | base64 -d
```

If that fails, read it locally if the repo is cloned, or use `WebFetch` against the
raw GitHub URL. You need this file -- it contains the full command reference, state
machine, directory structure, and architecture notes that you will distill into the
user-facing documentation.

Also read the plugin's `.claude-plugin/plugin.json` for name, version, description,
keywords, and license. And read its `README.md` for install instructions and quick-start.

### Step 2: Update marketplace.json

Add an entry to the `plugins` array in `.claude-plugin/marketplace.json`:

```json
{
  "name": "<plugin-name>",
  "source": {
    "source": "github",
    "repo": "pragnition/<plugin-name>"
  },
  "description": "<one-line description from plugin.json or DOCS.md>",
  "version": "<version from plugin.json>",
  "author": {
    "name": "Pragnition Labs"
  },
  "homepage": "https://github.com/pragnition/<plugin-name>",
  "repository": "https://github.com/pragnition/<plugin-name>",
  "license": "<license from plugin.json>",
  "category": "<single category word>",
  "tags": ["<from plugin.json keywords>"]
}
```

Required fields: `name`, `source`, `description`, `version`.
Always include: `author`, `homepage`, `repository`, `license`, `category`, `tags`.

### Step 3: Create documentation/\<plugin-name\>.md

Write human-readable documentation distilled from the plugin's DOCS.md. Follow this
template structure:

```markdown
# <Plugin Name>

<One paragraph description of what the plugin does and why you'd use it.>

## Install

<Marketplace install command and any alternative methods.>

## Requirements

<Runtime requirements -- Node.js version, Claude Code version, plan requirements, etc.>

## Quick Start

<The minimum commands to go from install to result, with a concrete example topic.>

## Commands

<For each command/skill the plugin provides:>

### `/<command-name>`

<What it does, what it expects, what it produces. Keep it practical.>

## Configuration

<Any config files, depth flags, environment variables, etc.>

## File Structure

<What the plugin writes to disk and where.>

## Tips

<Practical tips distilled from DOCS.md -- things like resumability, re-running steps,
editing intermediate files, cost awareness, etc.>
```

Guidelines for writing documentation:
- Write for a user who has never seen the plugin before.
- Use concrete examples (real topic strings, real command invocations).
- Document every command/skill the plugin exposes.
- Include the file structure the plugin creates on disk.
- Note any requirements (Node.js version, Claude Code version, plan tier).
- Keep it factual -- pull from DOCS.md, don't invent features.
- No emojis.

### Step 4: Update PLUGINS.md

Add a row to the plugin catalog table in `PLUGINS.md`:

```markdown
| [<plugin-name>](documentation/<plugin-name>.md) | <short description> | <version> | [Repo](https://github.com/pragnition/<plugin-name>) |
```

### Step 5: Update README.md

Add the plugin to the "Available Plugins" table in `README.md` following the existing
format.

### Step 6: Commit and push

Stage all changed files and commit with a message like:

```
Add <plugin-name> plugin to marketplace

- marketplace.json: add catalog entry
- documentation/<plugin-name>.md: user-facing docs
- PLUGINS.md: add to catalog table
- README.md: add to available plugins
```

Push to `main`.

## Updating an Existing Plugin

When a plugin bumps its version or changes its docs:

1. Re-read the plugin's DOCS.md and plugin.json for changes.
2. Update the `version` in marketplace.json.
3. Update `documentation/<plugin-name>.md` if commands or behavior changed.
4. Update the version in the PLUGINS.md table.
5. Commit and push.

## Rules

- Never bundle plugin source code in this repo. Plugins are always external GitHub repos.
- Every plugin MUST have a corresponding `documentation/<plugin-name>.md` file.
- Every plugin MUST have a row in `PLUGINS.md`.
- Every plugin MUST have an entry in `README.md`'s Available Plugins table.
- Keep marketplace.json as the single source of truth for plugin metadata.
- Validate JSON before committing: `python3 -m json.tool .claude-plugin/marketplace.json`
