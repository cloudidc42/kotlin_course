# Part 50: Full-Stack E-Commerce Project (Milestone)

## สารบัญ
1. [Project Architecture](#project-architecture)
2. [Domain Model](#domain-model)
3. [Application Layer](#application-layer)
4. [Infrastructure Layer](#infrastructure-layer)
5. [REST API](#rest-api)
6. [Security](#security)
7. [Testing Strategy](#testing-strategy)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Project Architecture

```
e-commerce-api/
├── src/main/kotlin/com/example/ecommerce/
│   ├── domain/
│   │   ├── model/
│   │   │   ├── product/
│   │   │   │   ├── Product.kt          # Aggregate Root
│   │   │   │   ├── ProductId.kt
│   │   │   │   ├── Money.kt            # Value Object
│   │   │   │   └── Category.kt
│   │   │   ├── order/
│   │   │   │   ├── Order.kt            # Aggregate Root
│   │   │   │   ├── OrderItem.kt
│   │   │   │   ├── OrderStatus.kt
│   │   │   │   └── ShippingAddress.kt  # Value Object
│   │   │   └── user/
│   │   │       ├── User.kt
│   │   │       ├── UserId.kt
│   │   │       └── Email.kt
│   │   ├── repository/
│   │   │   ├── ProductRepository.kt
│   │   │   ├── OrderRepository.kt
│   │   │   └── UserRepository.kt
│   │   ├── service/
│   │   │   ├── PricingService.kt
│   │   │   └── InventoryService.kt
│   │   └── event/
│   │       ├── DomainEvent.kt
│   │       ├── OrderCreatedEvent.kt
│   │       └── StockDeductedEvent.kt
│   ├── application/
│   │   ├── product/
│   │   │   ├── CreateProductUseCase.kt
│   │   │   ├── UpdateProductUseCase.kt
│   │   │   ├── SearchProductsUseCase.kt
│   │   │   └── GetProductUseCase.kt
│   │   ├── order/
│   │   │   ├── PlaceOrderUseCase.kt
│   │   │   ├── CancelOrderUseCase.kt
│   │   │   └── GetOrderHistoryUseCase.kt
│   │   └── user/
│   │       ├── RegisterUserUseCase.kt
│   │       └── GetUserProfileUseCase.kt
│   ├── infrastructure/
│   │   ├── persistence/
│   │   │   ├── ProductJpaRepository.kt
│   │   │   ├── OrderJpaRepository.kt
│   │   │   └── entities/
│   │   ├── messaging/
│   │   │   └── KafkaOrderEventPublisher.kt
│   │   ├── cache/
│   │   │   └── RedisCacheConfig.kt
│   │   └── search/
│   │       └── ElasticsearchProductRepository.kt
│   └── presentation/
│       ├── product/
│       │   └── ProductController.kt
│       ├── order/
│       │   └── OrderController.kt
│       └── auth/
│           └── AuthController.kt
└── src/test/kotlin/
    ├── unit/
    ├── integration/
    └── e2e/
```

---

## Domain Model

```kotlin
// ==================== Domain Events ====================
sealed class DomainEvent {
    abstract val occurredAt: java.time.Instant
    abstract val eventId: String
}

data class OrderCreatedEvent(
    val orderId: String,
    val customerId: String,
    val items: List<OrderItemSnapshot>,
    val totalAmount: Double,
    override val occurredAt: java.time.Instant = java.time.Instant.now(),
    override val eventId: String = java.util.UUID.randomUUID().toString()
) : DomainEvent()

data class OrderCancelledEvent(
    val orderId: String,
    val reason: String,
    override val occurredAt: java.time.Instant = java.time.Instant.now(),
    override val eventId: String = java.util.UUID.randomUUID().toString()
) : DomainEvent()

data class StockDeductedEvent(
    val productId: String,
    val quantity: Int,
    val orderId: String,
    override val occurredAt: java.time.Instant = java.time.Instant.now(),
    override val eventId: String = java.util.UUID.randomUUID().toString()
) : DomainEvent()

data class OrderItemSnapshot(val productId: String, val quantity: Int, val unitPrice: Double)

// ==================== Value Objects ====================
@JvmInline value class ProductId(val value: String) {
    init { require(value.isNotBlank()) { "ProductId cannot be blank" } }
    override fun toString() = value
}

@JvmInline value class OrderId(val value: String) {
    init { require(value.isNotBlank()) { "OrderId cannot be blank" } }
    override fun toString() = value
}

@JvmInline value class CustomerId(val value: String) {
    override fun toString() = value
}

data class Money(val amount: java.math.BigDecimal, val currency: String = "THB") {
    
    init {
        require(amount >= java.math.BigDecimal.ZERO) { "Amount cannot be negative" }
    }
    
    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Cannot add different currencies" }
        return Money(amount + other.amount, currency)
    }
    
    operator fun times(quantity: Int): Money = Money(amount * quantity.toBigDecimal(), currency)
    
    operator fun minus(other: Money): Money {
        require(currency == other.currency) { "Cannot subtract different currencies" }
        require(amount >= other.amount) { "Result would be negative" }
        return Money(amount - other.amount, currency)
    }
    
    fun isZero() = amount == java.math.BigDecimal.ZERO
    
    companion object {
        val ZERO = Money(java.math.BigDecimal.ZERO)
        fun of(amount: Double) = Money(amount.toBigDecimal().setScale(2, java.math.RoundingMode.HALF_UP))
        fun of(amount: String) = Money(java.math.BigDecimal(amount).setScale(2, java.math.RoundingMode.HALF_UP))
    }
}

data class ShippingAddress(
    val recipientName: String,
    val phone: String,
    val addressLine1: String,
    val addressLine2: String? = null,
    val district: String,
    val province: String,
    val postalCode: String,
    val country: String = "TH"
) {
    init {
        require(recipientName.isNotBlank()) { "Recipient name required" }
        require(phone.matches(Regex("^0[6-9]\\d{8}$"))) { "Invalid Thai phone number" }
        require(postalCode.matches(Regex("^\\d{5}$"))) { "Invalid postal code" }
    }
}

// ==================== Product Aggregate ====================
class Product private constructor(
    val id: ProductId,
    name: String,
    description: String,
    price: Money,
    stockQuantity: Int,
    val categoryId: String?,
    val sku: String?,
    isActive: Boolean
) {
    var name: String = name
        private set
    
    var description: String = description
        private set
    
    var price: Money = price
        private set
    
    var stockQuantity: Int = stockQuantity
        private set
    
    var isActive: Boolean = isActive
        private set
    
    private val _events = mutableListOf<DomainEvent>()
    val events: List<DomainEvent> get() = _events.toList()
    
    fun updateDetails(name: String, description: String, price: Money) {
        require(name.isNotBlank()) { "Product name required" }
        require(price.amount > java.math.BigDecimal.ZERO) { "Price must be positive" }
        this.name = name
        this.description = description
        this.price = price
    }
    
    fun deductStock(quantity: Int, orderId: OrderId) {
        require(quantity > 0) { "Quantity must be positive" }
        require(stockQuantity >= quantity) { "Insufficient stock: have $stockQuantity, need $quantity" }
        stockQuantity -= quantity
        _events.add(StockDeductedEvent(id.value, quantity, orderId.value))
    }
    
    fun addStock(quantity: Int) {
        require(quantity > 0) { "Quantity must be positive" }
        stockQuantity += quantity
    }
    
    fun isInStock(quantity: Int = 1) = stockQuantity >= quantity
    
    fun deactivate() { isActive = false }
    fun activate() { isActive = true }
    
    fun clearEvents() = _events.clear()
    
    companion object {
        fun create(
            name: String,
            description: String,
            price: Money,
            stockQuantity: Int,
            categoryId: String? = null,
            sku: String? = null
        ): Product {
            require(name.isNotBlank()) { "Product name required" }
            require(price.amount > java.math.BigDecimal.ZERO) { "Price must be positive" }
            require(stockQuantity >= 0) { "Stock cannot be negative" }
            
            return Product(
                id = ProductId(java.util.UUID.randomUUID().toString()),
                name = name,
                description = description,
                price = price,
                stockQuantity = stockQuantity,
                categoryId = categoryId,
                sku = sku,
                isActive = true
            )
        }
        
        fun reconstitute(
            id: String, name: String, description: String,
            price: Money, stockQuantity: Int,
            categoryId: String?, sku: String?, isActive: Boolean
        ) = Product(ProductId(id), name, description, price, stockQuantity, categoryId, sku, isActive)
    }
}

// ==================== Order Aggregate ====================
class Order private constructor(
    val id: OrderId,
    val customerId: CustomerId,
    status: OrderStatus,
    items: List<OrderLineItem>,
    val shippingAddress: ShippingAddress,
    val notes: String?
) {
    var status: OrderStatus = status
        private set
    
    private val _items = items.toMutableList()
    val items: List<OrderLineItem> get() = _items.toList()
    
    private val _events = mutableListOf<DomainEvent>()
    val events: List<DomainEvent> get() = _events.toList()
    
    val subtotal: Money get() = items.fold(Money.ZERO) { acc, item -> acc + item.lineTotal }
    
    val totalAmount: Money get() = subtotal  // Could add tax, shipping, discounts
    
    fun confirm() {
        check(status == OrderStatus.PENDING) { "Can only confirm pending orders" }
        status = OrderStatus.CONFIRMED
    }
    
    fun startProcessing() {
        check(status == OrderStatus.CONFIRMED) { "Can only process confirmed orders" }
        status = OrderStatus.PROCESSING
    }
    
    fun ship(trackingNumber: String? = null) {
        check(status == OrderStatus.PROCESSING) { "Can only ship processing orders" }
        status = OrderStatus.SHIPPED
    }
    
    fun deliver() {
        check(status == OrderStatus.SHIPPED) { "Can only deliver shipped orders" }
        status = OrderStatus.DELIVERED
    }
    
    fun cancel(reason: String) {
        check(status in setOf(OrderStatus.PENDING, OrderStatus.CONFIRMED)) {
            "Cannot cancel order in status $status"
        }
        status = OrderStatus.CANCELLED
        _events.add(OrderCancelledEvent(id.value, reason))
    }
    
    fun clearEvents() = _events.clear()
    
    companion object {
        fun place(
            customerId: CustomerId,
            items: List<OrderItemRequest>,
            products: Map<ProductId, Product>,
            shippingAddress: ShippingAddress,
            notes: String? = null
        ): Order {
            require(items.isNotEmpty()) { "Order must have items" }
            
            val lineItems = items.map { item ->
                val product = products[item.productId]
                    ?: throw IllegalArgumentException("Product ${item.productId} not found")
                
                require(product.isActive) { "Product ${product.name} is not available" }
                require(product.isInStock(item.quantity)) {
                    "Insufficient stock for ${product.name}"
                }
                
                OrderLineItem(
                    id = java.util.UUID.randomUUID().toString(),
                    productId = item.productId,
                    productName = product.name,
                    quantity = item.quantity,
                    unitPrice = product.price
                )
            }
            
            val order = Order(
                id = OrderId(java.util.UUID.randomUUID().toString()),
                customerId = customerId,
                status = OrderStatus.PENDING,
                items = lineItems,
                shippingAddress = shippingAddress,
                notes = notes
            )
            
            order._events.add(OrderCreatedEvent(
                orderId = order.id.value,
                customerId = customerId.value,
                items = lineItems.map { OrderItemSnapshot(it.productId.value, it.quantity, it.unitPrice.amount.toDouble()) },
                totalAmount = order.totalAmount.amount.toDouble()
            ))
            
            return order
        }
    }
}

data class OrderLineItem(
    val id: String,
    val productId: ProductId,
    val productName: String,
    val quantity: Int,
    val unitPrice: Money
) {
    val lineTotal: Money get() = unitPrice * quantity
}

data class OrderItemRequest(val productId: ProductId, val quantity: Int)

enum class OrderStatus {
    PENDING, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED, REFUNDED
}
```

---

## Application Layer

```kotlin
// ==================== Place Order Use Case ====================
@Service
@Transactional
class PlaceOrderUseCase(
    private val orderRepository: EcommerceOrderRepository,
    private val productRepository: EcommerceProductRepository,
    private val eventPublisher: DomainEventPublisher
) {
    
    fun execute(command: PlaceOrderCommand): PlaceOrderResult {
        // Load products
        val productIds = command.items.map { ProductId(it.productId) }.toSet()
        val products = productRepository.findAllByIds(productIds)
            .associateBy { it.id }
        
        val missingProducts = productIds - products.keys
        if (missingProducts.isNotEmpty()) {
            return PlaceOrderResult.ProductNotFound(missingProducts.map { it.value })
        }
        
        // Validate and place order
        val order = try {
            Order.place(
                customerId = CustomerId(command.customerId),
                items = command.items.map { OrderItemRequest(ProductId(it.productId), it.quantity) },
                products = products,
                shippingAddress = command.shippingAddress,
                notes = command.notes
            )
        } catch (e: IllegalArgumentException) {
            return PlaceOrderResult.ValidationError(e.message ?: "Validation failed")
        }
        
        // Deduct stock
        command.items.forEach { item ->
            val product = products[ProductId(item.productId)]!!
            product.deductStock(item.quantity, order.id)
            productRepository.save(product)
        }
        
        // Save order
        orderRepository.save(order)
        
        // Publish domain events
        order.events.forEach { eventPublisher.publish(it) }
        order.clearEvents()
        
        products.values.forEach { product ->
            product.events.forEach { eventPublisher.publish(it) }
            product.clearEvents()
        }
        
        return PlaceOrderResult.Success(order.id.value, order.totalAmount)
    }
}

data class PlaceOrderCommand(
    val customerId: String,
    val items: List<OrderItemCommand>,
    val shippingAddress: ShippingAddress,
    val notes: String? = null
)

data class OrderItemCommand(val productId: String, val quantity: Int)

sealed class PlaceOrderResult {
    data class Success(val orderId: String, val totalAmount: Money) : PlaceOrderResult()
    data class ProductNotFound(val productIds: List<String>) : PlaceOrderResult()
    data class ValidationError(val message: String) : PlaceOrderResult()
    data class InsufficientStock(val productId: String) : PlaceOrderResult()
}

// ==================== Search Products Use Case ====================
@Service
class SearchProductsUseCase(
    private val productRepository: EcommerceProductRepository
) {
    
    fun execute(query: SearchProductsQuery): ProductSearchResults {
        val page = productRepository.search(
            keyword = query.keyword,
            categoryId = query.categoryId,
            minPrice = query.minPrice?.let { Money.of(it) },
            maxPrice = query.maxPrice?.let { Money.of(it) },
            inStockOnly = query.inStockOnly,
            page = query.page,
            size = query.size,
            sortBy = query.sortBy
        )
        
        return ProductSearchResults(
            products = page.content.map { it.toSummary() },
            totalElements = page.totalElements,
            totalPages = page.totalPages,
            currentPage = page.number,
            pageSize = page.size
        )
    }
}

data class SearchProductsQuery(
    val keyword: String? = null,
    val categoryId: String? = null,
    val minPrice: Double? = null,
    val maxPrice: Double? = null,
    val inStockOnly: Boolean = false,
    val page: Int = 0,
    val size: Int = 20,
    val sortBy: ProductSortBy = ProductSortBy.RELEVANCE
)

enum class ProductSortBy { RELEVANCE, PRICE_ASC, PRICE_DESC, NEWEST, POPULAR }

data class ProductSearchResults(
    val products: List<ProductSummary>,
    val totalElements: Long,
    val totalPages: Int,
    val currentPage: Int,
    val pageSize: Int
)

data class ProductSummary(
    val id: String,
    val name: String,
    val price: Double,
    val stockQuantity: Int,
    val categoryId: String?,
    val isInStock: Boolean
)

fun Product.toSummary() = ProductSummary(
    id = id.value, name = name,
    price = price.amount.toDouble(),
    stockQuantity = stockQuantity,
    categoryId = categoryId,
    isInStock = stockQuantity > 0
)

// ==================== Repository Interfaces ====================
interface EcommerceProductRepository {
    fun findById(id: ProductId): Product?
    fun findAllByIds(ids: Set<ProductId>): List<Product>
    fun save(product: Product): Product
    fun search(
        keyword: String?,
        categoryId: String?,
        minPrice: Money?,
        maxPrice: Money?,
        inStockOnly: Boolean,
        page: Int,
        size: Int,
        sortBy: ProductSortBy
    ): PageResult<Product>
}

interface EcommerceOrderRepository {
    fun findById(id: OrderId): Order?
    fun findByCustomerId(customerId: CustomerId, page: Int, size: Int): PageResult<Order>
    fun save(order: Order): Order
}

interface DomainEventPublisher {
    fun publish(event: DomainEvent)
}

data class PageResult<T>(
    val content: List<T>,
    val totalElements: Long,
    val totalPages: Int,
    val number: Int,
    val size: Int
)
```

---

## REST API

```kotlin
// ==================== Product Controller ====================
@RestController
@RequestMapping("/api/v1/products")
class EcommerceProductController(
    private val searchProductsUseCase: SearchProductsUseCase,
    private val createProductUseCase: CreateProductUseCaseFacade,
    private val getProductUseCase: GetProductUseCaseFacade
) {
    
    @GetMapping
    fun searchProducts(
        @RequestParam(required = false) keyword: String?,
        @RequestParam(required = false) categoryId: String?,
        @RequestParam(required = false) minPrice: Double?,
        @RequestParam(required = false) maxPrice: Double?,
        @RequestParam(defaultValue = "false") inStockOnly: Boolean,
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int,
        @RequestParam(defaultValue = "RELEVANCE") sortBy: ProductSortBy
    ): ResponseEntity<ProductListResponse> {
        val results = searchProductsUseCase.execute(
            SearchProductsQuery(keyword, categoryId, minPrice, maxPrice, inStockOnly, page, size, sortBy)
        )
        return ResponseEntity.ok(results.toResponse())
    }
    
    @GetMapping("/{id}")
    fun getProduct(@PathVariable id: String): ResponseEntity<ProductDetailResponse> {
        val product = getProductUseCase.execute(id)
            ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(product.toDetailResponse())
    }
    
    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    fun createProduct(@RequestBody @Valid request: CreateProductRequest): ResponseEntity<ProductDetailResponse> {
        val product = createProductUseCase.execute(request)
        return ResponseEntity.status(HttpStatus.CREATED).body(product.toDetailResponse())
    }
    
    @PutMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    fun updateProduct(
        @PathVariable id: String,
        @RequestBody @Valid request: UpdateProductRequest
    ): ResponseEntity<ProductDetailResponse> {
        val product = createProductUseCase.update(id, request)
            ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(product.toDetailResponse())
    }
}

// ==================== Order Controller ====================
@RestController
@RequestMapping("/api/v1/orders")
class EcommerceOrderController(
    private val placeOrderUseCase: PlaceOrderUseCase,
    private val getOrderHistoryUseCase: GetOrderHistoryUseCaseFacade
) {
    
    @PostMapping
    fun placeOrder(
        @RequestBody @Valid request: PlaceOrderRequest,
        authentication: Authentication
    ): ResponseEntity<Any> {
        val command = PlaceOrderCommand(
            customerId = authentication.name,
            items = request.items.map { OrderItemCommand(it.productId, it.quantity) },
            shippingAddress = request.shippingAddress.toDomain(),
            notes = request.notes
        )
        
        return when (val result = placeOrderUseCase.execute(command)) {
            is PlaceOrderResult.Success -> ResponseEntity.status(HttpStatus.CREATED).body(
                mapOf("orderId" to result.orderId, "totalAmount" to result.totalAmount.amount)
            )
            is PlaceOrderResult.ProductNotFound -> ResponseEntity.badRequest().body(
                ProblemDetail("product-not-found", "Products not found", 400,
                    "Product IDs: ${result.productIds.joinToString()}")
            )
            is PlaceOrderResult.ValidationError -> ResponseEntity.badRequest().body(
                ProblemDetail("validation-error", "Validation failed", 400, result.message)
            )
            is PlaceOrderResult.InsufficientStock -> ResponseEntity.status(HttpStatus.CONFLICT).body(
                ProblemDetail("insufficient-stock", "Insufficient stock", 409,
                    "Product ${result.productId} has insufficient stock")
            )
        }
    }
    
    @GetMapping
    fun getOrderHistory(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "10") size: Int,
        authentication: Authentication
    ): ResponseEntity<OrderHistoryResponse> {
        val history = getOrderHistoryUseCase.execute(authentication.name, page, size)
        return ResponseEntity.ok(history)
    }
    
    @GetMapping("/{id}")
    fun getOrder(@PathVariable id: String, authentication: Authentication): ResponseEntity<OrderDetailResponse> {
        val order = getOrderHistoryUseCase.getById(id, authentication.name)
            ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(order)
    }
}

// ==================== DTOs ====================
data class PlaceOrderRequest(
    @field:NotEmpty val items: List<OrderItemDto>,
    @field:NotNull val shippingAddress: ShippingAddressDto,
    val notes: String?
)

data class OrderItemDto(
    @field:NotBlank val productId: String,
    @field:Min(1) val quantity: Int
)

data class ShippingAddressDto(
    val recipientName: String,
    val phone: String,
    val addressLine1: String,
    val addressLine2: String?,
    val district: String,
    val province: String,
    val postalCode: String
) {
    fun toDomain() = ShippingAddress(
        recipientName, phone, addressLine1, addressLine2,
        district, province, postalCode
    )
}

data class CreateProductRequest(
    @field:NotBlank val name: String,
    val description: String = "",
    @field:DecimalMin("0.01") val price: Double,
    @field:Min(0) val stockQuantity: Int,
    val categoryId: String?,
    val sku: String?
)

data class UpdateProductRequest(
    val name: String?,
    val description: String?,
    val price: Double?,
    val stockQuantity: Int?
)

data class ProblemDetail(val type: String, val title: String, val status: Int, val detail: String? = null)

data class ProductListResponse(
    val products: List<ProductSummary>,
    val totalElements: Long,
    val totalPages: Int,
    val currentPage: Int
)

data class ProductDetailResponse(
    val id: String, val name: String, val description: String,
    val price: Double, val stockQuantity: Int, val categoryId: String?,
    val sku: String?, val isActive: Boolean, val isInStock: Boolean
)

data class OrderHistoryResponse(
    val orders: List<OrderSummaryDto>,
    val totalElements: Long
)

data class OrderSummaryDto(
    val id: String, val status: String,
    val totalAmount: Double, val createdAt: String,
    val itemCount: Int
)

data class OrderDetailResponse(
    val id: String, val status: String,
    val items: List<OrderLineItemDto>,
    val totalAmount: Double,
    val shippingAddress: ShippingAddressDto,
    val notes: String?
)

data class OrderLineItemDto(
    val productId: String, val productName: String,
    val quantity: Int, val unitPrice: Double, val lineTotal: Double
)

// Extensions
fun ProductSearchResults.toResponse() = ProductListResponse(
    products, totalElements, totalPages, currentPage
)

fun Product.toDetailResponse() = ProductDetailResponse(
    id.value, name, description, price.amount.toDouble(),
    stockQuantity, categoryId, sku, isActive, stockQuantity > 0
)

// Facades (adapters connecting application layer to infrastructure)
interface CreateProductUseCaseFacade {
    fun execute(request: CreateProductRequest): Product
    fun update(id: String, request: UpdateProductRequest): Product?
}

interface GetProductUseCaseFacade {
    fun execute(id: String): Product?
}

interface GetOrderHistoryUseCaseFacade {
    fun execute(customerId: String, page: Int, size: Int): OrderHistoryResponse
    fun getById(id: String, customerId: String): OrderDetailResponse?
}
```

---

## Testing Strategy

```kotlin
// Unit test - Domain logic
class OrderPlacementTest {
    
    @Test
    fun `should place order with valid items`() {
        val products = mapOf(
            ProductId("prod-1") to Product.create(
                name = "Laptop",
                description = "High-performance laptop",
                price = Money.of("49999.00"),
                stockQuantity = 10
            )
        )
        
        val address = ShippingAddress(
            recipientName = "สมชาย ใจดี",
            phone = "0812345678",
            addressLine1 = "123 ถนนสุขุมวิท",
            district = "คลองเตย",
            province = "กรุงเทพมหานคร",
            postalCode = "10110"
        )
        
        val order = Order.place(
            customerId = CustomerId("cust-001"),
            items = listOf(OrderItemRequest(ProductId("prod-1"), 2)),
            products = products,
            shippingAddress = address
        )
        
        assertThat(order.status).isEqualTo(OrderStatus.PENDING)
        assertThat(order.items).hasSize(1)
        assertThat(order.totalAmount.amount).isEqualByComparingTo("99998.00")
        assertThat(order.events).hasSize(1)
        assertThat(order.events[0]).isInstanceOf(OrderCreatedEvent::class.java)
    }
    
    @Test
    fun `should reject order when out of stock`() {
        val product = Product.create("Widget", "", Money.of("9.99"), 0)
        val products = mapOf(product.id to product)
        
        val address = ShippingAddress("John", "0812345678", "123 Main St", 
            district = "A", province = "B", postalCode = "12345")
        
        assertThrows<IllegalArgumentException> {
            Order.place(
                customerId = CustomerId("cust-001"),
                items = listOf(OrderItemRequest(product.id, 1)),
                products = products,
                shippingAddress = address
            )
        }
    }
    
    @Test
    fun `money arithmetic should be correct`() {
        val price = Money.of("49.99")
        val qty2 = price * 3
        assertThat(qty2.amount).isEqualByComparingTo("149.97")
        
        val tax = Money.of("10.50")
        val total = qty2 + tax
        assertThat(total.amount).isEqualByComparingTo("160.47")
    }
}

// Integration test - Use Case
@SpringBootTest
@Transactional
class PlaceOrderUseCaseIntegrationTest {
    
    @Autowired lateinit var placeOrderUseCase: PlaceOrderUseCase
    @Autowired lateinit var productRepository: EcommerceProductRepository
    
    @Test
    fun `should place order and deduct stock`() {
        // Arrange
        val product = Product.create("Test Product", "Desc", Money.of("100.00"), 5)
        productRepository.save(product)
        
        val address = ShippingAddress("Test User", "0812345678",
            "456 Test St", district = "X", province = "Y", postalCode = "10200")
        
        val command = PlaceOrderCommand(
            customerId = "cust-001",
            items = listOf(OrderItemCommand(product.id.value, 2)),
            shippingAddress = address
        )
        
        // Act
        val result = placeOrderUseCase.execute(command)
        
        // Assert
        assertThat(result).isInstanceOf(PlaceOrderResult.Success::class.java)
        val saved = productRepository.findById(product.id)!!
        assertThat(saved.stockQuantity).isEqualTo(3)  // 5 - 2
    }
}
```

---

## แบบฝึกหัด

```kotlin
// เพิ่มฟีเจอร์ต่อไปนี้:

// 1. Wishlist: user สามารถ add/remove products จาก wishlist
//    - WishlistItem(userId, productId, addedAt)
//    - AddToWishlistUseCase, RemoveFromWishlistUseCase, GetWishlistUseCase

// 2. Review & Rating: user สามารถ review สินค้าหลัง delivered
//    - Review(id, productId, userId, rating 1-5, comment, createdAt)
//    - CreateReviewUseCase (validate: user must have bought and received product)
//    - GetProductReviewsUseCase with pagination
//    - Domain rule: 1 user = 1 review per product

// 3. Coupon System:
//    data class Coupon(
//        val code: String,
//        val type: CouponType,
//        val value: BigDecimal,
//        val minOrderAmount: Money?,
//        val maxUses: Int?,
//        val usedCount: Int = 0,
//        val expiresAt: LocalDate
//    )
//    - ApplyCouponUseCase: validate + calculate discount
//    - Integrate with PlaceOrderUseCase

// 4. Order Tracking:
//    - OrderStatusHistory (orderId, status, timestamp, note)
//    - UpdateOrderStatusUseCase for admin
//    - GetOrderTrackingUseCase for customer

println("Project milestone complete! Parts 01-50 represent core Kotlin development.")
```

---

## สรุป Part 50 (Milestone)

```
✅ Clean Architecture applied to real e-commerce
✅ Rich Domain Model: Product + Order aggregates
✅ Value Objects: Money, ProductId, OrderId, ShippingAddress
✅ Domain Events: OrderCreated, StockDeducted, OrderCancelled
✅ Use Cases: PlaceOrder, SearchProducts
✅ Repository pattern: domain interfaces, infrastructure implementations
✅ Business rules enforced in domain (not service)
✅ REST API: product search, order placement
✅ Input validation with Jakarta Bean Validation
✅ Error handling with sealed result types
✅ Unit tests: domain logic
✅ Integration tests: use cases with database

🎯 Milestone: Parts 01-50 completed!
   - Basics: Kotlin syntax, OOP, FP, Collections, Coroutines
   - Architecture: Clean, DDD, CQRS, Event Sourcing
   - Database: JPA, Flyway, Elasticsearch, Redis
   - APIs: REST, GraphQL, gRPC, WebSocket
   - Security: JWT, OAuth2, Social Login
   - Infrastructure: Docker, K8s, CI/CD
   - Testing: Unit, Integration, Contract, Architecture
   - Messaging: Kafka, Event-driven
   - Reactive: WebFlux, Reactor
```

---

*Part 50/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
