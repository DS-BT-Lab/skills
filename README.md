# bt-lab Skills

AI skills by BT Lab for Claude Code and compatible agents.

## Available Skills

| Skill | Description | Standalone |
|-------|-------------|------------|
| `init-docs` | Generate Second Brain documentation for any codebase (Obsidian vault or plain Markdown) | Yes |
| `doc-research` | Deep-dive research into a codebase topic, producing new vault documents | No (requires init-docs) |
| `doc-full-research` | Orchestrate deep research across an entire project via sequential subagents | No (requires init-docs, doc-research) |
| `doc-update` | Update vault documentation after code changes using git diff analysis | No (requires init-docs) |

## Installation

### Claude Code Plugin (recommended)

```
/plugin marketplace add DS-BT-Labs/skills
/plugin install bt-lab@bt-lab
```

### npx skills -- documentation suite

The documentation skills work together. `doc-research`, `doc-full-research`, and `doc-update` depend on shared references and templates from `init-docs` (conventions, Diataxis guide, document templates). Install them together:

```bash
npx skills add DS-BT-Labs/skills --skill init-docs --skill doc-research --skill doc-full-research --skill doc-update -a claude-code
```

### npx skills -- all skills

```bash
npx skills add DS-BT-Labs/skills --skill '*' -a claude-code
```

### npx skills -- init-docs only

Only `init-docs` works as a standalone skill:

```bash
npx skills add DS-BT-Labs/skills --skill init-docs -a claude-code
```

> Installing `doc-research`, `doc-full-research`, or `doc-update` without `init-docs` will result in missing file errors -- these skills reference shared templates and conventions from the init-docs directory.

## Usage

### Plugin installation

Skills are namespaced with the plugin name:

```
/bt-lab:init-docs
/bt-lab:init-docs Focus on payment and auth modules
/bt-lab:doc-research Authentication and authorization flow
/bt-lab:doc-full-research --autonomous
/bt-lab:doc-update HN-210: auto-create sales orders from e-shop orders
```

### npx installation

Skills are available without namespace:

```
/init-docs
/doc-research State management architecture
/doc-full-research --autonomous
/doc-update --range HEAD~5..HEAD
```

## License

[MIT](LICENSE)
