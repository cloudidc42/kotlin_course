# Part 86: Advanced JVM Performance Tuning

## สารบัญ
1. [JVM Memory Architecture](#jvm-memory-architecture)
2. [Garbage Collection Tuning](#garbage-collection-tuning)
3. [Profiling & Diagnostics](#profiling--diagnostics)
4. [JIT Compilation](#jit-compilation)
5. [Application-Level Optimizations](#application-level-optimizations)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## JVM Memory Architecture

```
JVM Memory Layout:

┌──────────────────────────────────────────────────┐
│                    JVM Memory                     │
├──────────────────┬───────────────────────────────┤
│      Heap        │         Non-Heap               │
│                  │                                │
│  ┌────────────┐  │  ┌──────────────────────────┐  │
│  │   Young    │  │  │      Metaspace            │  │
│  │ Generation │  │  │  (Class metadata,         │  │
│  │            │  │  │   Method bytecodes)       │  │
│  │ ┌────────┐ │  │  └──────────────────────────┘  │
│  │ │ Eden   │ │  │  ┌──────────────────────────┐  │
│  │ │ Space  │ │  │  │    Code Cache             │  │
│  │ └────────┘ │  │  │  (JIT compiled code)      │  │
│  │ ┌────────┐ │  │  └──────────────────────────┘  │
│  │ │ S0, S1 │ │  │  ┌──────────────────────────┐  │
│  │ │(Survivor│ │  │  │   Direct Memory           │  │
│  │ └────────┘ │  │  │  (NIO ByteBuffers,         │  │
│  └────────────┘  │  │   off-heap caches)         │  │
│  ┌────────────┐  │  └──────────────────────────┘  │
│  │    Old     │  │                                │
│  │ Generation │  │                                │
│  │  (Tenured) │  │                                │
│  └────────────┘  │                                │
└──────────────────┴───────────────────────────────┘

GC Types:
G1GC (default JDK 9+):
- Region-based heap management
- Predictable pause times
- Good for 4GB+ heaps
- -XX:MaxGCPauseMillis=200 (target pause)

ZGC (JDK 15+ production):
- Sub-millisecond pauses
- Scalable to TB heaps
- Good for latency-sensitive apps
- -XX:+UseZGC

Shenandoah (OpenJDK):
- Concurrent compaction
- Low pause regardless of heap size
```

---

## JVM Flags ที่สำคัญ

```kotlin
// Application startup configuration
// Docker/K8s environment (container-aware)
val jvmFlags = listOf(
    // Memory
    "-XX:+UseContainerSupport",           // ใช้ container memory limits
    "-XX:MaxRAMPercentage=75.0",          // ใช้ 75% ของ container RAM
    "-XX:InitialRAMPercentage=50.0",      // เริ่มต้นที่ 50%
    
    // GC
    "-XX:+UseG1GC",                       // G1 GC (หรือ ZGC สำหรับ latency)
    "-XX:MaxGCPauseMillis=200",           // Target pause time
    "-XX:G1HeapRegionSize=16m",           // Region size
    "-XX:+G1UseAdaptiveIHOP",             // Adaptive initiating heap occupancy
    
    // String deduplication (reduce string memory)
    "-XX:+UseStringDeduplication",
    
    // JIT Compilation
    "-XX:+TieredCompilation",             // Multi-tier JIT
    "-XX:ReservedCodeCacheSize=256m",     // Code cache size
    
    // Diagnostics
    "-XX:+HeapDumpOnOutOfMemoryError",    // Auto heap dump on OOM
    "-XX:HeapDumpPath=/tmp/heapdump.hprof",
    "-XX:+PrintGCDetails",
    "-XX:+PrintGCDateStamps",
    "-Xlog:gc*:file=/tmp/gc.log:time,uptime:filecount=5,filesize=10m",
    
    // JFR (Java Flight Recorder)
    "-XX:StartFlightRecording=duration=60s,filename=/tmp/app.jfr",
    
    // Crash dumps
    "-XX:ErrorFile=/tmp/hs_err_pid%p.log"
)

// application performance monitoring
@org.springframework.stereotype.Component
class JvmMetricsCollector(
    private val meterRegistry: io.micrometer.core.instrument.MeterRegistry
) {
    
    init {
        // Memory pools
        val memoryBean = java.lang.management.ManagementFactory.getMemoryMXBean()
        
        meterRegistry.gauge("jvm.heap.used", memoryBean) {
            it.heapMemoryUsage.used.toDouble()
        }
        meterRegistry.gauge("jvm.heap.max", memoryBean) {
            it.heapMemoryUsage.max.toDouble()
        }
        meterRegistry.gauge("jvm.nonheap.used", memoryBean) {
            it.nonHeapMemoryUsage.used.toDouble()
        }
        
        // GC stats
        java.lang.management.ManagementFactory.getGarbageCollectorMXBeans().forEach { gc ->
            meterRegistry.gauge("jvm.gc.count", listOf(io.micrometer.core.instrument.Tag.of("gc", gc.name)), gc) {
                it.collectionCount.toDouble()
            }
            meterRegistry.gauge("jvm.gc.time", listOf(io.micrometer.core.instrument.Tag.of("gc", gc.name)), gc) {
                it.collectionTime.toDouble()
            }
        }
        
        // Thread stats
        val threadBean = java.lang.management.ManagementFactory.getThreadMXBean()
        meterRegistry.gauge("jvm.threads.live", threadBean) { it.threadCount.toDouble() }
        meterRegistry.gauge("jvm.threads.daemon", threadBean) { it.daemonThreadCount.toDouble() }
        meterRegistry.gauge("jvm.threads.peak", threadBean) { it.peakThreadCount.toDouble() }
        
        // Class loading
        val classBean = java.lang.management.ManagementFactory.getClassLoadingMXBean()
        meterRegistry.gauge("jvm.classes.loaded", classBean) { it.loadedClassCount.toDouble() }
    }
}
```

---

## Garbage Collection Tuning

```kotlin
// GC Tuning examples by scenario

// Scenario 1: API Server (low latency)
// ต้องการ pause time < 20ms
/*
-XX:+UseZGC
-XX:ZCollectionInterval=5
-Xms2g -Xmx2g  # fix heap size เพื่อลด GC frequency
*/

// Scenario 2: Batch Job (throughput)
// ต้องการ throughput สูง ยอม pause นาน
/*
-XX:+UseParallelGC
-XX:GCTimeRatio=19  # 95% app time, 5% GC time
-XX:+UseAdaptiveSizePolicy
*/

// Scenario 3: Mixed workload
/*
-XX:+UseG1GC
-XX:MaxGCPauseMillis=100
-XX:G1NewSizePercent=30
-XX:G1MaxNewSizePercent=40
*/

// Object allocation optimization
class AllocationOptimizationExamples {
    
    // BAD: สร้าง string ใน loop
    fun badStringConcatenation(items: List<String>): String {
        var result = ""
        for (item in items) {
            result += item  // สร้าง String object ใหม่ทุก iteration
        }
        return result
    }
    
    // GOOD: ใช้ StringBuilder
    fun goodStringConcatenation(items: List<String>): String {
        return StringBuilder().apply {
            items.forEach { append(it) }
        }.toString()
    }
    
    // BAD: boxing/unboxing
    fun badBoxing(numbers: List<Int>): Long {
        var sum = 0L
        for (n in numbers) {
            sum += n  // n เป็น Int (primitive ถ้า list ไม่ใช่ nullable)
        }
        return sum
    }
    
    // GOOD: ใช้ primitive types ตรงๆ
    fun goodPrimitive(numbers: IntArray): Long {
        var sum = 0L
        for (n in numbers) {
            sum += n
        }
        return sum
    }
    
    // GOOD: ใช้ Kotlin stdlib
    fun betterSum(numbers: IntArray): Long = numbers.sumOf { it.toLong() }
    
    // Object pooling
    class ByteBufferPool(private val bufferSize: Int, private val poolSize: Int = 100) {
        private val pool = java.util.concurrent.ArrayBlockingQueue<java.nio.ByteBuffer>(poolSize)
        
        init {
            repeat(poolSize) {
                pool.offer(java.nio.ByteBuffer.allocateDirect(bufferSize))
            }
        }
        
        fun acquire(): java.nio.ByteBuffer {
            return pool.poll() ?: java.nio.ByteBuffer.allocateDirect(bufferSize)
        }
        
        fun release(buffer: java.nio.ByteBuffer) {
            buffer.clear()
            pool.offer(buffer)
        }
        
        inline fun <T> use(block: (java.nio.ByteBuffer) -> T): T {
            val buffer = acquire()
            try {
                return block(buffer)
            } finally {
                release(buffer)
            }
        }
    }
}
```

---

## Profiling & Diagnostics

```kotlin
// Async Profiler integration
class ProfilerUtils {
    
    fun startCpuProfiling(durationSeconds: Int, outputFile: String) {
        val profilerLib = System.getProperty("async.profiler.lib") ?: return
        val vm = com.sun.tools.attach.VirtualMachine.attach(
            java.lang.ProcessHandle.current().pid().toString()
        )
        
        try {
            vm.loadAgentPath(profilerLib, "start,event=cpu,file=$outputFile,duration=$durationSeconds")
        } finally {
            vm.detach()
        }
    }
    
    companion object {
        // Parse async-profiler output
        fun parseHotMethods(jfrFile: String): List<HotMethod> {
            // Use JDK Flight Recorder API to parse
            val repository = jdk.jfr.consumer.RecordingFile.readAllEvents(java.nio.file.Path.of(jfrFile))
            return repository
                .filter { it.eventType.name == "jdk.ExecutionSample" }
                .groupBy { event ->
                    val frame = event.getValue<jdk.jfr.consumer.RecordedStackTrace>("stackTrace")
                        ?.frames?.firstOrNull()
                    "${frame?.method?.type?.name}.${frame?.method?.name}"
                }
                .map { (method, samples) -> HotMethod(method, samples.size) }
                .sortedByDescending { it.sampleCount }
                .take(20)
        }
    }
}

data class HotMethod(val methodName: String, val sampleCount: Int)

// Heap analysis
class HeapAnalyzer {
    
    fun analyzeObjectRetention(): Map<String, Long> {
        val beans = java.lang.management.ManagementFactory.getMemoryPoolMXBeans()
        return beans.associate { pool ->
            pool.name to pool.usage.used
        }
    }
    
    fun triggerGC() {
        System.gc()
        System.runFinalization()
    }
    
    fun dumpHeap(outputFile: String) {
        val server = java.lang.management.ManagementFactory.getPlatformMBeanServer()
        val hotSpotDiag = java.lang.management.ManagementFactory.newPlatformMXBeanProxy(
            server,
            "com.sun.management:type=HotSpotDiagnostic",
            com.sun.management.HotSpotDiagnosticMXBean::class.java
        )
        hotSpotDiag.dumpHeap(outputFile, true)
    }
    
    // Memory leak detection
    @org.springframework.stereotype.Component
    class MemoryLeakDetector(
        private val meterRegistry: io.micrometer.core.instrument.MeterRegistry
    ) {
        private var previousHeapUsed = 0L
        private var growthCount = 0
        
        @org.springframework.scheduling.annotation.Scheduled(fixedDelay = 60_000)
        fun checkMemoryTrend() {
            val currentHeap = java.lang.management.ManagementFactory.getMemoryMXBean()
                .heapMemoryUsage.used
            
            if (currentHeap > previousHeapUsed) {
                growthCount++
                if (growthCount >= 10) {
                    meterRegistry.counter("jvm.memory.potential_leak").increment()
                    // Alert!
                }
            } else {
                growthCount = 0
            }
            
            previousHeapUsed = currentHeap
        }
    }
}

// Thread dump analysis
class ThreadDumpAnalyzer {
    
    fun captureThreadDump(): ThreadDumpReport {
        val threadBean = java.lang.management.ManagementFactory.getThreadMXBean()
        val threadInfos = threadBean.dumpAllThreads(true, true)
        
        val blockedThreads = threadInfos.filter { it.threadState == Thread.State.BLOCKED }
        val waitingThreads = threadInfos.filter { it.threadState == Thread.State.WAITING }
        val deadlockedThreads = threadBean.findDeadlockedThreads()?.toList() ?: emptyList()
        
        return ThreadDumpReport(
            totalThreads = threadInfos.size,
            blockedCount = blockedThreads.size,
            waitingCount = waitingThreads.size,
            deadlockCount = deadlockedThreads.size,
            topBlockedThreads = blockedThreads.take(5).map { it.threadName },
            timestamp = java.time.Instant.now()
        )
    }
}

data class ThreadDumpReport(
    val totalThreads: Int,
    val blockedCount: Int,
    val waitingCount: Int,
    val deadlockCount: Int,
    val topBlockedThreads: List<String>,
    val timestamp: java.time.Instant
)
```

---

## Application-Level Optimizations

```kotlin
// 1. Efficient data structures
class DataStructureOptimizations {
    
    // BAD: ArrayList for frequent contains() checks
    fun badContainsCheck(ids: List<String>, targetId: String): Boolean {
        return targetId in ids  // O(n)
    }
    
    // GOOD: HashSet for O(1) lookup
    fun goodContainsCheck(ids: Set<String>, targetId: String): Boolean {
        return targetId in ids  // O(1)
    }
    
    // GOOD: Sequence for lazy evaluation
    fun efficientLargeList(items: List<String>): List<String> {
        return items.asSequence()
            .filter { it.startsWith("A") }
            .map { it.lowercase() }
            .take(10)
            .toList()  // Only processes until 10 items found
    }
    
    // GOOD: groupBy instead of nested loops
    fun efficientGrouping(orders: List<Order>): Map<String, List<Order>> {
        return orders.groupBy { it.customerId }  // O(n)
        // vs nested loop O(n²)
    }
}

// 2. Caching strategies
@org.springframework.stereotype.Service
class ProductCacheService(
    private val cacheManager: org.springframework.cache.CacheManager,
    private val productRepository: ProductRepository
) {
    private val localCache = com.github.benmanes.caffeine.cache.Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(5, java.util.concurrent.TimeUnit.MINUTES)
        .recordStats()
        .build<String, Product>()
    
    // Two-level caching: L1 (Caffeine, local) → L2 (Redis, distributed)
    fun getProduct(id: String): Product? {
        // L1: local memory cache
        localCache.getIfPresent(id)?.let { return it }
        
        // L2: Redis
        val cache = cacheManager.getCache("products")
        val cached = cache?.get(id, Product::class.java)
        if (cached != null) {
            localCache.put(id, cached)
            return cached
        }
        
        // DB
        val product = productRepository.findById(id).orElse(null) ?: return null
        localCache.put(id, product)
        cache?.put(id, product)
        
        return product
    }
    
    fun invalidate(id: String) {
        localCache.invalidate(id)
        cacheManager.getCache("products")?.evict(id)
    }
    
    fun getCacheStats(): CacheStats {
        val stats = localCache.stats()
        return CacheStats(
            hitRate = stats.hitRate(),
            missRate = stats.missRate(),
            evictionCount = stats.evictionCount(),
            loadCount = stats.loadCount()
        )
    }
}

data class CacheStats(
    val hitRate: Double,
    val missRate: Double,
    val evictionCount: Long,
    val loadCount: Long
)

// 3. Database query optimization
@org.springframework.stereotype.Repository
class OptimizedOrderRepository(
    private val entityManager: jakarta.persistence.EntityManager
) {
    
    // GOOD: projection instead of fetching all fields
    fun getOrderSummaries(userId: String): List<OrderSummary> {
        return entityManager.createQuery("""
            SELECT new com.ecommerce.dto.OrderSummary(
                o.id, o.status, o.totalAmount, o.createdAt
            )
            FROM Order o
            WHERE o.customerId = :userId
            ORDER BY o.createdAt DESC
        """, OrderSummary::class.java)
            .setParameter("userId", userId)
            .setMaxResults(50)
            .resultList
    }
    
    // GOOD: batch loading to avoid N+1
    fun getOrdersWithItems(userId: String): List<Order> {
        return entityManager.createQuery("""
            SELECT DISTINCT o FROM Order o
            JOIN FETCH o.items i
            JOIN FETCH i.product p
            WHERE o.customerId = :userId
        """, Order::class.java)
            .setParameter("userId", userId)
            .resultList
    }
    
    // GOOD: bulk update
    fun bulkUpdateStatus(orderIds: List<String>, status: String): Int {
        return entityManager.createQuery("""
            UPDATE Order o SET o.status = :status
            WHERE o.id IN :ids
        """)
            .setParameter("status", status)
            .setParameter("ids", orderIds)
            .executeUpdate()
    }
    
    // GOOD: streaming results for large datasets
    fun streamAllOrders(block: (Order) -> Unit) {
        entityManager.createQuery("FROM Order o ORDER BY o.id", Order::class.java)
            .apply { setHint("org.hibernate.fetchSize", 1000) }
            .resultStream
            .use { stream -> stream.forEach(block) }
    }
}

// 4. Async processing
@org.springframework.stereotype.Service
class AsyncProductService(
    private val productRepository: ProductRepository,
    private val imageService: ImageService,
    private val searchIndexService: SearchIndexService
) {
    
    private val scope = kotlinx.coroutines.CoroutineScope(
        kotlinx.coroutines.Dispatchers.IO + kotlinx.coroutines.SupervisorJob()
    )
    
    // Fire and forget
    fun indexProductAsync(product: Product) {
        scope.launch {
            try {
                searchIndexService.index(product)
            } catch (e: Exception) {
                // Log, don't propagate
            }
        }
    }
    
    // Parallel I/O
    suspend fun enrichProduct(productId: String): EnrichedProduct = 
        kotlinx.coroutines.coroutineScope {
            val product = async { productRepository.findById(productId).orElseThrow() }
            val images = async { imageService.getProductImages(productId) }
            val reviews = async { productRepository.getProductReviews(productId) }
            val related = async { productRepository.getRelatedProducts(productId, limit = 5) }
            
            EnrichedProduct(
                product = product.await(),
                images = images.await(),
                reviews = reviews.await(),
                relatedProducts = related.await()
            )
        }
}

data class EnrichedProduct(
    val product: Product,
    val images: List<String>,
    val reviews: List<Any>,
    val relatedProducts: List<Product>
)

// 5. Connection pool tuning
@org.springframework.context.annotation.Configuration
class DataSourceConfig {
    
    @org.springframework.context.annotation.Bean
    @org.springframework.boot.autoconfigure.condition.ConditionalOnProperty("spring.datasource.url")
    fun dataSource(): javax.sql.DataSource {
        return com.zaxxer.hikari.HikariDataSource(com.zaxxer.hikari.HikariConfig().apply {
            jdbcUrl = System.getenv("DB_URL")
            username = System.getenv("DB_USER")
            password = System.getenv("DB_PASSWORD")
            driverClassName = "org.postgresql.Driver"
            
            // Pool sizing: formula = ((core_count * 2) + effective_spindle_count)
            // For SSD: effective_spindle_count ≈ 1
            val coreCount = Runtime.getRuntime().availableProcessors()
            maximumPoolSize = (coreCount * 2) + 1
            minimumIdle = maximumPoolSize / 2
            
            // Timeouts
            connectionTimeout = 30_000     // 30s to get connection from pool
            idleTimeout = 600_000          // 10min idle before removing
            maxLifetime = 1_800_000        // 30min max lifetime
            keepaliveTime = 60_000         // 1min keepalive ping
            
            // Performance
            connectionTestQuery = "SELECT 1"
            addDataSourceProperty("cachePrepStmts", "true")
            addDataSourceProperty("prepStmtCacheSize", "250")
            addDataSourceProperty("prepStmtCacheSqlLimit", "2048")
            addDataSourceProperty("useServerPrepStmts", "true")
            addDataSourceProperty("rewriteBatchedStatements", "true")
            
            // Metrics
            metricRegistry = io.micrometer.core.instrument.Metrics.globalRegistry
        })
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement Response Compression Benchmark

// สร้าง benchmark ที่วัด:
// 1. Throughput (requests/second) ที่ concurrency levels ต่างๆ
// 2. p50, p95, p99 latency
// 3. Memory usage during load
// 4. GC pause frequency and duration

import org.openjdk.jmh.annotations.*
import java.util.concurrent.TimeUnit

@State(Scope.Benchmark)
@BenchmarkMode(Mode.Throughput, Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@Fork(1)
@Warmup(iterations = 3, time = 2)
@Measurement(iterations = 5, time = 5)
class ApiPerformanceBenchmark {
    
    private lateinit var productService: ProductService
    
    @Setup
    fun setup() {
        productService = TODO("Initialize service")
    }
    
    @Benchmark
    fun getProductById(): Product? {
        return productService.getById("product-123")
    }
    
    @Benchmark
    @Threads(10)
    fun concurrentProductLookup(): Product? {
        return productService.getById("product-${Thread.currentThread().id % 100}")
    }
    
    @Benchmark
    fun searchProducts(): List<Product> {
        return productService.search("iPhone")
    }
    
    // Use JMH blackhole to prevent dead code elimination
    @Benchmark
    fun computeRecommendations(blackhole: org.openjdk.jmh.infra.Blackhole): Int {
        val recommendations = productService.getRecommendations("user-1")
        blackhole.consume(recommendations)
        return recommendations.size
    }
}

// k6 load test script (JavaScript)
// Run with: k6 run load_test.js
/*
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '30s', target: 10 },   // ramp up
    { duration: '2m', target: 100 },   // steady state
    { duration: '30s', target: 0 },    // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // p95 < 500ms
    http_req_failed: ['rate<0.01'],    // error rate < 1%
  },
};

export default function () {
  const res = http.get('http://localhost:8080/api/v1/products');
  check(res, { 'status is 200': (r) => r.status === 200 });
  sleep(0.1);
}
*/
```

---

## สรุป Part 86

```
✅ JVM Memory: Heap (Young/Old), Non-Heap (Metaspace, Code Cache, Direct)
✅ G1GC: region-based, predictable pauses, MaxGCPauseMillis
✅ ZGC: sub-millisecond pauses, scalable to TB heaps
✅ Container-aware: UseContainerSupport, MaxRAMPercentage
✅ HeapDumpOnOutOfMemoryError: auto dump for OOM investigation
✅ JFR: StartFlightRecording for performance data
✅ GC logging: Xlog:gc* to file with rotation
✅ String deduplication: -XX:+UseStringDeduplication
✅ Object pooling: ByteBufferPool with ArrayBlockingQueue
✅ StringBuilder vs string concatenation in loops
✅ Primitive types vs boxing: IntArray vs List<Int>
✅ Sequence: lazy evaluation for large list pipelines
✅ Two-level caching: Caffeine (L1) + Redis (L2)
✅ Caffeine stats: hitRate, missRate, evictionCount
✅ JPA projection: SELECT new DTO(...) instead of full entity
✅ JOIN FETCH: avoid N+1 query problem
✅ Bulk update: single UPDATE instead of loop
✅ Streaming results: setHint fetchSize for large tables
✅ Async fire-and-forget: CoroutineScope + SupervisorJob
✅ Parallel I/O: coroutineScope + async/await
✅ HikariCP tuning: pool size formula, connection lifecycle
✅ PreparedStatement cache: cachePrepStmts, prepStmtCacheSize
✅ ThreadDump analysis: blocked, waiting, deadlocked threads
✅ Memory leak detection: heap growth trend monitoring
✅ JMH benchmarks: throughput, latency, concurrent tests
```

---

*Part 86/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
