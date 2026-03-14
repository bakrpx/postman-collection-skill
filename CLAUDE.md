# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Claude Code Skill** (not a Laravel/PHP package) that teaches AI agents how to create, structure, test, and sync Postman collections for Laravel APIs. It contains no executable code — only Markdown documentation and pattern guidance.

**Version**: 2.0.0
**Category**: productivity
**Installation**: `npx skills add bakrpx/postman-collection-skill`

## Repository Structure

```
skills/
├── SKILL.md                    # Primary skill definition (~400 line limit)
└── references/
    ├── auth-patterns.md        # Deep-dive auth reference (JWT, Sanctum, API Keys)
    ├── negative-testing.md     # Testing anti-patterns and pitfalls
    └── variable-chaining.md    # Dynamic variable chaining across requests

.claude-plugin/
└── marketplace.json            # Skill marketplace metadata (name, version, tags)
```

## Architecture

- **SKILL.md** is the main entry point. It contains frontmatter (`name`, `description`, trigger phrases), activation rules ("When to Apply"), and all Postman patterns (collection structure, test assertions, response examples, Newman CI/CD, cloud sync, anti-patterns).
- **Reference files** under `references/` provide deep dives on specific topics. Each file covers one topic.
- **marketplace.json** defines the skill's identity for the skills marketplace.

The skill activates when users mention trigger phrases like "postman collection", "newman run", "postman tests", "collection variables", etc.

## Contributing Rules

From CONTRIBUTING.md:
- Keep SKILL.md under 400 lines
- One topic per reference file
- All patterns must be generic — no project-specific data (URLs, credentials, model names)
- Reference files go in `skills/references/`

## Key Conventions

- Postman Collection v2.1 format is used exclusively
- `event.script.exec` values are arrays of strings (one line per element)
- Auth inheritance flows: Collection → Folder → Request
- Variable resolution order: Local > Collection > Environment > Global
- Secrets belong in environment files, never in collection variables
- Response example naming: `"<code> - <status>"` (e.g., "200 - OK")
