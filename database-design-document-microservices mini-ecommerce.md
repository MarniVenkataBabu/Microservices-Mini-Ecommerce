Database Design Document — Microservices Mini-Ecommerce

Below is a complete database design document covering the ERD, table definitions, relationships, constraints, indexes, and both Liquibase and Flyway migration examples (so you can pick either).
I’ll assume Postgres as the relational engine (good ACID guarantees, uuid type, tsvector/full-text options). If you prefer MySQL, tell me and I’ll convert types & syntax.

1 — Summary / scope

This DB design covers the authoritative relational data for the microservices that require ACID correctness:

Auth / users (users)

Product catalog (products, categories, product_images)

Inventory (inventory, reservations)

Orders (orders, order_items)

Payments (payments)

Auxiliary (audit columns, optimistic locking version)

(Short-lived / scalable caches and token stores such as Redis are not covered here — refresh tokens and dedup keys will be stored in Redis in runtime; an optional refresh_tokens table is provided if you prefer DB storage.)

2 — ERD (PlantUML)

<img width="1213" height="814" alt="image" src="https://github.com/user-attachments/assets/c97f95d7-6a22-4562-b182-eab096a38ecf" />



3 — Table definitions (Postgres DDL style)

Below are the schemas with column types, constraints and recommended indexes.

NOTE: Use uuid_generate_v4() extension for uuid default values (CREATE EXTENSION IF NOT EXISTS "uuid-ossp";) — or rely on application-generated UUIDs.


