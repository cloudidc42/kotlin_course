# Part 43: Reactive Programming ด้วย Kotlin

## สารบัญ
1. [Reactive Concepts](#reactive-concepts)
2. [Spring WebFlux](#spring-webflux)
3. [Project Reactor](#project-reactor)
4. [R2DBC Reactive Database](#r2dbc-reactive-database)
5. [Reactive Kafka](#reactive-kafka)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Reactive Concepts

```
Reactive Programming:
- Asynchronous + Non-blocking
- Backpressure: consumer controls flow rate
- Event-driven data streams
- Better resource utilization (fewer threads)

Traditional (Blocking):
Thread 1: request -> wait for DB -> wait for API -> respond
Thread 2: request -> wait for DB -> wait for API -> respond
(ต้องใช้ thread จำนวนมาก)

Reactive (Non-blocking):
Event Loop: request -> [queue] -> DB callback -> [queue] -> API callback -> respond
Event Loop: request -> [queue] -> DB callback -> [queue] -> API callback -> respond
(ใช้ thread น้อย event loop handles many requests)
```

---

## Spring WebFlux

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webflux")
    implementation("org.springframework.boot:spring-boot-starter-data-r2dbc")
    implementation("io.r2dbc:r2dbc-postgresql")
    runtimeOnly("org.postgresql:r2dbc-postgresql")
}

// Reactive Controller
@RestController
@RequestMapping("/api/products")
class ProductController(private val productService: ReactiveProductService) {
    
    @GetMapping
    fun getAllProducts(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int
    ): Flux<ProductResponse> {
        return productService.findAll(page, size)
    }
    
    @GetMapping("/{id}")
    fun getProduct(@PathVariable id: String): Mono<ResponseEntity<ProductResponse>> {
        return productService.findById(id)
            .map { ResponseEntity.ok(it) }
            .defaultIfEmpty(ResponseEntity.notFound().build())
    }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun createProduct(@RequestBody @Valid request: Mono<CreateProductRequest>): Mono<ProductResponse> {
        return request.flatMap { productService.create(it) }
    }
    
    @PutMapping("/{id}")
    fun updateProduct(
        @PathVariable id: String,
        @RequestBody request: Mono<UpdateProductRequest>
    ): Mono<ResponseEntity<ProductResponse>> {
        return request.flatMap { productService.update(id, it) }
            .map { ResponseEntity.ok(it) }
            .defaultIfEmpty(ResponseEntity.notFound().build())
    }
    
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    fun deleteProduct(@PathVariable id: String): Mono<Void> {
        return productService.delete(id)
    }
    
    // Server-Sent Events
    @GetMapping("/stream", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
    fun streamProducts(): Flux<ServerSentEvent<ProductResponse>> {
        return productService.findAll(0, Int.MAX_VALUE)
            .map { product ->
                ServerSentEvent.builder(product)
                    .id(product.id)
                    .event("product")
                    .build()
            }
            .delayElements(Duration.ofMillis(100))
    }
}

// Data classes
data class ProductResponse(
    val id: String,
    val name: String,
    val price: Double,
    val stock: Int,
    val category: String
)

data class CreateProductRequest(
    @field:NotBlank val name: String,
    @field:Positive val price: Double,
    @field:PositiveOrZero val stock: Int,
    val category: String
)

data class UpdateProductRequest(
    val name: String?,
    val price: Double?,
    val stock: Int?
)
```

---

## Project Reactor Operators

```kotlin
import reactor.core.publisher.Flux
import reactor.core.publisher.Mono
import reactor.core.scheduler.Schedulers
import java.time.Duration

// Mono operations
fun monoExamples() {
    // Create Mono
    val fromValue = Mono.just("hello")
    val fromNull = Mono.justOrEmpty(null as String?)
    val empty = Mono.empty<String>()
    val error = Mono.error<String>(RuntimeException("oops"))
    val deferred = Mono.fromCallable { Thread.currentThread().name }
    val fromFuture = Mono.fromFuture(java.util.concurrent.CompletableFuture.completedFuture("value"))
    
    // Transform
    fromValue
        .map { it.uppercase() }
        .flatMap { value -> Mono.just("$value world") }
        .filter { it.length > 5 }
        .switchIfEmpty(Mono.just("default"))
        .onErrorReturn("fallback")
        .onErrorResume { e -> Mono.just("recovered: ${e.message}") }
        .timeout(Duration.ofSeconds(5))
        .doOnNext { println("value: $it") }
        .doOnError { println("error: $it") }
        .doOnTerminate { println("terminated") }
        .block()  // Only in tests! Never in reactive code
}

// Flux operations
fun fluxExamples() {
    // Create Flux
    val fromList = Flux.fromIterable(listOf(1, 2, 3, 4, 5))
    val range = Flux.range(1, 10)
    val interval = Flux.interval(Duration.ofSeconds(1)).take(5)
    val generate = Flux.generate<Int> { sink ->
        sink.next((1..100).random())
        if ((1..10).random() == 1) sink.complete()
    }
    
    // Transform
    fromList
        .map { it * 2 }
        .filter { it > 4 }
        .take(3)
        .skip(1)
        .sort()
        .distinct()
        .flatMap { value ->
            Mono.just(value * 10)
                .subscribeOn(Schedulers.boundedElastic())
        }
        .concatMap { value -> Mono.just(value) }  // ordered (slower than flatMap)
        .mergeWith(Flux.just(100, 200))
        .collectList()
        .block()
    
    // Aggregation
    range
        .reduce(0) { acc, n -> acc + n }  // sum
        .block()
    
    range
        .buffer(3)  // group into lists of 3
        .map { batch -> batch.sum() }
        .blockLast()
    
    range
        .window(3)  // window into Flux of Flux
        .flatMap { window -> window.collectList() }
        .blockLast()
    
    // Parallel processing
    range
        .parallel(4)  // 4 rails
        .runOn(Schedulers.parallel())
        .map { it * it }
        .sequential()
        .collectList()
        .block()
}

// Combining Mono and Flux
fun combineExamples() {
    val mono1 = Mono.just("A")
    val mono2 = Mono.just("B")
    val mono3 = Mono.just("C")
    
    // Zip: combine multiple Mono
    Mono.zip(mono1, mono2, mono3)
        .map { tuple -> "${tuple.t1}-${tuple.t2}-${tuple.t3}" }
        .block()
    
    // ZipWith
    mono1.zipWith(mono2)
        .map { tuple -> "${tuple.t1}-${tuple.t2}" }
        .block()
    
    // Flux zip
    val names = Flux.just("Alice", "Bob", "Carol")
    val ages = Flux.just(25, 30, 35)
    
    Flux.zip(names, ages)
        .map { tuple -> "${tuple.t1}: ${tuple.t2}" }
        .blockLast()
    
    // Merge (interleave)
    val fast = Flux.interval(Duration.ofMillis(100)).take(3)
    val slow = Flux.interval(Duration.ofMillis(200)).take(3)
    Flux.merge(fast, slow).blockLast()
    
    // Concat (sequential)
    Flux.concat(names, Flux.just("Dave")).blockLast()
}
```

---

## R2DBC Reactive Database

```kotlin
// Entity
@Table("products")
data class ProductEntity(
    @Id val id: String = java.util.UUID.randomUUID().toString(),
    val name: String,
    val price: Double,
    @Column("stock_quantity") val stock: Int,
    val category: String,
    @CreatedDate val createdAt: LocalDateTime = LocalDateTime.now(),
    @LastModifiedDate val updatedAt: LocalDateTime = LocalDateTime.now()
)

// Repository
@Repository
interface ReactiveProductRepository : ReactiveCrudRepository<ProductEntity, String> {
    
    fun findByCategory(category: String): Flux<ProductEntity>
    
    @Query("SELECT * FROM products WHERE price BETWEEN :minPrice AND :maxPrice")
    fun findByPriceRange(minPrice: Double, maxPrice: Double): Flux<ProductEntity>
    
    @Query("SELECT * FROM products WHERE stock_quantity < :threshold")
    fun findLowStock(threshold: Int): Flux<ProductEntity>
    
    @Query("SELECT COUNT(*) FROM products WHERE category = :category")
    fun countByCategory(category: String): Mono<Long>
    
    @Modifying
    @Query("UPDATE products SET stock_quantity = stock_quantity - :quantity WHERE id = :id AND stock_quantity >= :quantity")
    fun decrementStock(id: String, quantity: Int): Mono<Int>  // returns affected rows
}

// Service using R2DBC
@Service
class ReactiveProductService(
    private val repository: ReactiveProductRepository,
    private val transactionTemplate: TransactionalOperator
) {
    
    fun findAll(page: Int, size: Int): Flux<ProductResponse> {
        return repository.findAll()
            .skip((page * size).toLong())
            .take(size.toLong())
            .map { it.toResponse() }
    }
    
    fun findById(id: String): Mono<ProductResponse> {
        return repository.findById(id)
            .map { it.toResponse() }
    }
    
    fun create(request: CreateProductRequest): Mono<ProductResponse> {
        val entity = ProductEntity(
            name = request.name,
            price = request.price,
            stock = request.stock,
            category = request.category
        )
        return repository.save(entity).map { it.toResponse() }
    }
    
    fun update(id: String, request: UpdateProductRequest): Mono<ProductResponse> {
        return repository.findById(id)
            .flatMap { existing ->
                val updated = existing.copy(
                    name = request.name ?: existing.name,
                    price = request.price ?: existing.price,
                    stock = request.stock ?: existing.stock
                )
                repository.save(updated)
            }
            .map { it.toResponse() }
    }
    
    fun delete(id: String): Mono<Void> {
        return repository.deleteById(id)
    }
    
    // Transactional reactive
    fun purchaseProduct(productId: String, quantity: Int): Mono<PurchaseResult> {
        return transactionTemplate.transactional(
            repository.decrementStock(productId, quantity)
                .flatMap { affected ->
                    if (affected == 0) {
                        Mono.error(InsufficientStockException("Not enough stock"))
                    } else {
                        Mono.just(PurchaseResult(productId, quantity, "SUCCESS"))
                    }
                }
        )
    }
    
    private fun ProductEntity.toResponse() = ProductResponse(id, name, price, stock, category)
}

data class PurchaseResult(val productId: String, val quantity: Int, val status: String)
class InsufficientStockException(message: String) : Exception(message)
```

---

## Error Handling ใน Reactive

```kotlin
@RestControllerAdvice
class ReactiveExceptionHandler {
    
    @ExceptionHandler(InsufficientStockException::class)
    fun handleInsufficientStock(e: InsufficientStockException): ResponseEntity<ErrorResponse> {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(ErrorResponse("INSUFFICIENT_STOCK", e.message ?: "Not enough stock"))
    }
    
    @ExceptionHandler(WebExchangeBindException::class)
    fun handleValidation(e: WebExchangeBindException): ResponseEntity<ValidationErrorResponse> {
        val errors = e.bindingResult.fieldErrors.map { "${it.field}: ${it.defaultMessage}" }
        return ResponseEntity.badRequest()
            .body(ValidationErrorResponse("VALIDATION_ERROR", errors))
    }
}

// Retry with backoff
fun fetchWithRetry(client: WebClient, url: String): Mono<String> {
    return client.get()
        .uri(url)
        .retrieve()
        .bodyToMono(String::class.java)
        .retryWhen(
            Retry.backoff(3, Duration.ofSeconds(1))
                .maxBackoff(Duration.ofSeconds(10))
                .jitter(0.5)
                .filter { e -> e is java.io.IOException }
                .onRetryExhaustedThrow { _, signal ->
                    RuntimeException("Retry exhausted after ${signal.totalRetries()} attempts")
                }
        )
        .timeout(Duration.ofSeconds(30))
        .onErrorMap(java.util.concurrent.TimeoutException::class.java) { 
            RuntimeException("Request timed out") 
        }
}

data class ErrorResponse(val code: String, val message: String)
data class ValidationErrorResponse(val code: String, val errors: List<String>)

// WebClient
@Bean
fun webClient(): WebClient {
    return WebClient.builder()
        .baseUrl("https://api.example.com")
        .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .filter(ExchangeFilterFunction.ofRequestProcessor { request ->
            println("Request: ${request.method()} ${request.url()}")
            Mono.just(request)
        })
        .filter(ExchangeFilterFunction.ofResponseProcessor { response ->
            println("Response: ${response.statusCode()}")
            Mono.just(response)
        })
        .codecs { it.defaultCodecs().maxInMemorySize(16 * 1024 * 1024) }  // 16MB
        .build()
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Reactive order processing pipeline

@Service
class ReactiveOrderProcessor(
    private val orderRepository: ReactiveOrderRepository,
    private val productRepository: ReactiveProductRepository,
    private val paymentService: ReactivePaymentService,
    private val notificationService: ReactiveNotificationService
) {
    
    fun processOrder(request: PlaceOrderRequest): Mono<OrderResult> {
        // TODO: Implement reactive pipeline:
        // 1. Validate all products exist (use Flux.fromIterable + flatMap)
        // 2. Check stock availability for all items
        // 3. Reserve stock for all items (transactional)
        // 4. Process payment
        // 5. Create order record
        // 6. Send confirmation notification
        // 7. On any failure: compensate (release stock, refund payment)
        
        return Flux.fromIterable(request.items)
            .flatMap { item ->
                productRepository.findById(item.productId)
                    .switchIfEmpty(Mono.error(IllegalArgumentException("Product ${item.productId} not found")))
                    .filter { product -> product.stock >= item.quantity }
                    .switchIfEmpty(Mono.error(InsufficientStockException("Insufficient stock for ${item.productId}")))
            }
            .collectList()
            .flatMap { products ->
                // Process payment
                paymentService.charge(request.paymentToken, calculateTotal(request.items, products))
            }
            .flatMap { paymentId ->
                // Create order
                orderRepository.save(createOrderEntity(request, paymentId))
            }
            .flatMap { order ->
                // Notify customer
                notificationService.sendConfirmation(request.customerId, order.id)
                    .thenReturn(OrderResult(order.id, "SUCCESS"))
            }
            .onErrorResume { e ->
                // Compensation logic
                Mono.just(OrderResult("", "FAILED", e.message))
            }
    }
    
    private fun calculateTotal(items: List<OrderItemRequest>, products: List<ProductEntity>): Double {
        return items.sumOf { item ->
            val product = products.find { it.id == item.productId }!!
            product.price * item.quantity
        }
    }
    
    private fun createOrderEntity(request: PlaceOrderRequest, paymentId: String): ReactiveOrderEntity {
        TODO("Implement order entity creation")
    }
}

data class PlaceOrderRequest(
    val customerId: String,
    val items: List<OrderItemRequest>,
    val paymentToken: String
)

data class OrderItemRequest(val productId: String, val quantity: Int)

data class OrderResult(val orderId: String, val status: String, val message: String? = null)

interface ReactiveOrderRepository : ReactiveCrudRepository<ReactiveOrderEntity, String>
interface ReactivePaymentService { fun charge(token: String, amount: Double): Mono<String> }
interface ReactiveNotificationService { fun sendConfirmation(customerId: String, orderId: String): Mono<Void> }

// Placeholder
data class ReactiveOrderEntity(val id: String = "")

typealias ReactiveCrudRepository<T, ID> = org.springframework.data.repository.reactive.ReactiveCrudRepository<T, ID>
typealias TransactionalOperator = org.springframework.transaction.reactive.TransactionalOperator
typealias WebExchangeBindException = org.springframework.web.bind.support.WebExchangeBindException
typealias MediaType = org.springframework.http.MediaType
typealias HttpHeaders = org.springframework.http.HttpHeaders
typealias ResponseEntity<T> = org.springframework.http.ResponseEntity<T>
typealias HttpStatus = org.springframework.http.HttpStatus
typealias ServerSentEvent<T> = org.springframework.http.codec.ServerSentEvent<T>
```

---

## สรุป Part 43

```
✅ Reactive: non-blocking I/O, better thread utilization
✅ Mono<T>: 0 หรือ 1 value asynchronously
✅ Flux<T>: 0 ถึง N values (stream)
✅ Operators: map, flatMap, filter, zip, merge, concat
✅ Schedulers: parallel(), boundedElastic(), single()
✅ Spring WebFlux: reactive REST endpoints
✅ R2DBC: reactive database access (no JDBC blocking)
✅ Backpressure: consumer controls producer speed
✅ Error handling: onErrorReturn, onErrorResume, retryWhen
✅ SSE: Server-Sent Events สำหรับ real-time streaming
✅ WebClient: reactive HTTP client
✅ Transactional reactive: TransactionalOperator
```

---

*Part 43/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
