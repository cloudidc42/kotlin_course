# Part 76: Distributed Locking และ Leader Election

## สารบัญ
1. [ทำไมต้องใช้ Distributed Lock](#ทำไมต้องใช้-distributed-lock)
2. [Redis RedLock Algorithm](#redis-redlock-algorithm)
3. [Database-based Distributed Lock](#database-based-distributed-lock)
4. [Leader Election](#leader-election)
5. [Idempotency Keys](#idempotency-keys)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## ทำไมต้องใช้ Distributed Lock

```
ปัญหา:
- หลาย instances รันพร้อมกัน
- ต้องการให้ทำงานชิ้นหนึ่ง "เพียงครั้งเดียว" ใน cluster
- เช่น: cron job, inventory deduction, scheduled task

ตัวอย่าง Race Condition:
1. User A ซื้อ product, stock = 1
2. Instance 1 read: stock = 1 (available)
3. Instance 2 read: stock = 1 (available)
4. Instance 1 write: stock = 0, create order A
5. Instance 2 write: stock = -1, create order B ← BUG!

วิธีแก้:
1. Database lock: SELECT FOR UPDATE
2. Redis lock: SET key NX EX
3. Zookeeper: distributed coordination
4. Optimistic locking: version column + retry
```

---

## Redis RedLock Algorithm

```kotlin
// Simple Redis distributed lock
@Service
class RedisDistributedLockService(
    private val redisTemplate: org.springframework.data.redis.core.StringRedisTemplate
) {
    
    companion object {
        const val LOCK_PREFIX = "lock:"
        val DEFAULT_EXPIRY = java.time.Duration.ofSeconds(30)
    }
    
    // Acquire lock: returns true if acquired, false if already locked
    fun tryAcquire(lockKey: String, ownerId: String, expiry: java.time.Duration = DEFAULT_EXPIRY): Boolean {
        val key = "$LOCK_PREFIX$lockKey"
        
        // SET key value NX EX seconds
        // NX = only set if not exists
        // EX = expiry in seconds
        val acquired = redisTemplate.opsForValue().setIfAbsent(key, ownerId, expiry)
        
        return acquired == true
    }
    
    // Release lock: only release if we own it (prevents releasing someone else's lock)
    fun release(lockKey: String, ownerId: String): Boolean {
        val key = "$LOCK_PREFIX$lockKey"
        
        // Use Lua script for atomic check-and-delete
        val luaScript = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                return redis.call('del', KEYS[1])
            else
                return 0
            end
        """.trimIndent()
        
        val result = redisTemplate.execute(
            org.springframework.data.redis.core.script.DefaultRedisScript<Long>(luaScript, Long::class.java),
            listOf(key),
            ownerId
        )
        
        return result == 1L
    }
    
    // Execute action with distributed lock
    fun <T> withLock(
        lockKey: String,
        expiry: java.time.Duration = DEFAULT_EXPIRY,
        waitTimeMs: Long = 5000,
        retryIntervalMs: Long = 100,
        action: () -> T
    ): T {
        val ownerId = java.util.UUID.randomUUID().toString()
        val deadline = System.currentTimeMillis() + waitTimeMs
        
        while (System.currentTimeMillis() < deadline) {
            if (tryAcquire(lockKey, ownerId, expiry)) {
                try {
                    return action()
                } finally {
                    release(lockKey, ownerId)
                }
            }
            Thread.sleep(retryIntervalMs)
        }
        
        throw LockAcquisitionException("Could not acquire lock '$lockKey' within ${waitTimeMs}ms")
    }
}

class LockAcquisitionException(message: String) : RuntimeException(message)

typealias Service = org.springframework.stereotype.Service

// Usage: prevent duplicate order processing
@Service
class OrderProcessingService(
    private val lockService: RedisDistributedLockService,
    private val orderRepository: OrderRepository
) {
    
    fun processOrder(orderId: String) {
        lockService.withLock(
            lockKey = "order:process:$orderId",
            expiry = java.time.Duration.ofSeconds(60),
            waitTimeMs = 5000
        ) {
            val order = orderRepository.findById(orderId)
                ?: throw OrderNotFoundException("Order $orderId not found")
            
            if (order.status != OrderStatus.PENDING) {
                return@withLock  // Already processed, skip
            }
            
            // Safe to process - we have exclusive lock
            processOrderInternal(order)
        }
    }
    
    private fun processOrderInternal(order: Order) {
        // Business logic here...
    }
}

interface OrderRepository {
    fun findById(id: String): Order?
}

data class Order(val id: String, val status: OrderStatus)
enum class OrderStatus { PENDING, PROCESSING, COMPLETED, CANCELLED }
class OrderNotFoundException(msg: String) : Exception(msg)

// Coroutine-friendly lock
@Service
class CoroutineDistributedLock(
    private val redisTemplate: org.springframework.data.redis.core.ReactiveStringRedisTemplate
) {
    
    suspend fun <T> withLock(
        lockKey: String,
        expiry: java.time.Duration = java.time.Duration.ofSeconds(30),
        block: suspend () -> T
    ): T {
        val key = "lock:$lockKey"
        val ownerId = java.util.UUID.randomUUID().toString()
        
        // Try to acquire with reactive Redis
        val acquired = redisTemplate.opsForValue()
            .setIfAbsent(key, ownerId, expiry)
            .awaitFirst()
        
        if (!acquired) {
            throw LockAcquisitionException("Could not acquire lock: $lockKey")
        }
        
        return try {
            block()
        } finally {
            // Release with Lua script (atomic)
            val luaScript = """
                if redis.call('get', KEYS[1]) == ARGV[1] then
                    return redis.call('del', KEYS[1])
                else
                    return 0
                end
            """.trimIndent()
            
            redisTemplate.execute(
                org.springframework.data.redis.core.script.RedisScript.of(luaScript, Long::class.java),
                listOf(key),
                listOf(ownerId)
            ).awaitFirst()
        }
    }
}

// awaitFirst extension
suspend fun <T> reactor.core.publisher.Mono<T>.awaitFirst(): T =
    kotlinx.coroutines.reactive.awaitFirst()
```

---

## Database-based Distributed Lock

```kotlin
// Pessimistic Lock ด้วย SELECT FOR UPDATE

// Migration: create distributed_locks table
/*
CREATE TABLE distributed_locks (
    lock_key     VARCHAR(255) PRIMARY KEY,
    locked_by    VARCHAR(255) NOT NULL,
    locked_at    TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    expires_at   TIMESTAMP WITH TIME ZONE NOT NULL
);
*/

@Entity
@Table(name = "distributed_locks")
class DistributedLockEntity(
    @Id
    @Column(name = "lock_key")
    val lockKey: String = "",
    
    @Column(name = "locked_by")
    val lockedBy: String = "",
    
    @Column(name = "locked_at")
    val lockedAt: java.time.Instant = java.time.Instant.now(),
    
    @Column(name = "expires_at")
    val expiresAt: java.time.Instant = java.time.Instant.now().plusSeconds(30)
)

typealias Entity = javax.persistence.Entity
typealias Table = javax.persistence.Table
typealias Id = javax.persistence.Id
typealias Column = javax.persistence.Column

interface DistributedLockRepository : org.springframework.data.jpa.repository.JpaRepository<DistributedLockEntity, String> {
    
    @org.springframework.data.jpa.repository.Query(
        value = "SELECT l FROM DistributedLockEntity l WHERE l.lockKey = :lockKey",
        lockMode = javax.persistence.LockModeType.PESSIMISTIC_WRITE
    )
    fun findAndLock(lockKey: String): DistributedLockEntity?
}

@Service
@org.springframework.transaction.annotation.Transactional
class DatabaseDistributedLockService(
    private val lockRepository: DistributedLockRepository
) {
    
    fun acquireLock(lockKey: String, ownerId: String, ttlSeconds: Long = 30): Boolean {
        val now = java.time.Instant.now()
        
        // Try to find existing lock with pessimistic write lock
        val existing = lockRepository.findAndLock(lockKey)
        
        return if (existing == null) {
            // No lock exists, create one
            lockRepository.save(DistributedLockEntity(
                lockKey = lockKey,
                lockedBy = ownerId,
                expiresAt = now.plusSeconds(ttlSeconds)
            ))
            true
        } else if (existing.expiresAt.isBefore(now)) {
            // Lock expired, take it over
            lockRepository.save(existing.copy(
                lockedBy = ownerId,
                lockedAt = now,
                expiresAt = now.plusSeconds(ttlSeconds)
            ))
            true
        } else {
            // Lock held by someone else
            false
        }
    }
    
    fun releaseLock(lockKey: String, ownerId: String): Boolean {
        val lock = lockRepository.findAndLock(lockKey)
        
        return if (lock?.lockedBy == ownerId) {
            lockRepository.deleteById(lockKey)
            true
        } else {
            false
        }
    }
}

fun DistributedLockEntity.copy(
    lockedBy: String = this.lockedBy,
    lockedAt: java.time.Instant = this.lockedAt,
    expiresAt: java.time.Instant = this.expiresAt
) = DistributedLockEntity(this.lockKey, lockedBy, lockedAt, expiresAt)
```

---

## Leader Election

```kotlin
// Leader Election: ใน cluster ต้องการ "หัวหน้า" คนเดียวที่ทำงาน

@Service
class LeaderElectionService(
    private val lockService: RedisDistributedLockService,
    private val instanceId: String = java.net.InetAddress.getLocalHost().hostName
) {
    
    private val logger = org.slf4j.LoggerFactory.getLogger(javaClass)
    
    @Volatile
    private var isLeader = false
    
    companion object {
        const val LEADER_LOCK_KEY = "cluster:leader"
        val LEADER_TTL = java.time.Duration.ofSeconds(15)
    }
    
    // Try to become leader every 5 seconds
    @org.springframework.scheduling.annotation.Scheduled(fixedDelay = 5000)
    fun tryBecomeLeader() {
        val acquired = lockService.tryAcquire(LEADER_LOCK_KEY, instanceId, LEADER_TTL)
        
        if (acquired && !isLeader) {
            isLeader = true
            logger.info("[$instanceId] Became leader!")
            onBecameLeader()
        } else if (!acquired && isLeader) {
            isLeader = false
            logger.info("[$instanceId] Lost leadership")
            onLostLeadership()
        }
    }
    
    // Renew leadership every 10 seconds (before TTL expires)
    @org.springframework.scheduling.annotation.Scheduled(fixedDelay = 10000)
    fun renewLeadership() {
        if (isLeader) {
            val renewed = lockService.tryAcquire(LEADER_LOCK_KEY, instanceId, LEADER_TTL)
            if (!renewed) {
                isLeader = false
                logger.warn("[$instanceId] Failed to renew leadership")
                onLostLeadership()
            }
        }
    }
    
    fun isCurrentLeader() = isLeader
    
    private fun onBecameLeader() {
        // Start background tasks that should only run on leader
        logger.info("Starting leader-only tasks...")
    }
    
    private fun onLostLeadership() {
        // Stop background tasks
        logger.info("Stopping leader-only tasks...")
    }
}

// Use leader check in scheduled tasks
@Component
class ScheduledJobService(
    private val leaderElection: LeaderElectionService,
    private val reportService: ReportService
) {
    
    @org.springframework.scheduling.annotation.Scheduled(cron = "0 0 * * * *")  // Every hour
    fun generateHourlyReport() {
        if (!leaderElection.isCurrentLeader()) {
            return  // Only leader should run this
        }
        
        reportService.generateHourlyReport()
    }
    
    @org.springframework.scheduling.annotation.Scheduled(cron = "0 0 0 * * *")  // Daily midnight
    fun cleanupExpiredSessions() {
        if (!leaderElection.isCurrentLeader()) return
        
        reportService.cleanupExpiredData()
    }
}

typealias Component = org.springframework.stereotype.Component

interface ReportService {
    fun generateHourlyReport()
    fun cleanupExpiredData()
}
```

---

## Idempotency Keys

```kotlin
// Idempotency: ส่ง request เดิมซ้ำ แต่ผลลัพธ์เหมือนกัน

// ป้องกัน duplicate payment, duplicate order

@Service
class IdempotencyService(
    private val redisTemplate: org.springframework.data.redis.core.StringRedisTemplate,
    private val objectMapper: com.fasterxml.jackson.databind.ObjectMapper
) {
    
    companion object {
        const val KEY_PREFIX = "idempotency:"
        val TTL = java.time.Duration.ofHours(24)
    }
    
    // Check if request was already processed
    fun <T> executeOnce(
        idempotencyKey: String,
        requestHash: String,
        action: () -> T,
        resultClass: Class<T>
    ): T {
        val cacheKey = "$KEY_PREFIX$idempotencyKey"
        
        // Check if already processed
        val cached = redisTemplate.opsForValue().get(cacheKey)
        if (cached != null) {
            val cached_data = objectMapper.readValue(cached, IdempotencyRecord::class.java)
            
            // Verify request hash matches (prevent different requests with same key)
            if (cached_data.requestHash != requestHash) {
                throw IdempotencyConflictException(
                    "Idempotency key '$idempotencyKey' was used with different request"
                )
            }
            
            @Suppress("UNCHECKED_CAST")
            return objectMapper.convertValue(cached_data.response, resultClass) as T
        }
        
        // Execute action and cache result
        val result = action()
        
        val record = IdempotencyRecord(
            idempotencyKey = idempotencyKey,
            requestHash = requestHash,
            response = result,
            processedAt = java.time.Instant.now().toString()
        )
        
        redisTemplate.opsForValue().set(
            cacheKey,
            objectMapper.writeValueAsString(record),
            TTL
        )
        
        return result
    }
}

data class IdempotencyRecord(
    val idempotencyKey: String,
    val requestHash: String,
    val response: Any?,
    val processedAt: String
)

class IdempotencyConflictException(msg: String) : RuntimeException(msg)

// Payment controller with idempotency
@RestController
@RequestMapping("/api/v1/payments")
class PaymentController(
    private val paymentService: PaymentService,
    private val idempotencyService: IdempotencyService
) {
    
    @PostMapping("/process")
    fun processPayment(
        @RequestHeader("Idempotency-Key") idempotencyKey: String,
        @RequestBody request: PaymentRequest
    ): ResponseEntity<PaymentResponse> {
        val requestHash = generateHash(request)
        
        val response = idempotencyService.executeOnce(
            idempotencyKey = idempotencyKey,
            requestHash = requestHash,
            action = { paymentService.processPayment(request) },
            resultClass = PaymentResponse::class.java
        )
        
        return ResponseEntity.ok(response)
    }
    
    private fun generateHash(request: PaymentRequest): String {
        val json = com.fasterxml.jackson.databind.ObjectMapper()
            .writeValueAsString(request)
        return java.security.MessageDigest.getInstance("SHA-256")
            .digest(json.toByteArray())
            .let { java.util.Base64.getEncoder().encodeToString(it) }
    }
}

data class PaymentRequest(
    val orderId: String,
    val amount: java.math.BigDecimal,
    val currency: String,
    val cardToken: String
)

data class PaymentResponse(
    val paymentId: String,
    val status: String,
    val amount: java.math.BigDecimal
)

interface PaymentService {
    fun processPayment(request: PaymentRequest): PaymentResponse
}

typealias RestController = org.springframework.web.bind.annotation.RestController
typealias RequestMapping = org.springframework.web.bind.annotation.RequestMapping
typealias PostMapping = org.springframework.web.bind.annotation.PostMapping
typealias RequestHeader = org.springframework.web.bind.annotation.RequestHeader
typealias RequestBody = org.springframework.web.bind.annotation.RequestBody
typealias ResponseEntity = org.springframework.http.ResponseEntity<*>
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement inventory deduction with distributed lock
// ป้องกัน overselling ใน flash sale scenario

@Service
class FlashSaleService(
    private val lockService: RedisDistributedLockService,
    private val inventoryRepository: InventoryRepository,
    private val orderService: OrderCreationService
) {
    
    fun purchaseFlashSaleItem(userId: String, productId: String, quantity: Int): PurchaseResult {
        // TODO: Implement with distributed lock
        // 1. Acquire lock for product: "inventory:product:$productId"
        // 2. Read current stock
        // 3. Validate sufficient stock
        // 4. Create order
        // 5. Decrement stock
        // 6. Release lock
        // 7. Handle LockAcquisitionException (return sold-out error)
        
        return lockService.withLock(
            lockKey = "inventory:product:$productId",
            expiry = java.time.Duration.ofSeconds(10),
            waitTimeMs = 3000
        ) {
            val stock = inventoryRepository.getCurrentStock(productId)
            
            if (stock < quantity) {
                return@withLock PurchaseResult.SoldOut(productId)
            }
            
            val order = orderService.createOrder(userId, productId, quantity)
            inventoryRepository.decrementStock(productId, quantity)
            
            PurchaseResult.Success(order.orderId, order.amount)
        }
    }
}

sealed class PurchaseResult {
    data class Success(val orderId: String, val amount: java.math.BigDecimal) : PurchaseResult()
    data class SoldOut(val productId: String) : PurchaseResult()
    data class Failed(val reason: String) : PurchaseResult()
}

interface InventoryRepository {
    fun getCurrentStock(productId: String): Int
    fun decrementStock(productId: String, quantity: Int)
}

interface OrderCreationService {
    fun createOrder(userId: String, productId: String, quantity: Int): OrderCreatedResult
}

data class OrderCreatedResult(val orderId: String, val amount: java.math.BigDecimal)
```

---

## สรุป Part 76

```
✅ Distributed Lock: ป้องกัน race condition ใน cluster
✅ Redis SET NX EX: atomic lock acquisition
✅ Lua script: atomic check-and-delete (prevent race)
✅ withLock(): retry with backoff until deadline
✅ LockAcquisitionException: clean error on timeout
✅ ownerId UUID: prevent releasing others' locks
✅ Database lock: SELECT FOR UPDATE pessimistic locking
✅ Lock expiry: auto-release if owner crashes
✅ Leader Election: one instance runs cron jobs
✅ Scheduled renewal: refresh lock before TTL expires
✅ isCurrentLeader(): gate for leader-only tasks
✅ onBecameLeader/onLostLeadership: lifecycle hooks
✅ Idempotency Keys: prevent duplicate payments
✅ Request hash: validate same request with same key
✅ 24-hour TTL: cache idempotency result
✅ IdempotencyConflictException: different request, same key
✅ FlashSaleService: distributed lock + inventory check
✅ PurchaseResult sealed: Success/SoldOut/Failed
```

---

*Part 76/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
