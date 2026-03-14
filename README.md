# Postman Collection Skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-2.0.0-blue.svg)](.claude-plugin/marketplace.json)
[![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-Skill-blueviolet.svg)](https://docs.anthropic.com/en/docs/claude-code)
[![Laravel](https://img.shields.io/badge/Laravel-FF2D20?logo=laravel&logoColor=white)](#)
[![Postman](https://img.shields.io/badge/Postman-FF6C37?logo=postman&logoColor=white)](#)
[![GitHub Stars](https://img.shields.io/github/stars/bakrpx/postman-collection-skill?style=flat&logo=github)](https://github.com/bakrpx/postman-collection-skill)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

A [Claude Code skill](https://docs.anthropic.com/en/docs/claude-code) that teaches AI agents to create, structure, test, and sync Postman collections for Laravel APIs.

## Installation

### Claude Code Marketplace

```
/plugin marketplace add bakrpx/postman-collection-skill
/plugin install postman-collection@bakrpx
```

### npm

```bash
npx skills add bakrpx/postman-collection-skill
```

### Laravel Boost

```bash
php artisan boost:add-skill bakrpx/postman-collection-skill
```

### Manual

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

## Usage

The skill activates automatically on relevant prompts:

> Generate a Postman collection for my Laravel API routes
> Add Newman to my CI/CD pipeline
> Create test assertions for pagination responses
> Set up Postman cloud sync with ID preservation
> Show me auth patterns for Sanctum in Postman

## Skill Structure

```
skills/
├── SKILL.md
└── references/
    ├── auth-patterns.md
    ├── negative-testing.md
    └── variable-chaining.md
```

## Triggers

The skill activates on: `postman collection`, `newman run`, `postman tests`, `collection variables`, `response examples`, `postman auth`, `postman cloud sync`, `pre-request script`, `postman ci`.

## License

MIT
