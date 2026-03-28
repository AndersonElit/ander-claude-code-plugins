# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A collection of Claude Code plugins (skills) for software development. The repository acts as a plugin marketplace (`/.claude-plugin/marketplace.json`) containing individual plugins under `plugins/`.

Currently the only plugin is **springboot-hexagonal-builder**, which provides three skills:
- **hexagonal-architecture-builder** — Scaffolds reactive Spring Boot 3.4.1 / Java 21 / WebFlux microservices with Hexagonal Architecture using JBang
- **java-development-best-practices** — Reviews, refactors, and generates Java code applying SOLID, Clean Code (KISS/DRY/YAGNI), and GoF patterns
- **c4-architecture** — Generates C4 model architecture diagrams in Mermaid syntax

## Repository Structure

```
.claude-plugin/marketplace.json          # Marketplace index pointing to plugins
plugins/springboot-hexagonal-builder/
  .claude-plugin/plugin.json             # Plugin metadata
  skills/
    hexagonal-architecture-builder/SKILL.md
    java-development-best-practices/SKILL.md
      references/patterns-reference.md
    c4-architecture/SKILL.md
      references/                        # c4-syntax, common-mistakes, advanced-patterns
  templates/jbang/MavenHexagonalScaffold.java   # JBang scaffold generator
```

## Architecture

- **Marketplace layer**: Root `.claude-plugin/marketplace.json` registers plugins by name/version/source path.
- **Plugin layer**: Each plugin has `.claude-plugin/plugin.json` (name, description, version, author) and a `skills/` directory.
- **Skill layer**: Each skill is defined by a `SKILL.md` with YAML frontmatter (`name`, `description` used for activation matching) and markdown body containing the full prompt/instructions.
- **Templates**: The JBang scaffold (`MavenHexagonalScaffold.java`) is a self-contained Java script that generates multi-module Maven projects. It is invoked via `jbang <path>/MavenHexagonalScaffold.java --service-name=X --database=Y --messaging-system=Z`.

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
