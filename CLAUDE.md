# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A collection of Claude Code plugins (skills) for software development. The repository acts as a plugin marketplace (`/.claude-plugin/marketplace.json`) containing individual plugins under `plugins/`.

Currently the only plugin is **springboot-hexagonal-builder** (v1.2.0), which provides eleven skills and three agents:

**Skills:**
- **hexagonal-architecture-builder** — Scaffolds reactive Spring Boot 3.4.1 / Java 21 / WebFlux microservices with Hexagonal Architecture using JBang
- **java-development-best-practices** — Reviews, refactors, and generates Java code applying SOLID, Clean Code (KISS/DRY/YAGNI), and GoF patterns
- **java-testing-architect** — Generates, reviews, and refactors tests for Java microservices following hexagonal architecture testing best practices
- **c4-architecture** — Generates C4 model architecture diagrams in Mermaid syntax
- **srs-document-builder** — Generates Software Requirements Specification (SRS/ERS) documents based on IEEE 830 for the requirements analysis phase
- **relational-db-schema-builder** — Generates relational database schema documentation, ER diagrams, DDL scripts, and data dictionaries applying normalization and design best practices
- **nosql-schema-builder** — Designs and documents NoSQL database schemas (MongoDB, DynamoDB, Cassandra), collection structures, document modeling, JSON Schema validations, and indexing strategies
- **openapi-doc-builder** — Generates API documentation using OpenAPI 3.x/Swagger specification, including YAML specs, endpoint references, and integration guides
- **planning** — Guides interactive requirement gathering for development tasks
- **sdlc-workflow-supervisor** — Orchestrates the full SDLC workflow from raw client requirements to architectural design, enforcing quality gates and traceability between phases. Accepts client requirements as argument.
- **microservices-eda-architecture** — Designs microservices architectures and Event-Driven Architecture (EDA) systems from business requirements. Covers domain decomposition, event identification, communication patterns (choreography/orchestration), data consistency patterns (Saga, CQRS, Event Sourcing, Outbox), and resilience design (Circuit Breaker, DLQ, idempotency, distributed tracing).

**Agents:**
- **requirements-analyst** — Autonomous agent for project planning, requirements analysis, and PRD generation. Produces a PRD and then invokes `/srs-document-builder` to generate the formal IEEE 830 SRS document
- **software-architect-lead** — Elite Software Architect & Tech Lead agent for architectural design, technical decision-making, solution design, code review, RFC creation, stack selection, and technical documentation
- **backend-java-developer** — Elite Backend Java Developer agent for implementing reactive Spring Boot microservices based on design documentation. Handles initial development from design specs, bug fixes, new functionality, and requirement changes. Execution arm of Stage 4 (Development) in the SDLC workflow. Enforces a mandatory compilation verification loop (`mvn clean compile`) after each microservice — fixes errors iteratively until the build is clean. Unit and integration tests are obligatory and must strictly follow the testing documentation from the design phase (`docs/design/testing/`); runs `mvn clean verify` in a loop until all tests pass.

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

## Adding a New Agent to an Existing Plugin

1. Create `plugins/<plugin-name>/agents/<agent-name>.md` with YAML frontmatter and instructions
2. Required frontmatter fields: `name`, `description` (activation triggers — same importance as SKILL.md), `model` (sonnet/opus/haiku), `color`, `memory` (project/user/none)
3. The `description` field must include example user prompts and assistant responses showing when to invoke the agent
4. Agents can invoke skills internally via `Skill(skill: "<skill-name>")` and other agents via the Agent tool

## Agent Workflow

The `sdlc-workflow-supervisor` skill orchestrates all three agents sequentially with quality gates and formal INPUT/OUTPUT contracts between phases:

