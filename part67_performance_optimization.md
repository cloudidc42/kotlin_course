# Part 67: Performance Optimization สำหรับ Kotlin Applications

## สารบัญ
1. [JVM Performance Tuning](#jvm-performance-tuning)
2. [Database Query Optimization](#database-query-optimization)
3. [Caching Strategies](#caching-strategies)
4. [Coroutine Optimization](#coroutine-optimization)
5. [Memory Management](#memory-management)
6. [Load Testing ด้วย Gatling](#load-testing-ด้วย-gatling)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## JVM Performance Tuning

```kotlin
// JVM Flags ที่แนะนำสำหรับ Spring Boot

// Dockerfile หรือ start script:
// JAVA_OPTS="-Xmx512m -Xms256m
//            -XX:+UseContainerSupport
//            -XX:MaxRAMPercentage=75.0
//            -XX:+UseG1GC
//            -XX:MaxGCPauseMillis=200
//            -XX:+UseStringDeduplication
//            -Xlog:gc*:file=/logs/gc.log:time,tags:filecount=5,filesize=20m
//            -Djava.security.egd=file:/dev/./urandom"

// GC Logging analysis
// Look for: Full GC (pause > 500ms), allocation failure, heap pressure

// Kotlin-specific optimizations
class KotlinOptimizations {
    
    // 1. Use sequences for large collections (lazy evaluation)
    fun processLargeList(items: List<Int>): List<String> {
        // Bad: creates intermediate collections
        return items
            .filter { it > 100 }
            .map { it * 2 }
            .take(10)
            .map { it.toString() }
        
        // Good: lazy, processes only what's needed
        // return items.asSequence()
        //     .filter { it > 100 }
        //     .map { it * 2 }
        //     .take(10)
        //     .map { it.toString() }
        //     .toList()
    }
    
    // 2. Avoid unnecessary object creation
    private val reusableBuffer = StringBuilder(1024)
    
    fun buildMessageEfficient(parts: List<String>): String {
        reusableBuffer.clear()
        parts.forEachIndexed { index, part ->
            reusableBuffer.append(part)
            if (index < parts.size - 1) reusableBuffer.append(", ")
        }
        return reusableBuffer.toString()
    }
    
    // 3. Inline functions avoid lambda overhead
    inline fun <T> measureTime(block: () -> T): Pair<T, Long> {
        val start = System.nanoTime()
        val result = block()
        val elapsed = System.nanoTime() - start
        return result to elapsed
    }
    
    // 4. Value classes avoid boxing
    @JvmInline
    value class UserId(val value: String)  // No heap allocation!
    
    // 5. Avoid String concatenation in hot paths
    fun logEfficient(userId: String, action: String) {
        // Bad: creates String even if logger is disabled
        // log.debug("User $userId performed $action")
        
        // Good: lambda evaluated lazily
        // log.debug { "User $userId performed $action" }  // with Kotlin logging lib
    }
    
    // 6. Use arrays instead of lists for primitives
    fun sumEfficient(count: Int): Long {
        val values = IntArray(count) { it + 1 }  // primitive int[], not List<Int>
        return values.sumOf { it.toLong() }
    }
}

// Profiling with JVM Flight Recorder
// jcmd <pid> JFR.start duration=60s filename=profile.jfr
// jcmd <pid> JFR.dump filename=profile.jfr
// Analysis: IntelliJ IDEA → Analyze → Open .jfr file
```

---

## Database Query Optimization

```kotlin
// N+1 Problem and Solutions

// BAD: N+1 queries (1 for orders + N for each order's items)
@Repository
class BadOrderRepository {
    
    fun findAllOrdersWithItems(): List<OrderWithItems> {
        val orders = jdbcTemplate.query("SELECT * FROM orders") { rs, _ ->
            Order(rs.getString("id"), rs.getString("customer_id"))
        }
        
        // N additional queries!
        return orders.map { order ->
            val items = jdbcTemplate.query(
                "SELECT * FROM order_items WHERE order_id = ?",
                arrayOf(order.id)
            ) { rs, _ ->
                OrderItem(rs.getString("id"), rs.getBigDecimal("price"))
            }
            OrderWithItems(order, items)
        }
    }
}

// GOOD: Single query with JOIN
@Repository
class GoodOrderRepository(private val jdbcTemplate: Any) {
    
    fun findAllOrdersWithItems(): List<OrderWithItems> {
        data class Row(
            val orderId: String, val customerId: String,
            val itemId: String?, val itemPrice: java.math.BigDecimal?
        )
        
        // Single query — PostgreSQL optimizes this well
        val rows = mutableListOf<Row>()  // would use jdbcTemplate in real code
        
        // GROUP BY order_id to collect items
        return rows
            .groupBy { it.orderId }
            .map { (orderId, orderRows) ->
                val first = orderRows.first()
                val items = orderRows
                    .filter { it.itemId != null }
                    .map { OrderItem(it.itemId!!, it.itemPrice!!) }
                OrderWithItems(
                    Order(orderId, first.customerId),
                    items
                )
            }
    }
    
    // JPA: use EntityGraph to prevent N+1
    // @EntityGraph(attributePaths = ["items", "customer"])
    // @Query("SELECT o FROM Order o")
    // fun findAllWithItemsAndCustomer(): List<OrderJpa>
}

data class Order(val id: String, val customerId: String)
data class OrderItem(val id: String, val price: java.math.BigDecimal)
data class OrderWithItems(val order: Order, val items: List<OrderItem>)

// Query optimization techniques
@Repository
class OptimizedQueryRepository {
    
    // 1. Projections: SELECT only needed columns
    data class OrderSummary(val id: String, val customerId: String, val totalAmount: java.math.BigDecimal)
    
    // SELECT id, customer_id, total_amount FROM orders  (not SELECT *)
    
    // 2. Pagination: always page large results
    fun findWithPagination(page: Int, size: Int): List<Order> {
        // SELECT * FROM orders ORDER BY created_at DESC LIMIT ? OFFSET ?
        return emptyList()  // placeholder
    }
    
    // 3. Covering Index: index covers all columns in query
    // CREATE INDEX idx_orders_customer_status ON orders(customer_id, status, created_at)
    // Query: WHERE customer_id = ? AND status = ? ORDER BY created_at
    
    // 4. Partial Index: index only relevant rows
    // CREATE INDEX idx_active_orders ON orders(created_at)
    // WHERE status IN ('PENDING', 'CONFIRMED')
    
    // 5. EXPLAIN ANALYZE to understand query plans
    // EXPLAIN (ANALYZE, BUFFERS) SELECT ...
    
    // 6. Connection Pool tuning (HikariCP)
    // maximumPoolSize = (cores * 2) + disk spindles (usually 10-20)
    // minimumIdle = maximumPoolSize (avoid pool growth overhead)
    // connectionTimeout = 30000 (30s)
    // idleTimeout = 600000 (10min)
    // maxLifetime = 1800000 (30min)
}
```

---

## Caching Strategies

```kotlin
@Service
class ProductCacheService(
    private val productRepository: CacheProductRepository,
    private val redisTemplate: RedisTemplate<String, Any>,
    private val cacheManager: CacheManager
) {
    
    // Strategy 1: Cache-Aside (Lazy Loading) — most common
    fun getProduct(productId: String): CachedProduct? {
        val cacheKey = "product:$productId"
        
        // 1. Check cache
        val cached = redisTemplate.opsForValue().get(cacheKey) as? CachedProduct
        if (cached != null) return cached
        
        // 2. Load from database
        val product = productRepository.findById(productId) ?: return null
        
        // 3. Store in cache with TTL
        redisTemplate.opsForValue().set(cacheKey, product, java.time.Duration.ofMinutes(30))
        
        return product
    }
    
    // Strategy 2: Write-Through — update cache on every write
    fun updateProduct(product: CachedProduct): CachedProduct {
        val updated = productRepository.save(product)
        
        // Immediately update cache
        redisTemplate.opsForValue().set(
            "product:${product.id}",
            updated,
            java.time.Duration.ofMinutes(30)
        )
        
        return updated
    }
    
    // Strategy 3: Write-Behind (Write-Back) — async cache flush
    // Cache first, async batch write to DB (risk: data loss on crash)
    
    // Strategy 4: Spring Cache abstraction
    @Cacheable(value = ["products"], key = "#productId", unless = "#result == null")
    fun getProductCached(productId: String): CachedProduct? {
        return productRepository.findById(productId)
    }
    
    @CacheEvict(value = ["products"], key = "#product.id")
    fun deleteProductCache(product: CachedProduct) {
        // Cache automatically evicted
    }
    
    @CachePut(value = ["products"], key = "#product.id")
    fun updateProductCache(product: CachedProduct): CachedProduct {
        return productRepository.save(product)
    }
    
    // Cache warming on startup
    @PostConstruct
    fun warmupCache() {
        println("Warming up product cache...")
        productRepository.findMostViewed(limit = 100).forEach { product ->
            redisTemplate.opsForValue().set(
                "product:${product.id}",
                product,
                java.time.Duration.ofHours(1)
            )
        }
        println("Cache warmup complete")
    }
    
    // Multi-level cache: Local (Caffeine) + Distributed (Redis)
    @Cacheable(value = ["products-l1"])  // L1 = Caffeine (in-memory)
    fun getProductL1(productId: String): CachedProduct? {
        // Fallback to L2 (Redis) then DB
        return getProduct(productId)
    }
    
    // Cache invalidation with versioning
    private var cacheVersion = 1
    
    fun invalidateAllProducts() {
        cacheVersion++
        // All cache keys include version: "product:v2:abc123"
    }
    
    fun getProductVersioned(productId: String): CachedProduct? {
        val key = "product:v$cacheVersion:$productId"
        return redisTemplate.opsForValue().get(key) as? CachedProduct
            ?: productRepository.findById(productId)?.also { product ->
                redisTemplate.opsForValue().set(key, product, java.time.Duration.ofMinutes(30))
            }
    }
}

data class CachedProduct(val id: String, val name: String, val price: Double)

interface CacheProductRepository {
    fun findById(id: String): CachedProduct?
    fun save(product: CachedProduct): CachedProduct
    fun findMostViewed(limit: Int): List<CachedProduct>
}

typealias RedisTemplate<K, V> = org.springframework.data.redis.core.RedisTemplate<K, V>
typealias CacheManager = org.springframework.cache.CacheManager
typealias Cacheable = org.springframework.cache.annotation.Cacheable
typealias CacheEvict = org.springframework.cache.annotation.CacheEvict
typealias CachePut = org.springframework.cache.annotation.CachePut
typealias PostConstruct = jakarta.annotation.PostConstruct
```

---

## Coroutine Optimization

```kotlin
// Coroutine performance best practices

class CoroutineOptimization {
    
    // 1. Choose correct Dispatcher
    suspend fun fileOperation() = withContext(Dispatchers.IO) {
        // I/O bound: use IO dispatcher (thread pool)
        java.io.File("data.txt").readText()
    }
    
    suspend fun cpuIntensive() = withContext(Dispatchers.Default) {
        // CPU bound: use Default dispatcher (N CPU threads)
        (1..1_000_000).fold(0L) { acc, i -> acc + i }
    }
    
    // 2. Concurrent vs Sequential
    suspend fun parallelApiCalls(ids: List<String>): List<String> = coroutineScope {
        // Sequential (slow): 100ms * N requests
        // ids.map { fetchData(it) }
        
        // Parallel (fast): 100ms total regardless of N
        ids.map { id -> async { fetchData(id) } }.awaitAll()
    }
    
    suspend fun fetchData(id: String): String = "data-$id"
    
    // 3. Limit concurrency to avoid overwhelming downstream
    suspend fun batchProcess(ids: List<String>): List<String> {
        val semaphore = Semaphore(10)  // Max 10 concurrent requests
        return coroutineScope {
            ids.map { id ->
                async {
                    semaphore.withPermit {
                        fetchData(id)
                    }
                }
            }.awaitAll()
        }
    }
    
    // 4. chunked processing for very large lists
    suspend fun processLargeList(ids: List<String>): List<String> {
        return ids.chunked(50).flatMap { chunk ->
            coroutineScope {
                chunk.map { id -> async { fetchData(id) } }.awaitAll()
            }
        }
    }
    
    // 5. Avoid GlobalScope — use structured concurrency
    class BadService {
        // BAD: leaks coroutines, not cancellable
        // fun start() { GlobalScope.launch { infiniteLoop() } }
    }
    
    // GOOD: use CoroutineScope tied to lifecycle
    class GoodService(scope: CoroutineScope = CoroutineScope(SupervisorJob() + Dispatchers.Default)) {
        private val serviceScope = scope
        
        fun start() {
            serviceScope.launch { infiniteLoop() }
        }
        
        fun stop() {
            serviceScope.cancel()
        }
        
        private suspend fun infiniteLoop() {
            while (true) {
                delay(1000)
                // do work
            }
        }
    }
    
    // 6. Flow instead of suspend List for large streams
    fun streamLargeData(): Flow<String> = flow {
        // Emits items lazily — no need to load everything in memory
        for (i in 1..1_000_000) {
            emit("item-$i")
            if (i % 1000 == 0) yield()  // cooperative
        }
    }
    
    // 7. StateFlow vs SharedFlow
    // StateFlow: current state, always has value, replays last
    // SharedFlow: event stream, configurable replay
    
    private val _state = MutableStateFlow<String>("initial")
    val state: StateFlow<String> = _state.asStateFlow()
    
    private val _events = MutableSharedFlow<String>(extraBufferCapacity = 64)
    val events: SharedFlow<String> = _events.asSharedFlow()
}

typealias Semaphore = kotlinx.coroutines.sync.Semaphore
typealias Flow<T> = kotlinx.coroutines.flow.Flow<T>
typealias StateFlow<T> = kotlinx.coroutines.flow.StateFlow<T>
typealias MutableStateFlow<T> = kotlinx.coroutines.flow.MutableStateFlow<T>
typealias SharedFlow<T> = kotlinx.coroutines.flow.SharedFlow<T>
typealias MutableSharedFlow<T> = kotlinx.coroutines.flow.MutableSharedFlow<T>
typealias SupervisorJob = kotlinx.coroutines.SupervisorJob
```

---

## Load Testing ด้วย Gatling

```kotlin
// build.gradle.kts
plugins {
    id("io.gatling.gradle") version "3.13.1"
}

dependencies {
    gatling("io.gatling.highcharts:gatling-charts-highcharts:3.13.1")
}

// Load test simulation
class ProductApiLoadTest extends Simulation {
    
    val httpProtocol = http
        .baseUrl("https://api.example.com")
        .acceptHeader("application/json")
        .contentTypeHeader("application/json")
        .header("Authorization", "Bearer test-token")
    
    // Browse products scenario
    val browseProducts = scenario("Browse Products")
        .exec(
            http("Get product list")
                .get("/api/products")
                .queryParam("page", "0")
                .queryParam("size", "20")
                .check(status.is(200))
                .check(jsonPath("$.content").exists)
                .check(responseTimeInMillis.lt(500))
        )
        .pause(1, 3)  // Think time: 1-3 seconds
        .exec(
            http("Get product detail")
                .get("/api/products/#{productId}")
                .check(status.is(200))
                .check(jsonPath("$.id").exists)
        )
    
    // Place order scenario
    val placeOrder = scenario("Place Order")
        .exec(
            http("Create order")
                .post("/api/orders")
                .body(StringBody("""
                    {
                        "customerId": "cust-1",
                        "items": [
                            {"productId": "prod-1", "quantity": 2}
                        ]
                    }
                """))
                .check(status.is(201))
                .check(jsonPath("$.orderId").saveAs("orderId"))
        )
        .pause(2)
        .exec(
            http("Get order status")
                .get("/api/orders/#{orderId}")
                .check(status.is(200))
        )
    
    // Load profile
    setUp(
        // Ramp up to 100 users over 1 minute
        browseProducts.inject(rampUsers(100).during(60)),
        
        // Constant load of 50 users
        placeOrder.inject(constantUsersPerSec(50.0).during(120))
    ).protocols(httpProtocol)
        .assertions(
            global.responseTime.percentile(95).lt(500),  // p95 < 500ms
            global.successfulRequests.percent.gte(99.0), // 99%+ success rate
            global.requestsPerSec.gte(100.0)             // min 100 RPS
        )
}

// Run: ./gradlew gatlingRun
// Report: build/reports/gatling/
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Benchmark different data structures

import kotlin.system.measureNanoTime

fun benchmarkDataStructures() {
    val n = 100_000
    
    // List vs Array vs Set — benchmark insertion and lookup
    val data = (1..n).toList()
    
    // ArrayList lookup
    val arrayListTime = measureNanoTime {
        val list = ArrayList<Int>()
        data.forEach { list.add(it) }
        list.contains(n / 2)
    }
    
    // HashSet lookup
    val hashSetTime = measureNanoTime {
        val set = HashSet<Int>()
        data.forEach { set.add(it) }
        set.contains(n / 2)
    }
    
    // LinkedHashMap (ordered) lookup
    val mapTime = measureNanoTime {
        val map = LinkedHashMap<Int, Boolean>()
        data.forEach { map[it] = true }
        map.containsKey(n / 2)
    }
    
    println("ArrayList: ${arrayListTime / 1_000_000}ms")
    println("HashSet:   ${hashSetTime / 1_000_000}ms")
    println("HashMap:   ${mapTime / 1_000_000}ms")
    
    // TODO: Also benchmark:
    // 1. String concatenation vs StringBuilder
    // 2. for loop vs forEach vs Sequence
    // 3. Regex compilation (compile once vs every call)
    // 4. JSON parsing (Jackson vs kotlinx.serialization)
}

// Regex optimization
class RegexOptimization {
    
    // BAD: compiles regex every call (allocates new Regex object)
    fun isEmailBad(email: String): Boolean {
        return email.matches(Regex("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"))
    }
    
    // GOOD: compile once, reuse
    private val EMAIL_REGEX = Regex("[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}")
    
    fun isEmailGood(email: String): Boolean {
        return email.matches(EMAIL_REGEX)
    }
}
```

---

## สรุป Part 67

```
✅ JVM Flags: -XX:+UseContainerSupport, G1GC, MaxRAMPercentage
✅ JVM Flight Recorder: profiling without performance impact
✅ Sequences: lazy evaluation avoids intermediate collections
✅ Inline functions: avoid lambda object allocation
✅ Value classes: no heap allocation for wrapper types
✅ N+1 Problem: solve with JOIN queries and EntityGraph
✅ Covering Index: index contains all query columns
✅ EXPLAIN ANALYZE: understand PostgreSQL query plans
✅ HikariCP tuning: pool size = cores * 2 + disk spindles
✅ Cache-Aside: lazy loading, check cache then DB
✅ Write-Through: update cache on every write
✅ Spring Cache: @Cacheable, @CacheEvict, @CachePut
✅ Cache warming: @PostConstruct loads hot data on startup
✅ Multi-level cache: L1 Caffeine + L2 Redis
✅ Coroutine Dispatcher: IO for I/O, Default for CPU
✅ Semaphore: limit concurrency with withPermit
✅ Structured concurrency: avoid GlobalScope
✅ Flow: lazy streaming, no memory overhead
✅ Gatling: load testing with scenarios and assertions
✅ Benchmark: measureNanoTime for micro-benchmarks
```

---

*Part 67/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
