# Microservices Mini‑Ecommerce — Full SDLC Plan

> A complete, practical Software Development Life Cycle (SDLC) from requirements to deployment and monitoring for a **Microservices Mini‑Ecommerce** project. This document is structured so you can follow the same engineering process used in real teams: UML, architecture, detailed service specs, DB schemas, CI/CD, testing, deployment and sprint plan.

---

## 1 — Project Overview

**Goal:** Build a small but production‑like e‑commerce system composed of independent microservices demonstrating authentication, product catalog, order management and payments, with API Gateway, service discovery, inter‑service security (JWT), Redis caching and observability.

**High level services**:

* **auth-service**: user registration, login, JWT issuance, refresh tokens, roles
* **product-service**: product CRUD, search, inventory view
* **order-service**: create orders, checkout, order status, order history
* **payment-service**: simulate payments (card, wallet), payment verification
* **api-gateway**: route requests, rate limit, authentication enforcement
* **service-registry** (Eureka/Consul) or Kubernetes DNS for discovery
* **message-broker** (RabbitMQ / Kafka) for async events (order-created → payment)
* **redis** for caching product info and session/refresh-token store

**Non-functional goals:**

* Demo distributed auth via JWT, secure inter-service calls
* Implement caching and eventual consistency patterns
* Logging, tracing and metrics
* CI/CD with container images and automated tests
* Deployable on Kubernetes (or Docker Compose for local dev)

---

## 2 — Stakeholders & Users

* Product Manager (requirements)
* Backend Engineers (Java / Spring Boot)
* Frontend Engineers (Angular)
* DevOps (Kubernetes / CI)
* QA (integration + end‑to‑end tests)
* End users (customers)

---

## 3 — Functional Requirements

1. **User management**: sign up, sign in, role based (user, admin), refresh token
2. **Product catalog**: list products, get details, admin can add/update/delete products
3. **Inventory**: product stock check at order placement
4. **Checkout**: create order, reserve inventory, call payment, confirm order
5. **Payment**: simulate success/failure, webhook/callback to order service
6. **Order history**: user can view past orders
7. **Admin**: product management, view orders
8. **Search & filters**: basic search by name and category
9. **Resilience**: retries, circuit breaker for payment
10. **Observability**: metrics, logs, distributed tracing

---

## 4 — Non‑Functional Requirements (NFRs)

* **Latency**: 95th percentile API response < 500ms for catalog requests under light load
* **Availability**: Services should be stateless where possible; tolerate single node failures
* **Security**: JWTs, HTTPS assumed in production, input validation, rate limiting
* **Scalability**: Horizontal scaling of product and order services
* **Maintainability**: Clear modular code, API contracts, tests

---

## 5 — High Level Architecture

I'll provide PlantUML snippets (text) you can paste into [https://plantuml.com](https://plantuml.com) or any PlantUML renderer to view diagrams.

### 5.1 — Component Diagram (PlantUML)

<img width="1351" height="737" alt="image" src="https://github.com/user-attachments/assets/63b6f115-61da-4675-907a-14a5014cc42f" />


### 5.2 — Sequence Diagram: Checkout flow
<img width="1396" height="585" alt="image" src="https://github.com/user-attachments/assets/3ac26a5e-608b-4f85-8d3d-7a59dba48670" />


---

## 6 — Data Model (ERD)

Use a relational DB (MySQL/Postgres) for consistency for orders & inventory. Product service could use MySQL; optionally product catalog read models in MongoDB for search.

### Core tables (simplified)

**users**

* id (uuid)
* email
* password_hash
* role
* created_at

**products**

* id (uuid)
* name
* description
* price_cents (int)
* currency
* sku
* category
* stock_quantity (int)
* created_at

**orders**

* id (uuid)
* user_id
* total_cents
* currency
* status (PENDING, CONFIRMED, CANCELLED, FAILED)
* created_at

**order_items**

* id
* order_id
* product_id
* unit_price_cents
* quantity

**payments**

* id
* order_id
* amount_cents
* provider (SIMULATED)
* status (PENDING, SUCCESS, FAILED)
* created_at

