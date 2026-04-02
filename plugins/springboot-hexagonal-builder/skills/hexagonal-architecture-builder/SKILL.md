---
name: hexagonal-architecture-builder
description: "Generate reactive Spring Boot microservices with Hexagonal Architecture (Ports and Adapters) using JBang scaffolding. Use this skill whenever the user wants to create a new microservice, generate a Spring Boot project, scaffold a hexagonal architecture project, set up a reactive service with any database (PostgreSQL, MongoDB, MySQL, SQL Server, Redis, etc.), any messaging system (RabbitMQ, Kafka, SQS, etc.), or mentions creating a new backend service in Java. Also use it when the user wants to add new domain entities, use cases, ports, or adapters to an already-generated hexagonal project. If the user requests a technology not natively supported by the scaffold, this skill guides you to configure it manually in the project AND add support to MavenHexagonalScaffold.java for future generations."
---

# Hexagonal Architecture Builder

Generate and extend reactive Spring Boot microservices following the Hexagonal Architecture pattern (Ports and Adapters).

## When This Skill Activates

- User wants to **create** a new microservice from scratch
- User wants to **add** domain entities, use cases, ports, or adapters to an existing hexagonal project
- User mentions hexagonal architecture, ports and adapters, or reactive Spring Boot services
- User requests a database, messaging system, or infrastructure tool **not currently supported** by the scaffold (e.g., MySQL, SQL Server, Redis, Kafka, SQS, etc.)

## Phase 1: Scaffold a New Microservice

### Step 1: Parse Arguments

Extract from the user's request:

| Argument | Required | Natively Supported | Default |
|----------|----------|-------------------|---------|
| `--service-name` / `-n` | Yes | kebab-case string (e.g., `ms-users`) | — |
| `--database` / `-d` | Yes | `postgres`, `mongo` | `postgres` |
| `--messaging-system` / `-m` | No | `rabbit-producer`, `rabbit-consumer`, `none` | `none` |

If the user says something like "create a users service with MongoDB", infer: `--service-name=ms-users --database=mongo`.

If `--service-name` is missing, ask for it — don't guess.

**If the user requests a technology NOT in the natively supported column** (e.g., MySQL, Redis, Kafka, SQS), proceed with Phase 1 using the closest supported option, then jump to **Phase 3** to configure the unsupported technology in the project and add it to the scaffold.

### Step 2: Validate Prerequisites

Before running the scaffold, verify the environment:

```bash
java -version    # Needs Java 17+
jbang --version  # Needs JBang installed
```

If JBang is missing, guide the user:
```bash
curl -Ls https://sh.jbang.dev | bash -s - app setup
```

Also check that the target directory doesn't already exist to avoid overwriting.

### Step 3: Run the Scaffold

The scaffold script is bundled with this plugin. Locate it relative to the plugin's installation directory:

```bash
# The script is at <plugin-dir>/templates/jbang/MavenHexagonalScaffold.java
# Use the PLUGIN_DIR environment variable or find it relative to this skill file
jbang <plugin-dir>/templates/jbang/MavenHexagonalScaffold.java \
  --service-name=<name> \
  --database=<db> \
  --messaging-system=<messaging>
```

**Important:** `<plugin-dir>` is the root directory of this plugin (where `.claude-plugin/plugin.json` lives). When running the scaffold, resolve the full path to `MavenHexagonalScaffold.java` before executing. It generates the full multi-module Maven project in the user's current working directory.

### Step 4: Verify Generation

After running, confirm the project was created correctly:

```bash
ls -la <service-name>/
cat <service-name>/pom.xml
```

Check that all expected modules exist based on the chosen options.

### Step 5: Explain What Was Generated

Give the user a concise summary of the generated structure:

```
<service-name>/
├── pom.xml                              # Parent POM (Spring Boot 3.4.1, Java 21)
├── .env                                 # Environment variables (credentials, connections) — NOT committed to git
├── .env.example                         # Template without secrets — committed to git for team reference
├── .gitignore                           # Includes .env exclusion
├── domain/model/                        # Pure domain — no framework dependencies
├── application/use-cases/               # Business logic orchestration
├── infrastructure/
│   ├── driven-adapters/
│   │   ├── <postgres|mongo>/            # Persistence adapter
│   │   └── rabbit-producer/             # (if selected)
│   └── entry-points/
│       ├── rest-api/                    # REST controllers (WebFlux)
│       ├── app/                         # MainApplication + config (depends on rest-api)
│       │   └── src/main/resources/
│       │       └── application.yml      # Uses ${ENV_VARS} placeholders resolved from .env
│       └── rabbit-consumer/             # (if selected)
```

**Dependency flow:** rest-api/entry-points → use-cases → domain ← driven-adapters (app wires everything)

Explain that `domain/model` is the core — it has zero framework dependencies. Everything else depends inward toward it.

#### Environment Variables (.env)

The project uses **spring-dotenv** (`me.paulschwarz:spring-dotenv`) to load environment variables from a `.env` file at the project root. The `application.yml` references variables using `${VARIABLE_NAME}` syntax.

**Generated .env variables** (vary by chosen database/messaging):

