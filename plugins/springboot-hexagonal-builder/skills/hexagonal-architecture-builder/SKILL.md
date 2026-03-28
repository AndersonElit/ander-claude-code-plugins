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

### Adding a Domain Entity

Create in `domain/model/src/main/java/com/<safename>/model/`:

```java
package com.<safename>.model;

import lombok.Builder;
import lombok.Data;

@Data
@Builder
public class <Entity> {
    private String id;
    // domain fields — no framework annotations here
}
```

The domain model must remain pure: no Spring annotations, no persistence annotations, no framework dependencies. It defines the business truth.

### Adding a Port (Interface)

Ports are interfaces that define how the domain communicates with the outside world. They live in the **domain** or **application** layer.

**Output port** (domain defines what it needs from infrastructure):
Create in `domain/model/src/main/java/com/<safename>/model/`:

```java
package com.<safename>.model;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;

public interface <Entity>Repository {
    Mono<<Entity>> findById(String id);
    Flux<<Entity>> findAll();
    Mono<<Entity>> save(<Entity> entity);
    Mono<Void> deleteById(String id);
}
```

**Input port** (what the application offers to entry points):
Create in `application/use-cases/src/main/java/com/<safename>/usecases/`:

```java
package com.<safename>.usecases;

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

Create in `application/use-cases/src/main/java/com/<safename>/usecases/`:

```java
package com.<safename>.usecases;

import com.<safename>.model.<Entity>;
import com.<safename>.model.<Entity>Repository;
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

**For PostgreSQL (R2DBC):**
Create in `infrastructure/driven-adapters/postgres/src/main/java/com/<safename>/postgres/`:

```java
// 1. R2DBC Entity (infrastructure concern — separate from domain entity)
@Table("<table_name>")
public class <Entity>Data {
    @Id
    private String id;
    // mapped fields
}

// 2. Spring Data R2DBC Repository
public interface <Entity>R2dbcRepository extends ReactiveCrudRepository<<Entity>Data, String> {
}

// 3. Adapter that implements the domain port
@Repository
@RequiredArgsConstructor
public class <Entity>RepositoryAdapter implements <Entity>Repository {
    private final <Entity>R2dbcRepository r2dbcRepository;

    // Map between domain entity and R2DBC entity
    @Override
    public Mono<<Entity>> findById(String id) {
        return r2dbcRepository.findById(id).map(this::toDomain);
    }

    private <Entity> toDomain(<Entity>Data data) { /* mapping */ }
    private <Entity>Data toData(<Entity> entity) { /* mapping */ }
}
```

**For MongoDB:**
Same pattern but use `@Document` instead of `@Table`, and `ReactiveMongoRepository` instead of `ReactiveCrudRepository`.

**Important:** Add the domain model dependency to the adapter module POM:
```xml
<dependency>
    <groupId>com.<safename>.model</groupId>
    <artifactId>domain-model</artifactId>
    <version>${project.version}</version>
</dependency>
```

### Adding a REST Controller (Entry Point)

Create in `infrastructure/entry-points/rest-api/src/main/java/com/<safename>/restapi/`:

```java
package com.<safename>.restapi;

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
    public Mono<<Entity>> getById(@PathVariable String id) {
        return useCase.getById(id);
    }

    @GetMapping
    public Flux<<Entity>> getAll() {
        return useCase.getAll();
    }

    @PostMapping
    public Mono<<Entity>> create(@RequestBody <Entity> entity) {
        return useCase.create(entity);
    }

    @DeleteMapping("/{id}")
    public Mono<Void> delete(@PathVariable String id) {
        return useCase.delete(id);
    }
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
The scaffold generates `RabbitMQConfig.java` and `MessagePublisher.java`. To add custom messages, create a port interface in the domain and implement it in the rabbit-producer adapter following the same pattern as repository adapters.

**Consumer** (entry-point — messages arrive from outside into the application):
The scaffold generates `RabbitMQConfig.java` and `MessageListener.java`. The listener should delegate to a use case, just like a REST controller would.

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

### Supabase Database Modifications (When MCP is Available)

During microservice development, if the Supabase MCP server is connected and the user's project uses a Supabase-hosted PostgreSQL database, you can modify the database schema directly when needed. This is useful when:

- Adding a new domain entity that requires a new table
- A use case requires altering an existing table (adding columns, constraints, indexes)
- The driven adapter needs database objects that don't exist yet (e.g., missing table for a new repository)

**Workflow:**

1. **Detect the need** — When creating a driven adapter (repository) for an entity and the corresponding table doesn't exist in Supabase, inform the user:
   > "The table `<table_name>` doesn't exist yet in Supabase. Would you like me to create it?"

2. **Generate and show the DDL** — Based on the domain entity and the adapter's `@Table`/`@Document` mapping, generate the CREATE TABLE statement. Show it to the user before executing.

3. **Execute on approval** — Use the Supabase MCP tools to run the DDL against the project database.

4. **Keep docs updated** — If a `docs/database/` directory exists with schema documentation, update it to reflect the changes.

**Important rules:**
- **Never modify the database without asking** — Always show the SQL and get explicit approval
- **Prefer additive changes** — CREATE TABLE, ADD COLUMN, CREATE INDEX. For destructive changes (DROP, ALTER column type), warn the user about potential data loss
- **Respect the schema design** — If `relational-db-schema-builder` was used to design the model, follow that design as the source of truth. Only suggest deviations if there's a technical reason

### Build and Run

```bash
cd <service-name>
mvn clean install                                         # Build all modules
mvn spring-boot:run -pl infrastructure/entry-points/app   # Run the application
```

The app starts on port 8080 by default. The starter `/hello` endpoint verifies everything works.

## Examples

### Full scaffold + entity workflow

```
User: "Create a users microservice with PostgreSQL"
→ Run scaffold: --service-name=ms-users --database=postgres
→ Offer to add User entity
→ Create User.java in domain/model
→ Create UserRepository port in domain/model
→ Create UserUseCase interface + impl in application/use-cases
→ Create UserRepositoryAdapter in driven-adapters/postgres
→ Create UserController in entry-points/app
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
