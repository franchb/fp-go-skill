# Contributing to fp-go-skill

## Repository Structure

```text
fp-go-skill/
├── .claude-plugin/
│   └── marketplace.json      # Root marketplace manifest
├── plugins/
│   └── fp-go/
│       ├── .claude-plugin/
│       │   └── plugin.json   # Per-plugin manifest
│       └── skills/
│           └── fp-go/        # Skill content files
└── .skill-hashes.sha256      # Integrity hashes
```

## Skill File Requirements

### Frontmatter

Every `SKILL.md` must include YAML frontmatter with:

```yaml
---
name: skill-name
description: >
  What the skill does and when it triggers.
allowed-tools: Read, Grep, Glob, LSP
---
```

**`allowed-tools` is mandatory.** Skills must use read-only tools only. No `Bash`, `Write`, `Edit`, or network-access tools.

### Layered Documentation

Skills use a layered docs pattern:

- **SKILL.md** (~300 lines max): Compact reference loaded into every session. Must be self-contained for common tasks.
- **Supporting files** (cookbook.md, core-patterns.md, etc.): Deeper docs loaded on demand when the user asks about specific topics.

Keep SKILL.md focused. Move detailed API references, advanced patterns, and migration guides to supporting files.

## Integrity Hashes

After modifying any skill file or plugin.json, regenerate `.skill-hashes.sha256`:

```bash
find . -name "SKILL.md" -not -path './.git/*' -exec sha256sum {} \; | sort > .skill-hashes.sha256
find . -path './plugins/*/skills/*/*.md' -not -path './.git/*' -exec sha256sum {} \; | sort >> .skill-hashes.sha256
find . -path './plugins/*/.claude-plugin/plugin.json' -exec sha256sum {} \; >> .skill-hashes.sha256
```

## CI Checks

Pull requests run these checks automatically:

- **Markdown lint**: All `.md` files checked against `.markdownlint.jsonc`
- **JSON validation**: All `.json` files validated for syntax
- **Hidden Unicode detection**: Scans changed `.md` and `.json` files for hidden characters
- **Prompt injection scan**: Flags suspicious patterns in changed `.md` files (manual review)
- **Tool restriction check**: Verifies every `SKILL.md` has `allowed-tools`
- **Integrity hash check**: Warns if `.skill-hashes.sha256` doesn't match current files
- **Secret scanning**: Gitleaks checks for leaked credentials
- **Dependency review**: Flags new dependencies with known vulnerabilities

## Markdown Style

Follow the rules in `.markdownlint.jsonc`:

- ATX-style headings (`# Heading`, not underline style)
- Dash list markers (`-`, not `*` or `+`)
- Blank lines around headings and fenced code blocks
- Language specified on all fenced code blocks

## Testing Locally

- Clone the repo
- From the **parent directory**, add as a local marketplace:
