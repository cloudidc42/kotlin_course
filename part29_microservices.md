# Part 29: Microservices กับ Kotlin

## สารบัญ
1. [Microservices Architecture](#microservices-architecture)
2. [Service Communication](#service-communication)
3. [API Gateway Pattern](#api-gateway-pattern)
4. [Event-Driven Architecture](#event-driven-architecture)
5. [Circuit Breaker Pattern](#circuit-breaker-pattern)
6. [Service Discovery](#service-discovery)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Microservices Architecture

```kotlin
// ตัวอย่าง: E-Commerce Microservices
// Services:
// 1. Order Service
// 2. Product Service  
// 3. User Service
// 4. Payment Service
// 5. Notification Service

// build.gradle.kts for microservice
plugins {
    kotlin("jvm") version "1.9.22"
    kotlin("plugin.spring") version "1.9.22"
    id("org.springframework.boot") version "3.2.0"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-client")
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
    implementation("org.springframework.cloud:spring-cloud-starter-circuitbreaker-resilience4j")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")
    
    // Messaging
    implementation("org.springframework.kafka:spring-kafka")
    
    // Distributed tracing
    implementation("io.micrometer:micrometer-tracing-bridge-brave")
}
```

---

## Service Communication

```kotlin
// HTTP Client with Feign
@FeignClient(
    name = "product-service",
    fallbackFactory = ProductClientFallbackFactory::class
)
interface ProductServiceClient {
    @GetMapping("/api/products/{id}")
    fun getProduct(@PathVariable id: Long): ProductDto?
    
    @GetMapping("/api/products")
    fun getProducts(@RequestParam ids: List<Long>): List<ProductDto>
    
    @PutMapping("/api/products/{id}/stock")
    fun decreaseStock(
        @PathVariable id: Long,
        @RequestParam quantity: Int
    ): ProductDto
}

data class ProductDto(
    val id: Long,
    val name: String,
    val price: Double,
    val stock: Int
)

@Component
class ProductClientFallbackFactory : FallbackFactory<ProductServiceClient> {
    override fun create(cause: Throwable): ProductServiceClient {
        return object : ProductServiceClient {
            override fun getProduct(id: Long): ProductDto? {
                println("Fallback: getProduct($id) - ${cause.message}")
                return null
            }
            
            override fun getProducts(ids: List<Long>): List<ProductDto> {
                println("Fallback: getProducts - ${cause.message}")
                return emptyList()
            }
            
            override fun decreaseStock(id: Long, quantity: Int): ProductDto {
                throw ServiceUnavailableException("Product service unavailable")
            }
        }
    }
}

class ServiceUnavailableException(message: String) : RuntimeException(message)

// User Service Client
@FeignClient(name = "user-service")
interface UserServiceClient {
    @GetMapping("/api/users/{id}")
    fun getUser(@PathVariable id: Long): UserDto?
    
    @GetMapping("/api/users/{id}/address")
    fun getUserAddress(@PathVariable id: Long): AddressDto?
}

data class UserDto(val id: Long, val name: String, val email: String)
data class AddressDto(val street: String, val city: String, val country: String)

// Using WebClient (reactive)
@Service
class ReactiveProductClient(
    private val webClient: WebClient.Builder
) {
    private val client = webClient
        .baseUrl("http://product-service")
        .defaultHeader("Content-Type", "application/json")
        .build()
    
    suspend fun getProduct(id: Long): ProductDto? {
        return try {
            client.get()
                .uri("/api/products/$id")
                .retrieve()
                .awaitBodyOrNull<ProductDto>()
        } catch (e: Exception) {
            null
        }
    }
    
    fun getProductFlow(ids: List<Long>) = ids.asFlow()
        .flatMapMerge { id ->
            flow {
                val product = getProduct(id)
                if (product != null) emit(product)
            }
        }
}
```

---

## Event-Driven Architecture

```kotlin
// Kafka with Spring
import org.springframework.kafka.annotation.KafkaListener
import org.springframework.kafka.core.KafkaTemplate

// Events
data class OrderCreatedEvent(
    val orderId: Long,
    val userId: Long,
    val items: List<OrderItemDto>,
    val total: Double,
    val timestamp: java.time.Instant = java.time.Instant.now()
)

data class OrderItemDto(
    val productId: Long,
    val productName: String,
    val quantity: Int,
    val price: Double
)

data class PaymentProcessedEvent(
    val orderId: Long,
    val paymentId: String,
    val amount: Double,
    val status: String,
    val timestamp: java.time.Instant = java.time.Instant.now()
)

data class StockReservedEvent(
    val orderId: Long,
    val items: List<OrderItemDto>,
    val reserved: Boolean,
    val reason: String? = null
)

// Order Service - Producer
@Service
class OrderService(
    private val orderRepo: OrderRepository,
    private val kafkaTemplate: KafkaTemplate<String, Any>
) {
    fun createOrder(request: CreateOrderRequest): Order {
        val order = orderRepo.save(Order(
            userId = request.userId,
            items = request.items.map { OrderItem(it.productId, it.quantity, it.price) },
            status = OrderStatus.PENDING
        ))
        
        // Publish event
        kafkaTemplate.send("order.created", order.id.toString(), OrderCreatedEvent(
            orderId = order.id,
            userId = request.userId,
            items = request.items.map { OrderItemDto(it.productId, it.productName, it.quantity, it.price) },
            total = request.items.sumOf { it.price * it.quantity }
        ))
        
        return order
    }
    
    fun confirmOrder(orderId: Long): Order {
        val order = orderRepo.findById(orderId).orElseThrow()
        order.status = OrderStatus.CONFIRMED
        return orderRepo.save(order)
    }
    
    fun cancelOrder(orderId: Long, reason: String): Order {
        val order = orderRepo.findById(orderId).orElseThrow()
        order.status = OrderStatus.CANCELLED
        order.cancellationReason = reason
        return orderRepo.save(order)
    }
}

// Inventory Service - Consumer
@Service
class InventoryService(
    private val inventoryRepo: InventoryRepository,
    private val kafkaTemplate: KafkaTemplate<String, Any>
) {
    @KafkaListener(topics = ["order.created"], groupId = "inventory-service")
    fun handleOrderCreated(event: OrderCreatedEvent) {
        println("Processing inventory for order ${event.orderId}")
        
        val canReserve = event.items.all { item ->
            val inventory = inventoryRepo.findByProductId(item.productId)
            inventory != null && inventory.quantity >= item.quantity
        }
        
        if (canReserve) {
            event.items.forEach { item ->
                val inventory = inventoryRepo.findByProductId(item.productId)!!
                inventory.quantity -= item.quantity
                inventory.reserved += item.quantity
                inventoryRepo.save(inventory)
            }
            
            kafkaTemplate.send("stock.reserved", event.orderId.toString(), StockReservedEvent(
                orderId = event.orderId,
                items = event.items,
                reserved = true
            ))
        } else {
            kafkaTemplate.send("stock.reserved", event.orderId.toString(), StockReservedEvent(
                orderId = event.orderId,
                items = event.items,
                reserved = false,
                reason = "Insufficient stock"
            ))
        }
    }
    
    @KafkaListener(topics = ["order.cancelled"], groupId = "inventory-service")
    fun handleOrderCancelled(event: OrderCancelledEvent) {
        event.items.forEach { item ->
            val inventory = inventoryRepo.findByProductId(item.productId)
            inventory?.let {
                it.quantity += item.quantity
                it.reserved -= item.quantity
                inventoryRepo.save(it)
            }
        }
    }
}

// Notification Service - Consumer
@Service
class NotificationService(
    private val emailService: EmailService
) {
    @KafkaListener(topics = ["order.created"])
    fun onOrderCreated(event: OrderCreatedEvent) {
        emailService.send(
            to = "user@email.com",
            subject = "Order Confirmed #${event.orderId}",
            body = "Your order has been placed. Total: ฿${event.total}"
        )
    }
    
    @KafkaListener(topics = ["payment.processed"])
    fun onPaymentProcessed(event: PaymentProcessedEvent) {
        when (event.status) {
            "SUCCESS" -> emailService.send(
                to = "user@email.com",
                subject = "Payment Received",
                body = "Payment of ฿${event.amount} received for order #${event.orderId}"
            )
            "FAILED" -> emailService.send(
                to = "user@email.com",
                subject = "Payment Failed",
                body = "Payment failed for order #${event.orderId}. Please retry."
            )
        }
    }
}

// Kafka Configuration
@Configuration
class KafkaConfig {
    @Bean
    fun producerFactory(kafkaProperties: KafkaProperties): ProducerFactory<String, Any> {
        val props = kafkaProperties.buildProducerProperties()
        props[ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG] = StringSerializer::class.java
        props[ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG] = JsonSerializer::class.java
        return DefaultKafkaProducerFactory(props)
    }
    
    @Bean
    fun kafkaTemplate(producerFactory: ProducerFactory<String, Any>) =
        KafkaTemplate(producerFactory)
    
    @Bean
    fun consumerFactory(kafkaProperties: KafkaProperties): ConsumerFactory<String, Any> {
        val props = kafkaProperties.buildConsumerProperties()
        props[ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG] = StringDeserializer::class.java
        props[ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG] = JsonDeserializer::class.java
        props[JsonDeserializer.TRUSTED_PACKAGES] = "*"
        return DefaultKafkaConsumerFactory(props)
    }
}
```

---

## Circuit Breaker Pattern

```kotlin
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker
import io.github.resilience4j.retry.annotation.Retry
import io.github.resilience4j.bulkhead.annotation.Bulkhead
import io.github.resilience4j.timelimiter.annotation.TimeLimiter

@Service
class ResilientOrderService(
    private val productClient: ProductServiceClient,
    private val paymentClient: PaymentServiceClient
) {
    // Circuit Breaker: จะ trip เมื่อ failure rate > threshold
    @CircuitBreaker(name = "product-service", fallbackMethod = "getProductFallback")
    @Retry(name = "product-service")
    fun getProduct(id: Long): ProductDto? {
        return productClient.getProduct(id)
    }
    
    fun getProductFallback(id: Long, ex: Throwable): ProductDto? {
        println("Circuit breaker: product-service is open. Error: ${ex.message}")
        return null
    }
    
    // Bulkhead: จำกัด concurrent calls
    @Bulkhead(name = "payment-service", type = Bulkhead.Type.SEMAPHORE)
    @TimeLimiter(name = "payment-service")
    suspend fun processPayment(orderId: Long, amount: Double): Boolean {
        return paymentClient.processPayment(orderId, amount)
    }
    
    // Retry with exponential backoff
    @Retry(name = "external-api", fallbackMethod = "externalApiFallback")
    fun callExternalApi(data: String): String {
        return externalApiClient.call(data)
    }
    
    fun externalApiFallback(data: String, ex: Throwable): String {
        return "cached_result_for_$data"
    }
}

// Configuration in application.yml:
/*
resilience4j:
  circuitbreaker:
    instances:
      product-service:
        sliding-window-size: 10
        minimum-number-of-calls: 5
        permitted-number-of-calls-in-half-open-state: 3
        wait-duration-in-open-state: 30s
        failure-rate-threshold: 50
        slow-call-rate-threshold: 80
        slow-call-duration-threshold: 3s
  
  retry:
    instances:
      product-service:
        max-attempts: 3
        wait-duration: 500ms
        retry-exceptions:
          - java.io.IOException
          - feign.FeignException$ServiceUnavailable
  
  bulkhead:
    instances:
      payment-service:
        max-concurrent-calls: 10
        max-wait-duration: 1s
  
  timelimiter:
    instances:
      payment-service:
        timeout-duration: 5s
*/
```

---

## Distributed Tracing

```kotlin
// OpenTelemetry / Micrometer Tracing
import io.micrometer.tracing.Tracer
import io.micrometer.tracing.annotation.NewSpan
import io.micrometer.tracing.annotation.SpanTag

@Service
class TracedOrderService(
    private val tracer: Tracer
) {
    @NewSpan("create-order")
    fun createOrder(
        @SpanTag("userId") userId: Long,
        items: List<OrderItemDto>
    ): Long {
        val span = tracer.currentSpan()
        span?.tag("itemCount", items.size.toString())
        
        // Business logic
        val orderId = 123L
        
        span?.tag("orderId", orderId.toString())
        return orderId
    }
    
    fun processWithCustomSpan(data: String): String {
        val span = tracer.nextSpan().name("process-data").start()
        
        return try {
            tracer.withSpan(span).use {
                span.tag("input", data)
                val result = data.uppercase()
                span.tag("output", result)
                result
            }
        } catch (e: Exception) {
            span.error(e)
            throw e
        } finally {
            span.end()
        }
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง Saga Pattern สำหรับ Order Processing

sealed class SagaStep {
    abstract val name: String
    abstract suspend fun execute(context: SagaContext): SagaResult
    abstract suspend fun compensate(context: SagaContext)
}

data class SagaContext(
    val orderId: Long,
    val userId: Long,
    val items: List<OrderItemDto>,
    val total: Double,
    val metadata: MutableMap<String, Any> = mutableMapOf()
)

sealed class SagaResult {
    object Success : SagaResult()
    data class Failure(val reason: String) : SagaResult()
}

// Step 1: Validate Order
class ValidateOrderStep : SagaStep() {
    override val name = "validate-order"
    
    override suspend fun execute(context: SagaContext): SagaResult {
        println("[$name] Validating order ${context.orderId}")
        if (context.items.isEmpty()) return SagaResult.Failure("No items")
        if (context.total <= 0) return SagaResult.Failure("Invalid total")
        return SagaResult.Success
    }
    
    override suspend fun compensate(context: SagaContext) {
        println("[$name] Compensating: nothing to undo for validation")
    }
}

// Step 2: Reserve Stock
class ReserveStockStep : SagaStep() {
    override val name = "reserve-stock"
    
    override suspend fun execute(context: SagaContext): SagaResult {
        println("[$name] Reserving stock for ${context.items.size} items")
        // simulated: 90% success
        return if (Math.random() > 0.1) {
            context.metadata["reservationId"] = "RES-${System.currentTimeMillis()}"
            SagaResult.Success
        } else {
            SagaResult.Failure("Insufficient stock")
        }
    }
    
    override suspend fun compensate(context: SagaContext) {
        val reservationId = context.metadata["reservationId"] as? String
        println("[$name] Releasing reservation $reservationId")
    }
}

// Step 3: Charge Payment
class ChargePaymentStep : SagaStep() {
    override val name = "charge-payment"
    
    override suspend fun execute(context: SagaContext): SagaResult {
        println("[$name] Charging ฿${context.total}")
        return if (Math.random() > 0.05) {
            context.metadata["paymentId"] = "PAY-${System.currentTimeMillis()}"
            SagaResult.Success
        } else {
            SagaResult.Failure("Payment declined")
        }
    }
    
    override suspend fun compensate(context: SagaContext) {
        val paymentId = context.metadata["paymentId"] as? String
        println("[$name] Refunding payment $paymentId")
    }
}

// Saga Orchestrator
class OrderSaga(private val steps: List<SagaStep>) {
    suspend fun execute(context: SagaContext): Boolean {
        val executedSteps = mutableListOf<SagaStep>()
        
        for (step in steps) {
            when (val result = step.execute(context)) {
                SagaResult.Success -> {
                    executedSteps.add(step)
                    println("✅ ${step.name} succeeded")
                }
                is SagaResult.Failure -> {
                    println("❌ ${step.name} failed: ${result.reason}")
                    println("Starting compensation...")
                    
                    executedSteps.reversed().forEach { completedStep ->
                        completedStep.compensate(context)
                    }
                    
                    return false
                }
            }
        }
        
        return true
    }
}

suspend fun main() {
    val saga = OrderSaga(listOf(
        ValidateOrderStep(),
        ReserveStockStep(),
        ChargePaymentStep()
    ))
    
    repeat(3) { attempt ->
        println("\n=== Attempt ${attempt + 1} ===")
        val context = SagaContext(
            orderId = 1L,
            userId = 42L,
            items = listOf(
                OrderItemDto(1L, "Laptop", 1, 25000.0),
                OrderItemDto(2L, "Mouse", 2, 500.0)
            ),
            total = 26000.0
        )
        
        val success = saga.execute(context)
        println("Saga result: ${if (success) "SUCCESS" else "FAILED"}")
    }
}

// Placeholder interfaces
interface OrderRepository {
    fun findById(id: Long): Order?
    fun save(order: Order): Order
}

interface InventoryRepository {
    fun findByProductId(id: Long): Inventory?
    fun save(inv: Inventory): Inventory
}

interface PaymentServiceClient {
    suspend fun processPayment(orderId: Long, amount: Double): Boolean
}

interface ExternalApiClient {
    fun call(data: String): String
}

interface EmailService {
    fun send(to: String, subject: String, body: String)
}

data class Order(
    val id: Long = 0,
    val userId: Long = 0,
    val items: List<OrderItem> = emptyList(),
    var status: OrderStatus = OrderStatus.PENDING,
    var cancellationReason: String? = null
)

data class OrderItem(val productId: Long, val quantity: Int, val price: Double)
enum class OrderStatus { PENDING, CONFIRMED, CANCELLED }
data class Inventory(val productId: Long, var quantity: Int, var reserved: Int = 0)
data class OrderCancelledEvent(val orderId: Long, val items: List<OrderItemDto>)
data class CreateOrderRequest(val userId: Long, val items: List<OrderItemDto>)
```

---

## สรุป Part 29

```
✅ Microservices: แยก services ตาม business domain
✅ Feign: declarative HTTP client ระหว่าง services
✅ FallbackFactory: handle service failures gracefully
✅ WebClient: reactive HTTP client ด้วย coroutines
✅ Kafka: event-driven communication ระหว่าง services
✅ @KafkaListener: consume messages
✅ Circuit Breaker: ป้องกัน cascade failures
✅ Retry: retry ด้วย backoff เมื่อ transient failure
✅ Bulkhead: จำกัด concurrent access
✅ Distributed Tracing: ติดตาม request ข้าม services
✅ Saga Pattern: distributed transaction ด้วย compensation
✅ Event Sourcing: record events แทน state
```

---

*Part 29/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
