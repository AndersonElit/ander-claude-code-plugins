# Claude Code Plugins

Coleccion de plugins para [Claude Code](https://docs.anthropic.com/en/docs/claude-code) que extienden sus capacidades con skills especializados para desarrollo de software.

## Plugins disponibles

| Plugin | Descripcion |
|--------|-------------|
| [springboot-hexagonal-builder](./plugins/springboot-hexagonal-builder/) | Genera microservicios reactivos con Spring Boot + Arquitectura Hexagonal, revision de codigo Java, documentacion de arquitectura C4 y especificacion de requisitos ERS/SRS |

### springboot-hexagonal-builder

Plugin con cuatro skills para desarrollo backend en Java, documentacion de arquitectura y analisis de requisitos:

- **hexagonal-architecture-builder** - Scaffold completo de microservicios reactivos (Spring Boot 3.4.1, Java 21, WebFlux) con Arquitectura Hexagonal usando JBang. Soporta PostgreSQL, MongoDB, RabbitMQ y permite agregar tecnologias adicionales.
- **java-development-best-practices** - Revision, refactorizacion y generacion de codigo Java aplicando principios SOLID, Clean Code (KISS/DRY/YAGNI) y patrones GoF.
- **c4-architecture** - Genera documentacion de arquitectura usando diagramas C4 (Context, Container, Component, Deployment, Dynamic) en sintaxis Mermaid.
- **srs-document-builder** - Genera documentos de Especificacion de Requisitos de Software (ERS/SRS) basados en IEEE 830, con requisitos funcionales, no funcionales, casos de uso, modelo de datos y matriz de trazabilidad.

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
        │   └── srs-document-builder/
        │       ├── SKILL.md             # Skill de documentos ERS/SRS
        │       └── references/
        │           ├── ieee-830-checklist.md
        │           └── requirements-patterns.md
        ├── templates/
        │   └── jbang/
        │       └── MavenHexagonalScaffold.java  # Generador JBang
        └── README.md
```

## Requisitos

- **Java 17+**
- **JBang** - `curl -Ls https://sh.jbang.dev | bash -s - app setup`
- **Maven** - Para compilar los proyectos generados

## Contribuir

Para agregar un nuevo plugin al repositorio:

1. Crea un directorio en `plugins/` con el nombre del plugin
2. Agrega el archivo `.claude-plugin/plugin.json` con la metadata
3. Crea los skills en `skills/<nombre-del-skill>/SKILL.md`
4. Incluye un `README.md` con la documentacion del plugin
5. Registra el plugin en `.claude-plugin/marketplace.json`

## Licencia

MIT
