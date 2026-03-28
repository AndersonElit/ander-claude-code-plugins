# Claude Code Plugins

Coleccion de plugins para [Claude Code](https://docs.anthropic.com/en/docs/claude-code) que extienden sus capacidades con skills especializados para desarrollo de software.

## Plugins disponibles

| Plugin | Descripcion |
|--------|-------------|
| [springboot-hexagonal-builder](./plugins/springboot-hexagonal-builder/) | Genera microservicios reactivos con Spring Boot + Arquitectura Hexagonal, revision de codigo Java, documentacion de arquitectura C4, especificacion de requisitos ERS/SRS, modelado de bases de datos relacionales y NoSQL, y documentacion de APIs OpenAPI/Swagger |

### springboot-hexagonal-builder

Plugin con siete skills para desarrollo backend en Java, documentacion de arquitectura, analisis de requisitos, modelado de datos y documentacion de APIs:

- **hexagonal-architecture-builder** - Scaffold completo de microservicios reactivos (Spring Boot 3.4.1, Java 21, WebFlux) con Arquitectura Hexagonal usando JBang. Soporta PostgreSQL, MongoDB, RabbitMQ y permite agregar tecnologias adicionales.
- **java-development-best-practices** - Revision, refactorizacion y generacion de codigo Java aplicando principios SOLID, Clean Code (KISS/DRY/YAGNI) y patrones GoF.
- **c4-architecture** - Genera documentacion de arquitectura usando diagramas C4 (Context, Container, Component, Deployment, Dynamic) en sintaxis Mermaid.
- **srs-document-builder** - Genera documentos de Especificacion de Requisitos de Software (ERS/SRS) basados en IEEE 830, con requisitos funcionales, no funcionales, casos de uso, modelo de datos y matriz de trazabilidad.
- **relational-db-schema-builder** - Genera documentacion de esquemas de bases de datos relacionales, diagramas ER en Mermaid, scripts DDL y diccionarios de datos aplicando normalizacion y buenas practicas de diseño.
- **nosql-schema-builder** - Diseña y documenta esquemas NoSQL (MongoDB, DynamoDB, Cassandra), estructuras de colecciones, modelado de documentos (embedding vs referencing), validaciones JSON Schema, estrategias de indexacion y sharding.
- **openapi-doc-builder** - Genera documentacion de APIs usando especificacion OpenAPI 3.x/Swagger, incluyendo specs YAML, referencia de endpoints, guias de integracion y patrones comunes (paginacion, autenticacion, manejo de errores).

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
├── README.md
└── plugins/
    └── springboot-hexagonal-builder/
        ├── .claude-plugin/
        │   └── plugin.json              # Metadata del plugin
        ├── skills/
        │   ├── hexagonal-architecture-builder/
        │   │   └── SKILL.md             # Skill de scaffolding
        │   ├── java-development-best-practices/
        │   │   ├── SKILL.md             # Skill de buenas practicas
        │   │   └── references/
        │   │       └── patterns-reference.md
        │   ├── c4-architecture/
        │   │   ├── SKILL.md             # Skill de diagramas C4
        │   │   └── references/
        │   │       ├── c4-syntax.md
        │   │       ├── common-mistakes.md
        │   │       └── advanced-patterns.md
        │   ├── srs-document-builder/
        │   │   ├── SKILL.md             # Skill de documentos ERS/SRS
        │   │   └── references/
        │   │       ├── ieee-830-checklist.md
        │   │       └── requirements-patterns.md
        │   ├── relational-db-schema-builder/
        │   │   ├── SKILL.md             # Skill de modelado de BD relacional
        │   │   └── references/
        │   │       ├── normalization-guide.md
        │   │       └── design-patterns.md
        │   ├── nosql-schema-builder/
        │   │   ├── SKILL.md             # Skill de modelado de BD NoSQL
        │   │   └── references/
        │   │       ├── modeling-patterns.md
        │   │       └── mongodb-best-practices.md
        │   └── openapi-doc-builder/
        │       ├── SKILL.md             # Skill de documentacion API OpenAPI/Swagger
        │       └── references/
        │           ├── openapi-syntax.md
        │           ├── common-patterns.md
        │           └── rest-principles.md
        ├── templates/
        │   └── jbang/
        │       └── MavenHexagonalScaffold.java  # Generador JBang
        └── README.md
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
