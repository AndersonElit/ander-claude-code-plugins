# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A collection of Claude Code plugins (skills) for software development. The repository acts as a plugin marketplace (`/.claude-plugin/marketplace.json`) containing individual plugins under `plugins/`.

Currently the only plugin is **springboot-hexagonal-builder** (v1.2.0), which provides eight skills and one agent:

**Skills:**
- **hexagonal-architecture-builder** — Scaffolds reactive Spring Boot 3.4.1 / Java 21 / WebFlux microservices with Hexagonal Architecture using JBang
- **java-development-best-practices** — Reviews, refactors, and generates Java code applying SOLID, Clean Code (KISS/DRY/YAGNI), and GoF patterns
- **c4-architecture** — Generates C4 model architecture diagrams in Mermaid syntax
- **srs-document-builder** — Generates Software Requirements Specification (SRS/ERS) documents based on IEEE 830 for the requirements analysis phase
- **relational-db-schema-builder** — Generates relational database schema documentation, ER diagrams, DDL scripts, and data dictionaries applying normalization and design best practices
- **nosql-schema-builder** — Designs and documents NoSQL database schemas (MongoDB, DynamoDB, Cassandra), collection structures, document modeling, JSON Schema validations, and indexing strategies
- **openapi-doc-builder** — Generates API documentation using OpenAPI 3.x/Swagger specification, including YAML specs, endpoint references, and integration guides
- **planning** — Guides interactive requirement gathering for development tasks

**Agents:**
- **requirements-analyst** — Autonomous agent for project planning, requirements analysis, and PRD generation. Produces a PRD and then invokes `/srs-document-builder` to generate the formal IEEE 830 SRS document

## Architecture

- **Marketplace layer**: Root `.claude-plugin/marketplace.json` registers plugins by name/version/source path.
- **Plugin layer**: Each plugin has `.claude-plugin/plugin.json` (name, description, version, author, mcpServers) and a `skills/` directory.
- **Skill layer**: Each skill is defined by a `SKILL.md` with YAML frontmatter (`name`, `description` used for activation matching) and markdown body containing the full prompt/instructions. Reference material lives in `references/` subdirectories and is loaded on demand.
- **MCP layer**: Plugins can bundle MCP servers in `plugin.json` under `mcpServers`. These start automatically when the plugin is enabled.
- **Agent layer**: Plugins can include autonomous agents under `agents/`. Each agent is a `.md` file with YAML frontmatter (`name`, `description`, `model`, `color`, `memory`) and a markdown body with full instructions. Agents are launched via the Agent tool and can invoke skills internally.
- **Templates**: The JBang scaffold (`MavenHexagonalScaffold.java`) is a self-contained Java script that generates multi-module Maven projects. It is invoked via `jbang <path>/MavenHexagonalScaffold.java --service-name=X --database=Y --messaging-system=Z`.

## MCP Servers

The **springboot-hexagonal-builder** plugin bundles two MCP servers in `plugin.json`. They start automatically when the plugin is enabled.

### Supabase MCP (`@supabase/mcp-server-supabase`)

Provides tools for interacting with Supabase projects (database management, auth, storage, edge functions, etc.).

**Prerequisites**: Node.js (for npx) and the environment variable `SUPABASE_ACCESS_TOKEN` set before launching Claude Code. Obtain the token from Supabase Dashboard > Account Settings > Access Tokens.

### MongoDB MCP (`@mongodb-js/mongodb-mcp-server`)

Provides tools for interacting with MongoDB instances (database and collection management, querying, indexing, etc.).

**Prerequisites**: Node.js (for npx) and the environment variable `MDB_MCP_CONNECTION_STRING` set before launching Claude Code. Example: `MDB_MCP_CONNECTION_STRING=mongodb://localhost:27017`.

### Cross-Skill Database Workflow

The intended cross-skill workflow with the MCP database servers is:

1. **`relational-db-schema-builder`** designs relational models (ER diagram + DDL + data dictionary) → offers to deploy to Supabase on user request (Step 6)
2. **`nosql-schema-builder`** designs NoSQL models (collection diagram + JSON Schema + indexes) → offers to deploy to MongoDB on user request (Step 6)
3. **`hexagonal-architecture-builder`** builds the microservice → when adding entities/adapters that need new or modified tables/collections, it can create/alter them in Supabase or MongoDB on user approval

Key rule: **never execute DDL/commands against Supabase or MongoDB without explicit user confirmation**. Always show the SQL/commands first.

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

- SKILL.md `description` field is critical — it determines when Claude Code activates the skill. Write it as a comprehensive trigger list covering both English and Spanish trigger phrases.
- SKILL.md YAML frontmatter requires `name` and `description`. The markdown body contains the full prompt/instructions.
- **Version sync**: When bumping the plugin version, update both `plugins/<name>/.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` — they must match.
- The hexagonal scaffold supports `--database` (postgres, mongo) and `--messaging-system` (rabbit-producer, rabbit-consumer, none). Unsupported technologies trigger Phase 3 in the skill (manual config + scaffold extension).
- The repository language is primarily Spanish (README, comments) but skills are written in English.
- Prerequisites for scaffold usage: Java 17+, JBang, Maven.
