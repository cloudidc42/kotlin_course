# Part 33: Performance Optimization

## สารบัญ
1. [Profiling และ Benchmarking](#profiling-และ-benchmarking)
2. [Memory Optimization](#memory-optimization)
3. [Coroutines Performance](#coroutines-performance)
4. [Data Structure Optimization](#data-structure-optimization)
5. [Lazy Evaluation](#lazy-evaluation)
6. [Caching Strategies](#caching-strategies)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Profiling และ Benchmarking

```kotlin
import kotlin.system.measureTimeMillis
import kotlin.system.measureNanoTime

// Simple benchmarking
fun <T> benchmark(name: String, iterations: Int = 1000, block: () -> T): T {
    // Warmup
    repeat(100) { block() }
    
    val times = LongArray(iterations)
    var result: T? = null
    
    for (i in times.indices) {
        times[i] = measureNanoTime { result = block() }
    }
    
    times.sort()
    val avg = times.average().toLong()
    val p50 = times[iterations / 2]
    val p95 = times[(iterations * 0.95).toInt()]
    val p99 = times[(iterations * 0.99).toInt()]
    
    println("=== $name ===")
    println("  Avg: ${avg}ns (${avg/1000}μs)")
    println("  P50: ${p50}ns")
    println("  P95: ${p95}ns")
    println("  P99: ${p99}ns")
    
    return result!!
}

// JMH-style benchmark (add JMH dependency for proper benchmarking)
// @BenchmarkMode(Mode.AverageTime)
// @OutputTimeUnit(TimeUnit.MICROSECONDS)
// @State(Scope.Thread)
// @Fork(2)
// @Warmup(iterations = 5)
// @Measurement(iterations = 10)
// class MyBenchmark {
//     @Benchmark
//     fun stringConcatenation(): String {
//         var s = ""
//         repeat(1000) { s += "a" }
//         return s
//     }
//
//     @Benchmark
//     fun stringBuilder(): String {
//         val sb = StringBuilder()
//         repeat(1000) { sb.append("a") }
//         return sb.toString()
//     }
// }

fun main() {
    // String concatenation comparison
    val iterations = 100
    
    benchmark("String concat (+=)", 100) {
        var s = ""
        repeat(iterations) { s += "a" }
        s
    }
    
    benchmark("StringBuilder", 100) {
        val sb = StringBuilder()
        repeat(iterations) { sb.append("a") }
        sb.toString()
    }
    
    benchmark("buildString DSL", 100) {
        buildString {
            repeat(iterations) { append("a") }
        }
    }
    
    println()
    
    // Collection operations
    val list = (1..100_000).toList()
    
    benchmark("filter+map (eager)", 10) {
        list.filter { it % 2 == 0 }.map { it * it }.take(100)
    }
    
    benchmark("asSequence (lazy)", 10) {
        list.asSequence().filter { it % 2 == 0 }.map { it * it }.take(100).toList()
    }
    
    println()
    
    // Allocation comparison
    benchmark("ArrayList with add", 100) {
        val list = ArrayList<Int>()
        repeat(1000) { list.add(it) }
        list
    }
    
    benchmark("ArrayList with capacity", 100) {
        val list = ArrayList<Int>(1000)
        repeat(1000) { list.add(it) }
        list
    }
    
    benchmark("buildList", 100) {
        buildList(1000) {
            repeat(1000) { add(it) }
        }
    }
}
```

---

## Memory Optimization

```kotlin
// 1. Value classes: zero-cost abstraction
@JvmInline
value class UserId(val value: Long)  // เหมือน Long ทั่วไป, ไม่มี object overhead

@JvmInline
value class Meters(val value: Double) {
    operator fun plus(other: Meters) = Meters(value + other.value)
    fun toKilometers() = Kilometers(value / 1000)
}

@JvmInline
value class Kilometers(val value: Double)

// 2. Object pools: reuse objects
class ObjectPool<T>(private val factory: () -> T, private val reset: T.() -> Unit = {}) {
    private val pool = ArrayDeque<T>()
    private var created = 0
    
    fun acquire(): T {
        return pool.removeLastOrNull() ?: factory().also { created++ }
    }
    
    fun release(obj: T) {
        obj.reset()
        pool.addLast(obj)
    }
    
    fun <R> use(block: (T) -> R): R {
        val obj = acquire()
        return try {
            block(obj)
        } finally {
            release(obj)
        }
    }
    
    val stats get() = "created=$created, pooled=${pool.size}"
}

// StringBuilder pool
val stringBuilderPool = ObjectPool(
    factory = { StringBuilder(256) },
    reset = { clear() }
)

// 3. Lazy properties: only compute when needed
class HeavyObject {
    val data1: List<Int> by lazy { (1..1_000_000).toList() }
    val data2: Map<Int, Int> by lazy { (1..1_000).associate { it to it * it } }
    
    // Only load if accessed
    val expensiveComputation: Double by lazy {
        data1.filter { it % 7 == 0 }.sumOf { it.toDouble() }
    }
}

// 4. WeakReference for caches
import java.lang.ref.WeakReference

class WeakCache<K, V : Any> {
    private val cache = HashMap<K, WeakReference<V>>()
    
    operator fun get(key: K): V? = cache[key]?.get()
    
    operator fun set(key: K, value: V) {
        cache[key] = WeakReference(value)
    }
    
    fun getOrPut(key: K, default: () -> V): V {
        return get(key) ?: default().also { set(key, it) }
    }
    
    fun cleanup() {
        cache.entries.removeIf { it.value.get() == null }
    }
    
    val size: Int get() = cache.count { it.value.get() != null }
}

// 5. Avoid boxing: use primitive arrays when possible
fun sumBoxed(list: List<Int>): Int = list.sum()

fun sumPrimitive(arr: IntArray): Int = arr.sum()

fun main() {
    // Value class: no overhead
    val id = UserId(12345L)
    val distance = Meters(1500.0)
    println("$distance = ${distance.toKilometers()}")
    
    // Object pool
    val results = mutableListOf<String>()
    repeat(100) {
        stringBuilderPool.use { sb ->
            sb.append("Result: ").append(it)
            results.add(sb.toString())
        }
    }
    println("Pool: ${stringBuilderPool.stats}")
    
    // Memory comparison
    val n = 100_000
    val boxed = List(n) { it }  // List<Int> = boxed
    val primitive = IntArray(n) { it }  // IntArray = primitive
    
    println("\nBoxed sum: ${sumBoxed(boxed)}")
    println("Primitive sum: ${sumPrimitive(primitive)}")
    
    // Benchmark both
    val timeBoxed = measureTimeMillis { repeat(100) { sumBoxed(boxed) } }
    val timePrimitive = measureTimeMillis { repeat(100) { sumPrimitive(primitive) } }
    
    println("Boxed time: ${timeBoxed}ms")
    println("Primitive time: ${timePrimitive}ms")
    println("Speedup: ${timeBoxed.toDouble() / timePrimitive}x")
}
```

---

## Coroutines Performance

```kotlin
import kotlinx.coroutines.*

suspend fun main() {
    // 1. Thread vs Coroutine comparison
    val threadTime = measureTimeMillis {
        val threads = (1..1000).map { i ->
            Thread { Thread.sleep(100) }
        }
        threads.forEach { it.start() }
        threads.forEach { it.join() }
    }
    println("1000 threads: ${threadTime}ms")
    
    val coroutineTime = measureTimeMillis {
        coroutineScope {
            repeat(1000) {
                launch { delay(100) }
            }
        }
    }
    println("1000 coroutines: ${coroutineTime}ms")
    
    println()
    
    // 2. Sequential vs Parallel
    suspend fun fetchData(id: Int): String {
        delay(100)  // simulate I/O
        return "Data#$id"
    }
    
    val seqTime = measureTimeMillis {
        val results = (1..10).map { fetchData(it) }
    }
    println("Sequential (10 calls): ${seqTime}ms")
    
    val parTime = measureTimeMillis {
        val results = coroutineScope {
            (1..10).map { async { fetchData(it) } }.awaitAll()
        }
    }
    println("Parallel (10 calls): ${parTime}ms")
    
    println()
    
    // 3. Dispatcher selection
    val ioTime = measureTimeMillis {
        withContext(Dispatchers.IO) {
            // I/O bound: use IO dispatcher (thread pool)
            repeat(10) {
                // simulated blocking I/O
            }
        }
    }
    
    val defaultTime = measureTimeMillis {
        withContext(Dispatchers.Default) {
            // CPU bound: use Default dispatcher
            repeat(10) {
                (1..10000).sum()  // CPU work
            }
        }
    }
    
    // 4. Avoid unnecessary context switches
    fun badPattern() = runBlocking {
        withContext(Dispatchers.IO) {
            withContext(Dispatchers.Default) {  // unnecessary switch
                withContext(Dispatchers.IO) {   // switch back
                    "done"
                }
            }
        }
    }
    
    fun goodPattern() = runBlocking {
        withContext(Dispatchers.IO) {
            // Stay in IO for the whole thing
            "done"
        }
    }
    
    // 5. Channel capacity tuning
    val buffered = kotlinx.coroutines.channels.Channel<Int>(capacity = 100)
    val unlimited = kotlinx.coroutines.channels.Channel<Int>(capacity = Channel.UNLIMITED)
    val rendezvous = kotlinx.coroutines.channels.Channel<Int>(capacity = 0)
    
    buffered.close()
    unlimited.close()
    rendezvous.close()
}
```

---

## Caching Strategies

```kotlin
import java.util.concurrent.ConcurrentHashMap

// LRU Cache
class LruCache<K, V>(private val maxSize: Int) {
    private val cache = object : LinkedHashMap<K, V>(maxSize, 0.75f, true) {
        override fun removeEldestEntry(eldest: Map.Entry<K, V>): Boolean {
            return size > maxSize
        }
    }
    
    @Synchronized
    operator fun get(key: K): V? = cache[key]
    
    @Synchronized
    operator fun set(key: K, value: V) { cache[key] = value }
    
    @Synchronized
    fun getOrPut(key: K, default: () -> V): V {
        return cache[key] ?: default().also { cache[key] = it }
    }
    
    @Synchronized
    fun remove(key: K): V? = cache.remove(key)
    
    val size: Int @Synchronized get() = cache.size
}

// TTL Cache
class TtlCache<K, V>(private val ttlMs: Long) {
    private data class Entry<V>(val value: V, val expiresAt: Long)
    
    private val cache = ConcurrentHashMap<K, Entry<V>>()
    
    operator fun get(key: K): V? {
        val entry = cache[key] ?: return null
        return if (System.currentTimeMillis() < entry.expiresAt) entry.value
        else { cache.remove(key); null }
    }
    
    operator fun set(key: K, value: V) {
        cache[key] = Entry(value, System.currentTimeMillis() + ttlMs)
    }
    
    fun getOrPut(key: K, default: () -> V): V {
        return get(key) ?: default().also { set(key, it) }
    }
    
    fun cleanup() {
        val now = System.currentTimeMillis()
        cache.entries.removeIf { it.value.expiresAt <= now }
    }
}

// Memoization
fun <T, R> memoize(function: (T) -> R): (T) -> R {
    val cache = ConcurrentHashMap<T, R>()
    return { input -> cache.getOrPut(input) { function(input) } }
}

fun <T1, T2, R> memoize(function: (T1, T2) -> R): (T1, T2) -> R {
    val cache = ConcurrentHashMap<Pair<T1, T2>, R>()
    return { t1, t2 -> cache.getOrPut(t1 to t2) { function(t1, t2) } }
}

// Write-through cache
class WriteThroughCache<K, V>(
    private val loader: (K) -> V?,
    private val writer: (K, V) -> Unit,
    private val maxSize: Int = 1000
) {
    private val cache = LruCache<K, V>(maxSize)
    
    fun get(key: K): V? {
        return cache[key] ?: loader(key)?.also { cache[key] = it }
    }
    
    fun put(key: K, value: V) {
        cache[key] = value
        writer(key, value)  // write to backing store
    }
    
    fun invalidate(key: K) {
        cache.remove(key)
    }
}

fun main() {
    // LRU Cache
    val lru = LruCache<String, Int>(3)
    lru["a"] = 1
    lru["b"] = 2
    lru["c"] = 3
    lru["d"] = 4  // evicts "a"
    
    println("LRU cache:")
    println("  a: ${lru["a"]}")  // null
    println("  b: ${lru["b"]}")  // 2
    println("  d: ${lru["d"]}")  // 4
    
    // TTL Cache
    val ttl = TtlCache<String, String>(ttlMs = 500L)
    ttl["key"] = "value"
    println("\nTTL cache:")
    println("  Before expiry: ${ttl["key"]}")
    Thread.sleep(600)
    println("  After expiry: ${ttl["key"]}")
    
    // Memoization
    val fibMemo = memoize<Int, Long> { n ->
        if (n <= 1) n.toLong()
        else fibMemo(n - 1) + fibMemo(n - 2)
    }
    
    println("\nFibonacci with memoization:")
    val time = measureTimeMillis {
        println("  fib(40) = ${fibMemo(40)}")
    }
    println("  Time: ${time}ms")
    
    // Without memoization (exponential)
    fun fibSlow(n: Int): Long = if (n <= 1) n.toLong() else fibSlow(n-1) + fibSlow(n-2)
    
    val slowTime = measureTimeMillis {
        println("  fib(35) = ${fibSlow(35)}")
    }
    println("  Slow time: ${slowTime}ms")
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Optimize a data processing pipeline

// Original (slow):
fun processDataSlow(data: List<Map<String, Any>>): Map<String, Double> {
    val result = mutableMapOf<String, Double>()
    
    for (item in data) {
        val category = item["category"] as? String ?: continue
        val value = (item["value"] as? Number)?.toDouble() ?: continue
        
        // Expensive: recreate filtered list each time
        val sameCategory = data.filter { it["category"] == category }
        val avg = sameCategory.mapNotNull { (it["value"] as? Number)?.toDouble() }.average()
        
        result[category] = avg
    }
    
    return result
}

// Optimized:
fun processDataFast(data: List<Map<String, Any>>): Map<String, Double> {
    // Group first, then compute - O(n) vs O(n²)
    return data
        .asSequence()
        .mapNotNull { item ->
            val category = item["category"] as? String ?: return@mapNotNull null
            val value = (item["value"] as? Number)?.toDouble() ?: return@mapNotNull null
            category to value
        }
        .groupBy({ it.first }, { it.second })
        .mapValues { (_, values) -> values.average() }
}

// Even faster with fold:
fun processDataOptimal(data: List<Map<String, Any>>): Map<String, Double> {
    data class Accumulator(val sum: Double, val count: Int)
    
    return data
        .fold(mutableMapOf<String, Accumulator>()) { acc, item ->
            val category = item["category"] as? String ?: return@fold acc
            val value = (item["value"] as? Number)?.toDouble() ?: return@fold acc
            
            val current = acc[category] ?: Accumulator(0.0, 0)
            acc[category] = Accumulator(current.sum + value, current.count + 1)
            acc
        }
        .mapValues { (_, v) -> v.sum / v.count }
}

fun main() {
    val data = (1..10_000).map { i ->
        mapOf(
            "category" to listOf("A", "B", "C", "D", "E").random(),
            "value" to (1..100).random().toDouble()
        )
    }
    
    // Small sample for slow version
    val small = data.take(100)
    
    val slowTime = measureTimeMillis { processDataSlow(small) }
    val fastTime = measureTimeMillis { processDataFast(data) }
    val optimalTime = measureTimeMillis { processDataOptimal(data) }
    
    println("Slow (100 items): ${slowTime}ms")
    println("Fast (10k items): ${fastTime}ms")
    println("Optimal (10k items): ${optimalTime}ms")
    
    // Results should be same
    val fastResult = processDataFast(data)
    val optimalResult = processDataOptimal(data)
    
    println("\nResults match: ${fastResult.all { (k, v) -> 
        Math.abs(v - (optimalResult[k] ?: 0.0)) < 0.001 
    }}")
    
    fastResult.forEach { (cat, avg) ->
        println("  $cat: avg=${String.format("%.2f", avg)}")
    }
}
```

---

## สรุป Part 33

```
✅ benchmarking: measureTimeMillis/measureNanoTime + warmup
✅ @JvmInline value class: zero-cost abstraction
✅ Object pools: reuse objects แทนสร้างใหม่
✅ Sequence (lazy): ลด intermediate allocations
✅ IntArray/DoubleArray: primitives เร็วกว่า List<Int>
✅ ArrayList(capacity): preallocate สำหรับ known size
✅ Coroutines vs Threads: coroutines เบากว่า, scale ดีกว่า
✅ async/awaitAll: parallel I/O operations
✅ Dispatcher selection: IO vs Default vs Main
✅ LRU Cache: evict least recently used entries
✅ TTL Cache: expire entries after time
✅ Memoization: cache function results
✅ O(n) algorithm ดีกว่า O(n²) เสมอ
```

---

*Part 33/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
