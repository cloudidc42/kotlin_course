# Part 81: Kotlin Multiplatform (KMP)

## สารบัญ
1. [KMP คืออะไร](#kmp-คืออะไร)
2. [Shared Business Logic](#shared-business-logic)
3. [Networking ด้วย Ktor Client](#networking-ด้วย-ktor-client)
4. [Local Storage ด้วย SQLDelight](#local-storage-ด้วย-sqldelight)
5. [Platform-Specific Code](#platform-specific-code)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## KMP คืออะไร

```
Kotlin Multiplatform: เขียน Kotlin code ครั้งเดียว ใช้ได้หลาย platform

Platform ที่รองรับ:
- Android (Kotlin/JVM)
- iOS (Kotlin/Native → Swift/ObjC interop)
- Web (Kotlin/JS → TypeScript interop)
- Desktop (Kotlin/JVM: macOS, Windows, Linux)
- Server (Kotlin/JVM, Kotlin/Native)

Source Sets:
commonMain/   → shared code (ทุก platform)
androidMain/  → Android-specific
iosMain/      → iOS-specific
jsMain/       → JavaScript-specific
jvmMain/      → JVM-specific (server)

Shared code ทำอะไรได้:
✅ Business logic
✅ Data models
✅ Validation
✅ Networking (Ktor Client)
✅ Serialization (kotlinx.serialization)
✅ Coroutines
✅ Database (SQLDelight)

ต้องทำแยก per-platform:
❌ UI (ใช้ Compose Multiplatform สำหรับ UI sharing)
❌ Device APIs (camera, GPS, etc.)
❌ Platform-specific libraries
```

---

## Project Setup

```kotlin
// build.gradle.kts (root)
plugins {
    kotlin("multiplatform") version "1.9.22"
    kotlin("plugin.serialization") version "1.9.22"
    id("com.android.library") version "8.1.4"
    id("app.cash.sqldelight") version "2.0.1"
}

kotlin {
    // Android target
    androidTarget {
        compilations.all {
            kotlinOptions {
                jvmTarget = "17"
            }
        }
    }
    
    // iOS targets
    listOf(
        iosX64(),
        iosArm64(),
        iosSimulatorArm64()
    ).forEach {
        it.binaries.framework {
            baseName = "shared"
            isStatic = true
        }
    }
    
    // JVM target (server)
    jvm()
    
    // Source sets
    sourceSets {
        val commonMain by getting {
            dependencies {
                // Coroutines
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
                
                // Serialization
                implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.2")
                
                // Ktor Client
                implementation("io.ktor:ktor-client-core:2.3.7")
                implementation("io.ktor:ktor-client-content-negotiation:2.3.7")
                implementation("io.ktor:ktor-serialization-kotlinx-json:2.3.7")
                implementation("io.ktor:ktor-client-logging:2.3.7")
                implementation("io.ktor:ktor-client-auth:2.3.7")
                
                // SQLDelight
                implementation("app.cash.sqldelight:runtime:2.0.1")
                implementation("app.cash.sqldelight:coroutines-extensions:2.0.1")
                
                // DateTime
                implementation("org.jetbrains.kotlinx:kotlinx-datetime:0.5.0")
            }
        }
        
        val commonTest by getting {
            dependencies {
                implementation(kotlin("test"))
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
            }
        }
        
        val androidMain by getting {
            dependencies {
                implementation("io.ktor:ktor-client-okhttp:2.3.7")
                implementation("app.cash.sqldelight:android-driver:2.0.1")
            }
        }
        
        val iosMain by creating {
            dependsOn(commonMain)
            dependencies {
                implementation("io.ktor:ktor-client-darwin:2.3.7")
                implementation("app.cash.sqldelight:native-driver:2.0.1")
            }
        }
        
        val jvmMain by getting {
            dependencies {
                implementation("io.ktor:ktor-client-cio:2.3.7")
                implementation("app.cash.sqldelight:sqlite-driver:2.0.1")
            }
        }
    }
}

sqldelight {
    databases {
        create("AppDatabase") {
            packageName.set("com.ecommerce.db")
            srcDirs.setFrom("src/commonMain/sqldelight")
        }
    }
}
```

---

## Shared Business Logic

```kotlin
// commonMain/kotlin/com/ecommerce/domain/Product.kt

import kotlinx.serialization.Serializable
import kotlinx.datetime.Instant

@Serializable
data class Product(
    val id: String,
    val name: String,
    val description: String,
    val price: Double,
    val currency: String,
    val imageUrl: String?,
    val stockQuantity: Int,
    val categoryId: String,
    val createdAt: Instant
) {
    val isAvailable: Boolean get() = stockQuantity > 0
    val formattedPrice: String get() = "$price $currency"
}

@Serializable
data class ProductFilter(
    val categoryId: String? = null,
    val searchQuery: String? = null,
    val minPrice: Double? = null,
    val maxPrice: Double? = null,
    val sortBy: SortBy = SortBy.CREATED_AT_DESC,
    val page: Int = 0,
    val size: Int = 20
)

@Serializable
enum class SortBy {
    PRICE_ASC, PRICE_DESC, NAME_ASC, CREATED_AT_DESC, POPULARITY
}

@Serializable
data class PagedResult<T>(
    val items: List<T>,
    val totalCount: Int,
    val page: Int,
    val size: Int,
    val hasMore: Boolean
)

// Shared validation
object ProductValidator {
    
    fun validateCreate(
        name: String,
        price: Double,
        sku: String
    ): ValidationResult {
        val errors = mutableListOf<String>()
        
        if (name.isBlank()) errors.add("Name is required")
        if (name.length > 500) errors.add("Name must be at most 500 characters")
        if (price <= 0) errors.add("Price must be positive")
        if (sku.isBlank()) errors.add("SKU is required")
        if (sku.length < 3 || sku.length > 50) errors.add("SKU must be 3-50 characters")
        
        return if (errors.isEmpty()) ValidationResult.Valid
               else ValidationResult.Invalid(errors)
    }
}

sealed class ValidationResult {
    object Valid : ValidationResult()
    data class Invalid(val errors: List<String>) : ValidationResult()
}

// Shared cart logic
data class CartItem(
    val product: Product,
    val quantity: Int
) {
    val subtotal: Double get() = product.price * quantity
}

class Cart {
    private val _items = mutableMapOf<String, CartItem>()
    val items: List<CartItem> get() = _items.values.toList()
    
    val total: Double get() = items.sumOf { it.subtotal }
    val itemCount: Int get() = items.sumOf { it.quantity }
    val isEmpty: Boolean get() = _items.isEmpty()
    
    fun addItem(product: Product, quantity: Int = 1): Cart {
        val existing = _items[product.id]
        _items[product.id] = if (existing != null) {
            existing.copy(quantity = existing.quantity + quantity)
        } else {
            CartItem(product, quantity)
        }
        return this
    }
    
    fun removeItem(productId: String): Cart {
        _items.remove(productId)
        return this
    }
    
    fun updateQuantity(productId: String, quantity: Int): Cart {
        if (quantity <= 0) {
            removeItem(productId)
        } else {
            _items[productId]?.let {
                _items[productId] = it.copy(quantity = quantity)
            }
        }
        return this
    }
    
    fun clear(): Cart {
        _items.clear()
        return this
    }
}

// Test in commonTest
class CartTest {
    @Test
    fun `adding same product should increase quantity`() {
        val product = Product("p1", "Test", "", 100.0, "THB", null, 10, "cat1", kotlinx.datetime.Clock.System.now())
        val cart = Cart()
        
        cart.addItem(product, 2)
        cart.addItem(product, 3)
        
        kotlin.test.assertEquals(5, cart.items.first().quantity)
        kotlin.test.assertEquals(500.0, cart.total)
    }
    
    @Test
    fun `removing item should decrease total`() {
        val p1 = Product("p1", "P1", "", 100.0, "THB", null, 10, "cat1", kotlinx.datetime.Clock.System.now())
        val p2 = Product("p2", "P2", "", 200.0, "THB", null, 10, "cat1", kotlinx.datetime.Clock.System.now())
        val cart = Cart().addItem(p1).addItem(p2)
        
        cart.removeItem("p1")
        
        kotlin.test.assertEquals(200.0, cart.total)
        kotlin.test.assertEquals(1, cart.items.size)
    }
}
```

---

## Networking ด้วย Ktor Client

```kotlin
// commonMain: shared HTTP client

import io.ktor.client.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.client.plugins.logging.*
import io.ktor.client.plugins.auth.*
import io.ktor.client.plugins.auth.providers.*
import io.ktor.serialization.kotlinx.json.*
import kotlinx.serialization.json.Json

// Platform-specific engine is injected
expect fun createHttpClient(): HttpClient

// Shared client configuration
fun createConfiguredClient(engine: HttpClient): HttpClient {
    return engine.config {
        install(ContentNegotiation) {
            json(Json {
                ignoreUnknownKeys = true
                isLenient = true
                encodeDefaults = true
            })
        }
        
        install(Logging) {
            level = LogLevel.INFO
        }
        
        install(Auth) {
            bearer {
                loadTokens {
                    val token = TokenStorage.getAccessToken()
                    if (token != null) BearerTokens(token, "") else null
                }
                
                refreshTokens {
                    val refreshToken = TokenStorage.getRefreshToken() ?: return@refreshTokens null
                    val newTokens = authRepository.refreshToken(refreshToken)
                    TokenStorage.saveTokens(newTokens.accessToken, newTokens.refreshToken)
                    BearerTokens(newTokens.accessToken, newTokens.refreshToken)
                }
            }
        }
    }
}

expect object TokenStorage {
    fun getAccessToken(): String?
    fun getRefreshToken(): String?
    fun saveTokens(accessToken: String, refreshToken: String)
    fun clearTokens()
}

interface AuthRepository {
    suspend fun refreshToken(refreshToken: String): TokenPair
}

data class TokenPair(val accessToken: String, val refreshToken: String)

// API client
class ProductApiClient(private val client: HttpClient, private val baseUrl: String) {
    
    suspend fun getProducts(filter: ProductFilter): PagedResult<Product> {
        return client.get("$baseUrl/api/v1/products") {
            filter.categoryId?.let { parameter("categoryId", it) }
            filter.searchQuery?.let { parameter("search", it) }
            filter.minPrice?.let { parameter("minPrice", it) }
            filter.maxPrice?.let { parameter("maxPrice", it) }
            parameter("sort", filter.sortBy.name.lowercase())
            parameter("page", filter.page)
            parameter("size", filter.size)
        }.body()
    }
    
    suspend fun getProduct(id: String): Product {
        return client.get("$baseUrl/api/v1/products/$id").body()
    }
    
    suspend fun searchProducts(query: String, page: Int = 0): PagedResult<Product> {
        return client.get("$baseUrl/api/v1/products/search") {
            parameter("q", query)
            parameter("page", page)
        }.body()
    }
}

// Ktor get/parameter extension functions
private fun io.ktor.client.request.HttpRequestBuilder.parameter(key: String, value: Any?) {
    if (value != null) url.parameters.append(key, value.toString())
}

private suspend fun io.ktor.client.HttpClient.get(url: String): io.ktor.client.statement.HttpResponse {
    return get(url) {}
}

private suspend fun io.ktor.client.HttpClient.get(
    url: String,
    block: io.ktor.client.request.HttpRequestBuilder.() -> Unit
): io.ktor.client.statement.HttpResponse {
    return get(io.ktor.client.request.HttpRequestBuilder().apply {
        io.ktor.http.takeFrom(url)
        block()
    })
}

private suspend inline fun <reified T> io.ktor.client.statement.HttpResponse.body(): T =
    io.ktor.client.call.body()

// Android implementation (androidMain)
actual fun createHttpClient(): HttpClient {
    return HttpClient(io.ktor.client.engine.okhttp.OkHttp)
}

actual object TokenStorage {
    private lateinit var prefs: android.content.SharedPreferences
    
    fun init(context: android.content.Context) {
        prefs = context.getSharedPreferences("tokens", android.content.Context.MODE_PRIVATE)
    }
    
    actual fun getAccessToken() = prefs.getString("access_token", null)
    actual fun getRefreshToken() = prefs.getString("refresh_token", null)
    actual fun saveTokens(accessToken: String, refreshToken: String) {
        prefs.edit()
            .putString("access_token", accessToken)
            .putString("refresh_token", refreshToken)
            .apply()
    }
    actual fun clearTokens() {
        prefs.edit().clear().apply()
    }
}

// iOS implementation (iosMain)
actual fun createHttpClient(): HttpClient {
    return HttpClient(io.ktor.client.engine.darwin.Darwin)
}

actual object TokenStorage {
    actual fun getAccessToken() = platform.Foundation.NSUserDefaults.standardUserDefaults
        .stringForKey("access_token")
    actual fun getRefreshToken() = platform.Foundation.NSUserDefaults.standardUserDefaults
        .stringForKey("refresh_token")
    actual fun saveTokens(accessToken: String, refreshToken: String) {
        platform.Foundation.NSUserDefaults.standardUserDefaults.apply {
            setObject(accessToken, "access_token")
            setObject(refreshToken, "refresh_token")
        }
    }
    actual fun clearTokens() {
        platform.Foundation.NSUserDefaults.standardUserDefaults.apply {
            removeObjectForKey("access_token")
            removeObjectForKey("refresh_token")
        }
    }
}

val authRepository: AuthRepository = object : AuthRepository {
    override suspend fun refreshToken(refreshToken: String): TokenPair = TODO()
}
```

---

## Platform-Specific Code

```kotlin
// expect/actual: platform-specific implementations

// commonMain: define what you need
expect class PlatformInfo {
    val name: String
    val version: String
    val isDebug: Boolean
}

expect fun getCurrentTimeMillis(): Long

expect fun generateUUID(): String

expect fun logDebug(tag: String, message: String)

// androidMain: provide Android implementation
actual class PlatformInfo {
    actual val name: String = "Android"
    actual val version: String = android.os.Build.VERSION.RELEASE
    actual val isDebug: Boolean = com.ecommerce.BuildConfig.DEBUG
}

actual fun getCurrentTimeMillis(): Long = System.currentTimeMillis()

actual fun generateUUID(): String = java.util.UUID.randomUUID().toString()

actual fun logDebug(tag: String, message: String) {
    android.util.Log.d(tag, message)
}

// iosMain: provide iOS implementation
actual class PlatformInfo {
    actual val name: String = "iOS"
    actual val version: String = platform.UIKit.UIDevice.currentDevice.systemVersion
    actual val isDebug: Boolean = platform.Foundation.NSProcessInfo.processInfo
        .environment["DEBUG"] != null
}

actual fun getCurrentTimeMillis(): Long = 
    (platform.Foundation.NSDate.date().timeIntervalSince1970 * 1000).toLong()

actual fun generateUUID(): String = 
    platform.Foundation.NSUUID.UUID().UUIDString

actual fun logDebug(tag: String, message: String) {
    println("[$tag] $message")  // iOS: NSLog or print
}

// Compose Multiplatform for shared UI
// (optional, requires Compose Multiplatform plugin)
@Composable
fun ProductListScreen(viewModel: ProductViewModel) {
    val products by viewModel.products.collectAsState()
    
    LazyColumn {
        items(products) { product ->
            ProductCard(product = product)
        }
    }
}

@Composable
fun ProductCard(product: Product) {
    Card(modifier = Modifier.fillMaxWidth().padding(8.dp)) {
        Row {
            AsyncImage(
                model = product.imageUrl,
                contentDescription = product.name
            )
            Column {
                Text(product.name, style = MaterialTheme.typography.titleMedium)
                Text(product.formattedPrice)
            }
        }
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: เพิ่ม Offline Support ใน KMP App

// Requirements:
// 1. Cache products ใน SQLDelight database
// 2. Show cached data เมื่อไม่มี internet
// 3. Sync ข้อมูลใหม่เมื่อ network กลับมา
// 4. Show "offline mode" indicator ให้ user รู้

// commonMain/sqldelight/com/ecommerce/db/Products.sq
// CREATE TABLE Product (
//   id TEXT NOT NULL PRIMARY KEY,
//   name TEXT NOT NULL,
//   price REAL NOT NULL,
//   currency TEXT NOT NULL,
//   imageUrl TEXT,
//   stockQuantity INTEGER NOT NULL,
//   categoryId TEXT NOT NULL,
//   cachedAt INTEGER NOT NULL
// );
// 
// selectAll:
// SELECT * FROM Product ORDER BY name;
// 
// selectByCategory:
// SELECT * FROM Product WHERE categoryId = :categoryId ORDER BY name;
// 
// upsert:
// INSERT OR REPLACE INTO Product VALUES (?, ?, ?, ?, ?, ?, ?, ?);
// 
// deleteAll:
// DELETE FROM Product;

class ProductRepository(
    private val apiClient: ProductApiClient,
    private val database: AppDatabase,
    private val networkMonitor: NetworkMonitor
) {
    
    // Offline-first strategy
    fun getProducts(filter: ProductFilter): kotlinx.coroutines.flow.Flow<List<Product>> {
        return kotlinx.coroutines.flow.flow {
            // 1. Emit cached data immediately
            val cached = database.productQueries.selectAll().executeAsList().map { it.toDomain() }
            emit(cached)
            
            // 2. Fetch fresh data if online
            if (networkMonitor.isConnected()) {
                try {
                    val fresh = apiClient.getProducts(filter)
                    
                    // Update cache
                    database.transaction {
                        database.productQueries.deleteAll()
                        fresh.items.forEach { product ->
                            database.productQueries.upsert(
                                id = product.id,
                                name = product.name,
                                price = product.price,
                                currency = product.currency,
                                imageUrl = product.imageUrl,
                                stockQuantity = product.stockQuantity.toLong(),
                                categoryId = product.categoryId,
                                cachedAt = kotlinx.datetime.Clock.System.now().toEpochMilliseconds()
                            )
                        }
                    }
                    
                    emit(fresh.items)
                } catch (e: Exception) {
                    // Network error: keep showing cached data
                    logDebug("ProductRepository", "Network error: ${e.message}")
                }
            }
        }
    }
}

interface NetworkMonitor {
    fun isConnected(): Boolean
    fun observeConnectivity(): kotlinx.coroutines.flow.Flow<Boolean>
}

// Android implementation
class AndroidNetworkMonitor(context: android.content.Context) : NetworkMonitor {
    private val connectivityManager = context.getSystemService(android.content.Context.CONNECTIVITY_SERVICE) as android.net.ConnectivityManager
    
    override fun isConnected(): Boolean {
        val network = connectivityManager.activeNetwork ?: return false
        val capabilities = connectivityManager.getNetworkCapabilities(network) ?: return false
        return capabilities.hasCapability(android.net.NetworkCapabilities.NET_CAPABILITY_INTERNET)
    }
    
    override fun observeConnectivity() = kotlinx.coroutines.flow.callbackFlow<Boolean> {
        val callback = object : android.net.ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: android.net.Network) { trySend(true) }
            override fun onLost(network: android.net.Network) { trySend(false) }
        }
        connectivityManager.registerDefaultNetworkCallback(callback)
        awaitClose { connectivityManager.unregisterNetworkCallback(callback) }
    }
}

fun Any.toDomain(): Product = Product("", "", "", 0.0, "", null, 0, "", kotlinx.datetime.Clock.System.now())
```

---

## สรุป Part 81

```
✅ KMP: write once, run on Android/iOS/Web/Desktop/Server
✅ Source sets: commonMain, androidMain, iosMain, jvmMain
✅ expect/actual: platform-specific implementations
✅ Ktor Client: multiplatform HTTP with platform engines
✅ OkHttp (Android), Darwin (iOS), CIO (JVM)
✅ ContentNegotiation: kotlinx.serialization JSON
✅ Auth plugin: Bearer token with auto-refresh
✅ TokenStorage expect/actual: SharedPreferences vs NSUserDefaults
✅ SQLDelight: multiplatform SQL database
✅ .sq files: type-safe SQL queries
✅ database.transaction: atomic batch operations
✅ kotlinx.serialization: @Serializable data classes
✅ kotlinx.datetime: Instant, Clock
✅ PlatformInfo actual: device info per platform
✅ generateUUID actual: UUID per platform
✅ Compose Multiplatform: shared UI (optional)
✅ Offline-first: cache-first, network-second strategy
✅ NetworkMonitor: connectivity state as Flow
✅ callbackFlow: bridge callback APIs to Flow
✅ Cart: shared business logic tested in commonTest
```

---

*Part 81/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
