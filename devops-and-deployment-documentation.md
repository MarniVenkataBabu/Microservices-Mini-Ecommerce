# DevOps & Deployment Documentation (Microservices Mini‑Ecommerce)

This document contains all DevOps‑related specifications for the **Mini‑Ecommerce Microservices System** (Auth, Product, Order, Payment Services).

It includes:

1. CI/CD Pipelines (GitHub Actions)
2. Environment Strategy (dev → staging → prod)
3. Dockerfiles (for each service)
4. Kubernetes YAMLs (Deployments, Services, ConfigMaps, Secrets, Ingress)
5. Logging, Monitoring & Alerting Setup
6. Deployment Strategy (blue/green, rolling updates)
7. Versioning & Release Process

---

# 1. CI/CD PIPELINES (GitHub Actions)

## 1.1 Repo Structure

```
root/
  auth-service/
  product-service/
  order-service/
  payment-service/
  k8s/
  .github/workflows/
```

## 1.2 Common CI Workflow (build + test + docker build)

**File:** `.github/workflows/ci.yml`

```yaml
name: CI Build
on:
  push:
    branches: [main, dev]
  pull_request:

jobs:
  build-test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Build all microservices
        run: |
          mvn -q -DskipTests=false clean install

      - name: Build Docker images
        run: |
          docker build -t ${{ github.repository }}-auth ./auth-service
          docker build -t ${{ github.repository }}-product ./product-service
          docker build -t ${{ github.repository }}-order ./order-service
          docker build -t ${{ github.repository }}-payment ./payment-service
```

## 1.3 CD Workflow — Push to Docker Hub + Deploy to Kubernetes

**File:** `.github/workflows/cd.yml`

```yaml
name: Deploy to Kubernetes
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Login to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build & Push Images
        run: |
          docker build -t myrepo/auth-service:latest ./auth-service
          docker push myrepo/auth-service:latest

          docker build -t myrepo/product-service:latest ./product-service
          docker push myrepo/product-service:latest

          docker build -t myrepo/order-service:latest ./order-service
          docker push myrepo/order-service:latest

          docker build -t myrepo/payment-service:latest ./payment-service
          docker push myrepo/payment-service:latest

      - name: Configure kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: v1.29.0

      - name: Apply Kubernetes Manifests
        run: |
          kubectl apply -f k8s/
```

---

# 2. ENVIRONMENTS

## 2.1 Environment Setup

| Environment | Purpose               | Deployment        | URL                     |
| ----------- | --------------------- | ----------------- | ----------------------- |
| **Local**   | Developer machine     | Docker Compose    | localhost:8080          |
| **Dev**     | Integration with team | CI auto deploy    | dev.api.company.com     |
| **Staging** | Pre-production        | Manual promote    | staging.api.company.com |
| **Prod**    | Live traffic          | Manual + approval | api.company.com         |

## 2.2 Environment Variables

Stored in **GitHub Secrets / Kubernetes Secrets**.

```
DB_URL=
DB_USERNAME=
DB_PASSWORD=
JWT_SECRET=
RABBITMQ_URL=
REDIS_URL=
PAYMENT_GATEWAY_KEY=
```

---

# 3. DOCKERFILES (for each service)

## 3.1 Base Dockerfile Template

```
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY . .
RUN mvn -q -DskipTests clean package

FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Each service uses the same Dockerfile.

---

# 4. KUBERNETES DEPLOYMENT

Folder: `/k8s/`

## 4.1 Namespace

```yaml
apiVersion: v1\kind: Namespace
metadata:
  name: ecommerce
```

## 4.2 ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ecommerce-config
  namespace: ecommerce
data:
  SPRING_PROFILES_ACTIVE: prod
```

## 4.3 Secrets (sample)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ecommerce-secrets
  namespace: ecommerce
type: Opaque
stringData:
  DB_USERNAME: root
  DB_PASSWORD: root
  JWT_SECRET: supersecretkey
```

---

# 4.4 Deployment Template

(Listed once because all services follow the same structure.)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: ecommerce
spec:
  replicas: 2
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
        - name: auth-service
          image: myrepo/auth-service:latest
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: ecommerce-config
            - secretRef:
                name: ecommerce-secrets
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
```

---

# 4.5 Service (LoadBalancer or ClusterIP)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  namespace: ecommerce
spec:
  type: ClusterIP
  selector:
    app: auth-service
  ports:
    - port: 80
      targetPort: 8080
```

---

# 4.6 Ingress (API Gateway)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
spec:
  rules:
    - host: api.company.com
      http:
        paths:
          - path: /auth
            pathType: Prefix
            backend:
              service:
                name: auth-service
                port:
                  number: 80
          - path: /products
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 80
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
          - path: /payments
            pathType: Prefix
            backend:
              service:
                name: payment-service
                port:
                  number: 80
```

---

# 5. LOGGING, MONITORING, ALERTING

## 5.1 Logging

* JSON logs
* traceId propagated across services
* ELK/Loki recommended

## 5.2 Monitoring (Micrometer + Prometheus)

* JVM metrics
* HTTP request metrics
* Database connection pool metrics
* Message queue lag
* Custom metrics:

  * `orders.created.count`
  * `payments.success.rate`

## 5.3 Dashboards (Grafana)

* Error rate panel
* Latency per service
* RabbitMQ queue depth
* CPU/Mem usage per pod

## 5.4 Alerting

* High failure rate alert
* Pod restart loop
* DLQ message count exceeds threshold

---

# 6. DEPLOYMENT STRATEGY

## 6.1 Rolling Updates (default)

* Zero downtime
* Always recommended for microservices

## 6.2 Blue/Green Deployment

* Used for risky releases
* Traffic switched using Ingress

## 6.3 Canary Deployment

* Release 5%, then 25%, 50%, 100%
* Requires service mesh (Istio/Linkerd)

---

# 7. VERSIONING & RELEASE PROCESS

Semantic Versioning:

```
MAJOR.MINOR.PATCH
```

Example:

* `1.0.0`: Initial release
* `1.1.0`: New features added
* `1.1.1`: Patch fix

Release Branch Strategy:

```
main → stable
release/* → pre-prod
feature/* → new features
dev → integration environment
```

GitHub Release Tags:

```
v1.0.0
v1.1.0
```

---

# Want Next?

Say **“Give me the Final SRS document”** and I will generate the entire company-level SRS in a single file.md.
