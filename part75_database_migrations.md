# Part 75: Database Migrations ด้วย Flyway และ Liquibase

## สารบัญ
1. [Database Migration Fundamentals](#database-migration-fundamentals)
2. [Flyway ใน Spring Boot](#flyway-ใน-spring-boot)
3. [Migration Strategies](#migration-strategies)
4. [Zero-Downtime Migrations](#zero-downtime-migrations)
5. [Liquibase Comparison](#liquibase-comparison)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Database Migration Fundamentals

```
Database Migration คือ การเปลี่ยน schema อย่างมีระเบียบ

ทำไมต้องใช้ migration tool:
- Version control สำหรับ database schema
- Reproducible: สร้าง database ใหม่ได้เหมือนกันทุกครั้ง
- Team collaboration: ทุกคน run migration เดียวกัน
- Rollback: undo changes ได้
- CI/CD: migrate อัตโนมัติก่อน deploy

Flyway vs Liquibase:
Flyway:
  - SQL-first: เขียน SQL migrations
  - Simple: ง่ายเข้าใจ
  - Strict: fail fast if checksum mismatch

Liquibase:
  - XML/YAML/JSON/SQL: หลาย format
  - Flexible: supports rollback built-in
  - Complex: feature มากกว่า

Migration File Naming (Flyway):
V{version}__{description}.sql
V1__Create_users_table.sql
V2__Add_email_index.sql
V3__Create_orders_table.sql

Repeatable Migrations:
R__{description}.sql  (re-run when content changes)
R__seed_test_data.sql
```

---

## Flyway ใน Spring Boot

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.flywaydb:flyway-core:10.4.1")
    implementation("org.flywaydb:flyway-database-postgresql:10.4.1")
    runtimeOnly("org.postgresql:postgresql")
}

// application.yml
/*
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    out-of-order: false
    validate-on-migrate: true
    clean-disabled: true  # Never clean in production!
    placeholders:
      schema_name: public
*/

// Flyway configuration in Kotlin
@Configuration
class FlywayConfig {
    
    @Bean
    fun flyway(dataSource: javax.sql.DataSource): Flyway {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration", "classpath:db/seed")
            .baselineOnMigrate(true)
            .validateOnMigrate(true)
            .outOfOrder(false)
            .cleanDisabled(true)  // Safety: never drop all tables
            .callbacks(FlywayCallback())
            .load()
            .also { it.migrate() }
    }
}

class FlywayCallback : Callback {
    override fun supports(event: Event, context: Context) = event in listOf(
        Event.BEFORE_MIGRATE, Event.AFTER_MIGRATE, Event.AFTER_MIGRATE_ERROR
    )
    
    override fun handle(event: Event, context: Context) {
        when (event) {
            Event.BEFORE_MIGRATE -> println("Starting database migration...")
            Event.AFTER_MIGRATE -> println("Migration completed. Applied: ${context.migrateResult?.migrationsExecuted}")
            Event.AFTER_MIGRATE_ERROR -> println("Migration FAILED: ${context.migrateResult?.failedMigrations?.firstOrNull()?.description}")
            else -> {}
        }
    }
    
    override fun getCallbackName() = "FlywayCallback"
    override fun isUndo() = false
}

typealias Configuration = org.springframework.context.annotation.Configuration
typealias Bean = org.springframework.context.annotation.Bean
typealias Flyway = org.flywaydb.core.Flyway
typealias Callback = org.flywaydb.core.api.callback.Callback
typealias Event = org.flywaydb.core.api.callback.Event
typealias Context = org.flywaydb.core.api.callback.Context
```

---

## Migration SQL Files

```sql
-- src/main/resources/db/migration/V1__Create_users_table.sql

CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    name        VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role        VARCHAR(50) NOT NULL DEFAULT 'USER',
    active      BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
CREATE INDEX idx_users_active ON users(active) WHERE active = true;

-- Trigger: auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

```sql
-- V2__Create_products_table.sql

CREATE TABLE categories (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(255) NOT NULL,
    slug        VARCHAR(255) NOT NULL UNIQUE,
    parent_id   BIGINT REFERENCES categories(id),
    active      BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE products (
    id              BIGSERIAL PRIMARY KEY,
    sku             VARCHAR(100) NOT NULL UNIQUE,
    name            VARCHAR(500) NOT NULL,
    description     TEXT,
    price           DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    category_id     BIGINT REFERENCES categories(id),
    stock_quantity  INTEGER NOT NULL DEFAULT 0,
    active          BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_active ON products(active, category_id) WHERE active = true;

-- Full text search index
CREATE INDEX idx_products_search ON products USING gin(to_tsvector('english', name || ' ' || COALESCE(description, '')));

CREATE TRIGGER update_products_updated_at
    BEFORE UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

```sql
-- V3__Create_orders_table.sql

CREATE TYPE order_status AS ENUM (
    'PENDING', 'CONFIRMED', 'PROCESSING', 
    'SHIPPED', 'DELIVERED', 'CANCELLED', 'REFUNDED'
);

CREATE TABLE orders (
    id              BIGSERIAL PRIMARY KEY,
    order_number    VARCHAR(50) NOT NULL UNIQUE,
    user_id         BIGINT NOT NULL REFERENCES users(id),
    status          order_status NOT NULL DEFAULT 'PENDING',
    total_amount    DECIMAL(10,2) NOT NULL,
    currency        VARCHAR(3) NOT NULL DEFAULT 'THB',
    shipping_address JSONB NOT NULL,
    notes           TEXT,
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE TABLE order_items (
    id          BIGSERIAL PRIMARY KEY,
    order_id    BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id  BIGINT NOT NULL REFERENCES products(id),
    sku         VARCHAR(100) NOT NULL,
    name        VARCHAR(500) NOT NULL,  -- snapshot
    quantity    INTEGER NOT NULL CHECK (quantity > 0),
    unit_price  DECIMAL(10,2) NOT NULL,
    subtotal    DECIMAL(10,2) GENERATED ALWAYS AS (quantity * unit_price) STORED
);

CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created ON orders(created_at DESC);
CREATE INDEX idx_order_items_order ON order_items(order_id);

-- Partition orders by year for performance (optional for large tables)
-- CREATE TABLE orders_2024 PARTITION OF orders
--     FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

---

## Zero-Downtime Migrations

```kotlin
// Zero-downtime migration pattern: Expand → Migrate → Contract

// Scenario: rename column user_name → full_name

// STEP 1: Expand (Backward Compatible) - V4__Expand_add_full_name.sql
/*
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);
UPDATE users SET full_name = user_name;
ALTER TABLE users ALTER COLUMN full_name SET NOT NULL;

-- Keep user_name for backward compatibility during deployment
-- Both old and new app versions work
*/

// Kotlin code (during expansion phase): write to both columns
@Entity
class UserEntity(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(name = "user_name")  // Old column still exists
    val userName: String = "",
    
    @Column(name = "full_name")  // New column
    val fullName: String = ""
)

// STEP 2: New code reads from full_name, old code reads from user_name
// Deploy new application version

// STEP 3: Contract (Remove old column) - V5__Contract_remove_user_name.sql
/*
-- Only after ALL instances use new code
ALTER TABLE users DROP COLUMN user_name;
*/

// Pattern 2: Adding NOT NULL column to large table
// ❌ Bad: This locks the entire table
/*
ALTER TABLE orders ADD COLUMN tracking_number VARCHAR(100) NOT NULL DEFAULT '';
*/

// ✅ Good: Step-by-step
/*
-- V6__Add_tracking_number_nullable.sql
ALTER TABLE orders ADD COLUMN tracking_number VARCHAR(100);  -- Nullable first

-- V7__Backfill_tracking_number.sql (run in batches for large tables)
DO $$
DECLARE
    batch_size INT := 1000;
    offset_val INT := 0;
    rows_updated INT;
BEGIN
    LOOP
        UPDATE orders
        SET tracking_number = 'LEGACY-' || id::TEXT
        WHERE id IN (
            SELECT id FROM orders
            WHERE tracking_number IS NULL
            ORDER BY id
            LIMIT batch_size
        );
        
        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;
        
        PERFORM pg_sleep(0.01);  -- Small pause to reduce load
    END LOOP;
END $$;

-- V8__Set_tracking_number_not_null.sql (after backfill completes)
ALTER TABLE orders ALTER COLUMN tracking_number SET NOT NULL;
ALTER TABLE orders ALTER COLUMN tracking_number SET DEFAULT '';
*/
```

---

## Testing Migrations

```kotlin
// Test every migration with real database

@TestConfiguration
class TestFlywayConfig {
    
    @Bean
    fun testFlyway(dataSource: javax.sql.DataSource): Flyway {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration", "classpath:db/testdata")
            .cleanDisabled(false)  // Allow clean in tests
            .load()
            .also { flyway ->
                flyway.clean()    // Start fresh
                flyway.migrate()  // Apply all migrations
            }
    }
}

@SpringBootTest
@Testcontainers
class MigrationTest {
    
    companion object {
        @JvmField
        val postgres = PostgreSQLContainer<Nothing>("postgres:16-alpine").apply {
            withReuse(true)
        }
        
        init { postgres.start() }
        
        @JvmStatic
        @DynamicPropertySource
        fun configure(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
        }
    }
    
    @Autowired private lateinit var flyway: Flyway
    @Autowired private lateinit var jdbcTemplate: JdbcTemplate
    
    @Test
    fun `all migrations should apply successfully`() {
        val result = flyway.info()
        val failed = result.all().filter { it.state.isFailed }
        
        failed shouldBe emptyList()
    }
    
    @Test
    fun `users table should have required columns`() {
        val columns = jdbcTemplate.queryForList("""
            SELECT column_name, data_type, is_nullable
            FROM information_schema.columns
            WHERE table_name = 'users'
            ORDER BY ordinal_position
        """).map { it["column_name"] as String }
        
        columns shouldContain "id"
        columns shouldContain "email"
        columns shouldContain "name"
        columns shouldContain "password_hash"
        columns shouldContain "created_at"
        columns shouldContain "updated_at"
    }
    
    @Test
    fun `orders table should have required indexes`() {
        val indexes = jdbcTemplate.queryForList("""
            SELECT indexname FROM pg_indexes
            WHERE tablename = 'orders'
        """).map { it["indexname"] as String }
        
        indexes shouldContain "idx_orders_user"
        indexes shouldContain "idx_orders_status"
    }
    
    @Test
    fun `migration checksum should not change`() {
        // Detect if someone modified existing migration file
        val migrations = flyway.info().applied()
        
        for (migration in migrations) {
            migration.state shouldBe MigrationState.SUCCESS
        }
    }
}

typealias Testcontainers = org.testcontainers.junit.jupiter.Testcontainers
typealias SpringBootTest = org.springframework.boot.test.context.SpringBootTest
typealias TestConfiguration = org.springframework.boot.test.context.TestConfiguration
typealias Autowired = org.springframework.beans.factory.annotation.Autowired
typealias DynamicPropertySource = org.springframework.test.context.DynamicPropertySource
typealias DynamicPropertyRegistry = org.springframework.test.context.DynamicPropertyRegistry
typealias JdbcTemplate = org.springframework.jdbc.core.JdbcTemplate
typealias MigrationState = org.flywaydb.core.api.MigrationState
```

---

## แบบฝึกหัด

```sql
-- Exercise: เขียน migration สำหรับ feature ใหม่

-- Scenario: เพิ่ม product reviews feature
-- Table structure:
-- reviews: id, product_id, user_id, rating(1-5), title, body, status, created_at
-- product_ratings: product_id, avg_rating, review_count (computed/cached)

-- V9__Create_reviews_table.sql
-- TODO: สร้าง table ที่มี:
-- 1. Foreign keys ถูกต้อง
-- 2. CHECK constraint สำหรับ rating (1-5)
-- 3. ENUM สำหรับ status (PENDING, APPROVED, REJECTED)
-- 4. Indexes สำหรับ product_id, user_id, status
-- 5. Unique constraint: user สามารถ review product ได้แค่ครั้งเดียว

-- V10__Create_product_ratings_cache.sql
-- TODO: สร้าง materialized view หรือ table สำหรับ avg_rating
-- ควรจะ update อัตโนมัติเมื่อมี review ใหม่ (trigger)

-- Model:
-- 1 user → many reviews
-- 1 product → many reviews
-- 1 user + 1 product → max 1 review (unique)

CREATE TYPE review_status AS ENUM ('PENDING', 'APPROVED', 'REJECTED');

CREATE TABLE reviews (
    id          BIGSERIAL PRIMARY KEY,
    product_id  BIGINT NOT NULL REFERENCES products(id),
    user_id     BIGINT NOT NULL REFERENCES users(id),
    rating      SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    title       VARCHAR(255),
    body        TEXT,
    status      review_status NOT NULL DEFAULT 'PENDING',
    helpful_count INTEGER NOT NULL DEFAULT 0,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    
    UNIQUE(product_id, user_id)  -- One review per user per product
);

CREATE INDEX idx_reviews_product ON reviews(product_id);
CREATE INDEX idx_reviews_user ON reviews(user_id);
CREATE INDEX idx_reviews_status ON reviews(product_id, status) WHERE status = 'APPROVED';

-- Trigger to update product rating cache
CREATE OR REPLACE FUNCTION update_product_rating()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE products
    SET avg_rating = (
        SELECT AVG(rating)::DECIMAL(3,2)
        FROM reviews
        WHERE product_id = NEW.product_id AND status = 'APPROVED'
    ),
    review_count = (
        SELECT COUNT(*)
        FROM reviews
        WHERE product_id = NEW.product_id AND status = 'APPROVED'
    )
    WHERE id = NEW.product_id;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Note: Run "ALTER TABLE products ADD COLUMN avg_rating..." in separate migration first
```

---

## สรุป Part 75

```
✅ Migration Fundamentals: versioned, repeatable, baseline
✅ Flyway: SQL-first, strict checksum validation
✅ FlywayCallback: before/after migration hooks
✅ V{version}__{description}.sql: naming convention
✅ R__{description}.sql: repeatable migrations
✅ cleanDisabled=true: production safety
✅ Expand-Migrate-Contract: zero-downtime column rename
✅ Nullable first: add NOT NULL column to large table
✅ Batch backfill: update large tables incrementally
✅ pg_sleep: reduce load during migration
✅ Migration testing: real PostgreSQL via Testcontainers
✅ checksum: detect unauthorized migration file changes
✅ information_schema: verify column structure in tests
✅ ENUM type: review status (PENDING/APPROVED/REJECTED)
✅ CHECK constraint: rating BETWEEN 1 AND 5
✅ UNIQUE(product_id, user_id): one review per user
✅ Trigger: auto-update product rating cache
✅ Generated column: subtotal = quantity * unit_price
✅ GIN index: full-text search on product name
```

---

*Part 75/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
