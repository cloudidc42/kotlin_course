# Part 90: Reactive Programming ด้วย Spring WebFlux

## สารบัญ
1. [Reactive Programming แนวคิด](#reactive-programming-แนวคิด)
2. [Project Reactor: Mono & Flux](#project-reactor-mono--flux)
3. [Spring WebFlux](#spring-webflux)
4. [Reactive Database ด้วย R2DBC](#reactive-database-ด้วย-r2dbc)
5. [Reactive Testing](#reactive-testing)
6. [Backpressure & Error Handling](#backpressure--error-handling)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Reactive Programming แนวคิด

```
Reactive vs Traditional (Blocking):

Traditional (Servlet-based):
Thread 1: request → DB query (WAITING) → response    ← thread blocked!
Thread 2: request → HTTP call (WAITING) → response   ← thread blocked!
Thread 3: request → waiting for thread...            ← queue!

Problems:
- 1 request = 1 thread
- Thread pool เต็มง่าย (default 200 threads)
- Memory overhead สูง (~1MB/thread)
- C10K problem: 10,000 concurrent connections

Reactive (Non-blocking):
Thread 1: request → DB query (register callback) → other work
Thread 1: DB query done → callback → response
Thread 1: HTTP call (register callback) → other work  
Thread 1: HTTP response → callback → response

Benefits:
✅ เดิม: 200 req concurrent, ใหม่: 10,000+ req
✅ น้อย thread, ใช้ CPU ได้มีประสิทธิภาพกว่า
✅ เหมาะกับ I/O-heavy workloads
✅ Backpressure: consumer controls flow rate

Reactive Streams Specification:
Publisher  → produces items
Subscriber → consumes items
Subscription → connects, controls request rate
Processor  → both publisher & subscriber

Project Reactor (Spring's implementation):
Mono<T>  → 0 or 1 item (like Optional but async)
Flux<T>  → 0 to N items (stream of data)

เมื่อไหร่ใช้ Reactive:
✅ Many concurrent connections (API gateway, chat)
✅ Stream processing (real-time data)
✅ Microservice fan-out (call 5 services parallel)
✅ SSE (Server-Sent Events)

เมื่อไหร่ใช้ Coroutines แทน:
✅ Business logic complexity (easier to read)
✅ CPU-bound work
✅ Existing coroutine codebase
(Spring WebFlux + Kotlin Coroutines ทำงานร่วมกันได้)
```

---

## Project Reactor: Mono & Flux

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    implementation("org.springframework.boot:spring-boot-starter-data-r2dbc")
    implementation("io.r2dbc:r2dbc-postgresql:1.0.4.RELEASE")
    implementation("org.springframework.boot:spring-boot-starter-data-redis-reactive")
    testImplementation("io.projectreactor:reactor-test")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}

import reactor.core.publisher.Mono
import reactor.core.publisher.Flux
import java.time.Duration

// Mono: 0 or 1 item
fun monoExamples() {
    // สร้าง Mono
    val mono1 = Mono.just("Hello")
    val mono2 = Mono.empty<String>()
    val mono3 = Mono.error<String>(RuntimeException("Error!"))
    val mono4 = Mono.fromCallable { expensiveComputation() }
    val mono5 = Mono.fromSupplier { "lazy value" }
    val mono6 = Mono.defer { Mono.just(System.currentTimeMillis().toString()) }
    
    // Transform
    val upper = mono1.map { it.uppercase() }
    val length = mono1.flatMap { text -> Mono.just(text.length) }
    val withDefault = mono2.defaultIfEmpty("default")
    val withSwitch = mono2.switchIfEmpty(Mono.just("fallback"))
    
    // Filter
    val filtered = mono1.filter { it.length > 3 }
    
    // Combine
    val combined = Mono.zip(
        Mono.just("Hello"),
        Mono.just(42)
    ) { text, number -> "$text - $number" }
    
    // Error handling
    val recovered = mono3
        .onErrorReturn("recovery value")
        .onErrorResume { e -> Mono.just("resumed: ${e.message}") }
        .onErrorMap { e -> IllegalStateException("Wrapped: ${e.message}", e) }
    
    // Subscribe (terminal operation)
    mono1.subscribe(
        { value -> println("Got: $value") },
        { error -> println("Error: $error") },
        { println("Completed!") }
    )
    
    // Block (อย่าใช้ใน production reactive code)
    val result = mono1.block()
    
    println(combined)
}

// Flux: 0 to N items
fun fluxExamples() {
    // สร้าง Flux
    val flux1 = Flux.just(1, 2, 3, 4, 5)
    val flux2 = Flux.fromList(listOf("a", "b", "c"))
    val flux3 = Flux.fromIterable(1..100)
    val flux4 = Flux.range(1, 10)  // 1 to 10
    val flux5 = Flux.interval(Duration.ofSeconds(1))  // infinite tick every 1s
    val flux6 = Flux.error<Int>(RuntimeException("flux error"))
    
    // Transform each item
    val doubled = flux1.map { it * 2 }
    val filtered = flux1.filter { it % 2 == 0 }
    val strings = flux1.map { it.toString() }
    
    // FlatMap: each item → Publisher
    val expanded = flux1.flatMap { n ->
        Flux.range(1, n).map { "$n:$it" }
    }
    
    // Collect
    val list: Mono<List<Int>> = flux1.collectList()
    val map: Mono<Map<Int, Int>> = flux1.collectMap({ it }, { it * it })
    val sorted = flux1.sort()
    val count: Mono<Long> = flux1.count()
    
    // Aggregate
    val sum: Mono<Int> = flux1.reduce(0) { acc, v -> acc + v }
    val scan = flux1.scan(0) { acc, v -> acc + v }  // running total
    
    // Combine streams
    val merged = Flux.merge(flux1, Flux.just(10, 11, 12))  // interleave
    val concatenated = Flux.concat(flux1, Flux.just(10, 11, 12))  // sequential
    val zipped = Flux.zip(flux1, flux2) { n, s -> "$n-$s" }  // pair by index
    
    // Windowing / Grouping
    val windows: Flux<Flux<Int>> = flux1.window(2)  // chunks of 2
    val buffers: Flux<List<Int>> = flux1.buffer(3)  // collect 3 then emit List
    val grouped: Flux<reactor.core.publisher.GroupedFlux<Boolean, Int>> = 
        flux1.groupBy { it % 2 == 0 }
    
    // Take / Skip
    val first3 = flux1.take(3)
    val skip2 = flux1.skip(2)
    val until = flux1.takeUntil { it > 3 }
    
    // Time operations
    val withTimeout = flux1.timeout(Duration.ofSeconds(5))
    val delayed = flux1.delayElements(Duration.ofMillis(100))
    val debounced = flux1.debounce(Duration.ofMillis(300))
    val sampled = flux5.sample(Duration.ofSeconds(1))  // one per second
    
    // Error handling
    val withRetry = flux1.retry(3)
    val withRetryWhen = flux1.retryWhen(
        reactor.util.retry.Retry.backoff(3, Duration.ofSeconds(1))
            .maxBackoff(Duration.ofSeconds(10))
    )
    
    // subscribe
    flux1.subscribe(
        { value -> print("$value ") },
        { error -> println("Error: $error") },
        { println("Done!") }
    )
    
    println()
}

// Hot vs Cold Publishers
fun hotVsCold() {
    // Cold: each subscriber gets its own stream (re-executes)
    val cold = Flux.defer {
        println("Started cold producer")
        Flux.range(1, 5)
    }
    cold.subscribe { print("Sub1: $it ") }   // prints "Started..." and 1-5
    cold.subscribe { print("Sub2: $it ") }   // prints "Started..." again
    
    // Hot: shared stream, late subscribers miss early items
    val hotSource = reactor.core.publisher.Sinks.many().multicast().onBackpressureBuffer<Int>()
    val hot = hotSource.asFlux()
    
    hot.subscribe { print("Sub1: $it ") }
    hotSource.tryEmitNext(1)  // Sub1 gets 1
    hotSource.tryEmitNext(2)  // Sub1 gets 2
    
    hot.subscribe { print("Sub2: $it ") }  // Sub2 misses 1, 2
    hotSource.tryEmitNext(3)  // Both get 3
    hotSource.tryEmitNext(4)  // Both get 4
    
    // Publish: cold → hot (unicast to all subscribers)
    val published = cold.publish()  // ConnectableFlux
    published.subscribe { print("A: $it ") }
    published.subscribe { print("B: $it ") }
    published.connect()  // starts the cold producer once
    
    // Share: cold → hot with auto-connect when first subscriber
    val shared = cold.share()
    
    println()
}

fun expensiveComputation(): String = "computed"
```

---

## Spring WebFlux

```kotlin
// Functional routing (alternative to @Controller)
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.reactive.function.server.*
import org.springframework.http.MediaType
import reactor.core.publisher.Mono

@Configuration
class ProductRouter(private val productHandler: ProductHandler) {
    
    @Bean
    fun productRoutes(): RouterFunction<ServerResponse> = router {
        "/api/v1/products".nest {
            GET("", productHandler::listProducts)
            GET("/{id}", productHandler::getProduct)
            POST("", productHandler::createProduct)
            PUT("/{id}", productHandler::updateProduct)
            DELETE("/{id}", productHandler::deleteProduct)
            GET("/search", productHandler::searchProducts)
            
            "/stream".nest {
                GET("/prices", accept(MediaType.TEXT_EVENT_STREAM), productHandler::streamPrices)
                GET("/inventory", accept(MediaType.TEXT_EVENT_STREAM), productHandler::streamInventory)
            }
        }
    }
}

@org.springframework.stereotype.Component
class ProductHandler(
    private val productService: ReactiveProductService,
    private val validator: org.springframework.validation.Validator
) {
    
    fun listProducts(request: ServerRequest): Mono<ServerResponse> {
        val categoryId = request.queryParam("categoryId").orElse(null)
        val page = request.queryParam("page").map { it.toInt() }.orElse(0)
        val size = request.queryParam("size").map { it.toInt() }.orElse(20)
        
        return productService.findAll(categoryId, page, size)
            .flatMap { page ->
                ServerResponse.ok()
                    .contentType(MediaType.APPLICATION_JSON)
                    .bodyValue(page)
            }
    }
    
    fun getProduct(request: ServerRequest): Mono<ServerResponse> {
        val id = request.pathVariable("id")
        
        return productService.findById(id)
            .flatMap { product ->
                ServerResponse.ok()
                    .contentType(MediaType.APPLICATION_JSON)
                    .bodyValue(product)
            }
            .switchIfEmpty(
                ServerResponse.notFound().build()
            )
    }
    
    fun createProduct(request: ServerRequest): Mono<ServerResponse> {
        return request.bodyToMono(CreateProductDto::class.java)
            .switchIfEmpty(Mono.error(IllegalArgumentException("Request body required")))
            .flatMap { dto ->
                productService.create(dto)
            }
            .flatMap { product ->
                ServerResponse.status(org.springframework.http.HttpStatus.CREATED)
                    .contentType(MediaType.APPLICATION_JSON)
                    .bodyValue(product)
            }
    }
    
    fun updateProduct(request: ServerRequest): Mono<ServerResponse> {
        val id = request.pathVariable("id")
        
        return request.bodyToMono(UpdateProductDto::class.java)
            .flatMap { dto -> productService.update(id, dto) }
            .flatMap { product ->
                ServerResponse.ok()
                    .contentType(MediaType.APPLICATION_JSON)
                    .bodyValue(product)
            }
            .switchIfEmpty(ServerResponse.notFound().build())
    }
    
    fun deleteProduct(request: ServerRequest): Mono<ServerResponse> {
        val id = request.pathVariable("id")
        
        return productService.delete(id)
            .then(ServerResponse.noContent().build())
            .onErrorResume(NotFoundException::class.java) {
                ServerResponse.notFound().build()
            }
    }
    
    fun searchProducts(request: ServerRequest): Mono<ServerResponse> {
        val query = request.queryParam("q")
            .orElseThrow { IllegalArgumentException("Query parameter 'q' required") }
        
        return ServerResponse.ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(productService.search(query), ProductDto::class.java)
    }
    
    // SSE: Server-Sent Events
    fun streamPrices(request: ServerRequest): Mono<ServerResponse> {
        return ServerResponse.ok()
            .contentType(MediaType.TEXT_EVENT_STREAM)
            .body(productService.priceUpdates(), PriceUpdateDto::class.java)
    }
    
    fun streamInventory(request: ServerRequest): Mono<ServerResponse> {
        return ServerResponse.ok()
            .contentType(MediaType.TEXT_EVENT_STREAM)
            .body(productService.inventoryUpdates(), InventoryUpdateDto::class.java)
    }
}

// Alternative: @RestController style (เหมือน MVC แต่ reactive)
@org.springframework.web.bind.annotation.RestController
@org.springframework.web.bind.annotation.RequestMapping("/api/v2/products")
class ProductWebFluxController(
    private val productService: ReactiveProductService
) {
    
    @org.springframework.web.bind.annotation.GetMapping
    fun listProducts(
        @org.springframework.web.bind.annotation.RequestParam(defaultValue = "0") page: Int,
        @org.springframework.web.bind.annotation.RequestParam(defaultValue = "20") size: Int
    ): Mono<ProductPageDto> {
        return productService.findAll(null, page, size)
    }
    
    @org.springframework.web.bind.annotation.GetMapping("/{id}")
    fun getProduct(@org.springframework.web.bind.annotation.PathVariable id: String): Mono<ProductDto> {
        return productService.findById(id)
    }
    
    @org.springframework.web.bind.annotation.PostMapping
    @org.springframework.web.bind.annotation.ResponseStatus(org.springframework.http.HttpStatus.CREATED)
    fun createProduct(
        @org.springframework.web.bind.annotation.RequestBody dto: CreateProductDto
    ): Mono<ProductDto> {
        return productService.create(dto)
    }
    
    // Streaming endpoint
    @org.springframework.web.bind.annotation.GetMapping(
        "/stream",
        produces = [MediaType.TEXT_EVENT_STREAM_VALUE]
    )
    fun streamAllProducts(): Flux<ProductDto> {
        return productService.streamAll()
    }
    
    // WebFlux + Coroutines (suspend functions work directly)
    @org.springframework.web.bind.annotation.GetMapping("/coroutine/{id}")
    suspend fun getProductCoroutine(
        @org.springframework.web.bind.annotation.PathVariable id: String
    ): ProductDto {
        return productService.findById(id).awaitSingleOrNull()
            ?: throw NotFoundException("Product $id not found")
    }
}

// WebFilter: cross-cutting concerns
@org.springframework.stereotype.Component
class CorrelationIdFilter : org.springframework.web.server.WebFilter {
    
    override fun filter(
        exchange: org.springframework.web.server.ServerWebExchange,
        chain: org.springframework.web.server.WebFilterChain
    ): Mono<Void> {
        val correlationId = exchange.request.headers
            .getFirst("X-Correlation-ID") ?: java.util.UUID.randomUUID().toString()
        
        return chain.filter(exchange)
            .contextWrite(
                reactor.util.context.Context.of("correlationId", correlationId)
            )
            .doOnSuccess {
                exchange.response.headers.add("X-Correlation-ID", correlationId)
            }
    }
}

// WebExceptionHandler: global error handling
@org.springframework.stereotype.Component
@org.springframework.core.annotation.Order(-2)
class GlobalExceptionHandler(
    serverCodecConfigurer: org.springframework.http.codec.ServerCodecConfigurer
) : org.springframework.boot.autoconfigure.web.reactive.error.AbstractErrorWebExceptionHandler(
    org.springframework.boot.web.reactive.error.DefaultErrorAttributes(),
    org.springframework.web.reactive.config.WebFluxConfigurationSupport().webFluxContentTypeResolver().let {
        org.springframework.boot.autoconfigure.web.WebProperties.Resources()
    },
    null
) {
    
    init {
        this.setMessageWriters(serverCodecConfigurer.writers)
        this.setMessageReaders(serverCodecConfigurer.readers)
    }
    
    override fun getRoutingFunction(
        errorAttributes: org.springframework.boot.web.reactive.error.ErrorAttributes
    ): RouterFunction<ServerResponse> = router {
        RequestPredicates.all().invoke(this@GlobalExceptionHandler::handleError)
    }
    
    private fun handleError(request: ServerRequest): Mono<ServerResponse> {
        val error = getError(request)
        
        val (status, message) = when (error) {
            is NotFoundException -> org.springframework.http.HttpStatus.NOT_FOUND to error.message
            is IllegalArgumentException -> org.springframework.http.HttpStatus.BAD_REQUEST to error.message
            is org.springframework.security.access.AccessDeniedException ->
                org.springframework.http.HttpStatus.FORBIDDEN to "Access denied"
            else -> org.springframework.http.HttpStatus.INTERNAL_SERVER_ERROR to "Internal error"
        }
        
        return ServerResponse.status(status)
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(ErrorResponse(status.value(), message ?: "Unknown error"))
    }
}

data class ErrorResponse(val status: Int, val message: String)

// Service layer
interface ReactiveProductService {
    fun findAll(categoryId: String?, page: Int, size: Int): Mono<ProductPageDto>
    fun findById(id: String): Mono<ProductDto>
    fun create(dto: CreateProductDto): Mono<ProductDto>
    fun update(id: String, dto: UpdateProductDto): Mono<ProductDto>
    fun delete(id: String): Mono<Void>
    fun search(query: String): Flux<ProductDto>
    fun streamAll(): Flux<ProductDto>
    fun priceUpdates(): Flux<PriceUpdateDto>
    fun inventoryUpdates(): Flux<InventoryUpdateDto>
}

data class CreateProductDto(val name: String, val price: Double, val stockQuantity: Int, val categoryId: String)
data class UpdateProductDto(val name: String?, val price: Double?, val stockQuantity: Int?)
data class ProductDto(val id: String, val name: String, val price: Double)
data class ProductPageDto(val items: List<ProductDto>, val total: Long)
data class PriceUpdateDto(val productId: String, val newPrice: Double)
data class InventoryUpdateDto(val productId: String, val newStock: Int)
class NotFoundException(message: String) : RuntimeException(message)

import reactor.core.publisher.Flux as RFlux
```

---

## Reactive Database ด้วย R2DBC

```kotlin
// R2DBC (Reactive Relational Database Connectivity)
// ไม่ blocking เหมือน JPA/JDBC

// Entity
import org.springframework.data.annotation.Id
import org.springframework.data.relational.core.mapping.Table
import org.springframework.data.relational.core.mapping.Column
import org.springframework.data.r2dbc.repository.Query
import org.springframework.data.repository.reactive.ReactiveCrudRepository

@Table("products")
data class ProductEntity(
    @Id val id: String? = null,
    val name: String,
    val description: String? = null,
    val price: java.math.BigDecimal,
    val currency: String = "THB",
    @Column("stock_quantity") val stockQuantity: Int,
    @Column("category_id") val categoryId: String,
    @Column("created_at") val createdAt: java.time.LocalDateTime = java.time.LocalDateTime.now()
)

// Reactive Repository
interface ProductR2dbcRepository : ReactiveCrudRepository<ProductEntity, String> {
    
    fun findByCategoryId(categoryId: String): Flux<ProductEntity>
    
    fun findByPriceBetween(minPrice: java.math.BigDecimal, maxPrice: java.math.BigDecimal): Flux<ProductEntity>
    
    @Query("SELECT * FROM products WHERE stock_quantity > 0 ORDER BY created_at DESC LIMIT :limit OFFSET :offset")
    fun findAvailable(limit: Int, offset: Int): Flux<ProductEntity>
    
    @Query("SELECT COUNT(*) FROM products WHERE category_id = :categoryId")
    fun countByCategoryId(categoryId: String): Mono<Long>
    
    @Query("""
        SELECT * FROM products 
        WHERE name ILIKE '%' || :query || '%' 
           OR description ILIKE '%' || :query || '%'
        ORDER BY created_at DESC
    """)
    fun search(query: String): Flux<ProductEntity>
    
    @org.springframework.data.r2dbc.repository.Modifying
    @Query("UPDATE products SET stock_quantity = stock_quantity - :quantity WHERE id = :id AND stock_quantity >= :quantity")
    fun decrementStock(id: String, quantity: Int): Mono<Int>  // returns affected rows
}

// DatabaseClient: low-level reactive queries
@org.springframework.stereotype.Repository
class ProductRepositoryImpl(
    private val r2dbc: org.springframework.r2dbc.core.DatabaseClient,
    private val repo: ProductR2dbcRepository
) {
    
    fun findWithComplexFilter(filter: ProductFilter): Flux<ProductEntity> {
        val conditions = mutableListOf<String>()
        val bindings = mutableMapOf<String, Any>()
        
        filter.categoryId?.let {
            conditions.add("category_id = :categoryId")
            bindings["categoryId"] = it
        }
        
        filter.minPrice?.let {
            conditions.add("price >= :minPrice")
            bindings["minPrice"] = it
        }
        
        filter.maxPrice?.let {
            conditions.add("price <= :maxPrice")
            bindings["maxPrice"] = it
        }
        
        if (filter.inStock == true) {
            conditions.add("stock_quantity > 0")
        }
        
        val where = if (conditions.isNotEmpty()) "WHERE ${conditions.joinToString(" AND ")}" else ""
        val sql = "SELECT * FROM products $where ORDER BY created_at DESC"
        
        var spec = r2dbc.sql(sql)
        bindings.forEach { (k, v) -> spec = spec.bind(k, v) }
        
        return spec.map { row, _ ->
            ProductEntity(
                id = row["id", String::class.java],
                name = row["name", String::class.java]!!,
                price = row["price", java.math.BigDecimal::class.java]!!,
                currency = row["currency", String::class.java]!!,
                stockQuantity = row["stock_quantity", Int::class.java]!!,
                categoryId = row["category_id", String::class.java]!!
            )
        }.all()
    }
    
    fun findPageable(page: Int, size: Int): Flux<ProductEntity> {
        return repo.findAll()
            .skip((page * size).toLong())
            .take(size.toLong())
    }
    
    // Reactive transaction
    fun transferStock(fromId: String, toId: String, quantity: Int): Mono<Void> {
        return org.springframework.transaction.reactive.TransactionalOperator
            .create(
                org.springframework.r2dbc.connection.R2dbcTransactionManager(
                    io.r2dbc.spi.ConnectionFactories.get("")
                )
            )
            .transactional(
                repo.decrementStock(fromId, quantity)
                    .flatMap { affected ->
                        if (affected == 0) {
                            Mono.error(InsufficientStockException("Not enough stock in $fromId"))
                        } else {
                            r2dbc.sql("UPDATE products SET stock_quantity = stock_quantity + :qty WHERE id = :id")
                                .bind("qty", quantity)
                                .bind("id", toId)
                                .fetch().rowsUpdated()
                        }
                    }
                    .then()
            )
    }
}

data class ProductFilter(val categoryId: String? = null, val minPrice: java.math.BigDecimal? = null, val maxPrice: java.math.BigDecimal? = null, val inStock: Boolean? = null)
class InsufficientStockException(msg: String) : RuntimeException(msg)
```

---

## Reactive Testing

```kotlin
import io.projectreactor.test.StepVerifier
import io.projectreactor.test.publisher.TestPublisher
import org.junit.jupiter.api.Test
import reactor.test.StepVerifier
import reactor.test.publisher.TestPublisher

class ProductServiceTest {
    
    private val repository = mockk<ProductR2dbcRepository>()
    private val service = ProductServiceImpl(repository)
    
    @Test
    fun `should return product when found`() {
        val product = ProductEntity(id = "1", name = "MacBook", price = 50000.toBigDecimal(), 
            currency = "THB", stockQuantity = 10, categoryId = "electronics")
        
        every { repository.findById("1") } returns Mono.just(product)
        
        StepVerifier.create(service.findById("1"))
            .expectNextMatches { it.id == "1" && it.name == "MacBook" }
            .verifyComplete()
    }
    
    @Test
    fun `should return empty when product not found`() {
        every { repository.findById("999") } returns Mono.empty()
        
        StepVerifier.create(service.findById("999"))
            .verifyComplete()  // no items emitted
    }
    
    @Test
    fun `should emit error on repository failure`() {
        val dbError = RuntimeException("Database connection lost")
        every { repository.findById("1") } returns Mono.error(dbError)
        
        StepVerifier.create(service.findById("1"))
            .expectError(RuntimeException::class.java)
            .verify()
    }
    
    @Test
    fun `should return all products in category`() {
        val products = (1..5).map { i ->
            ProductEntity(id = "$i", name = "Product $i", price = (i * 100).toBigDecimal(),
                currency = "THB", stockQuantity = i * 5, categoryId = "electronics")
        }
        
        every { repository.findByCategoryId("electronics") } returns Flux.fromIterable(products)
        
        StepVerifier.create(service.findByCategory("electronics"))
            .expectNextCount(5)
            .verifyComplete()
    }
    
    @Test
    fun `should handle backpressure`() {
        val manyProducts = (1..100).map { i ->
            ProductEntity(id = "$i", name = "P$i", price = 100.toBigDecimal(),
                currency = "THB", stockQuantity = 10, categoryId = "c1")
        }
        
        every { repository.findAll() } returns Flux.fromIterable(manyProducts)
        
        StepVerifier.create(
            service.streamAll().limitRate(10),  // request 10 at a time
            10  // initial demand
        )
            .expectNextCount(10)
            .thenRequest(10)
            .expectNextCount(10)
            .thenCancel()
            .verify()
    }
    
    @Test
    fun `should timeout when service is slow`() {
        every { repository.findById("1") } returns Mono.never<ProductEntity>()  // never completes
        
        StepVerifier.create(
            service.findById("1").timeout(Duration.ofMillis(100))
        )
            .expectError(java.util.concurrent.TimeoutException::class.java)
            .verify()
    }
    
    @Test
    fun `should retry on transient errors`() {
        var callCount = 0
        
        every { repository.findById("1") } answers {
            callCount++
            if (callCount < 3) Mono.error(RuntimeException("Transient error"))
            else Mono.just(ProductEntity(id = "1", name = "Product", price = 100.toBigDecimal(),
                currency = "THB", stockQuantity = 5, categoryId = "c1"))
        }
        
        StepVerifier.create(
            service.findByIdWithRetry("1", maxRetries = 3)
        )
            .expectNextMatches { it.id == "1" }
            .verifyComplete()
        
        assert(callCount == 3)
    }
    
    @Test
    fun `TestPublisher for hot stream testing`() {
        val publisher = TestPublisher.create<String>()
        
        val flux = publisher.flux().map { it.uppercase() }
        
        val verifier = StepVerifier.create(flux)
            .then { publisher.next("hello") }
            .expectNext("HELLO")
            .then { publisher.next("world") }
            .expectNext("WORLD")
            .then { publisher.complete() }
            .verifyComplete()
    }
    
    @Test
    fun `WebTestClient for reactive controller`() {
        val client = org.springframework.test.web.reactive.server.WebTestClient
            .bindToRouterFunction(
                ProductRouter(ProductHandler(service)).productRoutes()
            )
            .build()
        
        every { repository.findAll() } returns Flux.just(
            ProductEntity(id = "1", name = "iPhone", price = 35000.toBigDecimal(), 
                currency = "THB", stockQuantity = 5, categoryId = "phones"),
            ProductEntity(id = "2", name = "iPad", price = 25000.toBigDecimal(), 
                currency = "THB", stockQuantity = 3, categoryId = "tablets")
        )
        
        client.get().uri("/api/v1/products")
            .accept(MediaType.APPLICATION_JSON)
            .exchange()
            .expectStatus().isOk
            .expectHeader().contentType(MediaType.APPLICATION_JSON)
            .expectBodyList(ProductDto::class.java)
            .hasSize(2)
    }
    
    @Test
    fun `should stream SSE events`() {
        val prices = Flux.interval(Duration.ofMillis(100))
            .take(5)
            .map { i -> PriceUpdateDto("product-$i", i * 100.0) }
        
        every { service.priceUpdates() } returns prices
        
        val client = org.springframework.test.web.reactive.server.WebTestClient
            .bindToRouterFunction(
                ProductRouter(ProductHandler(service)).productRoutes()
            )
            .build()
        
        client.get().uri("/api/v1/products/stream/prices")
            .accept(MediaType.TEXT_EVENT_STREAM)
            .exchange()
            .expectStatus().isOk
            .expectBodyList(PriceUpdateDto::class.java)
            .hasSize(5)
    }
}

// Mock setup
fun mockk(relaxed: Boolean = false): ProductR2dbcRepository = TODO()
fun <T> every(block: () -> T): io.mockk.MockKStubScope<T, T> = TODO()
fun <T> io.mockk.MockKStubScope<T, T>.returns(value: T): Unit = TODO()
fun <T> io.mockk.MockKStubScope<T, T>.answers(block: (io.mockk.MockKAnswerScope<T, T>) -> T): Unit = TODO()

class ProductServiceImpl(private val repository: ProductR2dbcRepository) : ReactiveProductService {
    override fun findAll(categoryId: String?, page: Int, size: Int): Mono<ProductPageDto> = TODO()
    override fun findById(id: String): Mono<ProductDto> = TODO()
    override fun create(dto: CreateProductDto): Mono<ProductDto> = TODO()
    override fun update(id: String, dto: UpdateProductDto): Mono<ProductDto> = TODO()
    override fun delete(id: String): Mono<Void> = TODO()
    override fun search(query: String): Flux<ProductDto> = TODO()
    override fun streamAll(): Flux<ProductDto> = TODO()
    override fun priceUpdates(): Flux<PriceUpdateDto> = TODO()
    override fun inventoryUpdates(): Flux<InventoryUpdateDto> = TODO()
    fun findByCategory(categoryId: String): Flux<ProductDto> = TODO()
    fun findByIdWithRetry(id: String, maxRetries: Int): Mono<ProductDto> = TODO()
}
```

---

## Backpressure & Error Handling

```kotlin
// Backpressure strategies
fun backpressureExamples() {
    val fastProducer = Flux.interval(Duration.ofMillis(1))
        .map { "item-$it" }
    
    // BUFFER: เก็บไว้ใน queue (อาจ OOM ถ้า consumer ช้ามาก)
    fastProducer
        .onBackpressureBuffer(1000)  // max 1000 in buffer
        .subscribe { slowConsume(it) }
    
    // DROP: ทิ้ง item ที่มาเกิน capacity
    fastProducer
        .onBackpressureBuffer(100, 
            { dropped -> println("Dropped: $dropped") },
            reactor.core.publisher.BufferOverflowStrategy.DROP_LATEST
        )
        .subscribe { slowConsume(it) }
    
    // LATEST: keep only most recent, drop others
    fastProducer
        .onBackpressureLatest()
        .subscribe { slowConsume(it) }
    
    // ERROR: throw error when buffer full
    fastProducer
        .onBackpressureError()
        .subscribe(
            { slowConsume(it) },
            { e -> println("Overflow error: $e") }
        )
    
    // limitRate: control prefetch
    fastProducer
        .limitRate(100)  // request 100 from upstream at a time
        .subscribe { slowConsume(it) }
}

fun slowConsume(item: String) {
    Thread.sleep(10)  // simulate slow processing
    println("Consumed: $item")
}

// Advanced error handling patterns
fun errorHandlingPatterns() {
    val flux = Flux.just(1, 2, 3, 0, 5)
        .map { 10 / it }  // will throw on 0
    
    // onErrorReturn: replace error with value
    flux.onErrorReturn(-1)
        .subscribe { println(it) }  // 10, 5, 3, -1
    
    // onErrorResume: fallback to another publisher
    flux.onErrorResume { e ->
        println("Error: ${e.message}, switching to fallback")
        Flux.just(99, 100)
    }
    
    // onErrorContinue: skip item that caused error
    flux.onErrorContinue { e, item ->
        println("Skipping $item due to ${e.message}")
    }
    
    // retry with exponential backoff
    val unreliableService = Flux.defer {
        if (Math.random() < 0.7) Flux.error(RuntimeException("Service unavailable"))
        else Flux.just("success")
    }
    
    unreliableService
        .retryWhen(
            reactor.util.retry.Retry.backoff(5, Duration.ofMillis(100))
                .maxBackoff(Duration.ofSeconds(30))
                .jitter(0.5)
                .filter { it is RuntimeException }
                .doBeforeRetry { signal ->
                    println("Retrying attempt ${signal.totalRetries()}")
                }
        )
        .subscribe(
            { println("Got: $it") },
            { println("Failed after all retries: $it") }
        )
    
    // timeout per item
    Flux.interval(Duration.ofMillis(500))
        .flatMap { i ->
            Mono.fromCallable { fetchData(i) }
                .timeout(Duration.ofMillis(200))
                .onErrorReturn("timeout-fallback")
        }
        .take(5)
        .subscribe { println(it) }
}

fun fetchData(i: Long): String {
    Thread.sleep(if (i % 3 == 0L) 300 else 100)  // slow every 3rd
    return "data-$i"
}

// Reactive security context
fun reactiveSecurityContext(): Mono<String> {
    return reactor.core.publisher.Mono.deferContextual { context ->
        val userId = context.getOrDefault("userId", "anonymous")
        Mono.just("Current user: $userId")
    }
}

// Scheduler: control which thread runs the work
fun schedulerExamples() {
    val flux = Flux.range(1, 10)
    
    // subscribeOn: upstream runs on this scheduler
    flux.subscribeOn(reactor.core.scheduler.Schedulers.boundedElastic())
        .subscribe { println("[${Thread.currentThread().name}] $it") }
    
    // publishOn: downstream runs on this scheduler
    flux.publishOn(reactor.core.scheduler.Schedulers.parallel())
        .map { it * 2 }
        .subscribe { println("[${Thread.currentThread().name}] $it") }
    
    // For I/O work: boundedElastic (thread pool, auto-sized)
    // For CPU work: parallel (fixed = core count)
    // For quick tasks: immediate (current thread)
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Reactive Order Processing Pipeline

// สร้าง reactive pipeline ที่:
// 1. รับ stream ของ order requests เข้ามา
// 2. Validate แต่ละ order
// 3. ตรวจสอบ stock (parallel สำหรับหลาย items)
// 4. ถ้า stock พอ → charge payment
// 5. ถ้าชำระเงินสำเร็จ → create order + deduct stock (transaction)
// 6. ถ้าขั้นตอนใดล้มเหลว → return error result (ไม่ terminate stream)

data class OrderRequest(val customerId: String, val items: List<OrderItemRequest>)
data class OrderItemRequest(val productId: String, val quantity: Int)
data class OrderResult(val success: Boolean, val orderId: String?, val error: String?)

@org.springframework.stereotype.Service
class ReactiveOrderPipeline(
    private val productRepository: ProductR2dbcRepository,
    private val paymentService: ReactivePaymentService,
    private val orderRepository: OrderR2dbcRepository
) {
    
    fun processOrders(requests: Flux<OrderRequest>): Flux<OrderResult> {
        return requests
            .flatMap { request ->
                processOrder(request)
                    .onErrorResume { e ->
                        Mono.just(OrderResult(false, null, e.message))
                    }
            }
    }
    
    private fun processOrder(request: OrderRequest): Mono<OrderResult> {
        return validateOrder(request)
            .flatMap { checkAllStocks(it) }
            .flatMap { chargePayment(it) }
            .flatMap { createOrderWithDeduction(it) }
            .map { OrderResult(true, it, null) }
    }
    
    private fun validateOrder(request: OrderRequest): Mono<OrderRequest> {
        return when {
            request.items.isEmpty() -> Mono.error(IllegalArgumentException("No items"))
            request.items.any { it.quantity <= 0 } -> Mono.error(IllegalArgumentException("Invalid quantity"))
            else -> Mono.just(request)
        }
    }
    
    private fun checkAllStocks(request: OrderRequest): Mono<OrderRequest> {
        // Check all items in parallel
        val checks = request.items.map { item ->
            productRepository.findById(item.productId)
                .switchIfEmpty(Mono.error(NotFoundException("Product ${item.productId} not found")))
                .flatMap { product ->
                    if (product.stockQuantity < item.quantity) {
                        Mono.error(InsufficientStockException("Not enough stock for ${item.productId}"))
                    } else {
                        Mono.just(true)
                    }
                }
        }
        
        return Flux.merge(checks).then(Mono.just(request))
    }
    
    private fun chargePayment(request: OrderRequest): Mono<OrderRequest> {
        val total = request.items.sumOf { it.quantity * 100.0 }  // simplified pricing
        
        return paymentService.charge(request.customerId, total)
            .flatMap { success ->
                if (success) Mono.just(request)
                else Mono.error(RuntimeException("Payment failed"))
            }
    }
    
    private fun createOrderWithDeduction(request: OrderRequest): Mono<String> {
        val orderId = java.util.UUID.randomUUID().toString()
        
        val deductions = request.items.map { item ->
            productRepository.decrementStock(item.productId, item.quantity)
                .flatMap { affected ->
                    if (affected == 0) Mono.error(InsufficientStockException("Race condition on ${item.productId}"))
                    else Mono.just(affected)
                }
        }
        
        return Flux.merge(deductions)
            .then(orderRepository.save(orderId, request.customerId, request.items))
            .thenReturn(orderId)
    }
}

interface ReactivePaymentService {
    fun charge(customerId: String, amount: Double): Mono<Boolean>
}

interface OrderR2dbcRepository {
    fun save(orderId: String, customerId: String, items: List<OrderItemRequest>): Mono<Void>
}

// Test the pipeline
class ReactiveOrderPipelineTest {
    
    @Test
    fun `should process orders and skip failures`() {
        val productRepo = mockk<ProductR2dbcRepository>()
        val paymentService = mockk<ReactivePaymentService>()
        val orderRepo = mockk<OrderR2dbcRepository>()
        
        val pipeline = ReactiveOrderPipeline(productRepo, paymentService, orderRepo)
        
        val requests = Flux.just(
            OrderRequest("user1", listOf(OrderItemRequest("p1", 2))),
            OrderRequest("user2", listOf()),  // invalid → should skip
            OrderRequest("user3", listOf(OrderItemRequest("p2", 1)))
        )
        
        StepVerifier.create(pipeline.processOrders(requests))
            .expectNextMatches { it.success }
            .expectNextMatches { !it.success && it.error == "No items" }
            .expectNextMatches { it.success }
            .verifyComplete()
    }
}
```

---

## สรุป Part 90

```
✅ Reactive Programming: non-blocking, event-driven
✅ Mono<T>: async 0 or 1 item
✅ Flux<T>: async 0 to N items
✅ map/flatMap: transform synchronous/asynchronous
✅ filter/take/skip: control elements
✅ merge/concat/zip: combine streams
✅ buffer/window/groupBy: batch/window/group
✅ Cold vs Hot: cold re-runs per subscriber, hot is shared
✅ Sinks.Many: create hot Flux programmatically
✅ publish().connect(): cold → hot with explicit start
✅ RouterFunction: functional routing DSL
✅ ServerRequest/ServerResponse: functional handlers
✅ @RestController: annotation style, same as MVC but reactive
✅ SSE: produces = TEXT_EVENT_STREAM_VALUE
✅ WebFilter: cross-cutting (correlationId, auth)
✅ R2DBC: reactive SQL (no blocking JDBC)
✅ ReactiveCrudRepository: reactive Spring Data
✅ DatabaseClient: low-level reactive SQL
✅ @Modifying: write operations return Mono<Int>
✅ TransactionalOperator: reactive transactions
✅ StepVerifier: test reactive publishers
✅ expectNextMatches/expectNextCount: assert items
✅ then/thenRequest: manual demand control
✅ TestPublisher: hot publisher for testing
✅ WebTestClient: reactive HTTP testing
✅ onBackpressureBuffer/Drop/Latest: backpressure strategies
✅ limitRate: control upstream request rate
✅ onErrorReturn/Resume/Continue: error recovery
✅ retryWhen + Retry.backoff: exponential backoff retry
✅ subscribeOn/publishOn: choose scheduler (thread)
✅ Schedulers.boundedElastic: for I/O work
✅ Reactive pipeline: validate → stock check → charge → create
✅ Flux.merge(checks).then(): parallel checks
```

---

*Part 90/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
