# Part 70: Multi-tenancy Architecture

## สารบัญ
1. [Multi-tenancy คืออะไร](#multi-tenancy-คืออะไร)
2. [Tenant Context Propagation](#tenant-context-propagation)
3. [Schema-per-Tenant](#schema-per-tenant)
4. [Row-Level Security](#row-level-security)
5. [Tenant-aware Caching](#tenant-aware-caching)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Multi-tenancy คืออะไร

```
Multi-tenancy คือ pattern ที่ให้ application เดียวรองรับหลาย organizations (tenants)

3 รูปแบบหลัก:

1. Database-per-Tenant (Database Isolation)
   - แต่ละ tenant มี database แยกกัน
   - ✅ Isolation สูงสุด
   - ❌ Cost สูง, ยาก scale

2. Schema-per-Tenant (Schema Isolation)
   - แต่ละ tenant มี schema ใน database เดียวกัน
   - ✅ Isolation ดี, cost ต่ำกว่า
   - ❌ Connection pool ซับซ้อน

3. Shared Schema (Row-Level Security)
   - ทุก tenant ใช้ table เดียวกัน มีคอลัมน์ tenant_id
   - ✅ Cost ถูก, ง่าย manage
   - ❌ Isolation ต่ำกว่า, risk data leak

เลือกตามความต้องการ:
- Healthcare/Finance → Database หรือ Schema (compliance)
- SaaS ทั่วไป → Row-Level Security เพียงพอ
```

---

## Tenant Context Propagation

```kotlin
// Tenant resolver - ดึง tenant ID จาก request
interface TenantResolver {
    fun resolve(request: jakarta.servlet.http.HttpServletRequest): String?
}

// Resolve from subdomain: acme.myapp.com → "acme"
@Component
class SubdomainTenantResolver : TenantResolver {
    override fun resolve(request: jakarta.servlet.http.HttpServletRequest): String? {
        val host = request.serverName  // acme.myapp.com
        val parts = host.split(".")
        
        // Need at least subdomain.domain.tld
        return if (parts.size >= 3) parts[0] else null
    }
}

// Resolve from header: X-Tenant-ID: acme
@Component
class HeaderTenantResolver : TenantResolver {
    override fun resolve(request: jakarta.servlet.http.HttpServletRequest): String? {
        return request.getHeader("X-Tenant-ID")
    }
}

// Resolve from JWT claim: { "tenant_id": "acme" }
@Component
class JwtTenantResolver(private val jwtUtils: JwtUtils) : TenantResolver {
    override fun resolve(request: jakarta.servlet.http.HttpServletRequest): String? {
        val token = extractToken(request) ?: return null
        return jwtUtils.getClaim(token, "tenant_id")
    }
    
    private fun extractToken(request: jakarta.servlet.http.HttpServletRequest): String? {
        val header = request.getHeader("Authorization") ?: return null
        return if (header.startsWith("Bearer ")) header.substring(7) else null
    }
}

class JwtUtils {
    fun getClaim(token: String, claim: String): String? = null // TODO: real JWT parsing
}

// Tenant context holder - ThreadLocal-based
object TenantContext {
    private val currentTenant = ThreadLocal<String?>()
    
    fun setCurrentTenant(tenant: String?) {
        currentTenant.set(tenant)
    }
    
    fun getCurrentTenant(): String? = currentTenant.get()
    
    fun getCurrentTenantOrThrow(): String {
        return currentTenant.get()
            ?: throw TenantNotSetException("No tenant set in context")
    }
    
    fun clear() {
        currentTenant.remove()
    }
    
    // Convenience block function
    fun <T> withTenant(tenant: String, block: () -> T): T {
        val previous = currentTenant.get()
        return try {
            setCurrentTenant(tenant)
            block()
        } finally {
            if (previous != null) setCurrentTenant(previous) else clear()
        }
    }
}

class TenantNotSetException(message: String) : RuntimeException(message)

// Filter: resolve tenant from each request
@Component
@Order(1)  // Run early in filter chain
class TenantFilter(
    private val resolvers: List<TenantResolver>
) : jakarta.servlet.Filter {
    
    override fun doFilter(
        request: jakarta.servlet.ServletRequest,
        response: jakarta.servlet.ServletResponse,
        chain: jakarta.servlet.FilterChain
    ) {
        val httpRequest = request as jakarta.servlet.http.HttpServletRequest
        
        val tenantId = resolvers.firstNotNullOfOrNull { it.resolve(httpRequest) }
        
        try {
            TenantContext.setCurrentTenant(tenantId)
            chain.doFilter(request, response)
        } finally {
            TenantContext.clear()  // Always clean up
        }
    }
}

typealias Component = org.springframework.stereotype.Component
typealias Order = org.springframework.core.annotation.Order
```

---

## Schema-per-Tenant

```kotlin
// Multi-tenant DataSource using Hibernate AbstractDataSourceBasedMultiTenantConnectionProviderImpl

@Configuration
class MultiTenantHibernateConfig {
    
    @Bean
    fun multiTenantConnectionProvider(
        tenantDataSources: Map<String, javax.sql.DataSource>
    ): MultiTenantConnectionProvider<String> {
        return DataSourceBasedMultiTenantConnectionProviderImpl(tenantDataSources)
    }
    
    @Bean
    fun tenantIdentifierResolver(): CurrentTenantIdentifierResolver<String> {
        return TenantContextIdentifierResolver()
    }
    
    @Bean
    fun entityManagerFactory(
        builder: LocalContainerEntityManagerFactoryBuilder,
        dataSource: javax.sql.DataSource,
        multiTenantConnectionProvider: MultiTenantConnectionProvider<String>,
        tenantIdentifierResolver: CurrentTenantIdentifierResolver<String>
    ): LocalContainerEntityManagerFactoryBean {
        return builder
            .dataSource(dataSource)
            .packages("com.example.domain")
            .properties(
                mapOf(
                    "hibernate.multiTenancy" to "SCHEMA",
                    "hibernate.multi_tenant_connection_provider" to multiTenantConnectionProvider,
                    "hibernate.tenant_identifier_resolver" to tenantIdentifierResolver
                )
            )
            .build()
    }
}

class TenantContextIdentifierResolver :
    org.hibernate.context.spi.CurrentTenantIdentifierResolver<String> {
    
    override fun resolveCurrentTenantIdentifier(): String {
        return TenantContext.getCurrentTenant() ?: "public"
    }
    
    override fun validateExistingCurrentSessions(): Boolean = true
}

class DataSourceBasedMultiTenantConnectionProviderImpl(
    private val tenantDataSources: Map<String, javax.sql.DataSource>
) : org.hibernate.engine.jdbc.connections.spi.AbstractDataSourceBasedMultiTenantConnectionProviderImpl<String>() {
    
    private val defaultDataSource = tenantDataSources["default"]
        ?: error("No default data source configured")
    
    override fun selectAnyDataSource() = defaultDataSource
    
    override fun selectDataSource(tenantId: String): javax.sql.DataSource {
        return tenantDataSources[tenantId]
            ?: throw TenantNotFoundException("No datasource for tenant: $tenantId")
    }
}

class TenantNotFoundException(msg: String) : RuntimeException(msg)

typealias Configuration = org.springframework.context.annotation.Configuration
typealias Bean = org.springframework.context.annotation.Bean
typealias LocalContainerEntityManagerFactoryBean = org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean
typealias LocalContainerEntityManagerFactoryBuilder = org.springframework.boot.orm.jpa.EntityManagerFactoryBuilder
typealias MultiTenantConnectionProvider = org.hibernate.engine.jdbc.connections.spi.MultiTenantConnectionProvider
typealias CurrentTenantIdentifierResolver = org.hibernate.context.spi.CurrentTenantIdentifierResolver

// Tenant provisioning service
@Service
class TenantProvisioningService(
    private val flyway: org.flywaydb.core.Flyway,
    private val dataSource: javax.sql.DataSource
) {
    
    // Create schema and run migrations for a new tenant
    fun provisionTenant(tenantId: String) {
        // Validate tenant ID (alphanumeric + underscore only)
        require(tenantId.matches(Regex("[a-z0-9_]+"))) {
            "Invalid tenant ID: $tenantId"
        }
        
        // Create schema
        createSchema(tenantId)
        
        // Run Flyway migrations on the new schema
        val tenantFlyway = org.flywaydb.core.Flyway.configure()
            .dataSource(dataSource)
            .schemas(tenantId)
            .locations("classpath:db/migration/tenant")
            .load()
        
        tenantFlyway.migrate()
    }
    
    private fun createSchema(schemaName: String) {
        dataSource.connection.use { conn ->
            conn.createStatement().use { stmt ->
                // Use parameterized identifier to prevent SQL injection
                stmt.execute("CREATE SCHEMA IF NOT EXISTS \"$schemaName\"")
            }
        }
    }
    
    fun dropTenant(tenantId: String) {
        dataSource.connection.use { conn ->
            conn.createStatement().use { stmt ->
                stmt.execute("DROP SCHEMA IF EXISTS \"$tenantId\" CASCADE")
            }
        }
    }
}

typealias Service = org.springframework.stereotype.Service
```

---

## Row-Level Security

```kotlin
// Shared schema approach with tenant_id column
// PostgreSQL Row-Level Security (RLS)

// SQL to enable RLS:
// ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
// CREATE POLICY tenant_isolation ON orders
//   USING (tenant_id = current_setting('app.tenant_id'));
// CREATE INDEX ON orders(tenant_id);  -- critical for performance

@Entity
@Table(name = "orders")
class OrderEntity(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(name = "tenant_id", nullable = false)
    val tenantId: String = TenantContext.getCurrentTenantOrThrow(),
    
    @Column(nullable = false)
    val customerName: String = "",
    
    @Column(nullable = false)
    val totalAmount: java.math.BigDecimal = java.math.BigDecimal.ZERO,
    
    @Column(nullable = false)
    val status: String = "PENDING"
)

typealias Entity = javax.persistence.Entity
typealias Table = javax.persistence.Table
typealias Id = javax.persistence.Id
typealias Column = javax.persistence.Column
typealias GeneratedValue = javax.persistence.GeneratedValue
typealias GenerationType = javax.persistence.GenerationType

// Hibernate interceptor: automatically injects tenant_id into queries
@Component
class TenantHibernateInterceptor : org.hibernate.EmptyInterceptor() {
    
    // Called before loading an entity
    override fun onLoad(
        entity: Any,
        id: java.io.Serializable?,
        state: Array<Any?>,
        propertyNames: Array<String>,
        types: Array<org.hibernate.type.Type>
    ): Boolean {
        // Validate tenant when loading
        if (entity is TenantAware) {
            val tenantId = TenantContext.getCurrentTenant()
            if (tenantId != null && entity.getTenantId() != tenantId) {
                throw IllegalAccessException("Tenant mismatch: cannot access other tenant's data")
            }
        }
        return false
    }
}

interface TenantAware {
    fun getTenantId(): String
}

// Spring Data JPA repository with automatic tenant filtering
interface OrderRepository : JpaRepository<OrderEntity, Long> {
    // These queries automatically filter by tenant_id if RLS is set up
    // Or we add explicit where clause
    @Query("SELECT o FROM OrderEntity o WHERE o.tenantId = :#{T(com.example.TenantContext).getCurrentTenantOrThrow()}")
    fun findAllForCurrentTenant(): List<OrderEntity>
    
    fun findByIdAndTenantId(id: Long, tenantId: String): OrderEntity?
}

typealias JpaRepository = org.springframework.data.jpa.repository.JpaRepository
typealias Query = org.springframework.data.jpa.repository.Query

// Service layer with tenant isolation
@Service
@Transactional
class OrderService(private val orderRepository: OrderRepository) {
    
    fun findById(id: Long): OrderEntity {
        val tenantId = TenantContext.getCurrentTenantOrThrow()
        
        return orderRepository.findByIdAndTenantId(id, tenantId)
            ?: throw OrderNotFoundException("Order $id not found for tenant $tenantId")
    }
    
    fun findAll(): List<OrderEntity> {
        return orderRepository.findAllForCurrentTenant()
    }
    
    fun create(request: CreateOrderRequest): OrderEntity {
        val tenantId = TenantContext.getCurrentTenantOrThrow()
        
        val order = OrderEntity(
            tenantId = tenantId,
            customerName = request.customerName,
            totalAmount = request.totalAmount
        )
        
        return orderRepository.save(order)
    }
}

data class CreateOrderRequest(
    val customerName: String,
    val totalAmount: java.math.BigDecimal
)

class OrderNotFoundException(msg: String) : RuntimeException(msg)

typealias Transactional = org.springframework.transaction.annotation.Transactional

// Set PostgreSQL session variable for RLS
@Component
class TenantSessionInterceptor : org.springframework.web.servlet.HandlerInterceptor {
    
    @org.springframework.beans.factory.annotation.Autowired
    private lateinit var dataSource: javax.sql.DataSource
    
    override fun preHandle(
        request: jakarta.servlet.http.HttpServletRequest,
        response: jakarta.servlet.http.HttpServletResponse,
        handler: Any
    ): Boolean {
        val tenantId = TenantContext.getCurrentTenant()
        
        if (tenantId != null) {
            // Set session variable for PostgreSQL RLS
            dataSource.connection.use { conn ->
                conn.createStatement().execute(
                    "SELECT set_config('app.tenant_id', '$tenantId', true)"
                )
            }
        }
        
        return true
    }
}
```

---

## Tenant-aware Caching

```kotlin
// Cache keys must include tenant ID to avoid cross-tenant data leaks

@Service
class TenantAwareProductService(
    private val productRepository: ProductRepository,
    private val redisTemplate: org.springframework.data.redis.core.RedisTemplate<String, String>,
    private val objectMapper: com.fasterxml.jackson.databind.ObjectMapper
) {
    
    // Cache key includes tenant ID: "product:acme:123"
    private fun cacheKey(productId: Long): String {
        val tenantId = TenantContext.getCurrentTenantOrThrow()
        return "product:$tenantId:$productId"
    }
    
    private fun tenantProductsKey(): String {
        val tenantId = TenantContext.getCurrentTenantOrThrow()
        return "products:$tenantId:all"
    }
    
    fun findById(productId: Long): ProductDto? {
        val key = cacheKey(productId)
        
        // Try cache
        val cached = redisTemplate.opsForValue().get(key)
        if (cached != null) {
            return objectMapper.readValue(cached, ProductDto::class.java)
        }
        
        // Load from DB
        val product = productRepository.findByIdAndTenantId(
            productId,
            TenantContext.getCurrentTenantOrThrow()
        ) ?: return null
        
        val dto = product.toDto()
        
        // Cache with tenant-scoped key
        redisTemplate.opsForValue().set(
            key,
            objectMapper.writeValueAsString(dto),
            java.time.Duration.ofMinutes(10)
        )
        
        return dto
    }
    
    fun evictTenantCache() {
        val tenantId = TenantContext.getCurrentTenantOrThrow()
        val pattern = "product:$tenantId:*"
        
        // Scan and delete all keys for this tenant
        val keys = redisTemplate.keys(pattern)
        if (keys.isNotEmpty()) {
            redisTemplate.delete(keys)
        }
    }
    
    // Evict all cache when tenant configuration changes
    fun onTenantConfigChanged(tenantId: String) {
        TenantContext.withTenant(tenantId) {
            evictTenantCache()
        }
    }
}

interface ProductRepository {
    fun findByIdAndTenantId(id: Long, tenantId: String): ProductEntity?
}

data class ProductEntity(val id: Long, val tenantId: String, val name: String) {
    fun toDto() = ProductDto(id, name)
}

data class ProductDto(val id: Long, val name: String)

// Spring Cache with tenant-aware key generator
@Configuration
class CacheConfig {
    
    @Bean
    fun tenantAwareKeyGenerator(): org.springframework.cache.interceptor.KeyGenerator {
        return org.springframework.cache.interceptor.KeyGenerator { target, method, params ->
            val tenantId = TenantContext.getCurrentTenant() ?: "default"
            val paramKey = params.joinToString(":")
            "$tenantId:${method.name}:$paramKey"
        }
    }
}

// Usage with Spring Cache
@Service
class CachedProductService(private val productRepository: ProductRepository) {
    
    @org.springframework.cache.annotation.Cacheable(
        cacheNames = ["products"],
        keyGenerator = "tenantAwareKeyGenerator"
    )
    fun findById(productId: Long): ProductDto? {
        return productRepository.findByIdAndTenantId(
            productId,
            TenantContext.getCurrentTenantOrThrow()
        )?.toDto()
    }
    
    @org.springframework.cache.annotation.CacheEvict(
        cacheNames = ["products"],
        keyGenerator = "tenantAwareKeyGenerator"
    )
    fun update(productId: Long, dto: ProductDto): ProductDto {
        // Update and return
        return dto
    }
}
```

---

## Multi-tenant Configuration

```kotlin
// Each tenant can have their own configuration/settings
@Entity
@Table(name = "tenant_config")
class TenantConfig(
    @Id
    val tenantId: String = "",
    
    @Column(name = "display_name")
    val displayName: String = "",
    
    @Column(name = "max_users")
    val maxUsers: Int = 100,
    
    @Column(name = "plan")
    val plan: TenantPlan = TenantPlan.FREE,
    
    @Column(name = "custom_domain")
    val customDomain: String? = null,
    
    @Column(name = "timezone")
    val timezone: String = "UTC",
    
    @Column(name = "locale")
    val locale: String = "en",
    
    @Column(name = "active")
    val active: Boolean = true,
    
    @Column(name = "created_at")
    val createdAt: java.time.Instant = java.time.Instant.now()
)

enum class TenantPlan { FREE, STARTER, PROFESSIONAL, ENTERPRISE }

// Tenant config service with caching
@Service
class TenantConfigService(
    private val tenantConfigRepository: TenantConfigRepository,
    private val cache: org.springframework.cache.CacheManager
) {
    
    fun getConfig(tenantId: String): TenantConfig {
        return cache.getCache("tenant-config")
            ?.get(tenantId, TenantConfig::class.java)
            ?: loadAndCache(tenantId)
    }
    
    private fun loadAndCache(tenantId: String): TenantConfig {
        val config = tenantConfigRepository.findById(tenantId)
            ?: throw TenantNotFoundException("Tenant not found: $tenantId")
        
        cache.getCache("tenant-config")?.put(tenantId, config)
        
        return config
    }
    
    fun getCurrentTenantConfig(): TenantConfig {
        return getConfig(TenantContext.getCurrentTenantOrThrow())
    }
    
    // Check if feature is available for tenant's plan
    fun isPlanFeatureAvailable(feature: PlanFeature): Boolean {
        val config = getCurrentTenantConfig()
        
        return when (feature) {
            PlanFeature.ADVANCED_ANALYTICS -> config.plan >= TenantPlan.PROFESSIONAL
            PlanFeature.SSO -> config.plan >= TenantPlan.ENTERPRISE
            PlanFeature.API_ACCESS -> config.plan >= TenantPlan.STARTER
            PlanFeature.BASIC_REPORTS -> true  // Available to all plans
        }
    }
}

enum class PlanFeature {
    BASIC_REPORTS, API_ACCESS, ADVANCED_ANALYTICS, SSO
}

interface TenantConfigRepository {
    fun findById(tenantId: String): TenantConfig?
}

// Tenant validation in controller
@RestController
@RequestMapping("/api/v1")
class TenantController(
    private val tenantConfigService: TenantConfigService
) {
    
    @GetMapping("/current-tenant")
    fun getCurrentTenant(): TenantInfo {
        val config = tenantConfigService.getCurrentTenantConfig()
        
        return TenantInfo(
            tenantId = config.tenantId,
            displayName = config.displayName,
            plan = config.plan.name,
            timezone = config.timezone,
            locale = config.locale,
            features = PlanFeature.values().filter { feature ->
                tenantConfigService.isPlanFeatureAvailable(feature)
            }.map { it.name }
        )
    }
}

data class TenantInfo(
    val tenantId: String,
    val displayName: String,
    val plan: String,
    val timezone: String,
    val locale: String,
    val features: List<String>
)

typealias RestController = org.springframework.web.bind.annotation.RestController
typealias RequestMapping = org.springframework.web.bind.annotation.RequestMapping
typealias GetMapping = org.springframework.web.bind.annotation.GetMapping
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement tenant onboarding workflow

// Requirements:
// 1. Create tenant record in tenant_config table
// 2. Provision database schema (if schema-per-tenant)
// 3. Seed initial data (admin user, default settings)
// 4. Send welcome email to tenant admin
// 5. Return tenant credentials

data class TenantOnboardingRequest(
    val tenantId: String,
    val displayName: String,
    val adminEmail: String,
    val adminName: String,
    val plan: TenantPlan = TenantPlan.FREE
)

data class TenantOnboardingResult(
    val tenantId: String,
    val adminUserId: String,
    val temporaryPassword: String,
    val loginUrl: String
)

@Service
class TenantOnboardingService(
    private val tenantConfigRepository: TenantConfigRepository,
    private val tenantProvisioningService: TenantProvisioningService,
    private val userService: UserService,
    private val emailService: EmailService
) {
    
    @Transactional
    fun onboardTenant(request: TenantOnboardingRequest): TenantOnboardingResult {
        // 1. Validate tenant ID uniqueness
        // TODO: check if tenantId already exists
        
        // 2. Create tenant config
        val config = TenantConfig(
            tenantId = request.tenantId,
            displayName = request.displayName,
            plan = request.plan
        )
        tenantConfigRepository.save(config)
        
        // 3. Provision schema (for schema-per-tenant)
        tenantProvisioningService.provisionTenant(request.tenantId)
        
        // 4. Create admin user in new tenant context
        val (adminUser, tempPassword) = TenantContext.withTenant(request.tenantId) {
            userService.createAdminUser(request.adminEmail, request.adminName)
        }
        
        // 5. Send welcome email
        emailService.sendWelcomeEmail(
            email = request.adminEmail,
            name = request.adminName,
            tenantId = request.tenantId,
            tempPassword = tempPassword
        )
        
        return TenantOnboardingResult(
            tenantId = request.tenantId,
            adminUserId = adminUser.id,
            temporaryPassword = tempPassword,
            loginUrl = "https://${request.tenantId}.myapp.com/login"
        )
    }
}

data class TenantConfig(
    val tenantId: String,
    val displayName: String,
    val plan: TenantPlan = TenantPlan.FREE,
    val active: Boolean = true
)

interface TenantConfigRepository {
    fun save(config: TenantConfig)
}

interface UserService {
    fun createAdminUser(email: String, name: String): Pair<UserDto, String>
}

interface EmailService {
    fun sendWelcomeEmail(email: String, name: String, tenantId: String, tempPassword: String)
}

data class UserDto(val id: String, val email: String)
```

---

## สรุป Part 70

```
✅ Multi-tenancy models: database, schema, row-level
✅ TenantContext: ThreadLocal-based tenant isolation
✅ TenantResolver: subdomain, header, JWT claim
✅ TenantFilter: set/clear tenant per request
✅ withTenant(): safe scoped context block
✅ Schema-per-Tenant: Hibernate MultiTenancy=SCHEMA
✅ TenantProvisioningService: flyway migration per schema
✅ Row-Level Security: PostgreSQL RLS + tenant_id column
✅ TenantHibernateInterceptor: validate tenant on load
✅ Tenant-aware caching: "product:acme:123" key pattern
✅ Cache eviction: evict all keys for a tenant
✅ TenantConfig: plan (FREE/STARTER/PRO/ENTERPRISE)
✅ PlanFeature: check feature availability per plan
✅ TenantOnboardingService: full provisioning workflow
✅ withTenant block: create admin user in new schema
```

---

*Part 70/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
