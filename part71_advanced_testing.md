# Part 71: Advanced Testing Patterns

## สารบัญ
1. [Property-Based Testing](#property-based-testing)
2. [Contract Testing](#contract-testing)
3. [Mutation Testing](#mutation-testing)
4. [Architecture Testing](#architecture-testing)
5. [Test Containers Integration](#test-containers-integration)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Property-Based Testing

```kotlin
// Property-based testing ด้วย Kotest
// แทนที่จะเขียน test case แบบตายตัว เราบอก "property" ที่ต้องเป็นจริงเสมอ
// Kotest จะ generate inputs สุ่มมา verify

dependencies {
    testImplementation("io.kotest:kotest-runner-junit5:5.8.0")
    testImplementation("io.kotest:kotest-property:5.8.0")
    testImplementation("io.kotest:kotest-assertions-core:5.8.0")
}

import io.kotest.core.spec.style.StringSpec
import io.kotest.property.forAll
import io.kotest.property.Arb
import io.kotest.property.arbitrary.*
import io.kotest.matchers.shouldBe
import io.kotest.matchers.shouldBeGreaterThanOrEqualTo

class MoneyPropertyTest : StringSpec({
    
    // Property: การบวกเงินต้องเป็น commutative (a + b == b + a)
    "Money addition should be commutative" {
        forAll(Arb.positiveInt(1000), Arb.positiveInt(1000)) { a, b ->
            val moneyA = Money(a.toLong(), "THB")
            val moneyB = Money(b.toLong(), "THB")
            
            (moneyA + moneyB) == (moneyB + moneyA)
        }
    }
    
    // Property: การบวกเงิน 0 ต้องได้ค่าเดิม (identity element)
    "Adding zero money should return original amount" {
        forAll(Arb.positiveInt(10000)) { amount ->
            val money = Money(amount.toLong(), "THB")
            val zero = Money(0, "THB")
            
            (money + zero) == money
        }
    }
    
    // Property: discount ต้องไม่เกิน original price
    "Discounted price should never exceed original price" {
        forAll(
            Arb.positiveInt(100000),  // price 0-100000
            Arb.int(0..100)            // discount 0-100%
        ) { price, discountPercent ->
            val originalPrice = price.toLong()
            val discounted = applyDiscount(originalPrice, discountPercent)
            
            discounted <= originalPrice && discounted >= 0
        }
    }
    
    // Property: string reverse twice == original
    "Reversing string twice should return original" {
        forAll(Arb.string(0..100)) { str ->
            str.reversed().reversed() == str
        }
    }
    
    // Property: sort must be idempotent
    "Sorting a sorted list should return same list" {
        forAll(Arb.list(Arb.int(), 0..50)) { list ->
            val sorted = list.sorted()
            sorted.sorted() == sorted
        }
    }
    
    // Property: encode then decode == original
    "Base64 encode then decode should return original" {
        forAll(Arb.byteArray(Arb.int(0..1000))) { bytes ->
            val encoded = java.util.Base64.getEncoder().encode(bytes)
            val decoded = java.util.Base64.getDecoder().decode(encoded)
            
            decoded.contentEquals(bytes)
        }
    }
})

data class Money(val amount: Long, val currency: String) : Comparable<Money> {
    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Cannot add different currencies" }
        return Money(amount + other.amount, currency)
    }
    
    override fun compareTo(other: Money): Int = amount.compareTo(other.amount)
}

fun applyDiscount(price: Long, discountPercent: Int): Long {
    return price - (price * discountPercent / 100)
}

// Custom Arb generators for domain objects
object Arbs {
    
    fun email(): Arb<String> = Arb.string(5..20, Codepoint.az())
        .map { name -> "$name@example.com" }
    
    fun positiveAmount(): Arb<java.math.BigDecimal> =
        Arb.bigDecimal(java.math.BigDecimal("0.01"), java.math.BigDecimal("999999.99"))
    
    fun productName(): Arb<String> = Arb.string(3..100, Codepoint.az() + Codepoint.of(' '))
        .map { it.trim() }
        .filter { it.isNotBlank() }
    
    fun order(userArb: Arb<User> = user()): Arb<Order> = Arb.bind(
        userArb,
        Arb.list(orderItem(), 1..10),
        Arb.instant()
    ) { user, items, createdAt ->
        Order(
            id = java.util.UUID.randomUUID().toString(),
            user = user,
            items = items,
            createdAt = createdAt
        )
    }
    
    fun user(): Arb<User> = Arb.bind(
        Arb.uuid(),
        email()
    ) { id, email ->
        User(id.toString(), email)
    }
    
    fun orderItem(): Arb<OrderItem> = Arb.bind(
        Arb.uuid(),
        Arb.positiveInt(10),
        positiveAmount()
    ) { productId, quantity, unitPrice ->
        OrderItem(productId.toString(), quantity, unitPrice)
    }
}

data class User(val id: String, val email: String)
data class Order(val id: String, val user: User, val items: List<OrderItem>, val createdAt: java.time.Instant)
data class OrderItem(val productId: String, val quantity: Int, val unitPrice: java.math.BigDecimal)

// Usage of custom arbs
class OrderPropertyTest : StringSpec({
    
    "Order total should equal sum of item totals" {
        forAll(Arbs.order()) { order ->
            val expectedTotal = order.items.sumOf { it.unitPrice * it.quantity.toBigDecimal() }
            val actualTotal = calculateOrderTotal(order)
            
            actualTotal == expectedTotal
        }
    }
    
    "Order with at least one item should have positive total" {
        forAll(Arbs.order()) { order ->
            calculateOrderTotal(order) > java.math.BigDecimal.ZERO
        }
    }
})

fun calculateOrderTotal(order: Order): java.math.BigDecimal {
    return order.items.sumOf { it.unitPrice * it.quantity.toBigDecimal() }
}
```

---

## Contract Testing

```kotlin
// Consumer-Driven Contract Testing ด้วย Pact
// Producer (API server) ต้องทำตาม contract ที่ Consumer กำหนด

dependencies {
    testImplementation("au.com.dius.pact.consumer:junit5:4.6.5")
    testImplementation("au.com.dius.pact.provider:junit5spring:4.6.5")
}

// CONSUMER side: Product Service client
@ExtendWith(PactConsumerTestExt::class)
@PactTestFor(providerName = "ProductService", port = "8080")
class ProductClientContractTest {
    
    @Pact(consumer = "OrderService")
    fun getProductPact(builder: PactDslWithProvider): RequestResponsePact {
        return builder
            .given("product with id 1 exists")
            .uponReceiving("a request for product 1")
            .path("/api/products/1")
            .method("GET")
            .headers(mapOf("Accept" to "application/json"))
            .willRespondWith()
            .status(200)
            .headers(mapOf("Content-Type" to "application/json"))
            .body(
                PactDslJsonBody()
                    .integerType("id", 1)
                    .stringType("name", "Test Product")
                    .decimalType("price", 99.99)
                    .stringValue("currency", "THB")
                    .booleanType("inStock", true)
            )
            .toPact()
    }
    
    @Test
    @PactTestFor(pactMethod = "getProductPact")
    fun `should get product by id`(mockServer: MockServer) {
        val client = ProductApiClient(mockServer.getUrl())
        val product = client.getProduct(1L)
        
        product.id shouldBe 1L
        product.name shouldNotBe null
        product.price shouldBeGreaterThan java.math.BigDecimal.ZERO
    }
    
    @Pact(consumer = "OrderService")
    fun getProductNotFoundPact(builder: PactDslWithProvider): RequestResponsePact {
        return builder
            .given("product with id 999 does not exist")
            .uponReceiving("a request for non-existent product")
            .path("/api/products/999")
            .method("GET")
            .willRespondWith()
            .status(404)
            .body(
                PactDslJsonBody()
                    .stringType("error", "NOT_FOUND")
                    .stringType("message")
            )
            .toPact()
    }
}

// PROVIDER side: verify contracts against real service
@Provider("ProductService")
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@PactFolder("src/test/pacts")
class ProductServicePactTest {
    
    @LocalServerPort
    private var port: Int = 0
    
    @BeforeEach
    fun setup(context: PactVerificationContext) {
        context.target = HttpTestTarget("localhost", port)
    }
    
    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider::class)
    fun pactVerificationTest(context: PactVerificationContext) {
        context.verifyInteraction()
    }
    
    // Setup test state from "given" clauses
    @State("product with id 1 exists")
    fun productExists() {
        // Insert test product into database
        testProductRepository.save(TestProduct(1L, "Test Product", 99.99.toBigDecimal()))
    }
    
    @State("product with id 999 does not exist")
    fun productDoesNotExist() {
        testProductRepository.deleteById(999L)
    }
    
    @Autowired
    private lateinit var testProductRepository: TestProductRepository
}

interface TestProductRepository {
    fun save(product: TestProduct)
    fun deleteById(id: Long)
}

data class TestProduct(val id: Long, val name: String, val price: java.math.BigDecimal)

class ProductApiClient(private val baseUrl: String) {
    private val restTemplate = org.springframework.web.client.RestTemplate()
    
    fun getProduct(id: Long): ProductResponse {
        return restTemplate.getForObject("$baseUrl/api/products/$id", ProductResponse::class.java)!!
    }
}

data class ProductResponse(
    val id: Long = 0,
    val name: String = "",
    val price: java.math.BigDecimal = java.math.BigDecimal.ZERO,
    val currency: String = "",
    val inStock: Boolean = false
)

typealias ExtendWith = org.junit.jupiter.api.extension.ExtendWith
typealias Test = org.junit.jupiter.api.Test
typealias BeforeEach = org.junit.jupiter.api.BeforeEach
typealias TestTemplate = org.junit.jupiter.api.TestTemplate
```

---

## Mutation Testing

```kotlin
// Mutation Testing ด้วย Pitest
// Pitest เปลี่ยน (mutate) code เล็กน้อย แล้ว run tests
// ถ้า test ไม่ fail = tests ไม่ดีพอ (mutation survived)

// build.gradle.kts
plugins {
    id("info.solidsoft.pitest") version "1.9.11"
}

pitest {
    testPlugin.set("junit5")
    targetClasses.set(listOf("com.example.*"))
    targetTests.set(listOf("com.example.*Test"))
    threads.set(4)
    outputFormats.set(listOf("HTML", "XML"))
    mutators.set(listOf("DEFAULTS"))
    coverageThreshold.set(60)  // 60% line coverage minimum
    mutationThreshold.set(70)  // 70% mutation score minimum
}

// ตัวอย่าง mutants ที่ Pitest สร้าง:
// Original: if (price > 0) → Mutant: if (price >= 0)
// Original: a + b → Mutant: a - b
// Original: return true → Mutant: return false
// Original: x++ → Mutant: x--

// Bad test: doesn't catch mutations
class BadDiscountTest {
    @Test
    fun `apply discount works`() {
        val service = DiscountService()
        val result = service.applyDiscount(100.0, 10)
        
        // Only checks it runs, not the value!
        result shouldNotBe null  // Mutation on the math will survive
    }
}

// Good test: catches mutations
class GoodDiscountTest {
    
    private val service = DiscountService()
    
    @Test
    fun `10 percent discount on 100 should give 90`() {
        service.applyDiscount(100.0, 10) shouldBe 90.0
    }
    
    @Test
    fun `0 percent discount should return original price`() {
        service.applyDiscount(100.0, 0) shouldBe 100.0
    }
    
    @Test
    fun `100 percent discount should return 0`() {
        service.applyDiscount(100.0, 100) shouldBe 0.0
    }
    
    @Test
    fun `discount should not produce negative price`() {
        service.applyDiscount(50.0, 100) shouldBeGreaterThanOrEqualTo 0.0
    }
    
    @Test
    fun `negative discount percentage should throw`() {
        shouldThrow<IllegalArgumentException> {
            service.applyDiscount(100.0, -1)
        }
    }
}

class DiscountService {
    fun applyDiscount(price: Double, discountPercent: Int): Double {
        require(discountPercent in 0..100) { "Discount must be 0-100%" }
        return price * (1 - discountPercent / 100.0)
    }
}

// Coverage vs Mutation Score
// Code coverage 100% ≠ good tests
// Example: high coverage but bad assertions:
class PoorTestExample {
    @Test
    fun `this test has 100% coverage but zero mutation score`() {
        val service = OrderService()
        
        // Exercises all paths, but no meaningful assertions
        service.calculateTotal(emptyList())
        service.calculateTotal(listOf(OrderItem("p1", 1, 10.0.toBigDecimal())))
        service.calculateTotal(listOf(
            OrderItem("p1", 2, 50.0.toBigDecimal()),
            OrderItem("p2", 1, 30.0.toBigDecimal())
        ))
        
        // This assertion will never catch a mutation!
        true shouldBe true
    }
}

class OrderService {
    fun calculateTotal(items: List<OrderItem>): java.math.BigDecimal {
        return items.sumOf { it.unitPrice * it.quantity.toBigDecimal() }
    }
}
```

---

## Architecture Testing

```kotlin
// ArchUnit: verify architectural rules at test time
// ป้องกัน import ที่ผิดกฎ เช่น domain ไม่ควร import infrastructure

dependencies {
    testImplementation("com.tngtech.archunit:archunit-junit5:1.2.0")
}

@AnalyzeClasses(packages = ["com.example"])
class ArchitectureTest {
    
    // Domain layer must not depend on infrastructure
    @ArchTest
    val domainMustNotDependOnInfrastructure: ArchRule =
        noClasses()
            .that().resideInAPackage("..domain..")
            .should().dependOnClassesThat()
            .resideInAnyPackage("..infrastructure..", "..adapter..", "..repository..")
            .`as`("Domain layer must not depend on infrastructure")
    
    // Controllers should only be in web package
    @ArchTest
    val controllersMustBeInWebPackage: ArchRule =
        classes()
            .that().areAnnotatedWith(RestController::class.java)
            .should().resideInAPackage("..web..")
            .`as`("Controllers must be in web package")
    
    // Services must be annotated with @Service
    @ArchTest
    val serviceClassesMustHaveServiceAnnotation: ArchRule =
        classes()
            .that().haveNameEndingWith("Service")
            .and().areNotInterfaces()
            .should().beAnnotatedWith(Service::class.java)
            .`as`("Service classes must have @Service annotation")
    
    // No field injection allowed
    @ArchTest
    val noFieldInjection: ArchRule =
        noFields()
            .should().beAnnotatedWith(Autowired::class.java)
            .`as`("Use constructor injection instead of field injection")
    
    // No cycles between packages
    @ArchTest
    val noCyclesBetweenPackages: ArchRule =
        slices()
            .matching("com.example.(*)..")
            .should().beFreeOfCycles()
    
    // Use cases should only be called from controllers/handlers
    @ArchTest
    val useCasesMustOnlyBeCalledFromControllers: ArchRule =
        classes()
            .that().haveNameEndingWith("UseCase")
            .should().onlyBeAccessed()
            .byClassesThat()
            .resideInAnyPackage("..web..", "..adapter..")
    
    // Repository interfaces should be in domain, implementations in infrastructure
    @ArchTest
    val repositoryInterfacesInDomain: ArchRule =
        classes()
            .that().haveNameEndingWith("Repository")
            .and().areInterfaces()
            .should().resideInAPackage("..domain..")
    
    // All public methods in Services should have @Transactional if they modify data
    @ArchTest
    val writingServiceMethodsMustBeTransactional: ArchRule =
        methods()
            .that().arePublic()
            .and().haveNameStartingWith("save")
            .and().areDeclaredInClassesThat().areAnnotatedWith(Service::class.java)
            .should().beAnnotatedWith(Transactional::class.java)
}

// Custom ArchUnit rules
object CustomRules {
    
    val noLoggerInDomain: ArchRule = noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .haveFullyQualifiedName("org.slf4j.Logger")
        .`as`("Domain objects should not have loggers")
    
    val dtosMustBeDataClasses: ArchRule =
        classes()
            .that().haveNameEndingWith("Dto")
            .or().haveNameEndingWith("Request")
            .or().haveNameEndingWith("Response")
            .should(object : ArchCondition<JavaClass>("be Kotlin data classes") {
                override fun check(item: JavaClass, events: ConditionEvents) {
                    if (!item.isAnnotatedWith("kotlin.Metadata")) return
                    // Check if it's a data class by looking for componentN methods
                    val isDataClass = item.methods.any { it.name.startsWith("component") }
                    if (!isDataClass) {
                        events.add(SimpleConditionEvent.violated(
                            item,
                            "${item.name} is not a data class"
                        ))
                    }
                }
            })
}

typealias RestController = org.springframework.web.bind.annotation.RestController
typealias Service = org.springframework.stereotype.Service
typealias Autowired = org.springframework.beans.factory.annotation.Autowired
typealias Transactional = org.springframework.transaction.annotation.Transactional
```

---

## Test Containers Integration

```kotlin
// Testcontainers: spin up real Docker containers for integration tests

dependencies {
    testImplementation("org.testcontainers:junit-jupiter:1.19.3")
    testImplementation("org.testcontainers:postgresql:1.19.3")
    testImplementation("org.testcontainers:kafka:1.19.3")
    testImplementation("org.testcontainers:redis:1.19.3")
}

// Base class for all integration tests
@SpringBootTest
@ActiveProfiles("test")
abstract class AbstractIntegrationTest {
    
    companion object {
        @JvmField
        val postgres = PostgreSQLContainer<Nothing>("postgres:16-alpine").apply {
            withDatabaseName("testdb")
            withUsername("test")
            withPassword("test")
            withReuse(true)  // Reuse container across test classes
        }
        
        @JvmField
        val redis = GenericContainer<Nothing>("redis:7-alpine").apply {
            withExposedPorts(6379)
            withReuse(true)
        }
        
        @JvmField
        val kafka = KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.4.0")).apply {
            withReuse(true)
        }
        
        init {
            // Start all containers in parallel
            com.github.dockerjava.api.command.CreateContainerCmd::class
            listOf(postgres, redis, kafka).parallelStream().forEach { it.start() }
        }
        
        @JvmStatic
        @DynamicPropertySource
        fun configureProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
            
            registry.add("spring.data.redis.host", redis::getHost)
            registry.add("spring.data.redis.port") { redis.getMappedPort(6379) }
            
            registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers)
        }
    }
}

// Product integration test using real database
@Testcontainers
class ProductIntegrationTest : AbstractIntegrationTest() {
    
    @Autowired
    private lateinit var productService: ProductService
    
    @Autowired
    private lateinit var productRepository: ProductRepository
    
    @BeforeEach
    fun setup() {
        productRepository.deleteAll()
    }
    
    @Test
    fun `should create and retrieve product`() {
        val request = CreateProductRequest(
            name = "Integration Test Product",
            price = java.math.BigDecimal("199.99"),
            sku = "ITP-001"
        )
        
        val created = productService.create(request)
        
        val retrieved = productService.findById(created.id)
        
        retrieved.name shouldBe "Integration Test Product"
        retrieved.price shouldBe java.math.BigDecimal("199.99")
    }
    
    @Test
    fun `should search products by name`() {
        repeat(5) { i ->
            productService.create(
                CreateProductRequest("Product $i", java.math.BigDecimal("100"), "SKU-$i")
            )
        }
        productService.create(
            CreateProductRequest("Different Name", java.math.BigDecimal("100"), "SKU-999")
        )
        
        val results = productService.search("Product")
        
        results.size shouldBe 5
        results.all { it.name.contains("Product") } shouldBe true
    }
}

// Kafka integration test
@Testcontainers
class OrderEventIntegrationTest : AbstractIntegrationTest() {
    
    @Autowired
    private lateinit var orderService: OrderService
    
    @Autowired
    private lateinit var testKafkaConsumer: TestKafkaConsumer
    
    @Test
    fun `should publish order placed event when order is created`() {
        val order = orderService.placeOrder(PlaceOrderRequest("user-1", listOf()))
        
        // Wait for event to arrive (max 5 seconds)
        val event = testKafkaConsumer.waitForEvent("orders.placed", 5000)
        
        event shouldNotBe null
        event!!.orderId shouldBe order.id
    }
}

// Supporting classes
interface ProductService {
    fun create(request: CreateProductRequest): ProductDto
    fun findById(id: Long): ProductDto
    fun search(query: String): List<ProductDto>
}

interface ProductRepository {
    fun deleteAll()
}

interface OrderService {
    fun placeOrder(request: PlaceOrderRequest): OrderDto
}

interface TestKafkaConsumer {
    fun waitForEvent(topic: String, timeoutMs: Long): OrderPlacedEvent?
}

data class CreateProductRequest(val name: String, val price: java.math.BigDecimal, val sku: String)
data class ProductDto(val id: Long, val name: String, val price: java.math.BigDecimal)
data class PlaceOrderRequest(val userId: String, val items: List<Any>)
data class OrderDto(val id: String)
data class OrderPlacedEvent(val orderId: String)

typealias Testcontainers = org.testcontainers.junit.jupiter.Testcontainers
typealias SpringBootTest = org.springframework.boot.test.context.SpringBootTest
typealias ActiveProfiles = org.springframework.test.context.ActiveProfiles
typealias DynamicPropertySource = org.springframework.test.context.DynamicPropertySource
typealias DynamicPropertyRegistry = org.springframework.test.context.DynamicPropertyRegistry
typealias BeforeEach = org.junit.jupiter.api.BeforeEach
typealias Autowired = org.springframework.beans.factory.annotation.Autowired
```

---

## แบบฝึกหัด

```kotlin
// Exercise: เขียน property-based tests สำหรับ CartService

class CartService {
    
    fun addItem(cart: Cart, item: CartItem): Cart {
        val existing = cart.items.find { it.productId == item.productId }
        
        return if (existing != null) {
            cart.copy(items = cart.items.map {
                if (it.productId == item.productId) it.copy(quantity = it.quantity + item.quantity)
                else it
            })
        } else {
            cart.copy(items = cart.items + item)
        }
    }
    
    fun removeItem(cart: Cart, productId: String): Cart {
        return cart.copy(items = cart.items.filter { it.productId != productId })
    }
    
    fun calculateTotal(cart: Cart): java.math.BigDecimal {
        return cart.items.sumOf { it.price * it.quantity.toBigDecimal() }
    }
    
    fun applyVoucher(cart: Cart, voucher: Voucher): Cart {
        val discount = when (voucher.type) {
            VoucherType.FIXED -> voucher.value
            VoucherType.PERCENTAGE -> calculateTotal(cart) * voucher.value / 100.toBigDecimal()
        }
        
        return cart.copy(
            discountAmount = discount.min(calculateTotal(cart))  // Can't discount more than total
        )
    }
}

data class Cart(
    val userId: String,
    val items: List<CartItem> = emptyList(),
    val discountAmount: java.math.BigDecimal = java.math.BigDecimal.ZERO
)

data class CartItem(
    val productId: String,
    val quantity: Int,
    val price: java.math.BigDecimal
)

data class Voucher(
    val code: String,
    val type: VoucherType,
    val value: java.math.BigDecimal
)

enum class VoucherType { FIXED, PERCENTAGE }

// TODO: Write these property-based tests:
// 1. Adding same item twice should increase quantity, not add duplicate
// 2. Cart total after removing item should be <= original total
// 3. Voucher discount should never make total negative
// 4. Adding item with quantity 0 should not change total
// 5. Cart total with no items should always be 0

class CartServicePropertyTest : StringSpec({
    
    val service = CartService()
    
    "adding same product twice should consolidate quantities" {
        forAll(
            Arb.string(5..10),
            Arb.positiveInt(10),
            Arb.positiveInt(10),
            Arb.positiveDouble().map { java.math.BigDecimal(it).setScale(2, java.math.RoundingMode.HALF_UP) }
        ) { productId, qty1, qty2, price ->
            val cart = Cart("user1")
            val item1 = CartItem(productId, qty1, price)
            val item2 = CartItem(productId, qty2, price)
            
            val result = service.addItem(service.addItem(cart, item1), item2)
            
            // Should have only one entry for this product
            result.items.count { it.productId == productId } == 1 &&
            result.items.first { it.productId == productId }.quantity == qty1 + qty2
        }
    }
    
    "cart total after voucher should never be negative" {
        forAll(
            Arb.list(Arb.bind(
                Arb.uuid(),
                Arb.positiveInt(5),
                Arb.bigDecimal(java.math.BigDecimal("1.00"), java.math.BigDecimal("100.00"))
            ) { id, qty, price -> CartItem(id.toString(), qty, price) }, 1..5),
            Arb.bigDecimal(java.math.BigDecimal("1"), java.math.BigDecimal("100"))
        ) { items, discountValue ->
            val cart = Cart("user1", items)
            val voucher = Voucher("TEST", VoucherType.FIXED, discountValue)
            
            val result = service.applyVoucher(cart, voucher)
            val finalTotal = service.calculateTotal(result) - result.discountAmount
            
            finalTotal >= java.math.BigDecimal.ZERO
        }
    }
})
```

---

## สรุป Part 71

```
✅ Property-Based Testing: Kotest property + Arb generators
✅ forAll: verify property holds for all generated inputs
✅ Arb.string/int/double/list: built-in arbitrary generators
✅ Custom Arbs: domain-specific generators (Arbs.order())
✅ Mathematical properties: commutative, identity, idempotent
✅ Contract Testing: Pact consumer-driven contracts
✅ @Pact: define expected request/response
✅ @State: setup provider state for contract verification
✅ Mutation Testing: Pitest plugin configuration
✅ Mutation Score: 70% threshold (not code coverage)
✅ Mutations: arithmetic, conditional, return value changes
✅ Good assertions: catch all mutations in domain logic
✅ Architecture Testing: ArchUnit rules
✅ Layer rules: domain must not import infrastructure
✅ noFieldInjection: enforce constructor injection
✅ noCycles: no circular package dependencies
✅ Testcontainers: PostgreSQL, Redis, Kafka in Docker
✅ @DynamicPropertySource: inject container URLs into Spring
✅ Container reuse: withReuse(true) for faster tests
```

---

*Part 71/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
