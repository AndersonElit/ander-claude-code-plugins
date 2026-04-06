# Spring Boot Hexagonal Builder - Claude Code Plugin

Plugin para Claude Code que genera microservicios reactivos con Spring Boot siguiendo Arquitectura Hexagonal (Puertos y Adaptadores).

## Skills incluidos

| Skill | Descripcion |
|-------|-------------|
| `hexagonal-architecture-builder` | Genera y extiende microservicios reactivos con Arquitectura Hexagonal usando JBang |
| `java-development-best-practices` | Revisa, refactoriza y genera codigo Java aplicando SOLID, KISS/DRY/YAGNI, patrones GoF |
| `java-testing-architect` | Genera, revisa y refactoriza tests para microservicios Java. Define lineamientos de tests unitarios e integracion por capa hexagonal |
| `c4-architecture` | Genera diagramas de arquitectura C4 (Context, Container, Component, Deployment) en Mermaid |
| `srs-document-builder` | Genera documentos de Especificacion de Requisitos (ERS/SRS) basados en IEEE 830 |
| `relational-db-schema-builder` | Diseña esquemas de BD relacional: diagramas ER, DDL, diccionarios de datos |
| `nosql-schema-builder` | Diseña esquemas NoSQL (MongoDB, DynamoDB, Cassandra): modelado de documentos, validaciones JSON Schema, indices, sharding |
| `openapi-doc-builder` | Genera documentacion de APIs con OpenAPI 3.x/Swagger |
| `planning` | Guia la recopilacion interactiva de requisitos para tareas de desarrollo |
| `microservices-eda-architecture` | Diseña arquitecturas de microservicios y EDA: descomposicion de dominios, eventos, patrones de comunicacion (coreografia/orquestacion), consistencia de datos (Saga, CQRS, Event Sourcing, Outbox) y resiliencia (Circuit Breaker, DLQ, idempotencia) |

## Agentes autonomos

| Agente | Descripcion |
|--------|-------------|
| `requirements-analyst` | Planificacion de proyectos, analisis de requisitos y generacion de PRD + SRS (IEEE 830) |
| `software-architect-lead` | Evalua estilo arquitectonico (Monolito Modular vs Microservicios vs Microservicios + EDA) mediante scorecard, genera entregables de diseño (diagramas C4, modelo ER, OpenAPI, esquemas de eventos, estructura de proyecto, lineamientos de testing). Invoca `/microservices-eda-architecture` cuando aplica |
| `tech-lead` | Genera infraestructura Docker Compose (`infrastructure/docker-compose.yml`) con init scripts que crean BD y tablas automaticamente. Genera un spec autocontenido por cada microservicio en `docs/design/microservices/` con todo lo necesario para construirlo de forma independiente (entidades, esquema BD, endpoints API, eventos, reglas de negocio, diagrama de componentes, blueprint, conexion a infraestructura, estrategia de testing) |
| `backend-java-developer` | Desarrollador backend Java especializado en microservicios reactivos con Spring Boot y Arquitectura Hexagonal. Recibe las especificaciones del servicio a construir (nombre, BD, messaging, entidades, endpoints, eventos, reglas de negocio). Scaffold via JBang, implementacion de todas las capas, genera Dockerfile del servicio, ciclo obligatorio de compilacion (`mvn clean compile`) y tests (`mvn clean verify`). Genera entregables de desarrollo (TEST-REPORT, SERVICE-GUIDE, CURL-EXAMPLES, TECH-STACK, OpenAPI spec) |

### Flujo de trabajo

Los cuatro agentes pueden usarse de forma independiente o secuencial:

1. **`requirements-analyst`** recibe los requisitos del cliente → genera PRD (`docs/prd/`) y SRS (`docs/srs/`)
2. **`software-architect-lead`** recibe el PRD/SRS → evalua el estilo arquitectonico → genera los entregables de diseño en `docs/design/` (C4, ER model, OpenAPI, eventos, scaffold, testing)
3. **`tech-lead`** recibe las decisiones arquitectonicas → genera `infrastructure/docker-compose.yml` con init scripts → genera spec autocontenido por servicio en `docs/design/microservices/<service-name>.md`
4. **`backend-java-developer`** recibe las especificaciones del servicio a construir → scaffold → implementa todas las capas → genera Dockerfile → compila → quality review → tests → genera entregables en `docs/development/`

## Instalacion

### Opcion 1: Desde GitHub (recomendado)

Agrega este repositorio como marketplace en Claude Code:

```bash
/plugin marketplace add tu-usuario/springboot-microservice-builder-template
```

Luego instala el plugin:

```bash
/plugin install springboot-hexagonal-builder
```

