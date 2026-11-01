# Chapter 2: Contract-First API Design and Code Generation

Contract-first design guarantees that API consumers and producers align before writing business code. In this chapter, we create the OpenAPI 3.1 contract and configure Gradle to auto-generate the JAX-RS API interfaces and models.

---

## 1. OpenAPI Specification

Create `src/main/openapi/order-service-api.yaml`:

```yaml:src/main/openapi/order-service-api.yaml
openapi: 3.1.0
info:
  title: Order Management Service API
  version: 1.0.0
  description: Hexagonal Architecture Order Processing Engine
paths:
  /api/v1/orders:
    post:
      summary: Place a new order
      operationId: createOrder
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: Order created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderResponse'
        '400':
          description: Invalid request payload
    get:
      summary: List all orders
      operationId: listOrders
      responses:
        '200':
          description: List of stored orders
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/OrderResponse'

  /api/v1/orders/{orderId}:
    get:
      summary: Get order details by ID
      operationId: getOrderById
      parameters:
        - name: orderId
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Order details
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderResponse'
        '404':
          description: Order not found

components:
  schemas:
    CreateOrderRequest:
      type: object
      required:
        - customerEmail
        - items
      properties:
        customerEmail:
          type: string
          format: email
          example: "dev@example.com"
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItemPayload'
    OrderItemPayload:
      type: object
      required:
        - productSku
        - quantity
        - unitPrice
      properties:
        productSku:
          type: string
          example: "PROD-1001"
        quantity:
          type: integer
          minimum: 1
          example: 2
        unitPrice:
          type: number
          format: double
          example: 49.99
    OrderResponse:
      type: object
      required:
        - id
        - customerEmail
        - totalAmount
        - status
        - createdAt
      properties:
        id:
          type: string
          format: uuid
        customerEmail:
          type: string
        totalAmount:
          type: number
          format: double
        status:
          type: string
          enum: [PENDING, CONFIRMED, SHIPPED, CANCELLED]
        createdAt:
          type: string
          format: date-time
```

---

## 2. Configure OpenAPI Generator in Gradle

Append the following generator task configuration to `build.gradle.kts`:

```kotlin
val generatedSourcesDir = layout.buildDirectory.dir("generated/openapi").get().asFile

openApiGenerate {
    generatorName.set("jaxrs-spec")
    inputSpec.set("$rootDir/src/main/openapi/order-service-api.yaml")
    outputDir.set(generatedSourcesDir.absolutePath)
    apiPackage.set("com.example.orders.adapters.inbound.rest.api")
    modelPackage.set("com.example.orders.adapters.inbound.rest.dto")
    configOptions.putAll(
        mapOf(
            "interfaceOnly" to "true",
            "useJakartaEe" to "true",
            "useSwaggerAnnotations" to "false",
            "returnResponse" to "true",
            "generatePom" to "false",
            "openApiNullable" to "false"
        )
    )
}

sourceSets {
    main {
        java {
            srcDir("$generatedSourcesDir/src/gen/java")
        }
    }
}

tasks.named("compileJava") {
    dependsOn("openApiGenerate")
}
```

---

## 3. Verify Code Generation

Run the generation task:

```bash
./gradlew openApiGenerate
```

Generated interfaces such as `ApiV1OrdersApi.java` and record DTOs like `OrderResponse.java` and `CreateOrderRequest.java` will be available under `build/generated/openapi/src/gen/java`.