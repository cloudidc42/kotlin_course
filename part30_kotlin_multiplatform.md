# Part 30: Kotlin Multiplatform (KMP)

## สารบัญ
1. [KMP คืออะไร](#kmp-คืออะไร)
2. [โครงสร้าง KMP Project](#โครงสร้าง-kmp-project)
3. [Shared Business Logic](#shared-business-logic)
4. [expect/actual Mechanism](#expectactual-mechanism)
5. [Multiplatform Libraries](#multiplatform-libraries)
6. [KMP กับ Mobile](#kmp-กับ-mobile)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## KMP คืออะไร

```
Kotlin Multiplatform (KMP) = เขียน Kotlin code ครั้งเดียว รัน่ไปทุก platform

Platforms ที่รองรับ:
├── JVM (Android, Server)
├── JavaScript (Browser, Node.js)
├── Native
│   ├── iOS (iPhone, iPad)
│   ├── macOS
│   ├── Windows (MinGW)
│   └── Linux
└── WebAssembly (WASM)

ประโยชน์:
✅ Share business logic ระหว่าง Android & iOS
✅ Share code ระหว่าง Frontend & Backend
✅ ลด code duplication
✅ Type-safe ทุก platform
✅ ใช้ platform-specific code เมื่อจำเป็น (expect/actual)
```

---

## โครงสร้าง KMP Project

```kotlin
// settings.gradle.kts
rootProject.name = "MyKMPApp"
include(":shared")
include(":androidApp")
include(":iosApp")  // for iOS host (usually separate Xcode project)
include(":desktopApp")

// shared/build.gradle.kts
plugins {
    kotlin("multiplatform") version "1.9.22"
    kotlin("plugin.serialization") version "1.9.22"
}

kotlin {
    // JVM targets
    jvm()
    jvmToolchain(17)
    
    // JavaScript targets
    js(IR) {
        browser()
        nodejs()
    }
    
    // Native targets
    androidTarget()
    
    iosX64()
    iosArm64()
    iosSimulatorArm64()
    
    macosX64()
    macosArm64()
    
    // Source sets
    sourceSets {
        // Common code (shared by all platforms)
        val commonMain by getting {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
                implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.2")
                implementation("io.ktor:ktor-client-core:2.3.7")
                implementation("io.ktor:ktor-client-content-negotiation:2.3.7")
                implementation("io.ktor:ktor-serialization-kotlinx-json:2.3.7")
            }
        }
        
        val commonTest by getting {
            dependencies {
                implementation(kotlin("test"))
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
            }
        }
        
        // JVM-specific
        val jvmMain by getting {
            dependencies {
                implementation("io.ktor:ktor-client-okhttp:2.3.7")
            }
        }
        
        // Android-specific
        val androidMain by getting {
            dependencies {
                implementation("io.ktor:ktor-client-android:2.3.7")
            }
        }
        
        // iOS-specific
        val iosMain by creating {
            dependsOn(commonMain)
            dependencies {
                implementation("io.ktor:ktor-client-darwin:2.3.7")
            }
        }
        
        val iosX64Main by getting { dependsOn(iosMain) }
        val iosArm64Main by getting { dependsOn(iosMain) }
        val iosSimulatorArm64Main by getting { dependsOn(iosMain) }
        
        // JavaScript
        val jsMain by getting {
            dependencies {
                implementation("io.ktor:ktor-client-js:2.3.7")
            }
        }
    }
}
```

---

## Shared Business Logic

```kotlin
// commonMain/kotlin/models/Product.kt
import kotlinx.serialization.Serializable

@Serializable
data class Product(
    val id: Long,
    val name: String,
    val description: String,
    val price: Double,
    val category: String,
    val imageUrl: String? = null,
    val inStock: Boolean = true
)

@Serializable
data class ProductListResponse(
    val products: List<Product>,
    val total: Int,
    val page: Int
)

@Serializable
data class CartItem(
    val product: Product,
    val quantity: Int
) {
    val subtotal: Double get() = product.price * quantity
}

@Serializable
data class Cart(
    val items: List<CartItem> = emptyList()
) {
    val total: Double get() = items.sumOf { it.subtotal }
    val itemCount: Int get() = items.sumOf { it.quantity }
    
    fun addItem(product: Product, quantity: Int = 1): Cart {
        val existing = items.find { it.product.id == product.id }
        return if (existing != null) {
            copy(items = items.map { 
                if (it.product.id == product.id) it.copy(quantity = it.quantity + quantity)
                else it
            })
        } else {
            copy(items = items + CartItem(product, quantity))
        }
    }
    
    fun removeItem(productId: Long): Cart =
        copy(items = items.filter { it.product.id != productId })
    
    fun updateQuantity(productId: Long, quantity: Int): Cart =
        if (quantity <= 0) removeItem(productId)
        else copy(items = items.map {
            if (it.product.id == productId) it.copy(quantity = quantity)
            else it
        })
    
    fun clear(): Cart = copy(items = emptyList())
}

// commonMain/kotlin/repository/ProductRepository.kt
interface ProductRepository {
    suspend fun getProducts(page: Int, category: String? = null): ProductListResponse
    suspend fun getProduct(id: Long): Product?
    suspend fun searchProducts(query: String): List<Product>
}

// commonMain/kotlin/viewmodel/ProductViewModel.kt
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

class ProductViewModel(
    private val repository: ProductRepository,
    private val coroutineScope: CoroutineScope
) {
    private val _products = MutableStateFlow<List<Product>>(emptyList())
    val products: StateFlow<List<Product>> = _products.asStateFlow()
    
    private val _cart = MutableStateFlow(Cart())
    val cart: StateFlow<Cart> = _cart.asStateFlow()
    
    private val _loading = MutableStateFlow(false)
    val loading: StateFlow<Boolean> = _loading.asStateFlow()
    
    private val _error = MutableStateFlow<String?>(null)
    val error: StateFlow<String?> = _error.asStateFlow()
    
    private var currentPage = 0
    private var currentCategory: String? = null
    
    fun loadProducts(category: String? = null, refresh: Boolean = false) {
        if (refresh) {
            currentPage = 0
            currentCategory = category
            _products.value = emptyList()
        }
        
        coroutineScope.launch {
            _loading.value = true
            _error.value = null
            
            try {
                val result = repository.getProducts(currentPage, currentCategory)
                _products.value = if (refresh) result.products
                    else _products.value + result.products
                currentPage++
            } catch (e: Exception) {
                _error.value = e.message ?: "Unknown error"
            } finally {
                _loading.value = false
            }
        }
    }
    
    fun addToCart(product: Product) {
        _cart.update { it.addItem(product) }
    }
    
    fun removeFromCart(productId: Long) {
        _cart.update { it.removeItem(productId) }
    }
    
    fun updateCartQuantity(productId: Long, quantity: Int) {
        _cart.update { it.updateQuantity(productId, quantity) }
    }
    
    fun clearCart() {
        _cart.update { it.clear() }
    }
    
    fun searchProducts(query: String) {
        coroutineScope.launch {
            _loading.value = true
            try {
                val results = repository.searchProducts(query)
                _products.value = results
            } catch (e: Exception) {
                _error.value = e.message
            } finally {
                _loading.value = false
            }
        }
    }
}
```

---

## expect/actual Mechanism

```kotlin
// commonMain/kotlin/platform/Platform.kt
expect object Platform {
    val name: String
    val version: String
    fun isDebug(): Boolean
    fun log(message: String)
}

// commonMain/kotlin/platform/DateUtils.kt
expect class DateFormatter {
    fun format(timestamp: Long, pattern: String): String
    fun parse(dateString: String, pattern: String): Long?
}

expect fun getCurrentTimestamp(): Long

// commonMain/kotlin/storage/Storage.kt
expect interface LocalStorage {
    fun getString(key: String): String?
    fun setString(key: String, value: String)
    fun remove(key: String)
    fun clear()
}

// =====================
// JVM implementation
// jvmMain/kotlin/platform/Platform.jvm.kt
actual object Platform {
    actual val name = "JVM"
    actual val version: String = System.getProperty("java.version")
    actual fun isDebug() = System.getProperty("debug") == "true"
    actual fun log(message: String) = println("[JVM] $message")
}

// jvmMain/kotlin/platform/DateUtils.jvm.kt
import java.time.format.DateTimeFormatter
import java.time.Instant
import java.time.ZoneId

actual class DateFormatter {
    actual fun format(timestamp: Long, pattern: String): String {
        val formatter = DateTimeFormatter.ofPattern(pattern).withZone(ZoneId.systemDefault())
        return formatter.format(Instant.ofEpochMilli(timestamp))
    }
    
    actual fun parse(dateString: String, pattern: String): Long? {
        return try {
            val formatter = DateTimeFormatter.ofPattern(pattern).withZone(ZoneId.systemDefault())
            Instant.from(formatter.parse(dateString)).toEpochMilli()
        } catch (e: Exception) {
            null
        }
    }
}

actual fun getCurrentTimestamp() = System.currentTimeMillis()

// =====================
// iOS/Native implementation  
// iosMain/kotlin/platform/Platform.ios.kt
import platform.UIKit.UIDevice
import platform.Foundation.NSDate

actual object Platform {
    actual val name = "iOS"
    actual val version = UIDevice.currentDevice.systemVersion
    actual fun isDebug() = false  // check build config
    actual fun log(message: String) = NSLog("[iOS] $message")  // simplified
}

// iosMain/kotlin/platform/DateUtils.ios.kt
import platform.Foundation.*

actual class DateFormatter {
    actual fun format(timestamp: Long, pattern: String): String {
        val date = NSDate.dateWithTimeIntervalSince1970(timestamp / 1000.0)
        val formatter = NSDateFormatter()
        formatter.dateFormat = pattern
        return formatter.stringFromDate(date)
    }
    
    actual fun parse(dateString: String, pattern: String): Long? {
        val formatter = NSDateFormatter()
        formatter.dateFormat = pattern
        val date = formatter.dateFromString(dateString) ?: return null
        return (date.timeIntervalSince1970 * 1000).toLong()
    }
}

actual fun getCurrentTimestamp() = (NSDate().timeIntervalSince1970 * 1000).toLong()

// =====================
// JavaScript implementation
// jsMain/kotlin/platform/Platform.js.kt

actual object Platform {
    actual val name = "JavaScript"
    actual val version = js("typeof window !== 'undefined' ? 'Browser' : 'Node.js'") as String
    actual fun isDebug() = js("process.env.NODE_ENV === 'development'") as Boolean
    actual fun log(message: String) = console.log("[JS] $message")
}
```

---

## Multiplatform Libraries

```kotlin
// HTTP Client - Ktor (cross-platform)
// commonMain/kotlin/api/ProductApi.kt
import io.ktor.client.*
import io.ktor.client.request.*
import io.ktor.client.call.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.serialization.kotlinx.json.*

class ProductApi(
    baseUrl: String,
    httpClient: HttpClient = createDefaultClient()
) : ProductRepository {
    private val client = httpClient
    private val base = baseUrl
    
    override suspend fun getProducts(page: Int, category: String?): ProductListResponse {
        return client.get("$base/products") {
            parameter("page", page)
            parameter("size", 20)
            category?.let { parameter("category", it) }
        }.body()
    }
    
    override suspend fun getProduct(id: Long): Product? {
        return try {
            client.get("$base/products/$id").body<Product>()
        } catch (e: Exception) {
            null
        }
    }
    
    override suspend fun searchProducts(query: String): List<Product> {
        return client.get("$base/products/search") {
            parameter("q", query)
        }.body<ProductListResponse>().products
    }
}

// commonMain: create platform-specific HttpClient
expect fun createDefaultClient(): HttpClient

// androidMain:
actual fun createDefaultClient() = HttpClient(Android) {
    install(ContentNegotiation) { json() }
    engine {
        connectTimeout = 30_000
        socketTimeout = 30_000
    }
}

// iosMain:
actual fun createDefaultClient() = HttpClient(Darwin) {
    install(ContentNegotiation) { json() }
    engine {
        configureRequest { timeoutIntervalForRequest = 30.0 }
    }
}

// jvmMain:
actual fun createDefaultClient() = HttpClient(OkHttp) {
    install(ContentNegotiation) { json() }
}

// jsMain:
actual fun createDefaultClient() = HttpClient(Js) {
    install(ContentNegotiation) { json() }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Shared Currency Converter

@Serializable
data class ExchangeRate(
    val from: String,
    val to: String,
    val rate: Double,
    val timestamp: Long
)

@Serializable
data class ConversionResult(
    val fromCurrency: String,
    val toCurrency: String,
    val fromAmount: Double,
    val toAmount: Double,
    val rate: Double,
    val timestamp: Long
)

interface ExchangeRateRepository {
    suspend fun getRates(baseCurrency: String): Map<String, Double>
    suspend fun convert(from: String, to: String, amount: Double): ConversionResult
}

class CurrencyConverter(
    private val repository: ExchangeRateRepository,
    private val scope: CoroutineScope
) {
    private val _rates = MutableStateFlow<Map<String, Double>>(emptyMap())
    val rates: StateFlow<Map<String, Double>> = _rates.asStateFlow()
    
    private val _result = MutableStateFlow<ConversionResult?>(null)
    val result: StateFlow<ConversionResult?> = _result.asStateFlow()
    
    private val _loading = MutableStateFlow(false)
    val loading: StateFlow<Boolean> = _loading.asStateFlow()
    
    private val _error = MutableStateFlow<String?>(null)
    val error: StateFlow<String?> = _error.asStateFlow()
    
    private var baseCurrency = "THB"
    
    fun loadRates(base: String = "THB") {
        baseCurrency = base
        scope.launch {
            _loading.value = true
            try {
                _rates.value = repository.getRates(base)
                _error.value = null
            } catch (e: Exception) {
                _error.value = "Failed to load rates: ${e.message}"
            } finally {
                _loading.value = false
            }
        }
    }
    
    fun convert(from: String, to: String, amount: Double) {
        scope.launch {
            _loading.value = true
            try {
                _result.value = repository.convert(from, to, amount)
                _error.value = null
            } catch (e: Exception) {
                _error.value = "Conversion failed: ${e.message}"
            } finally {
                _loading.value = false
            }
        }
    }
    
    // Quick local conversion using loaded rates
    fun convertLocal(from: String, to: String, amount: Double): Double? {
        val fromRate = if (from == baseCurrency) 1.0 else rates.value[from] ?: return null
        val toRate = if (to == baseCurrency) 1.0 else rates.value[to] ?: return null
        return amount / fromRate * toRate
    }
    
    val supportedCurrencies: List<String>
        get() = (rates.value.keys + baseCurrency).sorted()
}

// Test in commonTest
class CurrencyConverterTest {
    @Test
    fun testLocalConversion() = runTest {
        val mockRepo = object : ExchangeRateRepository {
            override suspend fun getRates(baseCurrency: String) = mapOf(
                "USD" to 0.028,
                "EUR" to 0.026,
                "JPY" to 4.2
            )
            override suspend fun convert(from: String, to: String, amount: Double) =
                ConversionResult(from, to, amount, amount * 0.028, 0.028, System.currentTimeMillis())
        }
        
        val scope = CoroutineScope(StandardTestDispatcher())
        val converter = CurrencyConverter(mockRepo, scope)
        converter.loadRates("THB")
        scope.coroutineContext[Job]?.children?.forEach { it.join() }
        
        val result = converter.convertLocal("THB", "USD", 1000.0)
        assertEquals(28.0, result ?: 0.0, 0.01)
        
        val result2 = converter.convertLocal("USD", "EUR", 100.0)
        assertNotNull(result2)
    }
}
```

---

## สรุป Part 30

```
✅ KMP: เขียน Kotlin code ครั้งเดียว ใช้งานหลาย platform
✅ commonMain: code ที่ share กันทุก platform
✅ androidMain/iosMain/jvmMain/jsMain: platform-specific
✅ expect: ประกาศ interface ใน commonMain
✅ actual: implement ใน platform-specific source set
✅ Ktor Client: HTTP client ที่รองรับ multiplatform
✅ kotlinx.serialization: serialization ที่รองรับ multiplatform
✅ ViewModel/StateFlow: shared presentation logic
✅ ProductRepository: abstract data access layer
✅ Unit testing ใน commonTest รันได้ทุก platform
✅ ลด code duplication ระหว่าง Android/iOS/Web
```

---

*Part 30/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
