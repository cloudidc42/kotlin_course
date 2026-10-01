# Part 96: Kotlin Advanced Language Features

## สารบัญ
1. [Coroutines ขั้นสูง](#coroutines-ขั้นสูง)
2. [Kotlin DSL Construction](#kotlin-dsl-construction)
3. [Delegated Properties](#delegated-properties)
4. [Type System ขั้นสูง](#type-system-ขั้นสูง)
5. [Metaprogramming & Reflection](#metaprogramming--reflection)
6. [Compiler Plugins](#compiler-plugins)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Coroutines ขั้นสูง

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.flow.*
import kotlin.coroutines.*

// Structured Concurrency
suspend fun fetchDashboard(userId: String): Dashboard {
    return coroutineScope {  // creates a scope; all children must complete
        val user = async { userService.getUser(userId) }
        val orders = async { orderService.getRecentOrders(userId, limit = 5) }
        val recommendations = async { 
            withTimeout(2000) {  // timeout for slow service
                mlService.getRecommendations(userId)
            }
        }
        val notifications = async { notificationService.getUnread(userId) }
        
        // All 4 run concurrently, wait for all
        Dashboard(
            user = user.await(),
            recentOrders = orders.await(),
            recommendations = try { recommendations.await() } catch (e: TimeoutCancellationException) { emptyList() },
            unreadCount = notifications.await().size
        )
    }
}

// SupervisorScope: one child fails, others continue
suspend fun processAllOrders(orderIds: List<String>): Map<String, Result<ProcessedOrder>> {
    return supervisorScope {
        orderIds.associate { orderId ->
            orderId to async {
                runCatching { orderProcessor.process(orderId) }
            }
        }.mapValues { (_, deferred) -> deferred.await() }
    }
}

// Channel: communication between coroutines
fun CoroutineScope.produceNumbers(limit: Int): ReceiveChannel<Int> = produce {
    for (i in 1..limit) {
        send(i)
        delay(100)
    }
}

suspend fun pipelineExample() = coroutineScope {
    val numbers = produceNumbers(10)
    
    val squared = produce<Int> {
        for (n in numbers) send(n * n)
    }
    
    val filtered = produce<Int> {
        for (n in squared) if (n > 25) send(n)
    }
    
    for (n in filtered) println(n)
}

// StateFlow vs SharedFlow
class StockViewModel(private val stockService: StockService) {
    
    private val _stockLevel = MutableStateFlow(0)
    val stockLevel: StateFlow<Int> = _stockLevel.asStateFlow()
    
    private val _events = MutableSharedFlow<StockEvent>(extraBufferCapacity = 64)
    val events: SharedFlow<StockEvent> = _events.asSharedFlow()
    
    fun observeStock(productId: String): Job {
        return CoroutineScope(Dispatchers.IO).launch {
            stockService.observe(productId).collect { level ->
                _stockLevel.value = level
                if (level == 0) {
                    _events.emit(StockEvent.OutOfStock(productId))
                } else if (level < 10) {
                    _events.emit(StockEvent.LowStock(productId, level))
                }
            }
        }
    }
}

sealed class StockEvent {
    data class OutOfStock(val productId: String) : StockEvent()
    data class LowStock(val productId: String, val level: Int) : StockEvent()
}

// Custom CoroutineContext Element
class UserContext(val userId: String, val tenantId: String) : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<UserContext>
    override val key: CoroutineContext.Key<*> get() = Key
}

suspend fun getCurrentUser(): UserContext? = coroutineContext[UserContext]

suspend fun withUser(userId: String, tenantId: String, block: suspend () -> Unit) {
    withContext(UserContext(userId, tenantId)) {
        block()
    }
}

// Usage
suspend fun exampleUsage() {
    withUser("user-1", "tenant-1") {
        val user = getCurrentUser()  // returns UserContext("user-1", "tenant-1")
        println("Processing for: ${user?.userId}")
    }
}

// Flow operators advanced
fun <T> Flow<T>.retryWithExponentialBackoff(
    maxAttempts: Int = 3,
    initialDelay: Long = 100,
    maxDelay: Long = 10_000
): Flow<T> = flow {
    var attempt = 0
    var delay = initialDelay
    
    while (attempt < maxAttempts) {
        try {
            collect { emit(it) }
            return@flow  // success
        } catch (e: Exception) {
            attempt++
            if (attempt >= maxAttempts) throw e
            
            delay(delay)
            delay = minOf(delay * 2, maxDelay)
        }
    }
}

// Cooperative cancellation
suspend fun longRunningTask() {
    for (i in 1..1000) {
        ensureActive()  // check if coroutine was cancelled
        doWork(i)
    }
}

fun doWork(i: Int) { Thread.sleep(10) }

// CoroutineExceptionHandler
val handler = CoroutineExceptionHandler { _, exception ->
    println("Unhandled exception: $exception")
}

val scope = CoroutineScope(Dispatchers.IO + handler)

data class Dashboard(val user: Any, val recentOrders: List<Any>, val recommendations: List<Any>, val unreadCount: Int)
data class ProcessedOrder(val id: String)
interface userService { companion object { suspend fun getUser(id: String): Any = TODO() } }
interface orderService { companion object { suspend fun getRecentOrders(id: String, limit: Int): List<Any> = TODO() } }
interface mlService { companion object { suspend fun getRecommendations(id: String): List<Any> = TODO() } }
interface notificationService { companion object { suspend fun getUnread(id: String): List<Any> = TODO() } }
interface orderProcessor { companion object { suspend fun process(id: String): ProcessedOrder = TODO() } }
interface StockService { fun observe(productId: String): Flow<Int> }
```

---

## Kotlin DSL Construction

```kotlin
// ออกแบบ Type-safe DSL สำหรับ API routes

@DslMarker
annotation class RouterDsl

@RouterDsl
class RouterBuilder {
    private val routes = mutableListOf<Route>()
    
    fun get(path: String, handler: suspend (Request) -> Response) {
        routes.add(Route("GET", path, handler))
    }
    
    fun post(path: String, handler: suspend (Request) -> Response) {
        routes.add(Route("POST", path, handler))
    }
    
    fun put(path: String, handler: suspend (Request) -> Response) {
        routes.add(Route("PUT", path, handler))
    }
    
    fun delete(path: String, handler: suspend (Request) -> Response) {
        routes.add(Route("DELETE", path, handler))
    }
    
    fun route(prefix: String, block: RouterBuilder.() -> Unit) {
        val nested = RouterBuilder().apply(block)
        nested.routes.forEach { route ->
            routes.add(route.copy(path = prefix + route.path))
        }
    }
    
    fun build(): List<Route> = routes.toList()
}

fun router(block: RouterBuilder.() -> Unit): List<Route> =
    RouterBuilder().apply(block).build()

// Usage
val routes = router {
    get("/health") { Response.ok("OK") }
    
    route("/api/v1") {
        route("/products") {
            get("") { req -> Response.ok("list") }
            get("/{id}") { req -> Response.ok("get ${req.pathParam("id")}") }
            post("") { req -> Response.ok("create", 201) }
            put("/{id}") { req -> Response.ok("update") }
            delete("/{id}") { req -> Response.ok("delete", 204) }
        }
        
        route("/orders") {
            get("") { Response.ok("orders") }
            post("") { Response.ok("order created", 201) }
        }
    }
}

// HTML DSL
@DslMarker
annotation class HtmlDsl

@HtmlDsl
abstract class Tag(val name: String) {
    private val children = mutableListOf<Tag>()
    private val attributes = mutableMapOf<String, String>()
    
    var id: String by attributes
    var className: String
        get() = attributes["class"] ?: ""
        set(v) { attributes["class"] = v }
    
    operator fun String.unaryPlus() {
        children.add(TextNode(this))
    }
    
    fun <T : Tag> initTag(tag: T, block: T.() -> Unit): T {
        tag.apply(block)
        children.add(tag)
        return tag
    }
    
    fun render(indent: Int = 0): String {
        val attrs = attributes.entries.joinToString(" ") { (k, v) -> "$k=\"$v\"" }
        val attrsStr = if (attrs.isNotEmpty()) " $attrs" else ""
        val childrenHtml = children.joinToString("\n") { it.render(indent + 2) }
        val pad = " ".repeat(indent)
        
        return if (children.isEmpty()) "$pad<$name$attrsStr/>"
        else "$pad<$name$attrsStr>\n$childrenHtml\n$pad</$name>"
    }
}

class TextNode(val text: String) : Tag("text") {
    override fun render(indent: Int) = " ".repeat(indent) + text
}

class Div : Tag("div")
class H1 : Tag("h1")
class P : Tag("p")
class Ul : Tag("ul")
class Li : Tag("li")

fun div(block: Div.() -> Unit) = Div().apply(block)
fun Div.h1(block: H1.() -> Unit) = initTag(H1(), block)
fun Div.p(block: P.() -> Unit) = initTag(P(), block)
fun Div.ul(block: Ul.() -> Unit) = initTag(Ul(), block)
fun Ul.li(block: Li.() -> Unit) = initTag(Li(), block)
private operator fun MutableMap<String, String>.setValue(thisRef: Tag, property: kotlin.reflect.KProperty<*>, value: String) { this[property.name] = value }
private operator fun MutableMap<String, String>.getValue(thisRef: Tag, property: kotlin.reflect.KProperty<*>): String = this[property.name] ?: ""

val html = div {
    id = "container"
    className = "main-content"
    h1 { +"Product List" }
    p { +"Total: 42 products" }
    ul {
        li { +"MacBook Pro" }
        li { +"iPhone 15" }
        li { +"iPad Air" }
    }
}

// Query DSL (type-safe SQL-like)
class QueryBuilder<T> {
    private var table = ""
    private val conditions = mutableListOf<String>()
    private var limitVal: Int? = null
    private var offsetVal: Int? = null
    private val orderByList = mutableListOf<String>()
    
    fun from(tableName: String): QueryBuilder<T> = apply { table = tableName }
    
    fun where(condition: String): QueryBuilder<T> = apply { conditions.add(condition) }
    
    fun limit(n: Int): QueryBuilder<T> = apply { limitVal = n }
    
    fun offset(n: Int): QueryBuilder<T> = apply { offsetVal = n }
    
    fun orderBy(column: String, desc: Boolean = false): QueryBuilder<T> = apply {
        orderByList.add("$column ${if (desc) "DESC" else "ASC"}")
    }
    
    fun build(): String {
        val where = if (conditions.isNotEmpty()) "WHERE ${conditions.joinToString(" AND ")}" else ""
        val order = if (orderByList.isNotEmpty()) "ORDER BY ${orderByList.joinToString(", ")}" else ""
        val limit = limitVal?.let { "LIMIT $it" } ?: ""
        val offset = offsetVal?.let { "OFFSET $it" } ?: ""
        
        return listOf("SELECT * FROM $table", where, order, limit, offset)
            .filter { it.isNotBlank() }
            .joinToString(" ")
    }
}

data class Route(val method: String, val path: String, val handler: suspend (Request) -> Response)
data class Request(val method: String, val path: String, val params: Map<String, String> = emptyMap()) {
    fun pathParam(name: String): String = params[name] ?: ""
}
data class Response(val body: Any, val statusCode: Int = 200) {
    companion object {
        fun ok(body: Any, status: Int = 200) = Response(body, status)
    }
}
```

---

## Delegated Properties

```kotlin
import kotlin.properties.Delegates
import kotlin.reflect.KProperty

// Lazy loading
val heavyResource: HeavyResource by lazy {
    HeavyResource().also { println("Created!") }
}

// Observable: react to changes
class ViewModel {
    var count: Int by Delegates.observable(0) { _, old, new ->
        println("count changed: $old → $new")
        onCountChanged(new)
    }
    
    var name: String by Delegates.vetoable("") { _, _, new ->
        new.length <= 50  // reject if > 50 chars
    }
    
    private fun onCountChanged(value: Int) { /* update UI */ }
}

// Custom delegate
class Cached<T>(private val ttlMs: Long, private val loader: () -> T) {
    private var value: T? = null
    private var loadedAt: Long = 0
    
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T {
        val now = System.currentTimeMillis()
        if (value == null || (now - loadedAt) > ttlMs) {
            value = loader()
            loadedAt = now
        }
        return value!!
    }
}

fun <T> cached(ttlMs: Long, loader: () -> T) = Cached(ttlMs, loader)

class ProductService2 {
    val categories by cached(ttlMs = 60_000) {
        println("Loading categories from DB...")
        listOf("Electronics", "Clothing", "Books")
    }
}

// Config delegate: read from environment
class Env(private val key: String, private val default: String? = null) {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return System.getenv(key) ?: default ?: error("Missing env variable: $key")
    }
}

object AppConfig {
    val dbUrl: String by Env("DATABASE_URL")
    val jwtSecret: String by Env("JWT_SECRET")
    val appPort: String by Env("PORT", "8080")
    val debug: String by Env("DEBUG", "false")
}

// Map delegate
class User(private val data: Map<String, Any>) {
    val name: String by data
    val email: String by data
    val age: Int by data
}

// ProvideDelegate: intercept delegation creation
class ValidatedString(private val regex: Regex) {
    operator fun provideDelegate(thisRef: Any?, property: KProperty<*>): ValidatedStringDelegate {
        println("Setting up validator for '${property.name}' with pattern: ${regex.pattern}")
        return ValidatedStringDelegate(regex, property.name)
    }
}

class ValidatedStringDelegate(private val regex: Regex, private val name: String) {
    private var value: String = ""
    
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String = value
    
    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: String) {
        if (!newValue.matches(regex)) {
            throw IllegalArgumentException("Invalid $name: '$newValue' doesn't match ${regex.pattern}")
        }
        value = newValue
    }
}

class OrderForm {
    var email: String by ValidatedString(Regex("[^@]+@[^@]+\\.[^@]+"))
    var phone: String by ValidatedString(Regex("0[0-9]{9}"))
    var postalCode: String by ValidatedString(Regex("[0-9]{5}"))
}

class HeavyResource
```

---

## Type System ขั้นสูง

```kotlin
// Sealed interfaces: more flexible than sealed class
sealed interface Shape {
    fun area(): Double
    fun perimeter(): Double
}

data class Circle(val radius: Double) : Shape {
    override fun area() = Math.PI * radius * radius
    override fun perimeter() = 2 * Math.PI * radius
}

data class Rectangle(val width: Double, val height: Double) : Shape {
    override fun area() = width * height
    override fun perimeter() = 2 * (width + height)
}

data class Triangle(val a: Double, val b: Double, val c: Double) : Shape {
    override fun area(): Double {
        val s = (a + b + c) / 2
        return Math.sqrt(s * (s - a) * (s - b) * (s - c))
    }
    override fun perimeter() = a + b + c
}

fun describeShape(shape: Shape): String = when (shape) {
    is Circle -> "Circle with radius ${shape.radius}, area = ${shape.area()}"
    is Rectangle -> "Rectangle ${shape.width}x${shape.height}"
    is Triangle -> "Triangle with sides ${shape.a}, ${shape.b}, ${shape.c}"
}

// Context receivers (Kotlin 1.6.20+)
context(UserContext, TransactionContext)
suspend fun transferMoney(from: AccountId, to: AccountId, amount: Money) {
    // Implicitly has access to UserContext and TransactionContext
    val currentUser = this@UserContext.userId
    val txId = this@TransactionContext.transactionId
    // ...
}

// Inline value classes: zero-overhead type safety
@JvmInline value class UserId(val value: String)
@JvmInline value class Email(val value: String) {
    init { require(value.contains("@")) { "Invalid email" } }
}
@JvmInline value class Kelvin(val value: Double) {
    init { require(value >= 0) { "Temperature cannot be below 0K" } }
    fun toCelsius(): Double = value - 273.15
}
@JvmInline value class ProductCode(val value: String) {
    init { require(value.matches(Regex("[A-Z]{3}-[0-9]{4}"))) { "Invalid product code" } }
}

// These compile to primitives - no boxing!
fun processUser(id: UserId, email: Email) = println("Processing ${id.value} / ${email.value}")

// Type aliases for complex types
typealias ProductMap = Map<ProductCode, Int>
typealias AsyncResult<T> = Deferred<Result<T>>
typealias Predicate<T> = (T) -> Boolean
typealias Handler<Req, Resp> = suspend (Req) -> Resp
typealias Middleware<Req, Resp> = (Handler<Req, Resp>) -> Handler<Req, Resp>

// Generic constraints
fun <T : Comparable<T>> clamp(value: T, min: T, max: T): T {
    return when {
        value < min -> min
        value > max -> max
        else -> value
    }
}

// Multiple bounds
fun <T> Collection<T>.sortedAndFiltered(
    predicate: (T) -> Boolean,
    comparator: Comparator<T>
): List<T> where T : Any = filter(predicate).sortedWith(comparator)

// Star projection
fun printAll(list: List<*>) {
    list.forEach { println(it) }
}

// Variance: in/out
interface Source<out T> {  // covariant: only produces T
    fun next(): T
}

interface Sink<in T> {  // contravariant: only consumes T
    fun add(value: T)
}

// Use-site variance
fun copy(from: Array<out Any>, to: Array<in Any>) {
    from.forEachIndexed { i, v -> to[i] = v }
}

data class AccountId(val value: String)
class TransactionContext { val transactionId: String = "" }
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง Type-safe Configuration DSL

@DslMarker
annotation class ConfigDsl

@ConfigDsl
class DatabaseConfigBuilder {
    var host: String = "localhost"
    var port: Int = 5432
    var database: String = ""
    var username: String = ""
    var password: String = ""
    var maxPoolSize: Int = 10
    var connectionTimeout: Long = 30_000
    
    private val properties = mutableMapOf<String, String>()
    
    fun property(key: String, value: String) {
        properties[key] = value
    }
    
    fun build(): DatabaseConfig {
        require(database.isNotBlank()) { "Database name required" }
        require(username.isNotBlank()) { "Username required" }
        return DatabaseConfig(host, port, database, username, password, maxPoolSize, connectionTimeout, properties.toMap())
    }
}

@ConfigDsl
class AppConfigBuilder {
    var appName: String = ""
    var port: Int = 8080
    var debug: Boolean = false
    private var dbConfig: DatabaseConfig? = null
    
    fun database(block: DatabaseConfigBuilder.() -> Unit) {
        dbConfig = DatabaseConfigBuilder().apply(block).build()
    }
    
    fun build(): CompleteAppConfig {
        require(appName.isNotBlank()) { "App name required" }
        return CompleteAppConfig(
            name = appName,
            port = port,
            debug = debug,
            database = dbConfig ?: throw IllegalStateException("Database config required")
        )
    }
}

fun appConfig(block: AppConfigBuilder.() -> Unit): CompleteAppConfig =
    AppConfigBuilder().apply(block).build()

data class DatabaseConfig(
    val host: String, val port: Int, val database: String,
    val username: String, val password: String,
    val maxPoolSize: Int, val connectionTimeout: Long,
    val properties: Map<String, String>
)

data class CompleteAppConfig(
    val name: String, val port: Int, val debug: Boolean, val database: DatabaseConfig
)

// Usage:
val config = appConfig {
    appName = "EcommerceApp"
    port = 8080
    debug = false
    database {
        host = "db.example.com"
        port = 5432
        database = "ecommerce"
        username = "app"
        password = System.getenv("DB_PASSWORD") ?: "secret"
        maxPoolSize = 20
        property("cachePrepStmts", "true")
        property("prepStmtCacheSize", "250")
    }
}
```

---

## สรุป Part 96

```
✅ coroutineScope: structured concurrency, all children must complete
✅ supervisorScope: child failure is isolated
✅ async/await: parallel concurrent operations
✅ withTimeout: cancel slow operations
✅ produce { send() }: CSP-style channels
✅ Pipeline: chain ReceiveChannels
✅ StateFlow: state holder, replays latest to new collectors
✅ SharedFlow: event bus, configurable replay
✅ CoroutineContext.Element: custom context (UserContext)
✅ withContext(element): add element to context
✅ coroutineContext[Key]: read from context
✅ retryWithExponentialBackoff: Flow extension with retry
✅ ensureActive(): cooperative cancellation check
✅ CoroutineExceptionHandler: unhandled exception handler
✅ @DslMarker: prevent accidental nesting
✅ RouterBuilder: type-safe route definition
✅ HTML DSL: tag-based builder with children
✅ QueryBuilder: SQL-like DSL with method chaining
✅ lazy: one-time lazy initialization
✅ Delegates.observable: react to property changes
✅ Delegates.vetoable: reject invalid values
✅ Custom delegate: Cached<T> with TTL
✅ Env delegate: environment variable binding
✅ Map delegate: destructure from Map
✅ provideDelegate: intercept delegation setup
✅ @JvmInline value class: zero-overhead type safety
✅ Sealed interface: flexible sealed hierarchy
✅ Context receivers: implicit context parameters
✅ typealias: complex type abbreviations
✅ Variance in/out: covariant/contravariant generics
✅ Config DSL: type-safe nested builder
```

---

*Part 96/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
