# Part 73: Advanced Apache Kafka

## สารบัญ
1. [Kafka Architecture Review](#kafka-architecture-review)
2. [Kafka Streams API](#kafka-streams-api)
3. [Schema Registry และ Avro](#schema-registry-และ-avro)
4. [Kafka Transactions](#kafka-transactions)
5. [Consumer Group Management](#consumer-group-management)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Kafka Architecture Review

```
Kafka Concepts:
- Topic: named stream of records
- Partition: ordered, immutable sequence of records
- Offset: unique ID per message per partition
- Producer: writes messages to topics
- Consumer: reads messages from topics
- Consumer Group: consumers sharing work (each partition → one consumer)
- Broker: Kafka server
- Replication: copies of partitions across brokers

Delivery Semantics:
- At-most-once: may lose messages (auto-commit before process)
- At-least-once: may duplicate messages (commit after process)
- Exactly-once: no loss, no duplicates (transactions needed)

Important Configurations:
Producer:
  acks=all (wait for all replicas)
  retries=3
  enable.idempotence=true (exactly-once)
  
Consumer:
  auto.offset.reset=earliest/latest
  enable.auto.commit=false (manual commit)
  max.poll.records=500
```

---

## Kafka Streams API

```kotlin
// Kafka Streams: stateful stream processing ใน JVM

dependencies {
    implementation("org.apache.kafka:kafka-streams:3.6.0")
    implementation("io.confluent:kafka-streams-avro-serde:7.5.0")
}

// Real-time order analytics with Kafka Streams
@Configuration
class OrderAnalyticsStreams(
    @Value("\${kafka.bootstrap-servers}") private val bootstrapServers: String,
    @Value("\${schema-registry.url}") private val schemaRegistryUrl: String
) {
    
    @Bean
    fun streamsConfig(): Properties {
        return Properties().apply {
            put(StreamsConfig.APPLICATION_ID_CONFIG, "order-analytics")
            put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers)
            put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().javaClass)
            put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().javaClass)
            put(StreamsConfig.PROCESSING_GUARANTEE_CONFIG, StreamsConfig.EXACTLY_ONCE_V2)
            put(StreamsConfig.COMMIT_INTERVAL_MS_CONFIG, 1000)
            put(StreamsConfig.REPLICATION_FACTOR_CONFIG, 3)
        }
    }
    
    @Bean
    fun orderAnalyticsTopology(config: Properties): KafkaStreams {
        val builder = StreamsBuilder()
        
        // Input topic: raw order events
        val orders: KStream<String, String> = builder.stream("orders.placed")
        
        // Parse JSON orders
        val parsedOrders = orders.mapValues { orderJson ->
            objectMapper.readValue(orderJson, OrderEvent::class.java)
        }
        
        // 1. Revenue by category (5-minute tumbling window)
        parsedOrders
            .flatMap { _, order ->
                order.items.map { item ->
                    KeyValue(item.categoryId, item.price * item.quantity)
                }
            }
            .groupByKey(Grouped.with(Serdes.String(), Serdes.Double()))
            .windowedBy(TimeWindows.ofSizeWithNoGrace(java.time.Duration.ofMinutes(5)))
            .aggregate(
                { 0.0 },
                { _, revenue, aggregate -> aggregate + revenue },
                Materialized.`as`<String, Double, WindowStore<Bytes, ByteArray>>("revenue-by-category-store")
                    .withValueSerde(Serdes.Double())
            )
            .toStream()
            .map { windowedKey, total ->
                KeyValue(
                    "${windowedKey.key()}:${windowedKey.window().start()}",
                    """{"category":"${windowedKey.key()}","total":$total}"""
                )
            }
            .to("analytics.revenue-by-category")
        
        // 2. Order count per user (session window)
        parsedOrders
            .groupBy { _, order -> order.userId }
            .windowedBy(SessionWindows.ofInactivityGapWithNoGrace(java.time.Duration.ofMinutes(30)))
            .count(Materialized.`as`("order-count-store"))
            .toStream()
            .map { windowedKey, count ->
                KeyValue(windowedKey.key(), count.toString())
            }
            .to("analytics.order-count-per-session")
        
        // 3. Join orders with product details (table join)
        val productsTable: GlobalKTable<String, String> = builder.globalTable("products.snapshot")
        
        val enrichedOrders = parsedOrders.join(
            productsTable,
            { _, order -> order.items.firstOrNull()?.productId ?: "" },
            { order, productJson ->
                val product = objectMapper.readValue(productJson, ProductSnapshot::class.java)
                """{"orderId":"${order.orderId}","productName":"${product.name}","total":${order.total}}"""
            }
        )
        
        enrichedOrders.to("orders.enriched")
        
        // 4. Detect high-value orders (filter + branch)
        val (highValue, normal) = parsedOrders
            .branch(
                Predicate { _, order -> order.total >= 10000.0 },
                Predicate { _, _ -> true }
            )
        
        highValue.to("orders.high-value")
        normal.to("orders.normal")
        
        val topology = builder.build()
        return KafkaStreams(topology, config).also { it.start() }
    }
    
    private val objectMapper = com.fasterxml.jackson.databind.ObjectMapper()
        .apply { findAndRegisterModules() }
}

data class OrderEvent(
    val orderId: String,
    val userId: String,
    val items: List<OrderItem>,
    val total: Double,
    val timestamp: Long = System.currentTimeMillis()
)

data class OrderItem(
    val productId: String,
    val categoryId: String,
    val quantity: Int,
    val price: Double
)

data class ProductSnapshot(val id: String, val name: String, val price: Double)

// Interactive Queries: query Kafka Streams state stores
@RestController
@RequestMapping("/api/analytics")
class AnalyticsController(private val kafkaStreams: KafkaStreams) {
    
    @GetMapping("/revenue/{category}")
    fun getRevenueByCategory(@PathVariable category: String): Map<String, Any> {
        val storeQueryParams = QueryableStoreTypes.windowStore<String, Double>()
        val store = kafkaStreams.store(
            StoreQueryParameters.fromNameAndType("revenue-by-category-store", storeQueryParams)
        )
        
        val now = java.time.Instant.now()
        val from = now.minus(java.time.Duration.ofHours(1))
        
        val results = mutableListOf<Map<String, Any>>()
        store.fetch(category, from, now).forEach { windowedResult ->
            results.add(mapOf(
                "windowStart" to windowedResult.key.window().start(),
                "revenue" to (windowedResult.value ?: 0.0)
            ))
        }
        
        return mapOf("category" to category, "windows" to results)
    }
}

typealias Configuration = org.springframework.context.annotation.Configuration
typealias Bean = org.springframework.context.annotation.Bean
typealias Value = org.springframework.beans.factory.annotation.Value
typealias RestController = org.springframework.web.bind.annotation.RestController
typealias RequestMapping = org.springframework.web.bind.annotation.RequestMapping
typealias GetMapping = org.springframework.web.bind.annotation.GetMapping
typealias PathVariable = org.springframework.web.bind.annotation.PathVariable

// Kafka Streams type aliases
typealias StreamsConfig = org.apache.kafka.streams.StreamsConfig
typealias StreamsBuilder = org.apache.kafka.streams.StreamsBuilder
typealias KafkaStreams = org.apache.kafka.streams.KafkaStreams
typealias KStream = org.apache.kafka.streams.kstream.KStream
typealias GlobalKTable = org.apache.kafka.streams.kstream.GlobalKTable
typealias KeyValue = org.apache.kafka.streams.KeyValue
typealias Grouped = org.apache.kafka.streams.kstream.Grouped
typealias Materialized = org.apache.kafka.streams.kstream.Materialized
typealias TimeWindows = org.apache.kafka.streams.kstream.TimeWindows
typealias SessionWindows = org.apache.kafka.streams.kstream.SessionWindows
typealias Predicate = org.apache.kafka.streams.kstream.Predicate
typealias Serdes = org.apache.kafka.common.serialization.Serdes
typealias WindowStore = org.apache.kafka.streams.state.WindowStore
typealias Bytes = org.apache.kafka.common.utils.Bytes
typealias StoreQueryParameters = org.apache.kafka.streams.StoreQueryParameters
typealias QueryableStoreTypes = org.apache.kafka.streams.state.QueryableStoreTypes
```

---

## Schema Registry และ Avro

```kotlin
// Avro Schema สำหรับ type-safe serialization

// order_placed.avsc
val orderPlacedSchema = """
{
  "type": "record",
  "name": "OrderPlacedEvent",
  "namespace": "com.example.events",
  "fields": [
    {"name": "order_id", "type": "string"},
    {"name": "user_id", "type": "string"},
    {"name": "total_amount", "type": {"type": "bytes", "logicalType": "decimal", "precision": 10, "scale": 2}},
    {"name": "currency", "type": "string", "default": "THB"},
    {"name": "items", "type": {"type": "array", "items": {
      "type": "record",
      "name": "OrderItem",
      "fields": [
        {"name": "product_id", "type": "string"},
        {"name": "quantity", "type": "int"},
        {"name": "unit_price", "type": "double"}
      ]
    }}},
    {"name": "created_at", "type": {"type": "long", "logicalType": "timestamp-millis"}}
  ]
}
""".trimIndent()

// Kafka config with Schema Registry
@Configuration
class KafkaAvroConfig(
    @Value("\${kafka.bootstrap-servers}") private val bootstrapServers: String,
    @Value("\${schema-registry.url}") private val schemaRegistryUrl: String
) {
    
    @Bean
    fun producerFactory(): ProducerFactory<String, GenericRecord> {
        val configs = mapOf(
            ProducerConfig.BOOTSTRAP_SERVERS_CONFIG to bootstrapServers,
            ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG to StringSerializer::class.java,
            ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG to KafkaAvroSerializer::class.java,
            "schema.registry.url" to schemaRegistryUrl,
            ProducerConfig.ACKS_CONFIG to "all",
            ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG to "true",
            ProducerConfig.RETRIES_CONFIG to "3"
        )
        return DefaultKafkaProducerFactory(configs)
    }
    
    @Bean
    fun kafkaTemplate(producerFactory: ProducerFactory<String, GenericRecord>): KafkaTemplate<String, GenericRecord> {
        return KafkaTemplate(producerFactory)
    }
    
    @Bean
    fun consumerFactory(): ConsumerFactory<String, GenericRecord> {
        val configs = mapOf(
            ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG to bootstrapServers,
            ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG to StringDeserializer::class.java,
            ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG to KafkaAvroDeserializer::class.java,
            "schema.registry.url" to schemaRegistryUrl,
            "specific.avro.reader" to "true",
            ConsumerConfig.GROUP_ID_CONFIG to "order-service",
            ConsumerConfig.AUTO_OFFSET_RESET_CONFIG to "earliest",
            ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG to "false",
            ConsumerConfig.MAX_POLL_RECORDS_CONFIG to "500"
        )
        return DefaultKafkaConsumerFactory(configs)
    }
}

// Publisher with Avro
@Service
class OrderEventPublisher(
    private val kafkaTemplate: KafkaTemplate<String, Any>,
    private val schemaRegistryClient: SchemaRegistryClient
) {
    
    fun publishOrderPlaced(order: Order) {
        val schema = schemaRegistryClient.getLatestSchemaMetadata("orders.placed-value").schema
        val avroSchema = Schema.Parser().parse(schema)
        
        val record = GenericData.Record(avroSchema).apply {
            put("order_id", order.id)
            put("user_id", order.userId)
            put("total_amount", order.total.toLong())
            put("currency", "THB")
            put("items", order.items.map { item ->
                GenericData.Record(avroSchema.getField("items").schema().elementType).apply {
                    put("product_id", item.productId)
                    put("quantity", item.quantity)
                    put("unit_price", item.unitPrice)
                }
            })
            put("created_at", System.currentTimeMillis())
        }
        
        kafkaTemplate.send("orders.placed", order.id, record)
    }
}

data class Order(val id: String, val userId: String, val items: List<LineItem>, val total: Double)
data class LineItem(val productId: String, val quantity: Int, val unitPrice: Double)

typealias ProducerConfig = org.apache.kafka.clients.producer.ProducerConfig
typealias ConsumerConfig = org.apache.kafka.clients.consumer.ConsumerConfig
typealias StringSerializer = org.apache.kafka.common.serialization.StringSerializer
typealias StringDeserializer = org.apache.kafka.common.serialization.StringDeserializer
typealias KafkaAvroSerializer = io.confluent.kafka.serializers.KafkaAvroSerializer
typealias KafkaAvroDeserializer = io.confluent.kafka.serializers.KafkaAvroDeserializer
typealias GenericRecord = org.apache.avro.generic.GenericRecord
typealias GenericData = org.apache.avro.generic.GenericData
typealias Schema = org.apache.avro.Schema
typealias SchemaRegistryClient = io.confluent.kafka.schemaregistry.client.SchemaRegistryClient
typealias ProducerFactory = org.springframework.kafka.core.ProducerFactory
typealias ConsumerFactory = org.springframework.kafka.core.ConsumerFactory
typealias DefaultKafkaProducerFactory = org.springframework.kafka.core.DefaultKafkaProducerFactory
typealias DefaultKafkaConsumerFactory = org.springframework.kafka.core.DefaultKafkaConsumerFactory
typealias KafkaTemplate = org.springframework.kafka.core.KafkaTemplate
typealias Service = org.springframework.stereotype.Service
```

---

## Kafka Transactions

```kotlin
// Exactly-once semantics with Kafka transactions

@Configuration
class KafkaTransactionConfig {
    
    @Bean
    fun transactionalProducerFactory(): ProducerFactory<String, String> {
        val factory = DefaultKafkaProducerFactory<String, String>(mapOf(
            ProducerConfig.BOOTSTRAP_SERVERS_CONFIG to "localhost:9092",
            ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG to StringSerializer::class.java,
            ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG to StringSerializer::class.java,
            ProducerConfig.TRANSACTIONAL_ID_CONFIG to "order-service-tx",
            ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG to "true",
            ProducerConfig.ACKS_CONFIG to "all"
        ))
        
        factory.setTransactionIdPrefix("order-service-tx-")
        return factory
    }
    
    @Bean
    fun transactionalKafkaTemplate(
        factory: ProducerFactory<String, String>
    ): KafkaTemplate<String, String> {
        return KafkaTemplate(factory)
    }
    
    @Bean
    fun kafkaTransactionManager(
        factory: ProducerFactory<String, String>
    ): KafkaTransactionManager<String, String> {
        return KafkaTransactionManager(factory)
    }
}

// Transactional message processing
@Service
class TransactionalOrderProcessor(
    private val kafkaTemplate: KafkaTemplate<String, String>,
    private val orderRepository: OrderRepository
) {
    
    // Process and publish atomically - either both succeed or both fail
    @Transactional("kafkaTransactionManager")
    fun processOrderWithExactlyOnce(orderEvent: String) {
        val order = parseOrder(orderEvent)
        
        // Save to DB within same transaction
        orderRepository.save(order)
        
        // Publish result - within same Kafka transaction
        kafkaTemplate.send("orders.processed", order.id, buildProcessedEvent(order))
        kafkaTemplate.send("inventory.reserve", order.id, buildInventoryEvent(order))
        
        // If any failure above, ALL are rolled back (no partial publish)
    }
    
    // Consume-Process-Produce pattern
    @KafkaListener(
        topics = ["orders.placed"],
        groupId = "order-processor",
        containerFactory = "exactlyOnceContainerFactory"
    )
    fun handleOrderPlaced(record: ConsumerRecord<String, String>) {
        kafkaTemplate.executeInTransaction { ops ->
            val order = parseOrder(record.value())
            val result = processOrder(order)
            
            // Publish result within transaction
            ops.send("orders.completed", order.id, result)
            
            // Manual offset commit within transaction
            ops.sendOffsetsToTransaction(
                mapOf(
                    TopicPartition(record.topic(), record.partition()) to
                    OffsetAndMetadata(record.offset() + 1)
                ),
                "order-processor"
            )
        }
    }
    
    private fun parseOrder(json: String): ProcessedOrder = ProcessedOrder("order-1", "user-1")
    private fun processOrder(order: ProcessedOrder): String = "processed"
    private fun buildProcessedEvent(order: ProcessedOrder): String = "processed"
    private fun buildInventoryEvent(order: ProcessedOrder): String = "reserve"
}

data class ProcessedOrder(val id: String, val userId: String)

interface OrderRepository {
    fun save(order: ProcessedOrder): ProcessedOrder
}

typealias KafkaTransactionManager = org.springframework.kafka.transaction.KafkaTransactionManager
typealias KafkaListener = org.springframework.kafka.annotation.KafkaListener
typealias ConsumerRecord = org.apache.kafka.clients.consumer.ConsumerRecord
typealias TopicPartition = org.apache.kafka.common.TopicPartition
typealias OffsetAndMetadata = org.apache.kafka.clients.consumer.OffsetAndMetadata
typealias Transactional = org.springframework.transaction.annotation.Transactional
```

---

## Consumer Group Management

```kotlin
// Dead Letter Queue (DLQ) สำหรับ failed messages

@Component
class KafkaErrorHandler(
    private val kafkaTemplate: KafkaTemplate<String, String>
) : CommonErrorHandler {
    
    override fun handleRecord(
        thrownException: Exception,
        record: ConsumerRecord<*, *>,
        consumer: Consumer<*, *>,
        container: MessageListenerContainer
    ) {
        val retryCount = getRetryCount(record)
        
        if (retryCount >= 3) {
            // Send to Dead Letter Queue after 3 retries
            sendToDlq(record, thrownException)
        } else {
            // Retry with exponential backoff
            val backoffMs = Math.pow(2.0, retryCount.toDouble()).toLong() * 1000
            Thread.sleep(backoffMs)
            throw thrownException  // Re-throw to retry
        }
    }
    
    private fun getRetryCount(record: ConsumerRecord<*, *>): Int {
        return record.headers()
            .lastHeader("retry-count")
            ?.value()
            ?.let { String(it).toIntOrNull() }
            ?: 0
    }
    
    private fun sendToDlq(record: ConsumerRecord<*, *>, error: Exception) {
        val dlqTopic = "${record.topic()}.dlq"
        
        val message = MessageBuilder
            .withPayload(record.value()?.toString() ?: "")
            .setHeader("original-topic", record.topic())
            .setHeader("original-partition", record.partition())
            .setHeader("original-offset", record.offset())
            .setHeader("error-message", error.message)
            .setHeader("error-class", error.javaClass.name)
            .setHeader("failed-at", java.time.Instant.now().toString())
            .build()
        
        kafkaTemplate.send(dlqTopic, record.key()?.toString(), message.payload.toString())
    }
}

typealias Consumer = org.apache.kafka.clients.consumer.Consumer
typealias CommonErrorHandler = org.springframework.kafka.listener.CommonErrorHandler
typealias MessageListenerContainer = org.springframework.kafka.listener.MessageListenerContainer
typealias MessageBuilder = org.springframework.messaging.support.MessageBuilder
typealias Component = org.springframework.stereotype.Component

// Consumer with retry and DLQ
@Configuration
class KafkaConsumerConfig {
    
    @Bean
    fun retryTopicConfig(): RetryTopicConfiguration {
        return RetryTopicConfigurationBuilder
            .newInstance()
            .maxAttempts(4)
            .fixedBackOff(1000)
            .exponentialBackoff(1000, 2.0, 10000)
            .retryOn(listOf(
                RuntimeException::class.java,
                IllegalStateException::class.java
            ))
            .notRetryOn(listOf(
                IllegalArgumentException::class.java,  // Bad data, don't retry
                jakarta.validation.ValidationException::class.java
            ))
            .includeTopic("orders.placed")
            .autoCreateTopics(true, 1, 1)
            .build()
    }
}

// Reprocess messages from DLQ
@Service
class DlqReprocessService(
    private val kafkaTemplate: KafkaTemplate<String, String>,
    private val kafkaConsumer: org.apache.kafka.clients.consumer.KafkaConsumer<String, String>
) {
    
    fun reprocessDlqMessages(dlqTopic: String, originalTopic: String): Int {
        var count = 0
        
        kafkaConsumer.subscribe(listOf(dlqTopic))
        kafkaConsumer.poll(java.time.Duration.ofSeconds(5)).forEach { record ->
            // Re-publish to original topic
            kafkaTemplate.send(originalTopic, record.key(), record.value())
            count++
        }
        
        kafkaConsumer.commitSync()
        kafkaConsumer.unsubscribe()
        
        return count
    }
}

typealias RetryTopicConfiguration = org.springframework.kafka.retrytopic.RetryTopicConfiguration
typealias RetryTopicConfigurationBuilder = org.springframework.kafka.retrytopic.RetryTopicConfigurationBuilder
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement inventory reservation saga with Kafka

// Saga pattern: ลำดับ events ที่ต้องทำทั้งหมด หรือ compensate ทั้งหมด

sealed class SagaStep
data class ReserveInventory(val orderId: String, val items: List<OrderItem>) : SagaStep()
data class ProcessPayment(val orderId: String, val amount: Double) : SagaStep()
data class ConfirmShipment(val orderId: String, val address: String) : SagaStep()

sealed class CompensatingStep
data class ReleaseInventory(val orderId: String) : CompensatingStep()
data class RefundPayment(val orderId: String) : CompensatingStep()
data class CancelShipment(val orderId: String) : CompensatingStep()

@Service
class OrderSagaOrchestrator(
    private val kafkaTemplate: KafkaTemplate<String, String>
) {
    
    // Start saga
    fun startOrderSaga(orderId: String, items: List<OrderItem>, amount: Double) {
        kafkaTemplate.send(
            "saga.inventory.reserve",
            orderId,
            """{"orderId":"$orderId","step":"RESERVE_INVENTORY"}"""
        )
    }
    
    // Handle inventory reserved → proceed to payment
    @KafkaListener(topics = ["saga.inventory.reserved"])
    fun onInventoryReserved(event: String) {
        val orderId = parseOrderId(event)
        kafkaTemplate.send("saga.payment.process", orderId, event)
    }
    
    // Handle inventory failed → saga complete (no compensation needed yet)
    @KafkaListener(topics = ["saga.inventory.failed"])
    fun onInventoryFailed(event: String) {
        val orderId = parseOrderId(event)
        markOrderFailed(orderId, "Inventory unavailable")
    }
    
    // Handle payment successful → proceed to shipment
    @KafkaListener(topics = ["saga.payment.processed"])
    fun onPaymentProcessed(event: String) {
        val orderId = parseOrderId(event)
        kafkaTemplate.send("saga.shipment.confirm", orderId, event)
    }
    
    // Handle payment failed → compensate (release inventory)
    @KafkaListener(topics = ["saga.payment.failed"])
    fun onPaymentFailed(event: String) {
        val orderId = parseOrderId(event)
        // Compensating transaction
        kafkaTemplate.send("saga.inventory.release", orderId, event)
        markOrderFailed(orderId, "Payment failed")
    }
    
    // Handle shipment confirmed → saga complete
    @KafkaListener(topics = ["saga.shipment.confirmed"])
    fun onShipmentConfirmed(event: String) {
        val orderId = parseOrderId(event)
        markOrderCompleted(orderId)
    }
    
    private fun parseOrderId(event: String): String = "order-1"  // TODO: parse JSON
    private fun markOrderFailed(orderId: String, reason: String) { /* TODO */ }
    private fun markOrderCompleted(orderId: String) { /* TODO */ }
}
```

---

## สรุป Part 73

```
✅ Kafka Streams: stateful stream processing
✅ StreamsBuilder: define topology declaratively
✅ KStream: continuous record stream
✅ TimeWindows: tumbling 5-minute window
✅ SessionWindows: inactivity gap for user sessions
✅ GlobalKTable: enrichment join with reference data
✅ branch(): split stream by predicate
✅ Interactive Queries: query state stores via REST
✅ Schema Registry: centralized schema management
✅ Avro: binary serialization with schema evolution
✅ KafkaAvroSerializer: serialize with schema registry
✅ GenericData.Record: build Avro records programmatically
✅ Kafka Transactions: exactly-once semantics
✅ TRANSACTIONAL_ID_CONFIG: producer transaction ID
✅ executeInTransaction: consume-process-produce atomically
✅ sendOffsetsToTransaction: commit offset within tx
✅ Dead Letter Queue: send failed messages to DLQ
✅ CommonErrorHandler: retry with exponential backoff
✅ RetryTopicConfigurationBuilder: retry topic DSL
✅ Saga Orchestrator: coordinate distributed transactions
```

---

*Part 73/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
