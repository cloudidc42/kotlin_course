# Part 91: Microservices Patterns

## สารบัญ
1. [Service Discovery & Load Balancing](#service-discovery--load-balancing)
2. [Circuit Breaker Pattern](#circuit-breaker-pattern)
3. [API Gateway Pattern](#api-gateway-pattern)
4. [Event-Driven Microservices](#event-driven-microservices)
5. [Distributed Tracing](#distributed-tracing)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Microservices Architecture Overview

```
Monolith → Microservices:

Monolith:
[UI] → [Single App: Products + Orders + Users + Payments] → [Single DB]

Microservices:
[API Gateway]
    ↓
[Product Service] → [Product DB]
[Order Service]   → [Order DB]
[User Service]    → [User DB]
[Payment Service] → [Payment DB]
[Notification Service]

Komunikasi:
Synchronous: REST / gRPC (request-response, tight coupling)
Asynchronous: Kafka / RabbitMQ (events, loose coupling)

Benefits:
✅ Independent deployment
✅ Independent scaling
✅ Technology diversity
✅ Team autonomy
✅ Fault isolation

Challenges:
❌ Distributed system complexity
❌ Network latency
❌ Data consistency (no ACID across services)
❌ Debugging harder
❌ Service discovery needed
```

---

## Service Discovery & Load Balancing

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-netflix-eureka-client")
    implementation("org.springframework.cloud:spring-cloud-starter-loadbalancer")
    implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
    implementation("io.github.resilience4j:resilience4j-spring-boot3:2.1.0")
}

// application.yml
/*
spring:
  application:
    name: order-service
  cloud:
    discovery:
      client:
        simple:
          instances:
            product-service:
              - uri: http://product-service-1:8081
              - uri: http://product-service-2:8081
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka
  instance:
    prefer-ip-address: true
    instance-id: ${spring.application.name}:${random.value}
*/

// Feign Client: declarative REST client with load balancing
import org.springframework.cloud.openfeign.FeignClient
import org.springframework.web.bind.annotation.*

@FeignClient(
    name = "product-service",
    fallbackFactory = ProductClientFallbackFactory::class
)
interface ProductServiceClient {
    
    @GetMapping("/api/v1/products/{id}")
    fun getProduct(@PathVariable id: String): ProductDto
    
    @GetMapping("/api/v1/products")
    fun getProducts(
        @RequestParam(required = false) categoryId: String?,
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int
    ): ProductPageDto
    
    @PostMapping("/api/v1/products/{id}/reserve")
    fun reserveStock(
        @PathVariable id: String,
        @RequestBody request: ReserveStockRequest
    ): ReserveStockResponse
    
    @PostMapping("/api/v1/products/{id}/release")
    fun releaseStock(
        @PathVariable id: String,
        @RequestBody request: ReleaseStockRequest
    )
}

// Fallback factory for circuit breaking
@org.springframework.stereotype.Component
class ProductClientFallbackFactory : feign.hystrix.FallbackFactory<ProductServiceClient> {
    
    private val log = org.slf4j.LoggerFactory.getLogger(ProductClientFallbackFactory::class.java)
    
    override fun create(cause: Throwable): ProductServiceClient {
        log.error("Product service call failed", cause)
        
        return object : ProductServiceClient {
            override fun getProduct(id: String): ProductDto {
                throw ServiceUnavailableException("Product service unavailable", cause)
            }
            
            override fun getProducts(categoryId: String?, page: Int, size: Int): ProductPageDto {
                return ProductPageDto(emptyList(), 0)  // empty page as fallback
            }
            
            override fun reserveStock(id: String, request: ReserveStockRequest): ReserveStockResponse {
                throw ServiceUnavailableException("Cannot reserve stock", cause)
            }
            
            override fun releaseStock(id: String, request: ReleaseStockRequest) {
                log.warn("Could not release stock for $id - will need manual intervention")
            }
        }
    }
}

// WebClient with load balancing (reactive alternative to Feign)
@org.springframework.context.annotation.Configuration
class WebClientConfig {
    
    @org.springframework.context.annotation.Bean
    @org.springframework.cloud.client.loadbalancer.LoadBalanced
    fun loadBalancedWebClient(): org.springframework.web.reactive.function.client.WebClient.Builder {
        return org.springframework.web.reactive.function.client.WebClient.builder()
    }
}

@org.springframework.stereotype.Service
class ReactiveProductClient(
    @org.springframework.beans.factory.annotation.Qualifier("loadBalancedWebClient")
    private val webClientBuilder: org.springframework.web.reactive.function.client.WebClient.Builder
) {
    private val client = webClientBuilder.baseUrl("http://product-service").build()
    
    fun getProduct(id: String): reactor.core.publisher.Mono<ProductDto> {
        return client.get()
            .uri("/api/v1/products/{id}", id)
            .retrieve()
            .onStatus(org.springframework.http.HttpStatusCode::is4xxClientError) { response ->
                reactor.core.publisher.Mono.error(
                    NotFoundException("Product $id not found (${response.statusCode()})")
                )
            }
            .bodyToMono(ProductDto::class.java)
            .timeout(java.time.Duration.ofSeconds(5))
    }
}

data class ReserveStockRequest(val quantity: Int, val reservationId: String)
data class ReleaseStockRequest(val reservationId: String)
data class ReserveStockResponse(val success: Boolean, val reservationId: String)
data class ProductPageDto(val items: List<ProductDto>, val total: Long)
data class ProductDto(val id: String, val name: String, val price: Double)
class ServiceUnavailableException(msg: String, cause: Throwable? = null) : RuntimeException(msg, cause)
class NotFoundException(msg: String) : RuntimeException(msg)
```

---

## Circuit Breaker Pattern

```kotlin
// Resilience4j Circuit Breaker
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker
import io.github.resilience4j.retry.annotation.Retry
import io.github.resilience4j.bulkhead.annotation.Bulkhead
import io.github.resilience4j.timelimiter.annotation.TimeLimiter
import java.util.concurrent.CompletableFuture

// application.yml
/*
resilience4j:
  circuitbreaker:
    instances:
      product-service:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10
        minimum-number-of-calls: 5
        failure-rate-threshold: 50        # 50% failures → OPEN
        wait-duration-in-open-state: 30s  # wait 30s before HALF_OPEN
        permitted-number-of-calls-in-half-open-state: 3
        automatic-transition-from-open-to-half-open-enabled: true
        record-exceptions:
          - java.io.IOException
          - feign.FeignException
        ignore-exceptions:
          - com.example.NotFoundException
  
  retry:
    instances:
      product-service:
        max-attempts: 3
        wait-duration: 500ms
        exponential-backoff-multiplier: 2
        retry-exceptions:
          - java.io.IOException
          - feign.FeignException.ServiceUnavailable
  
  bulkhead:
    instances:
      product-service:
        max-concurrent-calls: 20
        max-wait-duration: 100ms
  
  timelimiter:
    instances:
      product-service:
        timeout-duration: 5s
*/

@org.springframework.stereotype.Service
class ProductServiceAdapter(
    private val productClient: ProductServiceClient,
    private val cacheService: CacheService
) {
    
    @CircuitBreaker(name = "product-service", fallbackMethod = "getProductFallback")
    @Retry(name = "product-service")
    @Bulkhead(name = "product-service")
    @TimeLimiter(name = "product-service")
    fun getProductAsync(id: String): CompletableFuture<ProductDto> {
        return CompletableFuture.supplyAsync {
            productClient.getProduct(id)
        }
    }
    
    @CircuitBreaker(name = "product-service", fallbackMethod = "getProductsFallback")
    fun getProducts(categoryId: String?): List<ProductDto> {
        return productClient.getProducts(categoryId).items
    }
    
    // Fallback method signature must match (plus Throwable)
    private fun getProductFallback(id: String, e: Throwable): CompletableFuture<ProductDto> {
        return cacheService.getCachedProduct(id)
            ?.let { CompletableFuture.completedFuture(it) }
            ?: CompletableFuture.failedFuture(ServiceUnavailableException("Product $id unavailable", e))
    }
    
    private fun getProductsFallback(categoryId: String?, e: Throwable): List<ProductDto> {
        return emptyList()
    }
}

// Manual circuit breaker (for more control)
@org.springframework.stereotype.Service
class ManualCircuitBreakerService(
    private val circuitBreakerRegistry: io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry
) {
    
    fun callWithCircuitBreaker(serviceId: String, block: () -> ProductDto): ProductDto {
        val cb = circuitBreakerRegistry.circuitBreaker(serviceId)
        
        return io.github.resilience4j.circuitbreaker.CircuitBreaker.decorateSupplier(cb) {
            block()
        }.get()
    }
    
    fun getCircuitBreakerStatus(serviceId: String): CircuitBreakerStatus {
        val cb = circuitBreakerRegistry.circuitBreaker(serviceId)
        val metrics = cb.metrics
        
        return CircuitBreakerStatus(
            state = cb.state.name,
            failureRate = metrics.failureRate,
            callsInLastWindow = metrics.numberOfBufferedCalls,
            failedCalls = metrics.numberOfFailedCalls,
            successfulCalls = metrics.numberOfSuccessfulCalls
        )
    }
}

data class CircuitBreakerStatus(
    val state: String,
    val failureRate: Float,
    val callsInLastWindow: Int,
    val failedCalls: Int,
    val successfulCalls: Int
)

interface CacheService {
    fun getCachedProduct(id: String): ProductDto?
}
```

---

## Event-Driven Microservices

```kotlin
// Apache Kafka integration
// build.gradle.kts additions:
// implementation("org.springframework.kafka:spring-kafka")

// application.yml
/*
spring:
  kafka:
    bootstrap-servers: kafka:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 1
    consumer:
      group-id: order-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      properties:
        spring.json.trusted.packages: "com.ecommerce.*"
*/

// Events (shared across services)
sealed class EcommerceEvent {
    abstract val eventId: String
    abstract val occurredAt: java.time.Instant
    abstract val aggregateId: String
}

data class OrderPlacedEvent(
    override val eventId: String = java.util.UUID.randomUUID().toString(),
    override val occurredAt: java.time.Instant = java.time.Instant.now(),
    override val aggregateId: String,
    val customerId: String,
    val items: List<OrderItemEvent>,
    val totalAmount: java.math.BigDecimal,
    val currency: String
) : EcommerceEvent()

data class OrderItemEvent(val productId: String, val quantity: Int, val unitPrice: java.math.BigDecimal)

data class PaymentProcessedEvent(
    override val eventId: String = java.util.UUID.randomUUID().toString(),
    override val occurredAt: java.time.Instant = java.time.Instant.now(),
    override val aggregateId: String,
    val orderId: String,
    val amount: java.math.BigDecimal,
    val paymentId: String,
    val status: String  // "SUCCESS" | "FAILED"
) : EcommerceEvent()

data class StockReservedEvent(
    override val eventId: String = java.util.UUID.randomUUID().toString(),
    override val occurredAt: java.time.Instant = java.time.Instant.now(),
    override val aggregateId: String,
    val orderId: String,
    val reservations: Map<String, Int>  // productId → quantity
) : EcommerceEvent()

// Kafka Topics constants
object KafkaTopics {
    const val ORDERS = "ecommerce.orders"
    const val PAYMENTS = "ecommerce.payments"
    const val INVENTORY = "ecommerce.inventory"
    const val NOTIFICATIONS = "ecommerce.notifications"
}

// Producer (Order Service)
@org.springframework.stereotype.Service
class OrderEventPublisher(
    private val kafkaTemplate: org.springframework.kafka.core.KafkaTemplate<String, Any>
) {
    
    private val log = org.slf4j.LoggerFactory.getLogger(OrderEventPublisher::class.java)
    
    fun publishOrderPlaced(event: OrderPlacedEvent) {
        kafkaTemplate.send(
            KafkaTopics.ORDERS,
            event.aggregateId,  // key = orderId for partitioning
            event
        ).thenAccept { result ->
            log.info(
                "Published OrderPlaced: orderId=${event.aggregateId}, " +
                "partition=${result.recordMetadata.partition()}, " +
                "offset=${result.recordMetadata.offset()}"
            )
        }.exceptionally { e ->
            log.error("Failed to publish OrderPlaced: ${event.aggregateId}", e)
            null
        }
    }
    
    // Transactional publish (exactly-once with DB transaction)
    @org.springframework.transaction.annotation.Transactional
    fun publishWithTransaction(event: OrderPlacedEvent) {
        // Publish in same transaction as DB write
        kafkaTemplate.executeInTransaction { ops ->
            ops.send(KafkaTopics.ORDERS, event.aggregateId, event)
        }
    }
}

// Consumer (Inventory Service)
@org.springframework.kafka.annotation.KafkaListener(
    topics = [KafkaTopics.ORDERS],
    groupId = "inventory-service",
    containerFactory = "kafkaListenerContainerFactory"
)
@org.springframework.stereotype.Component
class InventoryOrderConsumer(
    private val inventoryService: InventoryService
) {
    
    private val log = org.slf4j.LoggerFactory.getLogger(InventoryOrderConsumer::class.java)
    
    @org.springframework.kafka.annotation.KafkaHandler
    fun handleOrderPlaced(
        event: OrderPlacedEvent,
        @org.springframework.messaging.handler.annotation.Header(
            org.springframework.kafka.support.KafkaHeaders.RECEIVED_TOPIC
        ) topic: String,
        ack: org.springframework.kafka.support.Acknowledgment
    ) {
        log.info("Processing order: ${event.aggregateId}")
        
        try {
            inventoryService.reserveStock(event.aggregateId, event.items)
            ack.acknowledge()  // manual ack after successful processing
        } catch (e: Exception) {
            log.error("Failed to process order ${event.aggregateId}", e)
            // Don't ack → will be retried
            throw e
        }
    }
}

// Consumer (Payment Service)
@org.springframework.stereotype.Component
class PaymentOrderConsumer(
    private val paymentService: PaymentService,
    private val paymentEventPublisher: PaymentEventPublisher
) {
    
    @org.springframework.kafka.annotation.KafkaListener(
        topics = [KafkaTopics.ORDERS],
        groupId = "payment-service"
    )
    fun handleOrderPlaced(event: OrderPlacedEvent) {
        val result = paymentService.processPayment(event.customerId, event.totalAmount)
        
        val paymentEvent = PaymentProcessedEvent(
            aggregateId = java.util.UUID.randomUUID().toString(),
            orderId = event.aggregateId,
            amount = event.totalAmount,
            paymentId = result.paymentId,
            status = if (result.success) "SUCCESS" else "FAILED"
        )
        
        paymentEventPublisher.publish(paymentEvent)
    }
}

// Consumer (Notification Service)
@org.springframework.stereotype.Component
class NotificationConsumer(
    private val emailService: EmailService,
    private val smsService: SmsService
) {
    
    @org.springframework.kafka.annotation.KafkaListener(
        topics = [KafkaTopics.PAYMENTS],
        groupId = "notification-service"
    )
    fun handlePaymentProcessed(event: PaymentProcessedEvent) {
        when (event.status) {
            "SUCCESS" -> {
                emailService.sendOrderConfirmation(event.orderId)
                smsService.sendPaymentSuccess(event.orderId, event.amount)
            }
            "FAILED" -> {
                emailService.sendPaymentFailed(event.orderId)
            }
        }
    }
}

// Dead Letter Queue handling
@org.springframework.context.annotation.Configuration
class KafkaConfig {
    
    @org.springframework.context.annotation.Bean
    fun kafkaListenerContainerFactory(
        consumerFactory: org.springframework.kafka.core.ConsumerFactory<String, Any>,
        kafkaTemplate: org.springframework.kafka.core.KafkaTemplate<String, Any>
    ): org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory<String, Any> {
        val factory = org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory<String, Any>()
        factory.consumerFactory = consumerFactory
        factory.containerProperties.ackMode = 
            org.springframework.kafka.listener.ContainerProperties.AckMode.MANUAL
        
        // Retry 3 times, then send to DLQ
        factory.setCommonErrorHandler(
            org.springframework.kafka.listener.DefaultErrorHandler(
                org.springframework.kafka.listener.DeadLetterPublishingRecoverer(kafkaTemplate) { record, e ->
                    org.apache.kafka.common.TopicPartition("${record.topic()}.DLQ", record.partition())
                },
                org.springframework.util.backoff.FixedBackOff(1000L, 3L)  // 1s delay, 3 retries
            )
        )
        
        return factory
    }
}

// Idempotency: prevent duplicate processing
@org.springframework.stereotype.Component
class IdempotencyChecker(
    private val processedEvents: org.springframework.data.redis.core.RedisTemplate<String, String>
) {
    
    fun isProcessed(eventId: String): Boolean {
        return processedEvents.hasKey("processed:$eventId") ?: false
    }
    
    fun markProcessed(eventId: String) {
        processedEvents.opsForValue().set(
            "processed:$eventId",
            "true",
            java.time.Duration.ofDays(7)
        )
    }
}

interface InventoryService {
    fun reserveStock(orderId: String, items: List<OrderItemEvent>)
}

interface PaymentService {
    fun processPayment(customerId: String, amount: java.math.BigDecimal): PaymentResult
}

data class PaymentResult(val success: Boolean, val paymentId: String)

interface PaymentEventPublisher {
    fun publish(event: PaymentProcessedEvent)
}

interface EmailService {
    fun sendOrderConfirmation(orderId: String)
    fun sendPaymentFailed(orderId: String)
}

interface SmsService {
    fun sendPaymentSuccess(orderId: String, amount: java.math.BigDecimal)
}
```

---

## Distributed Tracing

```kotlin
// build.gradle.kts
// implementation("io.micrometer:micrometer-tracing-bridge-brave")
// implementation("io.zipkin.reporter2:zipkin-reporter-brave")
// implementation("io.micrometer:micrometer-tracing")

// application.yml
/*
management:
  tracing:
    sampling:
      probability: 1.0  # 100% in dev, 0.1 in prod
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
*/

// Trace propagation through Kafka messages
@org.springframework.stereotype.Component
class TracingKafkaProducer(
    private val tracer: io.micrometer.tracing.Tracer,
    private val kafkaTemplate: org.springframework.kafka.core.KafkaTemplate<String, Any>
) {
    
    fun sendWithTrace(topic: String, key: String, value: Any) {
        val span = tracer.currentSpan() ?: tracer.nextSpan().name("kafka.send.$topic").start()
        
        try {
            val headers = mutableListOf<org.apache.kafka.common.header.internals.RecordHeader>()
            
            // Inject trace context into Kafka headers
            tracer.propagation().injector { carrier: MutableList<org.apache.kafka.common.header.internals.RecordHeader>, key2, value2 ->
                carrier.add(org.apache.kafka.common.header.internals.RecordHeader(key2, value2.toByteArray()))
            }.inject(span.context(), headers)
            
            val record = org.apache.kafka.clients.producer.ProducerRecord(topic, null, key, value, headers)
            kafkaTemplate.send(record)
        } finally {
            span.end()
        }
    }
}

// Custom span for business operations
@org.springframework.stereotype.Service
class TracedOrderService(
    private val tracer: io.micrometer.tracing.Tracer,
    private val orderRepository: OrderRepository
) {
    
    fun placeOrder(command: PlaceOrderCommand): Order {
        val span = tracer.nextSpan()
            .name("order.place")
            .tag("customer.id", command.customerId)
            .start()
        
        return tracer.withSpan(span).use { ws ->
            try {
                // validate
                val validateSpan = tracer.nextSpan().name("order.validate").start()
                validateOrder(command)
                validateSpan.end()
                
                // create
                val order = orderRepository.save(command.toOrder())
                
                span.tag("order.id", order.id)
                span.event("order.created")
                
                order
            } catch (e: Exception) {
                span.error(e)
                throw e
            } finally {
                span.end()
            }
        }
    }
    
    private fun validateOrder(command: PlaceOrderCommand) { /* validation */ }
}

data class PlaceOrderCommand(val customerId: String, val items: List<Any>)
data class Order(val id: String, val customerId: String)
interface OrderRepository {
    fun save(order: Order): Order
}
fun PlaceOrderCommand.toOrder() = Order(java.util.UUID.randomUUID().toString(), customerId)
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง Microservice Communication Pattern

// ออกแบบ Order Checkout Flow:
// 1. Order Service รับ checkout request
// 2. ส่ง event OrderCheckoutInitiated ไปยัง Kafka
// 3. Inventory Service รับ event → reserve stock → ส่ง StockReserved event
// 4. Payment Service รับ StockReserved → process payment → ส่ง PaymentResult event
// 5. Order Service รับ PaymentResult → update order status
// 6. Notification Service รับทุก event → send appropriate notifications

// Key patterns ที่ต้องใช้:
// - Circuit Breaker: Inventory/Payment client calls
// - Idempotency: check eventId ก่อน process ทุกครั้ง
// - Dead Letter Queue: events ที่ process ไม่ได้
// - Distributed tracing: ติดตาม flow ผ่าน services ทั้งหมด

data class OrderCheckoutInitiatedEvent(
    val eventId: String = java.util.UUID.randomUUID().toString(),
    val orderId: String,
    val customerId: String,
    val items: List<CheckoutItem>,
    val totalAmount: java.math.BigDecimal
)

data class CheckoutItem(val productId: String, val quantity: Int, val price: java.math.BigDecimal)

// Order Service
@org.springframework.stereotype.Service
class CheckoutOrchestrator(
    private val orderRepository: OrderRepository,
    private val eventPublisher: EventPublisher,
    private val idempotencyChecker: IdempotencyChecker,
    private val tracer: io.micrometer.tracing.Tracer
) {
    
    fun initiateCheckout(command: CheckoutCommand): String {
        val orderId = java.util.UUID.randomUUID().toString()
        val order = Order(orderId, command.customerId)
        orderRepository.save(order)
        
        val event = OrderCheckoutInitiatedEvent(
            orderId = orderId,
            customerId = command.customerId,
            items = command.items.map { CheckoutItem(it.productId, it.quantity, it.price) },
            totalAmount = command.items.sumOf { it.price.multiply(java.math.BigDecimal(it.quantity)) }
        )
        
        eventPublisher.publish(KafkaTopics.ORDERS, orderId, event)
        return orderId
    }
    
    @org.springframework.kafka.annotation.KafkaListener(topics = ["ecommerce.payments"])
    fun handlePaymentResult(event: PaymentProcessedEvent) {
        if (idempotencyChecker.isProcessed(event.eventId)) return
        
        val status = if (event.status == "SUCCESS") "CONFIRMED" else "PAYMENT_FAILED"
        orderRepository.updateStatus(event.orderId, status)
        
        idempotencyChecker.markProcessed(event.eventId)
    }
}

interface EventPublisher {
    fun publish(topic: String, key: String, value: Any)
}

interface OrderRepository {
    fun save(order: Order): Order
    fun updateStatus(orderId: String, status: String)
}

data class CheckoutCommand(val customerId: String, val items: List<CheckoutCommandItem>)
data class CheckoutCommandItem(val productId: String, val quantity: Int, val price: java.math.BigDecimal)
```

---

## สรุป Part 91

```
✅ Service Discovery: Eureka client registration
✅ FeignClient: declarative REST client with @FeignClient
✅ FallbackFactory: fallback per exception type
✅ @LoadBalanced WebClient: client-side load balancing
✅ Circuit Breaker: CLOSED → OPEN → HALF_OPEN states
✅ @CircuitBreaker/@Retry/@Bulkhead/@TimeLimiter: annotations
✅ Fallback method: same signature + Throwable
✅ CircuitBreakerRegistry: programmatic control
✅ Event-Driven: loose coupling via Kafka messages
✅ Sealed class events: type-safe domain events
✅ KafkaTopics: constants for topic names
✅ @KafkaListener: consumer with groupId
✅ Acknowledgment.acknowledge(): manual ack
✅ KafkaTemplate.send(): async publish
✅ executeInTransaction: transactional kafka send
✅ DefaultErrorHandler + DeadLetterPublishingRecoverer: DLQ
✅ FixedBackOff: retry delay + max attempts
✅ Idempotency: Redis set to track processed eventIds
✅ Distributed Tracing: Micrometer + Zipkin
✅ span.tag/event/error: custom span metadata
✅ Trace propagation in Kafka headers
✅ withSpan: scope-based span context
✅ Choreography: services react to events independently
✅ Orchestration: one service coordinates the flow
✅ Checkout flow: order → inventory → payment → notification
```

---

*Part 91/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
