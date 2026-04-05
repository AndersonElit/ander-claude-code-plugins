# Claude Code Plugins

Coleccion de plugins para [Claude Code](https://docs.anthropic.com/en/docs/claude-code) que extienden sus capacidades con skills especializados para desarrollo de software.

## Plugins disponibles

| Plugin | Descripcion |
|--------|-------------|
| [springboot-hexagonal-builder](./plugins/springboot-hexagonal-builder/) | Genera microservicios reactivos con Spring Boot + Arquitectura Hexagonal, revision de codigo Java, documentacion de arquitectura C4, especificacion de requisitos ERS/SRS, modelado de bases de datos relacionales y NoSQL, y documentacion de APIs OpenAPI/Swagger |

### springboot-hexagonal-builder

Plugin con once skills y tres agentes autonomos para el ciclo completo de desarrollo backend en Java: desde el analisis de requisitos hasta el diseño arquitectonico, generacion de codigo, modelado de datos y documentacion de APIs.

**Skills:**

- **hexagonal-architecture-builder** - Scaffold completo de microservicios reactivos (Spring Boot 3.4.1, Java 21, WebFlux) con Arquitectura Hexagonal usando JBang. Soporta PostgreSQL, MongoDB, RabbitMQ y permite agregar tecnologias adicionales. Aplica convenciones de empaquetado por responsabilidad (`impl/`, `config/`, `entities/`, `ports/`, `repositories/`, `adapters/`, `dto/`, `mappers/`) y documenta el codigo generado con Javadoc inline. No ejecuta checkstyle ni crea `package-info.java`.
- **java-development-best-practices** - Revision, refactorizacion y generacion de codigo Java aplicando principios SOLID, Clean Code (KISS/DRY/YAGNI) y patrones GoF. Incluye verificacion de convenciones de empaquetado en proyectos hexagonales y documentacion Javadoc orientada a arquitectura.
- **java-testing-architect** - Genera, revisa y refactoriza tests para microservicios Java siguiendo buenas practicas de testing en arquitectura hexagonal. Define lineamientos para tests unitarios e integracion por capa.
- **c4-architecture** - Genera documentacion de arquitectura usando diagramas C4 (Context, Container, Component, Deployment, Dynamic) en sintaxis Mermaid.
- **srs-document-builder** - Genera documentos de Especificacion de Requisitos de Software (ERS/SRS) basados en IEEE 830, con requisitos funcionales, no funcionales, casos de uso, modelo de datos y matriz de trazabilidad.
- **relational-db-schema-builder** - Genera documentacion de esquemas de bases de datos relacionales, diagramas ER en Mermaid, scripts DDL y diccionarios de datos aplicando normalizacion y buenas practicas de diseño.
- **nosql-schema-builder** - Diseña y documenta esquemas NoSQL (MongoDB, DynamoDB, Cassandra), estructuras de colecciones, modelado de documentos (embedding vs referencing), validaciones JSON Schema, estrategias de indexacion y sharding.
- **openapi-doc-builder** - Genera documentacion de APIs usando especificacion OpenAPI 3.x/Swagger, incluyendo specs YAML, referencia de endpoints, guias de integracion y patrones comunes (paginacion, autenticacion, manejo de errores).
- **planning** - Guia la recopilacion interactiva de requisitos para tareas de desarrollo mediante entrevista estructurada al usuario.
- **sdlc-workflow-supervisor** - Orquesta el flujo SDLC completo desde requisitos del cliente hasta el desarrollo, aplicando quality gates y trazabilidad entre fases. Acepta requisitos del cliente como argumento.
- **microservices-eda-architecture** - Diseña arquitecturas de microservicios y Event-Driven Architecture (EDA) a partir de requisitos de negocio. Cubre descomposicion de dominios (bounded contexts), identificacion de eventos, patrones de comunicacion (coreografia/orquestacion), estrategias de consistencia de datos (Saga, CQRS, Event Sourcing, Outbox) y diseño de resiliencia (Circuit Breaker, DLQ, idempotencia, distributed tracing).

**Agentes autonomos:**

- **requirements-analyst** - Agente autonomo para planificacion de proyectos, analisis de requisitos y generacion de PRD. Produce un PRD y luego invoca `/srs-document-builder` para generar el documento formal SRS (IEEE 830).
- **software-architect-lead** - Arquitecto de Software & Tech Lead para diseño arquitectonico, decisiones tecnicas, diseño de soluciones, revision de codigo, seleccion de stack y documentacion tecnica. Evalua el estilo arquitectonico (Monolito Modular vs Microservicios vs Microservicios + EDA) mediante un scorecard de 5 criterios antes de generar los entregables obligatorios de diseño. Cuando el resultado es microservicios/EDA, invoca `/microservices-eda-architecture`.
- **backend-java-developer** - Desarrollador Backend Java especializado en microservicios reactivos con Spring Boot y Arquitectura Hexagonal. Brazo de ejecucion de la Fase 4 (Desarrollo) del flujo SDLC. Procesa **un microservicio a la vez** a traves del ciclo completo (scaffold → implementar → compilar → quality review → tests) antes de pasar al siguiente. Mantiene un tracker de progreso (`docs/development/PROGRESS.md`) para rastrear el estado de cada servicio y permitir reanudacion. **Re-lee la documentacion de diseño relevante antes de cada microservicio** para prevenir perdida de contexto. Genera todos los microservicios bajo `services/` con infraestructura compartida en `services/docker-compose.yml`. Incluye ciclo obligatorio de verificacion de compilacion (`mvn clean compile`) despues de cada microservicio. Tests unitarios e integracion obligatorios segun `docs/design/testing/`; ejecuta `mvn clean verify` en bucle hasta que todos los tests pasen. Cuando es re-invocado por correcciones de revision arquitectonica, aplica **cambios quirurgicos unicamente** — nunca re-scaffoldea o re-implementa microservicios ya completados.

**Flujo de trabajo:**

```
Requisitos del cliente
        |
        v
  requirements-analyst  --> PRD (docs/prd/) + SRS (docs/srs/)
        |
        v
  software-architect-lead  --> docs/design/
        |                      ├── architecture/  (decision de estilo + microservicios/EDA)
        |                      ├── openapi/       (OpenAPI spec)
        |                      ├── events/        (esquemas de eventos)
        |                      ├── scaffold/      (blueprint bajo services/)
        |                      ├── database/      (ER + schema.sql)
        |                      ├── testing/       (lineamientos de testing)
        |                      └── c4/            (diagramas C4)
        |
        v
  backend-java-developer  --> services/<service-name>/  (un microservicio a la vez)
        |                      services/docker-compose.yml
        |                      docs/development/
        |                      ├── PROGRESS.md    (tracker de progreso)
        |                      ├── TEST-REPORT
        |                      ├── SERVICE-GUIDE
        |                      ├── CURL-EXAMPLES
        |                      └── openapi/
        |
        v
  software-architect-lead  --> revision arquitectonica
  (reviewer)                   APROBADO → reporte final
                               RECHAZADO → REVIEW-CORRECTIONS.md
                               → re-desarrollo (correcciones quirurgicas, no desde cero)
```

## Instalacion

### Desde directorio local

```bash
claude --plugin-dir /ruta/a/este/repositorio/plugins/springboot-hexagonal-builder
```

### Clonar y usar

```bash
git clone https://github.com/AndersonElit/ander-claude-code-plugins.git
claude --plugin-dir ./ander-claude-code-plugins/plugins/springboot-hexagonal-builder
```

## Estructura del repositorio

```
ander-claude-code-plugins/
├── .claude-plugin/
│   └── marketplace.json                 # Indice del marketplace
├── CLAUDE.md                            # Instrucciones para Claude Code
├── README.md
└── plugins/
    └── springboot-hexagonal-builder/
        ├── .claude-plugin/
        │   └── plugin.json              # Metadata del plugin
        ├── agents/
        │   ├── requirements-analyst.md          # Agente de analisis de requisitos
        │   ├── software-architect-lead.md       # Agente arquitecto de software
        │   └── backend-java-developer.md        # Agente desarrollador backend Java
        ├── skills/
        │   ├── hexagonal-architecture-builder/
        │   │   └── SKILL.md             # Skill de scaffolding
        │   ├── java-development-best-practices/
        │   │   ├── SKILL.md             # Skill de buenas practicas
        │   │   └── references/
        │   ├── java-testing-architect/
        │   │   ├── SKILL.md             # Skill de testing
        │   │   └── references/
        │   ├── c4-architecture/
        │   │   ├── SKILL.md             # Skill de diagramas C4
        │   │   └── references/
        │   ├── srs-document-builder/
        │   │   ├── SKILL.md             # Skill de documentos ERS/SRS
        │   │   └── references/
        │   ├── relational-db-schema-builder/
        │   │   ├── SKILL.md             # Skill de modelado de BD relacional
        │   │   └── references/
        │   ├── nosql-schema-builder/
        │   │   ├── SKILL.md             # Skill de modelado de BD NoSQL
        │   │   └── references/
        │   ├── openapi-doc-builder/
        │   │   ├── SKILL.md             # Skill de documentacion API OpenAPI/Swagger
        │   │   └── references/
        │   ├── planning/
        │   │   └── SKILL.md             # Skill de recopilacion de requisitos
        │   ├── sdlc-workflow-supervisor/
        │   │   └── SKILL.md             # Skill orquestador del flujo SDLC
        │   └── microservices-eda-architecture/
        │       ├── SKILL.md             # Skill de arquitectura microservicios/EDA
        │       └── references/
        ├── templates/
        │   └── jbang/
        │       └── MavenHexagonalScaffold.java  # Generador JBang
        └── README.md
```

## Estructura generada en proyectos destino

Cuando se ejecuta el flujo SDLC completo, el proyecto destino tiene esta estructura:

```
<proyecto-destino>/
├── docs/
│   ├── prd/PRD-<nombre>.md                  # Product Requirements Document
│   ├── srs/SRS-<nombre>.md                  # Software Requirements Specification
│   ├── design/                              # Entregables de diseño
│   │   ├── architecture/                    # Decision de estilo + microservicios/EDA
│   │   ├── c4/                              # Diagramas C4
│   │   ├── openapi/                         # Especificacion OpenAPI
│   │   ├── database/                        # Modelo ER + DDL
│   │   ├── scaffold/                        # Blueprint de estructura (bajo services/)
│   │   ├── testing/                         # Lineamientos de testing
│   │   └── events/                          # Esquemas de eventos (si aplica)
│   ├── development/                         # Entregables de desarrollo
│   │   ├── PROGRESS.md                      # Tracker de progreso por microservicio
│   │   ├── TEST-REPORT.md
│   │   ├── SERVICE-GUIDE.md
│   │   ├── CURL-EXAMPLES.md
│   │   ├── LOCAL-TOOLS.md
│   │   ├── DIAGRAMS.md
│   │   ├── TECH-STACK.md
│   │   └── openapi/
│   └── sdlc-report/
│       └── SDLC-EXECUTION-REPORT.md         # Reporte final del flujo SDLC
└── services/                                # Codigo fuente e infraestructura
    ├── docker-compose.yml                   # Infraestructura compartida (DB, messaging, mocks)
    ├── ms-orders/                           # Microservicio (arquitectura hexagonal)
    ├── ms-inventory/                        # Microservicio
    └── ...
```

## Requisitos

- **Java 17+**
- **JBang** - `curl -Ls https://sh.jbang.dev | bash -s - app setup`
- **Maven** - Para compilar los proyectos generados
- **Node.js** - Para los servidores MCP (Supabase, MongoDB)

### Variables de entorno (opcionales, para servidores MCP)

| Variable | Descripcion | Como obtenerla |
|----------|-------------|----------------|
| `SUPABASE_ACCESS_TOKEN` | Token para el MCP de Supabase | Supabase Dashboard > Account Settings > Access Tokens |
| `MDB_MCP_CONNECTION_STRING` | Connection string para el MCP de MongoDB | Ej: `mongodb://localhost:27017` |

## Contribuir

Para agregar un nuevo plugin al repositorio:

1. Crea un directorio en `plugins/` con el nombre del plugin
2. Agrega el archivo `.claude-plugin/plugin.json` con la metadata
3. Crea los skills en `skills/<nombre-del-skill>/SKILL.md`
4. Incluye un `README.md` con la documentacion del plugin
5. Registra el plugin en `.claude-plugin/marketplace.json`

## Licencia

MIT
