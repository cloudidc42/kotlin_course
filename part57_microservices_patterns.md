# Part 57: Microservices Patterns ขั้นสูง

## สารบัญ
1. [Saga Pattern](#saga-pattern)
2. [CQRS ขั้นสูง](#cqrs-ขั้นสูง)
3. [Outbox Pattern](#outbox-pattern)
4. [Service Mesh Concepts](#service-mesh-concepts)
5. [Distributed Tracing](#distributed-tracing)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Saga Pattern

Saga แก้ปัญหา distributed transaction — แทนที่ 2PC ด้วย compensating transactions

```kotlin
// Choreography-based Saga (event-driven, loose coupling)

// Each service publishes events and listens to others' events
// ไม่มี central orchestrator

// Order Service: publishes OrderCreated
@Service
class OrderSagaInitiator(
    private val kafkaTemplate: KafkaTemplate<String, SagaEvent>
) {
    
    fun initiateOrderPlacement(command: PlaceOrderSagaCommand): String {
        val sagaId = java.util.UUID.randomUUID().toString()
        
        kafkaTemplate.send(
            "saga.order.created",
            sagaId,
            OrderCreatedSagaEvent(
                sagaId = sagaId,
                orderId = command.orderId,
                customerId = command.customerId,
                items = command.items,
                totalAmount = command.totalAmount
            )
        )
        
        return sagaId
    }
}

// Inventory Service: listens to OrderCreated, publishes InventoryReserved or InventoryFailed
@Component
class InventoryOrderListener(
    private val inventoryService: InventoryService2,
    private val kafkaTemplate: KafkaTemplate<String, SagaEvent>
) {
    
    @KafkaListener(topics = ["saga.order.created"])
    fun onOrderCreated(event: OrderCreatedSagaEvent) {
        val result = inventoryService.reserveItems(event.sagaId, event.items)
        
        val responseEvent = if (result.isSuccess) {
            InventoryReservedEvent(
                sagaId = event.sagaId,
                orderId = event.orderId,
                reservationId = result.getOrThrow()
            )
        } else {
            InventoryReservationFailedEvent(
                sagaId = event.sagaId,
                orderId = event.orderId,
                reason = result.exceptionOrNull()?.message ?: "Unknown error"
            )
        }
        
        kafkaTemplate.send("saga.inventory.result", event.sagaId, responseEvent)
    }
    
    // Compensation: when payment fails, release reserved inventory
    @KafkaListener(topics = ["saga.payment.failed"])
    fun onPaymentFailed(event: PaymentFailedEvent) {
        inventoryService.releaseReservation(event.sagaId)
        kafkaTemplate.send("saga.inventory.released", event.sagaId,
            InventoryReleasedEvent(event.sagaId, event.orderId))
    }
}

// Payment Service: listens to InventoryReserved, processes payment
@Component
class PaymentInventoryListener(
    private val paymentService: PaymentService2,
    private val kafkaTemplate: KafkaTemplate<String, SagaEvent>
) {
    
    @KafkaListener(topics = ["saga.inventory.result"])
    fun onInventoryResult(event: SagaEvent) {
        when (event) {
            is InventoryReservedEvent -> processPayment(event)
            is InventoryReservationFailedEvent -> cancelOrder(event.sagaId, event.orderId, event.reason)
        }
    }
    
    private fun processPayment(event: InventoryReservedEvent) {
        val result = paymentService.charge(event.sagaId, event.orderId)
        
        val responseEvent = if (result.isSuccess) {
            PaymentCompletedEvent(event.sagaId, event.orderId, result.getOrThrow())
        } else {
            PaymentFailedEvent(event.sagaId, event.orderId, result.exceptionOrNull()?.message ?: "Payment failed")
        }
        
        kafkaTemplate.send("saga.payment.result", event.sagaId, responseEvent)
    }
    
    private fun cancelOrder(sagaId: String, orderId: String, reason: String) {
        kafkaTemplate.send("saga.order.cancel", sagaId,
            OrderCancellationRequestedEvent(sagaId, orderId, reason))
    }
}

// Orchestration-based Saga (central coordinator, easier to track)
@Service
class OrderSagaOrchestrator(
    private val sagaStateRepository: SagaStateRepository,
    private val inventoryClient: InventoryServiceClient,
    private val paymentClient: PaymentServiceClient,
    private val notificationClient: NotificationServiceClient
) {
    
    suspend fun execute(command: PlaceOrderSagaCommand): SagaResult {
        val sagaId = java.util.UUID.randomUUID().toString()
        val state = SagaState(sagaId, SagaStatus.STARTED, command)
        sagaStateRepository.save(state)
        
        return try {
            // Step 1: Reserve inventory
            val reservation = withCompensation(
                action = { inventoryClient.reserve(sagaId, command.items) },
                compensation = { inventoryClient.release(sagaId) }
            )
            sagaStateRepository.update(sagaId, SagaStatus.INVENTORY_RESERVED)
            
            // Step 2: Process payment
            val payment = withCompensation(
                action = { paymentClient.charge(sagaId, command.customerId, command.totalAmount) },
                compensation = { paymentClient.refund(sagaId) }
            )
            sagaStateRepository.update(sagaId, SagaStatus.PAYMENT_COMPLETED)
            
            // Step 3: Confirm order (no compensation needed - saga is committed)
            notificationClient.sendOrderConfirmation(command.orderId, command.customerId)
            sagaStateRepository.update(sagaId, SagaStatus.COMPLETED)
            
            SagaResult.Success(sagaId)
            
        } catch (e: SagaCompensationException) {
            sagaStateRepository.update(sagaId, SagaStatus.COMPENSATED)
            SagaResult.Failure(sagaId, e.message ?: "Saga failed and compensated")
        }
    }
    
    private suspend fun <T> withCompensation(
        action: suspend () -> T,
        compensation: suspend () -> Unit
    ): T {
        return try {
            action()
        } catch (e: Exception) {
            try { compensation() } catch (_: Exception) { }
            throw SagaCompensationException(e.message, e)
        }
    }
}

// DTOs and state
sealed class SagaEvent
data class OrderCreatedSagaEvent(val sagaId: String, val orderId: String, val customerId: String,
    val items: List<Any>, val totalAmount: Double) : SagaEvent()
data class InventoryReservedEvent(val sagaId: String, val orderId: String, val reservationId: String) : SagaEvent()
data class InventoryReservationFailedEvent(val sagaId: String, val orderId: String, val reason: String) : SagaEvent()
data class InventoryReleasedEvent(val sagaId: String, val orderId: String) : SagaEvent()
data class PaymentCompletedEvent(val sagaId: String, val orderId: String, val transactionId: String) : SagaEvent()
data class PaymentFailedEvent(val sagaId: String, val orderId: String, val reason: String) : SagaEvent()
data class OrderCancellationRequestedEvent(val sagaId: String, val orderId: String, val reason: String) : SagaEvent()

enum class SagaStatus { STARTED, INVENTORY_RESERVED, PAYMENT_COMPLETED, COMPLETED, COMPENSATED, FAILED }
data class SagaState(val sagaId: String, val status: SagaStatus, val command: Any)
data class PlaceOrderSagaCommand(val orderId: String, val customerId: String, val items: List<Any>, val totalAmount: Double)
sealed class SagaResult {
    data class Success(val sagaId: String) : SagaResult()
    data class Failure(val sagaId: String, val reason: String) : SagaResult()
}

class SagaCompensationException(message: String?, cause: Throwable? = null) : Exception(message, cause)

interface SagaStateRepository {
    fun save(state: SagaState)
    fun update(sagaId: String, status: SagaStatus)
}
interface InventoryServiceClient {
    suspend fun reserve(sagaId: String, items: List<Any>): String
    suspend fun release(sagaId: String)
}
interface PaymentServiceClient {
    suspend fun charge(sagaId: String, customerId: String, amount: Double): String
    suspend fun refund(sagaId: String)
}
interface NotificationServiceClient {
    suspend fun sendOrderConfirmation(orderId: String, customerId: String)
}
interface InventoryService2 {
    fun reserveItems(sagaId: String, items: List<Any>): Result<String>
    fun releaseReservation(sagaId: String)
}
interface PaymentService2 {
    fun charge(sagaId: String, orderId: String): Result<String>
}
```

---

## Outbox Pattern

ป้องกัน dual-write problem: database + message queue ไม่ consistent กัน

```kotlin
// Outbox table entity
@Entity(tableName = "outbox_events")
data class OutboxEvent(
    @Id val id: String = java.util.UUID.randomUUID().toString(),
    val aggregateType: String,
    val aggregateId: String,
    val eventType: String,
    val payload: String,  // JSON
    val status: OutboxStatus = OutboxStatus.PENDING,
    val createdAt: java.time.Instant = java.time.Instant.now(),
    val processedAt: java.time.Instant? = null,
    val retryCount: Int = 0
)

enum class OutboxStatus { PENDING, PROCESSING, SENT, FAILED }

// Service layer: write to DB and outbox in one transaction
@Service
@Transactional
class OrderCommandService(
    private val orderRepository: OrderDbRepository,
    private val outboxRepository: OutboxRepository,
    private val objectMapper: com.fasterxml.jackson.databind.ObjectMapper
) {
    
    fun placeOrder(command: PlaceOrderCmd): String {
        val order = OrderRecord(
            id = java.util.UUID.randomUUID().toString(),
            customerId = command.customerId,
            status = "PENDING",
            totalAmount = command.totalAmount
        )
        orderRepository.save(order)
        
        // Write to outbox (same transaction!)
        val event = OrderPlacedPayload(
            orderId = order.id,
            customerId = order.customerId,
            items = command.items,
            totalAmount = order.totalAmount
        )
        
        outboxRepository.save(OutboxEvent(
            aggregateType = "Order",
            aggregateId = order.id,
            eventType = "OrderPlaced",
            payload = objectMapper.writeValueAsString(event)
        ))
        
        return order.id
    }
}

// Outbox Relay: polls outbox and publishes to Kafka
@Component
class OutboxRelay(
    private val outboxRepository: OutboxRepository,
    private val kafkaTemplate: KafkaTemplate<String, String>,
    private val objectMapper: com.fasterxml.jackson.databind.ObjectMapper
) {
    
    @Scheduled(fixedDelay = 1000)  // every 1 second
    @Transactional
    fun processOutbox() {
        val events = outboxRepository.findByStatus(OutboxStatus.PENDING, limit = 100)
        
        events.forEach { event ->
            try {
                outboxRepository.updateStatus(event.id, OutboxStatus.PROCESSING)
                
                val topic = "${event.aggregateType.lowercase()}.${event.eventType.camelToSnake()}"
                kafkaTemplate.send(topic, event.aggregateId, event.payload).get()
                
                outboxRepository.updateStatus(event.id, OutboxStatus.SENT, java.time.Instant.now())
                
            } catch (e: Exception) {
                val newRetryCount = event.retryCount + 1
                if (newRetryCount >= 3) {
                    outboxRepository.updateStatus(event.id, OutboxStatus.FAILED)
                } else {
                    outboxRepository.incrementRetry(event.id, newRetryCount)
                }
            }
        }
    }
}

// Debezium-based CDC (Change Data Capture) - alternative to polling
// Debezium reads PostgreSQL WAL (Write-Ahead Log) directly
// config in docker-compose.yml:
// connector config: {
//   "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
//   "database.server.name": "mydb",
//   "database.include.list": "public",
//   "table.include.list": "public.outbox_events",
//   "transforms": "outbox",
//   "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter"
// }

fun String.camelToSnake(): String =
    replace(Regex("([A-Z])")) { "_${it.value.lowercase()}" }.removePrefix("_")

data class PlaceOrderCmd(val customerId: String, val items: List<Any>, val totalAmount: Double)
data class OrderRecord(val id: String, val customerId: String, val status: String, val totalAmount: Double)
data class OrderPlacedPayload(val orderId: String, val customerId: String, val items: List<Any>, val totalAmount: Double)

interface OrderDbRepository {
    fun save(order: OrderRecord): OrderRecord
}
interface OutboxRepository {
    fun save(event: OutboxEvent): OutboxEvent
    fun findByStatus(status: OutboxStatus, limit: Int): List<OutboxEvent>
    fun updateStatus(id: String, status: OutboxStatus, processedAt: java.time.Instant? = null)
    fun incrementRetry(id: String, retryCount: Int)
}
```

---

## Distributed Tracing

```kotlin
// Spring Boot + Micrometer Tracing (OpenTelemetry)
dependencies {
    implementation("io.micrometer:micrometer-tracing-bridge-otel")
    implementation("io.opentelemetry.instrumentation:opentelemetry-spring-boot-starter:2.9.0")
    implementation("io.opentelemetry:opentelemetry-exporter-otlp")
}

// application.yml
// management.tracing.sampling.probability: 1.0  # 100% sampling in dev
// otel.exporter.otlp.endpoint: http://otel-collector:4318
// otel.service.name: order-service

@RestController
class TracedOrderController(
    private val orderService: OrderService2,
    private val tracer: io.micrometer.tracing.Tracer
) {
    
    @GetMapping("/orders/{id}")
    fun getOrder(@PathVariable id: String): OrderDto {
        // Auto-instrumented: Spring creates a span for each request
        // Span name: GET /orders/{id}
        
        // Manual span for specific operations
        val span = tracer.nextSpan()
            .name("fetch-order-details")
            .tag("order.id", id)
        
        return tracer.withSpan(span.start()).use {
            orderService.findById(id)
                ?.also { span.tag("order.status", it.status) }
                ?: throw NoSuchElementException("Order $id not found")
        }
    }
}

// Propagate trace context through Kafka
@Component
class TracedKafkaProducer(
    private val kafkaTemplate: KafkaTemplate<String, String>,
    private val tracer: io.micrometer.tracing.Tracer,
    private val propagator: io.micrometer.tracing.propagation.Propagator
) {
    
    fun sendWithTrace(topic: String, key: String, value: String) {
        val span = tracer.currentSpan()
        val headers = org.apache.kafka.common.header.internals.RecordHeaders()
        
        // Inject trace context into Kafka headers
        span?.let {
            propagator.inject(it.context(), headers) { carrier, name, headerValue ->
                carrier?.add(name, headerValue.toByteArray())
            }
        }
        
        kafkaTemplate.send(
            org.apache.kafka.clients.producer.ProducerRecord(topic, null, key, value, headers)
        )
    }
}

// Extract trace context from Kafka consumer
@Component
class TracedKafkaConsumer(
    private val tracer: io.micrometer.tracing.Tracer,
    private val propagator: io.micrometer.tracing.propagation.Propagator
) {
    
    @KafkaListener(topics = ["orders"])
    fun consume(record: org.apache.kafka.clients.consumer.ConsumerRecord<String, String>) {
        val extractedContext = propagator.extract(record.headers()) { carrier, name ->
            carrier?.headers(name)?.firstOrNull()?.value()?.decodeToString()
        }
        
        val span = tracer.nextSpan(extractedContext)
            .name("process-order-event")
            .tag("kafka.topic", record.topic())
            .tag("kafka.partition", record.partition().toString())
        
        tracer.withSpan(span.start()).use {
            processOrder(record.value())
        }
    }
    
    private fun processOrder(payload: String) {
        println("Processing: $payload")
    }
}

interface OrderService2 {
    fun findById(id: String): OrderDto?
}
data class OrderDto(val id: String, val status: String, val totalAmount: Double)
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement complete Saga with state machine

enum class OrderSagaStep {
    INITIAL, INVENTORY_PENDING, INVENTORY_CONFIRMED,
    PAYMENT_PENDING, PAYMENT_CONFIRMED, ORDER_CONFIRMED,
    COMPENSATING, COMPENSATED, FAILED
}

data class OrderSagaContext(
    val sagaId: String,
    val orderId: String,
    val customerId: String,
    val items: List<Any>,
    val totalAmount: Double,
    val currentStep: OrderSagaStep = OrderSagaStep.INITIAL,
    val reservationId: String? = null,
    val paymentId: String? = null,
    val failureReason: String? = null,
    val compensationLog: List<String> = emptyList()
)

class OrderSagaStateMachine {
    
    fun transition(context: OrderSagaContext, event: SagaEvent): OrderSagaContext {
        // TODO: Implement state transitions
        // INITIAL -> (OrderCreated) -> INVENTORY_PENDING
        // INVENTORY_PENDING -> (InventoryReserved) -> PAYMENT_PENDING
        // INVENTORY_PENDING -> (InventoryFailed) -> COMPENSATING
        // PAYMENT_PENDING -> (PaymentCompleted) -> ORDER_CONFIRMED
        // PAYMENT_PENDING -> (PaymentFailed) -> COMPENSATING
        // COMPENSATING -> (InventoryReleased) -> COMPENSATED
        TODO("Implement state machine transitions")
    }
    
    fun getNextActions(context: OrderSagaContext): List<String> {
        // Return list of actions to execute for current state
        TODO("Return actions for current state")
    }
}
```

---

## สรุป Part 57

```
✅ Saga Pattern: distributed transaction without 2PC
✅ Choreography Saga: event-driven, loosely coupled
✅ Orchestration Saga: central coordinator, easier to track
✅ Compensating transactions: rollback distributed state
✅ withCompensation: clean rollback pattern
✅ Outbox Pattern: atomic write to DB + message queue
✅ Outbox Relay: polling-based event publishing
✅ Debezium CDC: WAL-based change data capture
✅ Distributed Tracing: Micrometer + OpenTelemetry
✅ Trace propagation: across HTTP and Kafka
✅ Manual spans: instrument specific operations
✅ Span tags: attach business context to traces
✅ Saga State Machine: formal state transitions
✅ Idempotency: handle duplicate messages safely
```

---

*Part 57/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
