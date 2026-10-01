# Part 66: DDD — Bounded Contexts and Context Mapping

## สารบัญ
1. [Bounded Context คืออะไร](#bounded-context-คืออะไร)
2. [Strategic DDD Patterns](#strategic-ddd-patterns)
3. [Context Map ระหว่าง Contexts](#context-map-ระหว่าง-contexts)
4. [Anti-Corruption Layer](#anti-corruption-layer)
5. [Shared Kernel](#shared-kernel)
6. [Context Integration ด้วย Events](#context-integration-ด้วย-events)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Bounded Context คืออะไร

```
Bounded Context คือขอบเขตที่ Ubiquitous Language (ภาษาที่ทีมใช้ร่วมกัน) มีความหมายชัดเจน

ตัวอย่าง: ระบบ E-Commerce
┌─────────────────────────────────────────────────────────────────┐
│                        E-Commerce System                        │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   Catalog    │  │   Ordering   │  │      Inventory       │  │
│  │   Context    │  │   Context    │  │      Context         │  │
│  │              │  │              │  │                      │  │
│  │  Product     │  │  Product     │  │  StockItem           │  │
│  │  (has rich   │  │  (just id +  │  │  (tracks physical    │  │
│  │   details)   │  │   price)     │  │   location + qty)    │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │   Customer   │  │   Shipping   │  │      Payments        │  │
│  │   Context    │  │   Context    │  │      Context         │  │
│  │              │  │              │  │                      │  │
│  │  Customer    │  │  Consignment │  │  Transaction         │  │
│  │  (full CRM   │  │  (tracking   │  │  (financial          │  │
│  │   profile)   │  │   number)    │  │   record)            │  │
│  └──────────────┘  └──────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘

สังเกตว่า "Product" หมายความต่างกันใน Catalog vs Ordering!
```

---

## Catalog Context

```kotlin
// === CATALOG BOUNDED CONTEXT ===
// Product ใน Catalog มีรายละเอียดครบถ้วน

package com.example.catalog.domain

data class ProductId(val value: String)
data class CategoryId(val value: String)

data class ProductImage(
    val url: String,
    val altText: String,
    val isPrimary: Boolean
)

data class ProductAttribute(
    val name: String,
    val value: String
)

data class ProductPrice(
    val amount: java.math.BigDecimal,
    val currency: String,
    val taxIncluded: Boolean
)

// Rich Product in Catalog
class CatalogProduct private constructor(
    val id: ProductId,
    val sku: String,
    val name: String,
    val description: String,
    val longDescription: String,
    val images: List<ProductImage>,
    val attributes: List<ProductAttribute>,
    val price: ProductPrice,
    val categoryId: CategoryId,
    val tags: Set<String>,
    val status: CatalogProductStatus,
    val createdAt: java.time.Instant
) {
    companion object {
        fun create(
            sku: String,
            name: String,
            description: String,
            price: ProductPrice,
            categoryId: CategoryId
        ): CatalogProduct {
            return CatalogProduct(
                id = ProductId(java.util.UUID.randomUUID().toString()),
                sku = sku,
                name = name,
                description = description,
                longDescription = "",
                images = emptyList(),
                attributes = emptyList(),
                price = price,
                categoryId = categoryId,
                tags = emptySet(),
                status = CatalogProductStatus.DRAFT,
                createdAt = java.time.Instant.now()
            )
        }
    }
    
    fun publish(): CatalogProduct = copy(status = CatalogProductStatus.PUBLISHED)
    fun discontinue(): CatalogProduct = copy(status = CatalogProductStatus.DISCONTINUED)
    fun addImage(image: ProductImage): CatalogProduct = copy(images = images + image)
    fun addAttribute(attr: ProductAttribute): CatalogProduct = copy(attributes = attributes + attr)
    fun updatePrice(newPrice: ProductPrice): CatalogProduct = copy(price = newPrice)
}

enum class CatalogProductStatus { DRAFT, PUBLISHED, DISCONTINUED }

// Catalog Domain Events
data class ProductPublished(
    val productId: ProductId,
    val sku: String,
    val name: String,
    val price: ProductPrice,
    val occurredAt: java.time.Instant = java.time.Instant.now()
)

data class ProductPriceChanged(
    val productId: ProductId,
    val sku: String,
    val oldPrice: ProductPrice,
    val newPrice: ProductPrice,
    val occurredAt: java.time.Instant = java.time.Instant.now()
)
```

---

## Ordering Context

```kotlin
// === ORDERING BOUNDED CONTEXT ===
// Product ใน Ordering มีแค่ข้อมูลที่จำเป็นสำหรับการสั่งซื้อ

package com.example.ordering.domain

data class OrderProductId(val value: String)
data class OrderId(val value: String)
data class OrderCustomerId(val value: String)

// Lean Product representation ใน Ordering context
data class OrderProduct(
    val productId: OrderProductId,
    val sku: String,
    val name: String,          // ชื่อตอน order (snapshot)
    val unitPrice: java.math.BigDecimal,
    val currency: String
)

data class OrderLine(
    val product: OrderProduct,
    val quantity: Int,
    val lineTotal: java.math.BigDecimal = product.unitPrice * quantity.toBigDecimal()
)

class Order private constructor(
    val id: OrderId,
    val customerId: OrderCustomerId,
    private val lines: MutableList<OrderLine>,
    private var status: OrderStatus,
    val placedAt: java.time.Instant
) {
    val orderLines: List<OrderLine> get() = lines.toList()
    
    val totalAmount: java.math.BigDecimal
        get() = lines.sumOf { it.lineTotal }
    
    fun confirm(): Order {
        require(status == OrderStatus.PENDING) { "Can only confirm PENDING orders" }
        status = OrderStatus.CONFIRMED
        return this
    }
    
    fun cancel(reason: String): Order {
        require(status in listOf(OrderStatus.PENDING, OrderStatus.CONFIRMED)) {
            "Cannot cancel order in status $status"
        }
        status = OrderStatus.CANCELLED
        return this
    }
    
    companion object {
        fun place(customerId: OrderCustomerId, lines: List<OrderLine>): Order {
            require(lines.isNotEmpty()) { "Order must have at least one line" }
            return Order(
                id = OrderId(java.util.UUID.randomUUID().toString()),
                customerId = customerId,
                lines = lines.toMutableList(),
                status = OrderStatus.PENDING,
                placedAt = java.time.Instant.now()
            )
        }
    }
}

enum class OrderStatus { PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED }
```

---

## Anti-Corruption Layer

```kotlin
// ACL: translates between external context and our domain
// ป้องกันไม่ให้ concepts จาก external context "ปนเปื้อน" domain ของเรา

package com.example.ordering.adapter.acl

// External Catalog API response (external model — not our domain)
data class CatalogApiProduct(
    val product_id: String,
    val product_sku: String,
    val product_name: String,
    val product_price: Double,
    val currency_code: String,
    val is_active: Boolean,
    val stock_level: String  // "IN_STOCK", "LOW_STOCK", "OUT_OF_STOCK"
)

// Anti-Corruption Layer: translates Catalog model → Ordering model
@Component
class CatalogAntiCorruptionLayer(
    private val catalogApiClient: CatalogApiClient
) {
    
    // Translate external product to ordering domain concept
    fun getOrderProduct(productId: String): OrderProduct? {
        val catalogProduct = catalogApiClient.getProduct(productId) ?: return null
        
        if (!catalogProduct.is_active) return null
        if (catalogProduct.stock_level == "OUT_OF_STOCK") return null
        
        // Translate external model → domain model
        return OrderProduct(
            productId = OrderProductId(catalogProduct.product_id),
            sku = catalogProduct.product_sku,
            name = catalogProduct.product_name,
            unitPrice = catalogProduct.product_price.toBigDecimal(),
            currency = translateCurrency(catalogProduct.currency_code)
        )
    }
    
    fun getOrderProducts(productIds: List<String>): Map<String, OrderProduct> {
        return productIds
            .mapNotNull { id -> getOrderProduct(id)?.let { id to it } }
            .toMap()
    }
    
    // Map external currency codes to our domain's currency format
    private fun translateCurrency(externalCode: String): String {
        return when (externalCode) {
            "THB", "฿" -> "THB"
            "USD", "$" -> "USD"
            "EUR", "€" -> "EUR"
            else -> throw IllegalArgumentException("Unknown currency: $externalCode")
        }
    }
}

interface CatalogApiClient {
    fun getProduct(productId: String): CatalogApiProduct?
}

// HTTP implementation of CatalogApiClient
@Component
class HttpCatalogApiClient(private val webClient: Any) : CatalogApiClient {
    override fun getProduct(productId: String): CatalogApiProduct? {
        // Call catalog service REST API
        return null  // simplified
    }
}
```

---

## Context Integration ด้วย Events

```kotlin
// Event-based integration between bounded contexts

// === CATALOG publishes this event ===
data class CatalogProductPublishedEvent(
    val productId: String,
    val sku: String,
    val name: String,
    val price: Double,
    val currency: String,
    val occurredAt: String  // ISO-8601
)

// === ORDERING subscribes and translates ===
@Component
@KafkaListener(topics = ["catalog.products.published"])
class ProductCatalogIntegrationHandler(
    private val orderProductCacheRepository: OrderProductCacheRepository
) {
    
    private val log = org.slf4j.LoggerFactory.getLogger(this::class.java)
    
    @KafkaHandler
    fun onProductPublished(event: CatalogProductPublishedEvent) {
        log.info("Received product published event: ${event.productId}")
        
        // Translate catalog event → ordering domain concept
        val orderProduct = OrderProduct(
            productId = OrderProductId(event.productId),
            sku = event.sku,
            name = event.name,
            unitPrice = event.price.toBigDecimal(),
            currency = event.currency
        )
        
        // Cache in ordering context (eventual consistency)
        orderProductCacheRepository.save(event.productId, orderProduct)
    }
}

// === INVENTORY subscribes to ORDERING events ===
@Component
@KafkaListener(topics = ["ordering.orders.confirmed"])
class OrderConfirmedInventoryHandler(
    private val inventoryService: Any
) {
    
    @KafkaHandler
    fun onOrderConfirmed(event: OrderConfirmedIntegrationEvent) {
        // Reserve inventory for confirmed order
        event.lines.forEach { line ->
            // inventoryService.reserve(line.sku, line.quantity)
        }
    }
}

data class OrderConfirmedIntegrationEvent(
    val orderId: String,
    val customerId: String,
    val lines: List<OrderLineIntegrationDto>,
    val occurredAt: String
)

data class OrderLineIntegrationDto(val sku: String, val quantity: Int, val unitPrice: Double)

interface OrderProductCacheRepository {
    fun save(productId: String, product: OrderProduct)
    fun find(productId: String): OrderProduct?
}

typealias KafkaListener = org.springframework.kafka.annotation.KafkaListener
typealias KafkaHandler = org.springframework.kafka.annotation.KafkaHandler
```

---

## Shared Kernel

```kotlin
// Shared Kernel: code shared between contexts — ใช้ด้วยความระมัดระวัง!
// เหมาะสำหรับ fundamental types ที่ทั้ง team เห็นด้วย

package com.example.shared.kernel

// Money ที่ใช้ร่วมกันระหว่าง contexts
data class SharedMoney(
    val amount: java.math.BigDecimal,
    val currency: SharedCurrency
) {
    operator fun plus(other: SharedMoney): SharedMoney {
        require(currency == other.currency)
        return SharedMoney(amount + other.amount, currency)
    }
    
    operator fun compareTo(other: SharedMoney): Int {
        require(currency == other.currency)
        return amount.compareTo(other.amount)
    }
    
    override fun toString() = "$amount $currency"
}

enum class SharedCurrency { THB, USD, EUR, JPY, CNY }

// Common audit fields
data class AuditInfo(
    val createdAt: java.time.Instant = java.time.Instant.now(),
    val createdBy: String,
    val updatedAt: java.time.Instant? = null,
    val updatedBy: String? = null
)

// Standard pagination
data class PageRequest(
    val page: Int = 0,
    val size: Int = 20,
    val sortBy: String = "createdAt",
    val sortDirection: SortDirection = SortDirection.DESC
) {
    init {
        require(page >= 0) { "Page must be >= 0" }
        require(size in 1..100) { "Size must be between 1 and 100" }
    }
}

enum class SortDirection { ASC, DESC }

data class Page<T>(
    val content: List<T>,
    val page: Int,
    val size: Int,
    val totalElements: Long,
    val totalPages: Int
) {
    companion object {
        fun <T> of(content: List<T>, request: PageRequest, totalElements: Long): Page<T> {
            val totalPages = ((totalElements + request.size - 1) / request.size).toInt()
            return Page(content, request.page, request.size, totalElements, totalPages)
        }
    }
}

// Common integration event base
abstract class IntegrationEvent {
    val eventId: String = java.util.UUID.randomUUID().toString()
    val occurredAt: java.time.Instant = java.time.Instant.now()
    abstract val eventType: String
    abstract val aggregateId: String
    abstract val aggregateType: String
}

// Published Language: standard format for events going outside system
data class StandardIntegrationEvent(
    override val eventType: String,
    override val aggregateId: String,
    override val aggregateType: String,
    val payload: Map<String, Any>
) : IntegrationEvent()
```

---

## Context Map Strategies

```kotlin
// Context Map: describes relationships between bounded contexts

/*
Context Map สำหรับ E-Commerce:

Catalog → Ordering: Customer/Supplier relationship
  - Catalog (Supplier/Upstream) publishes events
  - Ordering (Customer/Downstream) consumes with ACL

Ordering → Inventory: Partnership
  - Both teams coordinate closely
  - Shared Kernel: Money, PageRequest

Ordering → Payments: Open Host Service
  - Payments ให้ standard API (Open Host)
  - Ordering ใช้ REST API
  
Customer → All: Conformist
  - Customer context เป็น "the truth" สำหรับ customer data
  - Other contexts conform to its model (no ACL needed)

Shipping → Ordering: Separate Ways
  - แยกกัน, integrate ผ่าน events เท่านั้น

Relationships:
  Partnership (P): teams work together
  Customer/Supplier (C/S): clear upstream/downstream
  Conformist (CF): downstream blindly conforms
  Anticorruption Layer (ACL): downstream translates
  Open Host Service (OHS): upstream provides standard API
  Published Language (PL): event schema agreed upon
  Separate Ways (SW): no integration
*/

// Domain Event published from Ordering (Published Language)
data class OrderPlacedPublishedEvent(
    val orderId: String,
    val customerId: String,
    val items: List<OrderItemPublished>,
    val totalAmount: java.math.BigDecimal,
    val currency: String,
    val shippingAddress: ShippingAddressPublished,
    val placedAt: String,  // ISO-8601
    val schemaVersion: Int = 1  // schema versioning
)

data class OrderItemPublished(
    val productId: String,
    val sku: String,
    val name: String,
    val quantity: Int,
    val unitPrice: java.math.BigDecimal
)

data class ShippingAddressPublished(
    val recipientName: String,
    val streetAddress: String,
    val city: String,
    val province: String,
    val postalCode: String,
    val country: String
)
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง ACL สำหรับ Payment context

// Payment Context ใช้ Stripe API ซึ่งมี model ของตัวเอง
data class StripeChargeResponse(
    val id: String,
    val amount: Int,  // in cents!
    val currency: String,
    val status: String,   // "succeeded", "pending", "failed"
    val payment_method: String,
    val receipt_url: String?,
    val failure_message: String?
)

// Our domain model
data class PaymentId(val value: String)
data class Payment(
    val id: PaymentId,
    val orderId: String,
    val amount: java.math.BigDecimal,
    val currency: String,
    val status: PaymentStatus,
    val receiptUrl: String?,
    val failureReason: String?
)

enum class PaymentStatus { PENDING, COMPLETED, FAILED, REFUNDED }

// TODO: สร้าง StripeAntiCorruptionLayer ที่:
// 1. Convert StripeChargeResponse.amount (cents) → BigDecimal (เงินจริง)
// 2. Map status "succeeded" → COMPLETED, "failed" → FAILED, "pending" → PENDING
// 3. Translate to our Payment domain model
// 4. Handle edge cases (null failure_message, etc.)

class StripeAntiCorruptionLayer {
    
    fun translate(stripeResponse: StripeChargeResponse, orderId: String): Payment {
        TODO("Implement translation logic")
    }
    
    private fun translateStatus(stripeStatus: String): PaymentStatus {
        TODO("Map stripe status to domain PaymentStatus")
    }
    
    private fun centsToDecimal(cents: Int): java.math.BigDecimal {
        TODO("Convert cents to decimal (100 cents = 1.00)")
    }
}
```

---

## สรุป Part 66

```
✅ Bounded Context: ขอบเขตที่ Ubiquitous Language มีความหมายชัดเจน
✅ Strategic DDD: design ระดับระบบ (ไม่ใช่ tactical)
✅ Context Map: เอกสารระบุความสัมพันธ์ระหว่าง contexts
✅ Customer/Supplier: clear upstream/downstream relationship
✅ Partnership: ทั้งสอง team coordinate ร่วมกัน
✅ Anti-Corruption Layer: แปลง external model → domain model
✅ Open Host Service: upstream ให้ standard API
✅ Published Language: schema ที่ตกลงร่วมกัน
✅ Conformist: downstream ใช้ model ของ upstream โดยตรง
✅ Separate Ways: แยกกัน, integrate เฉพาะที่จำเป็น
✅ Shared Kernel: code ที่แชร์ด้วยความระมัดระวัง
✅ "Product" มีความหมายต่างกันใน Catalog vs Ordering
✅ Event-based integration: eventual consistency ระหว่าง contexts
✅ Schema versioning: schemaVersion field ใน integration events
✅ Context isolation: ไม่แชร์ database ระหว่าง contexts
```

---

*Part 66/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
