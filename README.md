# agent-skills

A plugin marketplace for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that adds specialized workflows and domain knowledge.

## Available plugins

| Plugin | Skills | What it does |
|--------|--------|-------------|
| humanize-skills | [humanize](skills/humanize) | Detects and rewrites 24 categories of AI-generated text patterns. Based on Wikipedia's "Signs of AI writing" guide. |

## Installation

### From the marketplace

```
/plugin marketplace add skizha/agent-skills
/plugin install humanize-skills@skizha-skills
```

The skill will be available in your next Claude Code session. Invoke it with `/humanize` or by asking Claude to humanize text.

### Local testing

Load the repo directly as a plugin directory:

```bash
claude --plugin-dir /path/to/agent-skills
```

## Usage

Skills trigger automatically based on your prompt. For example, the humanize skill activates when you say things like:

- "humanize this text"
- "make this sound less robotic"
- "remove AI writing patterns from this file"

You can also invoke it directly with `/humanize`.

## Marketplace structure

```
agent-skills/
├── .claude-plugin/
│   └── marketplace.json    # marketplace manifest
└── skills/
    └── skill-name/
        ├── SKILL.md           # required -- instructions and metadata
        ├── references/        # optional -- detailed docs loaded on demand
        ├── scripts/           # optional -- executable code
        └── assets/            # optional -- templates, images, etc.
```

The `marketplace.json` defines plugins, each of which bundles one or more skills. Each skill lives under `skills/` with at least a `SKILL.md` containing YAML frontmatter (`description` field) and markdown instructions.

## Contributing

To add a new skill:

1. Create a folder under `skills/` with a `SKILL.md` file
2. Add your plugin entry to `.claude-plugin/marketplace.json` with a `name`, `description`, and `skills` array pointing to your skill folders
3. Submit a PR

## License

MIT
