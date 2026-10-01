# Part 62: Spring WebFlux ขั้นสูง

## สารบัญ
1. [Reactive Streams และ Backpressure](#reactive-streams-และ-backpressure)
2. [Schedulers และ Threading](#schedulers-และ-threading)
3. [R2DBC Reactive Database](#r2dbc-reactive-database)
4. [Functional Endpoints](#functional-endpoints)
5. [Server-Sent Events](#server-sent-events)
6. [WebClient ขั้นสูง](#webclient-ขั้นสูง)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Reactive Streams และ Backpressure

WebFlux ใช้ Project Reactor — `Mono<T>` (0-1 item) และ `Flux<T>` (0-N items)

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    implementation("org.springframework.boot:spring-boot-starter-data-r2dbc")
    implementation("org.postgresql:r2dbc-postgresql:1.0.5.RELEASE")
    implementation("io.r2dbc:r2dbc-pool:1.0.1.RELEASE")
    implementation("org.springframework.boot:spring-boot-starter-data-redis-reactive")
    implementation("io.projectreactor.kotlin:reactor-kotlin-extensions:1.2.3")
    testImplementation("io.projectreactor:reactor-test:3.6.10")
}

// Backpressure strategies
import reactor.core.publisher.*

fun backpressureDemo() {
    
    // BUFFER: buffer overflow items (default)
    Flux.range(1, 100)
        .onBackpressureBuffer(50)  // buffer up to 50 items
        .subscribe { println(it) }
    
    // DROP: drop items when downstream can't keep up
    Flux.range(1, 100)
        .onBackpressureDrop { dropped -> println("Dropped: $dropped") }
        .subscribe { Thread.sleep(10) }
    
    // LATEST: keep only the latest item
    Flux.range(1, 100)
        .onBackpressureLatest()
        .subscribe { println("Latest: $it") }
    
    // ERROR: fail when overflow
    Flux.range(1, 100)
        .onBackpressureError()
        .subscribe(
            { println(it) },
            { error -> println("Error: $error") }
        )
}

// Reactive operators
fun reactiveOperators() {
    // filter, map, flatMap
    Flux.range(1, 10)
        .filter { it % 2 == 0 }
        .map { it * 2 }
        .flatMap { n ->
            Mono.just("item-$n").delayElement(java.time.Duration.ofMillis(100))
        }
        .subscribe { println(it) }
    
    // concatMap: preserves order (unlike flatMap)
    Flux.range(1, 5)
        .concatMap { n ->
            Mono.just("item-$n").delayElement(java.time.Duration.ofMillis(n.toLong() * 100))
        }
        .subscribe { println(it) }
    
    // zipWith: combine two Flux
    val names = Flux.just("Alice", "Bob", "Charlie")
    val ages = Flux.just(30, 25, 35)
    names.zipWith(ages) { name, age -> "$name is $age years old" }
        .subscribe { println(it) }
    
    // merge vs concat
    val slow = Flux.interval(java.time.Duration.ofSeconds(1)).take(3).map { "slow-$it" }
    val fast = Flux.interval(java.time.Duration.ofMillis(200)).take(3).map { "fast-$it" }
    
    Flux.merge(slow, fast)  // interleaved by arrival time
        .subscribe { println(it) }
    
    Flux.concat(slow, fast)  // slow then fast, in order
        .subscribe { println(it) }
    
    // switchMap: cancel previous on new item (search-as-you-type)
    Flux.just("k", "ko", "kot", "kotl", "kotli", "kotlin")
        .delayElements(java.time.Duration.ofMillis(100))
        .switchMap { query ->
            searchProducts(query)  // cancels previous search
        }
        .subscribe { println(it) }
    
    // timeout
    Flux.range(1, 10)
        .delayElements(java.time.Duration.ofMillis(200))
        .timeout(java.time.Duration.ofSeconds(1))
        .onErrorReturn(-1)
        .subscribe { println(it) }
    
    // retry
    Flux.range(1, 3)
        .flatMap { n ->
            if (n == 2) Mono.error(RuntimeException("Transient error"))
            else Mono.just(n)
        }
        .retry(3)
        .retryWhen(Retry.backoff(3, java.time.Duration.ofSeconds(1))
            .filter { it is RuntimeException })
        .subscribe({ println(it) }, { println("Final error: $it") })
    
    // window: group into windows
    Flux.range(1, 20)
        .window(5)  // emit Flux of 5 items at a time
        .flatMap { window -> window.collectList() }
        .subscribe { println("Window: $it") }
    
    // buffer: collect N items
    Flux.range(1, 20)
        .buffer(5)
        .subscribe { batch -> println("Batch: $batch") }
    
    // groupBy
    Flux.range(1, 10)
        .groupBy { if (it % 2 == 0) "even" else "odd" }
        .flatMap { group ->
            group.collectList().map { "${group.key()}: $it" }
        }
        .subscribe { println(it) }
}

fun searchProducts(query: String): Flux<String> = Flux.just("Product matching $query")
```

---

## Schedulers และ Threading

```kotlin
import reactor.core.scheduler.Schedulers

fun schedulerExamples() {
    
    // publishOn: change thread for downstream operations
    Flux.range(1, 5)
        .map { "upstream on ${Thread.currentThread().name}: $it" }  // runs on caller thread
        .publishOn(Schedulers.boundedElastic())  // switch to elastic thread pool
        .map { "downstream on ${Thread.currentThread().name}: $it" }
        .subscribe { println(it) }
    
    // subscribeOn: change thread for entire subscription (including upstream)
    Flux.range(1, 5)
        .subscribeOn(Schedulers.parallel())  // entire chain on parallel thread
        .map { it * 2 }
        .subscribe { println("${Thread.currentThread().name}: $it") }
    
    // parallel: distribute work across CPUs
    Flux.range(1, 16)
        .parallel(4)                              // 4 rails
        .runOn(Schedulers.parallel())             // each rail on separate thread
        .map { processItem(it) }                  // parallel processing
        .sequential()                             // merge back to single Flux
        .subscribe { println(it) }
    
    // Blocking code in reactive (use boundedElastic!)
    fun loadFromDatabase(id: String): String = "data-$id"  // blocking
    
    Flux.just("id-1", "id-2", "id-3")
        .flatMap { id ->
            // Wrap blocking call in Mono.fromCallable on boundedElastic
            Mono.fromCallable { loadFromDatabase(id) }
                .subscribeOn(Schedulers.boundedElastic())
        }
        .subscribe { println("Loaded: $it") }
    
    // Schedulers types:
    // immediate() - caller thread, no scheduling
    // single()    - single reusable thread
    // parallel()  - CPU-bound, N threads = N CPU cores
    // boundedElastic() - I/O bound, elastic with max cap (default 10 * CPU)
    // fromExecutor() - custom executor
}

fun processItem(item: Int): String = "processed-$item"

// Context propagation
fun contextPropagation() {
    
    // Reactor Context: immutable key-value store passed downstream
    Mono.just("request")
        .flatMap { req ->
            Mono.deferContextual { context ->
                val userId = context.getOrDefault("userId", "anonymous")
                Mono.just("Hello $userId, processing $req")
            }
        }
        .contextWrite { ctx -> ctx.put("userId", "user-123") }  // write context upstream
        .subscribe { println(it) }
    
    // Read context in service
    @Service
    class AuditService {
        fun auditOperation(operation: String): Mono<Void> {
            return Mono.deferContextual { context ->
                val userId = context.getOrDefault("userId", "unknown")
                val requestId = context.getOrDefault("requestId", "")
                println("Audit: $userId performed $operation (requestId=$requestId)")
                Mono.empty()
            }
        }
    }
}
```

---

## R2DBC Reactive Database

```kotlin
// R2DBC configuration
@Configuration
class R2dbcConfig {
    
    @Bean
    fun connectionFactory(): ConnectionFactory {
        return ConnectionFactoryOptions.builder()
            .option(ConnectionFactoryOptions.DRIVER, "postgresql")
            .option(ConnectionFactoryOptions.HOST, "localhost")
            .option(ConnectionFactoryOptions.PORT, 5432)
            .option(ConnectionFactoryOptions.DATABASE, "myapp")
            .option(ConnectionFactoryOptions.USER, "postgres")
            .option(ConnectionFactoryOptions.PASSWORD, "secret")
            .option(ConnectionFactoryOptions.SSL, false)
            .build()
            .let { ConnectionFactories.get(it) }
    }
    
    @Bean
    fun r2dbcTransactionManager(connectionFactory: ConnectionFactory): ReactiveTransactionManager {
        return R2dbcTransactionManager(connectionFactory)
    }
}

// Entity สำหรับ R2DBC
@Table("products")
data class ProductEntity(
    @Id val id: String? = null,
    val name: String,
    val description: String,
    val price: java.math.BigDecimal,
    val stockQuantity: Int,
    val categoryId: String,
    @Column("created_at") val createdAt: java.time.Instant = java.time.Instant.now()
)

// R2DBC Repository
interface ProductR2dbcRepository : ReactiveCrudRepository<ProductEntity, String> {
    
    fun findByCategoryId(categoryId: String): Flux<ProductEntity>
    
    @Query("SELECT * FROM products WHERE price BETWEEN :minPrice AND :maxPrice ORDER BY price")
    fun findByPriceRange(
        @Param("minPrice") minPrice: java.math.BigDecimal,
        @Param("maxPrice") maxPrice: java.math.BigDecimal
    ): Flux<ProductEntity>
    
    @Query("""
        SELECT * FROM products 
        WHERE to_tsvector('english', name || ' ' || description) @@ plainto_tsquery('english', :query)
        LIMIT :limit
    """)
    fun fullTextSearch(
        @Param("query") query: String,
        @Param("limit") limit: Int
    ): Flux<ProductEntity>
    
    @Query("SELECT COUNT(*) FROM products WHERE category_id = :categoryId")
    fun countByCategoryId(@Param("categoryId") categoryId: String): Mono<Long>
}

// Custom reactive repository (for complex queries)
@Repository
class ProductReactiveRepository(private val template: R2dbcEntityTemplate) {
    
    fun findWithPagination(page: Int, size: Int, sortBy: String): Flux<ProductEntity> {
        val query = Query.query(Criteria.empty())
            .sort(Sort.by(Sort.Direction.ASC, sortBy))
            .offset(page.toLong() * size)
            .limit(size)
        
        return template.select(ProductEntity::class.java)
            .matching(query)
            .all()
    }
    
    fun findByCriteria(
        categoryId: String?,
        minPrice: java.math.BigDecimal?,
        maxPrice: java.math.BigDecimal?
    ): Flux<ProductEntity> {
        var criteria = Criteria.empty()
        
        categoryId?.let { criteria = criteria.and("category_id").`is`(it) }
        minPrice?.let { criteria = criteria.and("price").greaterThanOrEquals(it) }
        maxPrice?.let { criteria = criteria.and("price").lessThanOrEquals(it) }
        
        return template.select(ProductEntity::class.java)
            .matching(Query.query(criteria))
            .all()
    }
    
    @Transactional
    fun transferStock(fromProductId: String, toProductId: String, quantity: Int): Mono<Void> {
        return template.update(ProductEntity::class.java)
            .matching(Query.query(Criteria.where("id").`is`(fromProductId)))
            .apply(Update.update("stock_quantity", 
                template.select(ProductEntity::class.java)
                    .matching(Query.query(Criteria.where("id").`is`(fromProductId)))
                    .first()
                    .map { it.stockQuantity - quantity }
            ))
            .thenReturn(Unit)
            .then()
    }
}

// placeholder imports
typealias ReactiveCrudRepository<T, ID> = org.springframework.data.repository.reactive.ReactiveCrudRepository<T, ID>
typealias R2dbcEntityTemplate = org.springframework.data.r2dbc.core.R2dbcEntityTemplate
typealias ReactiveTransactionManager = org.springframework.transaction.ReactiveTransactionManager
typealias R2dbcTransactionManager = org.springframework.r2dbc.connection.R2dbcTransactionManager
typealias ConnectionFactory = io.r2dbc.spi.ConnectionFactory
typealias ConnectionFactories = io.r2dbc.spi.ConnectionFactories
typealias ConnectionFactoryOptions = io.r2dbc.spi.ConnectionFactoryOptions
typealias Query = org.springframework.data.r2dbc.query.Query
typealias Criteria = org.springframework.data.r2dbc.query.Criteria
typealias Sort = org.springframework.data.domain.Sort
typealias Update = org.springframework.data.relational.core.query.Update
typealias Param = org.springframework.data.repository.query.Param
```

---

## Functional Endpoints

```kotlin
// Functional style แทน @RestController (alternative approach)
@Component
class ProductHandler(private val productService: ReactiveProductService) {
    
    suspend fun getProduct(request: ServerRequest): ServerResponse {
        val id = request.pathVariable("id")
        
        return productService.findById(id).fold(
            ifEmpty = { ServerResponse.notFound().buildAndAwait() },
            ifSome = { product ->
                ServerResponse.ok()
                    .contentType(MediaType.APPLICATION_JSON)
                    .bodyValueAndAwait(product)
            }
        )
    }
    
    suspend fun createProduct(request: ServerRequest): ServerResponse {
        val body = request.awaitBody<CreateProductDto>()
        
        // Validate
        val errors = validateProduct(body)
        if (errors.isNotEmpty()) {
            return ServerResponse.badRequest().bodyValueAndAwait(errors)
        }
        
        return productService.create(body).fold(
            ifLeft = { error ->
                ServerResponse.status(HttpStatus.UNPROCESSABLE_ENTITY)
                    .bodyValueAndAwait(error)
            },
            ifRight = { product ->
                ServerResponse.created(
                    java.net.URI.create("/api/products/${product.id}")
                ).bodyValueAndAwait(product)
            }
        )
    }
    
    suspend fun listProducts(request: ServerRequest): ServerResponse {
        val page = request.queryParamOrNull("page")?.toInt() ?: 0
        val size = request.queryParamOrNull("size")?.toInt() ?: 20
        val category = request.queryParamOrNull("category")
        
        val products = productService.findAll(page, size, category)
        
        return ServerResponse.ok()
            .contentType(MediaType.APPLICATION_JSON)
            .bodyAndAwait(products)
    }
    
    // Streaming response
    suspend fun streamProducts(request: ServerRequest): ServerResponse {
        return ServerResponse.ok()
            .contentType(MediaType.TEXT_EVENT_STREAM)
            .bodyAndAwait(productService.streamAll())
    }
    
    private fun validateProduct(dto: CreateProductDto): List<String> {
        val errors = mutableListOf<String>()
        if (dto.name.isBlank()) errors.add("name is required")
        if (dto.price <= 0) errors.add("price must be positive")
        return errors
    }
}

// Router function
@Configuration
class RouterConfig(private val handler: ProductHandler) {
    
    @Bean
    fun productRouter(): RouterFunction<ServerResponse> = coRouter {
        "/api/products".nest {
            GET("", handler::listProducts)
            GET("/stream", handler::streamProducts)
            GET("/{id}", handler::getProduct)
            POST("", handler::createProduct)
        }
    }
}

// Service interface
interface ReactiveProductService {
    fun findById(id: String): reactor.core.publisher.Mono<out Any?>
    fun create(dto: CreateProductDto): reactor.core.publisher.Mono<out Any>
    fun findAll(page: Int, size: Int, category: String?): reactor.core.publisher.Flux<out Any>
    fun streamAll(): reactor.core.publisher.Flux<out Any>
}

data class CreateProductDto(val name: String, val price: Double)

// needed for functional approach
fun <T> reactor.core.publisher.Mono<T>.fold(ifEmpty: () -> Any, ifSome: (T) -> Any): Any = TODO()
fun <L, R> reactor.core.publisher.Mono<*>.fold(ifLeft: (L) -> Any, ifRight: (R) -> Any): Any = TODO()

typealias ServerRequest = org.springframework.web.reactive.function.server.ServerRequest
typealias ServerResponse = org.springframework.web.reactive.function.server.ServerResponse
typealias RouterFunction<T> = org.springframework.web.reactive.function.server.RouterFunction<T>
typealias MediaType = org.springframework.http.MediaType
typealias HttpStatus = org.springframework.http.HttpStatus
```

---

## Server-Sent Events

```kotlin
@RestController
@RequestMapping("/api/sse")
class SseController(
    private val orderService: ReactiveOrderService,
    private val stockService: ReactiveStockService
) {
    
    // Basic SSE endpoint
    @GetMapping("/orders/{userId}", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
    fun streamOrderUpdates(@PathVariable userId: String): Flux<ServerSentEvent<OrderUpdate>> {
        return orderService.streamOrderUpdates(userId)
            .map { update ->
                ServerSentEvent.builder(update)
                    .id(update.orderId)
                    .event("order-update")
                    .comment("Order status changed")
                    .retry(java.time.Duration.ofSeconds(3))
                    .build()
            }
            .doOnSubscribe { println("SSE: user $userId subscribed") }
            .doOnCancel { println("SSE: user $userId disconnected") }
    }
    
    // Multiple event types
    @GetMapping("/dashboard", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
    fun streamDashboard(): Flux<ServerSentEvent<Any>> {
        val orderStream = orderService.streamAllOrders()
            .map { order ->
                ServerSentEvent.builder<Any>(order)
                    .event("order")
                    .build()
            }
        
        val stockStream = stockService.streamLowStockAlerts()
            .map { alert ->
                ServerSentEvent.builder<Any>(alert)
                    .event("stock-alert")
                    .build()
            }
        
        // Merge multiple streams
        return Flux.merge(orderStream, stockStream)
    }
    
    // Heartbeat to keep connection alive
    @GetMapping("/live-prices", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
    fun streamLivePrices(): Flux<ServerSentEvent<Any>> {
        val priceStream = stockService.streamPriceUpdates()
            .map { ServerSentEvent.builder<Any>(it).event("price").build() }
        
        val heartbeat = Flux.interval(java.time.Duration.ofSeconds(30))
            .map { ServerSentEvent.builder<Any>("heartbeat").event("ping").build() }
        
        return Flux.merge(priceStream, heartbeat)
            .takeUntilOther(
                // Stop after 1 hour
                Mono.delay(java.time.Duration.ofHours(1))
            )
    }
}

data class OrderUpdate(val orderId: String, val status: String)
data class StockAlert(val productId: String, val currentStock: Int)

interface ReactiveOrderService {
    fun streamOrderUpdates(userId: String): Flux<OrderUpdate>
    fun streamAllOrders(): Flux<OrderUpdate>
}

interface ReactiveStockService {
    fun streamLowStockAlerts(): Flux<StockAlert>
    fun streamPriceUpdates(): Flux<Any>
}

typealias ServerSentEvent<T> = org.springframework.http.codec.ServerSentEvent<T>
```

---

## WebClient ขั้นสูง

```kotlin
@Configuration
class WebClientConfig {
    
    @Bean
    fun productApiWebClient(): WebClient {
        val httpClient = HttpClient.create()
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
            .responseTimeout(java.time.Duration.ofSeconds(10))
            .doOnConnected { conn ->
                conn.addHandlerLast(ReadTimeoutHandler(10))
                conn.addHandlerLast(WriteTimeoutHandler(10))
            }
        
        return WebClient.builder()
            .baseUrl("https://api.products.example.com")
            .clientConnector(ReactorClientHttpConnector(httpClient))
            .defaultHeader("Accept", "application/json")
            .defaultHeader("X-API-Version", "2024-01")
            .filter(loggingFilter())
            .filter(retryFilter())
            .filter(authFilter())
            .codecs { configurer ->
                configurer.defaultCodecs().maxInMemorySize(2 * 1024 * 1024)  // 2MB
            }
            .build()
    }
    
    // Logging filter
    private fun loggingFilter(): ExchangeFilterFunction {
        return ExchangeFilterFunction { request, next ->
            println("Request: ${request.method()} ${request.url()}")
            next.exchange(request).doOnNext { response ->
                println("Response: ${response.statusCode()}")
            }
        }
    }
    
    // Retry filter
    private fun retryFilter(): ExchangeFilterFunction {
        return ExchangeFilterFunction { request, next ->
            next.exchange(request).retryWhen(
                Retry.backoff(3, java.time.Duration.ofMillis(500))
                    .filter { error ->
                        error is WebClientResponseException &&
                        error.statusCode.is5xxServerError
                    }
            )
        }
    }
    
    // Auth filter
    private fun authFilter(): ExchangeFilterFunction {
        return ExchangeFilterFunction.ofRequestProcessor { request ->
            Mono.just(
                ClientRequest.from(request)
                    .header("Authorization", "Bearer ${getToken()}")
                    .build()
            )
        }
    }
    
    private fun getToken(): String = "api-token"  // in production: load from config/vault
}

// Service using WebClient
@Service
class ExternalProductService(
    @Qualifier("productApiWebClient") private val webClient: WebClient
) {
    
    fun getProduct(id: String): Mono<ExternalProduct> {
        return webClient.get()
            .uri("/products/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError) { response ->
                response.bodyToMono(ErrorResponse::class.java)
                    .flatMap { error ->
                        Mono.error(ExternalServiceException(error.message))
                    }
            }
            .onStatus(HttpStatusCode::is5xxServerError) { _ ->
                Mono.error(ExternalServiceException("External service unavailable"))
            }
            .bodyToMono(ExternalProduct::class.java)
            .timeout(java.time.Duration.ofSeconds(5))
            .onErrorReturn(ExternalProduct("", "Unknown", 0.0))
    }
    
    fun searchProducts(query: String): Flux<ExternalProduct> {
        return webClient.get()
            .uri { builder ->
                builder.path("/products/search")
                    .queryParam("q", query)
                    .queryParam("limit", 20)
                    .build()
            }
            .retrieve()
            .bodyToFlux(ExternalProduct::class.java)
    }
    
    // Multipart file upload
    fun uploadProductImage(productId: String, image: ByteArray): Mono<String> {
        val formData = MultipartBodyBuilder().apply {
            part("productId", productId)
            part("image", image).filename("image.jpg").contentType(MediaType.IMAGE_JPEG)
        }.build()
        
        return webClient.post()
            .uri("/products/$productId/images")
            .contentType(MediaType.MULTIPART_FORM_DATA)
            .bodyValue(formData)
            .retrieve()
            .bodyToMono(String::class.java)
    }
    
    // Parallel requests
    fun getProductWithReviews(productId: String): Mono<ProductWithReviews> {
        return Mono.zip(
            getProduct(productId),
            getReviews(productId)
        ) { product, reviews ->
            ProductWithReviews(product, reviews)
        }
    }
    
    private fun getReviews(productId: String): Mono<List<Review>> {
        return webClient.get()
            .uri("/products/$productId/reviews")
            .retrieve()
            .bodyToFlux(Review::class.java)
            .collectList()
    }
}

data class ExternalProduct(val id: String, val name: String, val price: Double)
data class Review(val id: String, val rating: Int, val comment: String)
data class ProductWithReviews(val product: ExternalProduct, val reviews: List<Review>)
data class ErrorResponse(val message: String)
class ExternalServiceException(message: String) : RuntimeException(message)

typealias WebClient = org.springframework.web.reactive.function.client.WebClient
typealias WebClientResponseException = org.springframework.web.reactive.function.client.WebClientResponseException
typealias ExchangeFilterFunction = org.springframework.web.reactive.function.client.ExchangeFilterFunction
typealias ClientRequest = org.springframework.web.reactive.function.client.ClientRequest
typealias MultipartBodyBuilder = org.springframework.http.client.MultipartBodyBuilder
typealias HttpStatusCode = org.springframework.http.HttpStatusCode
typealias HttpClient = reactor.netty.http.client.HttpClient
typealias ReactorClientHttpConnector = org.springframework.http.client.reactive.ReactorClientHttpConnector
typealias ChannelOption = io.netty.channel.ChannelOption
typealias ReadTimeoutHandler = io.netty.handler.timeout.ReadTimeoutHandler
typealias WriteTimeoutHandler = io.netty.handler.timeout.WriteTimeoutHandler
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Reactive order processing pipeline

@Service
class ReactiveOrderProcessingService(
    private val orderRepository: ReactiveOrderRepository,
    private val inventoryService: ReactiveInventoryService,
    private val paymentService: ReactivePaymentService,
    private val notificationService: ReactiveNotificationService
) {
    
    // Process an order reactively with proper error handling
    fun processOrder(request: OrderRequest): Mono<ProcessedOrder> {
        return Mono.just(request)
            .flatMap { req ->
                // Step 1: Validate and create order
                orderRepository.create(req.toOrder())
            }
            .flatMap { order ->
                // Step 2: Reserve inventory (fail fast)
                inventoryService.reserve(order.items)
                    .map { order }
                    .onErrorResume { error ->
                        // Compensate: cancel order on inventory failure
                        orderRepository.cancel(order.id)
                            .then(Mono.error(OrderProcessingException("Insufficient inventory: ${error.message}")))
                    }
            }
            .flatMap { order ->
                // Step 3: Process payment
                paymentService.charge(order.id, order.totalAmount)
                    .map { payment -> order to payment }
                    .onErrorResume { error ->
                        // Compensate: release inventory + cancel order
                        inventoryService.release(order.items)
                            .then(orderRepository.cancel(order.id))
                            .then(Mono.error(OrderProcessingException("Payment failed: ${error.message}")))
                    }
            }
            .flatMap { (order, payment) ->
                // Step 4: Confirm order
                orderRepository.confirm(order.id, payment.id)
            }
            .flatMap { confirmedOrder ->
                // Step 5: Send notification (non-blocking, don't fail order on notification error)
                notificationService.sendOrderConfirmation(confirmedOrder)
                    .onErrorResume { Mono.empty() }  // swallow notification errors
                    .thenReturn(confirmedOrder)
            }
            .map { order -> ProcessedOrder(order.id, "SUCCESS", order.totalAmount) }
    }
}

data class OrderRequest(val userId: String, val items: List<OrderItem>)
data class OrderItem(val productId: String, val quantity: Int)
data class ReactiveOrder(val id: String, val items: List<OrderItem>, val totalAmount: Double, val status: String)
data class Payment(val id: String, val amount: Double)
data class ProcessedOrder(val orderId: String, val status: String, val amount: Double)

fun OrderRequest.toOrder() = ReactiveOrder("", items, items.sumOf { it.quantity * 99.99 }, "PENDING")

interface ReactiveOrderRepository {
    fun create(order: ReactiveOrder): Mono<ReactiveOrder>
    fun cancel(orderId: String): Mono<Void>
    fun confirm(orderId: String, paymentId: String): Mono<ReactiveOrder>
}

interface ReactiveInventoryService {
    fun reserve(items: List<OrderItem>): Mono<Void>
    fun release(items: List<OrderItem>): Mono<Void>
}

interface ReactivePaymentService {
    fun charge(orderId: String, amount: Double): Mono<Payment>
}

interface ReactiveNotificationService {
    fun sendOrderConfirmation(order: ReactiveOrder): Mono<Void>
}

class OrderProcessingException(message: String) : RuntimeException(message)
```

---

## สรุป Part 62

```
✅ Project Reactor: Mono<T> (0-1 item), Flux<T> (0-N items)
✅ Backpressure: onBackpressureBuffer/Drop/Latest/Error
✅ Operators: flatMap, concatMap, switchMap, mergeWith, zip, window, buffer, groupBy
✅ retry/retryWhen: with backoff and error filter
✅ timeout: fail if no item within time limit
✅ Schedulers: immediate, single, parallel, boundedElastic
✅ publishOn: change downstream thread
✅ subscribeOn: change subscription thread
✅ Blocking code: wrap in Mono.fromCallable + boundedElastic
✅ Reactor Context: propagate request metadata reactively
✅ R2DBC: reactive database access
✅ R2dbcEntityTemplate: dynamic reactive queries
✅ @Transactional: reactive transactions with R2DBC
✅ Functional endpoints: RouterFunction, coRouter, ServerRequest/Response
✅ Server-Sent Events: Flux<ServerSentEvent<T>>
✅ WebClient: reactive HTTP client, filter chain, retry, error handling
✅ WebClient multipart: file upload reactively
✅ Parallel requests: Mono.zip for concurrent calls
```

---

*Part 62/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