### Opcion 2: Directorio local

Si clonaste el repositorio localmente:

```bash
claude --plugin-dir /ruta/al/springboot-microservice-builder-template
```

### Opcion 3: Probar antes de instalar

```bash
# Clona el repositorio
git clone https://github.com/tu-usuario/springboot-microservice-builder-template.git

# Ejecuta Claude Code con el plugin cargado localmente
claude --plugin-dir ./springboot-microservice-builder-template
```

## Uso

Una vez instalado, los skills estan disponibles en cualquier proyecto de Claude Code:

```bash
# Generar un nuevo microservicio
/springboot-hexagonal-builder:hexagonal-architecture-builder --service-name=ms-users --database=postgres

# Revisar codigo Java con mejores practicas
/springboot-hexagonal-builder:java-development-best-practices
```

## Servidores MCP integrados

Este plugin incluye dos servidores MCP que se inician automaticamente al habilitar el plugin:

| Servidor | Paquete | Variable de entorno | Proposito |
|----------|---------|---------------------|-----------|
| Supabase | `@supabase/mcp-server-supabase` | `SUPABASE_ACCESS_TOKEN` | Gestion de base de datos, auth, storage y edge functions en Supabase |
| MongoDB | `@mongodb-js/mongodb-mcp-server` | `MDB_MCP_CONNECTION_STRING` | Gestion de bases de datos, colecciones, indices y consultas en MongoDB |

Configura las variables de entorno antes de iniciar Claude Code:

```bash
export SUPABASE_ACCESS_TOKEN="tu-token-aqui"
export MDB_MCP_CONNECTION_STRING="mongodb://localhost:27017"
```

## Requisitos previos

Para usar el scaffold de generacion de proyectos:

- **Java 17+**
- **JBang**: `curl -Ls https://sh.jbang.dev | bash -s - app setup`
- **Maven** (para compilar los proyectos generados)
- **Node.js** (para los servidores MCP via npx)

## Tecnologias soportadas

### Bases de datos
| Tecnologia | Flag | Tipo |
|-----------|------|------|
| PostgreSQL | `--database=postgres` | R2DBC (reactivo) |
| MongoDB | `--database=mongo` | ReactiveMongoRepository |

### Mensajeria
| Tecnologia | Flag | Tipo |
|-----------|------|------|
| RabbitMQ Producer | `--messaging-system=rabbit-producer` | Driven adapter |
| RabbitMQ Consumer | `--messaging-system=rabbit-consumer` | Entry point |

### Tecnologias no soportadas nativamente

Si solicitas una tecnologia no soportada (MySQL, Kafka, Redis, SQS, etc.), el skill la configura manualmente en tu proyecto **y** actualiza el scaffold para soportarla en futuras generaciones.

## Estructura de proyecto generado

Estructura generada por el agente `backend-java-developer`:

```
<project-root>/
├── docs/
│   ├── prd/                                 # PRD del proyecto
│   ├── srs/                                 # SRS (IEEE 830)
│   ├── design/                              # Entregables de diseño
│   └── development/
│       ├── TEST-REPORT.md
│       ├── SERVICE-GUIDE.md
│       ├── CURL-EXAMPLES.md
│       ├── TECH-STACK.md
│       └── openapi/
└── <service-name>/                          # Microservicio (arquitectura hexagonal)
    ├── Dockerfile
    └── ...
```

## Arquitectura hexagonal por microservicio

Cada microservicio generado sigue esta estructura:

