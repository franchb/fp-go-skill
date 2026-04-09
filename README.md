# fp-go Skill for Claude Code

Active functional programming guidance for Go projects using [fp-go/v2](https://github.com/IBM/fp-go).

## What it does

- **Auto-detect**: When your project uses fp-go/v2, Claude applies correct conventions automatically
- **Monad selection**: Interactive decision tree to pick the right type (Option, Result, IOResult, ReaderIOResult, Effect)
- **Code review**: Catches fp-go anti-patterns — wrong monad choice, imperative style, missing composition
- **Migration**: Step-by-step conversion of imperative Go to fp-go pipelines
- **Deep reference**: Layered docs from compact cheatsheet to full API inventory (5,262 functions across 61 packages)

## Install

```bash
claude plugin add github:franchb/fp-go-skill
```

## Usage

Invoke with `/fp-go` in any Claude Code session, or just start working on a Go project that imports `github.com/IBM/fp-go/v2` — the skill activates automatically.

### Modes

| Command | What it does |
|---------|-------------|
| `/fp-go` | Load fp-go conventions into the session |
| "which monad should I use?" | Interactive monad selection wizard |
| "review my fp-go code" | Check for anti-patterns and suggest improvements |
| "convert this to fp-go" | Step-by-step migration from imperative Go |

## Documentation Layers

The skill loads a compact reference (~260 lines) by default and reads deeper docs on demand:

| File | Lines | When loaded |
|------|-------|-------------|
| SKILL.md | ~300 | Always (compact reference + guidance logic) |
| cookbook.md | 1,198 | Migration tasks, "how do I X?" |
| core-patterns.md | 1,598 | Type details, operations, composition |
| mastery.md | 1,437 | Advanced: Do-notation, DI, transformers, profunctors |
| full-reference.md | 6,080 | API lookup across all 61 packages |

## Requirements

- Go 1.24+ (for generic type aliases)
- [fp-go/v2](https://github.com/IBM/fp-go/v2) in your project

## License

Apache-2.0
