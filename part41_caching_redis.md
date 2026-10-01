# Part 41: Caching ด้วย Redis

## สารบัญ
1. [Redis Basics ใน Kotlin](#redis-basics-ใน-kotlin)
2. [Spring Cache Abstraction](#spring-cache-abstraction)
3. [Cache Patterns](#cache-patterns)
4. [Session Management](#session-management)
5. [Rate Limiting ด้วย Redis](#rate-limiting-ด้วย-redis)
6. [Pub/Sub](#pubsub)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Redis Basics ใน Kotlin

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.boot:spring-boot-starter-cache")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
}
```

```kotlin
// Redis configuration
@Configuration
@EnableCaching
class RedisConfig {
    
    @Bean
    fun redisConnectionFactory(
        @Value("\${spring.redis.host}") host: String,
        @Value("\${spring.redis.port}") port: Int
    ): RedisConnectionFactory {
        val config = RedisStandaloneConfiguration(host, port)
        return LettuceConnectionFactory(config)
    }
    
    @Bean
    fun redisTemplate(connectionFactory: RedisConnectionFactory): RedisTemplate<String, Any> {
        return RedisTemplate<String, Any>().apply {
            this.connectionFactory = connectionFactory
            
            val objectMapper = ObjectMapper().apply {
                registerModule(KotlinModule.Builder().build())
                configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
                activateDefaultTyping(
                    LaissezFaireSubTypeValidator.instance,
                    ObjectMapper.DefaultTyping.NON_FINAL,
                    JsonTypeInfo.As.PROPERTY
                )
            }
            
            val serializer = GenericJackson2JsonRedisSerializer(objectMapper)
            keySerializer = StringRedisSerializer()
            valueSerializer = serializer
            hashKeySerializer = StringRedisSerializer()
            hashValueSerializer = serializer
            afterPropertiesSet()
        }
    }
    
    @Bean
    fun cacheManager(connectionFactory: RedisConnectionFactory): CacheManager {
        val defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(
                GenericJackson2JsonRedisSerializer()
            ))
            .disableCachingNullValues()
        
        val cacheConfigurations = mapOf(
            "products" to defaultConfig.entryTtl(Duration.ofHours(1)),
            "users" to defaultConfig.entryTtl(Duration.ofMinutes(15)),
            "sessions" to defaultConfig.entryTtl(Duration.ofHours(24))
        )
        
        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigurations)
            .build()
    }
    
    @Bean
    fun stringRedisTemplate(connectionFactory: RedisConnectionFactory): StringRedisTemplate {
        return StringRedisTemplate(connectionFactory)
    }
}
```

---

## Spring Cache Abstraction

```kotlin
@Service
class ProductService(
    private val productRepository: ProductRepository,
    private val redisTemplate: RedisTemplate<String, Any>
) {
    
    // Cache result, key = product id
    @Cacheable(value = ["products"], key = "#id")
    fun findById(id: String): Product? {
        println("DB hit for product: $id")  // should print only on cache miss
        return productRepository.findById(id).orElse(null)
    }
    
    // Cache list, key = category
    @Cacheable(value = ["products"], key = "#category + ':list'")
    fun findByCategory(category: String): List<Product> {
        return productRepository.findByCategory(category)
    }
    
    // Update cache when product is updated
    @CachePut(value = ["products"], key = "#result.id")
    fun update(id: String, request: UpdateProductRequest): Product {
        val product = productRepository.findById(id)
            .orElseThrow { NoSuchElementException("Product $id not found") }
        
        val updated = product.copy(
            name = request.name ?: product.name,
            price = request.price ?: product.price,
            stock = request.stock ?: product.stock
        )
        
        return productRepository.save(updated)
    }
    
    // Evict specific cache entry
    @CacheEvict(value = ["products"], key = "#id")
    fun delete(id: String) {
        productRepository.deleteById(id)
    }
    
    // Evict all entries in cache
    @CacheEvict(value = ["products"], allEntries = true)
    fun invalidateAll() {
        println("All product cache cleared")
    }
    
    // Conditional caching
    @Cacheable(value = ["products"], key = "#id", condition = "#id != null", unless = "#result == null")
    fun findByIdSafe(id: String?): Product? {
        if (id == null) return null
        return productRepository.findById(id).orElse(null)
    }
}
```

---

## Manual Cache Operations

```kotlin
@Component
class CacheRepository(
    private val redisTemplate: RedisTemplate<String, Any>,
    private val stringRedisTemplate: StringRedisTemplate
) {
    
    fun <T : Any> get(key: String, type: Class<T>): T? {
        @Suppress("UNCHECKED_CAST")
        return redisTemplate.opsForValue().get(key) as T?
    }
    
    fun set(key: String, value: Any, ttl: Duration) {
        redisTemplate.opsForValue().set(key, value, ttl)
    }
    
    fun delete(key: String): Boolean {
        return redisTemplate.delete(key)
    }
    
    fun exists(key: String): Boolean {
        return redisTemplate.hasKey(key) == true
    }
    
    fun ttl(key: String): Duration? {
        val seconds = redisTemplate.getExpire(key, java.util.concurrent.TimeUnit.SECONDS)
        return if (seconds > 0) Duration.ofSeconds(seconds) else null
    }
    
    // Hash operations
    fun hSet(key: String, field: String, value: Any) {
        redisTemplate.opsForHash<String, Any>().put(key, field, value)
    }
    
    fun hGet(key: String, field: String): Any? {
        return redisTemplate.opsForHash<String, Any>().get(key, field)
    }
    
    fun hGetAll(key: String): Map<String, Any> {
        return redisTemplate.opsForHash<String, Any>().entries(key)
    }
    
    // List operations (queue/stack)
    fun lPush(key: String, vararg values: Any) {
        redisTemplate.opsForList().leftPushAll(key, *values)
    }
    
    fun rPop(key: String): Any? {
        return redisTemplate.opsForList().rightPop(key)
    }
    
    fun lRange(key: String, start: Long = 0, end: Long = -1): List<Any> {
        return redisTemplate.opsForList().range(key, start, end) ?: emptyList()
    }
    
    // Set operations
    fun sAdd(key: String, vararg values: Any) {
        redisTemplate.opsForSet().add(key, *values)
    }
    
    fun sMembers(key: String): Set<Any> {
        return redisTemplate.opsForSet().members(key) ?: emptySet()
    }
    
    // Sorted set (leaderboard etc.)
    fun zAdd(key: String, value: Any, score: Double) {
        redisTemplate.opsForZSet().add(key, value, score)
    }
    
    fun zRangeWithScores(key: String, start: Long, end: Long): Set<ZSetOperations.TypedTuple<Any>> {
        return redisTemplate.opsForZSet().rangeWithScores(key, start, end) ?: emptySet()
    }
    
    // Atomic increment
    fun increment(key: String, delta: Long = 1): Long {
        return stringRedisTemplate.opsForValue().increment(key, delta) ?: 0L
    }
    
    // Pattern scan
    fun keys(pattern: String): Set<String> {
        return redisTemplate.keys(pattern)
    }
}
```

---

## Cache Patterns

```kotlin
// Cache-Aside Pattern (Lazy Loading)
@Service
class UserCacheService(
    private val cacheRepo: CacheRepository,
    private val userRepository: UserRepository
) {
    companion object {
        private const val USER_PREFIX = "user:"
        private val USER_TTL = Duration.ofMinutes(30)
    }
    
    fun getUser(id: String): User? {
        // 1. Check cache
        val cached = cacheRepo.get("$USER_PREFIX$id", User::class.java)
        if (cached != null) return cached
        
        // 2. Load from DB
        val user = userRepository.findById(id) ?: return null
        
        // 3. Store in cache
        cacheRepo.set("$USER_PREFIX$id", user, USER_TTL)
        
        return user
    }
    
    fun updateUser(user: User): User {
        val updated = userRepository.save(user)
        
        // Invalidate cache
        cacheRepo.delete("$USER_PREFIX${user.id}")
        
        return updated
    }
}

// Write-Through Pattern
@Service
class WriteThroughUserService(
    private val cacheRepo: CacheRepository,
    private val userRepository: UserRepository
) {
    fun saveUser(user: User): User {
        // 1. Write to DB
        val saved = userRepository.save(user)
        
        // 2. Write to cache simultaneously
        cacheRepo.set("user:${saved.id}", saved, Duration.ofMinutes(30))
        
        return saved
    }
}

// Read-Through Pattern with Kotlin generics
class ReadThroughCache<K, V : Any>(
    private val prefix: String,
    private val ttl: Duration,
    private val loader: (K) -> V?,
    private val cacheRepo: CacheRepository,
    private val valueClass: Class<V>
) {
    fun get(key: K): V? {
        val cacheKey = "$prefix$key"
        
        return cacheRepo.get(cacheKey, valueClass) ?: run {
            val value = loader(key) ?: return null
            cacheRepo.set(cacheKey, value, ttl)
            value
        }
    }
    
    fun invalidate(key: K) {
        cacheRepo.delete("$prefix$key")
    }
}

// Usage
val productCache = ReadThroughCache(
    prefix = "product:",
    ttl = Duration.ofHours(1),
    loader = { id: String -> productRepository.findById(id) },
    cacheRepo = cacheRepo,
    valueClass = Product::class.java
)
```

---

## Rate Limiting ด้วย Redis (Lua Script)

```kotlin
@Service
class RedisRateLimiter(private val stringRedisTemplate: StringRedisTemplate) {
    
    // Atomic rate limiting using Lua script
    private val rateLimitScript = RedisScript.of("""
        local key = KEYS[1]
        local limit = tonumber(ARGV[1])
        local window = tonumber(ARGV[2])
        
        local current = redis.call('INCR', key)
        
        if current == 1 then
            redis.call('EXPIRE', key, window)
        end
        
        if current > limit then
            return 0
        end
        
        return current
    """.trimIndent(), Long::class.java)
    
    fun isAllowed(key: String, limit: Int, windowSeconds: Long): Boolean {
        val result = stringRedisTemplate.execute(
            rateLimitScript,
            listOf(key),
            limit.toString(),
            windowSeconds.toString()
        )
        return result != 0L && result != null
    }
    
    fun getRemainingRequests(key: String, limit: Int): Int {
        val current = stringRedisTemplate.opsForValue().get(key)?.toLongOrNull() ?: 0L
        return maxOf(0, limit - current.toInt())
    }
    
    // Sliding window rate limiter
    fun isAllowedSlidingWindow(key: String, limit: Int, windowMs: Long): Boolean {
        val now = System.currentTimeMillis()
        val windowStart = now - windowMs
        val uniqueMember = "$now-${java.util.UUID.randomUUID()}"
        
        return stringRedisTemplate.execute(object : SessionCallback<Boolean> {
            override fun <K, V> execute(operations: RedisOperations<K, V>): Boolean {
                @Suppress("UNCHECKED_CAST")
                val ops = operations as RedisOperations<String, String>
                
                ops.multi()
                ops.opsForZSet().removeRangeByScore(key, 0.0, windowStart.toDouble())
                ops.opsForZSet().add(key, uniqueMember, now.toDouble())
                ops.opsForZSet().count(key, Double.NEGATIVE_INFINITY, Double.POSITIVE_INFINITY)
                ops.expire(key, java.time.Duration.ofMillis(windowMs))
                val results = ops.exec()
                
                val count = results?.get(2) as? Long ?: 0L
                return count <= limit
            }
        }) ?: false
    }
}
```

---

## Pub/Sub

```kotlin
// Publisher
@Service
class EventPublisher(private val redisTemplate: RedisTemplate<String, Any>) {
    
    fun publish(channel: String, event: Any) {
        redisTemplate.convertAndSend(channel, event)
    }
    
    fun publishOrderEvent(event: OrderEvent) {
        publish("orders:events", event)
    }
}

// Subscriber
@Component
class OrderEventSubscriber : MessageListener {
    
    override fun onMessage(message: Message, pattern: ByteArray?) {
        val body = String(message.body)
        println("Received order event: $body")
        // Process event...
    }
}

// Configuration
@Configuration
class RedisMessagingConfig {
    
    @Bean
    fun orderEventSubscriber() = OrderEventSubscriber()
    
    @Bean
    fun redisMessageListenerContainer(
        connectionFactory: RedisConnectionFactory,
        subscriber: OrderEventSubscriber
    ): RedisMessageListenerContainer {
        return RedisMessageListenerContainer().apply {
            setConnectionFactory(connectionFactory)
            addMessageListener(subscriber, PatternTopic("orders:*"))
        }
    }
}

// Reactive Pub/Sub with Kotlin Flow
@Service
class ReactiveEventService(private val reactiveRedisTemplate: ReactiveRedisTemplate<String, String>) {
    
    fun subscribe(channel: String): kotlinx.coroutines.flow.Flow<String> {
        return reactiveRedisTemplate
            .listenToChannel(channel)
            .map { it.message }
            .asFlow()
    }
    
    suspend fun publish(channel: String, message: String) {
        reactiveRedisTemplate.convertAndSend(channel, message).awaitFirst()
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Leaderboard System

@Service
class LeaderboardService(
    private val cacheRepo: CacheRepository,
    private val userRepository: UserRepository
) {
    companion object {
        private const val LEADERBOARD_KEY = "leaderboard:global"
        private const val TOP_PLAYERS = 100L
    }
    
    // Add/update score
    fun addScore(userId: String, points: Long): Double {
        val key = LEADERBOARD_KEY
        cacheRepo.zAdd(key, userId, points.toDouble())  // Simplified
        return points.toDouble()
    }
    
    // Get top N players
    fun getTopPlayers(n: Int = 10): List<PlayerRank> {
        return cacheRepo.zRangeWithScores(LEADERBOARD_KEY, 0, n.toLong() - 1)
            .mapIndexed { index, tuple ->
                PlayerRank(
                    rank = index + 1,
                    userId = tuple.value.toString(),
                    score = tuple.score?.toLong() ?: 0
                )
            }
    }
    
    // Get player rank
    fun getPlayerRank(userId: String): Long? {
        // TODO: Implement using ZREVRANK
        return null
    }
    
    // Get players around a rank
    fun getNeighbors(userId: String, range: Int = 5): List<PlayerRank> {
        // TODO: Implement getting neighbors of a player in leaderboard
        return emptyList()
    }
}

data class PlayerRank(
    val rank: Int,
    val userId: String,
    val score: Long,
    val username: String = ""
)

// Placeholder interfaces
interface ProductRepository {
    fun findById(id: String): java.util.Optional<Product>
    fun findByCategory(category: String): List<Product>
    fun save(product: Product): Product
    fun deleteById(id: String)
}

interface UserRepository {
    fun findById(id: String): User?
    fun save(user: User): User
}

data class Product(val id: String, val name: String, val price: Double, val stock: Int, val category: String)
data class User(val id: String, val username: String, val email: String)
data class UpdateProductRequest(val name: String?, val price: Double?, val stock: Int?)
data class OrderEvent(val orderId: String, val type: String)

typealias ZSetOperations = org.springframework.data.redis.core.ZSetOperations
typealias SessionCallback<T> = org.springframework.data.redis.core.SessionCallback<T>
typealias RedisOperations<K, V> = org.springframework.data.redis.core.RedisOperations<K, V>
```

---

## สรุป Part 41

```
✅ Redis: in-memory key-value store สำหรับ caching
✅ @Cacheable/@CachePut/@CacheEvict: declarative caching
✅ RedisTemplate: low-level Redis operations
✅ String, Hash, List, Set, Sorted Set: data structures
✅ Cache-Aside: load on miss, invalidate on update
✅ Write-Through: write DB and cache simultaneously
✅ Lua scripts: atomic operations in Redis
✅ Rate limiting: Lua script + sliding window with ZSet
✅ Pub/Sub: publish and subscribe to channels
✅ TTL: automatic cache expiry
✅ Leaderboard: Sorted Set เหมาะมากสำหรับ ranking
```

---

*Part 41/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