| Variable | Description | Default Value |
|----------|-------------|---------------|
| `SERVER_PORT` | Application server port | `8080` |
| `R2DBC_URL` | R2DBC connection URL (PostgreSQL) | `r2dbc:postgresql://localhost:5432/mydb` |
| `DB_USERNAME` | Database username (PostgreSQL) | `postgres` |
| `DB_PASSWORD` | Database password (PostgreSQL) | `password` |
| `MONGODB_URI` | MongoDB connection URI (MongoDB) | `mongodb://localhost:27017/mydb` |
| `RABBITMQ_HOST` | RabbitMQ host (if messaging selected) | `localhost` |
| `RABBITMQ_PORT` | RabbitMQ port (if messaging selected) | `5672` |
| `RABBITMQ_USERNAME` | RabbitMQ username (if messaging selected) | `guest` |
| `RABBITMQ_PASSWORD` | RabbitMQ password (if messaging selected) | `guest` |

**Key rules:**
- `.env` is in `.gitignore` — never committed to version control
- `.env.example` IS committed — serves as documentation for the team (credentials left blank)
- When adding new infrastructure in Phase 2 or Phase 3, always add the corresponding variables to `.env`, `.env.example`, and reference them in `application.yml` with `${VARIABLE_NAME}`

### Step 6: Offer Next Steps

After scaffolding, ask the user what they want to build. Typical next steps:

- "Want me to add a domain entity (e.g., User, Order, Product)?"
- "Should I create a use case with its port interfaces?"
- "Want to set up the repository adapter for your entity?"

Then proceed to **Phase 2**.

---

## Phase 2: Extend the Generated Project

This is where the real value lives — helping the user add actual business logic following hexagonal architecture rules.

### Naming Conventions

The scaffold uses these naming patterns (important to follow them when adding code):

- **Group ID**: `com.<servicename>` with hyphens removed (e.g., `ms-users` → `com.msusers`)
- **Module packages**: `com.<safename>.<modulename>` (e.g., `com.msusers.model`, `com.msusers.usecases`)
- **MainApplication**: lives in `com.<safename>` package (one level above module packages) for component scanning

### Domain Layer — Package Structure

The `domain/model` module is organized into sub-packages by type. Never place all domain classes in the root package.

```
domain/model/src/main/java/com/<safename>/model/
├── entities/
│   └── <Entity>.java                  # Domain entities — pure Java, no framework
├── ports/
│   ├── <Entity>Repository.java        # Output port (persistence)
│   └── <Entity>EventPublisher.java    # Output port (messaging/events)
├── enums/
│   └── <Entity>Status.java            # Domain enumerations
├── events/
│   └── <Entity>CreatedEvent.java      # Domain events
├── exceptions/
│   └── <Entity>NotFoundException.java # Domain-specific exceptions
└── valueobjects/
    └── Email.java                     # Value objects (immutable, self-validating)
```

Use only the sub-packages your domain actually needs — don't create empty `events/` or `valueobjects/` directories upfront.

### Adding a Domain Entity

Create in `domain/model/src/main/java/com/<safename>/model/entities/`:

```java
package com.<safename>.model.entities;

import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class <Entity> {
    private String id;
    // domain fields — no Spring, no persistence annotations here
}
```

The domain entity must remain pure: no Spring annotations, no `@Table`, no `@Document`. It defines the business truth.

### Adding a Domain Enum

Create in `domain/model/src/main/java/com/<safename>/model/enums/`:

```java
package com.<safename>.model.enums;

public enum <Entity>Status {
    ACTIVE, INACTIVE, PENDING
}
```

### Adding a Domain Event

Create in `domain/model/src/main/java/com/<safename>/model/events/`:

```java
package com.<safename>.model.events;

import lombok.Builder;
import lombok.Value;
import java.time.Instant;

@Value
@Builder
public class <Entity>CreatedEvent {
    String entityId;
    Instant occurredAt;
    // relevant fields — immutable snapshot of what happened
}
```

### Adding a Domain Exception

Create in `domain/model/src/main/java/com/<safename>/model/exceptions/`:

```java
package com.<safename>.model.exceptions;

public class <Entity>NotFoundException extends RuntimeException {
    public <Entity>NotFoundException(String id) {
        super("<Entity> not found with id: " + id);
    }
}
```

### Adding a Value Object

Create in `domain/model/src/main/java/com/<safename>/model/valueobjects/`:

```java
package com.<safename>.model.valueobjects;

import lombok.Value;

@Value  // immutable by design
public class Email {
    String value;

    public Email(String value) {
        if (value == null || !value.contains("@"))
            throw new IllegalArgumentException("Invalid email: " + value);
        this.value = value.toLowerCase();
    }
}
```

Use value objects when a primitive carries business rules (validation, formatting, equality by value).

### Adding a Port (Interface)

Ports are interfaces that define how the domain communicates with the outside world.

**Output port** (domain declares what it needs from infrastructure):
Create in `domain/model/src/main/java/com/<safename>/model/ports/`:

```java
package com.<safename>.model.ports;

import com.<safename>.model.entities.<Entity>;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

public interface <Entity>Repository {
    Mono<<Entity>> findById(String id);
    Flux<<Entity>> findAll();
    Mono<<Entity>> save(<Entity> entity);
    Mono<Void> deleteById(String id);
}
```

```java
package com.<safename>.model.ports;

import com.<safename>.model.events.<Entity>CreatedEvent;
import reactor.core.publisher.Mono;

public interface <Entity>EventPublisher {
    Mono<Void> publish(<Entity>CreatedEvent event);
}
```

