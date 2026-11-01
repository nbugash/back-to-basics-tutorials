# Chapter 4: Inbound REST and Outbound jOOQ Adapters

In this chapter, we bridge the external world to our core domain by implementing the generated REST interface and the persistence port using jOOQ and Quarkus CDI.

---

## 1. Outbound Persistence Adapter (jOOQ)

Create `src/main/java/com/example/orders/adapters/outbound/persistence/OrderJooqRepository.java`:

```java:src/main/java/com/example/orders/adapters/outbound/persistence/OrderJooqRepository.java
package com.example.orders.adapters.outbound.persistence;

import com.example.orders.domain.model.Order;
import com.example.orders.domain.model.OrderItem;
import com.example.orders.domain.model.OrderStatus;
import com.example.orders.ports.outbound.OrderRepositoryPort;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jooq.DSLContext;
import org.jooq.Record;

import java.math.BigDecimal;
import java.sql.Timestamp;
import java.time.Instant;
import java.util.*;

import static org.jooq.impl.DSL.*;

@ApplicationScoped
public class OrderJooqRepository implements OrderRepositoryPort {

    private final DSLContext dsl;

    @Inject
    public OrderJooqRepository(DSLContext dsl) {
        this.dsl = dsl;
    }

    @Override
    public void save(Order order) {
        dsl.transaction(configuration -> {
            var ctx = using(configuration);

            ctx.insertInto(table(name("orders")))
                .columns(
                    field(name("id"), UUID.class),
                    field(name("customer_email"), String.class),
                    field(name("total_amount"), BigDecimal.class),
                    field(name("status"), String.class),
                    field(name("created_at"), Timestamp.class)
                )
                .values(
                    order.id(),
                    order.customerEmail(),
                    order.totalAmount(),
                    order.status().name(),
                    Timestamp.from(order.createdAt())
                )
                .execute();

            for (OrderItem item : order.items()) {
                ctx.insertInto(table(name("order_items")))
                    .columns(
                        field(name("order_id"), UUID.class),
                        field(name("product_sku"), String.class),
                        field(name("quantity"), Integer.class),
                        field(name("unit_price"), BigDecimal.class)
                    )
                    .values(
                        order.id(),
                        item.productSku(),
                        item.quantity(),
                        item.unitPrice()
                    )
                    .execute();
            }
        });
    }

    @Override
    public Optional<Order> findById(UUID orderId) {
        Record orderRecord = dsl.select()
                .from(table(name("orders")))
                .where(field(name("id"), UUID.class).eq(orderId))
                .fetchOne();

        if (orderRecord == null) {
            return Optional.empty();
        }

        List<OrderItem> items = dsl.select()
                .from(table(name("order_items")))
                .where(field(name("order_id"), UUID.class).eq(orderId))
                .fetch(r -> new OrderItem(
                        r.get("product_sku", String.class),
                        r.get("quantity", Integer.class),
                        r.get("unit_price", BigDecimal.class)
                ));

        return Optional.of(new Order(
                orderRecord.get("id", UUID.class),
                orderRecord.get("customer_email", String.class),
                items,
                OrderStatus.valueOf(orderRecord.get("status", String.class)),
                orderRecord.get("total_amount", BigDecimal.class),
                orderRecord.get("created_at", Timestamp.class).toInstant()
        ));
    }

    @Override
    public List<Order> findAll() {
        return dsl.select(field(name("id"), UUID.class))
                .from(table(name("orders")))
                .fetch(r -> findById(r.value1()).orElseThrow());
    }
}
```

---

## 2. Inbound REST Adapter

Create `src/main/java/com/example/orders/adapters/inbound/rest/OrdersResource.java` implementing the generated OpenAPI interface:

```java:src/main/java/com/example/orders/adapters/inbound/rest/OrdersResource.java
package com.example.orders.adapters.inbound.rest;

import com.example.orders.adapters.inbound.rest.api.ApiV1OrdersApi;
import com.example.orders.adapters.inbound.rest.dto.CreateOrderRequest;
import com.example.orders.adapters.inbound.rest.dto.OrderResponse;
import com.example.orders.domain.model.Order;
import com.example.orders.domain.model.OrderItem;
import com.example.orders.ports.inbound.OrderUseCasePort;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.core.Response;

import java.math.BigDecimal;
import java.time.OffsetDateTime;
import java.time.ZoneOffset;
import java.util.List;
import java.util.UUID;

@ApplicationScoped
public class OrdersResource implements ApiV1OrdersApi {

    private final OrderUseCasePort orderUseCase;

    @Inject
    public OrdersResource(OrderUseCasePort orderUseCase) {
        this.orderUseCase = orderUseCase;
    }

    @Override
    public Response createOrder(CreateOrderRequest request) {
        List<OrderItem> items = request.getItems().stream()
                .map(i -> new OrderItem(
                        i.getProductSku(),
                        i.getQuantity(),
                        BigDecimal.valueOf(i.getUnitPrice())
                ))
                .toList();

        Order order = orderUseCase.createOrder(request.getCustomerEmail(), items);
        return Response.status(Response.Status.CREATED)
                .entity(toResponse(order))
                .build();
    }

    @Override
    public Response getOrderById(UUID orderId) {
        return orderUseCase.getOrder(orderId)
                .map(order -> Response.ok(toResponse(order)).build())
                .orElseGet(() -> Response.status(Response.Status.NOT_FOUND).build());
    }

    @Override
    public Response listOrders() {
        List<OrderResponse> responses = orderUseCase.listAllOrders()
                .stream()
                .map(this::toResponse)
                .toList();
        return Response.ok(responses).build();
    }

    private OrderResponse toResponse(Order order) {
        OrderResponse resp = new OrderResponse();
        resp.setId(order.id());
        resp.setCustomerEmail(order.customerEmail());
        resp.setTotalAmount(order.totalAmount().doubleValue());
        resp.setStatus(OrderResponse.StatusEnum.valueOf(order.status().name()));
        resp.setCreatedAt(OffsetDateTime.ofInstant(order.createdAt(), ZoneOffset.UTC));
        return resp;
    }
}
```

---

## 3. Application Properties

Configure Quarkus in `src/main/resources/application.properties`:

```properties:src/main/resources/application.properties
quarkus.http.port=8080

# Agroal & Postgres Datasource
quarkus.datasource.db-kind=postgresql
quarkus.datasource.jdbc.url=jdbc:postgresql://localhost:5432/orderdb
quarkus.datasource.username=postgres
quarkus.datasource.password=postgres
quarkus.datasource.jdbc.max-size=16

# Flyway Migration
quarkus.flyway.migrate-at-start=true
quarkus.flyway.baseline-on-migrate=true

# jOOQ
quarkus.jooq.dialect=POSTGRES

# Health Check & Metrics
quarkus.smallrye-health.root-path=/health
```