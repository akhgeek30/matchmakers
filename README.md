# Matchmaking Platform

Legacy, microservice-based platform for matching candidates to jobs. The repository contains a Spring Cloud backend and an Angular single-page application. It is being revived as a private repository; expect dependency upgrades and environment-specific configuration work before production use.

## Architecture

The backend is a Maven reactor of Spring Boot 2.0 / Spring Cloud Finchley services. `ConfigServer` supplies externalized configuration, `EurekaRegistry` provides discovery, and `ZuulGateway` is the API edge. Domain services communicate through REST, Eureka, Kafka, and WebSocket integrations as implemented by each service. `Angular/` is the browser client.

| Area | Services |
| --- | --- |
| Platform | `ConfigServer`, `EurekaRegistry`, `ZuulGateway` |
| Candidate/profile | `RegistrationService`, `UserAuthentication`, `EducationService`, `ExperienceService`, `CertificationService`, `SkillService`, `InterestService`, `LocationService`, `UploadingPhoto` |
| Matching/search | `ProjectService`, `UpstreamService`, `DownstreamService`, `NlpPipeline`, `IndexerService`, `QueryService`, `QueryEngine`, `CacheService` |
| Community/realtime | `CommunityService`, `WebsocketService` |
| Client | `Angular` |

`docker-compose.yml` describes the legacy local stack and is useful as a service inventory. It includes Redis, ZooKeeper/Kafka, MongoDB, MySQL, Neo4j, the Spring services, gateway, and Angular client.

## Prerequisites

- JDK 10 and Maven 3.5+ (the root POM declares Java 10; modern JDKs may require dependency/plugin upgrades).
- Node.js compatible with Angular CLI 6 / Angular 6 (Node 10-era tooling is the safest starting point) and npm.
- Docker and Docker Compose for local infrastructure.
- A local or reachable MySQL, MongoDB, Neo4j, Redis, Kafka, and ZooKeeper stack if not using Compose.
- Access to a separate Git repository holding the Spring Cloud configuration consumed by `ConfigServer`.

## Local setup

1. Copy `.env.example` to `.env` and replace every `change-me` value with local secrets. `.env` is intentionally ignored.
2. Set the same values in the environment of services started outside Docker. In PowerShell:

   ```powershell
   $env:MYSQL_PASSWORD = '...'
   $env:NEO4J_PASSWORD = '...'
   $env:NEO4J_URI = 'bolt://localhost:7687'
   $env:CONFIG_REPOSITORY_URI = 'https://github.com/your-org/config-server.git'
   ```

3. Start infrastructure, for example `docker compose up -d redis zookeeper kafka mongodb mysql`. Neo4j is expected by several services but is not currently defined in the legacy Compose file; run it separately or add a reviewed service definition.
4. Ensure the configuration repository contains the application profiles expected by the services. It is deliberately not included here because it may contain environment-specific values.
5. Build the backend from the root with `mvn clean install`. A service can be run with `./mvnw spring-boot:run` from its own directory (on Windows, `./mvnw.cmd spring-boot:run`).

### Backend startup order

1. MySQL, MongoDB, Neo4j, Redis, ZooKeeper, and Kafka.
2. `ConfigServer` (requires `CONFIG_REPOSITORY_URI`).
3. `EurekaRegistry`.
4. `ZuulGateway`.
5. The domain, matching/search, and community services; start their dependencies first where applicable.

Verify service registration in Eureka (default port `8091`) before exercising gateway traffic (default port `8097`). Ports and routes are legacy configuration and should be validated against the external configuration repository.

### Angular client

```powershell
Set-Location Angular
npm ci
npm start
```

The dev server normally listens on port 4200. Review `Angular/proxy.conf.json` and the API URLs before use; they may assume the old gateway topology.

## Testing

- Backend reactor: `mvn test`
- One backend service: run `./mvnw test` in that service directory.
- Angular unit tests: `npm test -- --watch=false`
- Angular lint: `npm run lint`
- Angular end-to-end tests: `npm run e2e` (requires the UI and browser tooling)

The root Maven configuration sets `testFailureIgnore=true`, so a green reactor result is not proof that all tests passed. Review the Surefire reports in each service's `target/` directory.

## Secrets and configuration

No reusable credentials or private key material should be committed. Database and Neo4j passwords now come from environment variables, and the previous external Neo4j IP and Config Server repository URL have been replaced with configurable values. Legacy `.p12` keystores are intentionally excluded; provide a local keystore through the deployment secret store if TLS is re-enabled. The legacy GitLab Sonar configuration now expects protected CI variables (`SONAR_URL` and `SONAR_TOKEN`) instead of committed credentials.

Before enabling CI or sharing access, rotate any credentials previously present in the old GitLab history, use GitHub Actions/GitHub Secrets or another secret manager for CI, and keep the external config repository private. The GitHub repository should be published from a sanitized root commit, not with the historical GitLab commits, since old commits can retain removed secrets.

## Legacy constraints

- Spring Boot 2.0.5, Spring Cloud Finchley, Java 10, Angular 6, and the bundled Kafka tooling are end-of-life.
- The Compose file uses `network_mode: host` and older images, which are primarily Linux-oriented and need adaptation for Docker Desktop/Windows.
- Naming/casing is inconsistent (`ZuulGateway` in Maven versus a lowercase working-tree directory on case-insensitive filesystems); keep the Maven module spelling when changing it.
- `CommunityService` exists in the tree but is not listed in the root Maven modules; it may need separate build and integration work.
- Runtime logs, PID files, IDE metadata, build output, local environment files, and dependency directories are intentionally ignored.
