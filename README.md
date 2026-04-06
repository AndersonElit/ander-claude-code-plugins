# Claude Code Plugins

Coleccion de plugins para [Claude Code](https://docs.anthropic.com/en/docs/claude-code) que extienden sus capacidades con skills especializados para desarrollo de software.

## Plugins disponibles

| Plugin | Descripcion |
|--------|-------------|
| [springboot-hexagonal-builder](./plugins/springboot-hexagonal-builder/) | Genera microservicios reactivos con Spring Boot + Arquitectura Hexagonal, revision de codigo Java, documentacion de arquitectura C4, especificacion de requisitos ERS/SRS, modelado de bases de datos relacionales y NoSQL, y documentacion de APIs OpenAPI/Swagger |

### springboot-hexagonal-builder

Plugin con diez skills y cuatro agentes autonomos para el ciclo completo de desarrollo backend en Java: desde el analisis de requisitos hasta el diseño arquitectonico, generacion de codigo, modelado de datos y documentacion de APIs.

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
- **microservices-eda-architecture** - Diseña arquitecturas de microservicios y Event-Driven Architecture (EDA) a partir de requisitos de negocio. Cubre descomposicion de dominios (bounded contexts), identificacion de eventos, patrones de comunicacion (coreografia/orquestacion), estrategias de consistencia de datos (Saga, CQRS, Event Sourcing, Outbox) y diseño de resiliencia (Circuit Breaker, DLQ, idempotencia, distributed tracing).

**Agentes autonomos:**

- **requirements-analyst** - Agente autonomo para planificacion de proyectos, analisis de requisitos y generacion de PRD. Produce un PRD y luego invoca `/srs-document-builder` para generar el documento formal SRS (IEEE 830).
- **software-architect-lead** - Arquitecto de Software para diseño arquitectonico, decisiones tecnicas, diseño de soluciones, revision de codigo, seleccion de stack y documentacion tecnica. Evalua el estilo arquitectonico (Monolito Modular vs Microservicios vs Microservicios + EDA) mediante un scorecard de 5 criterios antes de generar los entregables de diseño (diagramas C4, modelo ER, OpenAPI, esquemas de eventos, estructura de proyecto, lineamientos de testing). Cuando el resultado es microservicios/EDA, invoca `/microservices-eda-architecture`.
- **tech-lead** - Tech Lead para provisionamiento de infraestructura y especificaciones por microservicio. Genera Docker Compose (`infrastructure/docker-compose.yml`) con scripts de inicializacion que crean bases de datos y tablas automaticamente. Genera un spec autocontenido por cada microservicio en `docs/design/microservices/<service-name>.md` con todo lo necesario para construirlo de forma independiente (entidades, esquema BD, endpoints API, eventos, reglas de negocio, diagrama de componentes, blueprint, conexion a infraestructura, estrategia de testing).
- **backend-java-developer** - Desarrollador Backend Java especializado en microservicios reactivos con Spring Boot y Arquitectura Hexagonal. Recibe las especificaciones del servicio a construir (nombre, base de datos, messaging, entidades, endpoints, eventos, reglas de negocio) y produce codigo listo para produccion. Scaffold via JBang, implementacion de todas las capas (dominio → aplicacion → adaptadores → entry-points), genera Dockerfile del servicio, ciclo obligatorio de verificacion de compilacion (`mvn clean compile`), revision de calidad de codigo, y tests unitarios e integracion obligatorios (`mvn clean verify` en bucle hasta que todos pasen). Genera entregables de desarrollo (TEST-REPORT, SERVICE-GUIDE, CURL-EXAMPLES, TECH-STACK, OpenAPI spec).

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
        │   ├── tech-lead.md                     # Agente tech lead (infraestructura + specs)
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
│   │   ├── microservices/                   # Spec autocontenido por servicio (generado por tech-lead)
│   │   │   ├── ms-orders.md
│   │   │   └── ms-inventory.md
│   │   ├── c4/                              # Diagramas C4
│   │   ├── openapi/                         # Especificacion OpenAPI
│   │   ├── database/                        # Modelo ER + DDL
│   │   ├── scaffold/                        # Blueprint de estructura
│   │   ├── testing/                         # Lineamientos de testing
│   │   └── events/                          # Esquemas de eventos (si aplica)
│   └── development/                         # Entregables de desarrollo
│       ├── TEST-REPORT.md
│       ├── SERVICE-GUIDE.md
│       ├── CURL-EXAMPLES.md
│       ├── TECH-STACK.md
│       └── openapi/
├── infrastructure/                          # Infraestructura Docker (generada por tech-lead)
│   ├── docker-compose.yml                   # Contenedores (BD, messaging, mocks)
│   ├── init-scripts/                        # Scripts de inicializacion de BD
│   └── wiremock/                            # Stubs de WireMock
└── <service-name>/                          # Microservicio (arquitectura hexagonal)
    ├── Dockerfile
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
