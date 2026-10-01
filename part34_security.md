# Part 34: Security ใน Kotlin Applications

## สารบัญ
1. [Authentication กับ JWT](#authentication-กับ-jwt)
2. [Password Hashing](#password-hashing)
3. [Input Validation](#input-validation)
4. [SQL Injection Prevention](#sql-injection-prevention)
5. [CSRF Protection](#csrf-protection)
6. [Rate Limiting](#rate-limiting)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Authentication กับ JWT

```kotlin
// JWT utility
import io.jsonwebtoken.*
import io.jsonwebtoken.security.Keys
import java.util.Date
import javax.crypto.SecretKey

data class JwtConfig(
    val secret: String,
    val accessTokenTtlMs: Long = 15 * 60 * 1000,      // 15 minutes
    val refreshTokenTtlMs: Long = 7 * 24 * 3600 * 1000 // 7 days
)

data class TokenPair(val accessToken: String, val refreshToken: String)

data class UserClaims(
    val userId: String,
    val email: String,
    val roles: List<String>
)

class JwtService(private val config: JwtConfig) {
    private val key: SecretKey = Keys.hmacShaKeyFor(config.secret.toByteArray())
    
    fun generateTokens(claims: UserClaims): TokenPair {
        return TokenPair(
            accessToken = createToken(claims, config.accessTokenTtlMs, "access"),
            refreshToken = createToken(claims, config.refreshTokenTtlMs, "refresh")
        )
    }
    
    private fun createToken(claims: UserClaims, ttlMs: Long, type: String): String {
        val now = Date()
        return Jwts.builder()
            .subject(claims.userId)
            .claim("email", claims.email)
            .claim("roles", claims.roles)
            .claim("type", type)
            .issuedAt(now)
            .expiration(Date(now.time + ttlMs))
            .signWith(key)
            .compact()
    }
    
    fun validateToken(token: String): Result<UserClaims> = runCatching {
        val claims = Jwts.parser()
            .verifyWith(key)
            .build()
            .parseSignedClaims(token)
            .payload
        
        UserClaims(
            userId = claims.subject,
            email = claims["email"] as String,
            roles = @Suppress("UNCHECKED_CAST") (claims["roles"] as List<String>)
        )
    }
    
    fun refreshTokens(refreshToken: String): Result<TokenPair> = runCatching {
        val claims = validateToken(refreshToken).getOrThrow()
        generateTokens(claims)
    }
    
    fun extractUserId(token: String): String? {
        return validateToken(token).getOrNull()?.userId
    }
}

// Ktor JWT middleware
fun Application.configureAuth(jwtService: JwtService) {
    install(Authentication) {
        bearer("jwt") {
            authenticate { credentials ->
                jwtService.validateToken(credentials.token)
                    .map { claims -> 
                        JWTPrincipal(claims)
                    }
                    .getOrNull()
            }
        }
    }
}

data class JWTPrincipal(val claims: UserClaims) : io.ktor.server.auth.Principal

// Route protection
fun Routing.protectedRoutes(jwtService: JwtService) {
    authenticate("jwt") {
        get("/profile") {
            val principal = call.principal<JWTPrincipal>()!!
            call.respond(mapOf("userId" to principal.claims.userId))
        }
        
        // Role-based access
        route("/admin") {
            intercept(ApplicationCallPipeline.Call) {
                val principal = call.principal<JWTPrincipal>()
                if (principal?.claims?.roles?.contains("ADMIN") != true) {
                    call.respond(HttpStatusCode.Forbidden, "Admin access required")
                    return@intercept finish()
                }
            }
            
            get("/users") {
                call.respond(listOf("admin endpoint"))
            }
        }
    }
}
```

---

## Password Hashing

```kotlin
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder
import java.security.MessageDigest
import java.security.SecureRandom
import java.util.Base64

// BCrypt (recommended for passwords)
class PasswordService {
    private val bcrypt = BCryptPasswordEncoder(12)  // cost factor 12
    
    fun hash(rawPassword: String): String {
        require(rawPassword.length >= 8) { "Password too short" }
        return bcrypt.encode(rawPassword)
    }
    
    fun verify(rawPassword: String, hashedPassword: String): Boolean {
        return bcrypt.matches(rawPassword, hashedPassword)
    }
    
    fun needsRehash(hashedPassword: String): Boolean {
        val currentCost = BCryptPasswordEncoder(12).upgradeEncoding(hashedPassword)
        return currentCost
    }
}

// Custom PBKDF2 implementation
import javax.crypto.SecretKeyFactory
import javax.crypto.spec.PBEKeySpec

class Pbkdf2PasswordEncoder {
    companion object {
        private const val ALGORITHM = "PBKDF2WithHmacSHA256"
        private const val ITERATIONS = 310_000
        private const val KEY_LENGTH = 256
        private const val SALT_LENGTH = 16
    }
    
    private val random = SecureRandom()
    
    fun encode(password: String): String {
        val salt = ByteArray(SALT_LENGTH).apply { random.nextBytes(this) }
        val hash = pbkdf2(password.toCharArray(), salt, ITERATIONS, KEY_LENGTH)
        val saltBase64 = Base64.getEncoder().encodeToString(salt)
        val hashBase64 = Base64.getEncoder().encodeToString(hash)
        return "$ITERATIONS:$saltBase64:$hashBase64"
    }
    
    fun matches(rawPassword: String, encodedPassword: String): Boolean {
        val parts = encodedPassword.split(":")
        if (parts.size != 3) return false
        
        val iterations = parts[0].toInt()
        val salt = Base64.getDecoder().decode(parts[1])
        val expectedHash = Base64.getDecoder().decode(parts[2])
        
        val actualHash = pbkdf2(rawPassword.toCharArray(), salt, iterations, KEY_LENGTH)
        return MessageDigest.isEqual(actualHash, expectedHash)
    }
    
    private fun pbkdf2(password: CharArray, salt: ByteArray, iterations: Int, keyLength: Int): ByteArray {
        val spec = PBEKeySpec(password, salt, iterations, keyLength)
        val factory = SecretKeyFactory.getInstance(ALGORITHM)
        return factory.generateSecret(spec).encoded
    }
}

fun main() {
    val pwService = PasswordService()
    
    val rawPassword = "MySecurePass123!"
    val hashed = pwService.hash(rawPassword)
    println("Hashed: $hashed")
    println("Verify correct: ${pwService.verify(rawPassword, hashed)}")
    println("Verify wrong: ${pwService.verify("wrongpass", hashed)}")
    
    val pbkdf2 = Pbkdf2PasswordEncoder()
    val encoded = pbkdf2.encode(rawPassword)
    println("\nPBKDF2 encoded: $encoded")
    println("Verify: ${pbkdf2.matches(rawPassword, encoded)}")
}
```

---

## Input Validation

```kotlin
// Comprehensive input sanitization and validation

object InputSanitizer {
    // Remove HTML tags
    fun stripHtml(input: String): String {
        return input.replace("<[^>]*>".toRegex(), "")
    }
    
    // Escape HTML entities
    fun escapeHtml(input: String): String {
        return input
            .replace("&", "&amp;")
            .replace("<", "&lt;")
            .replace(">", "&gt;")
            .replace("\"", "&quot;")
            .replace("'", "&#x27;")
            .replace("/", "&#x2F;")
    }
    
    // Remove potentially dangerous characters
    fun sanitizeForLog(input: String): String {
        return input.replace("[\r\n\t]".toRegex(), " ")
            .take(200)  // limit length
    }
    
    // SQL-safe (prefer parameterized queries, this is last resort)
    fun escapeSql(input: String): String {
        return input.replace("'", "''")
            .replace("\\", "\\\\")
    }
    
    // Normalize whitespace
    fun normalizeWhitespace(input: String): String {
        return input.trim().replace("\\s+".toRegex(), " ")
    }
}

// Validation DSL
class ValidationScope<T>(val value: T) {
    val errors = mutableListOf<String>()
    
    fun require(condition: Boolean, message: () -> String) {
        if (!condition) errors.add(message())
    }
    
    fun requireNotNull(message: String = "Value cannot be null") {
        if (value == null) errors.add(message)
    }
    
    val isValid: Boolean get() = errors.isEmpty()
}

fun <T> validate(value: T, block: ValidationScope<T>.() -> Unit): Result<T> {
    val scope = ValidationScope(value)
    scope.block()
    return if (scope.isValid) Result.success(value)
    else Result.failure(IllegalArgumentException(scope.errors.joinToString("; ")))
}

// Request validators
data class SignupRequest(val username: String, val email: String, val password: String)

fun validateSignupRequest(request: SignupRequest): Result<SignupRequest> {
    return validate(request) {
        require(value.username.length in 3..50) { "Username must be 3-50 characters" }
        require(value.username.matches("[a-zA-Z0-9_.-]+".toRegex())) { "Username: only letters, digits, _ . -" }
        require(value.email.matches("[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}".toRegex())) {
            "Invalid email format"
        }
        require(value.password.length >= 8) { "Password too short (min 8)" }
        require(value.password.any { it.isUpperCase() }) { "Password must have uppercase" }
        require(value.password.any { it.isDigit() }) { "Password must have digit" }
        require(value.password.any { "!@#\$%^&*()_+".contains(it) }) { "Password must have special char" }
        require(!value.password.contains(value.username, ignoreCase = true)) {
            "Password cannot contain username"
        }
    }
}

fun main() {
    // Sanitization
    val malicious = "<script>alert('xss')</script>Hello"
    println("Stripped: ${InputSanitizer.stripHtml(malicious)}")
    println("Escaped: ${InputSanitizer.escapeHtml(malicious)}")
    
    val logInput = "User input\r\nwith newlines\t and tabs"
    println("Safe log: ${InputSanitizer.sanitizeForLog(logInput)}")
    
    println()
    
    // Validation
    val valid = validateSignupRequest(SignupRequest("user123", "user@email.com", "Secure@Pass1"))
    println("Valid request: ${valid.isSuccess}")
    
    val invalid = validateSignupRequest(SignupRequest("ab", "not-an-email", "weak"))
    println("Invalid request: ${invalid.exceptionOrNull()?.message}")
}
```

---

## Rate Limiting

```kotlin
import java.util.concurrent.ConcurrentHashMap
import java.util.concurrent.atomic.AtomicInteger
import java.time.Instant

// Token Bucket algorithm
class TokenBucket(
    private val capacity: Int,
    private val refillRatePerSecond: Double
) {
    private var tokens = capacity.toDouble()
    private var lastRefillTime = System.currentTimeMillis()
    
    @Synchronized
    fun consume(tokens: Int = 1): Boolean {
        refill()
        return if (this.tokens >= tokens) {
            this.tokens -= tokens
            true
        } else {
            false
        }
    }
    
    private fun refill() {
        val now = System.currentTimeMillis()
        val elapsed = (now - lastRefillTime) / 1000.0
        tokens = minOf(capacity.toDouble(), tokens + elapsed * refillRatePerSecond)
        lastRefillTime = now
    }
    
    val currentTokens: Int get() = synchronized(this) { refill(); tokens.toInt() }
}

// Sliding window rate limiter
class SlidingWindowRateLimiter(
    private val maxRequests: Int,
    private val windowMs: Long
) {
    private val windows = ConcurrentHashMap<String, ArrayDeque<Long>>()
    
    @Synchronized
    fun isAllowed(key: String): Boolean {
        val now = System.currentTimeMillis()
        val window = windows.getOrPut(key) { ArrayDeque() }
        
        // Remove expired timestamps
        while (window.isNotEmpty() && window.first() < now - windowMs) {
            window.removeFirst()
        }
        
        return if (window.size < maxRequests) {
            window.addLast(now)
            true
        } else {
            false
        }
    }
    
    fun getRemainingRequests(key: String): Int {
        val now = System.currentTimeMillis()
        val window = windows[key] ?: return maxRequests
        val active = window.count { it >= now - windowMs }
        return maxOf(0, maxRequests - active)
    }
    
    fun cleanup() {
        val now = System.currentTimeMillis()
        windows.entries.removeIf { (_, window) ->
            window.removeIf { it < now - windowMs }
            window.isEmpty()
        }
    }
}

// Ktor rate limiting middleware
class RateLimitPlugin(
    private val limiter: SlidingWindowRateLimiter,
    private val keyExtractor: (ApplicationCall) -> String = { call ->
        call.request.header("X-Forwarded-For") ?: call.request.local.remoteAddress
    }
) {
    fun intercept(call: ApplicationCall, proceed: suspend () -> Unit) {
        val key = keyExtractor(call)
        val remaining = limiter.getRemainingRequests(key)
        
        call.response.header("X-RateLimit-Remaining", remaining.toString())
        call.response.header("X-RateLimit-Limit", "100")
        
        if (!limiter.isAllowed(key)) {
            call.respond(HttpStatusCode.TooManyRequests, mapOf(
                "error" to "Rate limit exceeded",
                "retryAfter" to 60
            ))
            return
        }
        
        proceed()
    }
}

fun main() {
    // Token bucket
    val bucket = TokenBucket(capacity = 10, refillRatePerSecond = 2.0)
    
    println("Token Bucket Test (capacity=10, refill=2/s):")
    repeat(15) { i ->
        val allowed = bucket.consume()
        println("  Request ${i+1}: ${if (allowed) "✅ allowed" else "❌ denied"} (tokens: ${bucket.currentTokens})")
        if (i == 9) Thread.sleep(2000)  // wait 2s to refill
    }
    
    println()
    
    // Sliding window
    val limiter = SlidingWindowRateLimiter(maxRequests = 5, windowMs = 1000L)
    
    println("Sliding Window Test (5 req/second):")
    repeat(8) { i ->
        val allowed = limiter.isAllowed("user-1")
        println("  Request ${i+1}: ${if (allowed) "✅ allowed" else "❌ denied"} (remaining: ${limiter.getRemainingRequests("user-1")})")
    }
    
    Thread.sleep(1100)
    println("  After 1s:")
    val allowed = limiter.isAllowed("user-1")
    println("  Request 9: ${if (allowed) "✅ allowed" else "❌ denied"} (remaining: ${limiter.getRemainingRequests("user-1")})")
}

// Placeholder imports for Ktor
typealias ApplicationCall = Any
typealias HttpStatusCode = Any
typealias ApplicationCallPipeline = Any
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Secure API Key Management

import java.security.SecureRandom
import java.util.Base64

data class ApiKey(
    val id: String,
    val hashedKey: String,
    val userId: String,
    val name: String,
    val scopes: Set<String>,
    val createdAt: Long = System.currentTimeMillis(),
    val lastUsedAt: Long? = null,
    val expiresAt: Long? = null,
    val isActive: Boolean = true
) {
    val isExpired: Boolean
        get() = expiresAt != null && expiresAt < System.currentTimeMillis()
    
    val isValid: Boolean
        get() = isActive && !isExpired
    
    fun hasScope(scope: String) = scope in scopes || "admin" in scopes
}

class ApiKeyService(
    private val passwordEncoder: Pbkdf2PasswordEncoder = Pbkdf2PasswordEncoder()
) {
    private val keys = mutableListOf<ApiKey>()
    private val random = SecureRandom()
    
    data class CreateResult(val apiKey: ApiKey, val rawKey: String)
    
    fun create(
        userId: String,
        name: String,
        scopes: Set<String>,
        expiresInDays: Int? = null
    ): CreateResult {
        val rawKey = generateKey()
        val hashed = passwordEncoder.encode(rawKey)
        val expiresAt = expiresInDays?.let { 
            System.currentTimeMillis() + it.toLong() * 24 * 3600 * 1000 
        }
        
        val apiKey = ApiKey(
            id = java.util.UUID.randomUUID().toString(),
            hashedKey = hashed,
            userId = userId,
            name = name,
            scopes = scopes,
            expiresAt = expiresAt
        )
        
        keys.add(apiKey)
        return CreateResult(apiKey, rawKey)
    }
    
    fun validate(rawKey: String, requiredScope: String? = null): ApiKey? {
        val key = keys.find { 
            it.isValid && passwordEncoder.matches(rawKey, it.hashedKey)
        } ?: return null
        
        if (requiredScope != null && !key.hasScope(requiredScope)) return null
        
        // Update last used
        val index = keys.indexOf(key)
        keys[index] = key.copy(lastUsedAt = System.currentTimeMillis())
        
        return keys[index]
    }
    
    fun revoke(keyId: String): Boolean {
        val index = keys.indexOfFirst { it.id == keyId }
        if (index < 0) return false
        keys[index] = keys[index].copy(isActive = false)
        return true
    }
    
    fun listForUser(userId: String) = keys.filter { it.userId == userId }
    
    private fun generateKey(): String {
        val bytes = ByteArray(32).apply { random.nextBytes(this) }
        return "sk_" + Base64.getUrlEncoder().withoutPadding().encodeToString(bytes)
    }
}

fun main() {
    val service = ApiKeyService()
    
    // Create API key
    val (key, rawKey) = service.create(
        userId = "user-123",
        name = "Production API",
        scopes = setOf("read:products", "write:orders"),
        expiresInDays = 30
    )
    
    println("Created API Key:")
    println("  Raw key: $rawKey")
    println("  Key ID: ${key.id}")
    println("  Scopes: ${key.scopes}")
    
    println()
    
    // Validate
    val validated = service.validate(rawKey, "read:products")
    println("Valid for read:products: ${validated != null}")
    
    val invalid = service.validate(rawKey, "admin")
    println("Valid for admin: ${invalid != null}")
    
    val wrongKey = service.validate("sk_wrongkey", "read:products")
    println("Wrong key: ${wrongKey != null}")
    
    // Revoke
    service.revoke(key.id)
    val afterRevoke = service.validate(rawKey, "read:products")
    println("After revoke: ${afterRevoke != null}")
}
```

---

## สรุป Part 34

```
✅ JWT: stateless authentication token
✅ createToken() + validateToken(): sign/verify JWT
✅ BCrypt: recommended password hashing algorithm
✅ PBKDF2: alternative password hashing
✅ PasswordEncoder.matches(): timing-safe comparison
✅ Input sanitization: escape HTML, strip tags
✅ Input validation: check length, format, content
✅ Parameterized queries: prevent SQL injection
✅ Token Bucket: smooth rate limiting
✅ Sliding Window: strict rate limiting per time window
✅ API Keys: hashed storage, scope-based access control
✅ Never store plaintext passwords or API keys
✅ Use SecureRandom for cryptographic operations
```

---

*Part 34/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
