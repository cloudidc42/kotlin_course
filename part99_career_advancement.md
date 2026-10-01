# Part 99: Career Advancement — จาก Developer สู่ Tech Lead

## สารบัญ
1. [Technical Skills Roadmap](#technical-skills-roadmap)
2. [System Design Interview](#system-design-interview)
3. [Code Review Best Practices](#code-review-best-practices)
4. [Technical Documentation](#technical-documentation)
5. [Open Source Contribution](#open-source-contribution)
6. [Certifications & Learning Path](#certifications--learning-path)

---

## Technical Skills Roadmap

```
Junior Developer (0-2 years):
✅ Kotlin syntax, OOP, functions
✅ Spring Boot basic (REST, JPA, tests)
✅ Git, GitHub, basic CI/CD
✅ Docker basics
✅ SQL fundamentals

Mid-Level Developer (2-4 years):
✅ Spring Security, JWT, OAuth2
✅ Kafka, Redis, microservices
✅ Testing: unit + integration + TDD
✅ Kubernetes basics
✅ Code review: give and receive
✅ Performance profiling
✅ Design patterns (SOLID, Clean Architecture)

Senior Developer (4-7 years):
✅ System design at scale
✅ DDD, CQRS, Event Sourcing
✅ Database optimization, query tuning
✅ Cloud platform (AWS/GCP/Azure)
✅ Mentoring junior developers
✅ Technical specifications writing
✅ On-call, incident management
✅ Cost optimization

Tech Lead / Staff (7+ years):
✅ Architecture decisions, ADRs
✅ Cross-team technical alignment
✅ Build vs buy decisions
✅ Hiring, interviewing, team building
✅ Engineering culture
✅ Roadmap planning with product
✅ External speaking, blog writing
✅ Open source leadership
```

---

## System Design Interview

```kotlin
// ตัวอย่าง: Design a URL Shortener (bit.ly)

// Step 1: Clarify requirements
/*
Functional:
- Create short URL from long URL
- Redirect short → long URL
- Custom alias (optional)
- Link expiry (optional)

Non-functional:
- 100M URLs created/day
- 10B reads/day (100:1 read:write)
- Availability: 99.9%
- Latency: < 10ms for redirect

Back-of-envelope:
- 100M writes/day = 1,157 writes/sec
- 10B reads/day  = 115,740 reads/sec (peak 200k/sec)
- URL storage: 1000 bytes × 100M = 100GB/day
- 3-year retention: 100GB × 365 × 3 = 109TB
*/

// Step 2: High-level design
/*
[Client] → [Load Balancer] → [URL Shortener Service]
                                    ↓
                            [Cache (Redis)]
                                    ↓
                            [Database (PostgreSQL)]
                                    ↓
                            [Object Storage (S3)]
                              (for analytics)

Short URL generation: Base62 encoding of unique ID
- 62 chars (a-z A-Z 0-9), 7 chars = 62^7 = 3.5 trillion URLs
- Alternative: MD5 hash (take first 7 chars)
*/

// Step 3: Detailed design

// ID generation: distributed unique IDs
@org.springframework.stereotype.Service
class SnowflakeIdGenerator(
    private val datacenterId: Long = 1L,
    private val workerId: Long = 1L
) {
    private val epoch = 1609459200000L  // 2021-01-01
    private val datacenterBits = 5L
    private val workerBits = 5L
    private val sequenceBits = 12L
    
    private val maxDatacenterId = (-1L).xor(-1L.shl(datacenterBits.toInt()))
    private val maxWorkerId = (-1L).xor(-1L.shl(workerBits.toInt()))
    private val maxSequence = (-1L).xor(-1L.shl(sequenceBits.toInt()))
    
    private val workerShift = sequenceBits
    private val datacenterShift = sequenceBits + workerBits
    private val timestampShift = sequenceBits + workerBits + datacenterBits
    
    private var sequence = 0L
    private var lastTimestamp = -1L
    
    @Synchronized
    fun nextId(): Long {
        var timestamp = currentTimestamp()
        
        if (timestamp == lastTimestamp) {
            sequence = (sequence + 1) and maxSequence
            if (sequence == 0L) {
                timestamp = waitForNextMillis(lastTimestamp)
            }
        } else {
            sequence = 0L
        }
        
        lastTimestamp = timestamp
        
        return ((timestamp - epoch) shl timestampShift.toInt()) or
               (datacenterId shl datacenterShift.toInt()) or
               (workerId shl workerShift.toInt()) or
               sequence
    }
    
    private fun currentTimestamp() = System.currentTimeMillis()
    
    private fun waitForNextMillis(lastTs: Long): Long {
        var ts = currentTimestamp()
        while (ts <= lastTs) ts = currentTimestamp()
        return ts
    }
}

// Base62 encoding
object Base62Encoder {
    private val chars = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
    
    fun encode(num: Long): String {
        var n = num
        val result = StringBuilder()
        while (n > 0) {
            result.insert(0, chars[(n % 62).toInt()])
            n /= 62
        }
        return result.toString().padStart(7, '0')
    }
    
    fun decode(str: String): Long {
        return str.fold(0L) { acc, c -> acc * 62 + chars.indexOf(c) }
    }
}

// URL Shortener Service
data class ShortenedUrl(
    val shortCode: String,
    val longUrl: String,
    val createdAt: java.time.Instant,
    val expiresAt: java.time.Instant?,
    val customAlias: String?
)

@org.springframework.stereotype.Service
class UrlShortenerService(
    private val idGenerator: SnowflakeIdGenerator,
    private val urlRepository: UrlRepository,
    private val cache: org.springframework.data.redis.core.RedisTemplate<String, String>
) {
    
    fun shorten(longUrl: String, customAlias: String? = null, ttlDays: Int? = null): ShortenedUrl {
        // Validate URL
        try { java.net.URL(longUrl) } catch (e: Exception) { throw IllegalArgumentException("Invalid URL") }
        
        val shortCode = customAlias ?: Base62Encoder.encode(idGenerator.nextId())
        
        // Check if custom alias already taken
        if (customAlias != null && urlRepository.findByCode(customAlias) != null) {
            throw IllegalStateException("Custom alias already taken")
        }
        
        val url = ShortenedUrl(
            shortCode = shortCode,
            longUrl = longUrl,
            createdAt = java.time.Instant.now(),
            expiresAt = ttlDays?.let { java.time.Instant.now().plus(java.time.Duration.ofDays(it.toLong())) },
            customAlias = customAlias
        )
        
        urlRepository.save(url)
        cache.opsForValue().set("url:$shortCode", longUrl, 
            java.time.Duration.ofHours(24))
        
        return url
    }
    
    fun resolve(shortCode: String): String {
        // Cache-first
        val cached = cache.opsForValue().get("url:$shortCode")
        if (cached != null) return cached
        
        val url = urlRepository.findByCode(shortCode)
            ?: throw NoSuchElementException("Short URL not found: $shortCode")
        
        if (url.expiresAt != null && url.expiresAt.isBefore(java.time.Instant.now())) {
            throw IllegalStateException("Short URL has expired")
        }
        
        cache.opsForValue().set("url:$shortCode", url.longUrl, java.time.Duration.ofHours(24))
        
        return url.longUrl
    }
}

interface UrlRepository {
    fun save(url: ShortenedUrl)
    fun findByCode(code: String): ShortenedUrl?
}
```

---

## Code Review Best Practices

```kotlin
// Good code review: constructive, educational, specific

// ❌ BAD review comment:
// "This is wrong"
// "Why would you do this?"
// "Terrible code"

// ✅ GOOD review comment:
// "This might cause a NullPointerException when X is null.
//  Consider using ?.let { ... } or add a null check here.
//  See: https://kotlinlang.org/docs/null-safety.html"

// Code Review Checklist as Code
data class ReviewChecklist(
    val correctness: List<String> = listOf(
        "Does the code do what it says it does?",
        "Are edge cases handled? (null, empty, concurrent)",
        "Are error cases handled correctly?",
        "Is the logic correct for the business requirements?"
    ),
    val security: List<String> = listOf(
        "No SQL injection (using parameterized queries)?",
        "No XSS vulnerabilities (input sanitization)?",
        "Sensitive data not logged?",
        "Secrets not hardcoded?"
    ),
    val performance: List<String> = listOf(
        "No N+1 queries?",
        "Appropriate caching where needed?",
        "No unnecessary loops or computations?",
        "Database indexes for queried columns?"
    ),
    val maintainability: List<String> = listOf(
        "Functions single responsibility?",
        "Good naming (classes, functions, variables)?",
        "No magic numbers (use named constants)?",
        "Tests cover the new code?"
    ),
    val style: List<String> = listOf(
        "Follows project conventions?",
        "Lint/formatter passes?",
        "No commented-out code?",
        "Imports organized?"
    )
)

// Example: Automated code review with detekt
// detekt.yml
/*
complexity:
  LongMethod:
    threshold: 60  # lines
  CyclomaticComplexMethod:
    threshold: 15
  LargeClass:
    threshold: 400

naming:
  FunctionNaming:
    functionPattern: '([a-z][a-zA-Z0-9]*|`[^`]+`)'
  VariableNaming:
    variablePattern: '[a-z][A-Za-z0-9]*'

style:
  MaxLineLength:
    maxLineLength: 120
  MagicNumber:
    ignoreNumbers: ['-1', '0', '1', '2', '100']

potential-bugs:
  UnnecessaryLet: true
  UnusedPrivateMember: true
*/
```

---

## Architecture Decision Records (ADR)

```markdown
# ADR-001: Use Event Sourcing for Order Service

## Status
Accepted (2024-01-15)

## Context
Order service needs to:
- Track complete history of order changes
- Support complex state transitions
- Enable audit logging for compliance
- Allow temporal queries ("what was the state at time T?")

## Decision
Use Event Sourcing with PostgreSQL as event store.
Read model (projections) stored in separate tables.

## Rationale
- Complete audit trail without extra effort
- Temporal queries naturally supported
- Replay events to rebuild state/projections
- Decoupled read/write models (CQRS)

## Consequences
Positive:
+ Full order history
+ Easy audit trail
+ Time travel queries
+ CQRS for optimized reads

Negative:
- Learning curve for team
- More complex than CRUD
- Eventual consistency in projections
- Snapshot needed for large aggregate histories

## Alternatives Considered
1. CRUD with audit log table: simpler but separate concerns
2. Bitemporal data model: complex, overkill
3. CDC (Change Data Capture): depends on DB, harder to test

## Related ADRs
- ADR-002: Kafka for Domain Event Bus
- ADR-003: PostgreSQL for Event Store
```

---

## Certifications & Learning Path

```
Kotlin Certifications:
- JetBrains Academy: Kotlin Developer Track
- Android Developer Certification (Google)

Cloud Certifications:
- AWS: Solutions Architect Associate → Professional
- GCP: Professional Cloud Architect
- Azure: AZ-204 Developer, AZ-305 Architect

DevOps/Platform:
- CKA (Certified Kubernetes Administrator)
- CKAD (Certified Kubernetes Application Developer)
- Terraform Associate

Books to Read (by level):
Junior:
- "Clean Code" - Robert Martin
- "The Pragmatic Programmer" - Hunt & Thomas
- "Kotlin in Action" - Jemerov & Isakova

Mid:
- "Designing Data-Intensive Applications" - Kleppmann ⭐
- "Building Microservices" - Newman
- "Spring Boot in Action" - Craig Walls

Senior:
- "Domain-Driven Design" - Eric Evans
- "Release It!" - Nygard
- "Site Reliability Engineering" - Google (free online)
- "Software Architecture: The Hard Parts" - Ford et al.

Tech Lead:
- "An Elegant Puzzle" - Will Larson
- "The Staff Engineer's Path" - Tanya Reilly
- "Accelerate" - Forsgren, Humble, Kim

YouTube Channels:
- ByteByteGo (system design)
- TechWorld with Nana (DevOps)
- Fireship (quick concepts)
- Hussein Nasser (backend engineering)
- Kotlin by JetBrains

Blogs:
- martin.fowler.com (patterns, architecture)
- netflixtechblog.com (scale)
- engineering.atspotify.com
- medium.com/google-cloud
```

---

## สรุป Part 99

```
✅ Career levels: Junior → Mid → Senior → Tech Lead / Staff
✅ Each level: specific skills, responsibilities, scope
✅ System design interview: clarify → estimate → design → detail
✅ Back-of-envelope: 100M writes/day = 1,157/sec
✅ Snowflake ID: timestamp + datacenter + worker + sequence
✅ Base62: 7 chars = 62^7 = 3.5T URLs
✅ URL Shortener: ID → Base62 → Redis cache → DB
✅ Code review: constructive, specific, educational
✅ Review checklist: correctness, security, performance, maintainability
✅ detekt.yml: automated style/complexity enforcement
✅ ADR (Architecture Decision Record): document decisions
✅ ADR sections: status, context, decision, rationale, consequences
✅ Certifications: Kotlin, AWS, GCP, CKA, Terraform
✅ Books by level: Clean Code → DDIA → DDD → Staff Engineer's Path
✅ Learning resources: ByteByteGo, Martin Fowler, Netflix Tech Blog
✅ Public speaking, blog writing: demonstrate thought leadership
✅ Mentoring: invest in others → compound career growth
✅ Architecture principles: simple > clever, evolutionary, explicit
```

---

*Part 99/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
