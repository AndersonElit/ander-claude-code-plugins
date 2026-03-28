# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A collection of Claude Code plugins (skills) for software development. The repository acts as a plugin marketplace (`/.claude-plugin/marketplace.json`) containing individual plugins under `plugins/`.

Currently the only plugin is **springboot-hexagonal-builder** (v1.1.0), which provides six skills:
- **hexagonal-architecture-builder** — Scaffolds reactive Spring Boot 3.4.1 / Java 21 / WebFlux microservices with Hexagonal Architecture using JBang
- **java-development-best-practices** — Reviews, refactors, and generates Java code applying SOLID, Clean Code (KISS/DRY/YAGNI), and GoF patterns
- **c4-architecture** — Generates C4 model architecture diagrams in Mermaid syntax
- **srs-document-builder** — Generates Software Requirements Specification (SRS/ERS) documents based on IEEE 830 for the requirements analysis phase
- **relational-db-schema-builder** — Generates relational database schema documentation, ER diagrams, DDL scripts, and data dictionaries applying normalization and design best practices
- **openapi-doc-builder** — Generates API documentation using OpenAPI 3.x/Swagger specification, including YAML specs, endpoint references, and integration guides

## Architecture

- **Marketplace layer**: Root `.claude-plugin/marketplace.json` registers plugins by name/version/source path.
- **Plugin layer**: Each plugin has `.claude-plugin/plugin.json` (name, description, version, author, mcpServers) and a `skills/` directory.
- **Skill layer**: Each skill is defined by a `SKILL.md` with YAML frontmatter (`name`, `description` used for activation matching) and markdown body containing the full prompt/instructions. Reference material lives in `references/` subdirectories and is loaded on demand.
- **MCP layer**: Plugins can bundle MCP servers in `plugin.json` under `mcpServers`. These start automatically when the plugin is enabled.
- **Templates**: The JBang scaffold (`MavenHexagonalScaffold.java`) is a self-contained Java script that generates multi-module Maven projects. It is invoked via `jbang <path>/MavenHexagonalScaffold.java --service-name=X --database=Y --messaging-system=Z`.

## MCP Servers

The **springboot-hexagonal-builder** plugin bundles the Supabase MCP server (`@supabase/mcp-server`) in `plugin.json`. It starts automatically when the plugin is enabled and provides tools for interacting with Supabase projects (database management, auth, storage, edge functions, etc.).

**Prerequisites**: Node.js (for npx) and the environment variable `SUPABASE_ACCESS_TOKEN` set before launching Claude Code. Obtain the token from Supabase Dashboard > Account Settings > Access Tokens.

### Supabase Integration Workflow

The intended cross-skill workflow with Supabase is:

1. **`relational-db-schema-builder`** designs the full database model (ER diagram + DDL + data dictionary) → after generation, offers to deploy the schema to Supabase on user request (Step 6 in the skill)
2. **`hexagonal-architecture-builder`** builds the microservice → when adding entities/adapters that need new or modified tables, it can create/alter them in Supabase on user approval

Key rule: **never execute DDL against Supabase without explicit user confirmation**. Always show the SQL first.

## Testing Changes to the Plugin

```bash
claude --plugin-dir ./plugins/springboot-hexagonal-builder
```

## Adding a New Plugin

1. Create `plugins/<plugin-name>/.claude-plugin/plugin.json` with name, description, version, author
2. Add skills under `plugins/<plugin-name>/skills/<skill-name>/SKILL.md`
3. Register in `.claude-plugin/marketplace.json` under the `plugins` array

## Adding a New Skill to an Existing Plugin

1. Create `plugins/<plugin-name>/skills/<skill-name>/SKILL.md` with YAML frontmatter (`name`, `description`) and instructions
2. Place reference material in `references/` subdirectory if needed — skills load these on demand

## Key Conventions

- SKILL.md `description` field is critical — it determines when Claude Code activates the skill. Write it as a comprehensive trigger list.
- The hexagonal scaffold supports `--database` (postgres, mongo) and `--messaging-system` (rabbit-producer, rabbit-consumer, none). Unsupported technologies trigger Phase 3 in the skill (manual config + scaffold extension).
- The repository language is primarily Spanish (README, comments) but skills are written in English.
- Prerequisites for scaffold usage: Java 17+, JBang, Maven.