```
Client Requirements
        ↓
┌───────────────────────────────────────────────────────────────┐
│  sdlc-workflow-supervisor (orchestrator skill)                │
│                                                               │
│  ESTADO 0: Ingesta ─────────────────────────────────────────┐ │
│  │  INPUT:  raw requirements                                │ │
│  │  Agent:  requirements-analyst                            │ │
│  │  OUTPUT: docs/prd/PRD-<name>.md + docs/srs/SRS-<name>.md│ │
│  ├──────────────────────────────────────────────────────────┘ │
│  │                                                            │
│  ESTADO 1: Quality Gate (supervisor validates PRD+SRS)        │
│  │  Score < 10/10 → re-invokes requirements-analyst with gaps │
│  ├────────────────────────────────────────────────────────────┤
│  │                                                            │
│  ESTADO 2: Design Hand-off ─────────────────────────────────┐ │
│  │  INPUT:  PRD + SRS paths + design roadmap                │ │
│  │  Agent:  software-architect-lead                         │ │
│  │  OUTPUT: docs/design/ (architecture/, c4/, openapi/,     │ │
│  │          database/, scaffold/, testing/, events/)         │ │
│  ├──────────────────────────────────────────────────────────┘ │
│  │                                                            │
│  ESTADO 3: Traceability Audit (RF-XXX ↔ design components)   │
│  │                                                            │
│  ESTADO 4: Development ─────────────────────────────────────┐ │
│  │  INPUT:  docs/design/ + PRD + SRS                        │ │
│  │  Agent:  backend-java-developer                          │ │
│  │  OUTPUT: source code + docs/development/*                │ │
│  ├──────────────────────────────────────────────────────────┘ │
│  │                                                            │
│  ESTADO 5: Architectural Review ────────────────────────────┐ │
│  │  Agent:  software-architect-lead (reviewer)              │ │
│  │  APPROVED → ESTADO 6                                     │ │
│  │  REJECTED → REVIEW-CORRECTIONS.md → re-invoke ESTADO 4  │ │
│  │  Max 3 review cycles                                     │ │
│  ├──────────────────────────────────────────────────────────┘ │
│  │                                                            │
│  ESTADO 6: Execution Report                                   │
│  │  OUTPUT: docs/sdlc-report/SDLC-EXECUTION-REPORT.md        │
└───────────────────────────────────────────────────────────────┘
```

**Agent handoff contracts:**
- `requirements-analyst` receives raw requirements → outputs PRD + SRS files
- `software-architect-lead` receives explicit paths to PRD + SRS + a synthesized design roadmap → outputs all design artifacts
- `backend-java-developer` receives design artifacts + PRD + SRS → outputs source code + development deliverables
- `software-architect-lead` (reviewer) receives code + development docs + design reference → outputs approval or REVIEW-CORRECTIONS.md
- The supervisor verifies file existence at each transition before proceeding

**Document output locations:**
- PRD → `docs/prd/PRD-<project-name>.md`
- SRS → `docs/srs/SRS-<project-name>.md`
- Design deliverables → `docs/design/` (architecture/, openapi/, events/, scaffold/, database/, testing/, c4/)
- Development deliverables → `docs/development/` (TEST-REPORT, SERVICE-GUIDE, CURL-EXAMPLES, LOCAL-TOOLS, DIAGRAMS, TECH-STACK, openapi/)
- Review corrections → `docs/development/REVIEW-CORRECTIONS.md` (when architectural review rejects)
- SDLC Execution Report → `docs/sdlc-report/SDLC-EXECUTION-REPORT.md`

## Key Conventions

- SKILL.md `description` field is critical — it determines when Claude Code activates the skill. Write it as a comprehensive trigger list covering both English and Spanish trigger phrases.
- SKILL.md YAML frontmatter requires `name` and `description`. The markdown body contains the full prompt/instructions.
- **Version sync**: When bumping the plugin version, update both `plugins/<name>/.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` — they must match.
- The hexagonal scaffold supports `--database` (postgres, mongo) and `--messaging-system` (rabbit-producer, rabbit-consumer, none). Unsupported technologies trigger Phase 3 in the skill (manual config + scaffold extension).
- The repository language is primarily Spanish (README, comments) but skills are written in English.
- Prerequisites for scaffold usage: Java 17+, JBang, Maven.
- **Never run `mvn checkstyle`**, `mvn pmd:check`, `mvn spotbugs:check`, or any static analysis command after generating code unless the user explicitly asks.
- **Never create `package-info.java` files** — they are not used in this project. Document generated classes with inline Javadoc on the class declaration instead.

