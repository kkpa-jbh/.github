# JBH

JBH is a personal finance platform. Users track money movements, products and monthly balances.
Teams can share their finances, and invite members by email.

The system is built as microservices. Each service is its own repository.
It runs in production on a single VPS, with Docker Compose and Caddy.

## Architecture

```mermaid
flowchart LR
    app[jbh-app<br/>Ionic / Angular] -->|/jbh-api/**| gw[jbh-gateway<br/>Spring Cloud Gateway]
    gw --> iam[jbh-iam<br/>auth, users, teams]
    gw --> pf[jbh-personal-finance<br/>finance, preferences, notifications]
    gw -. service discovery .-> consul[(Consul)]
    iam -. register .-> consul
    pf -. register .-> consul
    pf -->|gateway-client| gw
    iam -->|gateway-client| gw
    iam --> db1[(Postgres)]
    pf --> db2[(Postgres)]
```

- **One entry point.** The app calls only the gateway. The gateway checks the JWT and applies rate limits.
- **Service discovery.** Services register in Consul. The gateway finds them by name.
- **No direct calls between services.** They call each other through the gateway, with a shared Java client library.
- **Auth.** JWT access tokens (15 min) and refresh tokens (30 days), in headers or secure cookies. Google sign-in.

## Repositories

| Repository | What it does | Tech |
|------------|--------------|------|
| [jbh-personal-finance](https://github.com/kkpa-jbh/jbh-personal-finance) | Finance, preferences and notifications. Modular monolith with hexagonal modules. | Java 21, Quarkus, Maven, Liquibase |
| [jbh-deploy](https://github.com/kkpa-jbh/jbh-deploy) | Production deploy and the system map. | Docker Compose, Caddy, Make |
| `jbh-iam` | Login, users, teams, JWT. | Kotlin, Spring Boot, Spring Security, Gradle |
| `jbh-gateway` | Single API entry point: routing, JWT check, rate limit. | Kotlin, Spring Cloud Gateway |
| `jbh-gateway-client` | Java library for service-to-service calls. | Java 21, Maven, GitHub Packages |
| `jbh-app` | Web and mobile front end. | Ionic, Angular, Capacitor |
| `jbh-discovery-nexus` | Service discovery. | Consul, Docker |

## Delivery

- **CI:** GitHub Actions builds each service and pushes a container image to GHCR, tagged with the git SHA.
- **Shared libraries** are published to GitHub Packages (Maven), in dependency order.
- **Deploy:** the server only pulls images. Rollback is a tag change.
- **Edge:** Caddy serves HTTPS and the static app. Only the gateway and the web app are reachable from outside.
- **Databases:** PostgreSQL, with migrations that run at service start, and daily backups.
