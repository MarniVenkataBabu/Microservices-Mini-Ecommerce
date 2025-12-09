# High‑Level Architecture (HLA) — Microservices Mini‑Ecommerce

> Purpose: provide a single, shareable architecture artifact that maps the entire system at a high level. This document is intended to bridge the SDLC plan and low‑level designs (LLDs). Use it in design reviews, onboarding, and as a blueprint for implementation.

---

## 1 — Document Scope

This HLA covers:

* system components and boundaries
* runtime request flows (synchronous + async)
* deployment topology options (Docker Compose, Kubernetes)
* service responsibilities and constraints
* security and inter‑service auth model
* observability and operational considerations

It is intentionally implementation‑agnostic but includes recommended technologies.

---

## 2 — One‑Line Summary

An API Gateway fronts an Angular client and external clients, routing requests to four primary microservices (Auth, Product, Order, Payment). Services are discoverable via a Service Registry or Kubernetes DNS. Redis provides caching and token storage; a relational DB (MySQL/Postgres) stores core data; RabbitMQ/Kafka handles async events. Observability is provided by OpenTelemetry + Prometheus + Grafana.

---

## 3 — Component Diagram (text for PlantUML)

Paste into PlantUML to render a diagram.

```plantuml
@startuml
skinparam componentStyle rectangle
package "Clients" {
  [Angular Web App]
  [Mobile App]
}
package "Edge" {
  [API Gateway]
  [Ingress]
}
package "Core Services" {
  [Auth Service]
  [Product Service]
  [Order Service]
  [Payment Service]
}
package "Platform" {
  [Service Registry]
  [Config Server]
  [Message Broker]
  [Redis]
}
package "Persistence" {
  [MySQL/Postgres]
  [MongoDB (optional read model)]
}

[Angular Web App] --> [API Gateway]
[API Gateway] --> [Auth Service]
[API Gateway] --> [Product Service]
[API Gateway] --> [Order Service]
[API Gateway] --> [Payment Service]
[Order Service] --> [Product Service] : synchronous stock check
[Order Service] --> [Message Broker] : publish OrderCreated
[Payment Service] --> [Message Broker] : subscribe OrderCreated
[Auth Service] --> [Redis] : refresh token store
[Product Service] --> [Redis] : product cache
[Product Service] --> [MySQL/Postgres]
[Order Service] --> [MySQL/Postgres]
[Payment Service] --> [MySQL/Postgres]
@enduml
```

---

## 4 — Runtime Views

### 4.1 — Client Checkout (happy path)

1. User (Angular) calls `POST /api/v1/orders` with Authorization header.
2. API Gateway validates/access token (or forwards for service validation) and routes to Order Service.
3. Order Service synchronously calls Product Service `GET /products/{id}` to verify stock using `SELECT FOR UPDATE` or optimistic lock.
4. If stock reserved, Order Service creates order in DB with status `PENDING`, reduces stock in DB transaction, and publishes `OrderCreated` to Message Broker.
5. Payment Service consumes `OrderCreated`, processes payment (simulate), writes payment record, and publishes `PaymentResult`.
6. Order Service consumes `PaymentResult` and updates order `CONFIRMED` or `FAILED` and emits `OrderUpdated` for other consumers.
7. API returns created order to client (initially `PENDING`, followed by updates via polling or WebSocket if implemented).

### 4.2 — Auth flow

* Login → Auth Service issues short‑lived JWT (RS256) + refresh token stored in Redis.
* API Gateway validates JWT on ingress; services also optionally validate.
* Logout → revoke refresh token in Redis.

### 4.3 — Asynchronous Notifications

* Notification service (optional) subscribes to `OrderUpdated` and sends emails/SMS.

---

## 5 — Deployment Topologies

### Option A: Local / Dev — Docker Compose

* Starts: MySQL, Redis, RabbitMQ, Service Registry (Eureka), API Gateway, 4 services, Angular dev server.
* Use for debugging, local integration tests.

### Option B: Staging / Prod — Kubernetes

* Each service runs in its Deployment with HorizontalPodAutoscaler.
* API Gateway exposed via Ingress Controller (NGINX / Traefik).
* Config via ConfigMaps, secrets in Kubernetes Secrets or vault.
* Service discovery via Kubernetes DNS (prefer) or Eureka if using Spring Cloud.
* Use PersistentVolumeClaims for DB storage.

Network considerations:

