# Part 87: Enterprise Architecture Patterns

## สารบัญ
1. [Hexagonal Architecture](#hexagonal-architecture)
2. [Strangler Fig Pattern](#strangler-fig-pattern)
3. [Anti-Corruption Layer](#anti-corruption-layer)
4. [Saga Pattern](#saga-pattern)
5. [Outbox Pattern](#outbox-pattern)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Hexagonal Architecture

```
Hexagonal Architecture (Ports & Adapters):

                 ┌──────────────────────────────────────┐
                 │           APPLICATION CORE            │
   REST API ───► │  ┌───────────────────────────────┐   │ ◄─── DB Adapter
   GraphQL ───► │  │        Domain Layer            │   │ ◄─── Cache Adapter
   gRPC ─────► │  │  (Entities, Value Objects,     │   │ ◄─── Message Adapter
   CLI ──────► │  │   Aggregates, Domain Services) │   │
                │  └───────────────────────────────┘   │
                │                                        │
                │  Use Cases (Application Services)      │
                │  Orchestrate domain + call ports        │
                └──────────────────────────────────────┘

Ports (Interfaces):
- Inbound ports: PlaceOrderUseCase, GetProductQuery
- Outbound ports: OrderRepository, EventPublisher, PaymentGateway

Adapters (Implementations):
- Inbound: RestController, GraphQLResolver, KafkaConsumer
- Outbound: JpaOrderRepository, KafkaEventPublisher, StripePaymentGateway

Benefits:
✅ Domain ไม่ depend on infrastructure
✅ Test business logic ได้โดยไม่ต้องมี DB
✅ เปลี่ยน DB, message broker ได้โดยไม่กระทบ domain
✅ Multiple entry points: REST, gRPC, CLI, batch
```

---

## Project Structure

```kotlin
// Hexagonal Architecture structure
/*
com.ecommerce/
├── domain/                     # Core domain (no Spring, no DB)
│   ├── model/
│   │   ├── Product.kt          # Aggregate root
│   │   ├── Order.kt            # Aggregate root
│   │   ├── Money.kt            # Value object
│   │   └── ProductId.kt        # Value object
│   ├── event/
│   │   ├── OrderPlaced.kt      # Domain event
│   │   └── ProductCreated.kt
│   ├── exception/
│   │   ├── InsufficientStockException.kt
│   │   └── ProductNotFoundException.kt
│   └── service/
│       └── PricingService.kt   # Domain service
│
├── application/                # Use cases / Application services
│   ├── port/
│   │   ├── input/              # Inbound ports
│   │   │   ├── PlaceOrderUseCase.kt
│   │   │   ├── GetProductQuery.kt
│   │   │   └── CreateProductCommand.kt
│   │   └── output/             # Outbound ports
│   │       ├── OrderRepository.kt
│   │       ├── ProductRepository.kt
│   │       ├── EventPublisher.kt
│   │       └── PaymentGateway.kt
│   └── service/
│       ├── OrderService.kt     # Implements PlaceOrderUseCase
│       └── ProductService.kt   # Implements CRUD use cases
│
└── adapter/
    ├── inbound/
    │   ├── web/
    │   │   ├── ProductController.kt
    │   │   └── OrderController.kt
    │   └── messaging/
    │       └── OrderEventConsumer.kt
    └── outbound/
        ├── persistence/
        │   ├── JpaOrderRepository.kt
        │   └── JpaProductRepository.kt
        ├── messaging/
        │   └── KafkaEventPublisher.kt
        └── payment/
            └── StripePaymentGateway.kt
*/

// Domain layer (pure Kotlin, no framework dependencies)
data class ProductId(val value: String) {
    companion object {
        fun generate() = ProductId(java.util.UUID.randomUUID().toString())
        fun of(value: String) = ProductId(value)
    }
}

data class Money(
    val amount: java.math.BigDecimal,
    val currency: String = "THB"
) {
    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Cannot add different currencies" }
        return copy(amount = amount + other.amount)
    }
    
    operator fun times(multiplier: Int): Money = copy(amount = amount * multiplier.toBigDecimal())
    
    fun applyDiscount(percent: Int): Money {
        require(percent in 0..100) { "Discount must be 0-100%" }
        return copy(amount = amount * (100 - percent).toBigDecimal() / 100.toBigDecimal())
    }
    
    companion object {
        fun of(amount: Number, currency: String = "THB") = Money(amount.toString().toBigDecimal(), currency)
        val ZERO = Money(java.math.BigDecimal.ZERO)
    }
}

class Product private constructor(
    val id: ProductId,
    var name: String,
    var price: Money,
    var stockQuantity: Int,
    private val _events: MutableList<DomainEvent> = mutableListOf()
) {
    val events: List<DomainEvent> get() = _events.toList()
    
    fun adjustStock(delta: Int) {
        require(stockQuantity + delta >= 0) { "Insufficient stock" }
        stockQuantity += delta
        _events.add(StockAdjusted(id.value, delta, stockQuantity))
    }
    
    fun clearEvents() = _events.clear()
    
    companion object {
        fun create(name: String, price: Money, initialStock: Int): Product {
            require(name.isNotBlank()) { "Product name cannot be blank" }
            require(price.amount > java.math.BigDecimal.ZERO) { "Price must be positive" }
            require(initialStock >= 0) { "Stock cannot be negative" }
            
            val product = Product(ProductId.generate(), name, price, initialStock)
            product._events.add(ProductCreated(product.id.value, name, price.amount))
            return product
        }
    }
}

// Domain events
sealed class DomainEvent {
    abstract val aggregateId: String
    abstract val occurredAt: java.time.Instant
}

data class ProductCreated(
    override val aggregateId: String,
    val name: String,
    val price: java.math.BigDecimal,
    override val occurredAt: java.time.Instant = java.time.Instant.now()
) : DomainEvent()

data class StockAdjusted(
    override val aggregateId: String,
    val delta: Int,
    val newQuantity: Int,
    override val occurredAt: java.time.Instant = java.time.Instant.now()
) : DomainEvent()

data class OrderPlaced(
    override val aggregateId: String,
    val customerId: String,
    val totalAmount: java.math.BigDecimal,
    override val occurredAt: java.time.Instant = java.time.Instant.now()
) : DomainEvent()

// Inbound ports (use cases)
interface PlaceOrderUseCase {
    suspend fun placeOrder(command: PlaceOrderCommand): OrderId
}

data class PlaceOrderCommand(
    val customerId: String,
    val items: List<OrderItemCommand>
)

data class OrderItemCommand(val productId: String, val quantity: Int)

data class OrderId(val value: String)

interface GetProductQuery {
    suspend fun getById(id: ProductId): Product?
    suspend fun search(query: String, page: Int, size: Int): PagedResult<Product>
}

// Outbound ports
interface ProductRepository {
    suspend fun findById(id: ProductId): Product?
    suspend fun save(product: Product)
    suspend fun delete(id: ProductId)
    suspend fun findAll(page: Int, size: Int): PagedResult<Product>
}

interface EventPublisher {
    suspend fun publish(event: DomainEvent)
    suspend fun publishAll(events: List<DomainEvent>)
}

interface PaymentGateway {
    suspend fun charge(customerId: String, amount: Money): PaymentResult
    suspend fun refund(paymentId: String): RefundResult
}

data class PaymentResult(val paymentId: String, val status: PaymentStatus)
enum class PaymentStatus { SUCCESS, FAILED, PENDING }
data class RefundResult(val refundId: String, val status: String)

// Application service (implements use case)
@org.springframework.stereotype.Service
class OrderApplicationService(
    private val productRepository: ProductRepository,
    private val orderRepository: OrderRepository,
    private val paymentGateway: PaymentGateway,
    private val eventPublisher: EventPublisher
) : PlaceOrderUseCase {
    
    @org.springframework.transaction.annotation.Transactional
    override suspend fun placeOrder(command: PlaceOrderCommand): OrderId {
        // Load products
        val products = command.items.map { item ->
            val product = productRepository.findById(ProductId.of(item.productId))
                ?: throw ProductNotFoundException("Product ${item.productId} not found")
            product to item.quantity
        }
        
        // Reserve stock
        products.forEach { (product, qty) -> product.adjustStock(-qty) }
        
        // Calculate total
        val total = products.fold(Money.ZERO) { acc, (product, qty) ->
            acc + (product.price * qty)
        }
        
        // Process payment
        val payment = paymentGateway.charge(command.customerId, total)
        if (payment.status != PaymentStatus.SUCCESS) {
            // Rollback stock
            products.forEach { (product, qty) -> product.adjustStock(qty) }
            throw PaymentFailedException("Payment failed")
        }
        
        // Create order
        val order = Order.place(command.customerId, products.map { (product, qty) ->
            OrderLineItem(product.id, product.name, product.price, qty)
        })
        
        // Save
        products.forEach { (product, _) -> productRepository.save(product) }
        orderRepository.save(order)
        
        // Publish events
        eventPublisher.publishAll(
            products.flatMap { (product, _) -> product.events } + order.events
        )
        
        // Clear events (already published)
        products.forEach { (product, _) -> product.clearEvents() }
        order.clearEvents()
        
        return OrderId(order.id.value)
    }
}

// Outbound adapter: JPA implementation
@org.springframework.stereotype.Repository
class JpaProductRepositoryAdapter(
    private val jpaRepo: ProductJpaRepository
) : ProductRepository {
    
    override suspend fun findById(id: ProductId): Product? {
        return jpaRepo.findById(id.value).orElse(null)?.toDomain()
    }
    
    override suspend fun save(product: Product) {
        jpaRepo.save(product.toEntity())
    }
    
    override suspend fun delete(id: ProductId) {
        jpaRepo.deleteById(id.value)
    }
    
    override suspend fun findAll(page: Int, size: Int): PagedResult<Product> {
        val pageable = org.springframework.data.domain.PageRequest.of(page, size)
        val result = jpaRepo.findAll(pageable)
        return PagedResult(
            items = result.content.map { it.toDomain() },
            total = result.totalElements,
            page = page,
            size = size,
            hasMore = result.hasNext()
        )
    }
}

class ProductNotFoundException(message: String) : RuntimeException(message)
class PaymentFailedException(message: String) : RuntimeException(message)

interface OrderRepository {
    suspend fun findById(id: OrderId): Order?
    suspend fun save(order: Order)
}

data class Order(val id: OrderId, val customerId: String, val items: List<OrderLineItem>, private val _events: MutableList<DomainEvent> = mutableListOf()) {
    val events: List<DomainEvent> get() = _events.toList()
    val totalAmount get() = items.fold(Money.ZERO) { acc, item -> acc + (item.price * item.quantity) }
    fun clearEvents() = _events.clear()
    companion object {
        fun place(customerId: String, items: List<OrderLineItem>): Order {
            val order = Order(OrderId(java.util.UUID.randomUUID().toString()), customerId, items)
            order._events.add(OrderPlaced(order.id.value, customerId, order.totalAmount.amount))
            return order
        }
    }
}
data class OrderLineItem(val productId: ProductId, val productName: String, val price: Money, val quantity: Int)

data class PagedResult<T>(val items: List<T>, val total: Long, val page: Int, val size: Int, val hasMore: Boolean)

fun Any.toDomain(): Product = TODO()
fun Product.toEntity(): Any = TODO()
interface ProductJpaRepository : org.springframework.data.jpa.repository.JpaRepository<Any, String>
```

---

## Strangler Fig Pattern

```kotlin
// Gradually migrate from Legacy Monolith to Microservices

// Step 1: Facade routes traffic to old or new
@org.springframework.stereotype.Service
class ProductFacade(
    private val legacyClient: LegacyProductClient,
    private val newProductService: ProductService,
    private val featureFlags: FeatureFlagService
) {
    
    suspend fun getProduct(id: String): ProductResponse {
        return if (featureFlags.isEnabled("NEW_PRODUCT_SERVICE")) {
            newProductService.getById(id)?.toResponse()
                ?: throw NotFoundException("Product not found: $id")
        } else {
            legacyClient.getProduct(id)
        }
    }
    
    suspend fun createProduct(request: CreateProductRequest): ProductResponse {
        return if (featureFlags.isEnabled("NEW_PRODUCT_WRITE")) {
            val product = newProductService.create(request)
            
            // Write to legacy too until fully migrated
            if (featureFlags.isEnabled("DUAL_WRITE")) {
                try {
                    legacyClient.createProduct(request)
                } catch (e: Exception) {
                    // Log but don't fail - new system is authoritative
                }
            }
            
            product.toResponse()
        } else {
            val legacyResponse = legacyClient.createProduct(request)
            
            // Async sync to new system
            kotlinx.coroutines.GlobalScope.launch {
                newProductService.syncFromLegacy(legacyResponse.id)
            }
            
            legacyResponse
        }
    }
}

// Step 2: Data migration runner
@org.springframework.stereotype.Component
class MigrationRunner(
    private val legacyDatabase: LegacyDatabase,
    private val newProductService: ProductService,
    private val migrationStateRepository: MigrationStateRepository
) {
    
    suspend fun migrateProducts(batchSize: Int = 100) {
        var offset = migrationStateRepository.getLastMigratedOffset("products")
        
        while (true) {
            val legacyProducts = legacyDatabase.getProducts(offset, batchSize)
            if (legacyProducts.isEmpty()) break
            
            legacyProducts.forEach { legacy ->
                try {
                    newProductService.upsert(legacy.toNewFormat())
                    migrationStateRepository.markMigrated(legacy.id, "products")
                } catch (e: Exception) {
                    migrationStateRepository.markFailed(legacy.id, "products", e.message ?: "")
                }
            }
            
            offset += batchSize
            migrationStateRepository.updateOffset("products", offset)
        }
    }
}

interface LegacyProductClient {
    suspend fun getProduct(id: String): ProductResponse
    suspend fun createProduct(request: CreateProductRequest): ProductResponse
}

interface LegacyDatabase {
    fun getProducts(offset: Int, limit: Int): List<LegacyProduct>
}

data class LegacyProduct(val id: String, val productName: String, val unitPrice: Double) {
    fun toNewFormat(): CreateProductRequest = CreateProductRequest(name = productName, price = unitPrice, stockQuantity = 0, categoryId = "")
}

interface MigrationStateRepository {
    fun getLastMigratedOffset(entity: String): Int
    fun markMigrated(id: String, entity: String)
    fun markFailed(id: String, entity: String, reason: String)
    fun updateOffset(entity: String, offset: Int)
}
```

---

## Outbox Pattern

```kotlin
// Transactional Outbox: guarantee event publishing with DB transactions

@jakarta.persistence.Entity
@jakarta.persistence.Table(name = "outbox_events")
data class OutboxEvent(
    @jakarta.persistence.Id
    val id: String = java.util.UUID.randomUUID().toString(),
    
    val aggregateType: String,
    val aggregateId: String,
    val eventType: String,
    
    @jakarta.persistence.Column(columnDefinition = "TEXT")
    val payload: String,
    
    val occurredAt: java.time.Instant = java.time.Instant.now(),
    
    @jakarta.persistence.Enumerated(jakarta.persistence.EnumType.STRING)
    var status: OutboxStatus = OutboxStatus.PENDING,
    
    var processedAt: java.time.Instant? = null,
    var failureReason: String? = null,
    var retryCount: Int = 0
)

enum class OutboxStatus { PENDING, PROCESSING, PROCESSED, FAILED }

@org.springframework.stereotype.Repository
interface OutboxEventRepository : org.springframework.data.jpa.repository.JpaRepository<OutboxEvent, String> {
    
    @org.springframework.data.jpa.repository.Query(
        "SELECT e FROM OutboxEvent e WHERE e.status = 'PENDING' ORDER BY e.occurredAt ASC",
        lockMode = jakarta.persistence.LockModeType.PESSIMISTIC_WRITE
    )
    fun findPendingForProcessing(pageable: org.springframework.data.domain.Pageable): List<OutboxEvent>
    
    @org.springframework.data.jpa.repository.Query(
        "SELECT e FROM OutboxEvent e WHERE e.status = 'FAILED' AND e.retryCount < :maxRetries ORDER BY e.occurredAt ASC"
    )
    fun findFailedForRetry(@org.springframework.data.repository.query.Param("maxRetries") maxRetries: Int): List<OutboxEvent>
}

// Service that writes events to outbox in same transaction as business logic
@org.springframework.stereotype.Service
class OutboxEventStore(
    private val outboxRepository: OutboxEventRepository,
    private val objectMapper: com.fasterxml.jackson.databind.ObjectMapper
) {
    
    fun store(event: DomainEvent) {
        val outboxEvent = OutboxEvent(
            aggregateType = event.javaClass.simpleName.replace(Regex("(Created|Updated|Deleted).*"), ""),
            aggregateId = event.aggregateId,
            eventType = event.javaClass.simpleName,
            payload = objectMapper.writeValueAsString(event)
        )
        outboxRepository.save(outboxEvent)
    }
    
    fun storeAll(events: List<DomainEvent>) = events.forEach { store(it) }
}

// Outbox relay: polls outbox and publishes to Kafka
@org.springframework.stereotype.Component
class OutboxRelay(
    private val outboxRepository: OutboxEventRepository,
    private val kafkaTemplate: org.springframework.kafka.core.KafkaTemplate<String, String>,
    private val meterRegistry: io.micrometer.core.instrument.MeterRegistry
) {
    private val logger = org.slf4j.LoggerFactory.getLogger(javaClass)
    
    @org.springframework.scheduling.annotation.Scheduled(fixedDelay = 1000)
    @org.springframework.transaction.annotation.Transactional
    fun relay() {
        val events = outboxRepository.findPendingForProcessing(
            org.springframework.data.domain.PageRequest.of(0, 100)
        )
        
        events.forEach { event ->
            event.status = OutboxStatus.PROCESSING
            outboxRepository.save(event)
        }
        
        events.forEach { event ->
            try {
                val topic = "domain.${event.aggregateType.lowercase()}.${event.eventType.lowercase()}"
                kafkaTemplate.send(topic, event.aggregateId, event.payload).get(10, java.util.concurrent.TimeUnit.SECONDS)
                
                event.status = OutboxStatus.PROCESSED
                event.processedAt = java.time.Instant.now()
                
                meterRegistry.counter("outbox.events.processed", "type", event.eventType).increment()
            } catch (e: Exception) {
                logger.error("Failed to publish event ${event.id}", e)
                event.status = OutboxStatus.FAILED
                event.failureReason = e.message
                event.retryCount++
                
                meterRegistry.counter("outbox.events.failed", "type", event.eventType).increment()
            }
            
            outboxRepository.save(event)
        }
    }
    
    @org.springframework.scheduling.annotation.Scheduled(fixedDelay = 60_000)
    @org.springframework.transaction.annotation.Transactional
    fun retryFailed() {
        val failedEvents = outboxRepository.findFailedForRetry(maxRetries = 3)
        failedEvents.forEach { event ->
            event.status = OutboxStatus.PENDING
            outboxRepository.save(event)
        }
    }
}
```

---

## Saga Pattern

```kotlin
// Orchestration Saga for Order Processing

sealed class SagaState {
    object Started : SagaState()
    data class StockReserved(val reservationId: String) : SagaState()
    data class PaymentProcessed(val paymentId: String) : SagaState()
    data class OrderConfirmed(val orderId: String) : SagaState()
    data class Failed(val reason: String, val step: String) : SagaState()
    data class Compensated(val completedAt: java.time.Instant) : SagaState()
}

@jakarta.persistence.Entity
@jakarta.persistence.Table(name = "order_sagas")
data class OrderSaga(
    @jakarta.persistence.Id val sagaId: String,
    val customerId: String,
    
    @jakarta.persistence.Convert(converter = SagaStateConverter::class)
    var state: SagaState = SagaState.Started,
    
    val createdAt: java.time.Instant = java.time.Instant.now(),
    var updatedAt: java.time.Instant = java.time.Instant.now()
)

@org.springframework.stereotype.Service
class OrderSagaOrchestrator(
    private val sagaRepository: OrderSagaRepository,
    private val inventoryService: InventoryService,
    private val paymentService: PaymentService,
    private val orderService: OrderApplicationService,
    private val outboxEventStore: OutboxEventStore
) {
    
    private val logger = org.slf4j.LoggerFactory.getLogger(javaClass)
    
    @org.springframework.transaction.annotation.Transactional
    suspend fun startSaga(command: PlaceOrderCommand): String {
        val sagaId = java.util.UUID.randomUUID().toString()
        val saga = OrderSaga(sagaId = sagaId, customerId = command.customerId)
        sagaRepository.save(saga)
        
        // Step 1: Reserve inventory
        try {
            val reservationId = inventoryService.reserveStock(command.items)
            saga.state = SagaState.StockReserved(reservationId)
            sagaRepository.save(saga)
            
            // Step 2: Process payment
            val paymentResult = paymentService.charge(command.customerId, calculateTotal(command))
            if (paymentResult.status != PaymentStatus.SUCCESS) {
                compensateStockReservation(saga, reservationId)
                saga.state = SagaState.Failed("Payment failed", "PAYMENT")
                sagaRepository.save(saga)
                throw PaymentFailedException("Payment rejected")
            }
            
            saga.state = SagaState.PaymentProcessed(paymentResult.paymentId)
            sagaRepository.save(saga)
            
            // Step 3: Create order
            val orderId = orderService.placeOrder(command)
            saga.state = SagaState.OrderConfirmed(orderId.value)
            saga.updatedAt = java.time.Instant.now()
            sagaRepository.save(saga)
            
            return orderId.value
            
        } catch (e: InsufficientStockException) {
            saga.state = SagaState.Failed(e.message ?: "Insufficient stock", "INVENTORY")
            sagaRepository.save(saga)
            throw e
        } catch (e: Exception) {
            logger.error("Saga $sagaId failed", e)
            compensateSaga(saga)
            throw e
        }
    }
    
    private suspend fun compensateSaga(saga: OrderSaga) {
        when (val state = saga.state) {
            is SagaState.PaymentProcessed -> {
                // Refund payment
                paymentService.refund(state.paymentId)
                // Release stock reservation
                val prevState = saga.state as? SagaState.StockReserved
                prevState?.let { inventoryService.releaseReservation(it.reservationId) }
            }
            is SagaState.StockReserved -> {
                compensateStockReservation(saga, state.reservationId)
            }
            else -> {}
        }
        
        saga.state = SagaState.Compensated(java.time.Instant.now())
        sagaRepository.save(saga)
    }
    
    private suspend fun compensateStockReservation(saga: OrderSaga, reservationId: String) {
        try {
            inventoryService.releaseReservation(reservationId)
        } catch (e: Exception) {
            logger.error("Failed to release reservation $reservationId for saga ${saga.sagaId}", e)
        }
    }
    
    private fun calculateTotal(command: PlaceOrderCommand): Money = Money.ZERO  // simplified
}

interface InventoryService {
    suspend fun reserveStock(items: List<OrderItemCommand>): String
    suspend fun releaseReservation(reservationId: String)
    suspend fun confirmReservation(reservationId: String)
}

interface PaymentService {
    suspend fun charge(customerId: String, amount: Money): PaymentResult
    suspend fun refund(paymentId: String): RefundResult
}

interface OrderSagaRepository : org.springframework.data.jpa.repository.JpaRepository<OrderSaga, String>

class InsufficientStockException(message: String) : RuntimeException(message)
class NotFoundException(message: String) : RuntimeException(message)
class SagaStateConverter : jakarta.persistence.AttributeConverter<SagaState, String> {
    private val mapper = com.fasterxml.jackson.module.kotlin.jacksonObjectMapper()
    override fun convertToDatabaseColumn(state: SagaState) = mapper.writeValueAsString(state)
    override fun convertToEntityAttribute(data: String): SagaState = mapper.readValue(data, SagaState::class.java)
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement Anti-Corruption Layer สำหรับ Legacy CRM Integration

// Legacy CRM มี API แบบ XML และ data model แตกต่างจาก domain ของเรา
// ต้องสร้าง ACL ที่แปลง legacy format → domain model

data class LegacyCustomer(
    val CUST_ID: String,           // legacy naming
    val FULL_NM: String,
    val ADDR_LINE_1: String,
    val ADDR_LINE_2: String?,
    val POSTCODE: String,
    val PROVINCE_CD: String,
    val EMAIL_ADDR: String?,
    val TEL_NO: String?,
    val ACTIVE_FL: String          // "Y" or "N"
)

// Domain model
data class Customer(
    val id: CustomerId,
    val name: CustomerName,
    val address: Address,
    val contact: ContactInfo,
    val isActive: Boolean
)

data class CustomerId(val value: String)
data class CustomerName(val fullName: String) {
    val firstName get() = fullName.substringBefore(" ")
    val lastName get() = fullName.substringAfter(" ", "")
}

data class Address(
    val line1: String,
    val line2: String?,
    val postalCode: String,
    val province: Province
)

enum class Province(val code: String) {
    BANGKOK("BKK"), CHIANGMAI("CMI"), PHUKET("PKT");
    companion object {
        fun fromCode(code: String) = values().find { it.code == code }
            ?: throw IllegalArgumentException("Unknown province code: $code")
    }
}

data class ContactInfo(val email: String?, val phone: String?)

// Anti-Corruption Layer
class LegacyCrmAcl {
    
    fun translate(legacy: LegacyCustomer): Customer {
        return Customer(
            id = CustomerId(legacy.CUST_ID),
            name = CustomerName(legacy.FULL_NM),
            address = Address(
                line1 = legacy.ADDR_LINE_1,
                line2 = legacy.ADDR_LINE_2,
                postalCode = legacy.POSTCODE,
                province = Province.fromCode(legacy.PROVINCE_CD)
            ),
            contact = ContactInfo(
                email = legacy.EMAIL_ADDR?.takeIf { it.contains("@") },
                phone = normalizePhone(legacy.TEL_NO)
            ),
            isActive = legacy.ACTIVE_FL == "Y"
        )
    }
    
    fun translateBack(customer: Customer): LegacyCustomer {
        return LegacyCustomer(
            CUST_ID = customer.id.value,
            FULL_NM = customer.name.fullName,
            ADDR_LINE_1 = customer.address.line1,
            ADDR_LINE_2 = customer.address.line2,
            POSTCODE = customer.address.postalCode,
            PROVINCE_CD = customer.address.province.code,
            EMAIL_ADDR = customer.contact.email,
            TEL_NO = customer.contact.phone,
            ACTIVE_FL = if (customer.isActive) "Y" else "N"
        )
    }
    
    private fun normalizePhone(phone: String?): String? {
        return phone?.replace(Regex("[^0-9+]"), "")
            ?.let { if (it.length >= 9) it else null }
    }
}
```

---

## สรุป Part 87

```
✅ Hexagonal Architecture: domain ไม่ depend on infrastructure
✅ Inbound ports: PlaceOrderUseCase, GetProductQuery interfaces
✅ Outbound ports: ProductRepository, EventPublisher, PaymentGateway
✅ Adapters: JpaProductRepositoryAdapter, KafkaEventPublisher
✅ Domain events: sealed class hierarchy with aggregateId, occurredAt
✅ Aggregate root: Product.create() factory + event tracking
✅ Money value object: immutable, operator overloads, currency check
✅ ProductId value object: type-safe ID wrapper
✅ Strangler Fig: facade routing old/new via feature flags
✅ Dual write: new system authoritative + async legacy sync
✅ MigrationRunner: batch offset-based migration with failure tracking
✅ Outbox Pattern: guarantee events in same DB transaction
✅ OutboxRelay: @Scheduled poll + Kafka publish
✅ PESSIMISTIC_WRITE: prevent concurrent processing of same event
✅ Retry failed outbox events after delay
✅ Saga Pattern: orchestration with explicit state machine
✅ SagaState sealed class: Started/StockReserved/PaymentProcessed/etc.
✅ Compensation: reverse steps on failure
✅ compensateSaga: refund payment + release stock
✅ SagaStateConverter: serialize state to JSON for persistence
✅ Anti-Corruption Layer: translate legacy naming/formats to domain
✅ translateBack: domain → legacy format for writes
✅ Province enum: type-safe code mapping
✅ Phone normalization: strip non-numeric chars
```

---

*Part 87/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