```
{service-name}/
├── pom.xml                              # Parent POM (Spring Boot 3.4.1, Java 21)
├── .env                                 # Variables de entorno (NO se commitea)
├── .env.example                         # Plantilla sin secretos (se commitea)
├── domain/model/
│   └── com.{n}.model/
│       ├── entities/
│       │   └── {Entity}.java            # Entidad de dominio pura (sin framework)
│       ├── ports/
│       │   ├── {Entity}Repository.java  # Puerto de salida (persistencia)
│       │   └── {Entity}EventPublisher.java # Puerto de salida (mensajeria)
│       ├── enums/
│       │   └── {Entity}Status.java      # Enumeraciones de dominio
│       ├── events/
│       │   └── {Entity}CreatedEvent.java # Eventos de dominio (inmutables)
│       ├── exceptions/
│       │   └── {Entity}NotFoundException.java
│       └── valueobjects/
│           └── Email.java               # Value objects (opcional)
├── application/use-cases/
│   └── com.{n}.usecases/
│       ├── {Entity}UseCase.java         # Puerto de entrada (interfaz)
│       └── impl/
│           └── {Entity}UseCaseImpl.java # Implementacion del caso de uso
├── infrastructure/
│   ├── driven-adapters/
│   │   ├── postgres/                    # Adaptador R2DBC PostgreSQL
│   │   │   └── com.{n}.postgres/
│   │   │       ├── entities/
│   │   │       │   └── {Entity}Data.java
│   │   │       ├── repositories/
│   │   │       │   └── {Entity}R2dbcRepository.java
│   │   │       └── adapters/
│   │   │           └── {Entity}RepositoryAdapter.java
│   │   ├── mongo/                       # Adaptador MongoDB
│   │   │   └── com.{n}.mongo/
│   │   │       ├── entities/
│   │   │       │   └── {Entity}Document.java
│   │   │       ├── repositories/
│   │   │       │   └── {Entity}MongoRepository.java
│   │   │       └── adapters/
│   │   │           └── {Entity}RepositoryAdapter.java
│   │   └── rabbit-producer/             # Productor RabbitMQ
│   │       └── com.{n}.rabbitproducer/
│   │           ├── config/
│   │           │   └── RabbitMQConfig.java
│   │           └── adapters/
│   │               └── RabbitMQMessagePublisher.java
│   └── entry-points/
│       ├── rest-api/
│       │   └── com.{n}.restapi/
│       │       ├── {Entity}Controller.java
│       │       └── dto/
│       │           ├── {Entity}Request.java
│       │           └── {Entity}Response.java
│       ├── app/                         # MainApplication + config Spring
│       │   └── com.{n}.app/
│       │       └── config/
│       │           └── BeanConfig.java
│       └── rabbit-consumer/             # (si se selecciona)
│           └── com.{n}.rabbitconsumer/
│               ├── config/
│               │   └── RabbitMQConfig.java
│               └── MessageListener.java
```

## Documentacion del codigo generado

- **No** se ejecuta `mvn checkstyle`, `mvn pmd:check`, `mvn spotbugs:check` ni ninguna herramienta de analisis estatico a menos que el usuario lo solicite explicitamente.
- **No** se crean archivos `package-info.java`. No son parte de las convenciones de este proyecto.
- En su lugar, cada clase generada lleva un **Javadoc inline** en la declaracion de clase explicando su rol en la arquitectura. Los comentarios son arquitecturales ("por que existe esta clase, a que capa pertenece"), no mecanicos.

```java
/** Domain entity representing a customer. Pure Java — no framework dependencies. */
@Data @Builder public class Customer { ... }

/** Output port: persistence contract for {@link Customer}. Implemented in driven-adapters. */
public interface CustomerRepository { ... }

/** R2DBC entity mapped to the {@code customers} table. Infrastructure concern only. */
@Table("customers") public class CustomerData { ... }

/** Implements {@link CustomerRepository} via Spring Data R2DBC. Handles domain↔data mapping. */
@Repository public class CustomerRepositoryAdapter implements CustomerRepository { ... }
```

## Convenciones de empaquetado

Cada modulo Maven organiza sus clases en sub-paquetes por responsabilidad:

**Capa de dominio (`domain/model`) — nunca en el paquete raiz:**

| Tipo de clase | Sub-paquete | Descripcion |
|--------------|-------------|-------------|
| Entidades de dominio (puras, sin framework) | `entities/` | `@Data @Builder`, sin `@Table` ni `@Document` |
| Interfaces de puertos de salida | `ports/` | Repositorios, publishers que la infra debe implementar |
| Enumeraciones de negocio | `enums/` | Estados, tipos, categorias del dominio |
| Eventos de dominio | `events/` | Inmutables: `@Value @Builder` |
| Excepciones de dominio | `exceptions/` | Extienden `RuntimeException` |
| Objetos de valor | `valueobjects/` | Inmutables, auto-validados, tipados |

**Capas de infraestructura y aplicacion:**

| Tipo de clase | Sub-paquete | Regla |
|--------------|-------------|-------|
| Implementacion de interfaz | `impl/` | Siempre que existe la interfaz en el mismo modulo |
| Clases `@Configuration` | `config/` | Toda clase de configuracion Spring |
| Entidades de persistencia (`@Table`, `@Document`) | `entities/` | En driven-adapters, nunca en el paquete raiz |
| Interfaces Spring Data | `repositories/` | `ReactiveCrudRepository`, `ReactiveMongoRepository`, etc. |
| Implementaciones de puertos de dominio | `adapters/` | Clases que `implements` un puerto del dominio |
| Objetos de transferencia REST | `dto/` | Clases `Request`, `Response`, `Dto` |
| Mappers de entidad↔dominio | `mappers/` | Cuando el mapeo tiene mas de 3 campos o logica compleja |

## Licencia

MIT
