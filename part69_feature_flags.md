# Part 69: Feature Flags และ A/B Testing

## สารบัญ
1. [Feature Flags คืออะไร](#feature-flags-คืออะไร)
2. [Feature Flag ด้วย Spring และ Redis](#feature-flag-ด้วย-spring-และ-redis)
3. [Gradual Rollout](#gradual-rollout)
4. [A/B Testing](#ab-testing)
5. [Feature Flag Management](#feature-flag-management)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Feature Flags คืออะไร

```
Feature Flags (Feature Toggles) คือ pattern ที่ช่วยให้:

1. Deploy code บ่อยๆ โดยไม่ release feature ทันที
2. ทดสอบ feature กับ subset ของ users ก่อน
3. Kill switch: ปิด feature ที่มีปัญหาได้ทันที
4. A/B Testing: วัดว่า A หรือ B ดีกว่ากัน
5. Canary Release: release ทีละน้อย เช่น 1% → 10% → 100%

ประเภทของ Feature Flags:
- Release Toggle: ซ่อน incomplete feature
- Experiment Toggle: A/B testing
- Ops Toggle: operational control (enable/disable backend)
- Permission Toggle: feature สำหรับ premium users เท่านั้น
```

---

## Feature Flag ด้วย Spring และ Redis

```kotlin
// Feature Flag entity
data class FeatureFlag(
    val key: String,
    val enabled: Boolean,
    val rolloutPercentage: Int = 100,  // 0-100%
    val allowedUserIds: Set<String> = emptySet(),
    val allowedRoles: Set<String> = emptySet(),
    val metadata: Map<String, String> = emptyMap(),
    val createdAt: java.time.Instant = java.time.Instant.now(),
    val updatedAt: java.time.Instant = java.time.Instant.now()
)

// Feature Flag Service
@Service
class FeatureFlagService(
    private val redisTemplate: RedisTemplate<String, String>,
    private val flagRepository: FeatureFlagRepository,
    private val objectMapper: ObjectMapper
) {
    
    companion object {
        const val CACHE_PREFIX = "feature:flag:"
        val CACHE_TTL = java.time.Duration.ofMinutes(5)
    }
    
    // Check if feature is enabled for a specific user
    fun isEnabled(flagKey: String, userId: String? = null, userRoles: Set<String> = emptySet()): Boolean {
        val flag = getFlag(flagKey) ?: return false
        
        if (!flag.enabled) return false
        
        // Check explicit user allowlist
        if (userId != null && flag.allowedUserIds.isNotEmpty()) {
            if (userId in flag.allowedUserIds) return true
            // If there's an allowlist and user is not in it, fall through to percentage check
        }
        
        // Check role-based access
        if (flag.allowedRoles.isNotEmpty()) {
            if (userRoles.intersect(flag.allowedRoles).isNotEmpty()) return true
        }
        
        // Gradual rollout: hash user ID to 0-99
        if (userId != null && flag.rolloutPercentage < 100) {
            val bucket = getUserBucket(userId, flagKey)
            return bucket < flag.rolloutPercentage
        }
        
        return flag.rolloutPercentage == 100
    }
    
    fun getFlag(flagKey: String): FeatureFlag? {
        val cacheKey = "$CACHE_PREFIX$flagKey"
        
        // Check Redis cache first
        val cached = redisTemplate.opsForValue().get(cacheKey)
        if (cached != null) {
            return objectMapper.readValue(cached, FeatureFlag::class.java)
        }
        
        // Load from database
        val flag = flagRepository.findByKey(flagKey) ?: return null
        
        // Cache it
        redisTemplate.opsForValue().set(
            cacheKey,
            objectMapper.writeValueAsString(flag),
            CACHE_TTL
        )
        
        return flag
    }
    
    fun updateFlag(flag: FeatureFlag) {
        flagRepository.save(flag)
        // Invalidate cache
        redisTemplate.delete("$CACHE_PREFIX${flag.key}")
    }
    
    // Hash user to consistent bucket (0-99)
    // Same user always gets same bucket, so experience is consistent
    private fun getUserBucket(userId: String, flagKey: String): Int {
        val hash = "$userId:$flagKey".hashCode()
        return Math.abs(hash) % 100
    }
    
    // Get all flags for a user (for client-side use)
    fun getAllFlagsForUser(userId: String, userRoles: Set<String>): Map<String, Boolean> {
        return flagRepository.findAll().associate { flag ->
            flag.key to isEnabled(flag.key, userId, userRoles)
        }
    }
}

interface FeatureFlagRepository {
    fun findByKey(key: String): FeatureFlag?
    fun findAll(): List<FeatureFlag>
    fun save(flag: FeatureFlag): FeatureFlag
}

typealias RedisTemplate<K, V> = org.springframework.data.redis.core.RedisTemplate<K, V>
typealias ObjectMapper = com.fasterxml.jackson.databind.ObjectMapper
typealias Service = org.springframework.stereotype.Service
```

---

## Gradual Rollout

```kotlin
// Gradual rollout with consistent hashing

@Component
class GradualRolloutService(private val featureFlagService: FeatureFlagService) {
    
    // Check feature with user context
    fun isFeatureEnabled(
        feature: Feature,
        userContext: UserContext
    ): Boolean {
        return featureFlagService.isEnabled(
            flagKey = feature.key,
            userId = userContext.userId,
            userRoles = userContext.roles
        )
    }
    
    // Variant selection for A/B/n testing
    fun getVariant(experimentKey: String, userId: String): String {
        val experiment = featureFlagService.getFlag(experimentKey) ?: return "control"
        
        if (!experiment.enabled) return "control"
        
        val bucket = getUserBucket(userId, experimentKey)
        
        // Parse variants from metadata: "variants=control:50,treatment_a:25,treatment_b:25"
        val variants = parseVariants(experiment.metadata["variants"] ?: "control:100")
        
        var cumulative = 0
        for ((variant, percentage) in variants) {
            cumulative += percentage
            if (bucket < cumulative) return variant
        }
        
        return "control"
    }
    
    private fun parseVariants(variantStr: String): List<Pair<String, Int>> {
        return variantStr.split(",").map { entry ->
            val (name, pct) = entry.split(":")
            name.trim() to pct.trim().toInt()
        }
    }
    
    private fun getUserBucket(userId: String, key: String): Int {
        return Math.abs("$userId:$key".hashCode()) % 100
    }
}

enum class Feature(val key: String) {
    NEW_CHECKOUT("new_checkout_flow"),
    AI_RECOMMENDATIONS("ai_product_recommendations"),
    DARK_MODE("dark_mode"),
    BETA_DASHBOARD("beta_dashboard"),
    PREMIUM_SHIPPING("premium_shipping_option")
}

data class UserContext(
    val userId: String,
    val roles: Set<String>,
    val email: String? = null,
    val country: String? = null
)

// Usage in service
@Service
class CheckoutService(
    private val rolloutService: GradualRolloutService
) {
    
    fun processCheckout(userId: String, cart: Cart): CheckoutResult {
        val userContext = UserContext(userId, setOf("USER"))
        
        return if (rolloutService.isFeatureEnabled(Feature.NEW_CHECKOUT, userContext)) {
            processNewCheckout(cart)  // New checkout for X% of users
        } else {
            processLegacyCheckout(cart)  // Old checkout for the rest
        }
    }
    
    private fun processNewCheckout(cart: Cart): CheckoutResult {
        return CheckoutResult("SUCCESS", "new_checkout")
    }
    
    private fun processLegacyCheckout(cart: Cart): CheckoutResult {
        return CheckoutResult("SUCCESS", "legacy_checkout")
    }
}

data class Cart(val items: List<Any>)
data class CheckoutResult(val status: String, val checkoutType: String)
typealias Component = org.springframework.stereotype.Component
```

---

## A/B Testing

```kotlin
// A/B Test framework
@Service
class ABTestService(
    private val featureFlagService: FeatureFlagService,
    private val experimentTracker: ExperimentTracker
) {
    
    // Assign user to experiment variant
    fun assignVariant(experimentKey: String, userId: String): ExperimentAssignment {
        val bucket = Math.abs("$userId:$experimentKey".hashCode()) % 100
        
        val experiment = featureFlagService.getFlag(experimentKey)
            ?: return ExperimentAssignment(experimentKey, userId, "control", false)
        
        if (!experiment.enabled) {
            return ExperimentAssignment(experimentKey, userId, "control", false)
        }
        
        // Get variant based on bucket
        val variant = when {
            bucket < 50 -> "control"
            else -> "treatment"
        }
        
        // Track assignment
        experimentTracker.trackAssignment(
            ExperimentAssignmentEvent(
                experimentKey = experimentKey,
                userId = userId,
                variant = variant,
                timestamp = java.time.Instant.now()
            )
        )
        
        return ExperimentAssignment(experimentKey, userId, variant, true)
    }
    
    // Track conversion event
    fun trackConversion(
        experimentKey: String,
        userId: String,
        eventName: String,
        value: Double? = null
    ) {
        val assignment = assignVariant(experimentKey, userId)
        
        experimentTracker.trackConversion(
            ExperimentConversionEvent(
                experimentKey = experimentKey,
                userId = userId,
                variant = assignment.variant,
                eventName = eventName,
                value = value,
                timestamp = java.time.Instant.now()
            )
        )
    }
}

data class ExperimentAssignment(
    val experimentKey: String,
    val userId: String,
    val variant: String,
    val isEnrolled: Boolean
)

data class ExperimentAssignmentEvent(
    val experimentKey: String,
    val userId: String,
    val variant: String,
    val timestamp: java.time.Instant
)

data class ExperimentConversionEvent(
    val experimentKey: String,
    val userId: String,
    val variant: String,
    val eventName: String,
    val value: Double?,
    val timestamp: java.time.Instant
)

interface ExperimentTracker {
    fun trackAssignment(event: ExperimentAssignmentEvent)
    fun trackConversion(event: ExperimentConversionEvent)
}

// Statistical significance calculator
object StatisticsCalculator {
    
    // Z-test for proportions (conversion rates)
    fun calculateSignificance(
        controlConversions: Int,
        controlTotal: Int,
        treatmentConversions: Int,
        treatmentTotal: Int
    ): ABTestResult {
        val pControl = controlConversions.toDouble() / controlTotal
        val pTreatment = treatmentConversions.toDouble() / treatmentTotal
        
        val pPooled = (controlConversions + treatmentConversions).toDouble() / (controlTotal + treatmentTotal)
        
        val standardError = Math.sqrt(
            pPooled * (1 - pPooled) * (1.0 / controlTotal + 1.0 / treatmentTotal)
        )
        
        val zScore = (pTreatment - pControl) / standardError
        val pValue = 2 * (1 - normalCDF(Math.abs(zScore)))
        
        val uplift = if (pControl > 0) (pTreatment - pControl) / pControl * 100 else 0.0
        
        return ABTestResult(
            controlConversionRate = pControl,
            treatmentConversionRate = pTreatment,
            absoluteLift = pTreatment - pControl,
            relativeLift = uplift,
            zScore = zScore,
            pValue = pValue,
            isSignificant = pValue < 0.05,  // 95% confidence level
            confidenceLevel = (1 - pValue) * 100
        )
    }
    
    // Normal CDF approximation
    private fun normalCDF(z: Double): Double {
        val t = 1.0 / (1.0 + 0.2316419 * Math.abs(z))
        val d = 0.3989423 * Math.exp(-z * z / 2.0)
        val p = d * t * (0.3193815 + t * (-0.3565638 + t * (1.7814780 + t * (-1.8212560 + t * 1.3302740))))
        return if (z > 0) 1.0 - p else p
    }
}

data class ABTestResult(
    val controlConversionRate: Double,
    val treatmentConversionRate: Double,
    val absoluteLift: Double,
    val relativeLift: Double,
    val zScore: Double,
    val pValue: Double,
    val isSignificant: Boolean,
    val confidenceLevel: Double
)

// Usage example
fun analyzeCheckoutExperiment() {
    val result = StatisticsCalculator.calculateSignificance(
        controlConversions = 450,
        controlTotal = 1000,   // 45% conversion
        treatmentConversions = 510,
        treatmentTotal = 1000  // 51% conversion
    )
    
    println("Control: ${String.format("%.1f%%", result.controlConversionRate * 100)}")
    println("Treatment: ${String.format("%.1f%%", result.treatmentConversionRate * 100)}")
    println("Relative Lift: ${String.format("%.1f%%", result.relativeLift)}")
    println("P-Value: ${String.format("%.4f", result.pValue)}")
    println("Significant: ${result.isSignificant}")
    println("Confidence: ${String.format("%.1f%%", result.confidenceLevel)}")
}
```

---

## Feature Flag Management API

```kotlin
// Admin REST API for managing feature flags
@RestController
@RequestMapping("/api/admin/feature-flags")
@PreAuthorize("hasRole('ADMIN')")
class FeatureFlagController(
    private val featureFlagService: FeatureFlagService
) {
    
    @GetMapping
    fun listFlags(): List<FeatureFlag> {
        return featureFlagService.listAll()
    }
    
    @GetMapping("/{key}")
    fun getFlag(@PathVariable key: String): ResponseEntity<FeatureFlag> {
        return featureFlagService.getFlag(key)
            ?.let { ResponseEntity.ok(it) }
            ?: ResponseEntity.notFound().build()
    }
    
    @PutMapping("/{key}")
    fun updateFlag(
        @PathVariable key: String,
        @RequestBody update: UpdateFlagRequest
    ): FeatureFlag {
        val current = featureFlagService.getFlag(key)
            ?: throw ResourceNotFoundException("Flag $key not found")
        
        val updated = current.copy(
            enabled = update.enabled ?: current.enabled,
            rolloutPercentage = update.rolloutPercentage ?: current.rolloutPercentage,
            allowedUserIds = update.allowedUserIds ?: current.allowedUserIds,
            allowedRoles = update.allowedRoles ?: current.allowedRoles,
            metadata = update.metadata ?: current.metadata,
            updatedAt = java.time.Instant.now()
        )
        
        featureFlagService.updateFlag(updated)
        return updated
    }
    
    // Toggle flag on/off quickly
    @PostMapping("/{key}/toggle")
    fun toggleFlag(@PathVariable key: String): FeatureFlag {
        val current = featureFlagService.getFlag(key)
            ?: throw ResourceNotFoundException("Flag $key not found")
        
        val toggled = current.copy(
            enabled = !current.enabled,
            updatedAt = java.time.Instant.now()
        )
        
        featureFlagService.updateFlag(toggled)
        return toggled
    }
    
    // Check flag for a specific user (for debugging)
    @GetMapping("/{key}/check")
    fun checkFlagForUser(
        @PathVariable key: String,
        @RequestParam userId: String
    ): Map<String, Any> {
        val enabled = featureFlagService.isEnabled(key, userId)
        val flag = featureFlagService.getFlag(key)
        val bucket = Math.abs("$userId:$key".hashCode()) % 100
        
        return mapOf(
            "flagKey" to key,
            "userId" to userId,
            "enabled" to enabled,
            "rolloutPercentage" to (flag?.rolloutPercentage ?: 0),
            "userBucket" to bucket,
            "inRollout" to (bucket < (flag?.rolloutPercentage ?: 0))
        )
    }
}

data class UpdateFlagRequest(
    val enabled: Boolean? = null,
    val rolloutPercentage: Int? = null,
    val allowedUserIds: Set<String>? = null,
    val allowedRoles: Set<String>? = null,
    val metadata: Map<String, String>? = null
)

fun FeatureFlagService.listAll(): List<FeatureFlag> = TODO("implement in repository")

typealias PreAuthorize = org.springframework.security.access.prepost.PreAuthorize
typealias RestController = org.springframework.web.bind.annotation.RestController
typealias RequestMapping = org.springframework.web.bind.annotation.RequestMapping
typealias GetMapping = org.springframework.web.bind.annotation.GetMapping
typealias PutMapping = org.springframework.web.bind.annotation.PutMapping
typealias PostMapping = org.springframework.web.bind.annotation.PostMapping
typealias PathVariable = org.springframework.web.bind.annotation.PathVariable
typealias RequestParam = org.springframework.web.bind.annotation.RequestParam
typealias RequestBody = org.springframework.web.bind.annotation.RequestBody
typealias ResponseEntity = org.springframework.http.ResponseEntity<*>
class ResourceNotFoundException(msg: String) : Exception(msg)
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement percentage-based rollout with sticky sessions

// Requirements:
// 1. User should always get the same variant (sticky)
// 2. Rollout should be based on user ID hash
// 3. Support rollout groups: by country, by user tier, etc.
// 4. Support override: specific user always gets treatment

data class RolloutConfig(
    val percentage: Int,
    val stickyByUserId: Boolean = true,
    val groupBy: RolloutGrouping? = null,
    val overrides: Map<String, String> = emptyMap()  // userId → variant
)

enum class RolloutGrouping { USER_ID, EMAIL_DOMAIN, COUNTRY, USER_TIER }

class StickyRolloutService {
    
    fun getVariant(
        experimentKey: String,
        userId: String,
        config: RolloutConfig
    ): String {
        // Check overrides first
        config.overrides[userId]?.let { return it }
        
        // Sticky assignment based on userId hash
        val bucket = computeBucket(userId, experimentKey, config.groupBy)
        
        return if (bucket < config.percentage) "treatment" else "control"
    }
    
    private fun computeBucket(
        userId: String,
        experimentKey: String,
        groupBy: RolloutGrouping?
    ): Int {
        val key = when (groupBy) {
            RolloutGrouping.EMAIL_DOMAIN -> extractEmailDomain(userId)
            RolloutGrouping.COUNTRY -> getCountryForUser(userId)
            else -> userId
        }
        
        return Math.abs("$key:$experimentKey".hashCode()) % 100
    }
    
    private fun extractEmailDomain(userId: String): String {
        // TODO: look up user's email domain
        return "example.com"
    }
    
    private fun getCountryForUser(userId: String): String {
        // TODO: look up user's country from profile
        return "TH"
    }
}
```

---

## สรุป Part 69

```
✅ Feature Flags: release, experiment, ops, permission toggles
✅ FeatureFlagService: enabled check with user context
✅ Redis caching: 5-minute TTL for flag values
✅ Kill switch: toggle flag on/off instantly via API
✅ Gradual Rollout: percentage-based, consistent hashing
✅ getUserBucket: hashCode % 100 for stable assignment
✅ allowedUserIds: explicit allowlist for testing
✅ allowedRoles: permission-based feature access
✅ A/B Testing: variant assignment via bucket
✅ ExperimentTracker: track assignments and conversions
✅ Z-test: statistical significance for conversion rates
✅ P-value < 0.05: 95% confidence level
✅ Relative Lift: % improvement over control
✅ Admin API: CRUD for feature flags
✅ /check endpoint: debug flag for specific user
✅ Sticky Sessions: same user always gets same variant
✅ Rollout Grouping: by country, email domain, user tier
```

---

*Part 69/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
