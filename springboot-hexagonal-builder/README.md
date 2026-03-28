# Spring Boot Hexagonal Builder - Claude Code Plugin

Plugin para Claude Code que genera microservicios reactivos con Spring Boot siguiendo Arquitectura Hexagonal (Puertos y Adaptadores).

## Skills incluidos

| Skill | Descripcion |
|-------|-------------|
| `hexagonal-architecture-builder` | Genera y extiende microservicios reactivos con Arquitectura Hexagonal usando JBang |
| `java-development-best-practices` | Revisa, refactoriza y genera codigo Java aplicando SOLID, KISS/DRY/YAGNI, patrones GoF |

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

## Requisitos previos

Para usar el scaffold de generacion de proyectos:

- **Java 17+**
- **JBang**: `curl -Ls https://sh.jbang.dev | bash -s - app setup`
- **Maven** (para compilar los proyectos generados)

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