---

## 7 — APIs (OpenAPI style — condensed)

All responses use JSON. Use versioned paths e.g. `/api/v1/...`.

### Auth Service

* `POST /api/v1/auth/register` — body: {email, password}

  * success: 201
* `POST /api/v1/auth/login` — body: {email, password}

  * returns: {accessToken, refreshToken, expiresIn}
* `POST /api/v1/auth/refresh` — body: {refreshToken}

  * returns new access token
* `GET /api/v1/auth/me` — header: Authorization: Bearer <token>

### Product Service

* `GET /api/v1/products` — query: ?q=&category=&page=&size=
* `GET /api/v1/products/{id}`
* `POST /api/v1/products` — admin only
* `PUT /api/v1/products/{id}` — admin only
* `DELETE /api/v1/products/{id}` — admin only

### Order Service

* `POST /api/v1/orders` — body: {items: [{productId, quantity}], shippingAddr}

  * flow: check stock → create order (PENDING) → publish OrderCreated
* `GET /api/v1/orders/{id}`
* `GET /api/v1/orders?userId=` — list user orders

### Payment Service

* `POST /api/v1/payments/process` — called by Payment microservice on OrderCreated
* `POST /api/v1/payments/webhook` — simulate third party callback

---

## 8 — Authentication & Inter‑service security

**User authentication:**

* Auth service issues JWT access tokens (short lived, e.g. 15m) and refresh tokens (stored in Redis with TTL and revocation support).
* Access token includes `sub` (user id), `roles`, `iat`, `exp`, `aud` claims.

**Inter-service auth:**
Options:

* **a. Propagate user JWT**: API Gateway validates token and forwards the bearer token; downstream services validate token signature using shared public key.
* **b. Service-to-service mTLS or client credentials JWT**: each service authenticates using a machine/service token when calling other service to avoid forwarding user tokens for privileged operations.

For this project: use **JWT propagation** for simplicity (gateway validates, services validate signature). Use asymmetric keys (RS256) so services only need the public key.

**Refresh token storage:** store refresh tokens in Redis keyed by token id with TTL. Support revocation on logout.

---

## 9 — Inter‑service communication patterns

* **Synchronous**: REST between API Gateway and services. Order → Product (stock check) done synchronously.
* **Asynchronous**: RabbitMQ/Kafka for eventual processing: OrderCreated → PaymentService consumes event and initiates payment. Also for notifications (email, SMS) as separate consumers.

**Idempotency:** all consumers should handle duplicate events (idempotent handlers). Use dedup store (Redis) for event ids.

---

## 10 — Caching strategy

* **Redis** for product reads: cache `GET /products/{id}` with TTL (e.g., 60s) and cache invalidation on product update or stock change.
* **Redis** for session and refresh tokens.
* Use cache‑aside pattern: service looks in Redis, if miss -> DB -> populate cache.

---

## 11 — Resilience & Observability

**Resilience**

* Retry with exponential backoff when calling external services (Payment service or DB) — use Resilience4j.
* Circuit breaker on payment provider calls.
* Bulkhead pattern not necessary for a demo but mention it.

**Observability**

* **Logging**: structured JSON logs (timestamp, service, level, traceId, spanId, context). Use Logback + logstash encoder.
* **Tracing**: OpenTelemetry / Zipkin — instrument incoming/outgoing HTTP calls to propagate trace ids.
* **Metrics**: Prometheus metrics exporter on each service; Grafana dashboards.
* **Error tracking**: Sentry or similar (optional)

---

## 12 — Persistence & Data Consistency

* Use relational DB for orders/products (ACID for order creation & stock decrement).
* Use DB transactions when reducing stock and creating order to avoid oversell (optimistic locking with `version` column or `SELECT FOR UPDATE`).
* Use eventual consistency for cross‑service updates: after payment success, publish event to update order status.

---

## 13 — Development Tech Stack

**Backend:** Java 17, Spring Boot (Spring Web, Spring Data JPA, Spring Security), Resilience4j, Spring Cloud Netflix (Eureka) or Spring Cloud Kubernetes

