# Part 46: Database Migrations ด้วย Flyway และ Liquibase

## สารบัญ
1. [Flyway Basics](#flyway-basics)
2. [Migration Scripts](#migration-scripts)
3. [Repeatable Migrations](#repeatable-migrations)
4. [Undo Migrations](#undo-migrations)
5. [Liquibase](#liquibase)
6. [Testing Migrations](#testing-migrations)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Flyway Basics

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")
    runtimeOnly("org.postgresql:postgresql")
}
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    baseline-version: 0
    validate-on-migrate: true
    out-of-order: false
    schemas: public
    table: flyway_schema_history
    # For multiple environments:
    placeholders:
      env: ${SPRING_PROFILES_ACTIVE:local}
```

---

## Migration Scripts

```sql
-- src/main/resources/db/migration/V1__create_users_table.sql
CREATE TABLE users (
    id          VARCHAR(36)  NOT NULL,
    username    VARCHAR(50)  NOT NULL,
    email       VARCHAR(255) NOT NULL,
    first_name  VARCHAR(100) NOT NULL,
    last_name   VARCHAR(100) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role        VARCHAR(20)  NOT NULL DEFAULT 'USER',
    active      BOOLEAN      NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT pk_users PRIMARY KEY (id),
    CONSTRAINT uq_users_username UNIQUE (username),
    CONSTRAINT uq_users_email UNIQUE (email),
    CONSTRAINT chk_users_role CHECK (role IN ('USER', 'MODERATOR', 'ADMIN'))
);

CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_role ON users (role);
CREATE INDEX idx_users_created_at ON users (created_at DESC);
```

```sql
-- src/main/resources/db/migration/V2__create_products_table.sql
CREATE TABLE categories (
    id          VARCHAR(36)  NOT NULL,
    name        VARCHAR(100) NOT NULL,
    slug        VARCHAR(100) NOT NULL,
    description TEXT,
    parent_id   VARCHAR(36),
    active      BOOLEAN      NOT NULL DEFAULT TRUE,
    
    CONSTRAINT pk_categories PRIMARY KEY (id),
    CONSTRAINT uq_categories_slug UNIQUE (slug),
    CONSTRAINT fk_categories_parent FOREIGN KEY (parent_id) 
        REFERENCES categories(id) ON DELETE SET NULL
);

CREATE TABLE products (
    id              VARCHAR(36)     NOT NULL,
    name            VARCHAR(200)    NOT NULL,
    slug            VARCHAR(200)    NOT NULL,
    description     TEXT,
    price           DECIMAL(12,2)   NOT NULL,
    stock_quantity  INTEGER         NOT NULL DEFAULT 0,
    category_id     VARCHAR(36),
    sku             VARCHAR(100),
    active          BOOLEAN         NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT pk_products PRIMARY KEY (id),
    CONSTRAINT uq_products_slug UNIQUE (slug),
    CONSTRAINT uq_products_sku UNIQUE (sku),
    CONSTRAINT chk_products_price CHECK (price > 0),
    CONSTRAINT chk_products_stock CHECK (stock_quantity >= 0),
    CONSTRAINT fk_products_category FOREIGN KEY (category_id)
        REFERENCES categories(id) ON DELETE SET NULL
);

CREATE INDEX idx_products_category ON products (category_id);
CREATE INDEX idx_products_price ON products (price);
CREATE INDEX idx_products_active ON products (active) WHERE active = TRUE;
```

```sql
-- src/main/resources/db/migration/V3__create_orders_table.sql
CREATE TYPE order_status AS ENUM (
    'PENDING', 'CONFIRMED', 'PROCESSING', 'SHIPPED', 'DELIVERED', 'CANCELLED', 'REFUNDED'
);

CREATE TABLE orders (
    id              VARCHAR(36)     NOT NULL,
    user_id         VARCHAR(36)     NOT NULL,
    status          order_status    NOT NULL DEFAULT 'PENDING',
    total_amount    DECIMAL(12,2)   NOT NULL,
    shipping_address JSONB          NOT NULL,
    notes           TEXT,
    created_at      TIMESTAMPTZ     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      TIMESTAMPTZ     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT pk_orders PRIMARY KEY (id),
    CONSTRAINT fk_orders_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE RESTRICT
);

CREATE TABLE order_items (
    id          VARCHAR(36)     NOT NULL,
    order_id    VARCHAR(36)     NOT NULL,
    product_id  VARCHAR(36)     NOT NULL,
    quantity    INTEGER         NOT NULL,
    unit_price  DECIMAL(12,2)   NOT NULL,
    
    CONSTRAINT pk_order_items PRIMARY KEY (id),
    CONSTRAINT fk_order_items_order FOREIGN KEY (order_id)
        REFERENCES orders(id) ON DELETE CASCADE,
    CONSTRAINT fk_order_items_product FOREIGN KEY (product_id)
        REFERENCES products(id) ON DELETE RESTRICT,
    CONSTRAINT chk_order_items_qty CHECK (quantity > 0),
    CONSTRAINT chk_order_items_price CHECK (unit_price > 0)
);

CREATE INDEX idx_orders_user ON orders (user_id);
CREATE INDEX idx_orders_status ON orders (status);
CREATE INDEX idx_orders_created_at ON orders (created_at DESC);
CREATE INDEX idx_order_items_order ON order_items (order_id);
```

```sql
-- src/main/resources/db/migration/V4__add_audit_trigger.sql
-- Auto-update updated_at column

CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_products_updated_at
    BEFORE UPDATE ON products
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_orders_updated_at
    BEFORE UPDATE ON orders
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

```sql
-- src/main/resources/db/migration/V5__add_full_text_search.sql
-- PostgreSQL full-text search

ALTER TABLE products 
    ADD COLUMN search_vector TSVECTOR;

UPDATE products SET search_vector = 
    setweight(to_tsvector('english', COALESCE(name, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(description, '')), 'B');

CREATE INDEX idx_products_search ON products USING GIN (search_vector);

-- Trigger to auto-update search vector
CREATE OR REPLACE FUNCTION products_search_vector_update() RETURNS TRIGGER AS $$
BEGIN
    NEW.search_vector := 
        setweight(to_tsvector('english', COALESCE(NEW.name, '')), 'A') ||
        setweight(to_tsvector('english', COALESCE(NEW.description, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER products_search_vector_trigger
    BEFORE INSERT OR UPDATE ON products
    FOR EACH ROW EXECUTE FUNCTION products_search_vector_update();
```

---

## Repeatable Migrations

```sql
-- src/main/resources/db/migration/R__create_views.sql
-- Repeatable migration: re-run when checksum changes

CREATE OR REPLACE VIEW product_summary AS
SELECT 
    p.id,
    p.name,
    p.slug,
    p.price,
    p.stock_quantity,
    c.name AS category_name,
    p.active,
    COUNT(oi.id) AS total_sold,
    SUM(oi.quantity) AS total_quantity_sold,
    p.created_at
FROM products p
LEFT JOIN categories c ON p.category_id = c.id
LEFT JOIN order_items oi ON p.id = oi.product_id
GROUP BY p.id, p.name, p.slug, p.price, p.stock_quantity, c.name, p.active, p.created_at;

CREATE OR REPLACE VIEW user_order_stats AS
SELECT
    u.id AS user_id,
    u.username,
    u.email,
    COUNT(o.id) AS total_orders,
    SUM(o.total_amount) AS total_spent,
    MAX(o.created_at) AS last_order_date
FROM users u
LEFT JOIN orders o ON u.id = o.user_id AND o.status != 'CANCELLED'
GROUP BY u.id, u.username, u.email;
```

---

## Java-based Migrations

```kotlin
// src/main/resources/db/migration/V6__seed_admin_user.kt
// Can also use Java/Kotlin for complex logic

@Component
class V6__seed_admin_user(
    private val passwordEncoder: PasswordEncoder
) : JavaMigration {
    
    override fun getVersion() = MigrationVersion.fromVersion("6")
    
    override fun getDescription() = "Seed admin user"
    
    override fun getChecksum(): Int? = null
    
    override fun isUndo() = false
    
    override fun isBaselineMigration() = false
    
    override fun migrate(context: Context) {
        val adminPassword = System.getenv("ADMIN_INITIAL_PASSWORD") 
            ?: throw IllegalStateException("ADMIN_INITIAL_PASSWORD environment variable required")
        
        val hash = passwordEncoder.encode(adminPassword)
        val id = java.util.UUID.randomUUID().toString()
        val now = java.time.OffsetDateTime.now()
        
        context.connection.prepareStatement("""
            INSERT INTO users (id, username, email, first_name, last_name, password_hash, role, created_at, updated_at)
            VALUES (?, 'admin', 'admin@myapp.com', 'Admin', 'User', ?, 'ADMIN', ?, ?)
            ON CONFLICT (username) DO NOTHING
        """.trimIndent()).use { stmt ->
            stmt.setString(1, id)
            stmt.setString(2, hash)
            stmt.setObject(3, now)
            stmt.setObject(4, now)
            stmt.executeUpdate()
        }
    }
}
```

---

## Testing Migrations

```kotlin
// Test that migrations run correctly
@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class DatabaseMigrationTest {
    
    companion object {
        @Container
        val postgres = PostgreSQLContainer<Nothing>("postgres:16-alpine").apply {
            withDatabaseName("testdb")
            withUsername("testuser")
            withPassword("testpass")
        }
        
        @JvmStatic
        @DynamicPropertySource
        fun overrideProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
        }
    }
    
    @Autowired
    lateinit var flywayMigrations: Flyway
    
    @Autowired
    lateinit var dataSource: DataSource
    
    @Test
    fun `all migrations should succeed`() {
        val info = flywayMigrations.info()
        val pending = info.pending()
        assertTrue(pending.isEmpty(), "There should be no pending migrations after test setup")
        
        val applied = info.applied()
        assertTrue(applied.isNotEmpty(), "At least one migration should be applied")
    }
    
    @Test
    fun `users table should have correct structure`() {
        dataSource.connection.use { conn ->
            val rs = conn.metaData.getColumns(null, null, "users", null)
            val columns = mutableSetOf<String>()
            while (rs.next()) columns.add(rs.getString("COLUMN_NAME"))
            
            assertTrue("id" in columns)
            assertTrue("username" in columns)
            assertTrue("email" in columns)
            assertTrue("role" in columns)
            assertTrue("created_at" in columns)
        }
    }
    
    @Test
    fun `can insert and query user`() {
        dataSource.connection.use { conn ->
            val id = java.util.UUID.randomUUID().toString()
            conn.prepareStatement("""
                INSERT INTO users (id, username, email, first_name, last_name, password_hash, role)
                VALUES (?, ?, ?, ?, ?, ?, ?)
            """).use { stmt ->
                stmt.setString(1, id)
                stmt.setString(2, "testuser_${id.take(8)}")
                stmt.setString(3, "test_${id.take(8)}@example.com")
                stmt.setString(4, "Test")
                stmt.setString(5, "User")
                stmt.setString(6, "hashed_password")
                stmt.setString(7, "USER")
                stmt.executeUpdate()
            }
            
            conn.prepareStatement("SELECT COUNT(*) FROM users WHERE id = ?").use { stmt ->
                stmt.setString(1, id)
                val rs = stmt.executeQuery()
                rs.next()
                assertEquals(1, rs.getInt(1))
            }
        }
    }
}

// Placeholder types
typealias Flyway = org.flywaydb.core.Flyway
typealias DataSource = javax.sql.DataSource
typealias JavaMigration = org.flywaydb.core.api.migration.JavaMigration
typealias MigrationVersion = org.flywaydb.core.api.MigrationVersion
typealias Context = org.flywaydb.core.api.migration.Context
typealias PasswordEncoder = org.springframework.security.crypto.password.PasswordEncoder
typealias PostgreSQLContainer<T> = org.testcontainers.containers.PostgreSQLContainer<T>
typealias DynamicPropertyRegistry = org.springframework.test.context.DynamicPropertyRegistry
```

---

## แบบฝึกหัด

```sql
-- Exercise: Design migrations for a social media platform

-- TODO: Create migrations for:
-- 1. V7: posts table (title, content, slug, author_id, published_at, tags JSONB)
-- 2. V8: comments table (post_id, author_id, content, parent_id for nested comments)
-- 3. V9: likes table (user_id, target_type ENUM, target_id, created_at)
-- 4. V10: followers table (follower_id, following_id, created_at)
-- 5. V11: Add full-text search to posts (search_vector TSVECTOR)
-- 6. V12: Add indexes for common query patterns
-- 7. R__posts_stats_view: View with post count, like count, comment count per user

-- Hints:
-- Use ENUM for target_type in likes: 'POST', 'COMMENT'
-- Add unique constraint on followers to prevent duplicate follows
-- Add partial index on posts WHERE published_at IS NOT NULL for published posts
-- Consider using pg_trgm extension for LIKE search

-- V7__create_posts_table.sql
-- TODO: Write this migration

-- V8__create_comments_table.sql
-- CREATE TABLE comments (
--     id          VARCHAR(36)  NOT NULL,
--     post_id     VARCHAR(36)  NOT NULL,
--     author_id   VARCHAR(36)  NOT NULL,
--     parent_id   VARCHAR(36),  -- for nested comments
--     content     TEXT         NOT NULL,
--     created_at  TIMESTAMPTZ  NOT NULL DEFAULT CURRENT_TIMESTAMP,
--     CONSTRAINT pk_comments PRIMARY KEY (id)
-- );
-- TODO: Add foreign keys and indexes
```

---

## สรุป Part 46

```
✅ Flyway: version-based database migration tool
✅ Naming convention: V{version}__{description}.sql
✅ Repeatable: R__{description}.sql (re-runs on change)
✅ Undo: U{version}__{description}.sql (Flyway Teams)
✅ Java/Kotlin migrations: complex logic in code
✅ baseline-on-migrate: migrate existing databases
✅ validate-on-migrate: verify checksums
✅ Testcontainers: integration test with real PostgreSQL
✅ Schema history: flyway_schema_history table
✅ DDL patterns: constraints, indexes, triggers, views
✅ JSONB: flexible schema within PostgreSQL
✅ Full-text search: tsvector + GIN index
```

---

*Part 46/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
