# Part 16: Coroutines พื้นฐาน

## สารบัญ
1. [Coroutines คืออะไร](#coroutines-คืออะไร)
2. [launch และ async](#launch-และ-async)
3. [suspend functions](#suspend-functions)
4. [Coroutine Context และ Dispatchers](#coroutine-context-และ-dispatchers)
5. [Job และ Cancellation](#job-และ-cancellation)
6. [Flow - Cold Streams](#flow---cold-streams)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Coroutines คืออะไร

Coroutines คือ lightweight thread ที่ suspend ได้โดยไม่ block thread จริง

```
Thread (Traditional)              Coroutine
┌─────────────────┐               ┌─────────────────┐
│ Thread 1        │               │ Thread 1        │
│  [Task A ████████████████████]  │  [Task A ████]  │
│                 │               │   ↓ suspend     │
│ Thread 2        │               │  [Task B ████]  │
│  [Task B ████████████████████]  │   ↓ resume      │
│                 │               │  [Task A ████]  │
└─────────────────┘               └─────────────────┘
     2 threads needed                  1 thread, 2 coroutines
```

### Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
}
```

---

## launch และ async

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // runBlocking: สร้าง CoroutineScope, block thread จนกว่า coroutines จะเสร็จ
    
    // launch: fire-and-forget, return Job
    val job = launch {
        delay(1000)  // suspend ไม่ block thread
        println("Coroutine finished!")
    }
    
    println("Main continues...")
    job.join()  // รอให้ job เสร็จ
    println("Done")
    
    // Output:
    // Main continues...
    // (after ~1 second)
    // Coroutine finished!
    // Done
}
```

### Sequential vs Concurrent

```kotlin
import kotlinx.coroutines.*

suspend fun fetchUser(): String {
    delay(1000)  // simulate network call
    return "User: สมชาย"
}

suspend fun fetchProfile(): String {
    delay(1200)  // simulate network call
    return "Profile: Engineer"
}

suspend fun fetchSettings(): String {
    delay(800)   // simulate network call
    return "Settings: Dark mode"
}

fun main() = runBlocking {
    // Sequential (total ~3 seconds)
    val start = System.currentTimeMillis()
    val user = fetchUser()
    val profile = fetchProfile()
    val settings = fetchSettings()
    println("Sequential: ${System.currentTimeMillis() - start}ms")
    
    // Concurrent with async (total ~1.2 seconds)
    val start2 = System.currentTimeMillis()
    val userDeferred = async { fetchUser() }
    val profileDeferred = async { fetchProfile() }
    val settingsDeferred = async { fetchSettings() }
    
    val user2 = userDeferred.await()
    val profile2 = profileDeferred.await()
    val settings2 = settingsDeferred.await()
    println("Concurrent: ${System.currentTimeMillis() - start2}ms")
    
    println("$user2, $profile2, $settings2")
    
    // Structured concurrency
    val result = coroutineScope {
        val a = async { fetchUser() }
        val b = async { fetchProfile() }
        "${a.await()} | ${b.await()}"
    }
    println(result)
    
    // launch multiple
    val jobs = (1..5).map { i ->
        launch {
            delay(500L * i)
            println("Job $i done at ${System.currentTimeMillis() - start2}ms")
        }
    }
    jobs.forEach { it.join() }
    
    // awaitAll
    val results = (1..5).map { i ->
        async {
            delay(100L * i)
            "Result $i"
        }
    }.awaitAll()
    println(results)  // [Result 1, Result 2, Result 3, Result 4, Result 5]
}
```

---

## suspend functions

```kotlin
import kotlinx.coroutines.*

// suspend function: สามารถ suspend ได้ใน coroutine
suspend fun fetchData(id: Int): String {
    delay(100)  // non-blocking delay
    return "Data #$id"
}

// suspend function สามารถเรียก suspend function อื่นได้
suspend fun processData(id: Int): String {
    val raw = fetchData(id)           // เรียก suspend function
    delay(50)
    return "$raw (processed)"
}

// suspend function กับ exception handling
suspend fun riskyOperation(): Result<String> = runCatching {
    delay(100)
    if (Math.random() < 0.5) throw Exception("Random failure!")
    "Success!"
}

// withContext: เปลี่ยน dispatcher ชั่วคราว
suspend fun heavyComputation(): Int = withContext(Dispatchers.Default) {
    // รันบน thread pool สำหรับ CPU-intensive work
    (1..1_000_000).sum()
}

// coroutineScope: สร้าง scope ใหม่, ต้องรอ children ทั้งหมด
suspend fun loadUserPage(userId: Int): Map<String, Any> = coroutineScope {
    val userDeferred = async { fetchData(userId) }
    val postsDeferred = async { 
        delay(200)
        listOf("Post 1", "Post 2")
    }
    val notificationsDeferred = async {
        delay(150)
        listOf("Notification 1")
    }
    
    mapOf(
        "user" to userDeferred.await(),
        "posts" to postsDeferred.await(),
        "notifications" to notificationsDeferred.await()
    )
}

fun main() = runBlocking {
    println(fetchData(42))
    println(processData(42))
    
    val result = riskyOperation()
    result.onSuccess { println("Got: $it") }
          .onFailure { println("Error: ${it.message}") }
    
    val sum = heavyComputation()
    println("Sum: $sum")
    
    val page = loadUserPage(1)
    println("Page: $page")
}
```

---

## Coroutine Context และ Dispatchers

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // Dispatchers.Default: CPU-intensive, thread pool = CPU cores
    launch(Dispatchers.Default) {
        println("Default: ${Thread.currentThread().name}")
    }
    
    // Dispatchers.IO: I/O operations, larger thread pool
    launch(Dispatchers.IO) {
        println("IO: ${Thread.currentThread().name}")
    }
    
    // Dispatchers.Main: UI thread (ใน Android)
    // launch(Dispatchers.Main) { ... }
    
    // Dispatchers.Unconfined: เริ่มใน current thread
    launch(Dispatchers.Unconfined) {
        println("Unconfined start: ${Thread.currentThread().name}")
        delay(100)
        println("Unconfined after delay: ${Thread.currentThread().name}")
    }
    
    // Custom thread pool
    val singleThread = newSingleThreadContext("MyThread")
    launch(singleThread) {
        println("Custom: ${Thread.currentThread().name}")
    }
    singleThread.close()
    
    // withContext: switch dispatcher
    val result = withContext(Dispatchers.Default) {
        // heavy computation
        (1..100_000).sum()
    }
    println("Result: $result")
    
    // CoroutineName
    launch(CoroutineName("MyCoroutine") + Dispatchers.Default) {
        println("Name: ${coroutineContext[CoroutineName]?.name}")
    }
    
    delay(200)
}
```

---

## Job และ Cancellation

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // Job lifecycle
    val job = launch {
        repeat(1000) { i ->
            println("Processing $i...")
            delay(100)
        }
    }
    
    delay(500)
    println("Cancelling job...")
    job.cancel()  // ส่ง CancellationException
    job.join()    // รอให้ cancel เสร็จ
    println("Cancelled")
    
    // isActive check
    val job2 = launch {
        var i = 0
        while (isActive) {  // check cancellation
            println("Work $i")
            i++
            delay(100)
        }
        println("Job2 done after cancellation")
    }
    
    delay(350)
    job2.cancel()
    job2.join()
    
    // Finally block runs on cancellation
    val job3 = launch {
        try {
            repeat(100) {
                delay(100)
                println("Working $it")
            }
        } finally {
            println("Cleanup!")  // ✅ always runs
        }
    }
    
    delay(250)
    job3.cancel()
    job3.join()
    
    // withTimeout
    try {
        withTimeout(300) {
            repeat(100) {
                delay(100)
                println("Timeout task $it")
            }
        }
    } catch (e: TimeoutCancellationException) {
        println("Timed out!")
    }
    
    // withTimeoutOrNull
    val result = withTimeoutOrNull(300) {
        delay(100)
        "Completed"
    }
    println("Result: $result")  // Completed (completed before timeout)
    
    val result2 = withTimeoutOrNull(50) {
        delay(100)
        "Completed"
    }
    println("Result2: $result2")  // null (timed out)
    
    // SupervisorJob: children independent
    val supervisor = SupervisorJob()
    val scope = CoroutineScope(coroutineContext + supervisor)
    
    val child1 = scope.launch {
        delay(100)
        throw RuntimeException("Child 1 failed!")
    }
    
    val child2 = scope.launch {
        delay(200)
        println("Child 2 completed")  // ✅ not affected by child1's failure
    }
    
    delay(300)
    supervisor.cancel()
}
```

---

## Flow - Cold Streams

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

// Flow คือ cold stream of values
fun simpleFlow(): Flow<Int> = flow {
    println("Flow started")
    for (i in 1..5) {
        delay(100)
        emit(i)  // emit value
    }
}

// flowOf
fun staticFlow() = flowOf(1, 2, 3, 4, 5)

// asFlow
fun listFlow() = (1..10).asFlow()

fun main() = runBlocking {
    // Collect flow
    simpleFlow().collect { value ->
        println("Received: $value")
    }
    
    // Flow operators
    (1..10).asFlow()
        .filter { it % 2 == 0 }
        .map { it * it }
        .take(3)
        .collect { println(it) }  // 4, 16, 36
    
    // transform
    (1..5).asFlow()
        .transform { value ->
            emit("Before $value")
            delay(50)
            emit("After $value")
        }
        .collect { println(it) }
    
    // flatMapMerge - concurrent
    (1..3).asFlow()
        .flatMapMerge { id ->
            flow {
                delay(100L * (4 - id))  // different delays
                emit("Result $id")
            }
        }
        .collect { println(it) }  // might print in different order
    
    // Terminal operators
    val sum = (1..100).asFlow().reduce { acc, n -> acc + n }
    println("Sum: $sum")
    
    val first = (1..10).asFlow().filter { it > 5 }.first()
    println("First > 5: $first")  // 6
    
    val list = (1..5).asFlow().toList()
    println("List: $list")
    
    // Error handling
    flow {
        emit(1)
        emit(2)
        throw RuntimeException("Error!")
    }.catch { e ->
        println("Caught: ${e.message}")
        emit(-1)  // emit fallback
    }.collect { println(it) }
    
    // Buffer and conflation
    flow {
        repeat(5) {
            delay(100)
            emit(it)
        }
    }.buffer(3)  // buffer 3 items
     .collect { value ->
         delay(200)  // slow collector
         println("Collected: $value")
     }
    
    // StateFlow (hot, always has value)
    val stateFlow = MutableStateFlow(0)
    val job = launch {
        stateFlow.collect { println("State: $it") }
    }
    
    delay(10)
    stateFlow.value = 1
    delay(10)
    stateFlow.value = 2
    delay(10)
    job.cancel()
    
    // SharedFlow (hot, broadcast)
    val sharedFlow = MutableSharedFlow<String>()
    val collector1 = launch {
        sharedFlow.collect { println("Collector 1: $it") }
    }
    val collector2 = launch {
        sharedFlow.collect { println("Collector 2: $it") }
    }
    
    delay(10)
    sharedFlow.emit("Hello")
    sharedFlow.emit("World")
    delay(10)
    collector1.cancel()
    collector2.cancel()
}
```

---

## ตัวอย่างโปรแกรม - Async News Fetcher

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

data class Article(
    val id: Int,
    val title: String,
    val category: String,
    val readTime: Int  // minutes
)

// Simulate news API
class NewsRepository {
    private val db = mapOf(
        "tech" to listOf(
            Article(1, "Kotlin 2.0 Released", "tech", 5),
            Article(2, "AI in 2024", "tech", 8)
        ),
        "sports" to listOf(
            Article(3, "World Cup 2026", "sports", 3),
            Article(4, "Olympics Preview", "sports", 6)
        ),
        "business" to listOf(
            Article(5, "Market Update", "business", 4),
            Article(6, "Startup Funding", "business", 7)
        )
    )
    
    suspend fun getArticles(category: String): List<Article> {
        delay(200)  // simulate network
        return db[category] ?: emptyList()
    }
    
    fun articlesFlow(categories: List<String>): Flow<Article> = flow {
        for (cat in categories) {
            delay(100)
            getArticles(cat).forEach { emit(it) }
        }
    }
}

fun main() = runBlocking {
    val repo = NewsRepository()
    
    // Concurrent fetch multiple categories
    val categories = listOf("tech", "sports", "business")
    
    val start = System.currentTimeMillis()
    val articles = categories.map { cat ->
        async { repo.getArticles(cat) }
    }.awaitAll().flatten()
    
    println("Fetched ${articles.size} articles in ${System.currentTimeMillis() - start}ms")
    
    // Flow-based
    repo.articlesFlow(categories)
        .filter { it.readTime <= 5 }
        .sortedBy { it.readTime }  // terminal - returns List
        .forEach { println("${it.title} (${it.readTime} min)") }
    
    // Group by category using flow
    val grouped = repo.articlesFlow(categories)
        .toList()
        .groupBy { it.category }
    
    grouped.forEach { (cat, arts) ->
        val avgTime = arts.map { it.readTime }.average()
        println("$cat: ${arts.size} articles, avg ${String.format("%.1f", avgTime)} min")
    }
}
```

---

## แบบฝึกหัด

### Exercise: Countdown Timer

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun countdownTimer(seconds: Int): Flow<Int> = flow {
    for (i in seconds downTo 0) {
        emit(i)
        if (i > 0) delay(1000)
    }
}

fun main() = runBlocking {
    println("=== Countdown Timer ===")
    
    countdownTimer(5).collect { seconds ->
        if (seconds > 0) {
            println("$seconds...")
        } else {
            println("🎉 Time's up!")
        }
    }
    
    // With cancel support
    val job = launch {
        countdownTimer(60)
            .onEach { println("Remaining: $it") }
            .collect()
    }
    
    delay(3500)  // Let it run for 3.5 seconds
    job.cancel()
    println("Timer cancelled")
}
```

---

## สรุป Part 16

```
✅ Coroutines: lightweight concurrency
✅ runBlocking: block thread สำหรับ testing/main
✅ launch: fire-and-forget, returns Job
✅ async/await: concurrent computation, returns Deferred
✅ suspend fun: สามารถ suspend ได้
✅ delay(): non-blocking wait
✅ withContext(): เปลี่ยน Dispatcher
✅ Dispatchers: Default, IO, Main, Unconfined
✅ Job.cancel(): ยกเลิก coroutine
✅ withTimeout/withTimeoutOrNull
✅ SupervisorJob: independent children
✅ Flow: cold stream of values
✅ flow { emit() }: สร้าง Flow
✅ collect { }: consume Flow
✅ StateFlow/SharedFlow: hot streams
✅ Flow operators: filter, map, transform, catch
```

---

*Part 16/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