**Input port** (what the application offers to entry points):
Create in `application/use-cases/src/main/java/com/<safename>/usecases/`:

```java
package com.<safename>.usecases;

import com.<safename>.model.entities.<Entity>;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

public interface <Entity>UseCase {
    Mono<<Entity>> getById(String id);
    Flux<<Entity>> getAll();
    Mono<<Entity>> create(<Entity> entity);
    Mono<Void> delete(String id);
}
```

### Adding a Use Case (Implementation)

Implementations of use case interfaces go in an `impl` sub-package.

Create in `application/use-cases/src/main/java/com/<safename>/usecases/impl/`:

```java
package com.<safename>.usecases.impl;

import com.<safename>.model.entities.<Entity>;
import com.<safename>.model.ports.<Entity>Repository;
import com.<safename>.usecases.<Entity>UseCase;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@Service
@RequiredArgsConstructor
public class <Entity>UseCaseImpl implements <Entity>UseCase {

    private final <Entity>Repository repository;

    @Override
    public Mono<<Entity>> getById(String id) {
        return repository.findById(id);
    }

    @Override
    public Flux<<Entity>> getAll() {
        return repository.findAll();
    }

    @Override
    public Mono<<Entity>> create(<Entity> entity) {
        return repository.save(entity);
    }

    @Override
    public Mono<Void> delete(String id) {
        return repository.deleteById(id);
    }
}
```

The use case depends on the **port interface** (`<Entity>Repository`), not on any concrete adapter. This is the Dependency Inversion principle at work.

**Important:** Add the domain model dependency to the use-cases module POM:
```xml
<dependency>
    <groupId>com.<safename>.model</groupId>
    <artifactId>domain-model</artifactId>
    <version>${project.version}</version>
</dependency>
```

### Adding a Driven Adapter (Repository Implementation)

Each driven adapter module is organized into sub-packages by concern. This keeps entities, Spring Data repositories, and adapter implementations clearly separated.

**For PostgreSQL (R2DBC):**

```
infrastructure/driven-adapters/postgres/src/main/java/com/<safename>/postgres/
├── entities/
│   └── <Entity>Data.java           # R2DBC entity (infrastructure concern)
├── repositories/
│   └── <Entity>R2dbcRepository.java # Spring Data R2DBC interface
├── adapters/
│   └── <Entity>RepositoryAdapter.java # Implements domain port
└── mappers/
    └── <Entity>Mapper.java         # Domain ↔ Data mapping (if non-trivial)
```

```java
// entities/<Entity>Data.java
package com.<safename>.postgres.entities;

import org.springframework.data.annotation.Id;
import org.springframework.data.relational.core.mapping.Table;
import lombok.Data;

@Data
@Table("<table_name>")
public class <Entity>Data {
    @Id
    private String id;
    // mapped fields
}

// repositories/<Entity>R2dbcRepository.java
package com.<safename>.postgres.repositories;

import com.<safename>.postgres.entities.<Entity>Data;
import org.springframework.data.repository.reactive.ReactiveCrudRepository;

public interface <Entity>R2dbcRepository extends ReactiveCrudRepository<<Entity>Data, String> {
}

// adapters/<Entity>RepositoryAdapter.java
package com.<safename>.postgres.adapters;

import com.<safename>.model.entities.<Entity>;
import com.<safename>.model.ports.<Entity>Repository;
import com.<safename>.postgres.entities.<Entity>Data;
import com.<safename>.postgres.repositories.<Entity>R2dbcRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Repository;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@Repository
@RequiredArgsConstructor
public class <Entity>RepositoryAdapter implements <Entity>Repository {

    private final <Entity>R2dbcRepository r2dbcRepository;

    @Override
    public Mono<<Entity>> findById(String id) {
        return r2dbcRepository.findById(id).map(this::toDomain);
    }

    @Override
    public Flux<<Entity>> findAll() {
        return r2dbcRepository.findAll().map(this::toDomain);
    }

    @Override
    public Mono<<Entity>> save(<Entity> entity) {
        return r2dbcRepository.save(toData(entity)).map(this::toDomain);
    }

    @Override
    public Mono<Void> deleteById(String id) {
        return r2dbcRepository.deleteById(id);
    }

    private <Entity> toDomain(<Entity>Data data) { /* mapping */ }
    private <Entity>Data toData(<Entity> entity) { /* mapping */ }
}
```

**For MongoDB:**
Same sub-package structure but use `@Document` instead of `@Table`, and `ReactiveMongoRepository` instead of `ReactiveCrudRepository`:

```
infrastructure/driven-adapters/mongo/src/main/java/com/<safename>/mongo/
├── entities/
│   └── <Entity>Document.java       # MongoDB document (use @Document)
├── repositories/
│   └── <Entity>MongoRepository.java # ReactiveMongoRepository interface
├── adapters/
│   └── <Entity>RepositoryAdapter.java
└── mappers/
    └── <Entity>Mapper.java         # (optional, if mapping is complex)
```

**Important:** Add the domain model dependency to the adapter module POM:
```xml
<dependency>
    <groupId>com.<safename>.model</groupId>
    <artifactId>domain-model</artifactId>
    <version>${project.version}</version>
</dependency>
```

### Adding a REST Controller (Entry Point)

The rest-api module is organized with DTOs in a `dto` sub-package, keeping the controller clean and the domain entity unexposed:

