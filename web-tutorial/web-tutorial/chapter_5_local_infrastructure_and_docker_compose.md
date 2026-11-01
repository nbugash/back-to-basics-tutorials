# Chapter 5: Dockerfile and Local Docker Compose

In this chapter, we create an optimized multi-stage `Dockerfile` and orchestrate the service alongside PostgreSQL 18.6 using Docker Compose.

---

## 1. Multi-Stage Dockerfile

Create `src/main/docker/Dockerfile.jvm`:

```dockerfile:src/main/docker/Dockerfile.jvm
# Step 1: Build Stage
FROM eclipse-temurin:25-jdk-noble AS build
WORKDIR /workspace

# Copy build descriptors
COPY gradlew settings.gradle.kts build.gradle.kts /workspace/
COPY gradle /workspace/gradle
RUN ./gradlew --version

# Copy source and build
COPY src /workspace/src
RUN ./gradlew quarkusBuild --no-daemon -x test

# Step 2: Runtime Stage
FROM eclipse-temurin:25-jre-noble
ENV LANGUAGE='en_US:en'
WORKDIR /deployments

# Run as non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser

COPY --chown=appuser:appgroup --from=build /workspace/build/quarkus-app/lib/ /deployments/lib/
COPY --chown=appuser:appgroup --from=build /workspace/build/quarkus-app/*.jar /deployments/
COPY --chown=appuser:appgroup --from=build /workspace/build/quarkus-app/app/ /deployments/app/
COPY --chown=appuser:appgroup --from=build /workspace/build/quarkus-app/quarkus/ /deployments/quarkus/

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/deployments/quarkus-run.jar"]
```

---

## 2. Docker Compose Configuration

Create `docker-compose.yml` in the project root:

```yaml:docker-compose.yml
services:
  postgres:
    image: postgres:18-alpine
    container_name: order-postgres
    restart: always
    environment:
      POSTGRES_DB: orderdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d orderdb"]
      interval: 5s
      timeout: 5s
      retries: 5

  order-service:
    build:
      context: .
      dockerfile: src/main/docker/Dockerfile.jvm
    container_name: order-service
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "8080:8080"
    environment:
      QUARKUS_DATASOURCE_JDBC_URL: jdbc:postgresql://postgres:5432/orderdb
      QUARKUS_DATASOURCE_USERNAME: postgres
      QUARKUS_DATASOURCE_PASSWORD: postgres

volumes:
  postgres_data:
```

---

## 3. Running & Verifying Local Stack

Start services:

```bash
docker compose up --build -d
```

Verify health check:

```bash
curl -f http://localhost:8080/health
```

Place an order via the contract-generated endpoint:

```bash
curl -X POST http://localhost:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerEmail": "architect@example.com",
    "items": [
      {
        "productSku": "KEYBOARD-01",
        "quantity": 1,
        "unitPrice": 149.50
      },
      {
        "productSku": "MOUSE-02",
        "quantity": 2,
        "unitPrice": 45.00
      }
    ]
  }' | jq .
```

List persisted orders:

```bash
curl -s http://localhost:8080/api/v1/orders | jq .
```

Tear down the stack:

```bash
docker compose down -v
```