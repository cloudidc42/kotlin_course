# Part 95: Domain-Driven Design (DDD) ขั้นสูง

## สารบัญ
1. [Strategic DDD: Bounded Contexts](#strategic-ddd-bounded-contexts)
2. [Context Map & Integration Patterns](#context-map--integration-patterns)
3. [Aggregate Design ขั้นสูง](#aggregate-design-ขั้นสูง)
4. [Domain Events & Event Sourcing](#domain-events--event-sourcing)
5. [CQRS Pattern](#cqrs-pattern)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Strategic DDD: Bounded Contexts

```
DDD กับ E-Commerce:

Bounded Contexts (BC):
┌─────────────────────────────────────────────────────────────┐
│  Catalog BC          │  Order BC         │  Shipping BC     │
│  - Product           │  - Order          │  - Shipment      │
│  - Category          │  - OrderItem      │  - TrackingEvent │
│  - ProductReview     │  - Payment        │  - Carrier       │
│  - Inventory         │  - Discount       │                  │
├─────────────────────────────────────────────────────────────┤
│  Customer BC         │  Marketing BC     │  Finance BC      │
│  - Customer          │  - Campaign       │  - Invoice       │
│  - Address           │  - Voucher        │  - Transaction   │
│  - Loyalty           │  - Recommendation │  - Report        │
└─────────────────────────────────────────────────────────────┘

Ubiquitous Language (ภาษากลาง):
Order BC: "Customer" = who places orders, has OrderHistory
Shipping BC: "Customer" = delivery recipient, has Addresses
Marketing BC: "Customer" = marketing target, has Segments

→ Same word "Customer", different meaning in each BC
→ แต่ละ BC มี model ของตัวเองที่เหมาะกับ context

Aggregate Design Rules (Evans):
1. Model true invariants in consistency boundaries
2. Design small aggregates
3. Reference other aggregates by identity only
4. Update other aggregates using eventual consistency
```

---

## Aggregate Design ขั้นสูง

```kotlin
// Order Aggregate - comprehensive example
package com.ecommerce.order.domain

// Value Objects
data class OrderId(val value: String) {
    companion object {
        fun generate(): OrderId = OrderId(java.util.UUID.randomUUID().toString())
        fun of(value: String): OrderId {
            require(value.isNotBlank()) { "OrderId cannot be blank" }
            return OrderId(value)
        }
    }
}

data class CustomerId(val value: String) {
    companion object { fun of(v: String) = CustomerId(v) }
}

data class ProductId(val value: String) {
    companion object { fun of(v: String) = ProductId(v) }
}

data class Money(val amount: java.math.BigDecimal, val currency: String) : Comparable<Money> {
    init {
        require(amount >= java.math.BigDecimal.ZERO) { "Amount cannot be negative" }
        require(currency.length == 3) { "Currency must be 3-letter ISO code" }
    }
    
    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Cannot add different currencies" }
        return Money(amount + other.amount, currency)
    }
    
    operator fun times(quantity: Int): Money = Money(amount * quantity.toBigDecimal(), currency)
    
    fun applyDiscount(discountPercent: Int): Money {
        require(discountPercent in 0..100)
        return Money(amount * (100 - discountPercent).toBigDecimal() / 100.toBigDecimal(), currency)
    }
    
    override fun compareTo(other: Money): Int = amount.compareTo(other.amount)
    
    companion object {
        val ZERO_THB = Money(java.math.BigDecimal.ZERO, "THB")
    }
}

data class Address(
    val street: String,
    val city: String,
    val province: String,
    val postalCode: String,
    val country: String = "TH"
) {
    init {
        require(postalCode.matches(Regex("\\d{5}"))) { "Thai postal code must be 5 digits" }
    }
}

// Order Item - part of Order aggregate
data class OrderItem(
    val productId: ProductId,
    val productName: String,  // denormalized - snapshot at order time
    val quantity: Int,
    val unitPrice: Money
) {
    init {
        require(quantity > 0) { "Quantity must be positive" }
    }
    
    val subtotal: Money get() = unitPrice * quantity
}

// Domain Events
sealed class OrderEvent {
    abstract val orderId: OrderId
    abstract val occurredAt: java.time.Instant
}

data class OrderCreated(
    override val orderId: OrderId,
    override val occurredAt: java.time.Instant = java.time.Instant.now(),
    val customerId: CustomerId,
    val items: List<OrderItem>,
    val shippingAddress: Address
) : OrderEvent()

data class OrderConfirmed(
    override val orderId: OrderId,
    override val occurredAt: java.time.Instant = java.time.Instant.now(),
    val confirmedBy: String
) : OrderEvent()

data class OrderItemAdded(
    override val orderId: OrderId,
    override val occurredAt: java.time.Instant = java.time.Instant.now(),
    val item: OrderItem
) : OrderEvent()

data class OrderCancelled(
    override val orderId: OrderId,
    override val occurredAt: java.time.Instant = java.time.Instant.now(),
    val reason: String,
    val cancelledBy: CustomerId
) : OrderEvent()

data class PaymentReceived(
    override val orderId: OrderId,
    override val occurredAt: java.time.Instant = java.time.Instant.now(),
    val paymentId: String,
    val amount: Money
) : OrderEvent()

// Order Status
enum class OrderStatus {
    DRAFT, PENDING_PAYMENT, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED, REFUNDED;
    
    fun canTransitionTo(next: OrderStatus): Boolean = when (this) {
        DRAFT -> next in setOf(PENDING_PAYMENT, CANCELLED)
        PENDING_PAYMENT -> next in setOf(CONFIRMED, CANCELLED)
        CONFIRMED -> next in setOf(PROCESSING, CANCELLED)
        PROCESSING -> next in setOf(SHIPPED, CANCELLED)
        SHIPPED -> next in setOf(DELIVERED)
        DELIVERED -> next in setOf(REFUNDED)
        CANCELLED, REFUNDED -> false
    }
}

// Order Aggregate Root
class Order private constructor(
    val id: OrderId,
    val customerId: CustomerId,
    private var _status: OrderStatus,
    private val _items: MutableList<OrderItem>,
    val shippingAddress: Address,
    val createdAt: java.time.Instant,
    private var _version: Long = 0  // optimistic locking
) {
    val status: OrderStatus get() = _status
    val items: List<OrderItem> get() = _items.toList()
    val version: Long get() = _version
    
    private val _events: MutableList<OrderEvent> = mutableListOf()
    val events: List<OrderEvent> get() = _events.toList()
    fun clearEvents() = _events.clear()
    
    val totalAmount: Money get() = items.fold(Money.ZERO_THB) { acc, item -> acc + item.subtotal }
    val itemCount: Int get() = items.sumOf { it.quantity }
    
    // Business rule: minimum order amount
    companion object {
        private val MINIMUM_ORDER_AMOUNT = Money(java.math.BigDecimal("100"), "THB")
        
        fun create(
            customerId: CustomerId,
            items: List<OrderItem>,
            shippingAddress: Address
        ): Order {
            require(items.isNotEmpty()) { "Order must have at least one item" }
            
            val order = Order(
                id = OrderId.generate(),
                customerId = customerId,
                _status = OrderStatus.DRAFT,
                _items = items.toMutableList(),
                shippingAddress = shippingAddress,
                createdAt = java.time.Instant.now()
            )
            
            if (order.totalAmount < MINIMUM_ORDER_AMOUNT) {
                throw OrderDomainException("Minimum order amount is ${MINIMUM_ORDER_AMOUNT.amount} THB")
            }
            
            order._events.add(
                OrderCreated(order.id, customerId = customerId, items = items, shippingAddress = shippingAddress)
            )
            
            return order
        }
        
        // Reconstitute from persistence
        fun reconstitute(
            id: OrderId, customerId: CustomerId, status: OrderStatus,
            items: List<OrderItem>, shippingAddress: Address,
            createdAt: java.time.Instant, version: Long
        ): Order = Order(id, customerId, status, items.toMutableList(), shippingAddress, createdAt, version)
    }
    
    fun confirm(confirmedBy: String) {
        transitionTo(OrderStatus.CONFIRMED)
        _events.add(OrderConfirmed(id, confirmedBy = confirmedBy))
    }
    
    fun cancel(reason: String, by: CustomerId) {
        if (_status == OrderStatus.SHIPPED || _status == OrderStatus.DELIVERED) {
            throw OrderDomainException("Cannot cancel shipped or delivered order")
        }
        transitionTo(OrderStatus.CANCELLED)
        _events.add(OrderCancelled(id, reason = reason, cancelledBy = by))
    }
    
    fun recordPayment(paymentId: String, amount: Money) {
        require(amount == totalAmount) { 
            "Payment amount ${amount.amount} != order total ${totalAmount.amount}" 
        }
        transitionTo(OrderStatus.CONFIRMED)
        _events.add(PaymentReceived(id, paymentId = paymentId, amount = amount))
    }
    
    fun addItem(item: OrderItem) {
        if (_status != OrderStatus.DRAFT) {
            throw OrderDomainException("Can only add items to DRAFT orders")
        }
        
        // Merge with existing same product
        val existing = _items.find { it.productId == item.productId }
        if (existing != null) {
            _items.remove(existing)
            _items.add(existing.copy(quantity = existing.quantity + item.quantity))
        } else {
            _items.add(item)
        }
        
        _events.add(OrderItemAdded(id, item = item))
    }
    
    private fun transitionTo(next: OrderStatus) {
        if (!_status.canTransitionTo(next)) {
            throw OrderDomainException("Cannot transition from $_status to $next")
        }
        _status = next
        _version++
    }
}

class OrderDomainException(msg: String) : RuntimeException(msg)
```

---

## Event Sourcing

```kotlin
// Event Sourcing: state = series of events (not current state in DB)
// ทุก change เป็น event → replay events → reconstruct state

// Event Store interface
interface EventStore {
    fun appendEvents(aggregateId: String, events: List<OrderEvent>, expectedVersion: Long)
    fun loadEvents(aggregateId: String): List<OrderEvent>
    fun loadEventsFrom(aggregateId: String, fromVersion: Long): List<OrderEvent>
}

@org.springframework.stereotype.Repository
class PostgresEventStore(
    private val jdbcTemplate: org.springframework.jdbc.core.JdbcTemplate,
    private val objectMapper: com.fasterxml.jackson.databind.ObjectMapper
) : EventStore {
    
    override fun appendEvents(aggregateId: String, events: List<OrderEvent>, expectedVersion: Long) {
        if (events.isEmpty()) return
        
        // Check for concurrent modification (optimistic locking)
        val currentVersion = jdbcTemplate.queryForObject(
            "SELECT COALESCE(MAX(version), -1) FROM order_events WHERE aggregate_id = ?",
            Long::class.java,
            aggregateId
        ) ?: -1L
        
        if (currentVersion != expectedVersion - 1) {
            throw ConcurrentModificationException(
                "Optimistic locking conflict: expected version $expectedVersion, found ${currentVersion + 1}"
            )
        }
        
        events.forEachIndexed { i, event ->
            jdbcTemplate.update(
                """INSERT INTO order_events 
                   (aggregate_id, version, event_type, payload, occurred_at)
                   VALUES (?, ?, ?, ?::jsonb, ?)""",
                aggregateId,
                expectedVersion + i,
                event::class.simpleName,
                objectMapper.writeValueAsString(event),
                event.occurredAt
            )
        }
    }
    
    override fun loadEvents(aggregateId: String): List<OrderEvent> {
        return jdbcTemplate.query(
            "SELECT event_type, payload FROM order_events WHERE aggregate_id = ? ORDER BY version",
            { rs, _ -> deserializeEvent(rs.getString("event_type"), rs.getString("payload")) },
            aggregateId
        )
    }
    
    override fun loadEventsFrom(aggregateId: String, fromVersion: Long): List<OrderEvent> {
        return jdbcTemplate.query(
            "SELECT event_type, payload FROM order_events WHERE aggregate_id = ? AND version >= ? ORDER BY version",
            { rs, _ -> deserializeEvent(rs.getString("event_type"), rs.getString("payload")) },
            aggregateId, fromVersion
        )
    }
    
    private fun deserializeEvent(type: String, payload: String): OrderEvent {
        return when (type) {
            "OrderCreated" -> objectMapper.readValue(payload, OrderCreated::class.java)
            "OrderConfirmed" -> objectMapper.readValue(payload, OrderConfirmed::class.java)
            "OrderCancelled" -> objectMapper.readValue(payload, OrderCancelled::class.java)
            "PaymentReceived" -> objectMapper.readValue(payload, PaymentReceived::class.java)
            "OrderItemAdded" -> objectMapper.readValue(payload, OrderItemAdded::class.java)
            else -> throw IllegalStateException("Unknown event type: $type")
        }
    }
}

// Event-sourced Order Repository
@org.springframework.stereotype.Repository
class EventSourcedOrderRepository(private val eventStore: EventStore) {
    
    fun save(order: Order) {
        val events = order.events
        if (events.isEmpty()) return
        
        eventStore.appendEvents(order.id.value, events, order.version)
        order.clearEvents()
    }
    
    fun findById(id: OrderId): Order? {
        val events = eventStore.loadEvents(id.value)
        if (events.isEmpty()) return null
        
        return reconstitute(events)
    }
    
    private fun reconstitute(events: List<OrderEvent>): Order {
        require(events.isNotEmpty())
        
        val created = events.first() as? OrderCreated
            ?: throw IllegalStateException("First event must be OrderCreated")
        
        var order = Order.create(
            created.customerId, created.items, created.shippingAddress
        )
        
        // Replay all subsequent events
        events.drop(1).forEach { event ->
            order = applyEvent(order, event)
        }
        
        return order
    }
    
    private fun applyEvent(order: Order, event: OrderEvent): Order = when (event) {
        is OrderConfirmed -> order.apply { confirm(event.confirmedBy) }
        is OrderCancelled -> order.apply { cancel(event.reason, event.cancelledBy) }
        is PaymentReceived -> order.apply { recordPayment(event.paymentId, event.amount) }
        is OrderItemAdded -> order.apply { addItem(event.item) }
        is OrderCreated -> order  // already applied
    }
}
```

---

## CQRS Pattern

```kotlin
// CQRS: Command Query Responsibility Segregation
// Write side (Commands) → Domain Model
// Read side (Queries) → Optimized Read Models / Projections

// Commands
sealed class OrderCommand {
    abstract val orderId: OrderId
}

data class PlaceOrderCommand(
    override val orderId: OrderId = OrderId.generate(),
    val customerId: CustomerId,
    val items: List<OrderItemCommand>,
    val shippingAddress: Address
) : OrderCommand()

data class ConfirmOrderCommand(override val orderId: OrderId, val confirmedBy: String) : OrderCommand()
data class CancelOrderCommand(override val orderId: OrderId, val reason: String, val by: CustomerId) : OrderCommand()
data class AddItemCommand(override val orderId: OrderId, val item: OrderItem) : OrderCommand()

data class OrderItemCommand(val productId: String, val productName: String, val quantity: Int, val unitPrice: Money)

// Command Handler (write side)
@org.springframework.stereotype.Service
class OrderCommandHandler(
    private val orderRepository: EventSourcedOrderRepository,
    private val inventoryPort: InventoryPort,
    private val eventPublisher: DomainEventPublisher
) {
    
    fun handle(command: PlaceOrderCommand): OrderId {
        // Validate inventory
        command.items.forEach { item ->
            inventoryPort.reserve(item.productId, item.quantity)
        }
        
        val order = Order.create(
            customerId = command.customerId,
            items = command.items.map { item ->
                OrderItem(
                    productId = ProductId.of(item.productId),
                    productName = item.productName,
                    quantity = item.quantity,
                    unitPrice = item.unitPrice
                )
            },
            shippingAddress = command.shippingAddress
        )
        
        orderRepository.save(order)
        order.events.forEach { eventPublisher.publish(it) }
        
        return order.id
    }
    
    fun handle(command: ConfirmOrderCommand) {
        val order = orderRepository.findById(command.orderId)
            ?: throw OrderNotFoundException(command.orderId)
        
        order.confirm(command.confirmedBy)
        orderRepository.save(order)
        order.events.forEach { eventPublisher.publish(it) }
    }
    
    fun handle(command: CancelOrderCommand) {
        val order = orderRepository.findById(command.orderId)
            ?: throw OrderNotFoundException(command.orderId)
        
        order.cancel(command.reason, command.by)
        orderRepository.save(order)
        order.events.forEach { eventPublisher.publish(it) }
    }
}

// Read Models (optimized for queries)
data class OrderSummaryView(
    val orderId: String,
    val customerId: String,
    val status: String,
    val totalAmount: java.math.BigDecimal,
    val currency: String,
    val itemCount: Int,
    val createdAt: java.time.Instant
)

data class OrderDetailView(
    val orderId: String,
    val customerId: String,
    val customerName: String,
    val status: String,
    val items: List<OrderItemView>,
    val totalAmount: java.math.BigDecimal,
    val shippingAddress: AddressView,
    val timeline: List<OrderTimelineEntry>,
    val createdAt: java.time.Instant
)

data class OrderItemView(val productId: String, val productName: String, val quantity: Int, val unitPrice: java.math.BigDecimal, val subtotal: java.math.BigDecimal)
data class AddressView(val street: String, val city: String, val province: String, val postalCode: String)
data class OrderTimelineEntry(val status: String, val timestamp: java.time.Instant, val actor: String)

// Query Repository (read side - uses optimized SQL views)
@org.springframework.stereotype.Repository
class OrderQueryRepository(
    private val jdbcTemplate: org.springframework.jdbc.core.JdbcTemplate
) {
    
    fun findSummary(orderId: String): OrderSummaryView? {
        return jdbcTemplate.query(
            """SELECT o.id, o.customer_id, o.status, o.total_amount, o.currency, 
                      COUNT(oi.id) as item_count, o.created_at
               FROM order_summaries o
               LEFT JOIN order_items oi ON o.id = oi.order_id
               WHERE o.id = ?
               GROUP BY o.id, o.customer_id, o.status, o.total_amount, o.currency, o.created_at""",
            { rs, _ -> OrderSummaryView(
                orderId = rs.getString("id"),
                customerId = rs.getString("customer_id"),
                status = rs.getString("status"),
                totalAmount = rs.getBigDecimal("total_amount"),
                currency = rs.getString("currency"),
                itemCount = rs.getInt("item_count"),
                createdAt = rs.getTimestamp("created_at").toInstant()
            )},
            orderId
        ).firstOrNull()
    }
    
    fun findByCustomer(customerId: String, page: Int, size: Int): List<OrderSummaryView> {
        return jdbcTemplate.query(
            """SELECT * FROM order_summaries 
               WHERE customer_id = ?
               ORDER BY created_at DESC
               LIMIT ? OFFSET ?""",
            { rs, _ -> OrderSummaryView(
                orderId = rs.getString("id"),
                customerId = rs.getString("customer_id"),
                status = rs.getString("status"),
                totalAmount = rs.getBigDecimal("total_amount"),
                currency = rs.getString("currency"),
                itemCount = 0,
                createdAt = rs.getTimestamp("created_at").toInstant()
            )},
            customerId, size, page * size
        )
    }
}

// Projection: update read model from domain events
@org.springframework.stereotype.Component
class OrderProjection(
    private val jdbcTemplate: org.springframework.jdbc.core.JdbcTemplate
) {
    
    @org.springframework.context.event.EventListener
    fun on(event: OrderCreated) {
        jdbcTemplate.update(
            """INSERT INTO order_summaries (id, customer_id, status, total_amount, currency, created_at)
               VALUES (?, ?, 'DRAFT', ?, 'THB', ?)""",
            event.orderId.value,
            event.customerId.value,
            event.items.fold(java.math.BigDecimal.ZERO) { acc, item -> acc + item.subtotal.amount },
            event.occurredAt
        )
        
        event.items.forEach { item ->
            jdbcTemplate.update(
                """INSERT INTO order_items (order_id, product_id, product_name, quantity, unit_price)
                   VALUES (?, ?, ?, ?, ?)""",
                event.orderId.value,
                item.productId.value,
                item.productName,
                item.quantity,
                item.unitPrice.amount
            )
        }
    }
    
    @org.springframework.context.event.EventListener
    fun on(event: OrderConfirmed) {
        jdbcTemplate.update(
            "UPDATE order_summaries SET status = 'CONFIRMED' WHERE id = ?",
            event.orderId.value
        )
    }
    
    @org.springframework.context.event.EventListener
    fun on(event: OrderCancelled) {
        jdbcTemplate.update(
            "UPDATE order_summaries SET status = 'CANCELLED' WHERE id = ?",
            event.orderId.value
        )
    }
}

// Stubs
interface InventoryPort { fun reserve(productId: String, quantity: Int) }
interface DomainEventPublisher { fun publish(event: OrderEvent) }
class OrderNotFoundException(id: OrderId) : RuntimeException("Order not found: ${id.value}")
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement Loyalty Points Aggregate

// Business rules:
// 1. ลูกค้าได้รับ 1 point ต่อทุก 10 บาทที่ซื้อ
// 2. Points หมดอายุใน 1 ปีนับจากวันได้รับ
// 3. ใช้ points แทนเงินได้ (100 points = 10 บาท)
// 4. ยอด points ติดลบไม่ได้
// 5. Transaction history ต้องเก็บทุก operation

data class LoyaltyAccountId(val value: String)

sealed class LoyaltyEvent {
    abstract val accountId: LoyaltyAccountId
    abstract val occurredAt: java.time.Instant
}

data class PointsEarned(
    override val accountId: LoyaltyAccountId,
    override val occurredAt: java.time.Instant = java.time.Instant.now(),
    val points: Int,
    val sourceOrderId: String,
    val expiresAt: java.time.Instant = occurredAt.plus(java.time.Duration.ofDays(365))
) : LoyaltyEvent()

data class PointsRedeemed(
    override val accountId: LoyaltyAccountId,
    override val occurredAt: java.time.Instant = java.time.Instant.now(),
    val points: Int,
    val orderId: String,
    val discountAmount: Money
) : LoyaltyEvent()

data class PointsExpired(
    override val accountId: LoyaltyAccountId,
    override val occurredAt: java.time.Instant = java.time.Instant.now(),
    val expiredPoints: Int
) : LoyaltyEvent()

class LoyaltyAccount private constructor(
    val id: LoyaltyAccountId,
    val customerId: CustomerId,
    private val _transactions: MutableList<LoyaltyTransaction> = mutableListOf()
) {
    private val _events: MutableList<LoyaltyEvent> = mutableListOf()
    val events: List<LoyaltyEvent> get() = _events.toList()
    fun clearEvents() = _events.clear()
    
    // Calculate available points (not expired)
    val availablePoints: Int get() {
        val now = java.time.Instant.now()
        return _transactions
            .filter { it.expiresAt == null || it.expiresAt.isAfter(now) }
            .sumOf { it.points }
            .coerceAtLeast(0)
    }
    
    companion object {
        private const val POINTS_PER_BAHT = 10  // 1 point per 10 baht
        private const val POINTS_TO_BAHT = 100  // 100 points = 10 baht
        
        fun open(customerId: CustomerId): LoyaltyAccount {
            return LoyaltyAccount(
                id = LoyaltyAccountId(java.util.UUID.randomUUID().toString()),
                customerId = customerId
            )
        }
    }
    
    fun earnPoints(orderAmount: Money, orderId: String) {
        val pointsToEarn = (orderAmount.amount / POINTS_PER_BAHT.toBigDecimal()).toInt()
        if (pointsToEarn <= 0) return
        
        val expiresAt = java.time.Instant.now().plus(java.time.Duration.ofDays(365))
        
        _transactions.add(LoyaltyTransaction(pointsToEarn, "EARNED", orderId, expiresAt))
        _events.add(PointsEarned(id, points = pointsToEarn, sourceOrderId = orderId, expiresAt = expiresAt))
    }
    
    fun redeemPoints(points: Int, orderId: String): Money {
        if (points > availablePoints) {
            throw OrderDomainException("Insufficient points: available=$availablePoints, requested=$points")
        }
        
        val discountAmount = Money(
            (points / POINTS_TO_BAHT.toBigDecimal()).multiply(10.toBigDecimal()),
            "THB"
        )
        
        _transactions.add(LoyaltyTransaction(-points, "REDEEMED", orderId, null))
        _events.add(PointsRedeemed(id, points = points, orderId = orderId, discountAmount = discountAmount))
        
        return discountAmount
    }
    
    fun expirePoints(): Int {
        val now = java.time.Instant.now()
        val expired = _transactions
            .filter { it.expiresAt != null && it.expiresAt.isBefore(now) && it.points > 0 }
            .sumOf { it.points }
        
        if (expired > 0) {
            _transactions.add(LoyaltyTransaction(-expired, "EXPIRED", null, null))
            _events.add(PointsExpired(id, expiredPoints = expired))
        }
        
        return expired
    }
}

data class LoyaltyTransaction(
    val points: Int,
    val type: String,
    val referenceId: String?,
    val expiresAt: java.time.Instant?
)
```

---

## สรุป Part 95

```
✅ Strategic DDD: Bounded Contexts แยก domain
✅ Ubiquitous Language: shared vocabulary within BC
✅ Aggregate root: Order controls all Order entities
✅ Value Objects: OrderId, CustomerId, Money, Address
✅ Money invariants: amount >= 0, currency 3-letter ISO
✅ Money operators: plus, times, applyDiscount
✅ OrderStatus.canTransitionTo: valid state machine
✅ Domain events: OrderCreated, Confirmed, Cancelled, etc.
✅ _events + clearEvents: collect and dispatch
✅ private constructor + companion object.create(): validation factory
✅ Order.reconstitute(): restore from DB without events
✅ OrderDomainException: domain-level errors
✅ Event Sourcing: state = replay of events
✅ EventStore.appendEvents: optimistic locking via version
✅ deserializeEvent: type-safe event deserialization
✅ reconstitute/applyEvent: replay events to rebuild aggregate
✅ CQRS: Command (write) vs Query (read) separation
✅ Commands: PlaceOrderCommand, ConfirmOrderCommand, etc.
✅ CommandHandler: orchestrates domain + persistence
✅ Read Models: OrderSummaryView, OrderDetailView
✅ QueryRepository: optimized SQL for reads
✅ Projection: @EventListener updates read model
✅ Eventual consistency: projections may lag slightly
✅ LoyaltyAccount: points, expiry, earn/redeem
✅ availablePoints: filter expired transactions
✅ earnPoints: 1 point per 10 baht
✅ redeemPoints: validate sufficient balance
✅ expirePoints: mark expired + emit event
```

---

*Part 95/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
