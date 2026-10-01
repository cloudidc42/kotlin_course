# Part 80: โปรเจกต์สมบูรณ์ - E-Commerce Platform

## สารบัญ
1. [Architecture Overview](#architecture-overview)
2. [Project Setup](#project-setup)
3. [Domain Model](#domain-model)
4. [API Layer](#api-layer)
5. [Service Layer](#service-layer)
6. [Infrastructure](#infrastructure)

---

## Architecture Overview

```
E-Commerce Platform: Microservices Architecture

┌─────────────────────────────────────────────┐
│                   Frontend                  │
│              React / Next.js                │
└─────────────┬───────────────────────────────┘
              │ HTTPS
┌─────────────▼───────────────────────────────┐
│            API Gateway (Kong/nginx)          │
│         Rate Limiting / Auth / SSL           │
└──┬──────────┬──────────────┬────────────────┘
   │          │              │
   ▼          ▼              ▼
┌──────┐  ┌──────┐    ┌──────────┐
│Produ │  │Order │    │  User    │
│ct Svc│  │  Svc │    │  Svc     │
└──┬───┘  └──┬───┘    └────┬─────┘
   │          │              │
   ▼          ▼              ▼
┌──────┐  ┌──────┐    ┌──────────┐
│Produ │  │Order │    │  User    │
│ct DB │  │  DB  │    │   DB     │
└──────┘  └──────┘    └──────────┘
              │
              ▼
         ┌────────┐
         │ Kafka  │ Events
         └──┬─────┘
            │
   ┌────────┼────────────┐
   ▼        ▼            ▼
┌──────┐ ┌──────┐  ┌──────────┐
│Inven │ │Notif │  │Analytics │
│tory  │ │ation │  │  Svc     │
└──────┘ └──────┘  └──────────┘

Technologies:
- Kotlin + Spring Boot 3.x
- PostgreSQL + Redis
- Apache Kafka
- Kubernetes + Helm
- Prometheus + Grafana
- GitHub Actions CI/CD
```

---

## Project Setup

```kotlin
// build.gradle.kts
import org.jetbrains.kotlin.gradle.tasks.KotlinCompile

plugins {
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
    kotlin("jvm") version "1.9.22"
    kotlin("plugin.spring") version "1.9.22"
    kotlin("plugin.jpa") version "1.9.22"
    id("org.jlleitschuh.gradle.ktlint") version "12.0.3"
    id("io.gitlab.arturbosch.detekt") version "1.23.4"
    id("jacoco")
}

group = "com.ecommerce"
version = "1.0.0"

java {
    sourceCompatibility = JavaVersion.VERSION_21
}

repositories {
    mavenCentral()
}

extra["springCloudVersion"] = "2023.0.0"

dependencies {
    // Spring Boot
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("org.springframework.boot:spring-boot-starter-cache")
    
    // Kafka
    implementation("org.springframework.kafka:spring-kafka")
    
    // Kotlin
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-reactor")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    
    // Security
    implementation("io.jsonwebtoken:jjwt-api:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.3")
    
    // Database
    runtimeOnly("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")
    
    // Observability
    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("io.micrometer:micrometer-tracing-bridge-otel")
    implementation("io.opentelemetry:opentelemetry-exporter-otlp")
    
    // Cache
    implementation("com.github.ben-manes.caffeine:caffeine")
    
    // Tests
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.kafka:spring-kafka-test")
    testImplementation("org.springframework.security:spring-security-test")
    testImplementation("org.testcontainers:junit-jupiter")
    testImplementation("org.testcontainers:postgresql")
    testImplementation("org.testcontainers:kafka")
    testImplementation("io.kotest:kotest-runner-junit5:5.8.0")
    testImplementation("io.kotest:kotest-assertions-core:5.8.0")
    testImplementation("io.mockk:mockk:1.13.8")
    testImplementation("com.ninja-squad:springmockk:4.0.2")
}

dependencyManagement {
    imports {
        mavenBom("org.springframework.cloud:spring-cloud-dependencies:${property("springCloudVersion")}")
    }
}

tasks.withType<KotlinCompile> {
    kotlinOptions {
        freeCompilerArgs += "-Xjsr305=strict"
        jvmTarget = "21"
    }
}

tasks.test {
    useJUnitPlatform()
    finalizedBy(tasks.jacocoTestReport)
}

jacoco {
    toolVersion = "0.8.11"
}

tasks.jacocoTestReport {
    reports {
        xml.required = true
        html.required = true
    }
}
```

---

## Domain Model

```kotlin
// Product domain
package com.ecommerce.product.domain

import java.math.BigDecimal
import java.time.Instant
import java.util.UUID

data class ProductId(val value: String = UUID.randomUUID().toString())
data class CategoryId(val value: String)

data class Money(
    val amount: BigDecimal,
    val currency: String = "THB"
) {
    init {
        require(amount >= BigDecimal.ZERO) { "Amount must not be negative" }
    }
    
    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Cannot add different currencies" }
        return Money(amount + other.amount, currency)
    }
    
    fun applyDiscount(percent: Int): Money {
        require(percent in 0..100) { "Discount percent must be 0-100" }
        return Money(amount * (1 - percent / 100.0).toBigDecimal(), currency)
    }
}

sealed class ProductStatus {
    object Active : ProductStatus()
    object Inactive : ProductStatus()
    data class Discontinued(val reason: String) : ProductStatus()
}

data class ProductImage(
    val url: String,
    val altText: String,
    val isPrimary: Boolean = false,
    val sortOrder: Int = 0
)

class Product private constructor(
    val id: ProductId,
    var name: String,
    var description: String,
    var price: Money,
    val categoryId: CategoryId,
    var sku: String,
    var status: ProductStatus = ProductStatus.Active,
    var images: List<ProductImage> = emptyList(),
    var attributes: Map<String, String> = emptyMap(),
    var stockQuantity: Int = 0,
    val createdAt: Instant = Instant.now(),
    var updatedAt: Instant = Instant.now()
) {
    companion object {
        fun create(
            name: String,
            description: String,
            price: Money,
            categoryId: CategoryId,
            sku: String,
            stockQuantity: Int = 0
        ): Product {
            require(name.isNotBlank()) { "Product name must not be blank" }
            require(sku.isNotBlank()) { "SKU must not be blank" }
            
            return Product(
                id = ProductId(),
                name = name,
                description = description,
                price = price,
                categoryId = categoryId,
                sku = sku,
                stockQuantity = stockQuantity
            )
        }
    }
    
    fun updatePrice(newPrice: Money) {
        require(newPrice.currency == price.currency) { "Cannot change currency" }
        price = newPrice
        updatedAt = Instant.now()
    }
    
    fun addImage(image: ProductImage): Product {
        images = images + image
        updatedAt = Instant.now()
        return this
    }
    
    fun adjustStock(delta: Int): Product {
        require(stockQuantity + delta >= 0) {
            "Cannot reduce stock below zero. Current: $stockQuantity, Delta: $delta"
        }
        stockQuantity += delta
        updatedAt = Instant.now()
        return this
    }
    
    fun isAvailable(): Boolean = status == ProductStatus.Active && stockQuantity > 0
    
    fun discontinue(reason: String) {
        status = ProductStatus.Discontinued(reason)
        updatedAt = Instant.now()
    }
}

// Order domain
package com.ecommerce.order.domain

data class OrderId(val value: String = java.util.UUID.randomUUID().toString())
data class UserId(val value: String)

enum class OrderStatus {
    PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED, REFUNDED
}

data class ShippingAddress(
    val recipientName: String,
    val phone: String,
    val address: String,
    val city: String,
    val province: String,
    val postalCode: String,
    val country: String = "TH"
)

data class OrderLineItem(
    val productId: String,
    val sku: String,
    val name: String,        // Snapshot at time of order
    val unitPrice: Money,    // Snapshot at time of order
    val quantity: Int
) {
    init {
        require(quantity > 0) { "Quantity must be positive" }
    }
    
    val subtotal: Money get() = Money(unitPrice.amount * quantity.toBigDecimal(), unitPrice.currency)
}

class Order private constructor(
    val id: OrderId,
    val userId: UserId,
    val items: List<OrderLineItem>,
    var status: OrderStatus,
    val shippingAddress: ShippingAddress,
    val currency: String,
    var notes: String?,
    val placedAt: java.time.Instant,
    var updatedAt: java.time.Instant
) {
    companion object {
        fun place(
            userId: UserId,
            items: List<OrderLineItem>,
            shippingAddress: ShippingAddress,
            currency: String = "THB",
            notes: String? = null
        ): Order {
            require(items.isNotEmpty()) { "Order must have at least one item" }
            
            return Order(
                id = OrderId(),
                userId = userId,
                items = items,
                status = OrderStatus.PENDING,
                shippingAddress = shippingAddress,
                currency = currency,
                notes = notes,
                placedAt = java.time.Instant.now(),
                updatedAt = java.time.Instant.now()
            )
        }
    }
    
    val total: Money
        get() = items.fold(Money(BigDecimal.ZERO, currency)) { acc, item -> acc + item.subtotal }
    
    val itemCount: Int
        get() = items.sumOf { it.quantity }
    
    fun confirm(): Order {
        require(status == OrderStatus.PENDING) { "Can only confirm PENDING orders" }
        status = OrderStatus.CONFIRMED
        updatedAt = java.time.Instant.now()
        return this
    }
    
    fun cancel(reason: String): Order {
        require(status in listOf(OrderStatus.PENDING, OrderStatus.CONFIRMED)) {
            "Cannot cancel order with status: $status"
        }
        status = OrderStatus.CANCELLED
        notes = reason
        updatedAt = java.time.Instant.now()
        return this
    }
    
    fun ship(trackingNumber: String): Order {
        require(status == OrderStatus.PROCESSING) { "Can only ship PROCESSING orders" }
        status = OrderStatus.SHIPPED
        notes = "Tracking: $trackingNumber"
        updatedAt = java.time.Instant.now()
        return this
    }
}
```

---

## API Layer

```kotlin
// Product REST API
package com.ecommerce.product.web

@RestController
@RequestMapping("/api/v1/products")
class ProductController(
    private val productService: ProductApplicationService,
    private val metrics: ProductMetrics
) {
    
    @GetMapping
    fun listProducts(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int,
        @RequestParam(required = false) categoryId: String?,
        @RequestParam(required = false) search: String?,
        @RequestParam(defaultValue = "createdAt,desc") sort: String
    ): ResponseEntity<PagedResponse<ProductSummaryDto>> {
        val result = productService.findProducts(
            ProductFilter(
                categoryId = categoryId,
                search = search,
                page = page,
                size = size.coerceAtMost(100),
                sort = sort
            )
        )
        
        metrics.recordProductListView()
        
        return ResponseEntity.ok(
            PagedResponse(
                data = result.content.map { it.toSummaryDto() },
                pagination = PaginationMeta(
                    page = result.number,
                    size = result.size,
                    total = result.totalElements,
                    totalPages = result.totalPages
                )
            )
        )
    }
    
    @GetMapping("/{id}")
    fun getProduct(@PathVariable id: String): ResponseEntity<ProductDetailDto> {
        val product = productService.findById(id)
            ?: return ResponseEntity.notFound().build()
        
        metrics.recordProductView(id)
        
        return ResponseEntity.ok(product.toDetailDto())
    }
    
    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    fun createProduct(@Valid @RequestBody request: CreateProductRequest): ResponseEntity<ProductDetailDto> {
        val product = productService.create(request)
        return ResponseEntity
            .status(201)
            .header("Location", "/api/v1/products/${product.id}")
            .body(product.toDetailDto())
    }
    
    @PutMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    fun updateProduct(
        @PathVariable id: String,
        @Valid @RequestBody request: UpdateProductRequest
    ): ResponseEntity<ProductDetailDto> {
        val product = productService.update(id, request)
            ?: return ResponseEntity.notFound().build()
        
        return ResponseEntity.ok(product.toDetailDto())
    }
    
    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    fun discontinueProduct(
        @PathVariable id: String,
        @RequestParam reason: String
    ): ResponseEntity<Void> {
        productService.discontinue(id, reason)
        return ResponseEntity.noContent().build()
    }
    
    @GetMapping("/search")
    fun searchProducts(
        @RequestParam q: String,
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int
    ): ResponseEntity<PagedResponse<ProductSummaryDto>> {
        val results = productService.search(q, page, size.coerceAtMost(50))
        return ResponseEntity.ok(results)
    }
}

// DTOs
data class CreateProductRequest(
    @field:NotBlank val name: String,
    val description: String = "",
    @field:DecimalMin("0.01") val price: java.math.BigDecimal,
    val currency: String = "THB",
    @field:NotBlank val categoryId: String,
    @field:NotBlank @field:Size(min = 3, max = 50) val sku: String,
    @field:Min(0) val stockQuantity: Int = 0
)

data class UpdateProductRequest(
    @field:NotBlank val name: String?,
    val description: String?,
    @field:DecimalMin("0.01") val price: java.math.BigDecimal?,
    @field:Min(0) val stockQuantity: Int?
)

data class ProductSummaryDto(
    val id: String,
    val name: String,
    val price: java.math.BigDecimal,
    val currency: String,
    val primaryImageUrl: String?,
    val inStock: Boolean,
    val categoryId: String
)

data class ProductDetailDto(
    val id: String,
    val name: String,
    val description: String,
    val price: java.math.BigDecimal,
    val currency: String,
    val sku: String,
    val categoryId: String,
    val images: List<ProductImageDto>,
    val attributes: Map<String, String>,
    val stockQuantity: Int,
    val status: String,
    val createdAt: java.time.Instant,
    val updatedAt: java.time.Instant
)

data class ProductImageDto(val url: String, val altText: String, val isPrimary: Boolean)

data class ProductFilter(
    val categoryId: String?,
    val search: String?,
    val page: Int,
    val size: Int,
    val sort: String
)

data class PagedResponse<T>(
    val data: List<T>,
    val pagination: PaginationMeta
)

data class PaginationMeta(
    val page: Int,
    val size: Int,
    val total: Long,
    val totalPages: Int
)

// Extension functions for DTO conversion
fun Product.toSummaryDto() = ProductSummaryDto(
    id = id.value,
    name = name,
    price = price.amount,
    currency = price.currency,
    primaryImageUrl = images.find { it.isPrimary }?.url,
    inStock = isAvailable(),
    categoryId = categoryId.value
)

fun Product.toDetailDto() = ProductDetailDto(
    id = id.value,
    name = name,
    description = description,
    price = price.amount,
    currency = price.currency,
    sku = sku,
    categoryId = categoryId.value,
    images = images.map { ProductImageDto(it.url, it.altText, it.isPrimary) },
    attributes = attributes,
    stockQuantity = stockQuantity,
    status = when (status) {
        ProductStatus.Active -> "ACTIVE"
        ProductStatus.Inactive -> "INACTIVE"
        is ProductStatus.Discontinued -> "DISCONTINUED"
    },
    createdAt = createdAt,
    updatedAt = updatedAt
)

typealias RestController = org.springframework.web.bind.annotation.RestController
typealias RequestMapping = org.springframework.web.bind.annotation.RequestMapping
typealias GetMapping = org.springframework.web.bind.annotation.GetMapping
typealias PostMapping = org.springframework.web.bind.annotation.PostMapping
typealias PutMapping = org.springframework.web.bind.annotation.PutMapping
typealias DeleteMapping = org.springframework.web.bind.annotation.DeleteMapping
typealias PathVariable = org.springframework.web.bind.annotation.PathVariable
typealias RequestParam = org.springframework.web.bind.annotation.RequestParam
typealias RequestBody = org.springframework.web.bind.annotation.RequestBody
typealias Valid = javax.validation.Valid
typealias PreAuthorize = org.springframework.security.access.prepost.PreAuthorize
typealias ResponseEntity = org.springframework.http.ResponseEntity<*>
typealias NotBlank = javax.validation.constraints.NotBlank
typealias DecimalMin = javax.validation.constraints.DecimalMin
typealias Min = javax.validation.constraints.Min
typealias Size = javax.validation.constraints.Size
```

---

## Service Layer

```kotlin
// Product Application Service
package com.ecommerce.product.application

@Service
@Transactional
class ProductApplicationService(
    private val productRepository: ProductRepository,
    private val categoryRepository: CategoryRepository,
    private val eventPublisher: DomainEventPublisher,
    private val cacheManager: CacheManager
) {
    
    fun findById(id: String): Product? {
        return productRepository.findById(ProductId(id))
    }
    
    @Transactional(readOnly = true)
    fun findProducts(filter: ProductFilter): org.springframework.data.domain.Page<Product> {
        return productRepository.findWithFilter(filter)
    }
    
    fun create(request: CreateProductRequest): Product {
        // Validate category exists
        val category = categoryRepository.findById(CategoryId(request.categoryId))
            ?: throw CategoryNotFoundException("Category ${request.categoryId} not found")
        
        // Check SKU uniqueness
        if (productRepository.existsBySku(request.sku)) {
            throw DuplicateSkuException("SKU ${request.sku} already exists")
        }
        
        val product = Product.create(
            name = request.name,
            description = request.description,
            price = Money(request.price, request.currency),
            categoryId = CategoryId(request.categoryId),
            sku = request.sku,
            stockQuantity = request.stockQuantity
        )
        
        val saved = productRepository.save(product)
        
        // Publish domain event
        eventPublisher.publish(
            ProductCreatedEvent(
                productId = saved.id.value,
                name = saved.name,
                price = saved.price.amount,
                currency = saved.price.currency,
                categoryId = saved.categoryId.value
            )
        )
        
        return saved
    }
    
    fun update(id: String, request: UpdateProductRequest): Product? {
        val product = productRepository.findById(ProductId(id)) ?: return null
        
        request.name?.let { product.name = it }
        request.description?.let { product.description = it }
        request.price?.let { product.updatePrice(Money(it, product.price.currency)) }
        request.stockQuantity?.let {
            val delta = it - product.stockQuantity
            product.adjustStock(delta)
        }
        
        val saved = productRepository.save(product)
        
        // Evict cache
        cacheManager.getCache("products")?.evict(id)
        
        return saved
    }
    
    fun discontinue(id: String, reason: String) {
        val product = productRepository.findById(ProductId(id))
            ?: throw ProductNotFoundException("Product $id not found")
        
        product.discontinue(reason)
        productRepository.save(product)
        
        eventPublisher.publish(ProductDiscontinuedEvent(id, reason))
    }
    
    @Transactional(readOnly = true)
    fun search(query: String, page: Int, size: Int): PagedResponse<ProductSummaryDto> {
        val results = productRepository.search(query, page, size)
        return PagedResponse(
            data = results.content.map { it.toSummaryDto() },
            pagination = PaginationMeta(
                page = results.number,
                size = results.size,
                total = results.totalElements,
                totalPages = results.totalPages
            )
        )
    }
}

interface ProductRepository {
    fun findById(id: ProductId): Product?
    fun findWithFilter(filter: ProductFilter): org.springframework.data.domain.Page<Product>
    fun save(product: Product): Product
    fun existsBySku(sku: String): Boolean
    fun search(query: String, page: Int, size: Int): org.springframework.data.domain.Page<Product>
}

interface CategoryRepository {
    fun findById(id: CategoryId): Any?
}

interface DomainEventPublisher {
    fun publish(event: Any)
}

data class ProductCreatedEvent(
    val productId: String,
    val name: String,
    val price: java.math.BigDecimal,
    val currency: String,
    val categoryId: String
)

data class ProductDiscontinuedEvent(val productId: String, val reason: String)

class CategoryNotFoundException(msg: String) : RuntimeException(msg)
class DuplicateSkuException(msg: String) : RuntimeException(msg)
class ProductNotFoundException(msg: String) : RuntimeException(msg)

typealias Service = org.springframework.stereotype.Service
typealias Transactional = org.springframework.transaction.annotation.Transactional
typealias CacheManager = org.springframework.cache.CacheManager

class ProductMetrics {
    fun recordProductView(productId: String) {}
    fun recordProductListView() {}
}
```

---

## Infrastructure

```yaml
# docker-compose.yml for local development

version: '3.8'

services:
  product-service:
    build:
      context: ./product-service
      dockerfile: Dockerfile
    ports:
      - "8081:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/product_db
      SPRING_DATASOURCE_USERNAME: product_user
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD}
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_DATA_REDIS_HOST: redis
      JAVA_OPTS: >-
        -XX:+UseContainerSupport
        -XX:MaxRAMPercentage=75.0
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: product_db
      POSTGRES_USER: product_user
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U product_user -d product_db"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    depends_on:
      - zookeeper
    healthcheck:
      test: ["CMD", "kafka-topics", "--bootstrap-server", "localhost:9092", "--list"]
      interval: 30s
      timeout: 10s
      retries: 5

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

volumes:
  postgres_data:
  redis_data:
```

---

## สรุป Part 80

```
✅ Microservices architecture: product, order, user, inventory
✅ build.gradle.kts: complete dependency management
✅ Domain model: Product, Order, Money value objects
✅ ProductId/OrderId: value class wrappers for type safety
✅ Product.create(): factory with validation
✅ Product.adjustStock(): negative stock prevention
✅ Order.place(): factory with required items validation
✅ Order.total: computed from line items
✅ ProductController: CRUD + search with pagination
✅ PagedResponse<T>: standardized pagination response
✅ ProductSummaryDto vs ProductDetailDto: two views
✅ toSummaryDto()/toDetailDto(): extension functions
✅ ProductApplicationService: orchestrates domain + infra
✅ @Transactional(readOnly = true): optimization for reads
✅ Domain events: ProductCreatedEvent, ProductDiscontinuedEvent
✅ ProductRepository interface: port for infrastructure
✅ Cache eviction: after product update
✅ docker-compose.yml: complete local development stack
✅ Health checks: postgres, redis, kafka, app
✅ JAVA_OPTS: container-aware JVM settings
```

---

*Part 80/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
