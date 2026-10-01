# Part 94: Security ระดับ Enterprise

## สารบัญ
1. [Zero Trust Architecture](#zero-trust-architecture)
2. [API Security Best Practices](#api-security-best-practices)
3. [Secrets Management ด้วย Vault](#secrets-management-ด้วย-vault)
4. [OWASP Top 10 Prevention](#owasp-top-10-prevention)
5. [Security Testing](#security-testing)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Zero Trust Architecture

```
Zero Trust หลักการ: "Never trust, always verify"

Traditional: ภายใน network เชื่อถือได้
Zero Trust: ไม่เชื่อใครทั้งนั้น ตรวจสอบทุกครั้ง

Pillars:
1. Identity verification: ทุก request ต้องพิสูจน์ identity
2. Least privilege: ให้สิทธิ์น้อยที่สุดเท่าที่จำเป็น
3. Assume breach: ออกแบบระบบราวกับว่าจะถูก hack
4. Explicit verification: verify explicitly, not by location
5. Micro-segmentation: แบ่ง network zone ย่อย

Implementation:
┌─────────────────────────────────────────────┐
│  Internet                                   │
│    ↓                                        │
│  [WAF] → filter malicious traffic           │
│    ↓                                        │
│  [API Gateway] → auth, rate limit, routing  │
│    ↓                                        │
│  [mTLS] → service-to-service auth           │
│    ↓                                        │
│  [Service Mesh] → Istio/Linkerd             │
│    ↓                                        │
│  [RBAC] → fine-grained permissions          │
└─────────────────────────────────────────────┘
```

---

## API Security Best Practices

```kotlin
// 1. JWT Validation ที่ครบถ้วน
import com.auth0.jwt.JWT
import com.auth0.jwt.algorithms.Algorithm
import com.auth0.jwt.exceptions.*
import java.time.Instant

@org.springframework.stereotype.Service
class JwtValidationService(private val config: JwtConfig) {
    
    private val algorithm = Algorithm.RSA256(config.publicKey, config.privateKey)
    private val verifier = JWT.require(algorithm)
        .withIssuer(config.issuer)
        .withAudience(config.audience)
        .build()
    
    data class JwtClaims(
        val subject: String,
        val roles: List<String>,
        val tenantId: String,
        val tokenId: String,
        val issuedAt: Instant,
        val expiresAt: Instant
    )
    
    fun validate(token: String): JwtClaims {
        val decoded = try {
            verifier.verify(token)
        } catch (e: TokenExpiredException) {
            throw SecurityException("Token expired at ${e.expiredOn}", SecurityErrorCode.TOKEN_EXPIRED)
        } catch (e: SignatureVerificationException) {
            throw SecurityException("Invalid token signature", SecurityErrorCode.INVALID_SIGNATURE)
        } catch (e: JWTDecodeException) {
            throw SecurityException("Malformed token", SecurityErrorCode.MALFORMED_TOKEN)
        }
        
        // Check token not revoked
        val jti = decoded.id ?: throw SecurityException("Missing jti claim", SecurityErrorCode.MISSING_CLAIM)
        if (tokenRevocationService.isRevoked(jti)) {
            throw SecurityException("Token has been revoked", SecurityErrorCode.TOKEN_REVOKED)
        }
        
        return JwtClaims(
            subject = decoded.subject,
            roles = decoded.getClaim("roles").asList(String::class.java) ?: emptyList(),
            tenantId = decoded.getClaim("tenantId").asString() 
                ?: throw SecurityException("Missing tenantId", SecurityErrorCode.MISSING_CLAIM),
            tokenId = jti,
            issuedAt = decoded.issuedAtAsInstant,
            expiresAt = decoded.expiresAtAsInstant
        )
    }
    
    // Token rotation: generate new access token from refresh token
    fun rotateTokens(refreshToken: String): TokenPair {
        val claims = validate(refreshToken)
        
        // Revoke old refresh token (token rotation)
        tokenRevocationService.revoke(claims.tokenId)
        
        return tokenService.generatePair(claims.subject, claims.roles, claims.tenantId)
    }
}

data class TokenPair(val accessToken: String, val refreshToken: String)
enum class SecurityErrorCode { TOKEN_EXPIRED, INVALID_SIGNATURE, MALFORMED_TOKEN, TOKEN_REVOKED, MISSING_CLAIM }
class SecurityException(msg: String, val code: SecurityErrorCode) : RuntimeException(msg)
interface TokenRevocationService { fun isRevoked(jti: String): Boolean; fun revoke(jti: String) }
interface TokenService { fun generatePair(subject: String, roles: List<String>, tenantId: String): TokenPair }
class JwtConfig { val publicKey: java.security.interfaces.RSAPublicKey? = null; val privateKey: java.security.interfaces.RSAPrivateKey? = null; val issuer = ""; val audience = ""}
lateinit var tokenRevocationService: TokenRevocationService
lateinit var tokenService: TokenService

// 2. Rate Limiting ขั้นสูง
@org.springframework.stereotype.Component
class AdvancedRateLimiter(
    private val redisTemplate: org.springframework.data.redis.core.RedisTemplate<String, String>
) {
    
    // Sliding window rate limiter
    fun checkLimit(key: String, limit: Int, windowSeconds: Long): Boolean {
        val now = System.currentTimeMillis()
        val windowStart = now - (windowSeconds * 1000)
        val redisKey = "ratelimit:$key"
        
        val ops = redisTemplate.opsForZSet()
        
        // Remove expired entries
        ops.removeRangeByScore(redisKey, Double.NEGATIVE_INFINITY, windowStart.toDouble())
        
        // Count current requests
        val count = ops.zCard(redisKey) ?: 0
        
        if (count >= limit) return false
        
        // Add current request
        ops.add(redisKey, now.toString(), now.toDouble())
        redisTemplate.expire(redisKey, java.time.Duration.ofSeconds(windowSeconds))
        
        return true
    }
    
    // Token bucket for burst traffic
    fun tokenBucket(key: String, capacity: Int, refillRatePerSecond: Double): Boolean {
        val script = """
            local key = KEYS[1]
            local capacity = tonumber(ARGV[1])
            local refill_rate = tonumber(ARGV[2])
            local now = tonumber(ARGV[3])
            
            local bucket = redis.call('hmget', key, 'tokens', 'last_refill')
            local tokens = tonumber(bucket[1]) or capacity
            local last_refill = tonumber(bucket[2]) or now
            
            local elapsed = now - last_refill
            local new_tokens = math.min(capacity, tokens + (elapsed * refill_rate / 1000))
            
            if new_tokens < 1 then
                return 0
            end
            
            redis.call('hmset', key, 'tokens', new_tokens - 1, 'last_refill', now)
            redis.call('expire', key, 3600)
            return 1
        """
        
        val result = redisTemplate.execute(
            org.springframework.data.redis.core.script.DefaultRedisScript(script, Long::class.java),
            listOf("tokenbucket:$key"),
            capacity.toString(),
            refillRatePerSecond.toString(),
            System.currentTimeMillis().toString()
        )
        
        return result == 1L
    }
}

// 3. Input Validation & Sanitization
@org.springframework.stereotype.Service
class InputSanitizer {
    
    // SQL Injection prevention (use parameterized queries, but also validate)
    fun validateProductName(name: String): String {
        if (name.length > 200) throw IllegalArgumentException("Name too long")
        
        // Allow only safe characters
        val sanitized = name.trim()
        if (!sanitized.matches(Regex("^[\\p{L}\\p{N}\\s\\-_.,!?()]+$"))) {
            throw IllegalArgumentException("Name contains invalid characters")
        }
        
        return sanitized
    }
    
    // XSS prevention
    fun sanitizeHtml(input: String): String {
        return org.owasp.html.HtmlPolicyBuilder()
            .allowElements("p", "b", "i", "ul", "li", "br")
            .allowAttributes("class").onElements("p")
            .toFactory()
            .sanitize(input)
    }
    
    // Path traversal prevention
    fun validateFilePath(path: String, allowedBase: String): java.io.File {
        val file = java.io.File(allowedBase, path).canonicalFile
        val base = java.io.File(allowedBase).canonicalFile
        
        if (!file.path.startsWith(base.path)) {
            throw SecurityException("Path traversal detected: $path", SecurityErrorCode.MALFORMED_TOKEN)
        }
        
        return file
    }
    
    // SSRF prevention
    fun validateUrl(urlString: String): java.net.URL {
        val url = java.net.URL(urlString)
        
        if (url.protocol !in listOf("http", "https")) {
            throw IllegalArgumentException("Only HTTP/HTTPS allowed")
        }
        
        val blockedHosts = setOf("localhost", "127.0.0.1", "0.0.0.0", "169.254.169.254")
        val host = url.host.lowercase()
        
        if (host in blockedHosts || host.startsWith("192.168.") || host.startsWith("10.")) {
            throw IllegalArgumentException("Internal URLs not allowed")
        }
        
        // Resolve hostname to check for DNS rebinding
        val addresses = java.net.InetAddress.getAllByName(host)
        for (addr in addresses) {
            if (addr.isLoopbackAddress || addr.isSiteLocalAddress || addr.isLinkLocalAddress) {
                throw IllegalArgumentException("Internal IP not allowed: ${addr.hostAddress}")
            }
        }
        
        return url
    }
}

// 4. Audit Logging
@org.springframework.stereotype.Component
class SecurityAuditService(
    private val auditRepository: AuditRepository
) {
    
    private val log = org.slf4j.LoggerFactory.getLogger(SecurityAuditService::class.java)
    
    fun logAuthEvent(userId: String, event: AuthEventType, ipAddress: String, userAgent: String, success: Boolean, details: Map<String, Any> = emptyMap()) {
        val auditEvent = AuditEvent(
            id = java.util.UUID.randomUUID().toString(),
            userId = userId,
            eventType = event.name,
            ipAddress = ipAddress,
            userAgent = userAgent,
            success = success,
            details = com.fasterxml.jackson.databind.ObjectMapper().writeValueAsString(details),
            timestamp = java.time.Instant.now()
        )
        
        auditRepository.save(auditEvent)
        
        if (!success) {
            log.warn(
                "SECURITY_ALERT type=${event.name} userId=$userId ip=$ipAddress " +
                "userAgent=${userAgent.take(100)} details=$details"
            )
        }
    }
    
    fun detectAnomalies(userId: String): List<SecurityAnomaly> {
        val recentLogins = auditRepository.findRecentLogins(userId, java.time.Duration.ofHours(1))
        val anomalies = mutableListOf<SecurityAnomaly>()
        
        // Multiple IPs
        val uniqueIps = recentLogins.map { it.ipAddress }.toSet()
        if (uniqueIps.size > 5) {
            anomalies.add(SecurityAnomaly("Multiple IPs in 1 hour", "HIGH"))
        }
        
        // Failed login attempts
        val failedLogins = recentLogins.count { !it.success }
        if (failedLogins > 5) {
            anomalies.add(SecurityAnomaly("$failedLogins failed logins in 1 hour", "HIGH"))
        }
        
        return anomalies
    }
}

enum class AuthEventType { LOGIN, LOGOUT, TOKEN_REFRESH, PASSWORD_CHANGE, TOKEN_REVOKE, MFA_SETUP }
data class AuditEvent(val id: String, val userId: String, val eventType: String, val ipAddress: String, val userAgent: String, val success: Boolean, val details: String, val timestamp: java.time.Instant)
data class SecurityAnomaly(val description: String, val severity: String)
interface AuditRepository {
    fun save(event: AuditEvent)
    fun findRecentLogins(userId: String, duration: java.time.Duration): List<AuditEvent>
}
```

---

## Secrets Management ด้วย Vault

```kotlin
// Spring Cloud Vault integration
// implementation("org.springframework.cloud:spring-cloud-starter-vault-config")

// bootstrap.yml
/*
spring:
  cloud:
    vault:
      host: vault.internal.example.com
      port: 8200
      scheme: https
      authentication: KUBERNETES  # use K8s service account
      kubernetes:
        role: ecommerce-api
        service-account-token-file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kv:
        enabled: true
        backend: secret
        default-context: ecommerce-api
      database:
        enabled: true
        role: ecommerce-readonly
        backend: database
*/

// Vault dynamic credentials: database credentials rotate automatically
@org.springframework.context.annotation.Configuration
class VaultDatabaseConfig {
    
    @org.springframework.context.annotation.Bean
    @org.springframework.context.annotation.Primary
    fun dataSource(
        vaultTemplate: org.springframework.vault.core.VaultTemplate
    ): javax.sql.DataSource {
        val credentials = vaultTemplate.read("database/creds/ecommerce-readonly")
            ?: throw IllegalStateException("Could not read DB credentials from Vault")
        
        val username = credentials.data?.get("username") as String
        val password = credentials.data?.get("password") as String
        
        return com.zaxxer.hikari.HikariDataSource(
            com.zaxxer.hikari.HikariConfig().apply {
                jdbcUrl = "jdbc:postgresql://postgres:5432/ecommerce"
                this.username = username
                this.password = password
                maximumPoolSize = 10
            }
        )
    }
}

// Rotating secrets: refresh credentials before expiry
@org.springframework.stereotype.Component
@org.springframework.scheduling.annotation.EnableScheduling
class SecretsRotationService(
    private val vaultTemplate: org.springframework.vault.core.VaultTemplate,
    private val dataSource: com.zaxxer.hikari.HikariDataSource
) {
    
    private val log = org.slf4j.LoggerFactory.getLogger(SecretsRotationService::class.java)
    
    @org.springframework.scheduling.annotation.Scheduled(fixedDelay = 3600000)  // every hour
    fun rotateDbCredentials() {
        log.info("Rotating database credentials")
        
        try {
            val newCreds = vaultTemplate.read("database/creds/ecommerce-readonly")
                ?: return
            
            val username = newCreds.data?.get("username") as String
            val password = newCreds.data?.get("password") as String
            
            // HikariCP supports dynamic credentials update
            dataSource.username = username
            dataSource.password = password
            dataSource.hikariConfigMXBean.setPassword(password)
            
            log.info("Database credentials rotated successfully")
        } catch (e: Exception) {
            log.error("Failed to rotate credentials", e)
        }
    }
}

// Encrypt sensitive data at application level
@org.springframework.stereotype.Service
class FieldEncryptionService(config: EncryptionConfig) {
    
    private val cipher = javax.crypto.Cipher.getInstance("AES/GCM/NoPadding")
    private val keySpec = javax.crypto.spec.SecretKeySpec(
        java.util.Base64.getDecoder().decode(config.encryptionKey),
        "AES"
    )
    
    fun encrypt(plaintext: String): String {
        val iv = ByteArray(12).also { java.security.SecureRandom().nextBytes(it) }
        cipher.init(
            javax.crypto.Cipher.ENCRYPT_MODE,
            keySpec,
            javax.crypto.spec.GCMParameterSpec(128, iv)
        )
        
        val ciphertext = cipher.doFinal(plaintext.toByteArray(Charsets.UTF_8))
        val combined = iv + ciphertext
        
        return java.util.Base64.getEncoder().encodeToString(combined)
    }
    
    fun decrypt(encrypted: String): String {
        val combined = java.util.Base64.getDecoder().decode(encrypted)
        val iv = combined.copyOfRange(0, 12)
        val ciphertext = combined.copyOfRange(12, combined.size)
        
        cipher.init(
            javax.crypto.Cipher.DECRYPT_MODE,
            keySpec,
            javax.crypto.spec.GCMParameterSpec(128, iv)
        )
        
        return String(cipher.doFinal(ciphertext), Charsets.UTF_8)
    }
}

data class EncryptionConfig(val encryptionKey: String)
```

---

## OWASP Top 10 Prevention

```kotlin
// OWASP A01: Broken Access Control
@org.springframework.stereotype.Service
class AccessControlService {
    
    // Horizontal access control: user can only access their own data
    fun ensureOwnership(resourceId: String, userId: String) {
        val resource = resourceRepository.findById(resourceId)
            ?: throw NotFoundException("Resource not found")
        
        if (resource.ownerId != userId) {
            throw org.springframework.security.access.AccessDeniedException(
                "User $userId does not own resource $resourceId"
            )
        }
    }
    
    // Tenant isolation: multi-tenant data separation
    fun withTenantFilter(tenantId: String, block: (String) -> Any): Any {
        // All queries automatically filtered by tenantId
        return block(tenantId)
    }
}

// OWASP A02: Cryptographic Failures
@org.springframework.stereotype.Service
class CryptographyService {
    
    // Use strong hashing for passwords
    private val passwordEncoder = org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder(12)
    
    fun hashPassword(plaintext: String): String = passwordEncoder.encode(plaintext)
    fun verifyPassword(plaintext: String, hashed: String): Boolean = passwordEncoder.matches(plaintext, hashed)
    
    // Use secure random for tokens
    fun generateSecureToken(bytes: Int = 32): String {
        val random = ByteArray(bytes)
        java.security.SecureRandom.getInstanceStrong().nextBytes(random)
        return java.util.Base64.getUrlEncoder().withoutPadding().encodeToString(random)
    }
    
    // Hash PII for logs (never log plaintext email/phone)
    fun hashPii(value: String): String {
        val digest = java.security.MessageDigest.getInstance("SHA-256")
        return java.util.Base64.getEncoder().encodeToString(
            digest.digest(value.toByteArray())
        ).take(16) + "..."
    }
}

// OWASP A03: Injection - use parameterized queries
@org.springframework.stereotype.Repository
class SafeProductRepository(
    private val jdbcTemplate: org.springframework.jdbc.core.JdbcTemplate
) {
    
    // SAFE: parameterized query
    fun findByName(name: String): List<ProductEntity> {
        return jdbcTemplate.query(
            "SELECT * FROM products WHERE name = ?",
            { rs, _ -> mapRowToEntity(rs) },
            name
        )
    }
    
    // UNSAFE (never do this):
    // "SELECT * FROM products WHERE name = '$name'"  ← SQL injection!
    
    // SAFE: Spring Data JPA with @Param
    @org.springframework.data.jpa.repository.Query(
        "SELECT p FROM ProductEntity p WHERE p.name LIKE %:name%"
    )
    fun searchByName(@org.springframework.data.repository.query.Param("name") name: String): List<ProductEntity>
        = TODO()
    
    private fun mapRowToEntity(rs: java.sql.ResultSet): ProductEntity = TODO()
}

// OWASP A05: Security Misconfiguration
@org.springframework.context.annotation.Configuration
class SecurityHeadersConfig {
    
    @org.springframework.context.annotation.Bean
    fun securityFilterChain(http: org.springframework.security.config.annotation.web.builders.HttpSecurity): org.springframework.security.web.SecurityFilterChain {
        return http
            .headers { headers ->
                headers
                    .contentSecurityPolicy { csp ->
                        csp.policyDirectives("default-src 'self'; script-src 'self'; style-src 'self'")
                    }
                    .referrerPolicy { ref ->
                        ref.policy(org.springframework.security.web.header.writers.ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN)
                    }
                    .permissionsPolicy { perm ->
                        perm.policy("camera=(), microphone=(), location=()")
                    }
                    .frameOptions { it.deny() }
            }
            .csrf { csrf ->
                csrf.csrfTokenRepository(
                    org.springframework.security.web.csrf.CookieCsrfTokenRepository.withHttpOnlyFalse()
                )
            }
            .sessionManagement { session ->
                session.sessionCreationPolicy(
                    org.springframework.security.config.http.SessionCreationPolicy.STATELESS
                )
            }
            .build()
    }
}

interface NotFoundException : RuntimeException
interface ResourceRepository {
    fun findById(id: String): Resource?
}
data class Resource(val id: String, val ownerId: String)
interface NotFoundException {
    fun NotFoundException(msg: String): RuntimeException
}
class NotFoundException(msg: String) : RuntimeException(msg)
interface ResourceRepository { fun findById(id: String): Resource? }
```

---

## Security Testing

```kotlin
// SAST + DAST setup
// build.gradle.kts:
// id("io.gitlab.arturbosch.detekt") version "1.23.4"

// detekt.yml rules for security:
/*
detekt:
  rules:
    style:
      ForbiddenComment:
        active: true
    security:
      HardCodedCredentials:
        active: true
      UnsafeString:
        active: true
*/

// Security unit tests
class SecurityTest {
    
    @org.junit.jupiter.api.Test
    fun `password must be hashed`() {
        val service = CryptographyService()
        val hash = service.hashPassword("mypassword123")
        
        // Verify hash ≠ plaintext
        org.assertj.core.api.Assertions.assertThat(hash).isNotEqualTo("mypassword123")
        
        // Verify it's bcrypt
        org.assertj.core.api.Assertions.assertThat(hash).startsWith("\$2a\$")
        
        // Verify correct password matches
        org.assertj.core.api.Assertions.assertThat(service.verifyPassword("mypassword123", hash)).isTrue
        
        // Verify wrong password doesn't match
        org.assertj.core.api.Assertions.assertThat(service.verifyPassword("wrongpassword", hash)).isFalse
    }
    
    @org.junit.jupiter.api.Test
    fun `JWT validation rejects tampered tokens`() {
        val service = JwtValidationService(JwtConfig())
        val validToken = "eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyLTEifQ."
        
        // Tamper the payload
        val tamperedToken = validToken.replace("dXNlci0x", "dXNlci0y")  // change userId
        
        org.assertj.core.api.Assertions.assertThatThrownBy {
            service.validate(tamperedToken)
        }.isInstanceOf(SecurityException::class.java)
    }
    
    @org.junit.jupiter.api.Test
    fun `path traversal is prevented`() {
        val sanitizer = InputSanitizer()
        
        org.assertj.core.api.Assertions.assertThatThrownBy {
            sanitizer.validateFilePath("../../etc/passwd", "/uploads")
        }.isInstanceOf(SecurityException::class.java)
        
        // Valid path should work
        val valid = sanitizer.validateFilePath("images/product.jpg", "/uploads")
        org.assertj.core.api.Assertions.assertThat(valid.path).startsWith("/uploads")
    }
    
    @org.junit.jupiter.api.Test
    fun `SQL injection patterns are rejected`() {
        val sanitizer = InputSanitizer()
        
        val injections = listOf(
            "'; DROP TABLE products;--",
            "1 OR 1=1",
            "admin'--",
            "<script>alert('xss')</script>"
        )
        
        injections.forEach { injection ->
            org.assertj.core.api.Assertions.assertThatThrownBy {
                sanitizer.validateProductName(injection)
            }.withFailMessage("Should reject: $injection")
                .isInstanceOf(IllegalArgumentException::class.java)
        }
    }
    
    @org.junit.jupiter.api.Test
    fun `SSRF is prevented for internal hosts`() {
        val sanitizer = InputSanitizer()
        
        val internalUrls = listOf(
            "http://localhost/admin",
            "http://127.0.0.1/secrets",
            "http://169.254.169.254/latest/meta-data",
            "http://192.168.1.1/router",
            "file:///etc/passwd"
        )
        
        internalUrls.forEach { url ->
            org.assertj.core.api.Assertions.assertThatThrownBy {
                sanitizer.validateUrl(url)
            }.withFailMessage("Should reject: $url")
        }
    }
    
    @org.junit.jupiter.api.Test
    fun `encryption is reversible and secure`() {
        val config = EncryptionConfig(
            java.util.Base64.getEncoder().encodeToString(ByteArray(32).also {
                java.security.SecureRandom().nextBytes(it)
            })
        )
        val service = FieldEncryptionService(config)
        
        val plaintext = "sensitive-data@example.com"
        val encrypted = service.encrypt(plaintext)
        
        // Encrypted ≠ plaintext
        org.assertj.core.api.Assertions.assertThat(encrypted).isNotEqualTo(plaintext)
        
        // Decryption restores plaintext
        org.assertj.core.api.Assertions.assertThat(service.decrypt(encrypted)).isEqualTo(plaintext)
        
        // Each encryption produces different ciphertext (IV randomness)
        val encrypted2 = service.encrypt(plaintext)
        org.assertj.core.api.Assertions.assertThat(encrypted).isNotEqualTo(encrypted2)
        
        // But both decrypt to same value
        org.assertj.core.api.Assertions.assertThat(service.decrypt(encrypted2)).isEqualTo(plaintext)
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง Multi-Factor Authentication (MFA) System

@org.springframework.stereotype.Service
class MfaService(
    private val userRepository: UserRepository,
    private val auditService: SecurityAuditService,
    private val redisTemplate: org.springframework.data.redis.core.RedisTemplate<String, String>
) {
    
    // Generate TOTP secret for user
    fun setupMfa(userId: String): MfaSetupResponse {
        val secret = dev.samstevens.totp.secret.DefaultSecretGenerator().generate()
        val qrCode = generateQrCode(userId, secret)
        
        // Store temporarily until confirmed
        redisTemplate.opsForValue().set(
            "mfa:pending:$userId",
            secret,
            java.time.Duration.ofMinutes(10)
        )
        
        return MfaSetupResponse(secret = secret, qrCodeUrl = qrCode)
    }
    
    // Confirm MFA setup with valid code
    fun confirmMfa(userId: String, code: String): Boolean {
        val secret = redisTemplate.opsForValue().get("mfa:pending:$userId")
            ?: return false
        
        if (!verifyCode(secret, code)) return false
        
        userRepository.enableMfa(userId, secret)
        redisTemplate.delete("mfa:pending:$userId")
        
        auditService.logAuthEvent(userId, AuthEventType.MFA_SETUP, "", "", true)
        return true
    }
    
    // Verify TOTP code
    fun verify(userId: String, code: String): Boolean {
        val user = userRepository.findById(userId) ?: return false
        val secret = user.mfaSecret ?: return false
        
        return verifyCode(secret, code)
    }
    
    private fun verifyCode(secret: String, code: String): Boolean {
        val totp = dev.samstevens.totp.code.DefaultCodeGenerator()
        val verifier = dev.samstevens.totp.code.DefaultCodeVerifier(totp, dev.samstevens.totp.time.SystemTimeProvider())
        return verifier.isValidCode(secret, code)
    }
    
    private fun generateQrCode(userId: String, secret: String): String {
        val uri = dev.samstevens.totp.qr.QrData.Builder()
            .label(userId)
            .secret(secret)
            .issuer("EcommerceApp")
            .build()
            .getUri()
        return "https://quickchart.io/qr?text=${java.net.URLEncoder.encode(uri, "UTF-8")}"
    }
}

data class MfaSetupResponse(val secret: String, val qrCodeUrl: String)

interface UserRepository {
    fun findById(id: String): UserWithMfa?
    fun enableMfa(userId: String, secret: String)
}

data class UserWithMfa(val id: String, val email: String, val mfaEnabled: Boolean, val mfaSecret: String?)
```

---

## สรุป Part 94

```
✅ Zero Trust: never trust, always verify
✅ JWT full validation: expiry, signature, issuer, audience, revocation
✅ Token rotation: revoke old refresh token on use
✅ Sliding window rate limit: sorted set in Redis
✅ Token bucket: burst-friendly rate limiting with Lua
✅ Input validation: regex, length, character whitelist
✅ XSS prevention: OWASP Java HTML Sanitizer
✅ Path traversal: canonicalPath.startsWith(base)
✅ SSRF prevention: block loopback, private IPs, metadata endpoint
✅ Audit logging: structured events with severity
✅ Anomaly detection: multiple IPs, failed logins
✅ Spring Cloud Vault: Kubernetes auth, dynamic DB creds
✅ SecretsRotationService: rotate credentials without restart
✅ AES-GCM: authenticated encryption with IV prepended
✅ OWASP A01: ownership check, tenant isolation
✅ OWASP A02: BCrypt(12), SecureRandom, hash PII for logs
✅ OWASP A03: parameterized queries, @Param JPA
✅ OWASP A05: CSP, referrer-policy, frame-options headers
✅ CSRF: CookieCsrfTokenRepository
✅ Security tests: tampered JWT, path traversal, injection, SSRF
✅ Encryption tests: verify roundtrip + random IV per encryption
✅ TOTP MFA: DefaultCodeGenerator, DefaultCodeVerifier
✅ MFA setup: pending secret in Redis, confirm with valid code
✅ detekt security rules: HardCodedCredentials, UnsafeString
```

---

*Part 94/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