* Use NetworkPolicies to restrict traffic (only API Gateway accessible externally).
* Use TLS (Ingress?) terminating at the gateway in production.

---

## 6 — Service Responsibilities (brief)

### Auth Service

* Register, login, refresh, revoke tokens
* Issue JWTs (RS256), hold key material
* Store refresh tokens in Redis

### Product Service

* Product CRUD, search, categories
* Maintain stock quantity
* Cache product reads in Redis

### Order Service

* Validate cart, perform stock reservation/transactional order creation
* Publish `OrderCreated`
* Update order lifecycle based on payments

### Payment Service

* Consume `OrderCreated`, simulate payment flow
* Provide webhook or publish `PaymentResult`
* Implement retries, idempotency and circuit breaker

### API Gateway

* Routing, rate limiting, authentication gateway, basic request validation
* Central place to enforce CORS, headers, and versioning

---

## 7 — Security Model

* **Access tokens**: short‑lived JWTs signed with RS256; include `roles` and `aud`.
* **Refresh tokens**: stored in Redis with TTL; revocable.
* **Inter-service**: validate token signature (public key) or use service tokens for sensitive privileged calls.
* **Secrets**: private keys in Secrets manager (K8s secrets for demo) — never hardcode.
* **Transport**: HTTPS for production; mTLS optional for extra security.
* **Rate limiting**: at API Gateway to protect services from spikes.

---

## 8 — Observability & Operations

* **Metrics**: Prometheus metrics via Spring Boot actuator `/actuator/prometheus`.
* **Tracing**: OpenTelemetry/Jaeger with trace id propagation across services.
* **Logging**: structured JSON logs (include traceId) shipped to Loki/ELK.
* **Dashboards**: Grafana dashboards for latency, error rates, throughput per service.
* **Alerting**: basic alerts on error rate spike, high latency, or service down.

Operational runbooks:

* How to restart service, clear cache, rollback deploy, rotate keys.
* How to run integration tests via Testcontainers in CI.

---

## 9 — Scalability & Reliability Patterns

* **Stateless services**: use DB / Redis for state.
* **Horizontal scaling**: HPA based on CPU / request latency.
* **Circuit Breakers**: Resilience4j for external calls (payment/provider)
* **Retries & Backoff**: for transient failures on message consumption and HTTP calls.
* **Bulkheads**: isolate resources if implementing heavy background jobs.
* **Database scaling**: read replicas for product read scale; use CQRS for read models if needed.

---

## 10 — Cross‑Cutting Concerns

* **API versioning**: `/api/v1/` on endpoints.
* **Feature flags**: toggle features (e.g., payment provider simulation)
* **Config management**: central config server or environment variables.
* **DB migrations**: Liquibase/Flyway per service migrations.

---

## 11 — Integration Points & Third‑Party Services

* Email provider (SendGrid / SES) for notifications (optional)
* Payment gateway (simulated) — design abstraction to plug real provider later
* External logging / error tracker (Sentry) optional

---

## 12 — Risks & Mitigations (HLA level)

* **Overselling stock**: use DB transactions + pessimistic lock on critical path.
* **Single point of failure (Gateway / Registry)**: use redundancy and K8s to replicate.
* **Event duplication**: event idempotency store (Redis) and idempotent handlers.

---

## 13 — Next Steps (what to produce next)

1. Low‑Level Design (LLD) per service (class diagrams, DTOs, sequence diagrams).
2. OpenAPI (Swagger) spec for each service.
3. DB migration scripts and ERD (detailed schema).
4. Starter repo skeletons and basic CI pipeline.

---

## 14 — Appendix: Useful PlantUML snippets

**Sequence diagram — Order Checkout**

```plantuml
@startuml
actor User
participant "Angular" as UI
participant "API Gateway" as GW
participant "Auth Service" as Auth
participant "Order Service" as Order
participant "Product Service" as Product
participant "Message Broker" as MQ
participant "Payment Service" as Pay

User -> UI: Checkout
UI -> GW: POST /orders (JWT)
GW -> Auth: validate JWT
GW -> Order: POST /orders
Order -> Product: GET /products/{id} (lock)
Product --> Order: stock ok
Order -> Order: create order in tx
Order -> MQ: publish OrderCreated
MQ -> Pay: deliver
Pay -> MQ: publish PaymentResult
MQ -> Order: deliver
Order -> UI: return 201
@enduml
```

---

*Document created for Venkat. Use this HLA as the canonical architecture artifact before we dive into LLDs and implementation skeletons.*