**DB:** MySQL or Postgres
**Cache:** Redis
**Message Broker:** RabbitMQ (simpler) or Kafka (if you want to show streaming knowledge)
**Frontend:** Angular 17 (or current stable), RxJS, Angular Material
**Container:** Docker
**Orchestration:** Kubernetes (minikube / kind for local), Helm for deployment charts
**CI/CD:** GitHub Actions or GitLab CI
**Testing:** JUnit 5, Mockito, Testcontainers for integration tests
**Monitoring:** Prometheus + Grafana, Jaeger/Zipkin for tracing

---

## 14 — Project Structure & Repos

Option A: **Mono‑repo** with folders for each service.
Option B: **Multi‑repo** (one repo per service) — closer to production.

Suggested folder structure (mono‑repo):

```
micro-ecom/
  infra/                # docker-compose, k8s manifests, scripts
  auth-service/
    src/
    pom.xml
  product-service/
    src/
    pom.xml
  order-service/
    src/
    pom.xml
  payment-service/
  api-gateway/
  web-frontend/
  docs/
  ci/                  # pipeline definitions
  README.md
```

---

## 15 — Coding Standards & Conventions

* Java: follow Google Java Style or company style; use Lombok sparingly; DTOs separate from Entities
* REST: resource-based URLs, use proper HTTP verbs and statuses
* Logging: use SLF4J; never log secrets
* Security: parameter validation, sanitize inputs, BCrypt for password hashing
* API versioning: `/api/v1/` endpoints

---

## 16 — CI / CD Pipeline (example: GitHub Actions)

**Stages**:

1. Build (maven/gradle), run unit tests
2. Static analysis (SpotBugs, Checkstyle)
3. Build Docker image and push to registry (DockerHub or GHCR)
4. Run integration tests (Testcontainers) optionally
5. Deploy to staging k8s via `kubectl` / Helm

**Branch strategy**: `main` (release), `develop` (integration), feature branches `feature/*`, PRs for code review.

**Secrets**: store in GH secrets or use HashiCorp Vault for production.

---

## 17 — Testing Strategy

* **Unit tests**: JUnit + Mockito, aim 60–80% coverage for business logic
* **Integration tests**: Testcontainers starting MySQL, Redis, RabbitMQ to test service interactions
* **Contract tests**: Pact for consumer-driven contract testing between services
* **E2E tests**: Postman / Playwright for API/Frontend

---

## 18 — K8s & Deployment Blueprint

* Each service runs in its own Deployment and Service
* Use HorizontalPodAutoscaler for product/order under load
* Use ConfigMaps for non-secret config and Secrets for keys
* Use an Ingress (NGINX) pointing to API Gateway
* Example Kubernetes concerns: liveness/readiness probes, resource limits

Minimal k8s artifacts to create:

* `deployment.yaml` for each service
* `service.yaml` ClusterIP
* `ingress.yaml` for API
* `secret.yaml` for JWT private key (base64 encoded) — for local dev use a mounted config

---

## 19 — Logging, Tracing & Dashboard

* Use centralized log aggregator: ELK stack (Elasticsearch + Logstash + Kibana) or Loki + Grafana
* Expose Prometheus metrics endpoint `/actuator/prometheus`
* Create basic Grafana dashboards: request rate, error rate, latency, CPU/Memory
* Configure distributed tracing (Jaeger)

---

## 20 — Security Checklist

* Use HTTPS in production
* Use RSA keypair for signing JWT (store private key securely)
* Rate limit endpoints at API Gateway
* Validate and sanitize all inputs
* Use parameterized queries / JPA to avoid SQL injection
* Regularly rotate keys and tokens in demo

---

## 21 — Sprint Plan & Milestones (8 weeks example)

**Sprint 0 (1 week)** — Setup, specifications: requirements, diagrams, repo layout, base CI, infra (docker compose) and shared libs.

**Sprint 1 (2 weeks)** — Auth + API Gateway + Basic frontend auth flows

* Auth: register/login/refresh
* Gateway: routing & token validation
* Frontend: login, register, token storage
* Unit tests and basic e2e

