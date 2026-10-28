# Chapter 4: Persistence with PostgreSQL 18.6 and jOOQ

**Milestone:** Configure Flyway migrations for PostgreSQL 18.6 and implement the outbound `OrderRepositoryPort` using jOOQ via `quarkus-jooq`.

---

## 1. Flyway Migration

Create `src/main/resources/db/migration/V1.0.0__create_orders_table.sql`:

```sql
CREATE TABLE IF NOT EXISTS orders (
    id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    total_amount NUMERIC(15, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    status VARCHAR(32) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_orders_customer_id ON orders(customer_id);
```

---

## 2. Quarkus & jOOQ Configuration

Add database properties to `src/main/resources/application.properties`:

```properties
# PostgreSQL 18.6 Connection Settings
quarkus.datasource.db-kind=postgresql
quarkus.datasource.username=app_user
quarkus.datasource.password=app_secret
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/order_db
quarkus.datasource.jdbc.max-size=20

# Flyway Configuration
quarkus.flyway.migrate-at-start=true
quarkus.flyway.locations=db/migration

# Quarkus jOOQ Config
quarkus.jooq.dialect=POSTGRES
```

---

## 3. Implementing the jOOQ Outbound Adapter

Implement `OrderRepositoryPort` via Quarkus CDI injection of the jOOQ `DSLContext`:

```java
package com.company.order.infrastructure.adapter.outbound.persistence;

import com.company.order.application.port.out.OrderRepositoryPort;
import com.company.order.domain.model.Order;
import com.company.order.domain.model.OrderStatus;
import jakarta.enterprise.context.ApplicationScoped;
import org.jooq.DSLContext;
import org.jooq.Record;
import org.jooq.impl.DSL;

import java.math.BigDecimal;
import java.time.Instant;
import java.time.OffsetDateTime;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import static org.jooq.impl.DSL.field;
import static org.jooq.impl.DSL.table;

@ApplicationScoped
public class JooqOrderRepositoryAdapter implements OrderRepositoryPort {

    private final DSLContext dsl;
    private static final org.jooq.Table<?> ORDERS_TABLE = table("orders");

    public JooqOrderRepositoryAdapter(DSLContext dsl) {
        this.dsl = dsl;
    }

    @Override
    public void save(Order order) {
        dsl.insertInto(ORDERS_TABLE)
                .set(field("id", UUID.class), order.getId())
                .set(field("customer_id", UUID.class), order.getCustomerId())
                .set(field("total_amount", BigDecimal.class), order.getTotalAmount())
                .set(field("currency", String.class), order.getCurrency())
                .set(field("status", String.class), order.getStatus().name())
                .set(field("created_at", OffsetDateTime.class), order.getCreatedAt().atOffset(java.time.ZoneOffset.UTC))
                .onConflict(field("id", UUID.class))
                .doUpdate()
                .set(field("status", String.class), order.getStatus().name())
                .execute();
    }

    @Override
    public Optional<Order> findById(UUID id) {
        Record record = dsl.select()
                .from(ORDERS_TABLE)
                .where(field("id", UUID.class).eq(id))
                .fetchOne();

        return Optional.ofNullable(record).map(this::mapRecordToDomain);
    }

    @Override
    public List<Order> findAll() {
        return dsl.select()
                .from(ORDERS_TABLE)
                .fetch()
                .map(this::mapRecordToDomain);
    }

    private Order mapRecordToDomain(Record record) {
        return new Order(
                record.get(field("id", UUID.class)),
                record.get(field("customer_id", UUID.class)),
                record.get(field("total_amount", BigDecimal.class)),
                record.get(field("currency", String.class)),
                OrderStatus.valueOf(record.get(field("status", String.class))),
                record.get(field("created_at", OffsetDateTime.class)).toInstant()
        );
    }
}
```

---

## 4. Verification Checkpoint

Run a quick unit/integration test using Quarkus Dev Services or an embedded database to verify jOOQ table bindings:

```bash
./gradlew test
```