# Part 78: Advanced Coroutines Patterns

## สารบัญ
1. [Structured Concurrency](#structured-concurrency)
2. [Channels](#channels)
3. [Flow Operators Advanced](#flow-operators-advanced)
4. [Coroutine Cancellation](#coroutine-cancellation)
5. [Custom CoroutineContext](#custom-coroutinescope)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Structured Concurrency

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*
import kotlin.time.Duration.Companion.seconds
import kotlin.time.Duration.Companion.milliseconds

// Structured Concurrency: parent จัดการ lifecycle ของ children
// ถ้า parent cancelled → children ทั้งหมด cancelled ด้วย

// Bad: fire and forget (unstructured)
fun badExample() {
    CoroutineScope(Dispatchers.IO).launch {  // Orphan coroutine!
        // This runs without any parent tracking it
        expensiveOperation()
    }
}

// Good: structured
class OrderService(private val scope: CoroutineScope) {
    
    // Runs within service's scope - cancelled when service destroyed
    fun processOrdersAsync(orderIds: List<String>): Deferred<List<Order>> {
        return scope.async {
            orderIds.map { id ->
                async { fetchOrder(id) }
            }.awaitAll()
        }
    }
}

// SupervisorJob: children fail independently
class BackgroundJobManager(private val scope: CoroutineScope) {
    
    private val supervisor = SupervisorJob()
    private val supervisorScope = CoroutineScope(scope.coroutineContext + supervisor)
    
    fun launchIndependentTask(name: String, block: suspend () -> Unit): Job {
        return supervisorScope.launch {
            try {
                block()
            } catch (e: Exception) {
                println("Task '$name' failed: ${e.message}")
                // Other tasks continue running
            }
        }
    }
}

// coroutineScope vs supervisorScope
suspend fun processWithSupervisor(): List<Result<Int>> {
    return supervisorScope {
        val tasks = listOf(
            async { computeValue(1) },  // Might fail
            async { computeValue(2) },
            async { computeValue(3) }   // Might fail
        )
        
        tasks.map { deferred ->
            try {
                Result.success(deferred.await())
            } catch (e: Exception) {
                Result.failure(e)  // Capture individual failures
            }
        }
    }
}

// coroutineScope: ONE failure cancels ALL siblings
suspend fun processWithCoroutineScope(): List<Int> {
    return coroutineScope {
        listOf(1, 2, 3).map { id ->
            async { computeValue(id) }  // One failure → all cancelled
        }.awaitAll()
    }
}

private suspend fun fetchOrder(id: String): Order = Order(id, "user", emptyList())
private suspend fun expensiveOperation() = delay(1000)
private suspend fun computeValue(n: Int): Int {
    delay(100)
    if (n == 2) throw RuntimeException("Failed for $n")
    return n * 10
}

data class Order(val id: String, val userId: String, val items: List<Any>)
```

---

## Channels

```kotlin
// Channel: coroutine-safe queue สำหรับ communicate ระหว่าง coroutines

// Producer-Consumer pattern
fun CoroutineScope.orderProducer(orderIds: List<String>): ReceiveChannel<String> {
    return produce(capacity = 10) {
        for (orderId in orderIds) {
            send(orderId)
            println("Produced: $orderId")
        }
    }
}

suspend fun processOrders() {
    coroutineScope {
        val channel = orderProducer(listOf("O1", "O2", "O3", "O4", "O5"))
        
        // Multiple consumers
        repeat(3) { consumerId ->
            launch {
                for (orderId in channel) {
                    println("Consumer $consumerId processing: $orderId")
                    delay(100)
                }
            }
        }
    }
}

// Pipeline: connect coroutines with channels
fun CoroutineScope.generateNumbers(count: Int): ReceiveChannel<Int> = produce {
    repeat(count) { send(it + 1) }
}

fun CoroutineScope.filterEven(input: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (value in input) {
        if (value % 2 == 0) send(value)
    }
}

fun CoroutineScope.doubleValues(input: ReceiveChannel<Int>): ReceiveChannel<Int> = produce {
    for (value in input) {
        send(value * 2)
    }
}

suspend fun pipelineExample() = coroutineScope {
    val numbers = generateNumbers(10)
    val evens = filterEven(numbers)
    val doubled = doubleValues(evens)
    
    for (value in doubled) {
        print("$value ")  // 4 8 12 16 20
    }
}

// Fan-out: one producer, many consumers
suspend fun fanOutExample() {
    val channel = Channel<Int>(capacity = Channel.BUFFERED)
    
    coroutineScope {
        // Producer
        launch {
            repeat(10) { i ->
                channel.send(i)
            }
            channel.close()
        }
        
        // Multiple consumers
        repeat(3) { consumerId ->
            launch {
                for (item in channel) {
                    println("Worker $consumerId: $item")
                    delay(50)
                }
            }
        }
    }
}

// Fan-in: multiple producers, one consumer
suspend fun fanInExample(): List<String> {
    val results = mutableListOf<String>()
    
    coroutineScope {
        val resultChannel = Channel<String>(Channel.UNLIMITED)
        
        // Multiple producers
        val sources = listOf("API", "DB", "Cache")
        sources.forEach { source ->
            launch {
                delay((100..500).random().toLong())
                resultChannel.send("$source: result")
            }
        }
        
        // Wait for all, then close
        launch {
            // Close after all producers finish
            delay(1000)
            resultChannel.close()
        }
        
        // Single consumer
        for (result in resultChannel) {
            results.add(result)
        }
    }
    
    return results
}

// Ticker channel: periodic events
fun CoroutineScope.ticker(delayMs: Long): ReceiveChannel<Unit> = produce {
    while (isActive) {
        delay(delayMs)
        send(Unit)
    }
}

suspend fun heartbeatExample() {
    coroutineScope {
        val ticker = ticker(1000)
        
        // Run for 5 seconds
        withTimeout(5000) {
            for (tick in ticker) {
                println("Heartbeat at ${System.currentTimeMillis()}")
            }
        }
        
        ticker.cancel()
    }
}
```

---

## Flow Operators Advanced

```kotlin
// Advanced Flow operators

// flatMapMerge: concurrent flat mapping
suspend fun flatMapMergeExample() {
    flowOf(1, 2, 3, 4, 5)
        .flatMapMerge(concurrency = 3) { id ->
            flow {
                delay((50..200).random().toLong())
                emit("result-$id")
            }
        }
        .collect { println(it) }  // Unordered, concurrent
}

// flatMapConcat: sequential flat mapping (preserves order)
suspend fun flatMapConcatExample() {
    flowOf(1, 2, 3)
        .flatMapConcat { id ->
            flow {
                delay(100)
                emit("first-$id")
                emit("second-$id")
            }
        }
        .collect { println(it) }  // 1,1, 2,2, 3,3 (ordered)
}

// flatMapLatest: cancel previous when new value arrives
suspend fun flatMapLatestExample() {
    flowOf(1, 2, 3)
        .flatMapLatest { id ->
            flow {
                println("Starting $id")
                delay(500)  // Will be cancelled by next value
                emit("done-$id")
            }
        }
        .collect { println(it) }  // Only "done-3" (others cancelled)
}

// combine: combine latest from multiple flows
suspend fun combineExample() {
    val prices = flow {
        var price = 100.0
        while (true) {
            emit(price)
            delay(200)
            price += (Math.random() - 0.5) * 10
        }
    }
    
    val quantities = flow {
        var qty = 10
        while (true) {
            emit(qty)
            delay(500)
            qty += (Math.random() * 5).toInt()
        }
    }
    
    prices.combine(quantities) { price, qty ->
        "Total: ${price * qty}"
    }
        .take(5)
        .collect { println(it) }
}

// zip: pair up emissions one-to-one
suspend fun zipExample() {
    val names = flowOf("Alice", "Bob", "Charlie")
    val scores = flowOf(85, 92, 78)
    
    names.zip(scores) { name, score -> "$name: $score" }
        .collect { println(it) }
}

// buffer: decouple producer from consumer speed
suspend fun bufferExample() {
    flow {
        repeat(10) { i ->
            println("Emitting $i")
            emit(i)
            delay(100)  // Producer delay
        }
    }
        .buffer(capacity = 5)  // Buffer up to 5 items
        .collect { value ->
            delay(300)  // Consumer is slow
            println("Collected $value")
        }
}

// conflate: drop intermediate values (only keep latest)
suspend fun conflateExample() {
    flow {
        repeat(100) { i ->
            emit(i)
            delay(10)
        }
    }
        .conflate()
        .collect { value ->
            delay(100)  // Slow consumer
            println("Processing: $value")  // Skips many values
        }
}

// debounce: only emit after quiet period (search-as-you-type)
suspend fun debounceExample() {
    val searchQueries = flow {
        emit("k")
        delay(50)
        emit("ko")
        delay(50)
        emit("kot")
        delay(50)
        emit("kotl")
        delay(500)  // Quiet period
        emit("kotlin")
    }
    
    searchQueries
        .debounce(300.milliseconds)
        .collect { query ->
            println("Searching: $query")  // Only "kotlin"
        }
}

// retry with exponential backoff
suspend fun retryExample() {
    flow {
        emit(fetchFromApi())
    }
        .retry(3) { cause ->
            cause is java.io.IOException
        }
        .retryWhen { cause, attempt ->
            if (cause is java.io.IOException && attempt < 3) {
                delay((attempt + 1) * 1000)  // 1s, 2s, 3s
                true
            } else {
                false
            }
        }
        .catch { e -> emit("fallback") }
        .collect { result -> println(result) }
}

private suspend fun fetchFromApi(): String = "data"

// Custom Flow operators
fun <T> Flow<T>.rateLimit(maxPerSecond: Int): Flow<T> = flow {
    val interval = 1000L / maxPerSecond
    collect { value ->
        delay(interval)
        emit(value)
    }
}

fun <T> Flow<T>.chunked(size: Int): Flow<List<T>> = flow {
    val buffer = mutableListOf<T>()
    collect { value ->
        buffer.add(value)
        if (buffer.size >= size) {
            emit(buffer.toList())
            buffer.clear()
        }
    }
    if (buffer.isNotEmpty()) emit(buffer.toList())
}

// StateFlow vs SharedFlow
class ProductViewModel(private val repository: ProductRepository) {
    
    // StateFlow: single current value, always has initial state
    private val _products = MutableStateFlow<List<Product>>(emptyList())
    val products: StateFlow<List<Product>> = _products.asStateFlow()
    
    // SharedFlow: events, no initial value
    private val _events = MutableSharedFlow<UIEvent>(
        extraBufferCapacity = 64
    )
    val events: SharedFlow<UIEvent> = _events.asSharedFlow()
    
    suspend fun loadProducts() {
        repository.getProducts()
            .onEach { products -> _products.value = products }
            .catch { e -> _events.emit(UIEvent.Error(e.message ?: "Unknown error")) }
            .collect()
    }
}

sealed class UIEvent {
    data class Error(val message: String) : UIEvent()
    data class NavigateTo(val screen: String) : UIEvent()
}

interface ProductRepository {
    fun getProducts(): Flow<List<Product>>
}

data class Product(val id: Long, val name: String)
```

---

## Coroutine Cancellation

```kotlin
// Cancellation is cooperative in Kotlin coroutines

// Properly cancellable coroutine
suspend fun cancellableOperation() {
    // Check for cancellation explicitly
    for (i in 1..1000) {
        ensureActive()  // Throws CancellationException if cancelled
        
        // Or use isActive
        if (!coroutineContext.isActive) return
        
        processItem(i)
    }
}

// withTimeout: cancel after deadline
suspend fun timedOperation(): String {
    return try {
        withTimeout(5000) {  // 5 second timeout
            someSlowOperation()
        }
    } catch (e: TimeoutCancellationException) {
        "Operation timed out"
    }
}

// withTimeoutOrNull: null instead of exception
suspend fun timedOperationOrNull(): String? {
    return withTimeoutOrNull(5000) {
        someSlowOperation()
    }
}

// NonCancellable: run cleanup even during cancellation
suspend fun operationWithCleanup() {
    try {
        doSomething()
    } finally {
        withContext(NonCancellable) {
            // This runs even if coroutine is cancelled
            cleanup()
        }
    }
}

// Custom cancellation handling
class ResourceManager : AutoCloseable {
    private val job = SupervisorJob()
    private val scope = CoroutineScope(Dispatchers.IO + job)
    
    private val resources = mutableListOf<String>()
    
    fun acquire(resource: String) {
        resources.add(resource)
        println("Acquired: $resource")
    }
    
    override fun close() {
        println("Releasing ${resources.size} resources...")
        resources.forEach { println("Released: $it") }
        resources.clear()
        job.cancel()
    }
}

// Usage
suspend fun manageResources() {
    ResourceManager().use { manager ->
        coroutineScope {
            launch {
                manager.acquire("connection-1")
                delay(1000)
            }
            launch {
                manager.acquire("connection-2")
                delay(500)
            }
        }
    }  // Cleanup called here
}

private suspend fun processItem(n: Int) = delay(1)
private suspend fun someSlowOperation(): String { delay(1000); return "done" }
private suspend fun doSomething() = delay(100)
private suspend fun cleanup() { println("Cleanup done") }
```

---

## Custom CoroutineContext

```kotlin
// CoroutineContext เก็บ coroutine metadata
// แต่ละ element ระบุด้วย Key

// Custom context element: request ID propagation
class RequestIdElement(val requestId: String) : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<RequestIdElement>
    override val key get() = Key
}

val CoroutineContext.requestId: String?
    get() = this[RequestIdElement]?.requestId

// Usage
suspend fun handleRequest(requestId: String) {
    withContext(RequestIdElement(requestId)) {
        processStage1()
        processStage2()
    }
}

suspend fun processStage1() {
    val requestId = coroutineContext.requestId
    println("Stage 1 [requestId=$requestId]")
    // RequestId is automatically available in nested coroutines
    coroutineScope {
        launch { println("Nested [requestId=${coroutineContext.requestId}]") }
    }
}

suspend fun processStage2() {
    println("Stage 2 [requestId=${coroutineContext.requestId}]")
}

// Coroutine exception handler
val globalExceptionHandler = CoroutineExceptionHandler { context, exception ->
    println("Unhandled exception in ${context[CoroutineName]?.name}: $exception")
    // Log to monitoring system, Sentry, etc.
}

fun createApplicationScope(): CoroutineScope {
    return CoroutineScope(
        SupervisorJob() +
        Dispatchers.Default +
        CoroutineName("AppScope") +
        globalExceptionHandler
    )
}

// Spring Boot + Coroutines integration
@Service
class CoroutineOrderService(
    private val orderRepository: OrderRepository,
    private val emailService: EmailService
) {
    
    // Transaction-aware coroutine dispatchers
    @Transactional
    suspend fun placeOrder(request: PlaceOrderRequest): Order {
        // Transaction is propagated to all coroutines via MDC
        val order = orderRepository.save(
            Order(
                id = java.util.UUID.randomUUID().toString(),
                userId = request.userId,
                items = request.items
            )
        )
        
        // Run notification in background (non-blocking)
        CoroutineScope(Dispatchers.IO).launch {
            emailService.sendConfirmation(order)
        }
        
        return order
    }
    
    suspend fun getOrders(userId: String): Flow<Order> {
        return orderRepository.findByUserId(userId)
    }
}

interface OrderRepository {
    suspend fun save(order: Order): Order
    fun findByUserId(userId: String): Flow<Order>
}

interface EmailService {
    suspend fun sendConfirmation(order: Order)
}

data class PlaceOrderRequest(val userId: String, val items: List<Any>)

typealias Service = org.springframework.stereotype.Service
typealias Transactional = org.springframework.transaction.annotation.Transactional
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement rate-limited API client ด้วย Coroutines

class RateLimitedApiClient(
    private val maxRequestsPerSecond: Int,
    private val httpClient: HttpClient
) {
    
    private val semaphore = Semaphore(maxRequestsPerSecond)
    private val requestTimes = ArrayDeque<Long>()
    
    // Rate limiting with semaphore
    suspend fun <T> callWithRateLimit(block: suspend () -> T): T {
        semaphore.acquire()
        return try {
            block()
        } finally {
            // Release after 1 second to limit rate
            CoroutineScope(Dispatchers.IO).launch {
                delay(1000)
                semaphore.release()
            }
        }
    }
    
    // Retry with exponential backoff
    suspend fun <T> callWithRetry(
        maxAttempts: Int = 3,
        block: suspend () -> T
    ): T {
        var lastException: Exception? = null
        
        repeat(maxAttempts) { attempt ->
            try {
                return callWithRateLimit(block)
            } catch (e: RateLimitException) {
                val backoffMs = (Math.pow(2.0, attempt.toDouble()) * 1000).toLong()
                delay(backoffMs)
                lastException = e
            } catch (e: Exception) {
                throw e  // Non-rate-limit errors throw immediately
            }
        }
        
        throw lastException ?: RuntimeException("Unknown error")
    }
    
    // Batch API calls with concurrency control
    suspend fun <T, R> batchCall(
        items: List<T>,
        concurrency: Int = 5,
        block: suspend (T) -> R
    ): List<R> {
        return coroutineScope {
            items.chunked(concurrency).flatMap { batch ->
                batch.map { item -> async { callWithRetry { block(item) } } }
                    .awaitAll()
            }
        }
    }
}

class RateLimitException(message: String) : Exception(message)
class HttpClient
typealias Semaphore = kotlinx.coroutines.sync.Semaphore
```

---

## สรุป Part 78

```
✅ Structured Concurrency: parent controls children lifecycle
✅ SupervisorJob: children fail independently
✅ coroutineScope vs supervisorScope: failure propagation
✅ produce { send() }: typed producer coroutine
✅ Channel(capacity): BUFFERED, UNLIMITED, CONFLATED
✅ Pipeline: produce chains connected by channels
✅ Fan-out: one channel, multiple consumer coroutines
✅ Fan-in: multiple producers, one receiver channel
✅ flatMapMerge: concurrent, unordered mapping
✅ flatMapConcat: sequential, ordered mapping
✅ flatMapLatest: cancel previous on new value
✅ combine: latest values from multiple flows
✅ zip: pair emissions one-to-one
✅ buffer(5): decouple producer/consumer speeds
✅ conflate(): keep only latest value
✅ debounce(300ms): emit after quiet period
✅ retryWhen: conditional retry with backoff
✅ chunked: custom flow operator
✅ StateFlow: current value with initial state
✅ SharedFlow: event bus without initial value
✅ ensureActive(): cooperative cancellation check
✅ withTimeout: cancel after deadline
✅ NonCancellable: cleanup runs despite cancellation
✅ CoroutineContext.Element: custom context propagation
✅ CoroutineExceptionHandler: global error handler
✅ Semaphore: rate limiting in coroutines
```

---

*Part 78/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