```
infrastructure/entry-points/rest-api/src/main/java/com/<safename>/restapi/
├── <Entity>Controller.java         # @RestController
└── dto/
    ├── <Entity>Request.java        # Incoming payload (validated with Bean Validation)
    └── <Entity>Response.java       # Outgoing payload (never expose domain entity directly)
```

```java
// dto/<Entity>Request.java
package com.<safename>.restapi.dto;

import jakarta.validation.constraints.NotBlank;
import lombok.Data;

@Data
public class <Entity>Request {
    @NotBlank
    private String name;
    // request fields with validation annotations
}

// dto/<Entity>Response.java
package com.<safename>.restapi.dto;

import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class <Entity>Response {
    private String id;
    private String name;
    // response fields — only what the client needs
}

// <Entity>Controller.java
package com.<safename>.restapi;

import com.<safename>.restapi.dto.<Entity>Request;
import com.<safename>.restapi.dto.<Entity>Response;
import com.<safename>.usecases.<Entity>UseCase;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

@RestController
@RequestMapping("/<entities>")
@RequiredArgsConstructor
public class <Entity>Controller {

    private final <Entity>UseCase useCase;

    @GetMapping("/{id}")
    public Mono<<Entity>Response> getById(@PathVariable String id) {
        return useCase.getById(id).map(this::toResponse);
    }

    @GetMapping
    public Flux<<Entity>Response> getAll() {
        return useCase.getAll().map(this::toResponse);
    }

    @PostMapping
    public Mono<<Entity>Response> create(@RequestBody <Entity>Request request) {
        return useCase.create(toDomain(request)).map(this::toResponse);
    }

    @DeleteMapping("/{id}")
    public Mono<Void> delete(@PathVariable String id) {
        return useCase.delete(id);
    }

    private <Entity>Response toResponse(<Entity> entity) { /* mapping */ }
    private <Entity> toDomain(<Entity>Request request) { /* mapping */ }
}
```

**Important:** Add use-cases dependency to the rest-api module POM, and add driven-adapter dependencies to the app module POM:
```xml
<!-- In rest-api/pom.xml -->
<dependency>
    <groupId>com.<safename>.usecases</groupId>
    <artifactId>application-use-cases</artifactId>
    <version>${project.version}</version>
</dependency>

<!-- In app/pom.xml (rest-api is already a dependency) -->
<dependency>
    <groupId>com.<safename>.<db></groupId>
    <artifactId>infrastructure-driven-adapters-<db></artifactId>
    <version>${project.version}</version>
</dependency>
```

### Adding RabbitMQ Integration

**Producer** (driven-adapter — the application sends messages outward):
The scaffold generates `RabbitMQConfig.java` and `MessagePublisher.java`. These are organized into sub-packages:

```
infrastructure/driven-adapters/rabbit-producer/src/main/java/com/<safename>/rabbitproducer/
├── config/
│   └── RabbitMQConfig.java         # @Configuration — declares exchanges, queues, bindings
└── adapters/
    └── RabbitMQMessagePublisher.java # Implements domain port (EventPublisher)
```

To add custom messages, create a port interface in the domain and implement it in the `adapters/` sub-package following the same pattern as repository adapters.

**Consumer** (entry-point — messages arrive from outside into the application):

```
infrastructure/entry-points/rabbit-consumer/src/main/java/com/<safename>/rabbitconsumer/
├── config/
│   └── RabbitMQConfig.java         # @Configuration — listener container factory, bindings
└── MessageListener.java            # @RabbitListener — delegates to a use case
```

The listener should delegate to a use case, just like a REST controller would.

### Package Conventions (Always Apply)

Every module follows a consistent sub-package structure. Never dump all classes in the root package of a module.

| Module | Class type | Sub-package | Example |
|--------|-----------|-------------|---------|
| `domain/model` | Domain entity | `entities` | `com.<n>.model.entities.<Entity>` |
| `domain/model` | Output port interface | `ports` | `com.<n>.model.ports.<Entity>Repository` |
| `domain/model` | Enumeration | `enums` | `com.<n>.model.enums.<Entity>Status` |
| `domain/model` | Domain event | `events` | `com.<n>.model.events.<Entity>CreatedEvent` |
| `domain/model` | Domain exception | `exceptions` | `com.<n>.model.exceptions.<Entity>NotFoundException` |
| `domain/model` | Value object | `valueobjects` | `com.<n>.model.valueobjects.Email` |
| `application/use-cases` | Input port (interface) | _(root)_ | `com.<n>.usecases.<Entity>UseCase` |
| `application/use-cases` | Use case implementation | `impl` | `com.<n>.usecases.impl.<Entity>UseCaseImpl` |
| `driven-adapters/postgres` | R2DBC entity | `entities` | `com.<n>.postgres.entities.<Entity>Data` |
| `driven-adapters/postgres` | Spring Data repo | `repositories` | `com.<n>.postgres.repositories.<Entity>R2dbcRepository` |
| `driven-adapters/postgres` | Adapter (port impl) | `adapters` | `com.<n>.postgres.adapters.<Entity>RepositoryAdapter` |
| `driven-adapters/postgres` | Entity↔domain mapper | `mappers` | `com.<n>.postgres.mappers.<Entity>Mapper` |
| `driven-adapters/mongo` | Document entity | `entities` | `com.<n>.mongo.entities.<Entity>Document` |
| `driven-adapters/mongo` | Reactive repo | `repositories` | `com.<n>.mongo.repositories.<Entity>MongoRepository` |
| `driven-adapters/mongo` | Adapter | `adapters` | `com.<n>.mongo.adapters.<Entity>RepositoryAdapter` |
| `driven-adapters/rabbit-producer` | Spring config | `config` | `com.<n>.rabbitproducer.config.RabbitMQConfig` |
| `driven-adapters/rabbit-producer` | Publisher adapter | `adapters` | `com.<n>.rabbitproducer.adapters.RabbitMQMessagePublisher` |
| `entry-points/rabbit-consumer` | Spring config | `config` | `com.<n>.rabbitconsumer.config.RabbitMQConfig` |
| `entry-points/rest-api` | Controller | _(root)_ | `com.<n>.restapi.<Entity>Controller` |
| `entry-points/rest-api` | Request/Response DTO | `dto` | `com.<n>.restapi.dto.<Entity>Request` |
| `entry-points/app` | Spring config | `config` | `com.<n>.app.config.BeanConfig` |