## Package Structure Conventions (hexagonal projects)

All generated code follows a consistent sub-package structure inside each Maven module. Never place all classes in the root package of a module.

| Module | Class type | Sub-package | Example |
|--------|-----------|-------------|---------|
| `domain/model` | Domain entity | `entities` | `com.<n>.model.entities.<Entity>` |
| `domain/model` | Output port interface | `ports` | `com.<n>.model.ports.<Entity>Repository` |
| `domain/model` | Enumeration | `enums` | `com.<n>.model.enums.<Entity>Status` |
| `domain/model` | Domain event | `events` | `com.<n>.model.events.<Entity>CreatedEvent` |
| `domain/model` | Domain exception | `exceptions` | `com.<n>.model.exceptions.<Entity>NotFoundException` |
| `domain/model` | Value object | `valueobjects` | `com.<n>.model.valueobjects.Email` |
| `application/use-cases` | Input port (interface) | _(root)_ | `com.<n>.usecases.<Entity>UseCase` |
| `application/use-cases` | Implementation | `impl` | `com.<n>.usecases.impl.<Entity>UseCaseImpl` |
| `driven-adapters/postgres` | R2DBC entity | `entities` | `com.<n>.postgres.entities.<Entity>Data` |
| `driven-adapters/postgres` | Spring Data repo | `repositories` | `com.<n>.postgres.repositories.<Entity>R2dbcRepository` |
| `driven-adapters/postgres` | Adapter (port impl) | `adapters` | `com.<n>.postgres.adapters.<Entity>RepositoryAdapter` |
| `driven-adapters/mongo` | Document entity | `entities` | `com.<n>.mongo.entities.<Entity>Document` |
| `driven-adapters/mongo` | Reactive repo | `repositories` | `com.<n>.mongo.repositories.<Entity>MongoRepository` |
| `driven-adapters/mongo` | Adapter | `adapters` | `com.<n>.mongo.adapters.<Entity>RepositoryAdapter` |
| `driven-adapters/rabbit-producer` | Spring config | `config` | `com.<n>.rabbitproducer.config.RabbitMQConfig` |
| `driven-adapters/rabbit-producer` | Publisher adapter | `adapters` | `com.<n>.rabbitproducer.adapters.RabbitMQMessagePublisher` |
| `entry-points/rabbit-consumer` | Spring config | `config` | `com.<n>.rabbitconsumer.config.RabbitMQConfig` |
| `entry-points/rest-api` | Controller | _(root)_ | `com.<n>.restapi.<Entity>Controller` |
| `entry-points/rest-api` | Request/Response DTO | `dto` | `com.<n>.restapi.dto.<Entity>Request` |
| `entry-points/app` | Spring config | `config` | `com.<n>.app.config.BeanConfig` |

**Rules:**
- Domain classes **never** go in the root `com.<n>.model` package — always in a sub-package
- Domain entities → `entities/`; ports → `ports/`; enums → `enums/`; events → `events/`; exceptions → `exceptions/`; value objects → `valueobjects/`
- Interface + implementation → implementation always in `impl/`
- `@Configuration` classes → always in `config/`
- `@Table` / `@Document` classes (infra) → `entities/` in their adapter module
- Spring Data repository interfaces → always in `repositories/`
- Classes that `implements` a domain port → always in `adapters/`
- Request/Response/DTO classes → always in `dto/`
- Non-trivial entity↔domain mappers → `mappers/`
- Only create sub-packages that are actually needed

## Approach
- Think before acting. Read existing files before writing code.
- Be concise in output but thorough in reasoning.
- Prefer editing over rewriting whole files.
- Do not re-read files you have already read unless the file may have changed.
- Test your code before declaring done.
- No sycophantic openers or closing fluff.
- Keep solutions simple and direct.
- User instructions always override this file.
