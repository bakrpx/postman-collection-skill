# Postman Collection Skill

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code) that teaches AI agents to create, structure, test, and sync Postman collections for Laravel APIs.

## Installation

```bash
npx skills add bakrpx/postman-collection-skill
```

### Laravel Boost

```bash
php artisan boost:add-skill bakrpx/postman-collection-skill
```

Or clone manually:

```bash
git clone https://github.com/bakrpx/postman-collection-skill.git ~/.claude/skills/postman-collection-skill
```

## What It Does

This skill teaches AI agents how to:

- **Create** Postman collections from Laravel route definitions
- **Structure** collections with folder hierarchy, auth inheritance, and variable scoping
- **Test** with assertions for status codes, pagination, Content-Type, and variable chaining
- **Generate** response examples (success, error, 422 validation)
- **Produce** dynamic test data to prevent uniqueness violations on repeated runs
- **Configure** Newman for CI/CD pipeline testing
- **Sync** local collections with Postman cloud (with ID preservation)

## Skill Structure

```
postman-collection/
├── SKILL.md
└── references/
    └── auth-patterns.md
```

## Triggers

The skill activates on: `postman collection`, `newman run`, `postman tests`, `collection variables`, `response examples`, `postman auth`, `postman cloud sync`, `pre-request script`, `postman ci`.

## License

MIT