**Rules of thumb:**
- Domain classes **always** go in a sub-package — never in the root `com.<n>.model` package
- Output port interfaces (repository, publisher) → `ports/`
- Domain entities → `entities/`; enumerations → `enums/`; events → `events/`; exceptions → `exceptions/`; value objects → `valueobjects/`
- If you create an interface + implementation in the same module, the implementation goes in `impl/`
- `@Configuration` classes → always in `config/`
- `@Table` / `@Document` infrastructure entities → `entities/` in their adapter module
- Spring Data repository interfaces → `repositories/`
- Classes that `implements` a domain port → `adapters/`
- Request/Response/DTO classes → `dto/`
- Non-trivial entity↔domain mappers (> 3 fields or computed logic) → `mappers/`
- Only create sub-packages you actually need — don't pre-create empty `events/` or `valueobjects/`

### Dependency Rules (Never Violate These)

The hexagonal architecture enforces strict dependency direction. This is the most important thing to get right:

```
domain/model         → depends on NOTHING (pure Java + Reactor)
application/use-cases → depends on domain/model only
driven-adapters/*     → depends on domain/model (implements its ports)
entry-points/app      → depends on use-cases + driven-adapters (wires everything)
```

- Domain NEVER imports from `org.springframework` (except transitively via Reactor)
- Use cases depend on port interfaces, NEVER on adapter implementations
- Entry points depend on use case interfaces, NEVER on domain ports directly
- Only the app module (entry-points/app) wires adapters to ports via Spring DI

### Database Modifications via MCP (When Available)

During microservice development, if an MCP database server is connected, you can modify the database schema directly when needed. This is useful when:

- Adding a new domain entity that requires a new table or collection
- A use case requires altering an existing table/collection (adding columns/fields, constraints, indexes)
- The driven adapter needs database objects that don't exist yet (e.g., missing table/collection for a new repository)

#### Supabase (PostgreSQL projects)

If the Supabase MCP server is connected and the user's project uses a Supabase-hosted PostgreSQL database:

1. **Detect the need** — When creating a driven adapter (repository) for an entity and the corresponding table doesn't exist in Supabase, inform the user:
   > "The table `<table_name>` doesn't exist yet in Supabase. Would you like me to create it?"

2. **Generate and show the DDL** — Based on the domain entity and the adapter's `@Table` mapping, generate the CREATE TABLE statement. Show it to the user before executing.

3. **Execute on approval** — Use the Supabase MCP tools to run the DDL against the project database.

4. **Keep docs updated** — If a `docs/database/` directory exists with schema documentation, update it to reflect the changes.

#### MongoDB (MongoDB projects)

If the MongoDB MCP server is connected and the user's project uses MongoDB:

1. **Detect the need** — When creating a driven adapter (repository) for an entity and the corresponding collection doesn't exist, inform the user:
   > "The collection `<collection_name>` doesn't exist yet in MongoDB. Would you like me to create it?"

2. **Generate and show the commands** — Based on the domain entity and the adapter's `@Document` mapping, generate the collection creation command along with any indexes or validation schemas. Show them to the user before executing.

3. **Execute on approval** — Use the MongoDB MCP tools to create the collection, indexes, and validation rules against the target database.

4. **Keep docs updated** — If a `docs/database/` directory exists with schema documentation, update it to reflect the changes.

**Important rules (apply to both Supabase and MongoDB):**
- **Never modify the database without asking** — Always show the SQL/commands and get explicit approval
- **Prefer additive changes** — CREATE TABLE/collection, ADD COLUMN/field, CREATE INDEX. For destructive changes (DROP, ALTER column type, remove fields), warn the user about potential data loss
- **Respect the schema design** — If `relational-db-schema-builder` or `nosql-schema-builder` was used to design the model, follow that design as the source of truth. Only suggest deviations if there's a technical reason

### Build and Run

```bash
cd <service-name>
mvn clean install                                         # Build all modules
mvn spring-boot:run -pl infrastructure/entry-points/app   # Run the application
```

The app starts on port 8080 by default. The starter `/hello` endpoint verifies everything works.

### What NOT to Do After Generating Code

- **Never run `mvn checkstyle`** or any static analysis tool (`pmd`, `spotbugs`, etc.) unless the user explicitly asks for it.
- **Never create `package-info.java` files** in any module. They are not part of this project's conventions.
- Instead, document generated classes using **inline Javadoc comments** directly on the class and its public methods. Every generated class must have at least a one-line Javadoc on the class declaration explaining its role in the architecture. Example:

