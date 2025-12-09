# Auth Service — Low‑Level Design (LLD)

> Purpose: Detailed low‑level design for the **Auth Service** in the Microservices Mini‑Ecommerce project. Includes class diagrams (PlantUML), sequence diagrams, DTOs, API contracts (OpenAPI style), persistence, error handling strategy, security config, tests and logging suggestions.

---

## Table of contents

1. Goals & responsibilities
2. Tech stack & libraries
3. Data model
4. Class diagram (PlantUML)
5. Sequence diagrams (PlantUML)
6. DTOs & domain entities
7. API Contracts (OpenAPI style)
8. Error handling strategy
9. Security details (JWT, keys, rotation)
10. Logging, metrics & tracing
11. Testing strategy
12. Example folder structure
13. Appendix: helpful code snippets

---

## 1 — Goals & responsibilities

Auth Service responsibilities:

* User registration and login
* Password hashing and storage (BCrypt)
* Issue RS256 signed Access Tokens (JWT) and Refresh Tokens
* Persist refresh tokens (Redis) and support revocation
* Expose `GET /me` for authenticated user details
* Validate tokens (public key) for other services
* Provide rotating key support and expose JWKS endpoint (optional)

Non‑functional:

* Short token TTL (e.g., 15 minutes), refresh token TTL (e.g., 7 days)
* Secure storage of private keys
* High test coverage for security flows

---

## 2 — Tech stack & libraries

* Java 17
* Spring Boot 3.x
* Spring Security
* Spring Data JPA (MySQL/Postgres for users)
* Spring Data Redis (refresh tokens)
* JWT: `jjwt` or `nimbus-jose-jwt` (we’ll use Nimbus for JWKS support)
* BCrypt (Spring Security PasswordEncoder)
* Testcontainers, JUnit5, Mockito
* Lombok (optional)
* Actuator, Micrometer for metrics
* OpenTelemetry / Sleuth for tracing

---

## 3 — Data model

**users** (MySQL)

* id: UUID (PK)
* email: varchar(255) unique
* password_hash: varchar(255)
* roles: varchar(255) (CSV or separate join table if multi-role)
* full_name: varchar(255)
* created_at: timestamp
* updated_at: timestamp
* enabled: boolean

**refresh_tokens** (Redis) — store as key `refresh:{tokenId}` with value JSON:

* tokenId (uuid)
* userId
* expiresAt
* revoked: boolean

Alternative: store refresh tokens in relational DB for audit; Redis recommended for fast revocation and TTL.

---

## 4 — Class diagram (PlantUML)

Paste into PlantUML for rendering.
<img width="790" height="756" alt="image" src="https://github.com/user-attachments/assets/9d91927c-f396-49da-a29a-6c2bd037d250" />


---

## 5 — Sequence diagrams (PlantUML)

### 5.1 Login / Issue tokens

<img width="1466" height="604" alt="image" src="https://github.com/user-attachments/assets/7e7ea209-2b8d-4b8b-a3bb-870e634f7ae2" />


### 5.2 Refresh token flow

<img width="968" height="511" alt="image" src="https://github.com/user-attachments/assets/db93cdde-3761-4f2e-b3ee-18a3189d92fc" />


### 5.3 Revoke / Logout

<img width="743" height="330" alt="image" src="https://github.com/user-attachments/assets/554641e9-be88-470b-8d4b-a03a17e3116f" />


---

## 6 — DTOs & domain entities

### DTOs (Java classes)

**RegisterRequest**

```java
public class RegisterRequest {
  @Email
  private String email;
  @Size(min=8)
  private String password;
  private String fullName;
}
```

**LoginRequest**

```java
public class LoginRequest {
  private String email;
  private String password;
}
```

**RefreshRequest**

```java
public class RefreshRequest {
  private String refreshToken;
}
```

**AuthResponse**

```java
public class AuthResponse {
  private String accessToken;
  private String refreshToken; // optional if using httpOnly cookie
  private long expiresIn; // seconds
}
```

**MeResponse**

```java
public class MeResponse {
  private UUID id;
  private String email;
  private String fullName;
  private List<String> roles;
}
```

**ErrorResponse**

```java
public class ErrorResponse {
  private String code; // e.g., AUTH_INVALID_CREDENTIALS
  private String message;
  private String timestamp;
}
```

### Domain Entities (simplified)

**User (JPA Entity)**

* id (UUID)
* email
* passwordHash
* roles
* fullName
* enabled

**RefreshToken (Redis/Entity)**

* id (UUID string)
* userId (UUID)
* issuedAt (instant)
* expiresAt (instant)
* revoked (boolean)

---

## 7 — API Contracts (OpenAPI style)

Base path: `/api/v1/auth`

### 7.1 POST /register

* **Request**: `RegisterRequest`
* **Responses**:

  * `201 Created` — body: `{id, email, fullName}`
  * `400 Bad Request` — validation error
  * `409 Conflict` — email already exists

### 7.2 POST /login

* **Request**: `LoginRequest`
* **Responses**:

  * `200 OK` — `AuthResponse` {accessToken, refreshToken, expiresIn}
  * `401 Unauthorized` — invalid credentials
  * `429 Too Many Requests` — rate limit

### 7.3 POST /refresh

* **Request**: `RefreshRequest` (or httpOnly cookie)
* **Responses**:

  * `200 OK` — `AuthResponse` (new tokens)
  * `401 Unauthorized` — or `403 Forbidden` if token revoked/expired

### 7.4 POST /revoke

* **Request**: `{refreshToken}`
* **Responses**:

  * `200 OK` — token revoked
  * `400 / 404` — token not found

### 7.5 GET /me

