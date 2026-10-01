# Part 61: Spring Security ขั้นสูง

## สารบัญ
1. [Method Security](#method-security)
2. [RBAC ด้วย Database Roles](#rbac-ด้วย-database-roles)
3. [Custom Security Expressions](#custom-security-expressions)
4. [CORS และ CSRF](#cors-และ-csrf)
5. [Content Security Policy](#content-security-policy)
6. [Rate Limiting](#rate-limiting)
7. [Security Audit Logging](#security-audit-logging)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Method Security

```kotlin
// เปิดใช้งาน Method Security
@Configuration
@EnableMethodSecurity(
    prePostEnabled = true,   // @PreAuthorize, @PostAuthorize
    securedEnabled = true,   // @Secured
    jsr250Enabled = true     // @RolesAllowed
)
class MethodSecurityConfig

// ตัวอย่างการใช้งาน
@Service
class ProductService(private val productRepository: ProductRepository) {
    
    // ต้องมี role ADMIN หรือ PRODUCT_MANAGER
    @PreAuthorize("hasAnyRole('ADMIN', 'PRODUCT_MANAGER')")
    fun createProduct(request: CreateProductRequest): Product {
        return productRepository.save(request.toProduct())
    }
    
    // ต้องมี permission เฉพาะ
    @PreAuthorize("hasAuthority('PRODUCT:WRITE')")
    fun updateProduct(id: String, request: UpdateProductRequest): Product {
        val product = productRepository.findById(id)
            ?: throw NotFoundException("Product $id not found")
        return productRepository.save(product.update(request))
    }
    
    // ตรวจสอบ method arguments
    @PreAuthorize("#userId == authentication.principal.id or hasRole('ADMIN')")
    fun getUserProducts(userId: String): List<Product> {
        return productRepository.findByUserId(userId)
    }
    
    // ตรวจสอบ return value
    @PostAuthorize("returnObject.ownerId == authentication.principal.id or hasRole('ADMIN')")
    fun getProductById(id: String): Product {
        return productRepository.findById(id)
            ?: throw NotFoundException("Product $id not found")
    }
    
    // Filter collection — เอาเฉพาะของ user นั้น
    @PostFilter("filterObject.ownerId == authentication.principal.id or hasRole('ADMIN')")
    fun getAllProducts(): List<Product> {
        return productRepository.findAll()
    }
    
    // Filter input collection
    @PreFilter("filterObject.ownerId == authentication.principal.id")
    fun deleteProducts(products: MutableList<Product>): Int {
        productRepository.deleteAll(products)
        return products.size
    }
    
    // SPEL with complex expression
    @PreAuthorize("""
        hasRole('ADMIN') or 
        (hasRole('PRODUCT_MANAGER') and #product.categoryId == authentication.principal.managedCategoryId)
    """)
    fun publishProduct(product: Product): Product {
        return productRepository.save(product.copy(status = ProductStatus.PUBLISHED))
    }
}

// placeholder interfaces
interface ProductRepository {
    fun findById(id: String): Product?
    fun save(product: Product): Product
    fun findByUserId(userId: String): List<Product>
    fun findAll(): List<Product>
    fun deleteAll(products: List<Product>)
}

data class Product(val id: String, val name: String, val ownerId: String, val categoryId: String, val status: ProductStatus)
data class CreateProductRequest(val name: String) { fun toProduct() = Product("", name, "", "", ProductStatus.DRAFT) }
data class UpdateProductRequest(val name: String)
fun Product.update(req: UpdateProductRequest) = copy(name = req.name)
enum class ProductStatus { DRAFT, PUBLISHED, DISCONTINUED }
class NotFoundException(msg: String) : Exception(msg)
```

---

## RBAC ด้วย Database Roles

```kotlin
// Entity สำหรับ User, Role, Permission
@Entity
@Table(name = "users")
data class UserEntity(
    @Id val id: String = java.util.UUID.randomUUID().toString(),
    val username: String,
    val email: String,
    val passwordHash: String,
    val enabled: Boolean = true,
    
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "user_roles",
        joinColumns = [JoinColumn(name = "user_id")],
        inverseJoinColumns = [JoinColumn(name = "role_id")]
    )
    val roles: Set<RoleEntity> = emptySet()
)

@Entity
@Table(name = "roles")
data class RoleEntity(
    @Id val id: String = java.util.UUID.randomUUID().toString(),
    val name: String,  // ADMIN, USER, PRODUCT_MANAGER
    val description: String,
    
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "role_permissions",
        joinColumns = [JoinColumn(name = "role_id")],
        inverseJoinColumns = [JoinColumn(name = "permission_id")]
    )
    val permissions: Set<PermissionEntity> = emptySet()
)

@Entity
@Table(name = "permissions")
data class PermissionEntity(
    @Id val id: String = java.util.UUID.randomUUID().toString(),
    val resource: String,    // PRODUCT, ORDER, USER
    val action: String       // READ, WRITE, DELETE, PUBLISH
)

// UserDetails implementation
data class UserPrincipal(
    val userId: String,
    val email: String,
    val managedCategoryId: String?,
    private val authorities: Collection<GrantedAuthority>
) : UserDetails {
    override fun getAuthorities() = authorities
    override fun getPassword() = null  // password not needed after auth
    override fun getUsername() = email
    override fun isAccountNonExpired() = true
    override fun isAccountNonLocked() = true
    override fun isCredentialsNonExpired() = true
    override fun isEnabled() = true
    
    val id get() = userId
}

@Service
class CustomUserDetailsService(
    private val userRepository: UserJpaRepository
) : UserDetailsService {
    
    override fun loadUserByUsername(username: String): UserDetails {
        val user = userRepository.findByEmail(username)
            ?: throw UsernameNotFoundException("User not found: $username")
        
        val authorities = buildAuthorities(user)
        
        return UserPrincipal(
            userId = user.id,
            email = user.email,
            managedCategoryId = null,  // load from profile if needed
            authorities = authorities
        )
    }
    
    private fun buildAuthorities(user: UserEntity): Collection<GrantedAuthority> {
        val authorities = mutableListOf<GrantedAuthority>()
        
        user.roles.forEach { role ->
            // Add role itself
            authorities.add(SimpleGrantedAuthority("ROLE_${role.name}"))
            
            // Add permissions from role
            role.permissions.forEach { permission ->
                authorities.add(SimpleGrantedAuthority("${permission.resource}:${permission.action}"))
            }
        }
        
        return authorities
    }
}

interface UserJpaRepository : JpaRepository<UserEntity, String> {
    fun findByEmail(email: String): UserEntity?
}

// Security Config with custom user details
@Configuration
@EnableWebSecurity
class SecurityConfig(
    private val userDetailsService: CustomUserDetailsService,
    private val jwtAuthFilter: JwtAuthenticationFilter
) {
    
    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        return http
            .csrf { it.disable() }
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
            .authorizeHttpRequests { auth ->
                auth
                    .requestMatchers("/api/auth/**").permitAll()
                    .requestMatchers("/api/admin/**").hasRole("ADMIN")
                    .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                    .requestMatchers(HttpMethod.POST, "/api/products/**").hasAuthority("PRODUCT:WRITE")
                    .requestMatchers(HttpMethod.DELETE, "/api/products/**").hasAuthority("PRODUCT:DELETE")
                    .anyRequest().authenticated()
            }
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter::class.java)
            .build()
    }
    
    @Bean
    fun passwordEncoder(): PasswordEncoder = BCryptPasswordEncoder(12)
    
    @Bean
    fun authenticationManager(config: AuthenticationConfiguration): AuthenticationManager =
        config.authenticationManager
}

// Needed imports (type aliases for brevity)
typealias JpaRepository<T, ID> = org.springframework.data.jpa.repository.JpaRepository<T, ID>
typealias JoinColumn = jakarta.persistence.JoinColumn
typealias JoinTable = jakarta.persistence.JoinTable
typealias ManyToMany = jakarta.persistence.ManyToMany
typealias FetchType = jakarta.persistence.FetchType
typealias GrantedAuthority = org.springframework.security.core.GrantedAuthority
typealias UserDetails = org.springframework.security.core.userdetails.UserDetails
typealias UsernameNotFoundException = org.springframework.security.core.userdetails.UsernameNotFoundException
typealias SimpleGrantedAuthority = org.springframework.security.core.authority.SimpleGrantedAuthority
typealias UserDetailsService = org.springframework.security.core.userdetails.UserDetailsService
typealias PasswordEncoder = org.springframework.security.crypto.password.PasswordEncoder
typealias BCryptPasswordEncoder = org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder
typealias AuthenticationManager = org.springframework.security.authentication.AuthenticationManager
typealias AuthenticationConfiguration = org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration
typealias HttpSecurity = org.springframework.security.config.annotation.web.builders.HttpSecurity
typealias SecurityFilterChain = org.springframework.security.web.SecurityFilterChain
typealias HttpMethod = org.springframework.http.HttpMethod
typealias UsernamePasswordAuthenticationFilter = org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter
typealias SessionCreationPolicy = org.springframework.security.config.http.SessionCreationPolicy
typealias JwtAuthenticationFilter = javax.servlet.Filter
```

---

## Custom Security Expressions

```kotlin
// Custom Security Expression Root
class CustomSecurityExpressionRoot(
    authentication: Authentication,
    private val orderService: OrderQueryService,
    private val productService: ProductQueryService
) : SecurityExpressionRoot(authentication), MethodSecurityExpressionOperations {
    
    // ตรวจสอบ order ownership
    fun isOrderOwner(orderId: String): Boolean {
        val userId = (authentication.principal as UserPrincipal).userId
        return orderService.isOwner(orderId, userId)
    }
    
    // ตรวจสอบ subscription tier
    fun hasSubscription(tier: String): Boolean {
        val user = authentication.principal as UserPrincipal
        return userSubscriptionService.hasTier(user.userId, tier)
    }
    
    // ตรวจสอบว่าเป็นเวลา business hours
    fun isDuringBusinessHours(): Boolean {
        val hour = java.time.LocalTime.now().hour
        return hour in 8..17
    }
    
    // ตรวจสอบ IP whitelist
    fun isFromAllowedIp(): Boolean {
        val ip = getClientIp()
        return allowedIpRanges.any { range -> range.contains(ip) }
    }
    
    private val allowedIpRanges = listOf("192.168.1.")
    private val userSubscriptionService = object {
        fun hasTier(userId: String, tier: String) = true
    }
    
    private fun getClientIp(): String = "192.168.1.1"
    
    override fun setFilterObject(filterObject: Any?) {}
    override fun getFilterObject(): Any? = null
    override fun setReturnObject(returnObject: Any?) {}
    override fun getReturnObject(): Any? = null
    override fun getThis(): Any = this
}

// Register custom expression handler
@Bean
fun methodSecurityExpressionHandler(
    orderService: OrderQueryService,
    productService: ProductQueryService
): MethodSecurityExpressionHandler {
    return object : DefaultMethodSecurityExpressionHandler() {
        override fun createSecurityExpressionRoot(
            authentication: Authentication,
            mi: MethodInvocation
        ): SecurityExpressionRoot {
            return CustomSecurityExpressionRoot(authentication, orderService, productService).also { root ->
                root.setPermissionEvaluator(defaultPermissionEvaluator)
                root.setTrustResolver(trustResolver)
                root.setRoleHierarchy(roleHierarchy)
            }
        }
    }
}

// ใช้งาน custom expression
@Service
class OrderService(val orderRepo: OrderRepository) {
    
    @PreAuthorize("isOrderOwner(#orderId) or hasRole('ADMIN')")
    fun cancelOrder(orderId: String): Order {
        return orderRepo.findById(orderId)?.let { order ->
            orderRepo.save(order.copy(status = "CANCELLED"))
        } ?: throw NotFoundException("Order $orderId not found")
    }
    
    @PreAuthorize("hasSubscription('PREMIUM') or hasRole('ADMIN')")
    fun exportOrderHistory(userId: String): ByteArray {
        // Only PREMIUM users can export
        TODO("Export CSV/Excel")
    }
    
    @PreAuthorize("isDuringBusinessHours() or hasRole('ADMIN')")
    fun processRefund(orderId: String): Order {
        TODO("Process refund during business hours only")
    }
}

data class Order(val id: String, val status: String)
interface OrderRepository { fun findById(id: String): Order?; fun save(o: Order): Order }
interface OrderQueryService { fun isOwner(orderId: String, userId: String): Boolean }
interface ProductQueryService

typealias Authentication = org.springframework.security.core.Authentication
typealias SecurityExpressionRoot = org.springframework.security.access.expression.SecurityExpressionRoot
typealias MethodSecurityExpressionOperations = org.springframework.security.access.expression.method.MethodSecurityExpressionOperations
typealias DefaultMethodSecurityExpressionHandler = org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler
typealias MethodSecurityExpressionHandler = org.springframework.security.access.expression.method.MethodSecurityExpressionHandler
typealias MethodInvocation = org.aopalliance.intercept.MethodInvocation
```

---

## CORS และ CSRF

```kotlin
@Configuration
class CorsSecurityConfig {
    
    @Bean
    fun corsConfigurationSource(): CorsConfigurationSource {
        val config = CorsConfiguration().apply {
            // Allowed origins — ไม่ใช้ * ใน production
            allowedOrigins = listOf(
                "https://app.mycompany.com",
                "https://admin.mycompany.com"
            )
            // ใช้ pattern สำหรับ subdomain wildcards
            allowedOriginPatterns = listOf("https://*.mycompany.com")
            
            allowedMethods = listOf("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
            allowedHeaders = listOf(
                "Authorization",
                "Content-Type",
                "X-Requested-With",
                "Accept",
                "X-XSRF-TOKEN"
            )
            exposedHeaders = listOf(
                "X-Auth-Token",
                "X-Total-Count",
                "Link"
            )
            allowCredentials = true
            maxAge = 3600L  // Cache preflight response for 1 hour
        }
        
        return UrlBasedCorsConfigurationSource().apply {
            registerCorsConfiguration("/api/**", config)
            registerCorsConfiguration("/ws/**", CorsConfiguration().apply {
                allowedOrigins = listOf("https://app.mycompany.com")
                allowCredentials = true
            })
        }
    }
    
    // CSRF configuration for SPA
    @Bean
    fun securityFilterChainWithCsrf(http: HttpSecurity): SecurityFilterChain {
        return http
            .cors { it.configurationSource(corsConfigurationSource()) }
            .csrf { csrf ->
                // สำหรับ REST API ที่ใช้ JWT: disable CSRF
                csrf.disable()
                
                // สำหรับ web app ที่ใช้ session + cookie:
                // csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                //     .csrfTokenRequestHandler(SpaCsrfTokenRequestHandler())
            }
            .authorizeHttpRequests { it.anyRequest().authenticated() }
            .build()
    }
}

// Custom CSRF handler สำหรับ SPA (ถ้าใช้ session-based auth)
class SpaCsrfTokenRequestHandler : CsrfTokenRequestAttributeHandler() {
    private val delegate = XorCsrfTokenRequestAttributeHandler()
    
    override fun handle(request: HttpServletRequest, response: HttpServletResponse, csrfToken: Supplier<CsrfToken>) {
        delegate.handle(request, response, csrfToken)
    }
    
    override fun resolveCsrfTokenValue(request: HttpServletRequest, csrfToken: CsrfToken): String? {
        return if (request.getHeader(csrfToken.headerName) != null) {
            super.resolveCsrfTokenValue(request, csrfToken)
        } else {
            delegate.resolveCsrfTokenValue(request, csrfToken)
        }
    }
}

// placeholder types for CSRF
typealias CsrfTokenRequestAttributeHandler = org.springframework.security.web.csrf.CsrfTokenRequestAttributeHandler
typealias XorCsrfTokenRequestAttributeHandler = org.springframework.security.web.csrf.XorCsrfTokenRequestAttributeHandler
typealias CsrfToken = org.springframework.security.web.csrf.CsrfToken
typealias HttpServletRequest = jakarta.servlet.http.HttpServletRequest
typealias HttpServletResponse = jakarta.servlet.http.HttpServletResponse
typealias Supplier<T> = java.util.function.Supplier<T>
```

---

## Content Security Policy

```kotlin
@Configuration
class SecurityHeadersConfig {
    
    @Bean
    fun securityFilterChainWithHeaders(http: HttpSecurity): SecurityFilterChain {
        return http
            .headers { headers ->
                headers
                    // Strict-Transport-Security: force HTTPS
                    .httpStrictTransportSecurity { hsts ->
                        hsts
                            .includeSubDomains(true)
                            .maxAgeInSeconds(31536000)  // 1 year
                            .preload(true)
                    }
                    
                    // Content-Security-Policy
                    .contentSecurityPolicy { csp ->
                        csp.policyDirectives("""
                            default-src 'self';
                            script-src 'self' 'nonce-{random}' https://cdn.jsdelivr.net;
                            style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
                            img-src 'self' data: https:;
                            font-src 'self' https://fonts.gstatic.com;
                            connect-src 'self' https://api.mycompany.com wss://ws.mycompany.com;
                            frame-ancestors 'none';
                            form-action 'self';
                            base-uri 'self';
                            object-src 'none'
                        """.trimIndent())
                    }
                    
                    // X-Frame-Options
                    .frameOptions { it.deny() }
                    
                    // X-Content-Type-Options
                    .contentTypeOptions {}  // defaults to nosniff
                    
                    // Referrer-Policy
                    .referrerPolicy { it.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN) }
                    
                    // Permissions-Policy
                    .permissionsPolicy { policy ->
                        policy.policy("camera=(), microphone=(), geolocation=(self), payment=()")
                    }
            }
            .authorizeHttpRequests { it.anyRequest().authenticated() }
            .build()
    }
}

typealias ReferrerPolicyHeaderWriter = org.springframework.security.web.header.writers.ReferrerPolicyHeaderWriter
```

---

## Rate Limiting

```kotlin
// Rate Limiting with Bucket4j + Redis
@Component
class RateLimitingFilter(
    private val redisTemplate: RedisTemplate<String, String>
) : OncePerRequestFilter() {
    
    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val key = getRateLimitKey(request)
        val limit = getRateLimit(request)
        
        val count = redisTemplate.opsForValue().increment(key) ?: 1
        
        // Set expiry on first request
        if (count == 1L) {
            redisTemplate.expire(key, java.time.Duration.ofMinutes(1))
        }
        
        // Set rate limit headers
        response.setHeader("X-RateLimit-Limit", limit.toString())
        response.setHeader("X-RateLimit-Remaining", maxOf(0, limit - count).toString())
        response.setHeader("X-RateLimit-Reset", 
            System.currentTimeMillis().div(1000).plus(60).toString())
        
        if (count > limit) {
            response.status = 429
            response.contentType = "application/json"
            response.writer.write("""{"error": "Too many requests", "retryAfter": 60}""")
            return
        }
        
        filterChain.doFilter(request, response)
    }
    
    private fun getRateLimitKey(request: HttpServletRequest): String {
        val authentication = org.springframework.security.core.context.SecurityContextHolder
            .getContext().authentication
        
        return if (authentication?.isAuthenticated == true) {
            "ratelimit:user:${authentication.name}:${getEndpointKey(request)}"
        } else {
            "ratelimit:ip:${getClientIp(request)}:${getEndpointKey(request)}"
        }
    }
    
    private fun getRateLimit(request: HttpServletRequest): Long {
        val authentication = org.springframework.security.core.context.SecurityContextHolder
            .getContext().authentication
        
        return when {
            authentication == null || !authentication.isAuthenticated -> 10L   // 10/min for anonymous
            authentication.authorities.any { it.authority == "ROLE_PREMIUM" } -> 1000L
            authentication.authorities.any { it.authority == "ROLE_USER" } -> 100L
            else -> 50L
        }
    }
    
    private fun getEndpointKey(request: HttpServletRequest): String {
        return "${request.method}:${request.requestURI}"
    }
    
    private fun getClientIp(request: HttpServletRequest): String {
        return request.getHeader("X-Forwarded-For")?.split(",")?.first()?.trim()
            ?: request.remoteAddr
    }
}

@Configuration
class RateLimitConfig {
    
    @Bean
    fun rateLimitingFilter(): FilterRegistrationBean<RateLimitingFilter> {
        return FilterRegistrationBean(rateLimitingFilterInstance).apply {
            addUrlPatterns("/api/*")
            order = 1
        }
    }
    
    // In real use, inject the filter bean
    private val rateLimitingFilterInstance get() = throw UnsupportedOperationException()
}

typealias OncePerRequestFilter = org.springframework.web.filter.OncePerRequestFilter
typealias FilterChain = jakarta.servlet.FilterChain
typealias FilterRegistrationBean<T> = org.springframework.boot.web.servlet.FilterRegistrationBean<T>
typealias RedisTemplate<K, V> = org.springframework.data.redis.core.RedisTemplate<K, V>
```

---

## Security Audit Logging

```kotlin
// Audit events for security monitoring
@Component
class SecurityAuditLogger(
    private val auditLogRepository: AuditLogRepository
) : ApplicationListener<AbstractAuthenticationEvent> {
    
    override fun onApplicationEvent(event: AbstractAuthenticationEvent) {
        val logEntry = when (event) {
            is AuthenticationSuccessEvent -> AuditLog(
                event = "LOGIN_SUCCESS",
                username = event.authentication.name,
                details = "Login successful",
                severity = AuditSeverity.INFO
            )
            is AuthenticationFailureBadCredentialsEvent -> AuditLog(
                event = "LOGIN_FAILURE",
                username = event.authentication.name,
                details = "Bad credentials",
                severity = AuditSeverity.WARNING
            )
            is AuthenticationFailureLockedEvent -> AuditLog(
                event = "ACCOUNT_LOCKED",
                username = event.authentication.name,
                details = "Account locked",
                severity = AuditSeverity.ERROR
            )
            else -> return
        }
        
        auditLogRepository.save(logEntry)
    }
}

data class AuditLog(
    val id: String = java.util.UUID.randomUUID().toString(),
    val event: String,
    val username: String,
    val details: String,
    val severity: AuditSeverity,
    val timestamp: java.time.Instant = java.time.Instant.now(),
    val ipAddress: String? = null
)

enum class AuditSeverity { INFO, WARNING, ERROR, CRITICAL }

interface AuditLogRepository {
    fun save(log: AuditLog): AuditLog
}

// Method-level audit logging with AOP
@Component
@Aspect
class SecurityAuditAspect(private val auditLogRepository: AuditLogRepository) {
    
    @Around("@annotation(com.example.security.Audited)")
    fun auditMethodCall(joinPoint: ProceedingJoinPoint): Any? {
        val authentication = org.springframework.security.core.context.SecurityContextHolder
            .getContext().authentication
        
        val methodName = "${joinPoint.signature.declaringTypeName}.${joinPoint.signature.name}"
        
        return try {
            val result = joinPoint.proceed()
            auditLogRepository.save(AuditLog(
                event = "METHOD_ACCESS",
                username = authentication?.name ?: "anonymous",
                details = "Called $methodName",
                severity = AuditSeverity.INFO
            ))
            result
        } catch (ex: Exception) {
            auditLogRepository.save(AuditLog(
                event = "METHOD_ACCESS_DENIED",
                username = authentication?.name ?: "anonymous",
                details = "Access denied to $methodName: ${ex.message}",
                severity = AuditSeverity.WARNING
            ))
            throw ex
        }
    }
}

@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class Audited

typealias ApplicationListener<T> = org.springframework.context.ApplicationListener<T>
typealias AbstractAuthenticationEvent = org.springframework.security.authentication.event.AbstractAuthenticationEvent
typealias AuthenticationSuccessEvent = org.springframework.security.authentication.event.AuthenticationSuccessEvent
typealias AuthenticationFailureBadCredentialsEvent = org.springframework.security.authentication.event.AuthenticationFailureBadCredentialsEvent
typealias AuthenticationFailureLockedEvent = org.springframework.security.authentication.event.AuthenticationFailureLockedEvent
typealias ProceedingJoinPoint = org.aspectj.lang.ProceedingJoinPoint
typealias Aspect = org.aspectj.lang.annotation.Aspect
typealias Around = org.aspectj.lang.annotation.Around
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement IP-based account lockout

// หลังจาก 5 ครั้ง failed login จาก IP เดียวกัน ให้ block IP นั้น 30 นาที

@Component
class LoginAttemptService(
    private val redisTemplate: RedisTemplate<String, String>
) {
    
    private val MAX_ATTEMPTS = 5
    private val BLOCK_DURATION_MINUTES = 30L
    
    fun loginFailed(ip: String) {
        val key = "login:failed:$ip"
        val count = redisTemplate.opsForValue().increment(key) ?: 1L
        
        if (count == 1L) {
            redisTemplate.expire(key, java.time.Duration.ofMinutes(BLOCK_DURATION_MINUTES))
        }
        
        if (count >= MAX_ATTEMPTS) {
            blockIp(ip)
        }
    }
    
    fun loginSucceeded(ip: String) {
        redisTemplate.delete("login:failed:$ip")
    }
    
    fun isBlocked(ip: String): Boolean {
        return redisTemplate.hasKey("login:blocked:$ip") == true
    }
    
    private fun blockIp(ip: String) {
        val key = "login:blocked:$ip"
        redisTemplate.opsForValue().set(key, "blocked")
        redisTemplate.expire(key, java.time.Duration.ofMinutes(BLOCK_DURATION_MINUTES))
    }
}

// TODO: Wire LoginAttemptService into CustomAuthenticationProvider
// 1. Check isBlocked() before validating credentials
// 2. Call loginFailed() when credentials are wrong
// 3. Call loginSucceeded() when login succeeds
// 4. Return 429 with Retry-After header when blocked
```

---

## สรุป Part 61

```
✅ @EnableMethodSecurity: เปิดใช้ method-level security
✅ @PreAuthorize: ตรวจสอบ permission ก่อนเรียก method
✅ @PostAuthorize: ตรวจสอบ return value
✅ @PreFilter/@PostFilter: กรอง collection arguments/results
✅ SpEL: #argument, returnObject, authentication.principal
✅ RBAC: User → Roles → Permissions hierarchy
✅ UserDetails: load user + roles from database
✅ Custom SecurityExpressionRoot: custom SPEL functions
✅ CORS: CorsConfigurationSource, allowedOrigins, credentials
✅ CSRF: disable for JWT, CookieCsrfTokenRepository for sessions
✅ CSP: Content-Security-Policy header directives
✅ HSTS: Strict-Transport-Security, includeSubDomains
✅ Rate Limiting: Redis counter per user/IP, X-RateLimit headers
✅ Audit Logging: ApplicationListener, @Audited AOP aspect
✅ IP Lockout: block after N failed attempts with Redis TTL
```

---

*Part 61/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
