# Chapter 5: Docker Containerization & Docker Compose

**Milestone:** Package the Java 25 Quarkus application into a container image and orchestrate an end-to-end environment with PostgreSQL 18.6 using Docker Compose.

---

## 1. Multi-Stage Dockerfile with Java 25

Create `src/main/docker/Dockerfile.jvm`:

```dockerfile
# Stage 1: Build & Package
FROM eclipse-temurin:25-jdk-alpine AS builder
WORKDIR /workspace

# Copy Gradle distribution & descriptors
COPY gradle/ gradle/
COPY gradlew build.gradle.kts settings.gradle.kts ./
RUN ./gradlew dependencies --no-daemon

# Copy source and compile
COPY src/ src/
RUN ./gradlew build -x test --no-daemon

# Stage 2: Minimal Distroless / JRE Runtime
FROM eclipse-temurin:25-jre-alpine
WORKDIR /deployments

ENV LANGUAGE='en_US:en'

COPY --from=builder /workspace/build/quarkus-app/lib/ /deployments/lib/
COPY --from=builder /workspace/build/quarkus-app/*.jar /deployments/
COPY --from=builder /workspace/build/quarkus-app/app/ /deployments/app/
COPY --from=builder /workspace/build/quarkus-app/quarkus/ /deployments/quarkus/

EXPOSE 8080 8443
USER 1001

ENTRYPOINT ["java", "--enable-preview", "-jar", "/deployments/quarkus-run.jar"]
```

---

## 2. Docker Compose Orchestration

Create `docker-compose.yml` in the root of the project:

```yaml
services:
  postgres:
    image: postgres:18.6-alpine
    container_name: order-postgres
    restart: always
    environment:
      POSTGRES_DB: order_db
      POSTGRES_USER: app_user
      POSTGRES_PASSWORD: app_secret
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app_user -d order_db"]
      interval: 5s
      timeout: 5s
      retries: 5

  order-service:
    build:
      context: .
      dockerfile: src/main/docker/Dockerfile.jvm
    container_name: order-service-app
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      QUARKUS_DATASOURCE_JDBC_URL: jdbc:postgresql://postgres:5432/order_db
      QUARKUS_DATASOURCE_USERNAME: app_user
      QUARKUS_DATASOURCE_PASSWORD: app_secret
    ports:
      - "8080:8080"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/ready"]
      interval: 10s
      timeout: 5s
      retries: 3

volumes:
  pgdata:
```

---

## 3. End-to-End Verification Checkpoint

Start the stack:

```bash
docker compose up --build -d
```

Verify application health and create a sample order:

```bash
# Check Readiness
curl -i http://localhost:8080/health/ready

# Place Order via Inbound Contract
curl -X POST http://localhost:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "d3b07384-d113-4944-9c8e-c5e3d7b8d4f6",
    "totalAmount": 199.95,
    "currency": "USD"
  }'
```