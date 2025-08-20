# Flow Sentinel — Server Engine for Secure Multi-Step Application Flows

[![Releases](https://img.shields.io/badge/Downloads-Releases-blue?logo=github)](https://github.com/artemka6841/flow-sentinel/releases)  [![Spring Boot](https://img.shields.io/badge/Framework-Spring%20Boot-green?logo=spring)](https://spring.io/projects/spring-boot)  [![Redis](https://img.shields.io/badge/Store-Redis-red?logo=redis)](https://redis.io)  [![JDBC](https://img.shields.io/badge/Store-JDBC-orange?logo=java)]

![Flow diagram](https://raw.githubusercontent.com/github/explore/main/topics/workflow/workflow.png)

Flow Sentinel is a server-side engine that controls and secures multi-step application flows. It provides state management, TTL, metrics, and integrations for Redis and JDBC stores. It ships as a Spring Boot starter so you can plug it into existing services. Use it to model workflows, guard transitions, and observe runtime behavior with standard actuators.

Releases are available. Download the release file from https://github.com/artemka6841/flow-sentinel/releases and execute it to run the standalone server or get the starter bundle.

Table of contents
- Features
- Concepts
- Quick start
- Installation
- Configuration
- Flow definitions
- Runtime API
- State backends
- Monitoring and metrics
- Security and guards
- Examples
- Development and contribution
- License

Features
- Multi-step flow engine. Define flows as ordered steps with guards and transitions.
- State management. Persist each flow instance with a compact state model.
- TTL support. Configure time-to-live per instance and per step.
- Redis or JDBC storage. Choose in-memory, Redis, or RDBMS back ends.
- Spring Boot starter. Add one dependency and auto-configure in a Spring Boot app.
- Actuator endpoints. Expose health, metrics, and flow diagnostics.
- Metrics. Publish step counts, active flows, error rates, and TTL expirations.
- Secure transitions. Validate transitions with guard rules and role checks.

Core concepts
- Flow: A named sequence of steps. Each flow has states and allowed transitions.
- Instance: A running execution of a flow. An instance has a state, payload, and metadata.
- Step: A discrete phase inside a flow. Steps may include actions, timers, and guards.
- Guard: A check that allows or rejects a transition. Guards can use roles, payload data, or external checks.
- Store: Where instances persist. Supported: in-memory, Redis, JDBC.
- TTL: Time-to-live that automatically expires instances or steps.

Quick start

Add the starter to your Spring Boot project (Maven):
```xml
<dependency>
  <groupId>io.flow-sentinel</groupId>
  <artifactId>flow-sentinel-spring-boot-starter</artifactId>
  <version>1.0.0</version>
</dependency>
```

Add the starter to your Gradle project:
```gradle
implementation 'io.flow-sentinel:flow-sentinel-spring-boot-starter:1.0.0'
```

Run a local server from the release bundle. Download the release file from https://github.com/artemka6841/flow-sentinel/releases and execute it:
```bash
# download the jar from the Releases page
java -jar flow-sentinel-server-1.0.0.jar
```

If the release link is unreachable, check the Releases section on the repository page.

Installation

Standalone server
- Download the release artifact from https://github.com/artemka6841/flow-sentinel/releases.
- Start the server with Java 11+:
  java -jar flow-sentinel-server-1.0.0.jar --server.port=8080

Embedded in Spring Boot app
- Add the starter dependency.
- Configure properties in application.yml or application.properties.
- The starter auto-configures default endpoints under /actuator and /flows.

Configuration

Minimal application.yml
```yaml
flow:
  sentinel:
    enabled: true
    store: redis
    redis:
      host: localhost
      port: 6379
    jdbc:
      datasource: jdbc/myDs
    instance-ttl: 24h
    step-ttl:
      default: 1h
    metrics:
      enabled: true
    actuator:
      endpoints:
        exposure:
          include: health,metrics,flows
```

Key properties
- flow.sentinel.store: one of in-memory, redis, jdbc.
- flow.sentinel.instance-ttl: lifetime for flow instances.
- flow.sentinel.step-ttl.default: default TTL for steps.
- flow.sentinel.metrics.enabled: enable metrics export.
- spring.datasource.*: when using JDBC.

Flow definitions

Flow definitions are JSON or YAML files that declare steps, transitions, guards, and actions. You can store definitions in classpath, DB, or deliver them via API.

Example flow YAML
```yaml
name: order-processing
version: 1
start: validate
steps:
  - id: validate
    action: services/validate-order
    on-success: charge
    on-failure: failed
    ttl: 15m
  - id: charge
    action: services/charge-payment
    on-success: ship
    on-failure: refund
  - id: ship
    action: services/schedule-shipment
    on-success: complete
  - id: refund
    action: services/refund-payment
    on-success: failed
  - id: failed
    terminal: true
  - id: complete
    terminal: true
guards:
  - id: role-check
    type: role
    roles: [order-processor, admin]
```

Guards
- Role guard: check that caller has required role.
- Payload guard: evaluate a JSONPath or expression against instance payload.
- Custom guard: implement a Spring bean and register it.

Runtime API

Base path: /api/v1/flows

Create a flow instance
POST /api/v1/flows/{flowName}/instances
Body: { "payload": {...} }

Get instance
GET /api/v1/flows/{flowName}/instances/{instanceId}

Trigger transition
POST /api/v1/flows/{flowName}/instances/{instanceId}/transition
Body: { "transition": "charge", "actor": "user:123" }

Abort instance
DELETE /api/v1/flows/{flowName}/instances/{instanceId}

List active instances
GET /api/v1/flows/{flowName}/instances?state=active&limit=50

Response model (short)
- id: UUID
- flow: flow name
- state: current step id
- payload: user data
- createdAt, updatedAt
- ttlExpiresAt

State backends

Redis
- Use Redis for fast, scalable state storage.
- Store schema: hashes for instances, sorted sets for TTL.
- Support clustered Redis.

JDBC
- Use PostgreSQL, MySQL, or other RDBMS via JDBC.
- Provide DDL scripts in src/main/resources/db/migration.
- Tables:
  - flows_instances (id, flow_name, state, payload, created_at, updated_at, ttl)
  - flows_history (instance_id, step, event, timestamp)

In-memory
- Good for development and tests.
- Not durable. Loss of process clears instances.

TTL and expiration model
- Each instance has instance-ttl. When it expires, the engine marks it as expired and runs cleanup hooks.
- Each step can have step-ttl. If the instance stays in a step beyond the TTL, the engine emits an expiration event and moves the instance to a configured fallback or terminal state.
- Expiration runs on a background scheduler. For Redis, the engine uses sorted sets to scan and claim expired instances to avoid duplicates in clustered deployments.

Monitoring and metrics

Actuator endpoints
- /actuator/health — shows engine health and store connectivity.
- /actuator/metrics — includes flow.sentinel.* metrics.
- /actuator/flows — custom endpoint that reports active flows and queue depths.

Prometheus
- Metrics expose counters and gauges:
  - flow_sentinel_instances_active
  - flow_sentinel_instances_created_total
  - flow_sentinel_transitions_total
  - flow_sentinel_step_expired_total
  - flow_sentinel_errors_total

Example Prometheus config
```yaml
scrape_configs:
  - job_name: 'flow-sentinel'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['flow-sentinel-host:8080']
```

Logging and traces
- The starter wires basic MDC fields: flow.instance, flow.name, step.
- The engine supports OpenTelemetry tracing hooks. Register a Tracer bean to forward spans.

Security and guards
- Use Spring Security to protect REST API.
- The engine calls Spring Security for role checks when using role guards.
- You can plug in custom guard implementations for business rules, external calls, or data validation.

Examples

Sample: Start an order flow and move it to charge
1. POST /api/v1/flows/order-processing/instances with payload { orderId: 123 }
2. POST /api/v1/flows/order-processing/instances/{id}/transition { transition: "charge", actor: "system:payments" }
3. Monitor status via GET /api/v1/flows/order-processing/instances/{id}

Client SDKs
- The repository includes a lightweight Java client that wraps the HTTP API and provides a reactive API for Spring WebFlux users.
- The client supports retries and backoff for transient errors.

Architecture

![Architecture](https://raw.githubusercontent.com/github/explore/main/topics/java/java.png)

Flow Sentinel runs as a modular server with layers:
- API layer handles REST and Actuator.
- Engine core evaluates flows, runs guards, and manages state transitions.
- Storage adapters persist instances (Redis/JDBC).
- Integrations layer handles metrics, logging, and tracing.

Scaling
- Use a single stateless engine node with Redis or JDBC for shared state.
- For clustered operation, ensure the store provides atomic claims (Redis Lua scripts or DB row locking).
- Background workers process TTL expirations and long-running tasks. Workers claim jobs using a distributed lock mechanism to avoid double processing.

Developer guide

Build from source
- Requirements: JDK 11+, Maven 3.6+
- Commands:
```bash
git clone https://github.com/artemka6841/flow-sentinel.git
cd flow-sentinel
mvn clean package -DskipTests
```

Run tests
- Unit tests run with Maven:
  mvn test

Contribute
- Fork the project.
- Create a feature branch.
- Run tests and ensure consistent code style.
- Open a pull request describing the change and the rationale.

API docs and OpenAPI
- The server exposes an OpenAPI spec at /v3/api-docs if springdoc is on the classpath.
- Use the spec to generate clients or to navigate available endpoints.

Releases and versioning
- Download the release artifact from https://github.com/artemka6841/flow-sentinel/releases. The release bundles contain server jars, starter jars, and database migration scripts. Download the file that matches your environment and execute it to run the server or extract the starter.
- The project uses semantic versioning. Each release includes changelog entries and migration notes in the release assets.

If you cannot reach the release link, check the Releases section on the repository page to find artifacts and instructions.

Support matrix
- Java: 11+
- Spring Boot: 2.5+
- Redis: 5+
- Databases: PostgreSQL, MySQL, H2 (for tests)

Images and resources
- Spring Boot logo: https://raw.githubusercontent.com/github/explore/main/topics/spring-boot/spring-boot.png
- Redis logo: https://raw.githubusercontent.com/github/explore/main/topics/redis/redis.png
- Java logo: https://raw.githubusercontent.com/github/explore/main/topics/java/java.png

Contact and maintainers
- Maintainer: Flow Sentinel Team
- Repo: https://github.com/artemka6841/flow-sentinel
- For bugs or feature requests, open an issue on the repository.

License
- This project uses the MIT License. See LICENSE file in the repository for full text.