**Sprint 2 (2 weeks)** — Product service

* CRUD products, caching, admin UIs
* DB migrations (Liquibase)
* Integration tests

**Sprint 3 (2 weeks)** — Order & Payment

* Order create, stock check, transactional flow
* RabbitMQ wiring and Payment consumer
* Payment simulation and order status update
* Tracing + metrics

**Sprint 4 (1 week)** — Observability, Hardening & Deploy

* Prometheus/Grafana, Jaeger, logging
* CI/CD to staging, k8s manifests
* End‑to‑end testing, load tests

Deliverables: runnable system on local (docker‑compose) and staging (k8s), README and architecture docs.

---

## 22 — Example Implementation Details

### 22.1 — Access Token (JWT) Claims (example)

```json
{
  "alg":"RS256",
  "typ":"JWT"
}
{
  "sub":"user-uuid",
  "email":"venkat@example.com",
  "roles":["USER"],
  "iat":1680000000,
  "exp":1680000900,
  "aud":"micro-ecom"
}
```

**Key rotation**: keep key id (`kid`) in token header to allow rotation.

### 22.2 — Example Order Creation Flow (pseudocode)

1. Validate JWT, validate items
2. Begin DB transaction
3. For each item: `SELECT stock_quantity FROM products WHERE id = ? FOR UPDATE`
4. If stock sufficient, decrement stock, create order & items
5. Commit transaction
6. Publish OrderCreated event to MQ
7. Return order id with status `PENDING`

---

## 23 — Sample Folder Layout for a Service (product-service)

```
product-service/
  src/main/java/com/company/product
    controller/
    service/
    repository/
    dto/
    entity/
    config/
  src/test/java/...
  Dockerfile
  pom.xml
  application.yml
  deployment.yaml
```

---

## 24 — Useful Tools & Libraries

* Spring Boot, Spring Security
* Resilience4j
* Flyway / Liquibase
* Testcontainers
* OpenTelemetry / Jaeger
* Prometheus / Grafana
* RabbitMQ / Kafka
* Postman / Insomnia for API testing

---

## 25 — PR Template & Checklist (for code review)

**PR Title:** `[service-name] Short summary`

**Description:** What changed and why.

**Checklist:**

* [ ] Unit tests added
* [ ] Integration tests added (if applicable)
* [ ] Docs updated
* [ ] Security considerations checked
* [ ] Logging added for important events

---

## 26 — Risk & Mitigation

* **Oversell**: use DB transactions and optimistic locking.
* **Message duplication**: design idempotent handlers and dedup keys.
* **Secrets leak**: use Secrets manager.
* **Complexity**: for interview/demo, keep design minimal but show extensibility.

---

## 27 — Deliverables & What I can provide next (if you want)

I can produce any (or all) of the following right now (pick one or more):

* Full PlantUML export for diagrams (component, sequence, deployment)
* OpenAPI (Swagger) spec stubs for each service
* DB migration files (Liquibase) and sample seed data SQL
* Starter code skeletons (Spring Boot) for each service including Dockerfile
* Kubernetes deployment & Helm chart templates
* GitHub Actions pipeline YAML
* Angular starter app with auth flows

---

## 28 — Quick References / Commands

**Local dev:** use `docker-compose.yml` to start MySQL, Redis, RabbitMQ, and all services in dev mode.
**Build docker images:** `./mvnw -DskipTests package && docker build -t <name>:<tag> .`
**Run tests:** `mvn test` and `mvn verify` for integration tests

---

### Appendix: Useful PlantUML snippets (deployment)

```
@startuml
node "Kubernetes Cluster" {
  node "namespace: micro-ecom" {
    [auth-service] <<Deployment>>
    [product-service] <<Deployment>>
    [order-service] <<Deployment>>
    [payment-service] <<Deployment>>
    [api-gateway] <<Ingress>>
  }
}
@enduml
```

---

*Document created for Venkat — use this as the single source of truth to start designing, implementing and deploying your Microservices Mini‑Ecommerce project.*