* **Auth**: Bearer token
* **Responses**:

  * `200 OK` — `MeResponse`
  * `401 Unauthorized` — invalid/expired token

---

## 8 — Error handling strategy

### Principles

* Use **consistent error responses** (ErrorResponse) with `code` and `message`.
* Map exceptions to HTTP status codes centrally via `@ControllerAdvice`.
* Avoid leaking sensitive info in error messages.
* Log stack traces for server errors (500) with correlation ids (traceId) but return a generic message to clients.

### Example exception mapping

* `InvalidCredentialsException` → `401 Unauthorized` (code: AUTH_INVALID_CREDENTIALS)
* `UserAlreadyExistsException` → `409 Conflict` (code: AUTH_USER_EXISTS)
* `RefreshTokenNotFoundException` or `TokenExpiredException` → `401 Unauthorized` (code: AUTH_INVALID_REFRESH_TOKEN)
* `AccessDeniedException` → `403 Forbidden` (AUTH_ACCESS_DENIED)
* `ValidationException` → `400 Bad Request` (AUTH_VALIDATION_ERROR)
* `TooManyRequestsException` → `429 Too Many Requests` (AUTH_RATE_LIMIT)
* `Exception` → `500 Internal Server Error` (AUTH_INTERNAL_ERROR)

### Error response example

```json
{
  "code": "AUTH_INVALID_CREDENTIALS",
  "message": "Invalid email or password",
  "timestamp": "2025-12-09T12:00:00Z"
}
```

### Correlation & debugging

* Include `X-Request-Id` header on all responses; log it with every log line.
* Ensure `traceId` from OpenTelemetry is propagated in logs.

---

## 9 — Security details (JWT, keys, rotation)

### Access token

* **Algorithm**: RS256 (asymmetric)
* **Claims**: `sub` (userId), `email`, `roles`, `iat`, `exp`, `aud`, `kid`
* **TTL**: 15 minutes (configurable)
* **Signing**: private key; services verify with public key

**Sample JWT payload**

```json
{
  "sub":"d290f1ee-6c54-4b01-90e6-d701748f0851",
  "email":"venkat@example.com",
  "roles":["USER"],
  "iat":1700000000,
  "exp":1700000900,
  "aud":"micro-ecom",
  "kid":"key-v1"
}
```

### JWKS endpoint (optional)

* Provide `GET /.well-known/jwks.json` serving current public keys with `kid` for services to fetch.
* Useful when rotating keys.

### Refresh tokens

* Random opaque token (UUID) stored in Redis as key with TTL.
* Bind refresh token to userId and client (optional), allow revocation.
* Use httpOnly, Secure cookie for browser clients to avoid XSS stealing refresh token.

### Key rotation

* Keep multiple keys with `kid` in JWKS.
* Issue tokens with current `kid` → services use JWKS to verify.
* Rotate by adding new key (sign new tokens) and retiring old after its tokens expire.

---

## 10 — Logging, metrics & tracing

**Logs**

* Structured JSON logs with fields: `timestamp, service, level, message, traceId, requestId, userId, path`.
* Log login failures with reason (without password) and IP for security monitoring.

**Metrics** (Micrometer)

* `auth.login.attempts` (counter)
* `auth.login.success` (counter)
* `auth.login.failures` (counter)
* `auth.refresh.count`
* `auth.refresh.failures`
* `auth.token.revocations`

**Tracing**

* Instrument controllers and service entry points; propagate trace context to downstream services when validating tokens.

---

## 11 — Testing strategy

**Unit tests**

* AuthService: authenticate, register, refresh, revoke flows with mocked repos and jwt service.
* JwtService: token generation and validation edge cases.

**Integration tests**

* Use Testcontainers: MySQL and Redis for integration flows
* Test login → DB changes; refresh token life cycle; revoke flows

**Security tests**

* Test invalid token, expired token, revoked token cases
* Penetration smoke tests for SQL injection or improper error messages

**Contract tests**

* If other services consume JWKS or token validation endpoints, provide consumer driven contract tests.

---

## 12 — Example folder structure

```
auth-service/
  src/main/java/com/microecom/auth
    controller/
      AuthController.java
    service/
      AuthService.java
      JwtService.java
    repository/
      UserRepository.java
      RefreshTokenRepository.java
    security/
      SecurityConfig.java
      JwtAuthenticationFilter.java
    model/
      User.java
      RefreshToken.java
    dto/
      LoginRequest.java
      RegisterRequest.java
      AuthResponse.java
    exception/
      AuthExceptionHandlers.java
  src/test/java/...
  Dockerfile
  application.yml
  jwks/ (keys)
```

---

## 13 — Appendix: helpful code snippets

### 13.1 Example controller method (login)

```java
@PostMapping("/login")
public ResponseEntity<AuthResponse> login(@RequestBody @Valid LoginRequest req) {
  AuthResponse resp = authService.authenticate(req);
  return ResponseEntity.ok(resp);
}
```

### 13.2 Example exception handler snippet

```java
@ControllerAdvice
public class GlobalExceptionHandler {
  @ExceptionHandler(InvalidCredentialsException.class)
  public ResponseEntity<ErrorResponse> handleInvalidCreds(InvalidCredentialsException ex) {
    ErrorResponse err = new ErrorResponse("AUTH_INVALID_CREDENTIALS", "Invalid email or password", Instant.now().toString());
    return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(err);
  }
}
```

---

If you want, I can now:

* generate PlantUML PNG/SVG exports for the diagrams in this doc,
* create OpenAPI YAML for the auth service,
* scaffold a Spring Boot starter project (code skeleton + Dockerfile + test containers), or
* produce Liquibase migration SQL for `users` table.

Which one should I produce next?
