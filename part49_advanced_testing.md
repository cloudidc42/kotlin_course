# Part 49: Advanced Testing ด้วย Testcontainers และ ArchUnit

## สารบัญ
1. [Testcontainers](#testcontainers)
2. [Contract Testing with Spring Cloud Contract](#contract-testing)
3. [ArchUnit: Architecture Testing](#archunit)
4. [Mutation Testing with PIT](#mutation-testing)
5. [Performance Testing](#performance-testing)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Testcontainers

Testcontainers ช่วยให้เรา run Docker containers จริงๆ ใน integration tests — ไม่ต้อง mock database อีกต่อไป

```kotlin
// build.gradle.kts
dependencies {
    testImplementation("org.testcontainers:testcontainers:1.19.8")
    testImplementation("org.testcontainers:postgresql:1.19.8")
    testImplementation("org.testcontainers:kafka:1.19.8")
    testImplementation("org.testcontainers:redis:1.19.8")
    testImplementation("org.testcontainers:elasticsearch:1.19.8")
    testImplementation("org.testcontainers:junit-jupiter:1.19.8")
}
```

### PostgreSQL Container

```kotlin
@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class UserRepositoryIntegrationTest {
    
    companion object {
        @Container
        @JvmStatic
        val postgres = PostgreSQLContainer<Nothing>("postgres:16-alpine").apply {
            withDatabaseName("testdb")
            withUsername("test")
            withPassword("test")
            withInitScript("db/init.sql")  // optional initialization
        }
        
        @DynamicPropertySource
        @JvmStatic
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
        }
    }
    
    @Autowired
    lateinit var userRepository: UserJpaRepository
    
    @Test
    fun `should save and retrieve user`() {
        val user = UserEntity(
            id = java.util.UUID.randomUUID().toString(),
            username = "johndoe",
            email = "john@example.com",
            firstName = "John",
            lastName = "Doe",
            passwordHash = "hashed",
            role = "USER"
        )
        
        val saved = userRepository.save(user)
        val found = userRepository.findById(saved.id)
        
        assertThat(found).isPresent
        assertThat(found.get().username).isEqualTo("johndoe")
        assertThat(found.get().email).isEqualTo("john@example.com")
    }
    
    @Test
    fun `should find users by role`() {
        userRepository.deleteAll()
        
        val users = listOf(
            UserEntity(id = java.util.UUID.randomUUID().toString(), username = "admin1", email = "admin1@example.com", 
                firstName = "Admin", lastName = "One", passwordHash = "hash", role = "ADMIN"),
            UserEntity(id = java.util.UUID.randomUUID().toString(), username = "user1", email = "user1@example.com",
                firstName = "User", lastName = "One", passwordHash = "hash", role = "USER"),
            UserEntity(id = java.util.UUID.randomUUID().toString(), username = "user2", email = "user2@example.com",
                firstName = "User", lastName = "Two", passwordHash = "hash", role = "USER")
        )
        userRepository.saveAll(users)
        
        val admins = userRepository.findByRole("ADMIN")
        val regularUsers = userRepository.findByRole("USER")
        
        assertThat(admins).hasSize(1)
        assertThat(regularUsers).hasSize(2)
    }
}
```

### Redis Container

```kotlin
@SpringBootTest
@Testcontainers
class RedisCacheIntegrationTest {
    
    companion object {
        @Container
        @JvmStatic
        val redis = GenericContainer<Nothing>("redis:7-alpine").apply {
            withExposedPorts(6379)
        }
        
        @DynamicPropertySource
        @JvmStatic
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("spring.data.redis.host") { redis.host }
            registry.add("spring.data.redis.port") { redis.getMappedPort(6379) }
        }
    }
    
    @Autowired
    lateinit var productService: ProductCacheService
    
    @Autowired
    lateinit var cacheManager: CacheManager
    
    @Test
    fun `should cache product after first fetch`() {
        val productId = "prod-001"
        
        // First call - cache miss
        val product1 = productService.findById(productId)
        
        // Second call - cache hit (verify no DB call)
        val product2 = productService.findById(productId)
        
        assertThat(product1).isEqualTo(product2)
        
        val cache = cacheManager.getCache("products")
        assertThat(cache?.get(productId)).isNotNull
    }
    
    @Test
    fun `should invalidate cache on update`() {
        val productId = "prod-002"
        
        // Populate cache
        productService.findById(productId)
        
        // Update invalidates cache
        productService.update(productId, UpdateRequest("New Name", 99.99))
        
        val cache = cacheManager.getCache("products")
        // Cache should be evicted
        assertThat(cache?.get(productId)).isNull()
    }
}

data class UpdateRequest(val name: String, val price: Double)

interface ProductCacheService {
    fun findById(id: String): Any?
    fun update(id: String, request: UpdateRequest): Any?
}
```

### Kafka Container

```kotlin
@SpringBootTest
@Testcontainers
class OrderEventIntegrationTest {
    
    companion object {
        @Container
        @JvmStatic
        val kafka = KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"))
        
        @DynamicPropertySource
        @JvmStatic
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers)
        }
    }
    
    @Autowired
    lateinit var orderProducer: OrderEventProducer2
    
    @Autowired
    lateinit var kafkaTemplate: KafkaTemplate<String, String>
    
    @Test
    fun `should publish and consume order event`() {
        val latch = java.util.concurrent.CountDownLatch(1)
        val receivedEvents = mutableListOf<String>()
        
        // Set up consumer
        val consumer = KafkaConsumer<String, String>(mapOf(
            ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG to kafka.bootstrapServers,
            ConsumerConfig.GROUP_ID_CONFIG to "test-group",
            ConsumerConfig.AUTO_OFFSET_RESET_CONFIG to "earliest",
            ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG to StringDeserializer::class.java,
            ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG to StringDeserializer::class.java
        ))
        consumer.subscribe(listOf("orders.created"))
        
        // Produce event
        orderProducer.publishOrderCreated("order-001", "customer-001", 99.99)
        
        // Consume with timeout
        val deadline = System.currentTimeMillis() + 10_000
        while (System.currentTimeMillis() < deadline && receivedEvents.isEmpty()) {
            val records = consumer.poll(java.time.Duration.ofMillis(500))
            records.forEach { receivedEvents.add(it.value()) }
        }
        
        consumer.close()
        
        assertThat(receivedEvents).hasSize(1)
        assertThat(receivedEvents[0]).contains("order-001")
    }
}

interface OrderEventProducer2 {
    fun publishOrderCreated(orderId: String, customerId: String, amount: Double)
}
```

### Multiple Containers with Docker Compose

```kotlin
@SpringBootTest
@Testcontainers
class FullIntegrationTest {
    
    companion object {
        // หลาย containers ทำงานพร้อมกัน
        @Container
        @JvmStatic
        val postgres = PostgreSQLContainer<Nothing>("postgres:16-alpine")
        
        @Container
        @JvmStatic
        val redis = GenericContainer<Nothing>("redis:7-alpine")
            .withExposedPorts(6379)
        
        @Container
        @JvmStatic
        val kafka = KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.6.0"))
        
        @DynamicPropertySource
        @JvmStatic
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
            registry.add("spring.data.redis.host") { redis.host }
            registry.add("spring.data.redis.port") { redis.getMappedPort(6379) }
            registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers)
        }
    }
    
    // Test full workflows with real infrastructure
}

// หรือใช้ Docker Compose file
@SpringBootTest
@Testcontainers
class DockerComposeTest {
    
    companion object {
        @Container
        @JvmStatic
        val environment = DockerComposeContainer<Nothing>(
            java.io.File("src/test/resources/docker-compose-test.yml")
        ).apply {
            withExposedService("postgres", 5432, Wait.forListeningPort())
            withExposedService("redis", 6379, Wait.forListeningPort())
        }
    }
}
```

---

## Contract Testing with Spring Cloud Contract

Contract testing ให้ Producer และ Consumer ตกลง API specification ก่อนพัฒนา

```groovy
// contracts/order/shouldCreateOrder.groovy
import org.springframework.cloud.contract.spec.Contract

Contract.make {
    description "should create new order"
    
    request {
        method POST()
        url '/api/v1/orders'
        headers {
            contentType(applicationJson())
            header("Authorization", matching("Bearer .+"))
        }
        body([
            customerId: "customer-001",
            items: [[
                productId: "prod-001",
                quantity: 2,
                unitPrice: 49.99
            ]]
        ])
    }
    
    response {
        status CREATED()
        headers {
            contentType(applicationJson())
        }
        body([
            id: anyNonEmptyString(),
            customerId: "customer-001",
            status: "PENDING",
            totalAmount: 99.98,
            items: [[
                productId: "prod-001",
                quantity: 2,
                unitPrice: 49.99
            ]]
        ])
    }
}
```

```kotlin
// Producer side: verify contract
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@AutoConfigureMockMvc
class OrderContractTest : ContractVerifierBase() {
    
    @Autowired
    lateinit var mockMvc: MockMvc
    
    override fun setup() {
        RestAssuredMockMvc.mockMvc(mockMvc)
    }
}

// Consumer side: use generated stubs
@SpringBootTest
@AutoConfigureStubRunner(
    ids = ["com.example:order-service:+:stubs:8090"],
    stubsMode = StubRunnerProperties.StubsMode.LOCAL
)
class OrderClientContractTest {
    
    @Autowired
    lateinit var orderClient: OrderServiceClient
    
    @Test
    fun `should create order via stub`() {
        val response = orderClient.createOrder(
            CreateOrderRequest(
                customerId = "customer-001",
                items = listOf(OrderItem("prod-001", 2, 49.99))
            )
        )
        
        assertThat(response.status).isEqualTo("PENDING")
        assertThat(response.totalAmount).isEqualTo(99.98)
    }
}

data class CreateOrderRequest(val customerId: String, val items: List<OrderItem>)
data class OrderItem(val productId: String, val quantity: Int, val unitPrice: Double)
interface OrderServiceClient {
    fun createOrder(request: CreateOrderRequest): Any
}
```

---

## ArchUnit: Architecture Testing

ArchUnit ทดสอบ architecture rules — มั่นใจว่า code ตาม Clean Architecture

```kotlin
// build.gradle.kts
testImplementation("com.tngtech.archunit:archunit-junit5:1.3.0")

// src/test/kotlin/ArchitectureTest.kt
@AnalyzeClasses(packages = ["com.example.myapp"])
class ArchitectureTest {
    
    // Domain layer ต้องไม่ depend on Infrastructure
    @ArchTest
    val domainShouldNotDependOnInfrastructure: ArchRule = noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAPackage("..infrastructure..")
    
    // Domain ต้องไม่ depend on Spring
    @ArchTest
    val domainShouldNotDependOnSpring: ArchRule = noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAPackage("org.springframework..")
    
    // Application layer ต้องไม่ depend on Infrastructure
    @ArchTest
    val applicationShouldNotDependOnInfrastructure: ArchRule = noClasses()
        .that().resideInAPackage("..application..")
        .should().dependOnClassesThat()
        .resideInAPackage("..infrastructure..")
    
    // UseCase ต้องอยู่ใน application package
    @ArchTest
    val useCasesShouldBeInApplicationPackage: ArchRule = classes()
        .that().haveSimpleNameEndingWith("UseCase")
        .should().resideInAPackage("..application..")
    
    // Repository interfaces ต้องอยู่ใน domain
    @ArchTest
    val repositoryInterfacesShouldBeInDomain: ArchRule = classes()
        .that().haveSimpleNameEndingWith("Repository")
        .and().areInterfaces()
        .should().resideInAPackage("..domain..")
    
    // Repository implementations ต้องอยู่ใน infrastructure
    @ArchTest
    val repositoryImplementationsShouldBeInInfrastructure: ArchRule = classes()
        .that().haveSimpleNameEndingWith("RepositoryImpl")
        .should().resideInAPackage("..infrastructure..")
    
    // Controller ต้องอยู่ใน presentation
    @ArchTest
    val controllersShouldBeInPresentation: ArchRule = classes()
        .that().areAnnotatedWith(RestController::class.java)
        .should().resideInAPackage("..presentation..")
    
    // ห้าม field injection (@Autowired บน field)
    @ArchTest
    val noFieldInjection: ArchRule = noFields()
        .should().beAnnotatedWith(Autowired::class.java)
        .`as`("Use constructor injection instead of field injection")
    
    // ห้าม System.out.println (ใช้ Logger แทน)
    @ArchTest
    val noSystemOutPrintln: ArchRule = noClasses()
        .should().callMethod(System::class.java, "out")
        .`as`("Use SLF4J Logger instead of System.out")
    
    // Layered architecture
    @ArchTest
    val layeredArchitecture: ArchRule = layeredArchitecture()
        .consideringAllDependencies()
        .layer("Presentation").definedBy("..presentation..")
        .layer("Application").definedBy("..application..")
        .layer("Domain").definedBy("..domain..")
        .layer("Infrastructure").definedBy("..infrastructure..")
        .whereLayer("Presentation").mayOnlyBeAccessedByLayers("Infrastructure")
        .whereLayer("Application").mayOnlyBeAccessedByLayers("Presentation", "Infrastructure")
        .whereLayer("Domain").mayOnlyBeAccessedByLayers("Application", "Infrastructure")
        .whereLayer("Infrastructure").mayNotBeAccessedByAnyLayer()
    
    // ทุก class ใน domain ต้องไม่เป็น data class ที่ mutable (immutable domain)
    @ArchTest
    val domainEntitiesShouldBeFinal: ArchRule = classes()
        .that().resideInAPackage("..domain.model..")
        .should().beFinal()
    
    // Service annotation ต้องไม่ใช้ใน domain
    @ArchTest
    val domainShouldNotUseServiceAnnotation: ArchRule = noClasses()
        .that().resideInAPackage("..domain..")
        .should().beAnnotatedWith(Service::class.java)
}
```

### Cyclic Dependencies Check

```kotlin
@AnalyzeClasses(packages = ["com.example.myapp"])
class CyclicDependencyTest {
    
    @ArchTest
    val noCyclicDependencies: ArchRule = slices()
        .matching("com.example.myapp.(*)..")
        .should().beFreeOfCycles()
    
    // ตรวจ package-level cycles
    @ArchTest
    val noPackageCycles: ArchRule = slices()
        .matching("com.example.myapp.(*)")
        .namingSlices("module '$1'")
        .should().beFreeOfCycles()
}
```

### Custom Architecture Rules

```kotlin
// Custom rules using ArchCondition
class ArchitectureRules {
    
    companion object {
        
        fun haveAllFieldsFinal() = object : ArchCondition<JavaClass>("have all fields final") {
            override fun check(item: JavaClass, events: ConditionEvents) {
                item.fields
                    .filter { !it.modifiers.contains(JavaModifier.STATIC) }
                    .filter { !it.modifiers.contains(JavaModifier.FINAL) }
                    .forEach { field ->
                        events.add(SimpleConditionEvent.violated(
                            field,
                            "Field ${field.name} in ${item.name} is not final"
                        ))
                    }
            }
        }
        
        fun notThrowGenericExceptions() = object : ArchCondition<JavaMethod>("not throw generic exceptions") {
            override fun check(item: JavaMethod, events: ConditionEvents) {
                val genericExceptions = setOf(
                    "java.lang.RuntimeException",
                    "java.lang.Exception"
                )
                item.throwsClause
                    .filter { it.name in genericExceptions }
                    .forEach { exc ->
                        events.add(SimpleConditionEvent.violated(
                            item,
                            "Method ${item.name} throws generic ${exc.name}"
                        ))
                    }
            }
        }
    }
}

@AnalyzeClasses(packages = ["com.example.myapp"])
class CustomRulesTest {
    
    @ArchTest
    val domainValueObjectsShouldBeImmutable: ArchRule = classes()
        .that().resideInAPackage("..domain.vo..")
        .should(ArchitectureRules.haveAllFieldsFinal())
    
    @ArchTest  
    val servicesShouldNotThrowGenericExceptions: ArchRule = methods()
        .that().areDeclaredInClassesThat().haveSimpleNameEndingWith("Service")
        .should(ArchitectureRules.notThrowGenericExceptions())
}
```

---

## Mutation Testing with PIT

Mutation testing วัดคุณภาพของ test suite จริงๆ ว่า tests สามารถจับ bugs ได้แค่ไหน

```kotlin
// build.gradle.kts
plugins {
    id("info.solidsoft.pitest") version "1.15.0"
}

pitest {
    targetClasses = listOf("com.example.myapp.domain.*", "com.example.myapp.application.*")
    targetTests = listOf("com.example.myapp.*Test", "com.example.myapp.*Spec")
    
    mutators = listOf("DEFAULTS")  // หรือ "STRONGER" สำหรับ thorough testing
    
    outputFormats = listOf("HTML", "XML")
    
    threads = 4
    
    // Minimum mutation score (จะ fail build ถ้าต่ำกว่า)
    mutationThreshold = 80
    coverageThreshold = 90
    
    excludedClasses = listOf(
        "com.example.myapp.*.config.*",
        "com.example.myapp.*.dto.*",
        "com.example.myapp.*Application"
    )
}
```

```kotlin
// ตัวอย่าง: function ที่ mutation testing จะทดสอบ
class OrderPricingService {
    
    fun calculateDiscount(subtotal: Double, coupon: Coupon?): Double {
        if (coupon == null) return 0.0
        if (!coupon.isValid()) return 0.0
        
        return when (coupon.type) {
            CouponType.PERCENTAGE -> subtotal * (coupon.value / 100.0)
            CouponType.FIXED -> minOf(coupon.value, subtotal)
        }
    }
    
    fun calculateTotal(items: List<OrderItem2>, coupon: Coupon?): OrderTotal {
        val subtotal = items.sumOf { it.quantity * it.unitPrice }
        val discount = calculateDiscount(subtotal, coupon)
        val tax = (subtotal - discount) * 0.07  // 7% VAT
        val total = subtotal - discount + tax
        
        return OrderTotal(subtotal, discount, tax, total)
    }
}

data class OrderItem2(val productId: String, val quantity: Int, val unitPrice: Double)
data class OrderTotal(val subtotal: Double, val discount: Double, val tax: Double, val total: Double)
data class Coupon(val code: String, val type: CouponType, val value: Double, val expiresAt: java.time.LocalDate) {
    fun isValid() = expiresAt >= java.time.LocalDate.now()
}
enum class CouponType { PERCENTAGE, FIXED }

// Tests ที่ต้องครอบคลุม mutations ทั้งหมด
class OrderPricingServiceTest {
    
    private val service = OrderPricingService()
    
    @Test
    fun `no coupon returns zero discount`() {
        assertThat(service.calculateDiscount(100.0, null)).isEqualTo(0.0)
    }
    
    @Test
    fun `expired coupon returns zero discount`() {
        val expired = Coupon("SAVE10", CouponType.PERCENTAGE, 10.0, 
            java.time.LocalDate.now().minusDays(1))
        assertThat(service.calculateDiscount(100.0, expired)).isEqualTo(0.0)
    }
    
    @Test
    fun `percentage coupon calculates correctly`() {
        val coupon = Coupon("SAVE10", CouponType.PERCENTAGE, 10.0,
            java.time.LocalDate.now().plusDays(30))
        assertThat(service.calculateDiscount(100.0, coupon)).isEqualTo(10.0)
    }
    
    @Test
    fun `fixed coupon does not exceed subtotal`() {
        val coupon = Coupon("FLAT50", CouponType.FIXED, 50.0,
            java.time.LocalDate.now().plusDays(30))
        assertThat(service.calculateDiscount(30.0, coupon)).isEqualTo(30.0)  // not 50
        assertThat(service.calculateDiscount(100.0, coupon)).isEqualTo(50.0)
    }
    
    @Test
    fun `total calculation includes tax`() {
        val items = listOf(OrderItem2("prod-1", 2, 50.0))  // subtotal = 100
        val total = service.calculateTotal(items, null)
        
        assertThat(total.subtotal).isEqualTo(100.0)
        assertThat(total.discount).isEqualTo(0.0)
        assertThat(total.tax).isEqualTo(7.0)  // 7%
        assertThat(total.total).isEqualTo(107.0)
    }
}
```

---

## Performance Testing

```kotlin
// Gatling-style performance test in Kotlin
class OrderApiPerformanceTest {
    
    @Test
    fun `API should handle 100 concurrent requests`() = runBlocking {
        val url = "http://localhost:8080/api/v1/orders"
        val concurrency = 100
        val successCount = AtomicInteger(0)
        val errorCount = AtomicInteger(0)
        val times = ConcurrentLinkedQueue<Long>()
        
        val jobs = (1..concurrency).map {
            launch(Dispatchers.IO) {
                val start = System.nanoTime()
                try {
                    // HTTP call here (using Ktor client or OkHttp)
                    successCount.incrementAndGet()
                } catch (e: Exception) {
                    errorCount.incrementAndGet()
                } finally {
                    times.add(System.nanoTime() - start)
                }
            }
        }
        
        jobs.joinAll()
        
        val sortedTimes = times.sorted()
        val p50 = sortedTimes[sortedTimes.size / 2] / 1_000_000  // ms
        val p95 = sortedTimes[(sortedTimes.size * 0.95).toInt()] / 1_000_000
        val p99 = sortedTimes[(sortedTimes.size * 0.99).toInt()] / 1_000_000
        
        println("Success: ${successCount.get()}, Error: ${errorCount.get()}")
        println("P50: ${p50}ms, P95: ${p95}ms, P99: ${p99}ms")
        
        assertThat(successCount.get()).isGreaterThanOrEqualTo(95)
        assertThat(p95).isLessThan(500)  // P95 < 500ms
    }
}

// JMH Benchmark
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MICROSECONDS)
@State(Scope.Thread)
@Warmup(iterations = 3, time = 1)
@Measurement(iterations = 5, time = 1)
@Fork(1)
open class OrderCalculationBenchmark {
    
    private lateinit var items: List<OrderItem2>
    private lateinit var service: OrderPricingService
    
    @Setup
    fun setup() {
        service = OrderPricingService()
        items = (1..10).map { OrderItem2("prod-$it", it, it * 10.0) }
    }
    
    @Benchmark
    fun calculateTotal(): OrderTotal {
        return service.calculateTotal(items, null)
    }
    
    @Benchmark
    fun calculateTotalWithCoupon(): OrderTotal {
        val coupon = Coupon("SAVE10", CouponType.PERCENTAGE, 10.0,
            java.time.LocalDate.now().plusDays(30))
        return service.calculateTotal(items, coupon)
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise 1: Add Testcontainers for a complete E2E test
// Exercise 2: Create ArchUnit rules for your project's architecture

// Exercise 1 template:
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class OrderE2ETest {
    
    companion object {
        @Container
        @JvmStatic
        val postgres = PostgreSQLContainer<Nothing>("postgres:16-alpine")
        
        @Container
        @JvmStatic
        val redis = GenericContainer<Nothing>("redis:7-alpine").withExposedPorts(6379)
        
        @DynamicPropertySource
        @JvmStatic
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
            registry.add("spring.data.redis.host") { redis.host }
            registry.add("spring.data.redis.port") { redis.getMappedPort(6379) }
        }
    }
    
    @LocalServerPort
    var port: Int = 0
    
    @Test
    fun `complete order flow`() {
        val baseUrl = "http://localhost:$port"
        
        // 1. Create user
        // 2. Login to get JWT
        // 3. Create product  
        // 4. Create order
        // 5. Process payment
        // 6. Verify order status
        TODO("Implement complete E2E test")
    }
}

// Exercise 2: Add custom ArchUnit rule
// Rule: All DTO classes should have @Schema annotation for OpenAPI docs
// Rule: All @RestController classes should have @RequestMapping starting with /api/v
```

---

## สรุป Part 49

```
✅ Testcontainers: run real databases in tests
✅ PostgreSQL Container: real DB integration tests
✅ Redis Container: cache integration tests  
✅ Kafka Container: message queue integration tests
✅ Docker Compose Container: multi-service tests
✅ Contract Testing: consumer-driven contract verification
✅ ArchUnit: architecture rule enforcement
✅ Layered Architecture rule: enforce layer boundaries
✅ No field injection rule: promote constructor injection
✅ Cyclic dependency detection: prevent coupling
✅ Custom ArchConditions: project-specific rules
✅ Mutation Testing with PIT: measure test quality
✅ JMH Benchmarks: micro-benchmark performance
✅ Concurrent performance tests: validate under load
```

---

*Part 49/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