```java
/**
 * Domain entity representing a customer in the system.
 * Pure Java — no framework dependencies.
 */
@Data
@Builder
public class Customer { ... }

/**
 * Output port defining the persistence contract for {@link Customer}.
 * Implemented by the driven adapter in the infrastructure layer.
 */
public interface CustomerRepository { ... }

/**
 * R2DBC persistence entity mapped to the {@code customers} table.
 * Infrastructure concern — never exposed outside the postgres adapter module.
 */
@Data
@Table("customers")
public class CustomerData { ... }

/**
 * Implements {@link CustomerRepository} using Spring Data R2DBC.
 * Maps between {@link CustomerData} (infrastructure) and {@link Customer} (domain).
 */
@Repository
@RequiredArgsConstructor
public class CustomerRepositoryAdapter implements CustomerRepository { ... }
```

## Examples

### Full scaffold + entity workflow

```
User: "Create a users microservice with PostgreSQL"
→ Run scaffold: --service-name=ms-users --database=postgres
→ Offer to add User entity
→ Create User.java               in domain/model        → com.msusers.model.entities
→ Create UserRepository.java     in domain/model        → com.msusers.model.ports (output port)
→ Create UserStatus.java         in domain/model        → com.msusers.model.enums (if needed)
→ Create UserNotFoundException   in domain/model        → com.msusers.model.exceptions (if needed)
→ Create UserUseCase.java        in application/use-cases → com.msusers.usecases (input port)
→ Create UserUseCaseImpl.java    in application/use-cases → com.msusers.usecases.impl
→ Create UserData.java           in driven-adapters/postgres → com.msusers.postgres.entities
→ Create UserR2dbcRepository.java                       → com.msusers.postgres.repositories
→ Create UserRepositoryAdapter.java                     → com.msusers.postgres.adapters
→ Create UserRequest.java / UserResponse.java           → com.msusers.restapi.dto
→ Create UserController.java     in entry-points/rest-api → com.msusers.restapi
→ Update POMs with inter-module dependencies
→ Build and verify
```

### Scaffold with MongoDB and RabbitMQ

```bash
jbang templates/jbang/MavenHexagonalScaffold.java \
  --service-name=ms-notifications \
  --database=mongo \
  --messaging-system=rabbit-consumer
```

---

## Phase 3: Handle Unsupported Technologies (Extend the Scaffold)

When a user requests a database, messaging system, or infrastructure tool that `MavenHexagonalScaffold.java` does **not** currently support, you must do **two things**:

1. **Configure it in the user's project** — create the module, POM, Java files, and YAML config manually
2. **Add native support to `MavenHexagonalScaffold.java`** — so future scaffold runs can generate it with a CLI flag

### Currently Supported by the Scaffold

| Type | Supported Values |
|------|-----------------|
| `--database` | `postgres`, `mongo` |
| `--messaging-system` | `rabbit-producer`, `rabbit-consumer`, `none` |

Anything outside this table triggers Phase 3.

### Step 1: Identify the Technology Type

Classify the requested technology:

| Category | Examples | Module Location |
|----------|----------|----------------|
| Relational DB (R2DBC) | MySQL, SQL Server, MariaDB, Oracle | `infrastructure/driven-adapters/<db-name>` |
| NoSQL DB | Redis, Cassandra, DynamoDB, Elasticsearch | `infrastructure/driven-adapters/<db-name>` |
| Message Producer | Kafka producer, SQS producer, SNS | `infrastructure/driven-adapters/<name>-producer` |
| Message Consumer | Kafka consumer, SQS consumer | `infrastructure/entry-points/<name>-consumer` |
| External Service | gRPC client, REST client, SMTP | `infrastructure/driven-adapters/<name>` |

### Step 2: Configure in the User's Project

Follow the same hexagonal architecture patterns used by the existing modules. Each new module needs:

#### 2a. Create the module directory structure

```
<service-name>/infrastructure/driven-adapters/<new-tech>/
├── pom.xml
└── src/main/java/com/<safename>/<techmodule>/
    ├── <Tech>Config.java          # Spring @Configuration
    └── <Tech>Adapter.java         # Implements domain port (if applicable)
```

For entry-point modules (consumers):
```
<service-name>/infrastructure/entry-points/<new-tech>-consumer/
├── pom.xml
└── src/main/java/com/<safename>/<techmodule>/
    ├── <Tech>Config.java
    └── <Tech>Listener.java
```

#### 2b. Create the module POM

Follow the same pattern as existing modules. Key points:
- `<parent>` points to root POM with correct `<relativePath>` (`../../../pom.xml` for infrastructure modules)
- `<groupId>` follows `com.<safename>.<modulename>` convention
- `<artifactId>` follows `infrastructure-driven-adapters-<name>` convention
- Include the specific Spring Boot starters and driver dependencies for the technology

#### 2c. Reference table for common dependencies

**Databases:**

