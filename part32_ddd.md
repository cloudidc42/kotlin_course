# Part 32: Domain Driven Design (DDD) กับ Kotlin

## สารบัญ
1. [DDD Concepts](#ddd-concepts)
2. [Aggregates และ Entities](#aggregates-และ-entities)
3. [Value Objects](#value-objects)
4. [Domain Events](#domain-events)
5. [Repositories และ Factories](#repositories-และ-factories)
6. [Domain Services](#domain-services)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## DDD Concepts

```
Domain Driven Design = วิธีออกแบบ software โดยเอา business domain เป็นศูนย์กลาง

Building Blocks:
1. Entity: มี identity (id), state เปลี่ยนได้
2. Value Object: ไม่มี identity, immutable, defined by attributes
3. Aggregate: กลุ่มของ Entities ที่มี Aggregate Root
4. Domain Event: สิ่งที่เกิดขึ้นใน domain ที่สำคัญ
5. Repository: abstraction สำหรับเก็บ Aggregates
6. Factory: สร้าง complex objects/aggregates
7. Domain Service: logic ที่ไม่เหมาะใส่ใน entity ใดๆ
8. Bounded Context: ขอบเขตของ domain model

Ubiquitous Language:
- ใช้ภาษาเดียวกับ domain experts
- ชื่อ class/method สะท้อน business terms
```

---

## Aggregates และ Entities

```kotlin
import java.time.Instant
import java.util.UUID

// Aggregate Root: Order
class Order private constructor(
    val id: OrderId,
    val customerId: CustomerId,
    private val _items: MutableList<OrderItem> = mutableListOf(),
    private var _status: OrderStatus = OrderStatus.DRAFT,
    private val _events: MutableList<DomainEvent> = mutableListOf(),
    val createdAt: Instant = Instant.now()
) {
    // Read-only views
    val items: List<OrderItem> get() = _items.toList()
    val status: OrderStatus get() = _status
    val events: List<DomainEvent> get() = _events.toList()
    
    // Computed properties
    val subtotal: Money get() = _items.fold(Money.ZERO) { acc, item -> acc + item.totalPrice }
    val tax: Money get() = subtotal * 0.07
    val total: Money get() = subtotal + tax
    val itemCount: Int get() = _items.sumOf { it.quantity }
    
    // Business methods - enforce invariants
    fun addItem(product: Product, quantity: Int): Order {
        check(_status == OrderStatus.DRAFT) { "Cannot add items to ${_status} order" }
        require(quantity > 0) { "Quantity must be positive" }
        
        val existing = _items.find { it.productId == product.id }
        if (existing != null) {
            _items[_items.indexOf(existing)] = existing.increaseQuantity(quantity)
        } else {
            _items.add(OrderItem.create(product, quantity))
        }
        
        return this
    }
    
    fun removeItem(productId: ProductId): Order {
        check(_status == OrderStatus.DRAFT) { "Cannot remove items from ${_status} order" }
        check(_items.removeIf { it.productId == productId }) { "Item not found" }
        return this
    }
    
    fun updateItemQuantity(productId: ProductId, quantity: Int): Order {
        check(_status == OrderStatus.DRAFT) { "Cannot update ${_status} order" }
        require(quantity > 0) { "Quantity must be positive" }
        
        val index = _items.indexOfFirst { it.productId == productId }
        check(index >= 0) { "Item not found" }
        _items[index] = _items[index].copy(quantity = quantity)
        
        return this
    }
    
    fun submit(): Order {
        check(_status == OrderStatus.DRAFT) { "Can only submit DRAFT orders" }
        check(_items.isNotEmpty()) { "Cannot submit empty order" }
        check(total > Money.ZERO) { "Order total must be positive" }
        
        _status = OrderStatus.SUBMITTED
        _events.add(OrderSubmittedEvent(id, customerId, total, Instant.now()))
        
        return this
    }
    
    fun confirm(paymentId: PaymentId): Order {
        check(_status == OrderStatus.SUBMITTED) { "Can only confirm SUBMITTED orders" }
        
        _status = OrderStatus.CONFIRMED
        _events.add(OrderConfirmedEvent(id, paymentId, Instant.now()))
        
        return this
    }
    
    fun ship(trackingNumber: TrackingNumber): Order {
        check(_status == OrderStatus.CONFIRMED) { "Can only ship CONFIRMED orders" }
        
        _status = OrderStatus.SHIPPED
        _events.add(OrderShippedEvent(id, trackingNumber, Instant.now()))
        
        return this
    }
    
    fun cancel(reason: String): Order {
        check(_status in listOf(OrderStatus.DRAFT, OrderStatus.SUBMITTED)) {
            "Cannot cancel ${_status} order"
        }
        
        _status = OrderStatus.CANCELLED
        _events.add(OrderCancelledEvent(id, reason, Instant.now()))
        
        return this
    }
    
    fun clearEvents(): Order {
        _events.clear()
        return this
    }
    
    companion object {
        fun create(customerId: CustomerId): Order {
            val order = Order(
                id = OrderId(UUID.randomUUID().toString()),
                customerId = customerId
            )
            order._events.add(OrderCreatedEvent(order.id, customerId, order.createdAt))
            return order
        }
    }
}

// Entity within aggregate
data class OrderItem private constructor(
    val id: OrderItemId,
    val productId: ProductId,
    val productName: String,
    val unitPrice: Money,
    val quantity: Int
) {
    val totalPrice: Money get() = unitPrice * quantity
    
    fun increaseQuantity(by: Int) = copy(quantity = quantity + by)
    
    companion object {
        fun create(product: Product, quantity: Int) = OrderItem(
            id = OrderItemId(UUID.randomUUID().toString()),
            productId = product.id,
            productName = product.name,
            unitPrice = product.price,
            quantity = quantity
        )
    }
}

enum class OrderStatus { DRAFT, SUBMITTED, CONFIRMED, SHIPPED, DELIVERED, CANCELLED }
```

---

## Value Objects

```kotlin
// Value Objects: immutable, no identity, equality by value

@JvmInline
value class OrderId(val value: String)

@JvmInline
value class OrderItemId(val value: String)

@JvmInline
value class CustomerId(val value: String)

@JvmInline
value class ProductId(val value: String)

@JvmInline
value class PaymentId(val value: String)

@JvmInline
value class TrackingNumber(val value: String)

// Complex Value Object
data class Money(val amount: java.math.BigDecimal, val currency: Currency) {
    init {
        require(amount >= java.math.BigDecimal.ZERO) { "Money amount cannot be negative" }
    }
    
    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Currency mismatch: $currency vs ${other.currency}" }
        return Money(amount + other.amount, currency)
    }
    
    operator fun minus(other: Money): Money {
        require(currency == other.currency) { "Currency mismatch" }
        require(amount >= other.amount) { "Insufficient amount" }
        return Money(amount - other.amount, currency)
    }
    
    operator fun times(factor: Double): Money {
        return Money(amount * java.math.BigDecimal(factor).setScale(2, java.math.RoundingMode.HALF_UP), currency)
    }
    
    operator fun times(factor: Int): Money = times(factor.toDouble())
    
    operator fun compareTo(other: Money): Int {
        require(currency == other.currency) { "Currency mismatch" }
        return amount.compareTo(other.amount)
    }
    
    override fun toString() = "${currency.symbol}${amount.toPlainString()}"
    
    companion object {
        val ZERO = Money(java.math.BigDecimal.ZERO, Currency.THB)
        fun of(amount: Double, currency: Currency = Currency.THB) =
            Money(java.math.BigDecimal(amount).setScale(2, java.math.RoundingMode.HALF_UP), currency)
    }
}

enum class Currency(val symbol: String) {
    THB("฿"), USD("$"), EUR("€"), JPY("¥")
}

data class Address(
    val street: String,
    val district: String,
    val city: String,
    val province: String,
    val postalCode: String,
    val country: String = "Thailand"
) {
    init {
        require(street.isNotBlank()) { "Street required" }
        require(postalCode.matches("\\d{5}".toRegex())) { "Invalid postal code" }
    }
    
    val formatted: String
        get() = "$street, $district, $city, $province $postalCode, $country"
}

data class PhoneNumber(val number: String) {
    init {
        require(number.matches("[0-9+\\-\\s()]{7,15}".toRegex())) {
            "Invalid phone number: $number"
        }
    }
    
    val normalized: String
        get() = number.replace("[\\s\\-()]".toRegex(), "")
}

// Product entity (simplified)
data class Product(
    val id: ProductId,
    val name: String,
    val price: Money,
    val stock: Int
)
```

---

## Domain Events

```kotlin
// Domain Events: เกิดขึ้นแล้ว ไม่สามารถเปลี่ยนแปลงได้
sealed class DomainEvent {
    abstract val occurredAt: Instant
}

data class OrderCreatedEvent(
    val orderId: OrderId,
    val customerId: CustomerId,
    override val occurredAt: Instant
) : DomainEvent()

data class OrderSubmittedEvent(
    val orderId: OrderId,
    val customerId: CustomerId,
    val total: Money,
    override val occurredAt: Instant
) : DomainEvent()

data class OrderConfirmedEvent(
    val orderId: OrderId,
    val paymentId: PaymentId,
    override val occurredAt: Instant
) : DomainEvent()

data class OrderShippedEvent(
    val orderId: OrderId,
    val trackingNumber: TrackingNumber,
    override val occurredAt: Instant
) : DomainEvent()

data class OrderCancelledEvent(
    val orderId: OrderId,
    val reason: String,
    override val occurredAt: Instant
) : DomainEvent()

// Event Handler
interface DomainEventHandler<T : DomainEvent> {
    suspend fun handle(event: T)
}

class OrderEventDispatcher {
    private val handlers = mutableMapOf<String, MutableList<DomainEventHandler<*>>>()
    
    @Suppress("UNCHECKED_CAST")
    fun <T : DomainEvent> on(eventClass: Class<T>, handler: DomainEventHandler<T>) {
        handlers.getOrPut(eventClass.simpleName) { mutableListOf() }.add(handler)
    }
    
    inline fun <reified T : DomainEvent> on(handler: DomainEventHandler<T>) {
        on(T::class.java, handler)
    }
    
    @Suppress("UNCHECKED_CAST")
    suspend fun dispatch(event: DomainEvent) {
        val eventHandlers = handlers[event::class.simpleName] as? List<DomainEventHandler<DomainEvent>>
        eventHandlers?.forEach { it.handle(event) }
    }
    
    suspend fun dispatchAll(events: List<DomainEvent>) {
        events.forEach { dispatch(it) }
    }
}

// Handlers
class SendOrderConfirmationEmail : DomainEventHandler<OrderSubmittedEvent> {
    override suspend fun handle(event: OrderSubmittedEvent) {
        println("📧 Sending confirmation email for order ${event.orderId.value}")
        println("   Total: ${event.total}")
    }
}

class DeductInventoryHandler(private val inventoryService: InventoryService) : DomainEventHandler<OrderConfirmedEvent> {
    override suspend fun handle(event: OrderConfirmedEvent) {
        println("📦 Deducting inventory for order ${event.orderId.value}")
    }
}

class SendShippingNotification : DomainEventHandler<OrderShippedEvent> {
    override suspend fun handle(event: OrderShippedEvent) {
        println("🚚 Order ${event.orderId.value} shipped. Tracking: ${event.trackingNumber.value}")
    }
}

interface InventoryService {
    suspend fun deduct(orderId: OrderId)
}
```

---

## Domain Services

```kotlin
// Domain Service: logic ที่ไม่ fit เข้าใน entity ใดๆ

class OrderPricingService {
    fun calculateDiscount(order: Order, customer: Customer): Money {
        val baseTotal = order.subtotal
        
        val discount = when {
            customer.tier == CustomerTier.VIP && baseTotal > Money.of(5000.0) ->
                baseTotal * 0.10  // 10% for VIP orders over 5000
            customer.tier == CustomerTier.PREMIUM && baseTotal > Money.of(1000.0) ->
                baseTotal * 0.05  // 5% for Premium orders over 1000
            customer.isFirstOrder ->
                baseTotal * 0.03  // 3% first order discount
            else -> Money.ZERO
        }
        
        return discount
    }
    
    fun applyPromoCode(order: Order, code: PromoCode): Money {
        return when {
            !code.isValid -> Money.ZERO
            code.minOrderAmount > Money.ZERO && order.subtotal < code.minOrderAmount -> Money.ZERO
            code.isPercentage -> order.subtotal * (code.discountValue / 100.0)
            else -> code.discountAmount ?: Money.ZERO
        }
    }
}

class OrderFulfillmentService(
    private val inventoryService: InventoryService,
    private val paymentService: PaymentService
) {
    suspend fun canFulfill(order: Order): FulfillmentCheck {
        val stockIssues = mutableListOf<String>()
        
        // Check each item
        // (simplified - would check real inventory)
        val canFulfill = stockIssues.isEmpty()
        
        return FulfillmentCheck(
            canFulfill = canFulfill,
            stockIssues = stockIssues
        )
    }
    
    suspend fun fulfill(order: Order, paymentDetails: PaymentDetails): FulfillmentResult {
        return try {
            val payment = paymentService.charge(paymentDetails, order.total)
            FulfillmentResult.Success(PaymentId(payment.id))
        } catch (e: Exception) {
            FulfillmentResult.PaymentFailed(e.message ?: "Unknown error")
        }
    }
}

data class FulfillmentCheck(
    val canFulfill: Boolean,
    val stockIssues: List<String>
)

sealed class FulfillmentResult {
    data class Success(val paymentId: PaymentId) : FulfillmentResult()
    data class PaymentFailed(val reason: String) : FulfillmentResult()
    data class InsufficientStock(val issues: List<String>) : FulfillmentResult()
}

data class Customer(
    val id: CustomerId,
    val name: String,
    val tier: CustomerTier,
    val isFirstOrder: Boolean = false
)

enum class CustomerTier { REGULAR, PREMIUM, VIP }

data class PromoCode(
    val code: String,
    val isValid: Boolean,
    val minOrderAmount: Money = Money.ZERO,
    val isPercentage: Boolean = false,
    val discountValue: Double = 0.0,
    val discountAmount: Money? = null
)

data class PaymentDetails(val method: String, val cardToken: String? = null)
interface PaymentService {
    suspend fun charge(details: PaymentDetails, amount: Money): PaymentRecord
}
data class PaymentRecord(val id: String, val amount: Money)
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง Complete Order Flow

suspend fun main() {
    // Setup
    val eventDispatcher = OrderEventDispatcher()
    eventDispatcher.on(SendOrderConfirmationEmail())
    eventDispatcher.on(SendShippingNotification())
    
    val pricingService = OrderPricingService()
    
    // Create customer
    val customerId = CustomerId("cust-001")
    val customer = Customer(
        id = customerId,
        tier = CustomerTier.PREMIUM,
        name = "สมชาย ใจดี"
    )
    
    // Create products
    val laptop = Product(ProductId("p-001"), "Laptop Pro", Money.of(25000.0), stock = 5)
    val mouse = Product(ProductId("p-002"), "Wireless Mouse", Money.of(800.0), stock = 20)
    val keyboard = Product(ProductId("p-003"), "Mech Keyboard", Money.of(2200.0), stock = 10)
    
    // Create order
    val order = Order.create(customerId)
    
    order.addItem(laptop, 1)
    order.addItem(mouse, 2)
    order.addItem(keyboard, 1)
    
    println("=== Order Summary ===")
    println("Order ID: ${order.id.value}")
    println("Customer: ${customer.name}")
    println()
    
    order.items.forEach { item ->
        println("  ${item.productName} x${item.quantity} @ ${item.unitPrice} = ${item.totalPrice}")
    }
    println()
    println("Subtotal: ${order.subtotal}")
    
    val discount = pricingService.calculateDiscount(order, customer)
    println("Discount: -$discount")
    println("Tax: ${order.tax}")
    println("Total: ${order.total}")
    
    // Submit order
    println("\n=== Submitting Order ===")
    order.submit()
    println("Status: ${order.status}")
    
    // Dispatch events
    eventDispatcher.dispatchAll(order.events)
    order.clearEvents()
    
    // Confirm payment
    println("\n=== Confirming Payment ===")
    val paymentId = PaymentId("pay-${System.currentTimeMillis()}")
    order.confirm(paymentId)
    println("Status: ${order.status}")
    
    // Ship order
    println("\n=== Shipping Order ===")
    order.ship(TrackingNumber("TH123456789"))
    println("Status: ${order.status}")
    
    eventDispatcher.dispatchAll(order.events)
    order.clearEvents()
    
    println("\n=== Final State ===")
    println("Order ${order.id.value}: ${order.status}")
    println("Total: ${order.total}")
}
```

---

## สรุป Part 32

```
✅ DDD: เอา business domain เป็นศูนย์กลางการออกแบบ
✅ Entity: มี identity, state เปลี่ยนได้ (ใช้ class)
✅ Value Object: ไม่มี identity, immutable (ใช้ data class / value class)
✅ Aggregate: กลุ่ม entities ที่มี Aggregate Root
✅ Aggregate Root: single entry point สำหรับ modifications
✅ Invariants: ตรวจสอบ business rules ใน domain methods
✅ Domain Events: สิ่งที่เกิดขึ้นใน domain, immutable
✅ Event Dispatcher: กระจาย events ไปยัง handlers
✅ Repository: abstraction สำหรับ persistence
✅ Domain Service: logic ที่ไม่ fit ใน entity
✅ Factory: สร้าง complex objects ด้วย business rules
✅ Ubiquitous Language: ใช้ศัพท์ business ใน code
```

---

*Part 32/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
