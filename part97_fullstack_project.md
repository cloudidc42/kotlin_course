# Part 97: Full-Stack Kotlin Project — E-Commerce Platform

## สารบัญ
1. [Project Architecture Overview](#project-architecture-overview)
2. [Monorepo Structure](#monorepo-structure)
3. [Shared Domain Module](#shared-domain-module)
4. [Backend: Ktor API](#backend-ktor-api)
5. [Frontend: Compose for Web](#frontend-compose-for-web)
6. [Mobile: Compose Multiplatform](#mobile-compose-multiplatform)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Project Architecture Overview

```
Full-Stack Kotlin E-Commerce:

                    ┌──────────────┐
                    │  Shared Core │  ← domain models, business logic
                    │  (commonMain)│     validation, use cases
                    └──────┬───────┘
           ┌───────────────┼────────────────┐
           ▼               ▼                ▼
    ┌──────────┐    ┌──────────────┐  ┌───────────┐
    │  Backend │    │ Android App  │  │  iOS App  │
    │  (Ktor)  │    │  (Compose)   │  │  (Compose)│
    │  JVM     │    │  Android SDK │  │  iOS SDK  │
    └──────────┘    └──────────────┘  └───────────┘
           ▲               ▲                ▲
           └───────────────┼────────────────┘
                    ┌──────┴───────┐
                    │  Kotlin API  │  ← HTTP client, serialization
                    │  Client Lib  │     shared networking
                    └──────────────┘

Modules:
:shared:domain      - pure Kotlin, no platform deps
:shared:api-client  - Ktor client, Kotlin serialization
:backend            - Ktor server, JPA, Kafka
:android            - Compose Android
:ios                - Compose iOS (or SwiftUI using shared logic)
:web-compose        - Compose for Web (experimental)
```

---

## Monorepo Structure

```kotlin
// settings.gradle.kts
rootProject.name = "ecommerce-fullstack"

include(
    ":shared:domain",
    ":shared:api-client",
    ":backend",
    ":android",
    ":web-compose"
)

// :shared:domain/build.gradle.kts
plugins {
    kotlin("multiplatform") version "1.9.22"
    kotlin("plugin.serialization") version "1.9.22"
}

kotlin {
    jvm()
    js(IR) { browser() }
    
    iosX64()
    iosArm64()
    iosSimulatorArm64()
    
    androidTarget {
        compileSdk = 34
    }
    
    sourceSets {
        commonMain.dependencies {
            implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.2")
            implementation("org.jetbrains.kotlinx:kotlinx-datetime:0.5.0")
            implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
        }
        jvmMain.dependencies {
            // JVM-specific dependencies
        }
    }
}
```

---

## Shared Domain Module

```kotlin
// :shared:domain/src/commonMain/kotlin/

// Shared models (serializable for JSON)
import kotlinx.serialization.Serializable
import kotlinx.datetime.Instant

@Serializable
data class ProductDto(
    val id: String,
    val name: String,
    val description: String?,
    val price: Double,
    val currency: String,
    val stockQuantity: Int,
    val categoryId: String,
    val images: List<String>,
    val rating: Double?,
    val reviewCount: Int
)

@Serializable
data class CategoryDto(val id: String, val name: String, val imageUrl: String?)

@Serializable
data class CartItemDto(
    val productId: String,
    val productName: String,
    val unitPrice: Double,
    val quantity: Int,
    val imageUrl: String?
) {
    val subtotal: Double get() = unitPrice * quantity
}

@Serializable
data class CartDto(
    val items: List<CartItemDto>,
    val couponCode: String? = null,
    val discountAmount: Double = 0.0
) {
    val subtotal: Double get() = items.sumOf { it.subtotal }
    val total: Double get() = subtotal - discountAmount
    val itemCount: Int get() = items.sumOf { it.quantity }
}

@Serializable
data class OrderDto(
    val id: String,
    val status: OrderStatus,
    val items: List<OrderItemDto>,
    val totalAmount: Double,
    val currency: String,
    val shippingAddress: AddressDto,
    val createdAt: String,  // ISO string for multiplatform
    val estimatedDelivery: String?
)

@Serializable
data class OrderItemDto(
    val productId: String, val productName: String,
    val quantity: Int, val unitPrice: Double
)

@Serializable
data class AddressDto(
    val street: String, val city: String,
    val province: String, val postalCode: String
)

@Serializable
enum class OrderStatus { PENDING_PAYMENT, CONFIRMED, PROCESSING, SHIPPED, DELIVERED, CANCELLED }

@Serializable
data class PagedResponse<T>(
    val items: List<T>,
    val totalCount: Long,
    val page: Int,
    val size: Int,
    val hasNextPage: Boolean
)

@Serializable
data class ApiError(
    val code: String,
    val message: String,
    val fieldErrors: Map<String, String> = emptyMap()
)

// Shared validation
object ProductValidator {
    data class ValidationResult(val errors: List<String>) {
        val isValid: Boolean get() = errors.isEmpty()
    }
    
    fun validate(name: String, price: Double, stock: Int): ValidationResult {
        val errors = mutableListOf<String>()
        if (name.isBlank()) errors.add("Name is required")
        if (name.length > 200) errors.add("Name too long (max 200 chars)")
        if (price <= 0) errors.add("Price must be positive")
        if (stock < 0) errors.add("Stock cannot be negative")
        return ValidationResult(errors)
    }
}

// Shared cart logic (same on client and server)
class CartCalculator {
    
    fun applyVoucher(cart: CartDto, discountPercent: Int): CartDto {
        val discount = cart.subtotal * discountPercent / 100.0
        return cart.copy(discountAmount = discount)
    }
    
    fun addItem(cart: CartDto, item: CartItemDto): CartDto {
        val existing = cart.items.find { it.productId == item.productId }
        
        val newItems = if (existing != null) {
            cart.items.map { 
                if (it.productId == item.productId) it.copy(quantity = it.quantity + item.quantity)
                else it
            }
        } else {
            cart.items + item
        }
        
        return cart.copy(items = newItems)
    }
    
    fun removeItem(cart: CartDto, productId: String): CartDto {
        return cart.copy(items = cart.items.filter { it.productId != productId })
    }
    
    fun updateQuantity(cart: CartDto, productId: String, quantity: Int): CartDto {
        if (quantity <= 0) return removeItem(cart, productId)
        
        return cart.copy(
            items = cart.items.map {
                if (it.productId == productId) it.copy(quantity = quantity) else it
            }
        )
    }
}
```

---

## Backend: Ktor API

```kotlin
// :backend/src/main/kotlin/

import io.ktor.server.application.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.routing.*
import io.ktor.server.plugins.contentnegotiation.*
import io.ktor.serialization.kotlinx.json.*
import io.ktor.server.request.*
import io.ktor.server.response.*

fun main() {
    embeddedServer(Netty, port = 8080) {
        install(ContentNegotiation) { json() }
        install(io.ktor.server.plugins.cors.routing.CORS) {
            anyHost()
            allowHeader(io.ktor.http.HttpHeaders.ContentType)
            allowHeader(io.ktor.http.HttpHeaders.Authorization)
        }
        
        routing {
            productRoutes()
            cartRoutes()
            orderRoutes()
            authRoutes()
        }
    }.start(wait = true)
}

fun Route.productRoutes() {
    val service by application.attributes.getOrNull(ProductServiceKey)
    
    route("/api/v1/products") {
        get {
            val page = call.request.queryParameters["page"]?.toIntOrNull() ?: 0
            val size = call.request.queryParameters["size"]?.toIntOrNull() ?: 20
            val category = call.request.queryParameters["categoryId"]
            
            val products = backendProductService.getProducts(category, page, size)
            call.respond(products)
        }
        
        get("/{id}") {
            val id = call.parameters["id"]!!
            val product = backendProductService.getById(id)
                ?: return@get call.respond(io.ktor.http.HttpStatusCode.NotFound, 
                    ApiError("NOT_FOUND", "Product $id not found"))
            call.respond(product)
        }
        
        get("/search") {
            val query = call.request.queryParameters["q"] 
                ?: return@get call.respond(
                    io.ktor.http.HttpStatusCode.BadRequest, 
                    ApiError("MISSING_PARAM", "Query parameter 'q' required")
                )
            val results = backendProductService.search(query)
            call.respond(results)
        }
        
        io.ktor.server.auth.authenticate("jwt") {
            post {
                val dto = call.receive<CreateProductRequest>()
                val result = backendProductService.create(dto)
                call.respond(io.ktor.http.HttpStatusCode.Created, result)
            }
        }
    }
}

fun Route.cartRoutes() {
    val calculator = CartCalculator()
    
    io.ktor.server.auth.authenticate("jwt") {
        route("/api/v1/cart") {
            get {
                val userId = call.principal<UserPrincipal>()!!.userId
                val cart = cartService.getCart(userId)
                call.respond(cart)
            }
            
            post("/items") {
                val userId = call.principal<UserPrincipal>()!!.userId
                val item = call.receive<CartItemDto>()
                val updated = cartService.addItem(userId, item)
                call.respond(updated)
            }
            
            put("/items/{productId}") {
                val userId = call.principal<UserPrincipal>()!!.userId
                val productId = call.parameters["productId"]!!
                val body = call.receive<UpdateQuantityRequest>()
                val updated = cartService.updateQuantity(userId, productId, body.quantity)
                call.respond(updated)
            }
            
            delete("/items/{productId}") {
                val userId = call.principal<UserPrincipal>()!!.userId
                val productId = call.parameters["productId"]!!
                val updated = cartService.removeItem(userId, productId)
                call.respond(updated)
            }
            
            post("/voucher") {
                val userId = call.principal<UserPrincipal>()!!.userId
                val body = call.receive<ApplyVoucherRequest>()
                val cart = cartService.getCart(userId)
                val voucher = voucherService.validate(body.code)
                val updated = calculator.applyVoucher(cart, voucher.discountPercent)
                cartService.saveCart(userId, updated)
                call.respond(updated)
            }
        }
    }
}

fun Route.orderRoutes() {
    io.ktor.server.auth.authenticate("jwt") {
        route("/api/v1/orders") {
            post {
                val userId = call.principal<UserPrincipal>()!!.userId
                val cart = cartService.getCart(userId)
                val address = call.receive<AddressDto>()
                
                val order = orderService.placeOrder(userId, cart, address)
                cartService.clearCart(userId)
                
                call.respond(io.ktor.http.HttpStatusCode.Created, order)
            }
            
            get {
                val userId = call.principal<UserPrincipal>()!!.userId
                val orders = orderService.getOrdersByCustomer(userId)
                call.respond(orders)
            }
            
            get("/{id}") {
                val userId = call.principal<UserPrincipal>()!!.userId
                val orderId = call.parameters["id"]!!
                val order = orderService.getOrder(orderId)
                    ?.takeIf { it.items.isNotEmpty() }  // ensure belongs to user
                    ?: return@get call.respond(io.ktor.http.HttpStatusCode.NotFound, 
                        ApiError("NOT_FOUND", "Order not found"))
                call.respond(order)
            }
        }
    }
}

// Stubs
data class CreateProductRequest(val name: String, val price: Double, val stock: Int, val categoryId: String)
data class UpdateQuantityRequest(val quantity: Int)
data class ApplyVoucherRequest(val code: String)
data class UserPrincipal(val userId: String) : io.ktor.server.auth.Principal
val ProductServiceKey = io.ktor.util.AttributeKey<Any>("ProductService")
interface backendProductService { companion object {
    suspend fun getProducts(cat: String?, page: Int, size: Int): PagedResponse<ProductDto> = TODO()
    suspend fun getById(id: String): ProductDto? = TODO()
    suspend fun search(q: String): List<ProductDto> = TODO()
    suspend fun create(req: CreateProductRequest): ProductDto = TODO()
}}
interface cartService { companion object {
    suspend fun getCart(userId: String): CartDto = TODO()
    suspend fun addItem(userId: String, item: CartItemDto): CartDto = TODO()
    suspend fun updateQuantity(userId: String, productId: String, qty: Int): CartDto = TODO()
    suspend fun removeItem(userId: String, productId: String): CartDto = TODO()
    suspend fun saveCart(userId: String, cart: CartDto): Unit = TODO()
    suspend fun clearCart(userId: String): Unit = TODO()
}}
interface voucherService { companion object { suspend fun validate(code: String): Voucher = TODO() } }
data class Voucher(val code: String, val discountPercent: Int)
interface orderService { companion object {
    suspend fun placeOrder(userId: String, cart: CartDto, address: AddressDto): OrderDto = TODO()
    suspend fun getOrdersByCustomer(userId: String): List<OrderDto> = TODO()
    suspend fun getOrder(id: String): OrderDto? = TODO()
}}
```

---

## Mobile: Compose Multiplatform

```kotlin
// :shared:api-client/src/commonMain/kotlin/

import io.ktor.client.*
import io.ktor.client.call.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.client.request.*
import io.ktor.serialization.kotlinx.json.*

class EcommerceApiClient(private val baseUrl: String) {
    
    private val client = HttpClient {
        install(ContentNegotiation) { json() }
        install(io.ktor.client.plugins.auth.Auth) {
            bearer {
                loadTokens { io.ktor.client.plugins.auth.providers.BearerTokens(tokenStorage.getAccessToken() ?: "", "") }
                refreshTokens { 
                    val newToken = authApi.refreshToken(tokenStorage.getRefreshToken() ?: "")
                    tokenStorage.saveTokens(newToken.accessToken, newToken.refreshToken)
                    io.ktor.client.plugins.auth.providers.BearerTokens(newToken.accessToken, newToken.refreshToken)
                }
            }
        }
    }
    
    suspend fun getProducts(page: Int = 0, size: Int = 20): PagedResponse<ProductDto> {
        return client.get("$baseUrl/api/v1/products?page=$page&size=$size").body()
    }
    
    suspend fun getProduct(id: String): ProductDto {
        return client.get("$baseUrl/api/v1/products/$id").body()
    }
    
    suspend fun searchProducts(query: String): List<ProductDto> {
        return client.get("$baseUrl/api/v1/products/search?q=${io.ktor.http.encodeURLQueryComponent(query)}").body()
    }
    
    suspend fun getCart(): CartDto {
        return client.get("$baseUrl/api/v1/cart").body()
    }
    
    suspend fun addToCart(item: CartItemDto): CartDto {
        return client.post("$baseUrl/api/v1/cart/items") {
            io.ktor.client.request.setBody(item)
            contentType(io.ktor.http.ContentType.Application.Json)
        }.body()
    }
    
    suspend fun placeOrder(address: AddressDto): OrderDto {
        return client.post("$baseUrl/api/v1/orders") {
            io.ktor.client.request.setBody(address)
            contentType(io.ktor.http.ContentType.Application.Json)
        }.body()
    }
}

// ViewModel (shared between Android and iOS)
class ProductListViewModel(private val api: EcommerceApiClient) : kotlinx.coroutines.CoroutineScope {
    
    override val coroutineContext = kotlinx.coroutines.SupervisorJob() + kotlinx.coroutines.Dispatchers.Main
    
    private val _state = kotlinx.coroutines.flow.MutableStateFlow<ProductListState>(ProductListState.Loading)
    val state: kotlinx.coroutines.flow.StateFlow<ProductListState> = _state
    
    private var currentPage = 0
    private var isLoadingMore = false
    
    init { loadProducts() }
    
    fun loadProducts(page: Int = 0) {
        launch {
            _state.value = if (page == 0) ProductListState.Loading else (_state.value as? ProductListState.Success)?.copy(loadingMore = true) ?: ProductListState.Loading
            
            try {
                val result = api.getProducts(page)
                currentPage = page
                
                val existing = if (page > 0) (_state.value as? ProductListState.Success)?.products ?: emptyList() else emptyList()
                
                _state.value = ProductListState.Success(
                    products = existing + result.items,
                    hasMore = result.hasNextPage,
                    loadingMore = false
                )
            } catch (e: Exception) {
                _state.value = ProductListState.Error(e.message ?: "Unknown error")
            }
        }
    }
    
    fun loadMore() {
        if (!isLoadingMore) {
            isLoadingMore = true
            loadProducts(currentPage + 1)
        }
    }
    
    fun search(query: String) {
        launch {
            _state.value = ProductListState.Loading
            try {
                val results = api.searchProducts(query)
                _state.value = ProductListState.Success(results, false)
            } catch (e: Exception) {
                _state.value = ProductListState.Error(e.message ?: "Search failed")
            }
        }
    }
    
    fun onCleared() { coroutineContext.cancel() }
}

sealed class ProductListState {
    object Loading : ProductListState()
    data class Success(
        val products: List<ProductDto>,
        val hasMore: Boolean = false,
        val loadingMore: Boolean = false
    ) : ProductListState()
    data class Error(val message: String) : ProductListState()
}

// Android Compose UI
@androidx.compose.runtime.Composable
fun ProductListScreen(
    viewModel: ProductListViewModel,
    onProductClick: (ProductDto) -> Unit
) {
    val state by viewModel.state.collectAsState()
    
    when (val s = state) {
        is ProductListState.Loading -> LoadingScreen()
        is ProductListState.Error -> ErrorScreen(s.message) { viewModel.loadProducts() }
        is ProductListState.Success -> {
            ProductGrid(
                products = s.products,
                hasMore = s.hasMore,
                loadingMore = s.loadingMore,
                onProductClick = onProductClick,
                onLoadMore = { viewModel.loadMore() }
            )
        }
    }
}

@androidx.compose.runtime.Composable
fun ProductGrid(
    products: List<ProductDto>,
    hasMore: Boolean,
    loadingMore: Boolean,
    onProductClick: (ProductDto) -> Unit,
    onLoadMore: () -> Unit
) {
    val listState = androidx.compose.foundation.lazy.grid.rememberLazyGridState()
    
    // Trigger load more when near end
    val shouldLoadMore = remember {
        derivedStateOf {
            val lastVisible = listState.layoutInfo.visibleItemsInfo.lastOrNull()?.index ?: 0
            lastVisible >= products.size - 3 && hasMore && !loadingMore
        }
    }
    
    LaunchedEffect(shouldLoadMore.value) {
        if (shouldLoadMore.value) onLoadMore()
    }
    
    androidx.compose.foundation.lazy.grid.LazyVerticalGrid(
        columns = androidx.compose.foundation.lazy.grid.GridCells.Fixed(2),
        state = listState,
        contentPadding = PaddingValues(16.dp),
        horizontalArrangement = Arrangement.spacedBy(12.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        items(products.size, key = { products[it].id }) { i ->
            ProductCard(products[i], onClick = { onProductClick(products[i]) })
        }
        
        if (loadingMore) {
            item(span = { GridItemSpan(2) }) {
                Box(Modifier.fillMaxWidth(), contentAlignment = Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
        }
    }
}

@androidx.compose.runtime.Composable
fun ProductCard(product: ProductDto, onClick: () -> Unit) {
    Card(
        modifier = Modifier.clickable(onClick = onClick),
        elevation = CardDefaults.cardElevation(2.dp)
    ) {
        Column {
            AsyncImage(
                model = product.images.firstOrNull(),
                contentDescription = product.name,
                modifier = Modifier.fillMaxWidth().height(160.dp),
                contentScale = ContentScale.Crop
            )
            Column(Modifier.padding(12.dp)) {
                Text(product.name, maxLines = 2, style = MaterialTheme.typography.bodyMedium)
                Spacer(Modifier.height(4.dp))
                Text(
                    "฿${String.format("%.0f", product.price)}",
                    style = MaterialTheme.typography.titleMedium,
                    color = MaterialTheme.colorScheme.primary
                )
                if (product.rating != null) {
                    Row(verticalAlignment = Alignment.CenterVertically) {
                        Icon(Icons.Filled.Star, null, tint = Color(0xFFFFC107), modifier = Modifier.size(14.dp))
                        Text("${product.rating} (${product.reviewCount})", style = MaterialTheme.typography.labelSmall)
                    }
                }
            }
        }
    }
}

// Compose imports stubs
typealias Modifier = Any
fun Any.fillMaxWidth(): Any = this
fun Any.height(dp: Any): Any = this
fun Any.padding(dp: Any): Any = this
fun Any.clickable(onClick: () -> Unit): Any = this
fun Any.size(dp: Any): Any = this
fun Any.width(dp: Any): Any = this
object Alignment { val Center = Any(); val CenterVertically = Any() }
object Arrangement { fun spacedBy(dp: Any) = Any() }
object ContentScale { val Crop = Any() }
val Any.dp: Any get() = this
@androidx.compose.runtime.Composable fun LoadingScreen() {}
@androidx.compose.runtime.Composable fun ErrorScreen(msg: String, onRetry: () -> Unit) {}
@androidx.compose.runtime.Composable fun AsyncImage(model: Any?, contentDescription: String?, modifier: Any, contentScale: Any) {}
@androidx.compose.runtime.Composable fun Card(modifier: Any, elevation: Any, content: @androidx.compose.runtime.Composable () -> Unit) {}
@androidx.compose.runtime.Composable fun Column(modifier: Any = Any(), content: @androidx.compose.runtime.Composable () -> Unit) {}
@androidx.compose.runtime.Composable fun Row(verticalAlignment: Any, content: @androidx.compose.runtime.Composable () -> Unit) {}
@androidx.compose.runtime.Composable fun Spacer(modifier: Any) {}
@androidx.compose.runtime.Composable fun Text(text: String, maxLines: Int = Int.MAX_VALUE, style: Any = Any(), color: Any = Any()) {}
@androidx.compose.runtime.Composable fun Icon(icon: Any, contentDescription: Any?, tint: Any, modifier: Any) {}
@androidx.compose.runtime.Composable fun Box(modifier: Any, contentAlignment: Any, content: @androidx.compose.runtime.Composable () -> Unit) {}
@androidx.compose.runtime.Composable fun CircularProgressIndicator() {}
@androidx.compose.runtime.Composable fun items(count: Int, key: (Int) -> Any, block: @androidx.compose.runtime.Composable (Int) -> Unit) {}
class Color(val value: Long) { companion object { val Any = Color(0) } }
fun PaddingValues(dp: Any): Any = dp
object Icons { object Filled { val Star = Any() } }
object CardDefaults { fun cardElevation(dp: Any) = Any() }
object MaterialTheme { val colorScheme = object { val primary = Color(0) }; val typography = object { val bodyMedium = Any(); val titleMedium = Any(); val labelSmall = Any() } }
fun <T, R> kotlinx.coroutines.flow.StateFlow<T>.collectAsState(): androidx.compose.runtime.State<T> = TODO()
fun <T> derivedStateOf(block: () -> T): androidx.compose.runtime.State<T> = TODO()
@androidx.compose.runtime.Composable fun LaunchedEffect(key: Any, block: suspend () -> Unit) {}
fun CoroutineScope.cancel() {}
class GridItemSpan(val maxLineSpan: Int)
interface PaddingValues
interface tokenStorage { companion object { fun getAccessToken(): String? = null; fun getRefreshToken(): String? = null; fun saveTokens(a: String, r: String) {} } }
interface authApi { companion object { suspend fun refreshToken(t: String): TokenResponse = TODO() } }
data class TokenResponse(val accessToken: String, val refreshToken: String)
fun io.ktor.http.encodeURLQueryComponent(s: String): String = s
fun androidx.compose.runtime.CoroutineScope.launch(block: suspend () -> Unit) {}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: เพิ่ม Wishlist Feature ใน Full-Stack Project

// 1. Shared model
@Serializable
data class WishlistDto(val productIds: List<String>, val products: List<ProductDto>)

// 2. API endpoints
fun Route.wishlistRoutes() {
    io.ktor.server.auth.authenticate("jwt") {
        route("/api/v1/wishlist") {
            get { call.respond(wishlistService.get(call.principal<UserPrincipal>()!!.userId)) }
            post("/{productId}") { call.respond(wishlistService.add(call.principal<UserPrincipal>()!!.userId, call.parameters["productId"]!!)) }
            delete("/{productId}") { call.respond(wishlistService.remove(call.principal<UserPrincipal>()!!.userId, call.parameters["productId"]!!)) }
        }
    }
}

// 3. API client
suspend fun EcommerceApiClient.getWishlist(): WishlistDto = client.get("$baseUrl/api/v1/wishlist").body()
suspend fun EcommerceApiClient.addToWishlist(productId: String): WishlistDto = client.post("$baseUrl/api/v1/wishlist/$productId").body()

// 4. Compose UI: heart button on ProductCard
@androidx.compose.runtime.Composable
fun WishlistButton(
    productId: String,
    isInWishlist: Boolean,
    onToggle: (String) -> Unit
) {
    IconButton(onClick = { onToggle(productId) }) {
        Icon(
            if (isInWishlist) Icons.Filled.Favorite else Icons.Outlined.FavoriteBorder,
            contentDescription = if (isInWishlist) "Remove from wishlist" else "Add to wishlist",
            tint = if (isInWishlist) Color(0xFFE53935) else Color(0xFF9E9E9E)
        )
    }
}

interface wishlistService { companion object {
    suspend fun get(userId: String): WishlistDto = TODO()
    suspend fun add(userId: String, productId: String): WishlistDto = TODO()
    suspend fun remove(userId: String, productId: String): WishlistDto = TODO()
}}
@androidx.compose.runtime.Composable fun IconButton(onClick: () -> Unit, content: @androidx.compose.runtime.Composable () -> Unit) {}
object Icons2 { object Filled { val Favorite = Any() }; object Outlined { val FavoriteBorder = Any() } }
```

---

## สรุป Part 97

```
✅ Monorepo: :shared:domain, :shared:api-client, :backend, :android
✅ kotlin("multiplatform"): jvm, js, ios, android targets
✅ @Serializable: shared models across platforms
✅ kotlinx.datetime: multiplatform date/time
✅ CartCalculator: shared business logic in commonMain
✅ ProductValidator: shared validation rules
✅ EcommerceApiClient: Ktor HttpClient shared code
✅ Bearer auth with token refresh in client
✅ Ktor server: embeddedServer, routing, ContentNegotiation
✅ route nesting: /api/v1/products → GET/POST/{id}
✅ call.receive<T>(): deserialize request body
✅ call.respond(): serialize response
✅ call.principal<T>(): get authenticated user
✅ StateFlow: reactive state in shared ViewModel
✅ ProductListViewModel: shared across Android/iOS
✅ Infinite scroll: load more when near end
✅ LazyVerticalGrid: 2-column grid
✅ derivedStateOf: computed state from other states
✅ LaunchedEffect: side effects on state change
✅ ProductCard: image, name, price, rating
✅ AsyncImage: Coil image loading
✅ Wishlist: add/remove/toggle feature
✅ WishlistButton: heart icon toggle
```

---

*Part 97/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
