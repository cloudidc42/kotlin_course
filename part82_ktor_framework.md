# Part 82: Ktor Framework

## สารบัญ
1. [Ktor คืออะไร](#ktor-คืออะไร)
2. [Routing DSL](#routing-dsl)
3. [Plugins ที่สำคัญ](#plugins-ที่สำคัญ)
4. [Authentication & JWT](#authentication--jwt)
5. [Real-time WebSocket](#real-time-websocket)
6. [Testing Ktor](#testing-ktor)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Ktor คืออะไร

```
Ktor: Kotlin-native web framework จาก JetBrains

เปรียบเทียบกับ Spring Boot:
Feature           Spring Boot           Ktor
────────────────────────────────────────────
Startup time      ~3-10s                ~0.1-0.5s
Memory            ~256MB+               ~50-100MB
Coroutine-native  No (reactive wrapper) YES
Configuration     Convention/annotation Lambda DSL
DI                Complex (Spring IoC)  Simple/manual
Learning curve    Steep                 Moderate
Ecosystem         Huge                  Growing

Ktor เหมาะกับ:
✅ Microservices ขนาดเล็ก
✅ APIs ที่ต้องการ performance สูง
✅ KMP: shared server+mobile HTTP client
✅ Serverless/Lambda (fast startup)
✅ WebSocket-heavy apps
✅ ทีมที่ต้องการ full control

Spring Boot เหมาะกับ:
✅ Enterprise apps
✅ Ecosystem ที่ครบ
✅ ทีมที่มีพื้นฐาน Spring
```

---

## Project Setup

```kotlin
// build.gradle.kts
plugins {
    kotlin("jvm") version "1.9.22"
    kotlin("plugin.serialization") version "1.9.22"
    application
}

val ktor_version = "2.3.7"

dependencies {
    // Ktor Server
    implementation("io.ktor:ktor-server-core:$ktor_version")
    implementation("io.ktor:ktor-server-netty:$ktor_version")
    implementation("io.ktor:ktor-server-content-negotiation:$ktor_version")
    implementation("io.ktor:ktor-serialization-kotlinx-json:$ktor_version")
    implementation("io.ktor:ktor-server-auth:$ktor_version")
    implementation("io.ktor:ktor-server-auth-jwt:$ktor_version")
    implementation("io.ktor:ktor-server-websockets:$ktor_version")
    implementation("io.ktor:ktor-server-call-logging:$ktor_version")
    implementation("io.ktor:ktor-server-call-id:$ktor_version")
    implementation("io.ktor:ktor-server-request-validation:$ktor_version")
    implementation("io.ktor:ktor-server-status-pages:$ktor_version")
    implementation("io.ktor:ktor-server-rate-limit:$ktor_version")
    implementation("io.ktor:ktor-server-cors:$ktor_version")
    implementation("io.ktor:ktor-server-compression:$ktor_version")
    implementation("io.ktor:ktor-server-default-headers:$ktor_version")
    implementation("io.ktor:ktor-server-metrics-micrometer:$ktor_version")
    
    // Database
    implementation("org.jetbrains.exposed:exposed-core:0.45.0")
    implementation("org.jetbrains.exposed:exposed-dao:0.45.0")
    implementation("org.jetbrains.exposed:exposed-jdbc:0.45.0")
    implementation("org.postgresql:postgresql:42.7.1")
    implementation("com.zaxxer:HikariCP:5.1.0")
    
    // DI
    implementation("io.insert-koin:koin-ktor:3.5.3")
    implementation("io.insert-koin:koin-logger-slf4j:3.5.3")
    
    // Auth
    implementation("com.auth0:java-jwt:4.4.0")
    
    // Logging
    implementation("ch.qos.logback:logback-classic:1.4.14")
    
    // Testing
    testImplementation("io.ktor:ktor-server-test-host:$ktor_version")
    testImplementation("io.ktor:ktor-client-content-negotiation:$ktor_version")
    testImplementation(kotlin("test"))
}

application {
    mainClass.set("com.ecommerce.ApplicationKt")
}
```

---

## Main Application Setup

```kotlin
// Application.kt
import io.ktor.server.application.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*

fun main() {
    embeddedServer(
        factory = Netty,
        port = System.getenv("PORT")?.toInt() ?: 8080,
        host = "0.0.0.0",
        module = Application::module
    ).start(wait = true)
}

fun Application.module() {
    configureDependencyInjection()
    configureDatabase()
    configurePlugins()
    configureAuthentication()
    configureRouting()
}

// Plugins.kt
import io.ktor.serialization.kotlinx.json.*
import io.ktor.server.plugins.contentnegotiation.*
import io.ktor.server.plugins.calllogging.*
import io.ktor.server.plugins.callid.*
import io.ktor.server.plugins.statuspages.*
import io.ktor.server.plugins.requestvalidation.*
import io.ktor.server.plugins.cors.routing.*
import io.ktor.server.plugins.compression.*
import io.ktor.server.plugins.defaultheaders.*
import io.ktor.server.plugins.ratelimit.*
import kotlinx.serialization.json.Json
import kotlin.time.Duration.Companion.minutes

fun Application.configurePlugins() {
    install(ContentNegotiation) {
        json(Json {
            prettyPrint = false
            isLenient = true
            ignoreUnknownKeys = true
            encodeDefaults = true
        })
    }
    
    install(CallLogging) {
        level = org.slf4j.event.Level.INFO
        callIdMdc("callId")
        filter { call ->
            call.request.path().startsWith("/api")
        }
    }
    
    install(CallId) {
        retrieveFromHeader(io.ktor.http.HttpHeaders.XRequestId)
        generate { java.util.UUID.randomUUID().toString() }
        verify { callId: String -> callId.isNotEmpty() }
    }
    
    install(DefaultHeaders) {
        header("X-Content-Type-Options", "nosniff")
        header("X-Frame-Options", "DENY")
        header("X-XSS-Protection", "1; mode=block")
        header("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
    }
    
    install(Compression) {
        gzip { priority = 1.0 }
        deflate { priority = 10.0 }
    }
    
    install(CORS) {
        allowHost("localhost:3000")
        allowHost("app.ecommerce.com", schemes = listOf("https"))
        allowHeader(io.ktor.http.HttpHeaders.ContentType)
        allowHeader(io.ktor.http.HttpHeaders.Authorization)
        allowMethod(io.ktor.http.HttpMethod.Options)
        allowMethod(io.ktor.http.HttpMethod.Put)
        allowMethod(io.ktor.http.HttpMethod.Delete)
    }
    
    install(RateLimit) {
        global {
            rateLimiter(limit = 100, refillPeriod = 1.minutes)
        }
        register(RateLimitName("auth")) {
            rateLimiter(limit = 10, refillPeriod = 1.minutes)
            requestKey { call -> call.request.local.remoteAddress }
        }
    }
    
    install(StatusPages) {
        exception<ValidationException> { call, cause ->
            call.respond(
                io.ktor.http.HttpStatusCode.BadRequest,
                ErrorResponse(
                    code = "VALIDATION_ERROR",
                    message = cause.message ?: "Validation failed",
                    details = cause.errors
                )
            )
        }
        
        exception<NotFoundException> { call, cause ->
            call.respond(
                io.ktor.http.HttpStatusCode.NotFound,
                ErrorResponse(code = "NOT_FOUND", message = cause.message ?: "Not found")
            )
        }
        
        exception<UnauthorizedException> { call, cause ->
            call.respond(
                io.ktor.http.HttpStatusCode.Unauthorized,
                ErrorResponse(code = "UNAUTHORIZED", message = cause.message ?: "Unauthorized")
            )
        }
        
        exception<Exception> { call, cause ->
            call.application.log.error("Unhandled exception", cause)
            call.respond(
                io.ktor.http.HttpStatusCode.InternalServerError,
                ErrorResponse(code = "INTERNAL_ERROR", message = "An internal error occurred")
            )
        }
    }
    
    install(RequestValidation) {
        validate<CreateProductRequest> { request ->
            val errors = mutableListOf<String>()
            if (request.name.isBlank()) errors.add("Name is required")
            if (request.price <= 0) errors.add("Price must be positive")
            if (request.stockQuantity < 0) errors.add("Stock cannot be negative")
            if (errors.isEmpty()) ValidationResult.Valid
            else ValidationResult.Invalid(errors.joinToString(", "))
        }
    }
}

@kotlinx.serialization.Serializable
data class ErrorResponse(
    val code: String,
    val message: String,
    val details: List<String> = emptyList()
)

class ValidationException(val errors: List<String>) : Exception(errors.joinToString(", "))
class NotFoundException(message: String) : Exception(message)
class UnauthorizedException(message: String) : Exception(message)
```

---

## Routing DSL

```kotlin
// Routes.kt
import io.ktor.server.routing.*
import io.ktor.server.request.*
import io.ktor.server.response.*
import io.ktor.server.application.*
import io.ktor.server.auth.*
import io.ktor.server.auth.jwt.*
import io.ktor.http.*
import org.koin.ktor.ext.inject

fun Application.configureRouting() {
    routing {
        healthRoutes()
        apiRoutes()
    }
}

fun Route.healthRoutes() {
    route("/health") {
        get {
            call.respond(mapOf("status" to "ok", "version" to "1.0.0"))
        }
        
        get("/ready") {
            // Check DB connection, cache, etc.
            call.respond(mapOf("status" to "ready"))
        }
    }
}

fun Route.apiRoutes() {
    route("/api/v1") {
        val authService by application.inject<AuthService>()
        val productService by application.inject<ProductService>()
        val orderService by application.inject<OrderService>()
        
        // Public routes
        authRoutes(authService)
        
        // Protected routes
        authenticate("jwt") {
            productRoutes(productService)
            orderRoutes(orderService)
            
            // Admin routes
            authenticate("admin") {
                adminRoutes(productService)
            }
        }
    }
}

// Auth routes
fun Route.authRoutes(authService: AuthService) {
    route("/auth") {
        rateLimit(RateLimitName("auth")) {
            post("/register") {
                val request = call.receive<RegisterRequest>()
                val user = authService.register(request)
                call.respond(HttpStatusCode.Created, user.toResponse())
            }
            
            post("/login") {
                val request = call.receive<LoginRequest>()
                val tokens = authService.login(request)
                call.respond(tokens)
            }
            
            post("/refresh") {
                val request = call.receive<RefreshTokenRequest>()
                val tokens = authService.refreshToken(request.refreshToken)
                call.respond(tokens)
            }
        }
        
        authenticate("jwt") {
            post("/logout") {
                val principal = call.principal<JWTPrincipal>()!!
                authService.logout(principal.subject!!)
                call.respond(HttpStatusCode.NoContent)
            }
            
            get("/me") {
                val principal = call.principal<JWTPrincipal>()!!
                val user = authService.getUser(principal.subject!!)
                call.respond(user.toResponse())
            }
        }
    }
}

// Product routes
fun Route.productRoutes(productService: ProductService) {
    route("/products") {
        get {
            val page = call.request.queryParameters["page"]?.toIntOrNull() ?: 0
            val size = call.request.queryParameters["size"]?.toIntOrNull() ?: 20
            val search = call.request.queryParameters["search"]
            val categoryId = call.request.queryParameters["categoryId"]
            
            val products = productService.getProducts(
                ProductFilter(page = page, size = size, searchQuery = search, categoryId = categoryId)
            )
            call.respond(products)
        }
        
        get("/{id}") {
            val id = call.parameters["id"] ?: throw ValidationException(listOf("ID is required"))
            val product = productService.getById(id) ?: throw NotFoundException("Product $id not found")
            call.respond(product.toResponse())
        }
        
        post("/search") {
            val request = call.receive<SearchRequest>()
            val results = productService.search(request)
            call.respond(results)
        }
    }
}

// Admin routes (requires ADMIN role)
fun Route.adminRoutes(productService: ProductService) {
    route("/admin/products") {
        post {
            val request = call.receive<CreateProductRequest>()
            val product = productService.create(request)
            call.respond(HttpStatusCode.Created, product.toResponse())
        }
        
        put("/{id}") {
            val id = call.parameters["id"]!!
            val request = call.receive<UpdateProductRequest>()
            val product = productService.update(id, request)
            call.respond(product.toResponse())
        }
        
        delete("/{id}") {
            val id = call.parameters["id"]!!
            productService.delete(id)
            call.respond(HttpStatusCode.NoContent)
        }
        
        patch("/{id}/stock") {
            val id = call.parameters["id"]!!
            val request = call.receive<AdjustStockRequest>()
            val product = productService.adjustStock(id, request.delta)
            call.respond(product.toResponse())
        }
    }
}

// Order routes
fun Route.orderRoutes(orderService: OrderService) {
    route("/orders") {
        get {
            val principal = call.principal<JWTPrincipal>()!!
            val userId = principal.subject!!
            val orders = orderService.getUserOrders(userId)
            call.respond(orders.map { it.toResponse() })
        }
        
        post {
            val principal = call.principal<JWTPrincipal>()!!
            val userId = principal.subject!!
            val request = call.receive<CreateOrderRequest>()
            val order = orderService.placeOrder(userId, request)
            call.respond(HttpStatusCode.Created, order.toResponse())
        }
        
        get("/{id}") {
            val principal = call.principal<JWTPrincipal>()!!
            val userId = principal.subject!!
            val id = call.parameters["id"]!!
            val order = orderService.getOrder(id, userId) ?: throw NotFoundException("Order $id not found")
            call.respond(order.toResponse())
        }
        
        post("/{id}/cancel") {
            val principal = call.principal<JWTPrincipal>()!!
            val userId = principal.subject!!
            val id = call.parameters["id"]!!
            orderService.cancelOrder(id, userId)
            call.respond(HttpStatusCode.NoContent)
        }
    }
}

// Extension helpers
fun ApplicationCall.getPage() = request.queryParameters["page"]?.toIntOrNull() ?: 0
fun ApplicationCall.getSize() = request.queryParameters["size"]?.toIntOrNull() ?: 20
fun ApplicationCall.getPathId() = parameters["id"] ?: throw ValidationException(listOf("ID required"))
fun ApplicationCall.currentUserId() = principal<JWTPrincipal>()?.subject ?: throw UnauthorizedException("Not authenticated")

// Serializable request/response models
@kotlinx.serialization.Serializable
data class RegisterRequest(val email: String, val password: String, val name: String)

@kotlinx.serialization.Serializable
data class LoginRequest(val email: String, val password: String)

@kotlinx.serialization.Serializable
data class RefreshTokenRequest(val refreshToken: String)

@kotlinx.serialization.Serializable
data class CreateProductRequest(val name: String, val price: Double, val stockQuantity: Int, val categoryId: String)

@kotlinx.serialization.Serializable
data class UpdateProductRequest(val name: String? = null, val price: Double? = null)

@kotlinx.serialization.Serializable
data class AdjustStockRequest(val delta: Int)

@kotlinx.serialization.Serializable
data class SearchRequest(val query: String, val page: Int = 0, val size: Int = 20)

@kotlinx.serialization.Serializable
data class CreateOrderRequest(val items: List<OrderItemRequest>)

@kotlinx.serialization.Serializable
data class OrderItemRequest(val productId: String, val quantity: Int)
```

---

## Authentication & JWT

```kotlin
// Auth.kt
import io.ktor.server.auth.*
import io.ktor.server.auth.jwt.*
import com.auth0.jwt.JWT
import com.auth0.jwt.algorithms.Algorithm

fun Application.configureAuthentication() {
    val jwtConfig = JwtConfig(
        secret = environment.config.property("jwt.secret").getString(),
        issuer = environment.config.property("jwt.issuer").getString(),
        audience = environment.config.property("jwt.audience").getString(),
        expirationMinutes = environment.config.property("jwt.expirationMinutes").getString().toLong()
    )
    
    install(Authentication) {
        jwt("jwt") {
            realm = "ECommerce API"
            verifier(
                JWT.require(Algorithm.HMAC256(jwtConfig.secret))
                    .withIssuer(jwtConfig.issuer)
                    .withAudience(jwtConfig.audience)
                    .build()
            )
            
            validate { credential ->
                val userId = credential.payload.subject
                val email = credential.payload.getClaim("email").asString()
                val roles = credential.payload.getClaim("roles").asList(String::class.java)
                
                if (userId != null && email != null) {
                    UserPrincipal(userId = userId, email = email, roles = roles)
                } else null
            }
            
            challenge { _, _ ->
                call.respond(
                    io.ktor.http.HttpStatusCode.Unauthorized,
                    ErrorResponse("UNAUTHORIZED", "Token is not valid or has expired")
                )
            }
        }
        
        // Admin role check
        jwt("admin") {
            realm = "ECommerce Admin API"
            verifier(
                JWT.require(Algorithm.HMAC256(jwtConfig.secret))
                    .withIssuer(jwtConfig.issuer)
                    .build()
            )
            
            validate { credential ->
                val userId = credential.payload.subject ?: return@validate null
                val roles = credential.payload.getClaim("roles").asList(String::class.java) ?: emptyList()
                
                if ("ADMIN" in roles) {
                    UserPrincipal(userId = userId, email = "", roles = roles)
                } else null
            }
            
            challenge { _, _ ->
                call.respond(
                    io.ktor.http.HttpStatusCode.Forbidden,
                    ErrorResponse("FORBIDDEN", "Admin access required")
                )
            }
        }
    }
}

data class UserPrincipal(
    val userId: String,
    val email: String,
    val roles: List<String>
) : io.ktor.server.auth.Principal

data class JwtConfig(
    val secret: String,
    val issuer: String,
    val audience: String,
    val expirationMinutes: Long
)

// Token generation
class JwtTokenService(private val config: JwtConfig) {
    
    fun generateAccessToken(userId: String, email: String, roles: List<String>): String {
        return JWT.create()
            .withIssuer(config.issuer)
            .withAudience(config.audience)
            .withSubject(userId)
            .withClaim("email", email)
            .withClaim("roles", roles)
            .withClaim("type", "access")
            .withExpiresAt(
                java.util.Date(System.currentTimeMillis() + config.expirationMinutes * 60 * 1000)
            )
            .withIssuedAt(java.util.Date())
            .sign(Algorithm.HMAC256(config.secret))
    }
    
    fun generateRefreshToken(userId: String): String {
        return JWT.create()
            .withIssuer(config.issuer)
            .withSubject(userId)
            .withClaim("type", "refresh")
            .withExpiresAt(
                java.util.Date(System.currentTimeMillis() + 7 * 24 * 60 * 60 * 1000L) // 7 days
            )
            .withIssuedAt(java.util.Date())
            .sign(Algorithm.HMAC256(config.secret))
    }
    
    fun verifyRefreshToken(token: String): String? {
        return try {
            val verifier = JWT.require(Algorithm.HMAC256(config.secret))
                .withIssuer(config.issuer)
                .withClaim("type", "refresh")
                .build()
            verifier.verify(token).subject
        } catch (e: Exception) {
            null
        }
    }
}

// AuthService
class AuthService(
    private val userRepository: UserRepository,
    private val tokenService: JwtTokenService,
    private val passwordEncoder: PasswordEncoder
) {
    
    suspend fun register(request: RegisterRequest): User {
        if (userRepository.findByEmail(request.email) != null) {
            throw ValidationException(listOf("Email already registered"))
        }
        
        val user = User(
            id = java.util.UUID.randomUUID().toString(),
            email = request.email,
            name = request.name,
            passwordHash = passwordEncoder.encode(request.password),
            roles = listOf("USER")
        )
        
        return userRepository.save(user)
    }
    
    suspend fun login(request: LoginRequest): TokenPairResponse {
        val user = userRepository.findByEmail(request.email)
            ?: throw UnauthorizedException("Invalid email or password")
        
        if (!passwordEncoder.matches(request.password, user.passwordHash)) {
            throw UnauthorizedException("Invalid email or password")
        }
        
        return TokenPairResponse(
            accessToken = tokenService.generateAccessToken(user.id, user.email, user.roles),
            refreshToken = tokenService.generateRefreshToken(user.id),
            userId = user.id
        )
    }
    
    suspend fun refreshToken(refreshToken: String): TokenPairResponse {
        val userId = tokenService.verifyRefreshToken(refreshToken)
            ?: throw UnauthorizedException("Invalid refresh token")
        
        val user = userRepository.findById(userId)
            ?: throw NotFoundException("User not found")
        
        return TokenPairResponse(
            accessToken = tokenService.generateAccessToken(user.id, user.email, user.roles),
            refreshToken = tokenService.generateRefreshToken(user.id),
            userId = user.id
        )
    }
    
    suspend fun logout(userId: String) {
        // Invalidate refresh token (add to blocklist in Redis)
        tokenService.revokeUserTokens(userId)
    }
    
    suspend fun getUser(userId: String): User {
        return userRepository.findById(userId) ?: throw NotFoundException("User not found")
    }
}

@kotlinx.serialization.Serializable
data class TokenPairResponse(
    val accessToken: String,
    val refreshToken: String,
    val userId: String
)

interface PasswordEncoder {
    fun encode(raw: String): String
    fun matches(raw: String, encoded: String): Boolean
}

class BCryptPasswordEncoder : PasswordEncoder {
    override fun encode(raw: String) = org.mindrot.jbcrypt.BCrypt.hashpw(raw, org.mindrot.jbcrypt.BCrypt.gensalt())
    override fun matches(raw: String, encoded: String) = org.mindrot.jbcrypt.BCrypt.checkpw(raw, encoded)
}

fun JwtTokenService.revokeUserTokens(userId: String) { /* store in Redis blocklist */ }
```

---

## Real-time WebSocket

```kotlin
// WebSocket.kt
import io.ktor.server.websocket.*
import io.ktor.websocket.*
import kotlinx.coroutines.channels.consumeEach
import kotlinx.serialization.encodeToString
import kotlinx.serialization.json.Json
import java.util.concurrent.ConcurrentHashMap

fun Application.configureWebSocket() {
    install(WebSockets) {
        pingPeriod = java.time.Duration.ofSeconds(15)
        timeout = java.time.Duration.ofSeconds(15)
        maxFrameSize = Long.MAX_VALUE
        masking = false
    }
}

// Connection manager
class ConnectionManager {
    private val connections = ConcurrentHashMap<String, WebSocketSession>()
    private val userConnections = ConcurrentHashMap<String, MutableSet<String>>()
    
    fun addConnection(connectionId: String, userId: String, session: WebSocketSession) {
        connections[connectionId] = session
        userConnections.getOrPut(userId) { ConcurrentHashMap.newKeySet() }.add(connectionId)
    }
    
    fun removeConnection(connectionId: String, userId: String) {
        connections.remove(connectionId)
        userConnections[userId]?.remove(connectionId)
    }
    
    suspend fun sendToUser(userId: String, message: WsMessage) {
        val userConns = userConnections[userId] ?: return
        val json = Json.encodeToString(message)
        userConns.forEach { connId ->
            connections[connId]?.send(Frame.Text(json))
        }
    }
    
    suspend fun broadcast(message: WsMessage) {
        val json = Json.encodeToString(message)
        connections.values.forEach { session ->
            try {
                session.send(Frame.Text(json))
            } catch (e: Exception) {
                // Connection closed
            }
        }
    }
    
    fun getActiveCount() = connections.size
}

@kotlinx.serialization.Serializable
sealed class WsMessage {
    abstract val type: String
}

@kotlinx.serialization.Serializable
data class OrderStatusUpdate(
    override val type: String = "ORDER_STATUS_UPDATE",
    val orderId: String,
    val status: String,
    val timestamp: Long = System.currentTimeMillis()
) : WsMessage()

@kotlinx.serialization.Serializable
data class StockUpdate(
    override val type: String = "STOCK_UPDATE",
    val productId: String,
    val stockQuantity: Int
) : WsMessage()

@kotlinx.serialization.Serializable
data class PriceUpdate(
    override val type: String = "PRICE_UPDATE",
    val productId: String,
    val newPrice: Double
) : WsMessage()

// WebSocket routes
fun Route.webSocketRoutes(connectionManager: ConnectionManager) {
    authenticate("jwt") {
        webSocket("/ws/orders") {
            val principal = call.principal<UserPrincipal>()!!
            val userId = principal.userId
            val connectionId = java.util.UUID.randomUUID().toString()
            
            connectionManager.addConnection(connectionId, userId, this)
            
            try {
                // Send welcome message
                send(Frame.Text(Json.encodeToString(
                    mapOf("type" to "CONNECTED", "userId" to userId)
                )))
                
                incoming.consumeEach { frame ->
                    when (frame) {
                        is Frame.Text -> {
                            val text = frame.readText()
                            // Handle client messages (subscribe, ping, etc.)
                            handleClientMessage(text, userId, connectionManager)
                        }
                        is Frame.Ping -> send(Frame.Pong(frame.data))
                        is Frame.Close -> return@webSocket
                        else -> {}
                    }
                }
            } finally {
                connectionManager.removeConnection(connectionId, userId)
            }
        }
    }
    
    // Public stock updates (no auth required)
    webSocket("/ws/stock") {
        val connectionId = java.util.UUID.randomUUID().toString()
        connectionManager.addConnection(connectionId, "anonymous", this)
        
        try {
            incoming.consumeEach { frame ->
                if (frame is Frame.Ping) send(Frame.Pong(frame.data))
            }
        } finally {
            connectionManager.removeConnection(connectionId, "anonymous")
        }
    }
}

private suspend fun DefaultWebSocketSession.handleClientMessage(
    text: String,
    userId: String,
    connectionManager: ConnectionManager
) {
    try {
        val message = Json.decodeFromString<ClientMessage>(text)
        when (message.type) {
            "PING" -> send(Frame.Text("""{"type":"PONG"}"""))
            "SUBSCRIBE_ORDER" -> {
                val orderId = message.data["orderId"] as? String ?: return
                // Subscribe to specific order updates
                send(Frame.Text("""{"type":"SUBSCRIBED","orderId":"$orderId"}"""))
            }
        }
    } catch (e: Exception) {
        send(Frame.Text("""{"type":"ERROR","message":"Invalid message format"}"""))
    }
}

@kotlinx.serialization.Serializable
data class ClientMessage(
    val type: String,
    val data: Map<String, String> = emptyMap()
)

// Event publisher that broadcasts to WebSocket clients
class OrderEventPublisher(private val connectionManager: ConnectionManager) {
    
    suspend fun publishOrderStatusChange(userId: String, orderId: String, status: String) {
        connectionManager.sendToUser(userId, OrderStatusUpdate(orderId = orderId, status = status))
    }
    
    suspend fun publishStockChange(productId: String, newStock: Int) {
        connectionManager.broadcast(StockUpdate(productId = productId, stockQuantity = newStock))
    }
}
```

---

## Testing Ktor

```kotlin
// ProductRoutesTest.kt
import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.http.*
import io.ktor.serialization.kotlinx.json.*
import io.ktor.server.testing.*
import io.ktor.client.plugins.contentnegotiation.*
import kotlinx.serialization.json.Json
import kotlinx.serialization.decodeFromString
import kotlin.test.*

class ProductRoutesTest {
    
    private fun testApp(block: suspend ApplicationTestBuilder.() -> Unit) {
        testApplication {
            application {
                configureDependencyInjection()
                configurePlugins()
                configureAuthentication()
                configureRouting()
            }
            block()
        }
    }
    
    private fun ApplicationTestBuilder.createClient() = createClient {
        install(ContentNegotiation) {
            json(Json { ignoreUnknownKeys = true })
        }
    }
    
    @Test
    fun `GET products returns 200`() = testApp {
        val client = createClient()
        val response = client.get("/api/v1/products")
        
        assertEquals(HttpStatusCode.OK, response.status)
        val body = Json.decodeFromString<PagedResult<Any>>(response.bodyAsText())
        assertNotNull(body)
    }
    
    @Test
    fun `GET product by id returns 404 when not found`() = testApp {
        val client = createClient()
        val response = client.get("/api/v1/products/non-existent-id")
        
        assertEquals(HttpStatusCode.NotFound, response.status)
        val error = Json.decodeFromString<ErrorResponse>(response.bodyAsText())
        assertEquals("NOT_FOUND", error.code)
    }
    
    @Test
    fun `POST product without auth returns 401`() = testApp {
        val client = createClient()
        val response = client.post("/api/v1/admin/products") {
            contentType(ContentType.Application.Json)
            setBody("""{"name":"Test","price":100.0,"stockQuantity":10,"categoryId":"cat1"}""")
        }
        
        assertEquals(HttpStatusCode.Unauthorized, response.status)
    }
    
    @Test
    fun `register and login flow works`() = testApp {
        val client = createClient()
        
        // Register
        val registerResponse = client.post("/api/v1/auth/register") {
            contentType(ContentType.Application.Json)
            setBody(RegisterRequest(
                email = "test@example.com",
                password = "password123",
                name = "Test User"
            ))
        }
        assertEquals(HttpStatusCode.Created, registerResponse.status)
        
        // Login
        val loginResponse = client.post("/api/v1/auth/login") {
            contentType(ContentType.Application.Json)
            setBody(LoginRequest(email = "test@example.com", password = "password123"))
        }
        assertEquals(HttpStatusCode.OK, loginResponse.status)
        
        val tokens = Json.decodeFromString<TokenPairResponse>(loginResponse.bodyAsText())
        assertNotNull(tokens.accessToken)
        assertNotNull(tokens.refreshToken)
        
        // Access protected endpoint
        val meResponse = client.get("/api/v1/auth/me") {
            bearerAuth(tokens.accessToken)
        }
        assertEquals(HttpStatusCode.OK, meResponse.status)
    }
    
    @Test
    fun `WebSocket connection established`() = testApp {
        val client = createClient()
        // Auth first
        val tokens = login(client)
        
        client.webSocket("/ws/orders", {
            bearerAuth(tokens.accessToken)
        }) {
            val frame = incoming.receive()
            assertTrue(frame is Frame.Text)
            val text = (frame as Frame.Text).readText()
            assertTrue(text.contains("CONNECTED"))
        }
    }
    
    @Test
    fun `rate limit returns 429 after limit exceeded`() = testApp {
        val client = createClient()
        repeat(11) {
            client.post("/api/v1/auth/login") {
                contentType(ContentType.Application.Json)
                setBody(LoginRequest("x@x.com", "wrong"))
            }
        }
        
        val response = client.post("/api/v1/auth/login") {
            contentType(ContentType.Application.Json)
            setBody(LoginRequest("x@x.com", "wrong"))
        }
        
        assertEquals(HttpStatusCode.TooManyRequests, response.status)
    }
    
    private suspend fun login(client: io.ktor.client.HttpClient): TokenPairResponse {
        val response = client.post("/api/v1/auth/login") {
            contentType(ContentType.Application.Json)
            setBody(LoginRequest("admin@ecommerce.com", "admin123"))
        }
        return Json.decodeFromString(response.bodyAsText())
    }
}

// Integration test with Testcontainers
class ProductIntegrationTest {
    companion object {
        val postgres = org.testcontainers.containers.PostgreSQLContainer<Nothing>("postgres:16-alpine")
        val redis = org.testcontainers.containers.GenericContainer<Nothing>("redis:7-alpine")
        
        @BeforeAll @JvmStatic
        fun setup() {
            postgres.start()
            redis.withExposedPorts(6379).start()
        }
        
        @AfterAll @JvmStatic
        fun teardown() {
            postgres.stop()
            redis.stop()
        }
    }
    
    @Test
    fun `full product CRUD flow`() = testApplication {
        environment {
            config = io.ktor.server.config.MapApplicationConfig(
                "db.url" to postgres.jdbcUrl,
                "db.user" to postgres.username,
                "db.password" to postgres.password,
                "redis.url" to "redis://localhost:${redis.getMappedPort(6379)}"
            )
        }
        
        application {
            module()
        }
        
        val client = createClient {
            install(ContentNegotiation) { json() }
        }
        
        // Create product (as admin)
        val createResponse = client.post("/api/v1/admin/products") {
            bearerAuth(getAdminToken())
            contentType(ContentType.Application.Json)
            setBody(CreateProductRequest("Test Product", 99.99, 100, "electronics"))
        }
        assertEquals(HttpStatusCode.Created, createResponse.status)
    }
    
    private fun getAdminToken(): String = TODO("Generate admin JWT for test")
}
```

---

## Dependency Injection ด้วย Koin

```kotlin
// DI.kt
import io.insert.koin.dsl.module
import io.ktor.server.application.*
import io.insert.koin.ktor.ext.inject
import io.insert.koin.ktor.plugin.Koin

fun Application.configureDependencyInjection() {
    install(Koin) {
        slf4jLogger()
        modules(appModule)
    }
}

val appModule = module {
    // Config
    single { JwtConfig(
        secret = System.getenv("JWT_SECRET") ?: "dev-secret-please-change",
        issuer = "ecommerce-api",
        audience = "ecommerce-users",
        expirationMinutes = 60L
    )}
    
    // Services
    single { JwtTokenService(get()) }
    single<PasswordEncoder> { BCryptPasswordEncoder() }
    single { AuthService(get(), get(), get()) }
    single { ProductService(get(), get()) }
    single { OrderService(get(), get(), get()) }
    
    // Repositories
    single<UserRepository> { ExposedUserRepository(get()) }
    single<ProductRepository> { ExposedProductRepository(get()) }
    single<OrderRepository> { ExposedOrderRepository(get()) }
    
    // Infrastructure
    single { ConnectionManager() }
    single { OrderEventPublisher(get()) }
    single { createDataSource() }
    single { io.lettuce.core.RedisClient.create("redis://localhost:6379") }
}

fun createDataSource(): javax.sql.DataSource {
    return com.zaxxer.hikari.HikariDataSource(com.zaxxer.hikari.HikariConfig().apply {
        jdbcUrl = System.getenv("DB_URL") ?: "jdbc:postgresql://localhost:5432/ecommerce"
        username = System.getenv("DB_USER") ?: "postgres"
        password = System.getenv("DB_PASSWORD") ?: "password"
        maximumPoolSize = 20
        minimumIdle = 5
        connectionTimeout = 30_000
        idleTimeout = 600_000
        maxLifetime = 1_800_000
        connectionTestQuery = "SELECT 1"
    })
}

// application.conf (Hocon format)
/*
ktor {
    deployment {
        port = 8080
        port = ${?PORT}
    }
    application {
        modules = [ com.ecommerce.ApplicationKt.module ]
    }
}

jwt {
    secret = "change-me-in-production"
    secret = ${?JWT_SECRET}
    issuer = "ecommerce-api"
    audience = "ecommerce-users"
    expirationMinutes = 60
}
*/
```

---

## แบบฝึกหัด

```kotlin
// Exercise: เพิ่ม File Upload สำหรับ Product Images

// Requirements:
// 1. POST /api/v1/admin/products/{id}/image
// 2. รับ multipart/form-data
// 3. Validate: jpeg/png, max 5MB
// 4. Upload ไป S3 (หรือ local storage)
// 5. อัพเดท imageUrl ใน Product
// 6. Return URL ของ image

fun Route.imageUploadRoute(productService: ProductService, storageService: StorageService) {
    authenticate("admin") {
        post("/admin/products/{id}/image") {
            val id = call.parameters["id"]!!
            val multipart = call.receiveMultipart()
            
            var fileName: String? = null
            var fileBytes: ByteArray? = null
            var contentType: String? = null
            
            multipart.forEachPart { part ->
                when (part) {
                    is PartData.FileItem -> {
                        fileName = part.originalFileName
                        contentType = part.contentType?.toString()
                        fileBytes = part.streamProvider().readBytes()
                        part.dispose()
                    }
                    else -> part.dispose()
                }
            }
            
            // Validate
            if (fileBytes == null || fileName == null) {
                throw ValidationException(listOf("No file provided"))
            }
            
            val allowedTypes = setOf("image/jpeg", "image/png", "image/webp")
            if (contentType !in allowedTypes) {
                throw ValidationException(listOf("Only JPEG, PNG, and WebP are allowed"))
            }
            
            val maxSize = 5 * 1024 * 1024 // 5MB
            if (fileBytes!!.size > maxSize) {
                throw ValidationException(listOf("File size must be at most 5MB"))
            }
            
            // Upload
            val imageUrl = storageService.uploadProductImage(
                productId = id,
                fileName = fileName!!,
                bytes = fileBytes!!,
                contentType = contentType!!
            )
            
            val product = productService.updateImage(id, imageUrl)
            call.respond(product.toResponse())
        }
    }
}

interface StorageService {
    suspend fun uploadProductImage(productId: String, fileName: String, bytes: ByteArray, contentType: String): String
}

class LocalStorageService(private val uploadDir: java.io.File) : StorageService {
    override suspend fun uploadProductImage(
        productId: String, fileName: String, bytes: ByteArray, contentType: String
    ): String {
        val ext = fileName.substringAfterLast(".", "jpg")
        val file = uploadDir.resolve("products/$productId.$ext")
        file.parentFile.mkdirs()
        file.writeBytes(bytes)
        return "/uploads/products/$productId.$ext"
    }
}
```

---

## สรุป Part 82

```
✅ Ktor: Kotlin-native, coroutine-based, lightweight
✅ embeddedServer(Netty): fast startup, low memory
✅ Application.module(): modular configuration
✅ ContentNegotiation: kotlinx.serialization JSON
✅ CallLogging: structured request logging with callId MDC
✅ StatusPages: centralized exception handling
✅ RequestValidation: declarative input validation
✅ CORS: configurable allowed origins and methods
✅ RateLimit: per-endpoint or global rate limiting
✅ Compression: gzip/deflate response compression
✅ DefaultHeaders: security headers (HSTS, XSS, etc.)
✅ JWT Authentication: HMAC256, bearer token
✅ Multiple auth schemes: jwt + admin role check
✅ Routing DSL: nested routes, authenticate blocks
✅ WebSockets: real-time bidirectional communication
✅ ConnectionManager: concurrent connection tracking
✅ WsMessage sealed class: typed WebSocket messages
✅ Koin DI: lightweight dependency injection for Ktor
✅ testApplication: in-process testing, no network
✅ HttpClient.bearerAuth(): test with JWT tokens
✅ File upload: multipart/form-data, size/type validation
✅ application.conf: Hocon configuration
```

---

*Part 82/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