| Technology | Spring Boot Starter | Driver Dependency | Reactive? |
|-----------|-------------------|-------------------|-----------|
| MySQL (R2DBC) | `spring-boot-starter-data-r2dbc` | `io.asyncer:r2dbc-mysql` (runtime) | Yes |
| SQL Server (R2DBC) | `spring-boot-starter-data-r2dbc` | `io.r2dbc:r2dbc-mssql` (runtime) | Yes |
| MariaDB (R2DBC) | `spring-boot-starter-data-r2dbc` | `org.mariadb:r2dbc-mariadb` (runtime) | Yes |
| Redis | `spring-boot-starter-data-redis-reactive` | — | Yes |
| Cassandra | `spring-boot-starter-data-cassandra-reactive` | — | Yes |
| Elasticsearch | `spring-boot-starter-data-elasticsearch` | — | No (wrap with Mono) |

**Messaging:**

| Technology | Dependencies | Notes |
|-----------|-------------|-------|
| Kafka Producer | `spring-kafka` | Use `ReactiveKafkaProducerTemplate` from `reactor-kafka` |
| Kafka Consumer | `spring-kafka`, `io.projectreactor.kafka:reactor-kafka` | Use `@KafkaListener` or `ReactiveKafkaConsumerTemplate` |
| SQS Producer | `io.awspring.cloud:spring-cloud-aws-starter-sqs` | Use `SqsTemplate` wrapped in Mono |
| SQS Consumer | `io.awspring.cloud:spring-cloud-aws-starter-sqs` | Use `@SqsListener` |

#### 2d. Create Java configuration and adapter classes

Follow the same patterns as the existing RabbitMQ modules. The configuration class should:
- Be annotated with `@Configuration`
- Define necessary beans (templates, factories, converters)
- Use property values from `application.yml` via `@Value` or `@ConfigurationProperties`

The adapter class should:
- Implement a domain port interface (if applicable)
- Use reactive types (`Mono`, `Flux`)
- Follow the naming convention: `<Tech>Adapter`, `<Tech>Publisher`, or `<Tech>Listener`

#### 2e. Update `application.yml`

Add the corresponding configuration block to `infrastructure/entry-points/app/src/main/resources/application.yml`. **Always use `${VARIABLE_NAME}` placeholders** — never hardcode credentials or connection strings.

Examples:
```yaml
# MySQL R2DBC
spring:
  r2dbc:
    url: ${R2DBC_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

# Redis
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT}
      password: ${REDIS_PASSWORD}

# Kafka
spring:
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      group-id: ${spring.application.name}
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer

# SQS (AWS)
spring:
  cloud:
    aws:
      region:
        static: ${AWS_REGION}
      credentials:
        access-key: ${AWS_ACCESS_KEY}
        secret-key: ${AWS_SECRET_KEY}
      sqs:
        endpoint: ${AWS_SQS_ENDPOINT}
```

#### 2f. Update `.env` and `.env.example`

For every new `${VARIABLE_NAME}` added to `application.yml`, add the corresponding entry to both files at the project root:

- **`.env`**: Add the variable with a sensible local/development default value
- **`.env.example`**: Add the variable with the default value for non-sensitive settings, and leave the value blank for credentials/secrets

Example for Redis:
```bash
# .env
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# .env.example
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=
```

#### 2g. Register the module in root `pom.xml`

Add the new `<module>` entry in the `<modules>` section of the root `pom.xml`.

#### 2h. Add dependency in `entry-points/app` POM

The app module must depend on the new module so Spring Boot wires it at runtime:
```xml
<dependency>
    <groupId>com.<safename>.<techmodule></groupId>
    <artifactId>infrastructure-driven-adapters-<tech-name></artifactId>
    <version>${project.version}</version>
</dependency>
```

### Step 3: Add Support to MavenHexagonalScaffold.java

After configuring the project, update `templates/jbang/MavenHexagonalScaffold.java` so the scaffold can generate this technology natively in future runs. Follow these sub-steps:

#### 3a. Extend the CLI option validation

The `--database` or `--messaging-system` option description already documents valid values. Add the new value to the description string.

For a new **database** (e.g., `mysql`):
- Add the new value to the `@Option` description of `--database`

For a new **messaging system** (e.g., `kafka-producer`):
- Add the new value to the `@Option` description of `--messaging-system`

For a completely new technology category that is neither database nor messaging:
- Add a new `@Option` with a descriptive name (e.g., `--cache`, `--external-service`)

#### 3b. Update `run()` method — module resolution

In the `run()` method, update the module list logic. Currently:
```java
String dbAdapterModule = "mongo".equalsIgnoreCase(database)
        ? "infrastructure/driven-adapters/mongo"
        : "infrastructure/driven-adapters/postgres";
```

Add the new database as another condition:
```java
String dbAdapterModule;
if ("mongo".equalsIgnoreCase(database)) {
    dbAdapterModule = "infrastructure/driven-adapters/mongo";
} else if ("<new-db>".equalsIgnoreCase(database)) {
    dbAdapterModule = "infrastructure/driven-adapters/<new-db>";
} else {
    dbAdapterModule = "infrastructure/driven-adapters/postgres";
}
```

For new messaging systems, add new `else if` branches alongside the existing RabbitMQ checks.

#### 3c. Update `getModulePomTemplate()` — dependencies

Add a new `else if` branch for the new module path with its specific Maven dependencies. Follow the exact same pattern as the existing `postgres` and `mongo` blocks.

#### 3d. Update `getYamlContent()` — application.yml generation

Add the YAML configuration block for the new technology using `${VARIABLE_NAME}` placeholders. Follow the pattern of the existing database and messaging conditions. **Never hardcode credentials or connection strings.**

