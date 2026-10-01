# Part 58: Event-Driven Architecture ขั้นสูง

## สารบัญ
1. [Event Storming](#event-storming)
2. [CQRS + Event Sourcing](#cqrs--event-sourcing)
3. [Projection และ Read Model](#projection-และ-read-model)
4. [Event Schema Evolution](#event-schema-evolution)
5. [Dead Letter Queue](#dead-letter-queue)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Event Storming

```
Event Storming เป็น workshop technique สำหรับ explore domain

สิ่งที่พบใน E-Commerce:
───────────────────────────────────────────────────────
Domain Events (สีส้ม):
  CustomerRegistered → OrderPlaced → PaymentProcessed
  InventoryReserved → OrderShipped → OrderDelivered
  PaymentFailed → OrderCancelled → InventoryReleased

Commands (สีน้ำเงิน):
  RegisterCustomer → PlaceOrder → ProcessPayment
  ReserveInventory → ShipOrder → CancelOrder

Aggregates (สีเหลือง):
  Customer → Order → Payment
  Inventory → Shipment

Policies (สีม่วง):
  "When OrderPlaced → ReserveInventory"
  "When PaymentFailed → CancelOrder"
  "When OrderCancelled → ReleaseInventory"

Read Models (สีเขียว):
  OrderHistory, InventoryStatus, PaymentReport

External Systems (สีชมพู):
  Payment Gateway, Shipping Provider, Email Service
```

---

## CQRS + Event Sourcing

```kotlin
// ===== Events =====
@kotlinx.serialization.Serializable
sealed class OrderEvent {
    abstract val orderId: String
    abstract val timestamp: Long
    abstract val version: Long
}

@kotlinx.serialization.Serializable
data class OrderPlaced(
    override val orderId: String,
    val customerId: String,
    val items: List<OrderItem3>,
    val totalAmount: Double,
    override val timestamp: Long = System.currentTimeMillis(),
    override val version: Long = 1L
) : OrderEvent()

@kotlinx.serialization.Serializable
data class OrderConfirmed(
    override val orderId: String,
    override val timestamp: Long = System.currentTimeMillis(),
    override val version: Long = 1L
) : OrderEvent()

@kotlinx.serialization.Serializable
data class OrderShipped(
    override val orderId: String,
    val trackingNumber: String,
    val carrier: String,
    override val timestamp: Long = System.currentTimeMillis(),
    override val version: Long = 1L
) : OrderEvent()

@kotlinx.serialization.Serializable
data class OrderCancelled(
    override val orderId: String,
    val reason: String,
    override val timestamp: Long = System.currentTimeMillis(),
    override val version: Long = 1L
) : OrderEvent()

@kotlinx.serialization.Serializable
data class OrderItem3(val productId: String, val quantity: Int, val unitPrice: Double)

// ===== Aggregate with Event Sourcing =====
class OrderAggregate2 private constructor() {
    
    var orderId: String = ""
        private set
    var customerId: String = ""
        private set
    var status: String = ""
        private set
    var items: List<OrderItem3> = emptyList()
        private set
    var totalAmount: Double = 0.0
        private set
    var trackingNumber: String? = null
        private set
    private var currentVersion: Long = 0L
    
    private val uncommittedEvents = mutableListOf<OrderEvent>()
    
    // Command methods
    fun place(customerId: String, items: List<OrderItem3>, totalAmount: Double) {
        check(status.isEmpty()) { "Order already placed" }
        applyChange(OrderPlaced(orderId, customerId, items, totalAmount))
    }
    
    fun confirm() {
        check(status == "PENDING") { "Can only confirm pending orders" }
        applyChange(OrderConfirmed(orderId))
    }
    
    fun ship(trackingNumber: String, carrier: String) {
        check(status == "CONFIRMED") { "Can only ship confirmed orders" }
        applyChange(OrderShipped(orderId, trackingNumber, carrier))
    }
    
    fun cancel(reason: String) {
        check(status in setOf("PENDING", "CONFIRMED")) { "Cannot cancel order in status $status" }
        applyChange(OrderCancelled(orderId, reason))
    }
    
    // Apply events to state (no side effects!)
    private fun applyChange(event: OrderEvent) {
        apply(event)
        uncommittedEvents.add(event)
    }
    
    private fun apply(event: OrderEvent) {
        currentVersion++
        when (event) {
            is OrderPlaced -> {
                this.orderId = event.orderId
                this.customerId = event.customerId
                this.items = event.items
                this.totalAmount = event.totalAmount
                this.status = "PENDING"
            }
            is OrderConfirmed -> {
                this.status = "CONFIRMED"
            }
            is OrderShipped -> {
                this.status = "SHIPPED"
                this.trackingNumber = event.trackingNumber
            }
            is OrderCancelled -> {
                this.status = "CANCELLED"
            }
        }
    }
    
    fun getUncommittedEvents(): List<OrderEvent> = uncommittedEvents.toList()
    fun markEventsAsCommitted() = uncommittedEvents.clear()
    fun getVersion() = currentVersion
    
    companion object {
        // Create from scratch
        fun create(orderId: String): OrderAggregate2 {
            return OrderAggregate2().also { it.orderId = orderId }
        }
        
        // Reconstitute from event history (replay)
        fun reconstitute(events: List<OrderEvent>): OrderAggregate2 {
            require(events.isNotEmpty()) { "Cannot reconstitute from empty events" }
            return OrderAggregate2().also { agg ->
                events.forEach { agg.apply(it) }
            }
        }
    }
}

// ===== Event Store =====
interface EventStore {
    suspend fun appendEvents(streamId: String, events: List<OrderEvent>, expectedVersion: Long)
    suspend fun loadEvents(streamId: String, fromVersion: Long = 0): List<OrderEvent>
    suspend fun loadEvents(streamId: String, fromVersion: Long, toVersion: Long): List<OrderEvent>
}

@Repository
class PostgresEventStore(
    private val jdbcTemplate: org.springframework.jdbc.core.JdbcTemplate,
    private val objectMapper: com.fasterxml.jackson.databind.ObjectMapper
) : EventStore {
    
    override suspend fun appendEvents(
        streamId: String,
        events: List<OrderEvent>,
        expectedVersion: Long
    ) {
        // Optimistic concurrency: check current version first
        val currentVersion = jdbcTemplate.queryForObject(
            "SELECT COALESCE(MAX(version), 0) FROM event_store WHERE stream_id = ?",
            Long::class.java, streamId
        ) ?: 0L
        
        if (currentVersion != expectedVersion) {
            throw ConcurrencyException(
                "Expected version $expectedVersion but found $currentVersion for stream $streamId"
            )
        }
        
        var version = expectedVersion
        events.forEach { event ->
            version++
            jdbcTemplate.update("""
                INSERT INTO event_store (id, stream_id, event_type, payload, version, timestamp)
                VALUES (?, ?, ?, ?::jsonb, ?, ?)
            """,
                java.util.UUID.randomUUID().toString(),
                streamId,
                event::class.simpleName,
                objectMapper.writeValueAsString(event),
                version,
                java.time.Instant.now()
            )
        }
    }
    
    override suspend fun loadEvents(streamId: String, fromVersion: Long): List<OrderEvent> {
        return jdbcTemplate.query("""
            SELECT event_type, payload FROM event_store
            WHERE stream_id = ? AND version >= ?
            ORDER BY version ASC
        """, { rs, _ ->
            val eventType = rs.getString("event_type")
            val payload = rs.getString("payload")
            deserializeEvent(eventType, payload)
        }, streamId, fromVersion)
    }
    
    override suspend fun loadEvents(streamId: String, fromVersion: Long, toVersion: Long): List<OrderEvent> {
        return jdbcTemplate.query("""
            SELECT event_type, payload FROM event_store
            WHERE stream_id = ? AND version BETWEEN ? AND ?
            ORDER BY version ASC
        """, { rs, _ ->
            deserializeEvent(rs.getString("event_type"), rs.getString("payload"))
        }, streamId, fromVersion, toVersion)
    }
    
    private fun deserializeEvent(eventType: String, payload: String): OrderEvent {
        return when (eventType) {
            "OrderPlaced" -> objectMapper.readValue(payload, OrderPlaced::class.java)
            "OrderConfirmed" -> objectMapper.readValue(payload, OrderConfirmed::class.java)
            "OrderShipped" -> objectMapper.readValue(payload, OrderShipped::class.java)
            "OrderCancelled" -> objectMapper.readValue(payload, OrderCancelled::class.java)
            else -> throw IllegalArgumentException("Unknown event type: $eventType")
        }
    }
}

class ConcurrencyException(message: String) : Exception(message)
```

---

## Projection และ Read Model

```kotlin
// Read Model: denormalized view optimized for queries
@Entity(tableName = "order_read_model")
data class OrderReadModel(
    @Id val orderId: String,
    val customerId: String,
    val status: String,
    val totalAmount: Double,
    val itemCount: Int,
    val trackingNumber: String?,
    val createdAt: Long,
    val updatedAt: Long
)

// Projection: updates read model from events
@Component
class OrderProjection(
    private val orderReadModelRepository: OrderReadModelRepository
) {
    
    @KafkaListener(topics = ["order.events"])
    fun on(event: OrderEvent) {
        when (event) {
            is OrderPlaced -> orderReadModelRepository.save(
                OrderReadModel(
                    orderId = event.orderId,
                    customerId = event.customerId,
                    status = "PENDING",
                    totalAmount = event.totalAmount,
                    itemCount = event.items.size,
                    trackingNumber = null,
                    createdAt = event.timestamp,
                    updatedAt = event.timestamp
                )
            )
            
            is OrderConfirmed -> orderReadModelRepository.updateStatus(
                event.orderId, "CONFIRMED", event.timestamp
            )
            
            is OrderShipped -> orderReadModelRepository.updateStatusAndTracking(
                event.orderId, "SHIPPED", event.trackingNumber, event.timestamp
            )
            
            is OrderCancelled -> orderReadModelRepository.updateStatus(
                event.orderId, "CANCELLED", event.timestamp
            )
        }
    }
    
    // Rebuild projection from scratch (useful after schema changes)
    suspend fun rebuild(eventStore: EventStore) {
        orderReadModelRepository.deleteAll()
        
        // Get all order IDs from event store
        val orderIds = eventStore.loadAllStreamIds("order-")
        
        orderIds.forEach { orderId ->
            val events = eventStore.loadEvents(orderId)
            events.forEach { on(it) }
        }
    }
}

// Statistics projection
@Component
class OrderStatisticsProjection(
    private val statisticsRepository: OrderStatisticsRepository
) {
    
    @EventListener
    fun on(event: OrderPlaced) {
        statisticsRepository.incrementOrderCount()
        statisticsRepository.addRevenue(event.totalAmount)
        event.items.forEach { item ->
            statisticsRepository.incrementProductSales(item.productId, item.quantity)
        }
    }
    
    @EventListener
    fun on(event: OrderCancelled) {
        statisticsRepository.decrementOrderCount()
    }
}

interface OrderReadModelRepository {
    fun save(model: OrderReadModel): OrderReadModel
    fun updateStatus(orderId: String, status: String, timestamp: Long)
    fun updateStatusAndTracking(orderId: String, status: String, tracking: String, timestamp: Long)
    fun deleteAll()
}

interface OrderStatisticsRepository {
    fun incrementOrderCount()
    fun decrementOrderCount()
    fun addRevenue(amount: Double)
    fun incrementProductSales(productId: String, quantity: Int)
}

interface EventStore2 : EventStore {
    suspend fun loadAllStreamIds(prefix: String): List<String>
}
```

---

## Event Schema Evolution

```kotlin
// ===== Schema Registry + Avro/Protobuf =====

// Problem: เมื่อ event schema เปลี่ยน consumers เก่าต้องยังทำงานได้

// Strategy 1: Backward Compatible (เพิ่ม optional fields)
// v1: OrderPlaced { orderId, customerId, items, totalAmount }
// v2: OrderPlaced { orderId, customerId, items, totalAmount, couponCode? }  ← OK

// Strategy 2: Forward Compatible (ลบ fields ที่ไม่จำเป็น)
// Consumer เก่า ignore fields ที่ไม่รู้จัก

// Strategy 3: Full Compatible (backward + forward)
// ทั้งสองทิศทาง

// Upcaster: แปลง old events เป็น new format
interface EventUpcaster<T : OrderEvent> {
    val fromVersion: Long
    val toVersion: Long
    fun upcast(event: T): OrderEvent
}

class OrderPlacedV1ToV2Upcaster : EventUpcaster<OrderPlaced> {
    override val fromVersion = 1L
    override val toVersion = 2L
    
    override fun upcast(event: OrderPlaced): OrderEvent {
        // Add default couponCode = null
        return event.copy()  // In real code, add new fields with defaults
    }
}

// Event store with upcasting
class UpcastingEventStore(
    private val delegate: EventStore,
    private val upcasters: List<EventUpcaster<*>>
) : EventStore by delegate {
    
    override suspend fun loadEvents(streamId: String, fromVersion: Long): List<OrderEvent> {
        return delegate.loadEvents(streamId, fromVersion)
            .map { event -> upcastEvent(event) }
    }
    
    @Suppress("UNCHECKED_CAST")
    private fun upcastEvent(event: OrderEvent): OrderEvent {
        var current = event
        var eventVersion = current.version
        
        upcasters
            .filter { it.fromVersion == eventVersion }
            .sortedBy { it.fromVersion }
            .forEach { upcaster ->
                current = (upcaster as EventUpcaster<OrderEvent>).upcast(current)
                eventVersion = current.version
            }
        
        return current
    }
}
```

---

## Dead Letter Queue

```kotlin
// Dead Letter Queue: จัดการ failed messages ที่ retry หมดแล้ว

@Component
class OrderEventConsumer(
    private val orderService: OrderCommandService2,
    private val dlqTemplate: KafkaTemplate<String, String>
) {
    
    @RetryableTopic(
        attempts = "3",
        backoff = Backoff(delay = 1000, multiplier = 2.0, maxDelay = 10000),
        dltStrategy = DltStrategy.FAIL_ON_ERROR
    )
    @KafkaListener(topics = ["order.commands"])
    fun processCommand(
        record: org.apache.kafka.clients.consumer.ConsumerRecord<String, String>,
        @Header(KafkaHeaders.RECEIVED_TOPIC) topic: String
    ) {
        try {
            val command = deserializeCommand(record.value())
            orderService.handle(command)
        } catch (e: BusinessException) {
            // Business errors: don't retry, send to DLQ with reason
            sendToDlq(record, "BUSINESS_ERROR", e.message)
            throw e  // Re-throw to mark as failed
        } catch (e: Exception) {
            // Technical errors: retry up to maxAttempts
            throw e
        }
    }
    
    @DltHandler
    fun processDlt(
        record: org.apache.kafka.clients.consumer.ConsumerRecord<String, String>,
        @Header(KafkaHeaders.DLT_EXCEPTION_MESSAGE) errorMessage: String
    ) {
        // Log to dead letter queue for manual investigation
        println("DLQ received: ${record.key()} - $errorMessage")
        
        // Store in database for later replay
        dlqRepository.save(DeadLetterEvent(
            originalTopic = record.topic().replace("-dlt", ""),
            key = record.key(),
            payload = record.value(),
            errorMessage = errorMessage,
            receivedAt = System.currentTimeMillis()
        ))
    }
    
    private fun sendToDlq(record: org.apache.kafka.clients.consumer.ConsumerRecord<String, String>,
                          reason: String, message: String?) {
        dlqTemplate.send(
            "${record.topic()}-dlq",
            record.key(),
            record.value()
        )
    }
    
    private fun deserializeCommand(value: String): Any = TODO("Deserialize")
}

// DLQ Replay tool: manually replay failed messages
@Component
class DlqReplayService(
    private val dlqRepository: DlqRepository,
    private val kafkaTemplate: KafkaTemplate<String, String>
) {
    
    suspend fun replayAll(topic: String) {
        val events = dlqRepository.findByOriginalTopic(topic)
        
        events.forEach { event ->
            kafkaTemplate.send(event.originalTopic, event.key, event.payload)
            dlqRepository.markAsReplayed(event.id)
        }
    }
    
    suspend fun replayById(id: String) {
        val event = dlqRepository.findById(id) ?: return
        kafkaTemplate.send(event.originalTopic, event.key, event.payload)
        dlqRepository.markAsReplayed(id)
    }
}

data class DeadLetterEvent(
    val id: String = java.util.UUID.randomUUID().toString(),
    val originalTopic: String,
    val key: String,
    val payload: String,
    val errorMessage: String,
    val receivedAt: Long,
    val replayed: Boolean = false
)

interface DlqRepository {
    fun save(event: DeadLetterEvent): DeadLetterEvent
    fun findByOriginalTopic(topic: String): List<DeadLetterEvent>
    fun findById(id: String): DeadLetterEvent?
    fun markAsReplayed(id: String)
}

interface OrderCommandService2 { fun handle(command: Any) }
class BusinessException(message: String) : Exception(message)
val dlqRepository: DlqRepository get() = TODO("inject")
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Complete Order History projection with snapshots

// Snapshot: เก็บ aggregate state ณ จุดหนึ่ง เพื่อ avoid replaying all events

data class OrderSnapshot(
    val orderId: String,
    val status: String,
    val totalAmount: Double,
    val items: List<OrderItem3>,
    val version: Long,
    val timestamp: Long
)

interface SnapshotStore {
    suspend fun save(snapshot: OrderSnapshot)
    suspend fun load(orderId: String): OrderSnapshot?
}

class SnapshottingOrderRepository(
    private val eventStore: EventStore,
    private val snapshotStore: SnapshotStore,
    private val snapshotInterval: Int = 10
) {
    
    suspend fun load(orderId: String): OrderAggregate2 {
        // 1. Try to load snapshot
        val snapshot = snapshotStore.load(orderId)
        
        // 2. Load events after snapshot
        val fromVersion = snapshot?.version ?: 0L
        val events = eventStore.loadEvents(orderId, fromVersion)
        
        // 3. Reconstitute
        return if (snapshot != null && events.isEmpty()) {
            // Use snapshot directly
            OrderAggregate2.create(orderId)  // simplified
        } else {
            OrderAggregate2.reconstitute(events)
        }
    }
    
    suspend fun save(aggregate: OrderAggregate2) {
        val events = aggregate.getUncommittedEvents()
        eventStore.appendEvents(aggregate.orderId, events, aggregate.getVersion() - events.size.toLong())
        aggregate.markEventsAsCommitted()
        
        // Take snapshot every N events
        if (aggregate.getVersion() % snapshotInterval == 0L) {
            snapshotStore.save(OrderSnapshot(
                orderId = aggregate.orderId,
                status = aggregate.status,
                totalAmount = aggregate.totalAmount,
                items = aggregate.items,
                version = aggregate.getVersion(),
                timestamp = System.currentTimeMillis()
            ))
        }
    }
}
```

---

## สรุป Part 58

```
✅ Event Storming: domain exploration workshop
✅ Domain Events: orange stickies, business facts
✅ CQRS: separate write model (commands) from read model
✅ Event Sourcing: store events, derive state by replay
✅ Aggregate with event sourcing: applyChange pattern
✅ Event Store: append-only log with optimistic concurrency
✅ ConcurrencyException: conflict detection
✅ Projection: update read model from events
✅ Rebuild projection: replay from event store
✅ Schema Evolution: backward/forward compatibility
✅ Upcaster: migrate old events to new schema
✅ Dead Letter Queue: handle failed messages
✅ @RetryableTopic: auto retry with backoff
✅ @DltHandler: process permanently failed messages
✅ DLQ Replay: recover from failures manually
✅ Snapshots: optimize reconstitution performance
```

---

*Part 58/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
