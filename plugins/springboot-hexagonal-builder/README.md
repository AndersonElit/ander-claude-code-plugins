# Spring Boot Hexagonal Builder - Claude Code Plugin

Plugin para Claude Code que genera microservicios reactivos con Spring Boot siguiendo Arquitectura Hexagonal (Puertos y Adaptadores).

## Skills incluidos

| Skill | Descripcion |
|-------|-------------|
| `hexagonal-architecture-builder` | Genera y extiende microservicios reactivos con Arquitectura Hexagonal usando JBang |
| `java-development-best-practices` | Revisa, refactoriza y genera codigo Java aplicando SOLID, KISS/DRY/YAGNI, patrones GoF |
| `c4-architecture` | Genera diagramas de arquitectura C4 (Context, Container, Component, Deployment) en Mermaid |
| `srs-document-builder` | Genera documentos de Especificacion de Requisitos (ERS/SRS) basados en IEEE 830 |
| `relational-db-schema-builder` | Diseña esquemas de BD relacional: diagramas ER, DDL, diccionarios de datos |
| `nosql-schema-builder` | Diseña esquemas NoSQL (MongoDB, DynamoDB, Cassandra): modelado de documentos, validaciones JSON Schema, indices, sharding |
| `openapi-doc-builder` | Genera documentacion de APIs con OpenAPI 3.x/Swagger |

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

## Arquitectura generada

```
{service-name}/
├── pom.xml                              # Parent POM (Spring Boot 3.4.1, Java 21)
├── domain/model/                        # Modelo de dominio puro
├── application/use-cases/               # Logica de negocio
├── infrastructure/
│   ├── driven-adapters/                 # Adaptadores de salida (DB, mensajeria)
│   └── entry-points/
│       ├── rest-api/                    # Controladores REST (WebFlux)
│       ├── app/                         # MainApplication + config
│       └── rabbit-consumer/             # (si se selecciona)
```

## Licencia

MIT
