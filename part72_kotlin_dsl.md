# Part 72: Kotlin DSL Design Patterns

## สารบัญ
1. [DSL คืออะไร](#dsl-คืออะไร)
2. [Type-Safe Builders](#type-safe-builders)
3. [HTML DSL](#html-dsl)
4. [SQL Query DSL](#sql-query-dsl)
5. [Configuration DSL](#configuration-dsl)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## DSL คืออะไร

```
Domain-Specific Language (DSL) คือ mini-language สำหรับ domain เฉพาะ

Internal DSL (เขียนใน Kotlin):
- ใช้ Kotlin syntax ที่ flexible
- ไม่ต้องเขียน parser เอง
- Type-safe (compile-time checking)
- IDE support เต็มที่

เทคนิคหลักใน Kotlin DSL:
1. Lambda with receiver: { block: Builder.() -> Unit }
2. Extension functions: fun String.bold() = "<b>$this</b>"
3. Operator overloading: div, unaryPlus, invoke
4. Infix functions: infix fun String.shouldBe(expected: String)
5. @DslMarker: ป้องกัน implicit access ใน nested scope

ตัวอย่าง DSL ที่คุ้นเคย:
- Gradle build.gradle.kts
- Ktor routing { get("/") { ... } }
- Exposed SQL framework
- Kotest shouldBe assertions
```

---

## Type-Safe Builders

```kotlin
// HTML DSL แบบ type-safe

@DslMarker
annotation class HtmlDsl

@HtmlDsl
class HtmlBuilder {
    private val elements = mutableListOf<HtmlElement>()
    
    fun head(block: HeadBuilder.() -> Unit) {
        elements.add(HeadBuilder().apply(block).build())
    }
    
    fun body(block: BodyBuilder.() -> Unit) {
        elements.add(BodyBuilder().apply(block).build())
    }
    
    fun build(): String = buildString {
        append("<!DOCTYPE html>\n<html>\n")
        elements.forEach { append(it.render()) }
        append("</html>")
    }
}

@HtmlDsl
class HeadBuilder {
    private val elements = mutableListOf<HtmlElement>()
    
    var title: String = ""
    
    fun meta(name: String, content: String) {
        elements.add(SimpleElement("meta", mapOf("name" to name, "content" to content), selfClosing = true))
    }
    
    fun link(rel: String, href: String) {
        elements.add(SimpleElement("link", mapOf("rel" to rel, "href" to href), selfClosing = true))
    }
    
    fun build(): HtmlElement = object : HtmlElement {
        override fun render() = buildString {
            append("<head>\n")
            if (title.isNotEmpty()) append("  <title>$title</title>\n")
            elements.forEach { append("  ${it.render()}\n") }
            append("</head>\n")
        }
    }
}

@HtmlDsl
class BodyBuilder {
    private val elements = mutableListOf<HtmlElement>()
    
    fun div(id: String? = null, className: String? = null, block: DivBuilder.() -> Unit) {
        elements.add(DivBuilder(id, className).apply(block).build())
    }
    
    fun h1(content: String) = elements.add(SimpleElement("h1", content = content))
    fun h2(content: String) = elements.add(SimpleElement("h2", content = content))
    fun p(content: String) = elements.add(SimpleElement("p", content = content))
    
    fun ul(block: ListBuilder.() -> Unit) {
        elements.add(ListBuilder("ul").apply(block).build())
    }
    
    fun build(): HtmlElement = object : HtmlElement {
        override fun render() = buildString {
            append("<body>\n")
            elements.forEach { append(it.render()) }
            append("</body>\n")
        }
    }
}

@HtmlDsl
class DivBuilder(private val id: String?, private val className: String?) {
    private val elements = mutableListOf<HtmlElement>()
    private val attributes = mutableMapOf<String, String>()
    
    fun p(content: String) = elements.add(SimpleElement("p", content = content))
    fun span(content: String) = elements.add(SimpleElement("span", content = content))
    fun a(href: String, text: String) = elements.add(
        SimpleElement("a", mapOf("href" to href), content = text)
    )
    
    fun build(): HtmlElement = object : HtmlElement {
        override fun render() = buildString {
            val attrs = buildString {
                id?.let { append(""" id="$it"""") }
                className?.let { append(""" class="$it"""") }
                attributes.forEach { (k, v) -> append(""" $k="$v"""") }
            }
            append("<div$attrs>\n")
            elements.forEach { append("  ${it.render()}\n") }
            append("</div>\n")
        }
    }
}

@HtmlDsl
class ListBuilder(private val tag: String) {
    private val items = mutableListOf<String>()
    
    operator fun String.unaryPlus() {
        items.add(this)
    }
    
    fun build(): HtmlElement = object : HtmlElement {
        override fun render() = buildString {
            append("<$tag>\n")
            items.forEach { append("  <li>$it</li>\n") }
            append("</$tag>\n")
        }
    }
}

interface HtmlElement {
    fun render(): String
}

class SimpleElement(
    private val tag: String,
    private val attrs: Map<String, String> = emptyMap(),
    private val content: String = "",
    private val selfClosing: Boolean = false
) : HtmlElement {
    override fun render(): String {
        val attrStr = attrs.entries.joinToString(" ") { (k, v) -> """$k="$v"""" }
        val openTag = if (attrStr.isEmpty()) "<$tag>" else "<$tag $attrStr>"
        
        return if (selfClosing) "$openTag" else "$openTag$content</$tag>"
    }
}

// Entry point function
fun html(block: HtmlBuilder.() -> Unit): String {
    return HtmlBuilder().apply(block).build()
}

// Usage
fun buildProductPage(products: List<Product>): String {
    return html {
        head {
            title = "Product Catalog"
            meta("viewport", "width=device-width, initial-scale=1.0")
            link("stylesheet", "/css/style.css")
        }
        body {
            h1("Product Catalog")
            div(id = "products", className = "product-grid") {
                products.forEach { product ->
                    p("${product.name} - ${product.price} THB")
                }
            }
            ul {
                +"Item 1"
                +"Item 2"
                +"Item 3"
            }
        }
    }
}

data class Product(val id: Long, val name: String, val price: Double)
```

---

## SQL Query DSL

```kotlin
// Type-safe SQL query builder

@DslMarker
annotation class QueryDsl

// Column reference
data class Column<T>(val tableName: String, val columnName: String) {
    
    infix fun eq(value: T): Condition = Condition("$tableName.$columnName = ?", listOf(value as Any))
    infix fun neq(value: T): Condition = Condition("$tableName.$columnName != ?", listOf(value as Any))
    infix fun gt(value: T): Condition = Condition("$tableName.$columnName > ?", listOf(value as Any))
    infix fun gte(value: T): Condition = Condition("$tableName.$columnName >= ?", listOf(value as Any))
    infix fun lt(value: T): Condition = Condition("$tableName.$columnName < ?", listOf(value as Any))
    infix fun lte(value: T): Condition = Condition("$tableName.$columnName <= ?", listOf(value as Any))
    infix fun like(pattern: String): Condition = Condition("$tableName.$columnName LIKE ?", listOf(pattern))
    fun isNull(): Condition = Condition("$tableName.$columnName IS NULL", emptyList())
    fun isNotNull(): Condition = Condition("$tableName.$columnName IS NOT NULL", emptyList())
    infix fun inList(values: List<T>): Condition {
        val placeholders = values.joinToString(", ") { "?" }
        return Condition("$tableName.$columnName IN ($placeholders)", values.map { it as Any })
    }
    
    fun asc() = OrderBy("$tableName.$columnName ASC")
    fun desc() = OrderBy("$tableName.$columnName DESC")
    
    override fun toString() = "$tableName.$columnName"
}

data class Condition(val sql: String, val params: List<Any>) {
    infix fun and(other: Condition) = Condition(
        "(${this.sql}) AND (${other.sql})",
        this.params + other.params
    )
    
    infix fun or(other: Condition) = Condition(
        "(${this.sql}) OR (${other.sql})",
        this.params + other.params
    )
}

data class OrderBy(val sql: String)

// Query builder
@QueryDsl
class SelectBuilder(private val tableName: String) {
    private val columns = mutableListOf<String>()
    private val joins = mutableListOf<String>()
    private var whereCondition: Condition? = null
    private val orderBys = mutableListOf<OrderBy>()
    private var limitValue: Int? = null
    private var offsetValue: Int? = null
    
    fun select(vararg cols: Column<*>) = apply {
        columns.addAll(cols.map { it.toString() })
    }
    
    fun selectAll() = apply { columns.add("*") }
    
    fun innerJoin(otherTable: String, on: String) = apply {
        joins.add("INNER JOIN $otherTable ON $on")
    }
    
    fun leftJoin(otherTable: String, on: String) = apply {
        joins.add("LEFT JOIN $otherTable ON $on")
    }
    
    fun where(condition: Condition) = apply {
        whereCondition = condition
    }
    
    fun orderBy(vararg orders: OrderBy) = apply {
        orderBys.addAll(orders)
    }
    
    fun limit(n: Int) = apply { limitValue = n }
    fun offset(n: Int) = apply { offsetValue = n }
    
    fun build(): Query {
        val colList = if (columns.isEmpty()) "*" else columns.joinToString(", ")
        
        val sql = buildString {
            append("SELECT $colList FROM $tableName")
            if (joins.isNotEmpty()) append(" ${joins.joinToString(" ")}")
            whereCondition?.let { append(" WHERE ${it.sql}") }
            if (orderBys.isNotEmpty()) append(" ORDER BY ${orderBys.joinToString(", ") { it.sql }}")
            limitValue?.let { append(" LIMIT $it") }
            offsetValue?.let { append(" OFFSET $it") }
        }
        
        return Query(sql, whereCondition?.params ?: emptyList())
    }
}

data class Query(val sql: String, val params: List<Any>)

// Table definitions
object ProductsTable {
    const val TABLE = "products"
    val id = Column<Long>(TABLE, "id")
    val name = Column<String>(TABLE, "name")
    val price = Column<Double>(TABLE, "price")
    val categoryId = Column<Long>(TABLE, "category_id")
    val inStock = Column<Boolean>(TABLE, "in_stock")
    val createdAt = Column<java.time.LocalDate>(TABLE, "created_at")
}

object CategoriesTable {
    const val TABLE = "categories"
    val id = Column<Long>(TABLE, "id")
    val name = Column<String>(TABLE, "name")
}

fun select(tableName: String, block: SelectBuilder.() -> Unit): Query {
    return SelectBuilder(tableName).apply(block).build()
}

// Usage
fun queryExamples() {
    // Simple query
    val simpleQuery = select(ProductsTable.TABLE) {
        select(ProductsTable.name, ProductsTable.price)
        where(ProductsTable.inStock eq true)
        orderBy(ProductsTable.price.asc())
        limit(20)
    }
    println(simpleQuery.sql)
    // SELECT products.name, products.price FROM products WHERE (products.in_stock = ?) ORDER BY products.price ASC LIMIT 20
    
    // Complex query with join
    val complexQuery = select(ProductsTable.TABLE) {
        selectAll()
        leftJoin(
            CategoriesTable.TABLE,
            "${ProductsTable.categoryId} = ${CategoriesTable.id}"
        )
        where(
            (ProductsTable.price gte 100.0) and
            (ProductsTable.inStock eq true) and
            (ProductsTable.categoryId inList listOf(1L, 2L, 3L))
        )
        orderBy(ProductsTable.price.desc(), ProductsTable.name.asc())
        limit(50)
        offset(100)
    }
    println(complexQuery.sql)
    println(complexQuery.params)
}
```

---

## Configuration DSL

```kotlin
// Application configuration DSL

@DslMarker
annotation class ConfigDsl

@ConfigDsl
class ApplicationConfig {
    var name: String = ""
    var version: String = "1.0.0"
    var environment: Environment = Environment.DEVELOPMENT
    
    private var _database: DatabaseConfig? = null
    private var _cache: CacheConfig? = null
    private var _security: SecurityConfig? = null
    private var _server: ServerConfig? = null
    
    val database: DatabaseConfig get() = _database ?: error("Database not configured")
    val cache: CacheConfig get() = _cache ?: CacheConfig()
    val security: SecurityConfig get() = _security ?: SecurityConfig()
    val server: ServerConfig get() = _server ?: ServerConfig()
    
    fun database(block: DatabaseConfig.() -> Unit) {
        _database = DatabaseConfig().apply(block)
    }
    
    fun cache(block: CacheConfig.() -> Unit) {
        _cache = CacheConfig().apply(block)
    }
    
    fun security(block: SecurityConfig.() -> Unit) {
        _security = SecurityConfig().apply(block)
    }
    
    fun server(block: ServerConfig.() -> Unit) {
        _server = ServerConfig().apply(block)
    }
    
    fun validate() {
        require(name.isNotBlank()) { "Application name must not be blank" }
        _database?.validate() ?: error("Database configuration is required")
    }
}

@ConfigDsl
class DatabaseConfig {
    var host: String = "localhost"
    var port: Int = 5432
    var name: String = ""
    var username: String = ""
    var password: String = ""
    var maxPoolSize: Int = 10
    var minIdle: Int = 2
    var connectionTimeout: Long = 30000  // ms
    var ssl: Boolean = false
    
    val jdbcUrl: String
        get() = "jdbc:postgresql://$host:$port/$name"
    
    fun validate() {
        require(name.isNotBlank()) { "Database name is required" }
        require(username.isNotBlank()) { "Database username is required" }
        require(port in 1..65535) { "Invalid database port: $port" }
    }
}

@ConfigDsl
class CacheConfig {
    var type: CacheType = CacheType.CAFFEINE
    var ttlMinutes: Long = 60
    var maxSize: Long = 10_000
    
    // Redis specific
    var redisHost: String = "localhost"
    var redisPort: Int = 6379
    var redisPassword: String? = null
}

@ConfigDsl
class SecurityConfig {
    var jwtSecret: String = ""
    var jwtExpiryMinutes: Long = 60
    var jwtRefreshExpiryDays: Long = 7
    var bcryptStrength: Int = 12
    var allowedOrigins: List<String> = listOf("*")
    
    private val _rateLimits = mutableMapOf<String, RateLimit>()
    val rateLimits: Map<String, RateLimit> get() = _rateLimits
    
    fun rateLimit(endpoint: String, block: RateLimit.() -> Unit) {
        _rateLimits[endpoint] = RateLimit().apply(block)
    }
}

@ConfigDsl
class RateLimit {
    var requestsPerMinute: Int = 60
    var burstSize: Int = 10
}

@ConfigDsl
class ServerConfig {
    var port: Int = 8080
    var contextPath: String = "/"
    var shutdownGracePeriodSeconds: Int = 30
    var compressionEnabled: Boolean = true
    var accessLogEnabled: Boolean = true
}

enum class Environment { DEVELOPMENT, STAGING, PRODUCTION }
enum class CacheType { CAFFEINE, REDIS, NONE }

fun application(block: ApplicationConfig.() -> Unit): ApplicationConfig {
    return ApplicationConfig().apply(block).also { it.validate() }
}

// Usage
val config = application {
    name = "E-Commerce API"
    version = "2.0.0"
    environment = Environment.PRODUCTION
    
    database {
        host = "db.example.com"
        port = 5432
        name = "ecommerce"
        username = "app_user"
        password = System.getenv("DB_PASSWORD") ?: ""
        maxPoolSize = 20
        ssl = true
    }
    
    cache {
        type = CacheType.REDIS
        ttlMinutes = 30
        redisHost = "redis.example.com"
        redisPort = 6379
    }
    
    security {
        jwtSecret = System.getenv("JWT_SECRET") ?: ""
        jwtExpiryMinutes = 30
        jwtRefreshExpiryDays = 14
        allowedOrigins = listOf("https://myapp.com", "https://www.myapp.com")
        
        rateLimit("/api/auth/login") {
            requestsPerMinute = 10
            burstSize = 3
        }
        
        rateLimit("/api/products") {
            requestsPerMinute = 100
            burstSize = 20
        }
    }
    
    server {
        port = 8080
        shutdownGracePeriodSeconds = 60
        compressionEnabled = true
    }
}

println("App: ${config.name} v${config.version}")
println("DB: ${config.database.jdbcUrl}")
println("Cache: ${config.cache.type}")
```

---

## Router DSL

```kotlin
// Custom routing DSL สำหรับ HTTP endpoints

@DslMarker
annotation class RouterDsl

@RouterDsl
class RouterBuilder(private val prefix: String = "") {
    
    private val routes = mutableListOf<Route>()
    private val middleware = mutableListOf<Middleware>()
    
    fun get(path: String, handler: RouteHandler) {
        routes.add(Route("GET", prefix + path, middleware.toList(), handler))
    }
    
    fun post(path: String, handler: RouteHandler) {
        routes.add(Route("POST", prefix + path, middleware.toList(), handler))
    }
    
    fun put(path: String, handler: RouteHandler) {
        routes.add(Route("PUT", prefix + path, middleware.toList(), handler))
    }
    
    fun delete(path: String, handler: RouteHandler) {
        routes.add(Route("DELETE", prefix + path, middleware.toList(), handler))
    }
    
    // Nested route group
    fun group(path: String, block: RouterBuilder.() -> Unit) {
        val nested = RouterBuilder(prefix + path).apply(block)
        routes.addAll(nested.routes)
    }
    
    fun use(m: Middleware) {
        middleware.add(m)
    }
    
    fun build() = routes.toList()
}

typealias RouteHandler = suspend (Request) -> Response
typealias Middleware = suspend (Request, Next) -> Response
typealias Next = suspend (Request) -> Response

data class Route(
    val method: String,
    val path: String,
    val middleware: List<Middleware>,
    val handler: RouteHandler
)

data class Request(
    val method: String,
    val path: String,
    val headers: Map<String, String> = emptyMap(),
    val body: String? = null,
    val pathParams: Map<String, String> = emptyMap(),
    val queryParams: Map<String, String> = emptyMap()
)

data class Response(
    val status: Int,
    val body: String?,
    val headers: Map<String, String> = emptyMap()
)

fun router(block: RouterBuilder.() -> Unit): List<Route> {
    return RouterBuilder().apply(block).build()
}

// Common middleware
val loggingMiddleware: Middleware = { request, next ->
    val start = System.currentTimeMillis()
    val response = next(request)
    val duration = System.currentTimeMillis() - start
    println("[${request.method}] ${request.path} → ${response.status} (${duration}ms)")
    response
}

val authMiddleware: Middleware = { request, next ->
    val token = request.headers["Authorization"]
    if (token == null || !token.startsWith("Bearer ")) {
        Response(401, """{"error":"Unauthorized"}""")
    } else {
        next(request)
    }
}

// Usage
val routes = router {
    use(loggingMiddleware)
    
    get("/health") { Response(200, """{"status":"ok"}""") }
    
    group("/api/v1") {
        group("/products") {
            get("") { req ->
                Response(200, """{"products":[]}""")
            }
            
            get("/{id}") { req ->
                val id = req.pathParams["id"]
                Response(200, """{"id":"$id"}""")
            }
            
            use(authMiddleware)
            
            post("") { req ->
                Response(201, """{"created":true}""")
            }
            
            put("/{id}") { req ->
                Response(200, """{"updated":true}""")
            }
            
            delete("/{id}") { req ->
                Response(204, null)
            }
        }
        
        group("/users") {
            use(authMiddleware)
            get("/me") { Response(200, """{"user":"me"}""") }
        }
    }
}

routes.forEach { println("[${it.method}] ${it.path}") }
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Validation DSL

// สร้าง DSL สำหรับ validate objects แบบ type-safe

data class ValidationResult(
    val isValid: Boolean,
    val errors: List<String>
) {
    companion object {
        fun valid() = ValidationResult(true, emptyList())
        fun invalid(vararg errors: String) = ValidationResult(false, errors.toList())
    }
}

@DslMarker
annotation class ValidatorDsl

@ValidatorDsl
class Validator<T>(private val value: T) {
    private val errors = mutableListOf<String>()
    
    fun rule(message: String, check: (T) -> Boolean) {
        if (!check(value)) errors.add(message)
    }
    
    fun build() = if (errors.isEmpty()) {
        ValidationResult.valid()
    } else {
        ValidationResult(false, errors)
    }
}

fun <T> validate(value: T, block: Validator<T>.() -> Unit): ValidationResult {
    return Validator(value).apply(block).build()
}

// String validators
@ValidatorDsl
class StringValidator(private val value: String) {
    private val errors = mutableListOf<String>()
    
    fun notBlank(message: String = "Must not be blank") = apply {
        if (value.isBlank()) errors.add(message)
    }
    
    fun minLength(min: Int, message: String = "Must be at least $min characters") = apply {
        if (value.length < min) errors.add(message)
    }
    
    fun maxLength(max: Int, message: String = "Must be at most $max characters") = apply {
        if (value.length > max) errors.add(message)
    }
    
    fun matches(regex: Regex, message: String = "Invalid format") = apply {
        if (!value.matches(regex)) errors.add(message)
    }
    
    fun email() = matches(Regex("[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}"), "Invalid email format")
    
    fun build() = if (errors.isEmpty()) ValidationResult.valid()
                  else ValidationResult(false, errors)
}

fun validateString(value: String, block: StringValidator.() -> Unit): ValidationResult {
    return StringValidator(value).apply(block).build()
}

// Usage
fun validateUser(email: String, name: String, age: Int): ValidationResult {
    val emailResult = validateString(email) {
        notBlank("Email is required")
        email()
    }
    
    val nameResult = validateString(name) {
        notBlank("Name is required")
        minLength(2, "Name must be at least 2 characters")
        maxLength(100, "Name must be at most 100 characters")
    }
    
    val ageResult = validate(age) {
        rule("Age must be at least 18") { it >= 18 }
        rule("Age must be at most 120") { it <= 120 }
    }
    
    val allErrors = listOf(emailResult, nameResult, ageResult)
        .filter { !it.isValid }
        .flatMap { it.errors }
    
    return if (allErrors.isEmpty()) ValidationResult.valid()
           else ValidationResult(false, allErrors)
}

// Test it
fun main() {
    val result = validateUser("invalid-email", "J", 15)
    if (!result.isValid) {
        println("Validation failed:")
        result.errors.forEach { println("  - $it") }
    }
}
```

---

## สรุป Part 72

```
✅ DSL คืออะไร: internal DSL ใช้ Kotlin syntax
✅ @DslMarker: ป้องกัน implicit receiver access ใน nested scope
✅ Lambda with receiver: (Builder.() -> Unit) pattern
✅ HTML DSL: HtmlBuilder, HeadBuilder, BodyBuilder, DivBuilder
✅ unaryPlus operator: +"Item" สำหรับ list items
✅ Type-Safe Builders: compile-time checking ทุก element
✅ SQL Query DSL: Column<T> with infix functions
✅ infix fun eq/neq/gt/like: type-safe conditions
✅ and/or operators: compose conditions
✅ Table objects: ProductsTable.name, ProductsTable.price
✅ Configuration DSL: nested config blocks
✅ Validation in DSL: require() สำหรับ invariants
✅ Router DSL: get/post/put/delete/group/use
✅ Middleware: (Request, Next) -> Response
✅ Validation DSL: StringValidator with fluent API
✅ Composable validators: combine multiple validations
```

---

*Part 72/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
