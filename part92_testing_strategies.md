# Part 92: Testing Strategies ระดับ Enterprise

## สารบัญ
1. [Testing Pyramid](#testing-pyramid)
2. [Unit Testing Advanced](#unit-testing-advanced)
3. [Integration Testing](#integration-testing)
4. [Contract Testing ด้วย Pact](#contract-testing-ด้วย-pact)
5. [Performance Testing ด้วย Gatling](#performance-testing-ด้วย-gatling)
6. [Mutation Testing](#mutation-testing)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Testing Pyramid

```
                    /\
                   /E2E\         ← น้อย, แพง, ช้า
                  /------\
                 / Integr.\     ← ปานกลาง
                /----------\
               /   Unit     \   ← มาก, ถูก, เร็ว
              /--------------\

Unit Tests:    70-80% ของ tests ทั้งหมด
               - ทดสอบ 1 class/function
               - No external deps (mock everything)
               - Runs in ms

Integration:   15-20%
               - ทดสอบ interaction ระหว่าง components
               - Real DB, real Kafka (Testcontainers)
               - Runs in seconds

E2E:           5-10%
               - ทดสอบ full user flow
               - Browser automation / real API calls
               - Runs in minutes

Contract Tests: (อยู่ระหว่าง Integration และ E2E)
               - ทดสอบ API contracts ระหว่าง services
               - Consumer-driven

Performance:   separate
               - Load testing, stress testing
               - Gatling / k6 / JMeter
```

---

## Unit Testing Advanced

```kotlin
// build.gradle.kts
dependencies {
    testImplementation("io.kotest:kotest-runner-junit5:5.8.0")
    testImplementation("io.kotest:kotest-assertions-core:5.8.0")
    testImplementation("io.kotest:kotest-property:5.8.0")
    testImplementation("io.mockk:mockk:1.13.8")
    testImplementation("org.assertj:assertj-core:3.24.2")
    testImplementation("com.ninja-squad:springmockk:4.0.2")
    testImplementation("net.jqwik:jqwik:1.8.1")  // property-based testing
}

// Kotest: Kotlin-first testing framework
import io.kotest.core.spec.style.DescribeSpec
import io.kotest.core.spec.style.BehaviorSpec
import io.kotest.core.spec.style.FunSpec
import io.kotest.assertions.throwables.shouldThrow
import io.kotest.matchers.shouldBe
import io.kotest.matchers.shouldNotBe
import io.kotest.matchers.collections.shouldContain
import io.kotest.matchers.collections.shouldHaveSize
import io.kotest.matchers.string.shouldContain
import io.kotest.matchers.string.shouldStartWith
import io.kotest.property.Arb
import io.kotest.property.arbitrary.*
import io.kotest.property.checkAll
import io.mockk.*

// BDD-style tests (BehaviorSpec)
class PlaceOrderUseCaseTest : BehaviorSpec({
    
    val productRepository = mockk<ProductRepository>()
    val orderRepository = mockk<OrderRepository>()
    val eventPublisher = mockk<EventPublisher>()
    val paymentGateway = mockk<PaymentGateway>()
    
    val useCase = PlaceOrderUseCase(productRepository, orderRepository, eventPublisher, paymentGateway)
    
    beforeEach {
        clearAllMocks()
    }
    
    Given("a valid order with available products") {
        val product = Product.create("Laptop", Money(45000.toBigDecimal(), "THB"), 5)
        
        When("placing order for 2 units") {
            every { productRepository.findById(product.id) } returns product
            every { paymentGateway.charge(any(), any()) } returns PaymentResult(success = true, transactionId = "txn-123")
            every { orderRepository.save(any()) } answers { firstArg() }
            every { eventPublisher.publish(any()) } just Runs
            
            val command = PlaceOrderCommand("user-1", listOf(OrderItemCommand(product.id.value, 2)))
            val result = useCase.execute(command)
            
            Then("order should be created") {
                result shouldNotBe null
            }
            
            Then("stock should be decremented") {
                product.stockQuantity shouldBe 3
            }
            
            Then("payment should be charged") {
                verify { paymentGateway.charge("user-1", any()) }
            }
            
            Then("order placed event should be published") {
                verify { eventPublisher.publish(ofType<OrderPlacedEvent>()) }
            }
        }
        
        When("placing order for more than available stock") {
            every { productRepository.findById(product.id) } returns product
            
            val command = PlaceOrderCommand("user-1", listOf(OrderItemCommand(product.id.value, 10)))
            
            Then("should throw InsufficientStockException") {
                shouldThrow<InsufficientStockException> {
                    useCase.execute(command)
                }.message shouldContain "Insufficient stock"
            }
        }
    }
    
    Given("a payment failure") {
        val product = Product.create("iPhone", Money(35000.toBigDecimal(), "THB"), 3)
        
        When("payment gateway rejects") {
            every { productRepository.findById(product.id) } returns product
            every { paymentGateway.charge(any(), any()) } returns PaymentResult(success = false, transactionId = "")
            
            val command = PlaceOrderCommand("user-1", listOf(OrderItemCommand(product.id.value, 1)))
            
            Then("should throw PaymentFailedException") {
                shouldThrow<PaymentFailedException> { useCase.execute(command) }
            }
            
            Then("stock should be restored") {
                product.stockQuantity shouldBe 3  // unchanged
            }
        }
    }
})

// Property-based testing: test with generated inputs
class MoneyPropertyTest : FunSpec({
    
    test("addition is commutative") {
        checkAll(
            Arb.bigDecimal(min = 0.toBigDecimal(), max = 1_000_000.toBigDecimal()),
            Arb.bigDecimal(min = 0.toBigDecimal(), max = 1_000_000.toBigDecimal())
        ) { a, b ->
            val moneyA = Money(a, "THB")
            val moneyB = Money(b, "THB")
            
            (moneyA + moneyB) shouldBe (moneyB + moneyA)
        }
    }
    
    test("discount reduces amount") {
        checkAll(
            Arb.bigDecimal(min = 100.toBigDecimal(), max = 10000.toBigDecimal()),
            Arb.int(1..50)
        ) { amount, discountPercent ->
            val money = Money(amount, "THB")
            val discounted = money.applyDiscount(discountPercent)
            
            discounted.amount shouldBe money.amount.multiply((100 - discountPercent).toBigDecimal())
                .divide(100.toBigDecimal())
        }
    }
    
    test("money cannot be negative") {
        checkAll(Arb.negativeDouble()) { negative ->
            shouldThrow<IllegalArgumentException> {
                Money(negative.toBigDecimal(), "THB")
            }
        }
    }
    
    // Generate custom domain objects
    val arbProduct = Arb.bind(
        Arb.string(5..50),
        Arb.bigDecimal(min = 1.toBigDecimal(), max = 100000.toBigDecimal()),
        Arb.positiveInt()
    ) { name, price, stock ->
        Product.create(name, Money(price, "THB"), stock)
    }
    
    test("cart total equals sum of item prices") {
        checkAll(
            Arb.list(arbProduct, 1..10),
            Arb.list(Arb.positiveInt(max = 5), 1..10)
        ) { products, quantities ->
            val cart = Cart()
            products.zip(quantities).forEach { (product, qty) ->
                cart.addItem(product.id, qty, product.price)
            }
            
            val expectedTotal = products.zip(quantities)
                .sumOf { (p, q) -> p.price.amount.multiply(q.toBigDecimal()) }
            
            cart.totalAmount.amount shouldBe expectedTotal
        }
    }
})

// Parameterized tests
import org.junit.jupiter.params.ParameterizedTest
import org.junit.jupiter.params.provider.CsvSource
import org.junit.jupiter.params.provider.MethodSource

class ProductValidatorTest {
    
    @ParameterizedTest
    @CsvSource(
        "MacBook Pro, 45000.00, 10, true",
        "iPhone 15, 35000.00, 5, true",
        ", 35000.00, 5, false",         // blank name
        "iPhone, 0.00, 5, false",        // zero price
        "iPhone, 35000.00, -1, false"    // negative stock
    )
    fun `should validate product input`(name: String?, price: String, stock: Int, expected: Boolean) {
        val validator = ProductValidator()
        val result = validator.validate(name, price.toBigDecimal(), stock)
        
        org.assertj.core.api.Assertions.assertThat(result.isValid).isEqualTo(expected)
    }
    
    companion object {
        @JvmStatic
        fun discountTestCases(): List<org.junit.jupiter.params.provider.Arguments> {
            return listOf(
                org.junit.jupiter.params.provider.Arguments.of(1000.toBigDecimal(), 10, 900.toBigDecimal()),
                org.junit.jupiter.params.provider.Arguments.of(1000.toBigDecimal(), 50, 500.toBigDecimal()),
                org.junit.jupiter.params.provider.Arguments.of(500.toBigDecimal(), 20, 400.toBigDecimal()),
                org.junit.jupiter.params.provider.Arguments.of(1000.toBigDecimal(), 0, 1000.toBigDecimal())
            )
        }
    }
    
    @ParameterizedTest
    @MethodSource("discountTestCases")
    fun `should apply discount correctly`(
        amount: java.math.BigDecimal,
        discountPercent: Int,
        expected: java.math.BigDecimal
    ) {
        val money = Money(amount, "THB")
        val result = money.applyDiscount(discountPercent)
        
        org.assertj.core.api.Assertions.assertThat(result.amount).isEqualByComparingTo(expected)
    }
}

// Mock complex interactions
class OrderServiceMockTest {
    
    private val productRepo = mockk<ProductRepository>()
    private val orderRepo = mockk<OrderRepository>()
    
    @org.junit.jupiter.api.Test
    fun `should call repositories in correct order`() {
        val product = Product.create("Test", Money(100.toBigDecimal(), "THB"), 10)
        every { productRepo.findById(any()) } returns product
        every { orderRepo.save(any()) } answers { firstArg() }
        
        // Use spyk to spy on real object
        val service = spyk(
            object : OrderService {
                override fun createOrder(command: Any): Order = TODO()
            }
        )
        
        // Verify call order
        verifyOrder {
            productRepo.findById(any())
            orderRepo.save(any())
        }
    }
    
    @org.junit.jupiter.api.Test
    fun `should capture arguments`() {
        val productSlot = slot<Product>()
        every { orderRepo.save(capture(productSlot)) } answers { firstArg() }
        
        // After calling save...
        // productSlot.captured.name shouldBe "expected"
    }
    
    @org.junit.jupiter.api.Test
    fun `should verify no unexpected calls`() {
        every { productRepo.findById("1") } returns mockk()
        
        // After use case...
        confirmVerified(productRepo, orderRepo)  // fails if extra calls made
    }
}

// Stubs
class ProductRepository {
    fun findById(id: ProductId): Product? = TODO()
    fun save(product: Product): Product = TODO()
}
interface OrderRepository { fun save(order: Order): Order }
interface PaymentGateway { fun charge(customerId: String, amount: Money): PaymentResult }
data class PaymentResult(val success: Boolean, val transactionId: String)
class PaymentFailedException : RuntimeException("Payment failed")
class InsufficientStockException(msg: String) : RuntimeException(msg)
interface OrderService { fun createOrder(command: Any): Order }
class ProductValidator {
    data class ValidationResult(val isValid: Boolean)
    fun validate(name: String?, price: java.math.BigDecimal, stock: Int): ValidationResult =
        ValidationResult(!name.isNullOrBlank() && price > java.math.BigDecimal.ZERO && stock >= 0)
}
data class Cart() {
    val items = mutableListOf<Pair<ProductId, Pair<Int, Money>>>()
    val totalAmount: Money get() = items.fold(Money(java.math.BigDecimal.ZERO, "THB")) { acc, (_, pair) ->
        acc + pair.second.multiply(pair.first)
    }
    fun addItem(productId: ProductId, qty: Int, price: Money) { items.add(productId to (qty to price)) }
}
fun Money.multiply(qty: Int): Money = Money(amount.multiply(qty.toBigDecimal()), currency)
fun Money.multiply(qty: ProductId): Money = this
```

---

## Integration Testing ด้วย Testcontainers

```kotlin
// Testcontainers: run real dependencies in Docker
// implementation("org.testcontainers:testcontainers:1.19.3")
// implementation("org.testcontainers:postgresql:1.19.3")
// implementation("org.testcontainers:kafka:1.19.3")

import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.containers.KafkaContainer
import org.testcontainers.utility.DockerImageName
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource

// Abstract base for integration tests
@SpringBootTest
@org.springframework.transaction.annotation.Transactional
abstract class IntegrationTestBase {
    
    companion object {
        
        @JvmStatic
        val postgres = PostgreSQLContainer(DockerImageName.parse("postgres:16-alpine"))
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test")
        
        @JvmStatic
        val kafka = KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.5.0"))
        
        @JvmStatic
        val redis = org.testcontainers.containers.GenericContainer<Nothing>("redis:7-alpine")
            .apply { withExposedPorts(6379) }
        
        init {
            postgres.start()
            kafka.start()
            redis.start()
        }
        
        @JvmStatic
        @DynamicPropertySource
        fun configureProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
            registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers)
            registry.add("spring.data.redis.host") { redis.host }
            registry.add("spring.data.redis.port") { redis.getMappedPort(6379).toString() }
        }
    }
}

// Integration test for Order Service
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderServiceIntegrationTest : IntegrationTestBase() {
    
    @org.springframework.beans.factory.annotation.Autowired
    private lateinit var orderRepository: JpaOrderRepository
    
    @org.springframework.beans.factory.annotation.Autowired
    private lateinit var productRepository: JpaProductRepository
    
    @org.springframework.beans.factory.annotation.Autowired
    private lateinit var orderService: OrderApplicationService
    
    @org.springframework.beans.factory.annotation.Autowired
    private lateinit var kafkaConsumer: org.springframework.kafka.core.ConsumerFactory<String, Any>
    
    @org.junit.jupiter.api.Test
    fun `should persist order and publish event`() {
        // Arrange
        val category = categoryRepository.save(CategoryEntity(name = "Electronics"))
        val product = productRepository.save(
            ProductEntity(
                name = "MacBook",
                price = 50000.toBigDecimal(),
                currency = "THB",
                stockQuantity = 5,
                categoryId = category.id!!
            )
        )
        
        // Act
        val command = PlaceOrderCommand("user-1", listOf(OrderItemCommand(product.id!!, 2)))
        val orderId = orderService.placeOrder(command)
        
        // Assert DB
        val savedOrder = orderRepository.findById(orderId.value).orElseThrow()
        org.assertj.core.api.Assertions.assertThat(savedOrder.customerId).isEqualTo("user-1")
        org.assertj.core.api.Assertions.assertThat(savedOrder.status).isEqualTo("PENDING")
        
        // Assert stock decremented
        val updatedProduct = productRepository.findById(product.id!!).orElseThrow()
        org.assertj.core.api.Assertions.assertThat(updatedProduct.stockQuantity).isEqualTo(3)
        
        // Assert Kafka event published
        val consumer = org.springframework.kafka.test.utils.KafkaTestUtils.getRecords(
            org.apache.kafka.clients.consumer.KafkaConsumer<String, Any>(
                org.springframework.kafka.test.utils.KafkaTestUtils.consumerProps(
                    kafka.bootstrapServers,
                    "test-consumer",
                    "false"
                )
            ).apply { subscribe(listOf("ecommerce.orders")) },
            5000
        )
        
        org.assertj.core.api.Assertions.assertThat(consumer.records("ecommerce.orders"))
            .isNotEmpty
    }
    
    @org.junit.jupiter.api.Test
    fun `should rollback on payment failure`() {
        val product = productRepository.save(
            ProductEntity(name = "iPhone", price = 35000.toBigDecimal(), currency = "THB",
                stockQuantity = 3, categoryId = "category-1")
        )
        
        // Mock payment to fail
        // ...
        
        val command = PlaceOrderCommand("user-1", listOf(OrderItemCommand(product.id!!, 1)))
        
        org.assertj.core.api.Assertions.assertThatThrownBy { orderService.placeOrder(command) }
            .isInstanceOf(PaymentFailedException::class.java)
        
        // Stock should NOT be decremented (transaction rolled back)
        val unchanged = productRepository.findById(product.id!!).orElseThrow()
        org.assertj.core.api.Assertions.assertThat(unchanged.stockQuantity).isEqualTo(3)
    }
}

// REST API integration test
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProductApiIntegrationTest : IntegrationTestBase() {
    
    @org.springframework.beans.factory.annotation.Autowired
    private lateinit var testRestTemplate: org.springframework.boot.test.web.client.TestRestTemplate
    
    @org.junit.jupiter.api.Test
    fun `GET products returns paginated list`() {
        // Seed test data
        
        val response = testRestTemplate.getForEntity("/api/v1/products?page=0&size=5", String::class.java)
        
        org.assertj.core.api.Assertions.assertThat(response.statusCode)
            .isEqualTo(org.springframework.http.HttpStatus.OK)
        
        val page = com.fasterxml.jackson.databind.ObjectMapper().readTree(response.body)
        org.assertj.core.api.Assertions.assertThat(page["items"].isArray).isTrue
    }
    
    @org.junit.jupiter.api.Test
    fun `POST product returns 201 with location header`() {
        val request = mapOf(
            "name" to "New Product",
            "price" to 999.99,
            "stockQuantity" to 10,
            "categoryId" to "cat-1"
        )
        
        val headers = org.springframework.http.HttpHeaders()
        headers.contentType = org.springframework.http.MediaType.APPLICATION_JSON
        headers.setBearerAuth("admin-jwt-token")
        
        val response = testRestTemplate.postForEntity(
            "/api/v1/products",
            org.springframework.http.HttpEntity(request, headers),
            String::class.java
        )
        
        org.assertj.core.api.Assertions.assertThat(response.statusCode)
            .isEqualTo(org.springframework.http.HttpStatus.CREATED)
        org.assertj.core.api.Assertions.assertThat(response.headers.location).isNotNull
    }
}

// Stubs for compilation
interface JpaOrderRepository : org.springframework.data.jpa.repository.JpaRepository<OrderEntity2, String>
interface JpaProductRepository : org.springframework.data.jpa.repository.JpaRepository<ProductEntity, String>
interface categoryRepository { companion object { fun save(e: Any): CategoryEntity = TODO() } }
@javax.persistence.Entity @javax.persistence.Table(name = "orders2")
data class OrderEntity2(@javax.persistence.Id val id: String = "", val customerId: String = "", val status: String = "PENDING")
@javax.persistence.Entity @javax.persistence.Table(name = "categories")
data class CategoryEntity(@javax.persistence.Id @javax.persistence.GeneratedValue val id: String? = null, val name: String = "")
```

---

## Performance Testing ด้วย Gatling

```kotlin
// build.gradle.kts
// testImplementation("io.gatling:gatling-core:3.10.3")
// testImplementation("io.gatling.highcharts:gatling-charts-highcharts:3.10.3")

import io.gatling.core.Predef.*
import io.gatling.http.Predef.*
import scala.concurrent.duration.*

class EcommerceLoadTest extends Simulation {
  
  val httpConfig = http
    .baseUrl("http://localhost:8080")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")
    .header("Authorization", "Bearer ${System.getenv("TEST_JWT_TOKEN")}")

  // Scenarios
  val browseProducts = scenario("Browse Products")
    .exec(
      http("List Products")
        .get("/api/v1/products?page=0&size=20")
        .check(status.is(200))
        .check(jsonPath("$.items").exists)
    )
    .pause(1, 3)
    .exec(
      http("Get Product Detail")
        .get("/api/v1/products/${java.util.UUID.randomUUID()}")
        .check(status.in(200, 404))
    )
    .pause(2, 5)
  
  val checkout = scenario("Checkout Flow")
    .exec(
      http("Add to Cart")
        .post("/api/v1/cart/items")
        .body(StringBody("""{"productId":"prod-1","quantity":2}"""))
        .check(status.is(200))
    )
    .pause(1)
    .exec(
      http("Place Order")
        .post("/api/v1/orders")
        .body(StringBody("""{"items":[{"productId":"prod-1","quantity":2}]}"""))
        .check(status.is(201))
        .check(jsonPath("$.id").saveAs("orderId"))
    )
    .pause(1)
    .exec(
      http("Get Order Status")
        .get("/api/v1/orders/${orderId}")
        .check(status.is(200))
    )
  
  val searchProducts = scenario("Search Products")
    .feed(
      csv("test_queries.csv").random()  // productName, query
    )
    .exec(
      http("Search: #{query}")
        .get("/api/v1/products/search?q=#{query}")
        .check(status.is(200))
    )
  
  // Load profile
  setUp(
    browseProducts.inject(
      rampUsersPerSec(1).to(50).during(60.seconds),  // ramp up
      constantUsersPerSec(50).during(300.seconds),    // sustained load
      rampUsersPerSec(50).to(1).during(30.seconds)    // ramp down
    ),
    checkout.inject(
      rampUsersPerSec(1).to(10).during(60.seconds),
      constantUsersPerSec(10).during(300.seconds)
    ),
    searchProducts.inject(
      constantUsersPerSec(20).during(300.seconds)
    )
  )
    .protocols(httpConfig)
    .assertions(
      global.responseTime.percentile(95).lt(1000),  // P95 < 1s
      global.successfulRequests.percent.gt(99),      // 99% success rate
      forAll.failedRequests.count.lt(100)            // Less than 100 failures
    )
}
```

---

## Mutation Testing ด้วย Pitest

```kotlin
// build.gradle.kts
plugins {
    id("info.solidsoft.pitest") version "1.15.0"
}

pitest {
    targetClasses.set(listOf("com.ecommerce.domain.*", "com.ecommerce.application.*"))
    targetTests.set(listOf("com.ecommerce.*Test", "com.ecommerce.*Spec"))
    mutators.set(setOf("DEFAULTS", "STRONGER"))
    outputFormats.set(setOf("HTML", "XML"))
    mutationThreshold.set(80)  // fail if mutation score < 80%
    coverageThreshold.set(90)  // fail if line coverage < 90%
    threads.set(4)
    excludedMethods.set(listOf("hashCode", "equals", "toString"))
}

// Mutation testing หลักการ:
// 1. Pitest สร้าง "mutants" - เวอร์ชันที่เปลี่ยน code เล็กน้อย
// 2. รัน tests ทั้งหมด กับ mutant code
// 3. ถ้า test fail → mutant "killed" (test ดี)
// 4. ถ้า test pass → mutant "survived" (test ไม่ครอบคลุมพอ)

// Mutation operators:
// CONDITIONALS_BOUNDARY: < → <=, > → >=
// NEGATE_CONDITIONALS: == → !=, && → ||
// MATH: + → -, * → /, % → *
// INCREMENTS: i++ → i--
// VOID_METHOD_CALLS: remove void method calls
// RETURN_VALS: change return values (null, 0, empty)

// Example: ทำไม mutation testing สำคัญ
class DiscountCalculator {
    
    fun calculateDiscount(amount: Double, discountPercent: Int): Double {
        if (discountPercent < 0 || discountPercent > 100) {
            throw IllegalArgumentException("Invalid discount")
        }
        return amount * (100 - discountPercent) / 100.0
    }
}

class WeakDiscountTest {
    
    @org.junit.jupiter.api.Test
    fun `should calculate discount`() {
        val calc = DiscountCalculator()
        val result = calc.calculateDiscount(1000.0, 10)
        // This test passes but misses boundary conditions!
        // Mutation: discountPercent < 0 → discountPercent <= 0 
        //   Test still passes because we test with 10
    }
}

class StrongDiscountTest {
    
    @org.junit.jupiter.api.Test
    fun `boundary: exactly 0 percent is valid`() {
        val calc = DiscountCalculator()
        org.assertj.core.api.Assertions.assertThat(calc.calculateDiscount(1000.0, 0)).isEqualTo(1000.0)
    }
    
    @org.junit.jupiter.api.Test
    fun `boundary: exactly 100 percent is valid`() {
        val calc = DiscountCalculator()
        org.assertj.core.api.Assertions.assertThat(calc.calculateDiscount(1000.0, 100)).isEqualTo(0.0)
    }
    
    @org.junit.jupiter.api.Test
    fun `boundary: -1 percent is invalid`() {
        shouldThrow<IllegalArgumentException> {
            DiscountCalculator().calculateDiscount(1000.0, -1)
        }
    }
    
    @org.junit.jupiter.api.Test
    fun `boundary: 101 percent is invalid`() {
        shouldThrow<IllegalArgumentException> {
            DiscountCalculator().calculateDiscount(1000.0, 101)
        }
    }
    
    @org.junit.jupiter.api.Test
    fun `calculation accuracy`() {
        val calc = DiscountCalculator()
        org.assertj.core.api.Assertions.assertThat(calc.calculateDiscount(1000.0, 10)).isEqualTo(900.0)
        org.assertj.core.api.Assertions.assertThat(calc.calculateDiscount(1000.0, 50)).isEqualTo(500.0)
        org.assertj.core.api.Assertions.assertThat(calc.calculateDiscount(999.0, 33)).isCloseTo(669.33, org.assertj.core.api.Assertions.within(0.01))
    }
}

fun <T: Throwable> shouldThrow(block: () -> Any): T = TODO()
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง comprehensive tests สำหรับ Inventory Service

class InventoryServiceTest : DescribeSpec({
    
    val repo = mockk<InventoryRepository>()
    val publisher = mockk<EventPublisher>()
    val service = InventoryServiceImpl(repo, publisher)
    
    describe("reserveStock") {
        
        context("when stock is sufficient") {
            val item = InventoryItem("prod-1", currentStock = 10, reservedStock = 0)
            
            it("should create reservation") {
                every { repo.findByProductId("prod-1") } returns item
                every { repo.save(any()) } answers { firstArg() }
                every { publisher.publish(any()) } just Runs
                
                val result = service.reserve("order-1", "prod-1", 3)
                
                result.success shouldBe true
                result.reservationId shouldNotBe null
                
                verify { repo.save(match { it.reservedStock == 3 }) }
                verify { publisher.publish(ofType<StockReservedEvent>()) }
            }
        }
        
        context("when stock is insufficient") {
            it("should fail reservation") {
                val item = InventoryItem("prod-1", currentStock = 2, reservedStock = 0)
                every { repo.findByProductId("prod-1") } returns item
                
                val result = service.reserve("order-1", "prod-1", 5)
                
                result.success shouldBe false
                result.error shouldBe "Insufficient stock"
                
                verify(exactly = 0) { repo.save(any()) }
            }
        }
        
        context("property-based") {
            it("reserved stock should never exceed current stock") {
                checkAll(
                    Arb.int(1..100),
                    Arb.int(1..100)
                ) { currentStock, requestedQty ->
                    val item = InventoryItem("prod-1", currentStock = currentStock, reservedStock = 0)
                    every { repo.findByProductId("prod-1") } returns item
                    every { repo.save(any()) } answers { firstArg() }
                    every { publisher.publish(any()) } just Runs
                    
                    val result = service.reserve("order-1", "prod-1", requestedQty)
                    
                    if (requestedQty <= currentStock) {
                        result.success shouldBe true
                    } else {
                        result.success shouldBe false
                    }
                }
            }
        }
    }
})

data class InventoryItem(val productId: String, val currentStock: Int, val reservedStock: Int)
data class ReservationResult(val success: Boolean, val reservationId: String? = null, val error: String? = null)
data class StockReservedEvent(val orderId: String, val productId: String, val quantity: Int)

interface InventoryRepository {
    fun findByProductId(productId: String): InventoryItem?
    fun save(item: InventoryItem): InventoryItem
}

class InventoryServiceImpl(private val repo: InventoryRepository, private val publisher: EventPublisher) {
    fun reserve(orderId: String, productId: String, qty: Int): ReservationResult {
        val item = repo.findByProductId(productId) ?: return ReservationResult(false, error = "Product not found")
        if (item.currentStock < qty) return ReservationResult(false, error = "Insufficient stock")
        repo.save(item.copy(reservedStock = item.reservedStock + qty))
        publisher.publish(StockReservedEvent(orderId, productId, qty))
        return ReservationResult(true, java.util.UUID.randomUUID().toString())
    }
}
```

---

## สรุป Part 92

```
✅ Testing Pyramid: Unit (70%) → Integration (20%) → E2E (10%)
✅ Kotest BehaviorSpec: Given/When/Then BDD style
✅ shouldBe/shouldNotBe/shouldContain: Kotest matchers
✅ shouldThrow<ExceptionType>: exception assertion
✅ mockk<T>(): create mock
✅ every { }.returns()/just Runs: stub behavior
✅ verify { }: verify interactions
✅ verifyOrder { }: verify call order
✅ confirmVerified: no unexpected calls
✅ slot<T>()/capture: capture arguments
✅ spyk: spy on real object
✅ clearAllMocks/beforeEach: reset mocks between tests
✅ Property-based testing: checkAll { a, b -> ... }
✅ Arb.bigDecimal/int/string: generate test data
✅ Arb.bind: compose arbitraries
✅ @ParameterizedTest + @CsvSource: table-driven tests
✅ @MethodSource: complex parameterized data
✅ Testcontainers: real DB/Kafka in Docker
✅ DynamicPropertySource: override properties from containers
✅ @SpringBootTest + @Transactional: integration test base
✅ TestRestTemplate: REST API testing
✅ Gatling: load testing with DSL
✅ scenario/exec/http: Gatling simulation
✅ rampUsersPerSec/constantUsersPerSec: load profile
✅ assertions: SLA verification
✅ Pitest: mutation testing
✅ mutationThreshold: fail CI if score too low
✅ Boundary testing: kills CONDITIONALS_BOUNDARY mutations
✅ DescribeSpec: nested describe/context/it structure
```

---

*Part 92/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
