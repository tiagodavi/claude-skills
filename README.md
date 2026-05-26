# Claude Skills

A collection of [Claude](https://claude.com/claude-code) skills I use to keep my projects consistent, well-tested, and free of common pitfalls. Each skill is a self-contained set of guidelines that Claude loads on demand to shape how it writes and reviews code.

## What is a skill?

A skill is a `SKILL.md` file (plus any supporting reference docs) that tells Claude how to approach a specific kind of work — conventions to follow, anti-patterns to avoid, tools to run, and the bar for "done". Claude picks them up automatically when the task matches the skill's description.

## Available skills

### [elixir-react](./elixir-react)

Guidelines for full-stack projects built on **Elixir / Phoenix / Absinthe** on the backend and **TypeScript / React / Apollo Client** on the frontend.

Covers:

- TDD workflow (happy-path, failure-path, and edge-case tests required)
- Project layout for `jobs/`, `services/`, and `schemas/`
- GraphQL performance (DataLoader, N+1 elimination)
- Ecto query readability and migration safety
- Use of protocols, behaviours, and Elixir concurrency
- React/Apollo cache and UX considerations
- An [Elixir anti-patterns reference](./elixir-react/references/elixir-anti-patterns-reference.md) and a [SOLID React reference](./elixir-react/references/solid-react-reference.md) loaded before coding

## Installation

Skills live under `~/.claude/skills/`. To install one:

```bash
git clone https://github.com/tiagodavi/skills.git
ln -s "$(pwd)/skills/elixir-react" ~/.claude/skills/elixir-react
```

Claude Code will auto-discover the skill on its next run.

## Contributing

New skills are welcome. Each skill should:

1. Live in its own top-level directory.
2. Contain a `SKILL.md` with YAML frontmatter (`name`, `description`, `metadata`).
3. Keep reference material in a `references/` subdirectory.
4. Be specific because a skill that triggers on everything triggers on nothing.

## License

MIT
