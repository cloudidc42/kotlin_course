# Part 42: Message Queues ขั้นสูงด้วย Kafka และ RabbitMQ

## สารบัญ
1. [Kafka Producers ขั้นสูง](#kafka-producers-ขั้นสูง)
2. [Kafka Consumers ขั้นสูง](#kafka-consumers-ขั้นสูง)
3. [Kafka Streams](#kafka-streams)
4. [RabbitMQ](#rabbitmq)
5. [Event Sourcing Pattern](#event-sourcing-pattern)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Kafka Producers ขั้นสูง

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.kafka:spring-kafka")
    implementation("io.confluent:kafka-avro-serializer:7.6.0")
    implementation("org.apache.avro:avro:1.11.3")
}

// Kafka configuration
@Configuration
class KafkaProducerConfig {
    
    @Bean
    fun kafkaTemplate(producerFactory: ProducerFactory<String, Any>): KafkaTemplate<String, Any> {
        return KafkaTemplate(producerFactory).apply {
            setProducerListener(object : ProducerListener<String, Any> {
                override fun onSuccess(
                    producerRecord: ProducerRecord<String, Any>,
                    recordMetadata: RecordMetadata
                ) {
                    log.info("Sent to ${recordMetadata.topic()}:${recordMetadata.partition()} at offset ${recordMetadata.offset()}")
                }
                
                override fun onError(
                    producerRecord: ProducerRecord<String, Any>,
                    recordMetadata: RecordMetadata?,
                    exception: Exception
                ) {
                    log.error("Failed to send to ${producerRecord.topic()}", exception)
                }
            })
        }
    }
    
    @Bean
    fun producerFactory(): ProducerFactory<String, Any> {
        val configs = mapOf(
            ProducerConfig.BOOTSTRAP_SERVERS_CONFIG to "localhost:9092",
            ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG to StringSerializer::class.java,
            ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG to JsonSerializer::class.java,
            ProducerConfig.ACKS_CONFIG to "all",  // wait for all replicas
            ProducerConfig.RETRIES_CONFIG to 3,
            ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG to true,  // exactly-once
            ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION to 5,
            ProducerConfig.COMPRESSION_TYPE_CONFIG to "snappy",
            ProducerConfig.BATCH_SIZE_CONFIG to 32 * 1024,  // 32KB
            ProducerConfig.LINGER_MS_CONFIG to 5  // wait 5ms to batch more messages
        )
        return DefaultKafkaProducerFactory(configs)
    }
}

// Event types
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "eventType")
@JsonSubTypes(
    JsonSubTypes.Type(OrderCreatedEvent::class, name = "ORDER_CREATED"),
    JsonSubTypes.Type(OrderShippedEvent::class, name = "ORDER_SHIPPED"),
    JsonSubTypes.Type(OrderCancelledEvent::class, name = "ORDER_CANCELLED")
)
sealed class OrderDomainEvent {
    abstract val orderId: String
    abstract val timestamp: Long
}

data class OrderCreatedEvent(
    override val orderId: String,
    val customerId: String,
    val items: List<OrderItemData>,
    val total: Double,
    override val timestamp: Long = System.currentTimeMillis()
) : OrderDomainEvent()

data class OrderShippedEvent(
    override val orderId: String,
    val trackingNumber: String,
    val estimatedDelivery: String,
    override val timestamp: Long = System.currentTimeMillis()
) : OrderDomainEvent()

data class OrderCancelledEvent(
    override val orderId: String,
    val reason: String,
    val refundAmount: Double,
    override val timestamp: Long = System.currentTimeMillis()
) : OrderDomainEvent()

data class OrderItemData(val productId: String, val quantity: Int, val price: Double)

// Producer service
@Service
class OrderEventProducer(private val kafkaTemplate: KafkaTemplate<String, Any>) {
    
    companion object {
        const val ORDERS_TOPIC = "orders.events"
        const val ORDERS_COMMANDS_TOPIC = "orders.commands"
    }
    
    fun publishOrderCreated(event: OrderCreatedEvent): CompletableFuture<SendResult<String, Any>> {
        val record = ProducerRecord(
            ORDERS_TOPIC,
            null,          // partition (null = auto)
            event.timestamp,
            event.orderId, // key: orderId for same-partition ordering
            event as Any
        ).apply {
            // Custom headers
            headers().add("correlationId", java.util.UUID.randomUUID().toString().toByteArray())
            headers().add("source", "order-service".toByteArray())
        }
        
        return kafkaTemplate.send(record)
    }
    
    suspend fun publishOrderEvent(event: OrderDomainEvent) {
        kafkaTemplate.send(ORDERS_TOPIC, event.orderId, event).await()
    }
    
    // Transactional publish
    fun publishMultipleEvents(events: List<OrderDomainEvent>) {
        kafkaTemplate.executeInTransaction { template ->
            events.forEach { event ->
                template.send(ORDERS_TOPIC, event.orderId, event)
            }
        }
    }
}
```

---

## Kafka Consumers ขั้นสูง

```kotlin
@Configuration
class KafkaConsumerConfig {
    
    @Bean
    fun consumerFactory(): ConsumerFactory<String, Any> {
        val configs = mapOf(
            ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG to "localhost:9092",
            ConsumerConfig.GROUP_ID_CONFIG to "order-processor",
            ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG to StringDeserializer::class.java,
            ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG to JsonDeserializer::class.java,
            ConsumerConfig.AUTO_OFFSET_RESET_CONFIG to "earliest",
            ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG to false,  // manual commit
            ConsumerConfig.MAX_POLL_RECORDS_CONFIG to 50,
            JsonDeserializer.TRUSTED_PACKAGES to "com.example.*"
        )
        return DefaultKafkaConsumerFactory(configs)
    }
    
    @Bean
    fun kafkaListenerContainerFactory(
        consumerFactory: ConsumerFactory<String, Any>
    ): ConcurrentKafkaListenerContainerFactory<String, Any> {
        return ConcurrentKafkaListenerContainerFactory<String, Any>().apply {
            this.consumerFactory = consumerFactory
            containerProperties.ackMode = ContainerProperties.AckMode.MANUAL_IMMEDIATE
            setConcurrency(3)  // 3 consumer threads
            setCommonErrorHandler(DefaultErrorHandler(
                DeadLetterPublishingRecoverer(kafkaTemplate()),
                FixedBackOff(1000L, 3L)  // retry 3 times with 1s delay
            ))
            setRecordFilterStrategy { consumerRecord ->
                // Filter: skip test events in production
                consumerRecord.headers()
                    .lastHeader("source")
                    ?.value()
                    ?.let { String(it) } == "test"
            }
        }
    }
    
    @Bean
    fun kafkaTemplate(): KafkaTemplate<String, Any> = KafkaTemplate(DefaultKafkaProducerFactory(emptyMap()))
}

// Consumer
@Component
class OrderEventConsumer(
    private val orderService: OrderProcessingService,
    private val notificationService: NotificationService2
) {
    private val log = LoggerFactory.getLogger(javaClass)
    
    @KafkaListener(
        topics = ["orders.events"],
        groupId = "order-processor",
        containerFactory = "kafkaListenerContainerFactory"
    )
    fun consumeOrderEvents(
        record: ConsumerRecord<String, OrderDomainEvent>,
        acknowledgment: Acknowledgment
    ) {
        val correlationId = record.headers()
            .lastHeader("correlationId")?.value()?.let { String(it) } ?: "unknown"
        
        MDC.put("correlationId", correlationId)
        MDC.put("orderId", record.key())
        
        try {
            when (val event = record.value()) {
                is OrderCreatedEvent -> {
                    orderService.handleOrderCreated(event)
                    notificationService.notifyOrderCreated(event)
                }
                is OrderShippedEvent -> {
                    orderService.handleOrderShipped(event)
                    notificationService.notifyOrderShipped(event)
                }
                is OrderCancelledEvent -> {
                    orderService.handleOrderCancelled(event)
                    notificationService.notifyOrderCancelled(event)
                }
            }
            
            acknowledgment.acknowledge()  // manual commit
            log.info("Processed event: ${record.value()::class.simpleName}")
            
        } catch (e: Exception) {
            log.error("Failed to process event", e)
            // DefaultErrorHandler will handle retry and DLQ
            throw e
        } finally {
            MDC.clear()
        }
    }
    
    // Batch consumer
    @KafkaListener(
        topics = ["orders.events.batch"],
        groupId = "order-batch-processor",
        batch = "true"
    )
    fun consumeBatch(
        records: List<ConsumerRecord<String, OrderDomainEvent>>,
        acknowledgment: Acknowledgment
    ) {
        log.info("Processing batch of ${records.size} records")
        
        records.groupBy { it.value()::class }
            .forEach { (type, events) ->
                orderService.processBatch(type, events.map { it.value() })
            }
        
        acknowledgment.acknowledge()
    }
}
```

---

## Event Sourcing Pattern

```kotlin
// Event Store
@Entity
@Table(name = "event_store")
data class EventEntity(
    @Id val id: String = java.util.UUID.randomUUID().toString(),
    val aggregateId: String,
    val aggregateType: String,
    val eventType: String,
    @Column(columnDefinition = "TEXT") val eventData: String,
    val version: Long,
    val timestamp: Long = System.currentTimeMillis()
)

@Repository
interface EventStoreRepository : JpaRepository<EventEntity, String> {
    fun findByAggregateIdOrderByVersionAsc(aggregateId: String): List<EventEntity>
    fun findByAggregateIdAndVersionGreaterThan(aggregateId: String, version: Long): List<EventEntity>
    fun countByAggregateId(aggregateId: String): Long
}

@Service
class EventStore(
    private val repository: EventStoreRepository,
    private val objectMapper: ObjectMapper,
    private val kafkaTemplate: KafkaTemplate<String, Any>
) {
    fun save(aggregateId: String, aggregateType: String, events: List<DomainEvent>, expectedVersion: Long) {
        val currentVersion = repository.countByAggregateId(aggregateId)
        
        if (currentVersion != expectedVersion) {
            throw ConcurrencyException("Expected version $expectedVersion, but was $currentVersion")
        }
        
        events.forEachIndexed { index, event ->
            val entity = EventEntity(
                aggregateId = aggregateId,
                aggregateType = aggregateType,
                eventType = event::class.simpleName ?: "Unknown",
                eventData = objectMapper.writeValueAsString(event),
                version = expectedVersion + index + 1
            )
            repository.save(entity)
        }
        
        // Publish to Kafka for projections
        events.forEach { event ->
            kafkaTemplate.send("domain.events", aggregateId, event)
        }
    }
    
    fun load(aggregateId: String): List<DomainEvent> {
        return repository.findByAggregateIdOrderByVersionAsc(aggregateId)
            .mapNotNull { entity ->
                try {
                    val clazz = Class.forName("com.example.events.${entity.eventType}")
                    objectMapper.readValue(entity.eventData, clazz) as DomainEvent
                } catch (e: Exception) {
                    null
                }
            }
    }
}

// Aggregate with event sourcing
class OrderAggregate private constructor() {
    var id: String = ""
    var customerId: String = ""
    var status: String = "PENDING"
    var items: List<OrderItemData> = emptyList()
    var total: Double = 0.0
    private var version: Long = 0
    
    private val uncommittedEvents = mutableListOf<DomainEvent>()
    
    companion object {
        fun reconstitute(events: List<DomainEvent>): OrderAggregate {
            val order = OrderAggregate()
            events.forEach { event -> order.apply(event) }
            return order
        }
        
        fun create(customerId: String, items: List<OrderItemData>): OrderAggregate {
            val order = OrderAggregate()
            val event = OrderCreatedEvent(
                orderId = java.util.UUID.randomUUID().toString(),
                customerId = customerId,
                items = items,
                total = items.sumOf { it.price * it.quantity }
            )
            order.raiseEvent(event)
            return order
        }
    }
    
    fun ship(trackingNumber: String) {
        require(status == "CONFIRMED") { "Order must be confirmed before shipping" }
        raiseEvent(OrderShippedEvent(id, trackingNumber, "2024-12-31"))
    }
    
    fun cancel(reason: String) {
        require(status != "SHIPPED") { "Cannot cancel shipped order" }
        raiseEvent(OrderCancelledEvent(id, reason, total))
    }
    
    private fun raiseEvent(event: DomainEvent) {
        apply(event)
        uncommittedEvents.add(event)
    }
    
    private fun apply(event: DomainEvent) {
        when (event) {
            is OrderCreatedEvent -> {
                id = event.orderId
                customerId = event.customerId
                items = event.items
                total = event.total
                status = "PENDING"
            }
            is OrderShippedEvent -> {
                status = "SHIPPED"
            }
            is OrderCancelledEvent -> {
                status = "CANCELLED"
            }
        }
        version++
    }
    
    fun getUncommittedEvents() = uncommittedEvents.toList()
    fun getVersion() = version
    fun clearUncommittedEvents() = uncommittedEvents.clear()
}

interface DomainEvent
class ConcurrencyException(message: String) : Exception(message)
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Build an inventory management system using Kafka Streams

// Use Kafka Streams to:
// 1. Consume order events
// 2. Update inventory count in real-time
// 3. Produce low-stock alerts when quantity < threshold
// 4. Generate daily sales reports using windowed aggregation

// Hint: Use KTable for state store, KStream for event processing
/*
val streamsBuilder = StreamsBuilder()

val orderStream: KStream<String, OrderCreatedEvent> = 
    streamsBuilder.stream("orders.events")

val inventoryTable: KTable<String, InventoryItem> = 
    streamsBuilder.table("inventory.state")

// Join and process
orderStream
    .flatMapValues { event -> event.items }
    .join(inventoryTable) { orderItem, inventory ->
        inventory.copy(quantity = inventory.quantity - orderItem.quantity)
    }
    .filter { _, inventory -> inventory.quantity < inventory.lowStockThreshold }
    .to("inventory.low-stock-alerts")

// Windowed sales aggregation
orderStream
    .groupByKey()
    .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofDays(1)))
    .aggregate(
        { DailySales() },
        { _, event, sales -> sales.add(event) }
    )
    .toStream()
    .to("reports.daily-sales")
*/

data class InventoryItem(
    val productId: String,
    val quantity: Int,
    val lowStockThreshold: Int = 10
)

data class DailySales(
    val totalOrders: Int = 0,
    val totalRevenue: Double = 0.0
) {
    fun add(event: OrderCreatedEvent) = copy(
        totalOrders = totalOrders + 1,
        totalRevenue = totalRevenue + event.total
    )
}

interface OrderProcessingService {
    fun handleOrderCreated(event: OrderCreatedEvent)
    fun handleOrderShipped(event: OrderShippedEvent)
    fun handleOrderCancelled(event: OrderCancelledEvent)
    fun processBatch(type: kotlin.reflect.KClass<out OrderDomainEvent>, events: List<OrderDomainEvent>)
}

interface NotificationService2 {
    fun notifyOrderCreated(event: OrderCreatedEvent)
    fun notifyOrderShipped(event: OrderShippedEvent)
    fun notifyOrderCancelled(event: OrderCancelledEvent)
}

// Placeholder Kafka imports
typealias ProducerRecord<K, V> = org.apache.kafka.clients.producer.ProducerRecord<K, V>
typealias RecordMetadata = org.apache.kafka.clients.producer.RecordMetadata
typealias ConsumerRecord<K, V> = org.apache.kafka.clients.consumer.ConsumerRecord<K, V>
typealias SendResult<K, V> = org.springframework.kafka.support.SendResult<K, V>
```

---

## สรุป Part 42

```
✅ Kafka Producer: idempotent, compression, batching
✅ Kafka Consumer: manual commit, batch processing, DLQ
✅ Consumer Groups: horizontal scaling
✅ Headers: correlationId, source metadata
✅ Transactional: publish multiple events atomically
✅ Error Handler: retry + Dead Letter Queue
✅ Event Sourcing: store state as sequence of events
✅ Aggregate reconstitution: replay events to rebuild state
✅ Event Store: JPA + Kafka for persistence + streaming
✅ Kafka Streams: real-time stream processing
✅ KTable: materialized view of stream data
✅ Windowed aggregation: time-based analytics
```

---

*Part 42/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
