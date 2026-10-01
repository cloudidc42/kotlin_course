# Part 44: Coroutines Patterns ขั้นสูง

## สารบัญ
1. [Structured Concurrency Patterns](#structured-concurrency-patterns)
2. [Flow Advanced Patterns](#flow-advanced-patterns)
3. [Mutex และ Semaphore](#mutex-และ-semaphore)
4. [Coroutine Context](#coroutine-context)
5. [Testing Coroutines](#testing-coroutines)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Structured Concurrency Patterns

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.sync.Mutex
import kotlinx.coroutines.sync.Semaphore
import kotlinx.coroutines.sync.withLock
import kotlinx.coroutines.sync.withPermit

// Pattern 1: Parallel decomposition with error handling
suspend fun fetchUserDashboard(userId: String): Dashboard = coroutineScope {
    // Launch all requests in parallel
    val profileDeferred = async { userService.getProfile(userId) }
    val ordersDeferred = async { orderService.getRecentOrders(userId) }
    val recommendationsDeferred = async { 
        try {
            recommendationService.getRecommendations(userId)
        } catch (e: Exception) {
            emptyList()  // Non-critical: return empty on failure
        }
    }
    val notificationsDeferred = async { notificationService.getUnread(userId) }
    
    // Await all results
    Dashboard(
        profile = profileDeferred.await(),
        recentOrders = ordersDeferred.await(),
        recommendations = recommendationsDeferred.await(),
        unreadNotifications = notificationsDeferred.await()
    )
}

// Pattern 2: Fan-out with limited concurrency
suspend fun processOrders(orderIds: List<String>, concurrency: Int = 5): List<ProcessResult> {
    val semaphore = Semaphore(concurrency)
    
    return coroutineScope {
        orderIds.map { orderId ->
            async {
                semaphore.withPermit {
                    processOrder(orderId)
                }
            }
        }.awaitAll()
    }
}

// Pattern 3: Retry with exponential backoff
suspend fun <T> withRetry(
    maxAttempts: Int = 3,
    initialDelayMs: Long = 100,
    maxDelayMs: Long = 10_000,
    factor: Double = 2.0,
    retryOn: (Exception) -> Boolean = { true },
    block: suspend () -> T
): T {
    var currentDelay = initialDelayMs
    var lastException: Exception? = null
    
    repeat(maxAttempts) { attempt ->
        try {
            return block()
        } catch (e: Exception) {
            lastException = e
            if (!retryOn(e)) throw e
            
            if (attempt < maxAttempts - 1) {
                delay(currentDelay)
                currentDelay = (currentDelay * factor).toLong().coerceAtMost(maxDelayMs)
            }
        }
    }
    
    throw lastException!!
}

// Pattern 4: Timeout with fallback
suspend fun getProductWithFallback(id: String): Product {
    return withTimeoutOrNull(3000L) {
        productService.findById(id)
    } ?: productService.getCachedProduct(id)
        ?: Product.DEFAULT
}

// Pattern 5: Competing coroutines (race)
suspend fun <T> raceOf(vararg jobs: suspend () -> T): T = coroutineScope {
    val channel = Channel<T>(Channel.RENDEZVOUS)
    
    val deferred = jobs.map { job ->
        async {
            try {
                channel.send(job())
            } catch (e: Exception) {
                // ignore individual failures
            }
        }
    }
    
    val result = channel.receive()
    deferred.forEach { it.cancel() }
    result
}

// Pattern 6: Circuit breaker with coroutines
class CoroutineCircuitBreaker(
    private val failureThreshold: Int = 5,
    private val resetTimeoutMs: Long = 60_000
) {
    private var failures = 0
    private var lastFailureTime = 0L
    private var state = State.CLOSED
    private val mutex = Mutex()
    
    enum class State { CLOSED, OPEN, HALF_OPEN }
    
    suspend fun <T> execute(block: suspend () -> T): T {
        mutex.withLock {
            when (state) {
                State.OPEN -> {
                    if (System.currentTimeMillis() - lastFailureTime > resetTimeoutMs) {
                        state = State.HALF_OPEN
                    } else {
                        throw CircuitBreakerOpenException("Circuit breaker is OPEN")
                    }
                }
                else -> {}
            }
        }
        
        return try {
            val result = block()
            mutex.withLock {
                failures = 0
                state = State.CLOSED
            }
            result
        } catch (e: Exception) {
            mutex.withLock {
                failures++
                lastFailureTime = System.currentTimeMillis()
                if (failures >= failureThreshold) {
                    state = State.OPEN
                }
            }
            throw e
        }
    }
}

class CircuitBreakerOpenException(message: String) : Exception(message)

data class Dashboard(
    val profile: Any,
    val recentOrders: List<Any>,
    val recommendations: List<Any>,
    val unreadNotifications: List<Any>
)

data class ProcessResult(val orderId: String, val success: Boolean)
data class Product(val id: String = "") { companion object { val DEFAULT = Product("default") } }

// Placeholder services
interface UserService2 { suspend fun getProfile(id: String): Any }
interface OrderService2 { suspend fun getRecentOrders(id: String): List<Any> }
interface RecommendationService { suspend fun getRecommendations(id: String): List<Any> }
interface NotificationService3 { suspend fun getUnread(id: String): List<Any> }
interface ProductService2 {
    suspend fun findById(id: String): Product
    fun getCachedProduct(id: String): Product?
}
interface ProcessingService { suspend fun processOrder(id: String): ProcessResult }

val userService: UserService2 = TODO()
val orderService: OrderService2 = TODO()
val recommendationService: RecommendationService = TODO()
val notificationService: NotificationService3 = TODO()
val productService: ProductService2 = TODO()
suspend fun processOrder(id: String): ProcessResult = TODO()
```

---

## Flow Advanced Patterns

```kotlin
import kotlinx.coroutines.flow.*

// Pattern: State machine as Flow
sealed class LoginState {
    object Idle : LoginState()
    object Loading : LoginState()
    data class Success(val user: UserProfile) : LoginState()
    data class Error(val message: String) : LoginState()
}

data class UserProfile(val id: String, val name: String)

fun loginFlow(email: String, password: String): Flow<LoginState> = flow {
    emit(LoginState.Idle)
    emit(LoginState.Loading)
    
    try {
        delay(1000)  // simulate network
        val user = UserProfile("1", "Alice")
        emit(LoginState.Success(user))
    } catch (e: Exception) {
        emit(LoginState.Error(e.message ?: "Unknown error"))
    }
}

// Pattern: Hot Flow for shared state
class CounterViewModel {
    private val _state = MutableStateFlow(CounterState())
    val state: StateFlow<CounterState> = _state.asStateFlow()
    
    private val _events = MutableSharedFlow<UiEvent>()
    val events: SharedFlow<UiEvent> = _events.asSharedFlow()
    
    fun increment() {
        _state.update { it.copy(count = it.count + 1) }
    }
    
    fun decrement() {
        _state.update { current ->
            if (current.count > 0) current.copy(count = current.count - 1)
            else current
        }
    }
    
    suspend fun reset() {
        _state.value = CounterState()
        _events.emit(UiEvent.Snackbar("Counter reset!"))
    }
}

data class CounterState(val count: Int = 0, val isLoading: Boolean = false)
sealed class UiEvent {
    data class Snackbar(val message: String) : UiEvent()
    data class Navigate(val route: String) : UiEvent()
}

// Pattern: Flow with debounce for search
fun searchFlow(queryFlow: Flow<String>): Flow<List<SearchResult>> {
    return queryFlow
        .debounce(300L)  // wait 300ms after last keystroke
        .filter { it.length >= 2 }  // minimum 2 characters
        .distinctUntilChanged()  // skip if same query
        .flatMapLatest { query ->  // cancel previous search on new query
            flow {
                emit(emptyList())  // show loading
                emit(performSearch(query))
            }
        }
}

data class SearchResult(val id: String, val title: String)
suspend fun performSearch(query: String): List<SearchResult> = emptyList()

// Pattern: Buffering and batching
fun <T> Flow<T>.batchWithTimeout(size: Int, timeoutMs: Long): Flow<List<T>> = flow {
    val buffer = mutableListOf<T>()
    var job: Job? = null
    
    collect { item ->
        buffer.add(item)
        job?.cancel()
        
        if (buffer.size >= size) {
            emit(buffer.toList())
            buffer.clear()
        } else {
            job = coroutineScope {
                launch {
                    delay(timeoutMs)
                    if (buffer.isNotEmpty()) {
                        emit(buffer.toList())
                        buffer.clear()
                    }
                }
            }
        }
    }
    
    if (buffer.isNotEmpty()) emit(buffer.toList())
}

// Pattern: Flow error recovery
fun <T> Flow<T>.retryWithBackoff(
    maxRetries: Int = 3,
    initialDelayMs: Long = 100
): Flow<T> = retryWhen { cause, attempt ->
    if (attempt < maxRetries && cause is java.io.IOException) {
        delay(initialDelayMs * (attempt + 1))
        true
    } else {
        false
    }
}
```

---

## Mutex และ Semaphore

```kotlin
// Mutex: mutual exclusion (1 at a time)
class SafeCounter {
    private var count = 0
    private val mutex = Mutex()
    
    suspend fun increment() = mutex.withLock { count++ }
    suspend fun decrement() = mutex.withLock { count-- }
    suspend fun getCount() = mutex.withLock { count }
    
    // Optimistic locking pattern
    suspend fun incrementIfBelow(max: Int): Boolean = mutex.withLock {
        if (count < max) {
            count++
            true
        } else false
    }
}

// Semaphore: limit concurrent access (N at a time)
class ConnectionPool(private val maxConnections: Int) {
    private val semaphore = Semaphore(maxConnections)
    private val connections = mutableListOf<Connection>()
    
    suspend fun withConnection(block: suspend (Connection) -> Unit) {
        semaphore.withPermit {
            val connection = acquireConnection()
            try {
                block(connection)
            } finally {
                releaseConnection(connection)
            }
        }
    }
    
    private fun acquireConnection(): Connection = TODO()
    private fun releaseConnection(connection: Connection): Unit = TODO()
}

interface Connection

// Rate limiter using semaphore
class CoroutineRateLimiter(
    private val requestsPerPeriod: Int,
    private val period: Duration
) {
    private val semaphore = Semaphore(requestsPerPeriod)
    
    suspend fun execute(block: suspend () -> Unit) {
        semaphore.withPermit {
            try {
                block()
            } finally {
                // Refill after period
                launch {
                    delay(period.toMillis())
                    // semaphore refills automatically when withPermit exits
                }
            }
        }
    }
}
```

---

## Coroutine Context

```kotlin
import kotlinx.coroutines.*
import kotlin.coroutines.CoroutineContext

// Custom context element
class RequestId(val id: String) : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<RequestId>
    override val key: CoroutineContext.Key<*> = Key
}

class UserId(val id: String) : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<UserId>
    override val key: CoroutineContext.Key<*> = Key
}

// Access context in any coroutine
suspend fun processRequest() {
    val requestId = coroutineContext[RequestId]?.id ?: "unknown"
    val userId = coroutineContext[UserId]?.id ?: "anonymous"
    
    withContext(Dispatchers.IO) {
        // context propagates to child coroutines
        println("Processing request $requestId for user $userId")
    }
}

// Usage
suspend fun handleApiRequest(requestId: String, userId: String) {
    withContext(RequestId(requestId) + UserId(userId)) {
        processRequest()  // can access RequestId and UserId
    }
}

// MDC propagation with coroutines
class MDCCoroutineContext(
    private val contextMap: Map<String, String>
) : ThreadContextElement<Map<String, String>> {
    
    companion object Key : CoroutineContext.Key<MDCCoroutineContext>
    override val key: CoroutineContext.Key<*> = Key
    
    override fun updateThreadContext(context: CoroutineContext): Map<String, String> {
        val previous = org.slf4j.MDC.getCopyOfContextMap() ?: emptyMap()
        org.slf4j.MDC.setContextMap(contextMap)
        return previous
    }
    
    override fun restoreThreadContext(context: CoroutineContext, oldState: Map<String, String>) {
        org.slf4j.MDC.setContextMap(oldState)
    }
}

suspend fun withMdcContext(block: suspend () -> Unit) {
    val currentMdc = org.slf4j.MDC.getCopyOfContextMap() ?: emptyMap()
    withContext(MDCCoroutineContext(currentMdc)) {
        block()
    }
}
```

---

## Testing Coroutines

```kotlin
import kotlinx.coroutines.test.*
import org.junit.jupiter.api.Test
import kotlin.test.assertEquals
import kotlin.test.assertTrue

class CoroutineTest {
    
    // Standard test
    @Test
    fun `test simple coroutine`() = runTest {
        val result = async { 
            delay(1000)
            "hello"
        }.await()
        
        assertEquals("hello", result)
        // delay(1000) is virtual in runTest, doesn't actually wait
    }
    
    // Test StateFlow
    @Test
    fun `test counter view model`() = runTest {
        val viewModel = CounterViewModel()
        
        val states = mutableListOf<CounterState>()
        val job = launch {
            viewModel.state.take(3).toList(states)
        }
        
        viewModel.increment()
        viewModel.increment()
        advanceUntilIdle()
        job.join()
        
        assertEquals(3, states.size)  // initial + 2 increments
        assertEquals(0, states[0].count)
        assertEquals(1, states[1].count)
        assertEquals(2, states[2].count)
    }
    
    // Test with fake time
    @Test
    fun `test debounce search`() = runTest {
        val queries = MutableStateFlow("")
        val results = mutableListOf<List<SearchResult>>()
        
        val job = launch {
            searchFlow(queries).collect { results.add(it) }
        }
        
        queries.emit("a")    // too short, filtered
        queries.emit("ab")   // debounce starts
        advanceTimeBy(100)   // within debounce window
        queries.emit("abc")  // reset debounce
        advanceTimeBy(300)   // debounce fires
        
        job.cancel()
        assertTrue(results.size <= 1)  // only one search should fire
    }
    
    // Test retry behavior
    @Test
    fun `test retry with backoff`() = runTest {
        var attemptCount = 0
        
        val result = kotlin.runCatching {
            withRetry(maxAttempts = 3, initialDelayMs = 100) {
                attemptCount++
                if (attemptCount < 3) throw java.io.IOException("Network error")
                "success"
            }
        }
        
        assertEquals(3, attemptCount)
        assertEquals("success", result.getOrNull())
    }
    
    // Test flow with TestScope
    @Test
    fun `test flow emission order`() = runTest {
        val flow = loginFlow("test@email.com", "password")
        val states = flow.toList()
        
        assertTrue(states[0] is LoginState.Idle)
        assertTrue(states[1] is LoginState.Loading)
        assertTrue(states.last() is LoginState.Success || states.last() is LoginState.Error)
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement a concurrent download manager

class DownloadManager(
    private val maxConcurrent: Int = 3,
    private val maxRetries: Int = 3
) {
    private val semaphore = Semaphore(maxConcurrent)
    private val _progress = MutableSharedFlow<DownloadProgress>()
    val progress: SharedFlow<DownloadProgress> = _progress.asSharedFlow()
    
    suspend fun downloadAll(urls: List<String>): List<DownloadResult> = coroutineScope {
        urls.mapIndexed { index, url ->
            async {
                semaphore.withPermit {
                    downloadWithRetry(url, index)
                }
            }
        }.awaitAll()
    }
    
    private suspend fun downloadWithRetry(url: String, id: Int): DownloadResult {
        // TODO: Implement with withRetry()
        // - Emit DownloadProgress.Started before starting
        // - Emit DownloadProgress.Progress with percentage
        // - Emit DownloadProgress.Completed or DownloadProgress.Failed
        // - Retry on IOException
        TODO("Implement download with retry and progress reporting")
    }
}

sealed class DownloadProgress {
    data class Started(val id: Int, val url: String) : DownloadProgress()
    data class Progress(val id: Int, val percent: Int) : DownloadProgress()
    data class Completed(val id: Int, val path: String) : DownloadProgress()
    data class Failed(val id: Int, val error: String) : DownloadProgress()
}

data class DownloadResult(
    val id: Int,
    val url: String,
    val success: Boolean,
    val path: String?,
    val error: String?
)

// Test your implementation
suspend fun main() {
    val manager = DownloadManager(maxConcurrent = 2)
    
    val progressJob = CoroutineScope(Dispatchers.Default).launch {
        manager.progress.collect { progress ->
            println("Progress: $progress")
        }
    }
    
    val results = manager.downloadAll(listOf(
        "https://example.com/file1.zip",
        "https://example.com/file2.zip",
        "https://example.com/file3.zip",
        "https://example.com/file4.zip"
    ))
    
    progressJob.cancel()
    
    results.forEach { result ->
        if (result.success) println("✅ Downloaded: ${result.url}")
        else println("❌ Failed: ${result.url} - ${result.error}")
    }
}
```

---

## สรุป Part 44

```
✅ Parallel decomposition: async/await for concurrent requests
✅ Fan-out with Semaphore: limit concurrency
✅ Retry with exponential backoff
✅ Timeout with fallback: withTimeoutOrNull
✅ Race condition: take first result
✅ Circuit breaker: fail-fast pattern
✅ Flow debounce: avoid unnecessary calls
✅ Flow flatMapLatest: cancel on new value
✅ StateFlow: observable state
✅ SharedFlow: event broadcasting
✅ Mutex: single-access critical section
✅ Semaphore: N-concurrent access control
✅ CoroutineContext: propagate metadata
✅ runTest: virtual time in coroutine tests
```

---

*Part 44/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