#### 3e. Update `getEnvContent()` and `getEnvExampleContent()` — .env generation

Add the corresponding environment variables for the new technology. In `getEnvContent()` include sensible development defaults. In `getEnvExampleContent()` leave credential values blank. Follow the same conditional pattern used in `getYamlContent()`.

#### 3f. Create file generation method (if the technology needs config/adapter classes)

If the technology requires boilerplate Java files (like RabbitMQ does), create a new method following the pattern of `createRabbitProducerFiles()` or `createRabbitConsumerFiles()`:

```java
private void create<Tech>Files(Path rootPath, String safeProjectName) throws IOException {
    String modulePath = "infrastructure/driven-adapters/<tech-name>";
    String moduleName = "<techname>";  // no hyphens
    String basePackage = "com." + safeProjectName + "." + moduleName;
    String packagePath = "/src/main/java/" + basePackage.replace(".", "/");

    // Config class
    String configClass = "package " + basePackage + ";\n\n" + ...;
    Files.writeString(rootPath.resolve(modulePath + packagePath + "/<Tech>Config.java"), configClass);

    // Adapter/Publisher/Listener class
    String adapter = "package " + basePackage + ";\n\n" + ...;
    Files.writeString(rootPath.resolve(modulePath + packagePath + "/<Tech>Adapter.java"), adapter);
}
```

Call this method from `run()` after the module creation loop, following the same pattern as the RabbitMQ file creation calls.

#### 3g. Update `getRootPomTemplate()` — module listing

Add the conditional `<module>` inclusion for the new technology in the `getRootPomTemplate()` method, following the existing pattern for RabbitMQ modules.

### Step 4: Verify Everything Compiles

After both changes (project + scaffold), verify:

```bash
# 1. Verify the user's project compiles
cd <service-name> && mvn clean install -DskipTests

# 2. Verify the scaffold generates correctly with the new option
jbang templates/jbang/MavenHexagonalScaffold.java -n test-<tech> -d <new-db> -m <new-msg>
cd test-<tech> && mvn clean install -DskipTests && cd .. && rm -rf test-<tech>
```

### Step 5: Update CLAUDE.md

After adding a new technology to the scaffold, update `CLAUDE.md` to reflect:
- New valid values in the Arguments table
- Any new key methods added to `MavenHexagonalScaffold.java`

### Examples

#### User requests MySQL

```
User: "Create a users service with MySQL"
→ Scaffold supports: postgres, mongo — MySQL is NOT supported
→ Phase 3 activates:
  1. Run scaffold with --database=postgres (closest match for structure)
  2. Replace postgres module with mysql module in the generated project:
     - Create infrastructure/driven-adapters/mysql/ with R2DBC MySQL deps
     - Update application.yml with ${R2DBC_URL}, ${DB_USERNAME}, ${DB_PASSWORD} (reuse same vars, change default URL)
     - Update .env and .env.example with r2dbc:mysql://localhost:3306/mydb default
     - Update root pom.xml module list
     - Remove postgres module
  3. Update MavenHexagonalScaffold.java:
     - Add "mysql" to --database description
     - Add mysql branch in run(), getModulePomTemplate(), getYamlContent(), getEnvContent(), getEnvExampleContent(), getRootPomTemplate()
  4. Verify both the project and a fresh scaffold generation compile
```

#### User requests Kafka producer

```
User: "I need a microservice with Kafka for publishing events"
→ Scaffold supports: rabbit-producer, rabbit-consumer — Kafka is NOT supported
→ Phase 3 activates:
  1. Run scaffold with desired DB and --messaging-system=none
  2. Manually create infrastructure/driven-adapters/kafka-producer/ module:
     - POM with spring-kafka and reactor-kafka dependencies
     - KafkaConfig.java with ProducerFactory and ReactiveKafkaProducerTemplate beans
     - KafkaMessagePublisher.java implementing a domain port
     - Update application.yml with ${KAFKA_BOOTSTRAP_SERVERS} placeholder
     - Update .env and .env.example with KAFKA_BOOTSTRAP_SERVERS=localhost:9092
     - Register module in root pom.xml and app module dependencies
  3. Update MavenHexagonalScaffold.java:
     - Add "kafka-producer" to --messaging-system description
     - Add kafka-producer branch in run(), getModulePomTemplate(), getYamlContent(), getEnvContent(), getEnvExampleContent(), getRootPomTemplate()
     - Create createKafkaProducerFiles() method
  4. Verify compilation
```

#### User requests Redis as cache alongside PostgreSQL

```
User: "Add Redis caching to my existing ms-users project"
→ This is an ADDITIONAL infrastructure module, not a replacement
→ Phase 3 activates:
  1. Create infrastructure/driven-adapters/redis/ module in the existing project:
     - POM with spring-boot-starter-data-redis-reactive
     - RedisConfig.java with ReactiveRedisTemplate bean
     - RedisCacheAdapter.java
     - Update application.yml with ${REDIS_HOST}, ${REDIS_PORT}, ${REDIS_PASSWORD} placeholders
     - Update .env and .env.example with Redis variables (host=localhost, port=6379, password blank)
     - Register in root pom.xml and app module dependencies
  2. Update MavenHexagonalScaffold.java:
     - Add a new --cache option (since Redis is not a primary DB replacement here)
     - OR add "redis" to --database if user intends it as primary data store
  3. Verify compilation
```
