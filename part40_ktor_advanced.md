# Part 40: Ktor Advanced Features

## สารบัญ
1. [Ktor Architecture](#ktor-architecture)
2. [Routing ขั้นสูง](#routing-ขั้นสูง)
3. [Authentication Plugins](#authentication-plugins)
4. [WebSockets](#websockets)
5. [Server-Sent Events](#server-sent-events)
6. [Ktor Client](#ktor-client)
7. [Testing Ktor](#testing-ktor)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Ktor Architecture

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.ktor:ktor-server-core:2.3.8")
    implementation("io.ktor:ktor-server-netty:2.3.8")
    implementation("io.ktor:ktor-server-content-negotiation:2.3.8")
    implementation("io.ktor:ktor-serialization-kotlinx-json:2.3.8")
    implementation("io.ktor:ktor-server-auth:2.3.8")
    implementation("io.ktor:ktor-server-auth-jwt:2.3.8")
    implementation("io.ktor:ktor-server-websockets:2.3.8")
    implementation("io.ktor:ktor-server-rate-limit:2.3.8")
    implementation("io.ktor:ktor-server-call-logging:2.3.8")
    implementation("io.ktor:ktor-server-status-pages:2.3.8")
    implementation("io.ktor:ktor-server-cors:2.3.8")
    implementation("io.ktor:ktor-server-compression:2.3.8")
    implementation("io.ktor:ktor-client-core:2.3.8")
    implementation("io.ktor:ktor-client-cio:2.3.8")
    implementation("io.ktor:ktor-client-content-negotiation:2.3.8")
    testImplementation("io.ktor:ktor-server-test-host:2.3.8")
}
```

```kotlin
// Main application
import io.ktor.server.application.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.routing.*

fun main() {
    embeddedServer(
        Netty,
        port = System.getenv("PORT")?.toInt() ?: 8080,
        host = "0.0.0.0"
    ) {
        configurePlugins()
        configureSecurity()
        configureRoutes()
    }.start(wait = true)
}

fun Application.configurePlugins() {
    install(ContentNegotiation) {
        json(Json {
            prettyPrint = false
            isLenient = true
            ignoreUnknownKeys = true
            serializersModule = SerializersModule {
                // custom serializers
            }
        })
    }
    
    install(CallLogging) {
        level = Level.INFO
        filter { call -> !call.request.path().startsWith("/health") }
        format { call ->
            buildString {
                append(call.request.httpMethod.value)
                append(' ')
                append(call.request.path())
                append(' ')
                append(call.response.status()?.value)
                append(' ')
                append(call.processingTimeMillis())
                append("ms")
            }
        }
    }
    
    install(CORS) {
        allowMethod(HttpMethod.Options)
        allowMethod(HttpMethod.Get)
        allowMethod(HttpMethod.Post)
        allowMethod(HttpMethod.Put)
        allowMethod(HttpMethod.Delete)
        allowHeader(HttpHeaders.Authorization)
        allowHeader(HttpHeaders.ContentType)
        allowHeader("X-Request-ID")
        allowCredentials = true
        anyHost()  // In production, specify exact hosts
    }
    
    install(Compression) {
        gzip {
            priority = 1.0
            minimumSize(1024)
        }
        deflate {
            priority = 0.9
            minimumSize(1024)
        }
    }
    
    install(StatusPages) {
        exception<IllegalArgumentException> { call, cause ->
            call.respond(HttpStatusCode.BadRequest, ErrorResponse(
                code = "INVALID_REQUEST",
                message = cause.message ?: "Invalid request"
            ))
        }
        
        exception<AuthenticationException> { call, _ ->
            call.respond(HttpStatusCode.Unauthorized, ErrorResponse(
                code = "UNAUTHORIZED",
                message = "Authentication required"
            ))
        }
        
        exception<ForbiddenException> { call, _ ->
            call.respond(HttpStatusCode.Forbidden, ErrorResponse(
                code = "FORBIDDEN",
                message = "Access denied"
            ))
        }
        
        exception<NotFoundException> { call, cause ->
            call.respond(HttpStatusCode.NotFound, ErrorResponse(
                code = "NOT_FOUND",
                message = cause.message ?: "Resource not found"
            ))
        }
        
        exception<Throwable> { call, cause ->
            call.application.log.error("Unhandled exception", cause)
            call.respond(HttpStatusCode.InternalServerError, ErrorResponse(
                code = "INTERNAL_ERROR",
                message = "An unexpected error occurred"
            ))
        }
        
        status(HttpStatusCode.NotFound) { call, status ->
            call.respond(status, ErrorResponse("NOT_FOUND", "The requested resource was not found"))
        }
    }
    
    install(WebSockets) {
        pingPeriod = Duration.ofSeconds(15)
        timeout = Duration.ofSeconds(30)
        maxFrameSize = Long.MAX_VALUE
        masking = false
    }
    
    install(RateLimit) {
        global {
            rateLimiter(limit = 100, refillPeriod = 1.minutes)
        }
        
        register(RateLimitName("auth")) {
            rateLimiter(limit = 5, refillPeriod = 15.minutes)
            requestKey { call ->
                call.request.origin.remoteAddress
            }
        }
    }
}

@Serializable
data class ErrorResponse(val code: String, val message: String)

class AuthenticationException : Exception()
class ForbiddenException : Exception()
class NotFoundException(message: String) : Exception(message)
```

---

## Routing ขั้นสูง

```kotlin
fun Application.configureRoutes() {
    routing {
        // Health check (no auth)
        get("/health") {
            call.respond(mapOf("status" to "UP"))
        }
        
        // API v1
        route("/api/v1") {
            
            // Users
            route("/users") {
                get { 
                    val page = call.request.queryParameters["page"]?.toIntOrNull() ?: 1
                    val limit = call.request.queryParameters["limit"]?.toIntOrNull() ?: 20
                    val users = userService.findAll(page, limit)
                    call.respond(users)
                }
                
                post {
                    val request = call.receive<CreateUserRequest>()
                    val user = userService.create(request)
                    call.respond(HttpStatusCode.Created, user)
                }
                
                route("/{id}") {
                    get {
                        val id = call.parameters["id"] ?: throw IllegalArgumentException("ID required")
                        val user = userService.findById(id) ?: throw NotFoundException("User $id not found")
                        call.respond(user)
                    }
                    
                    put {
                        val id = call.parameters["id"] ?: throw IllegalArgumentException("ID required")
                        val request = call.receive<UpdateUserRequest>()
                        val user = userService.update(id, request)
                        call.respond(user)
                    }
                    
                    delete {
                        val id = call.parameters["id"] ?: throw IllegalArgumentException("ID required")
                        userService.delete(id)
                        call.respond(HttpStatusCode.NoContent)
                    }
                }
            }
        }
        
        // File upload
        post("/upload") {
            val multipart = call.receiveMultipart()
            var filename = ""
            var fileBytes = ByteArray(0)
            
            multipart.forEachPart { part ->
                when (part) {
                    is PartData.FileItem -> {
                        filename = part.originalFileName ?: "upload"
                        fileBytes = part.streamProvider().readBytes()
                    }
                    is PartData.FormItem -> {
                        // handle form fields
                    }
                    else -> {}
                }
                part.dispose()
            }
            
            val url = storageService.upload(filename, fileBytes)
            call.respond(mapOf("url" to url))
        }
    }
}
```

---

## JWT Authentication

```kotlin
fun Application.configureSecurity() {
    val jwtConfig = JwtConfig(
        secret = System.getenv("JWT_SECRET") ?: "dev-secret-key",
        issuer = "kotlin-app",
        audience = "kotlin-app-users"
    )
    
    install(Authentication) {
        jwt("user-jwt") {
            realm = "kotlin-app"
            verifier(
                JWT.require(Algorithm.HMAC256(jwtConfig.secret))
                    .withAudience(jwtConfig.audience)
                    .withIssuer(jwtConfig.issuer)
                    .build()
            )
            validate { credential ->
                val userId = credential.payload.getClaim("userId")?.asString()
                val roles = credential.payload.getClaim("roles")?.asList(String::class.java) ?: emptyList()
                
                if (userId != null) {
                    UserPrincipal(userId, roles)
                } else null
            }
            challenge { _, _ ->
                call.respond(HttpStatusCode.Unauthorized, ErrorResponse(
                    "UNAUTHORIZED", "Invalid or expired token"
                ))
            }
        }
        
        jwt("admin-jwt") {
            realm = "kotlin-app-admin"
            verifier(
                JWT.require(Algorithm.HMAC256(jwtConfig.secret))
                    .withAudience(jwtConfig.audience)
                    .withIssuer(jwtConfig.issuer)
                    .build()
            )
            validate { credential ->
                val userId = credential.payload.getClaim("userId")?.asString()
                val roles = credential.payload.getClaim("roles")?.asList(String::class.java) ?: emptyList()
                
                if (userId != null && "ADMIN" in roles) {
                    UserPrincipal(userId, roles)
                } else null
            }
        }
    }
    
    routing {
        // Public routes
        withRateLimit(RateLimitName("auth")) {
            post("/auth/login") {
                val request = call.receive<LoginRequest>()
                val tokens = authService.login(request)
                call.respond(tokens)
            }
            
            post("/auth/register") {
                val request = call.receive<RegisterRequest>()
                val result = authService.register(request)
                call.respond(HttpStatusCode.Created, result)
            }
        }
        
        // Protected routes
        authenticate("user-jwt") {
            route("/api/v1") {
                get("/profile") {
                    val principal = call.principal<UserPrincipal>()!!
                    val user = userService.findById(principal.userId)!!
                    call.respond(user)
                }
                
                put("/profile") {
                    val principal = call.principal<UserPrincipal>()!!
                    val request = call.receive<UpdateProfileRequest>()
                    val user = userService.updateProfile(principal.userId, request)
                    call.respond(user)
                }
            }
        }
        
        // Admin routes
        authenticate("admin-jwt") {
            route("/admin") {
                get("/users") {
                    val users = userService.findAll()
                    call.respond(users)
                }
            }
        }
    }
}

data class UserPrincipal(val userId: String, val roles: List<String>) : Principal
```

---

## WebSockets

```kotlin
import io.ktor.server.websocket.*
import io.ktor.websocket.*
import kotlinx.coroutines.channels.consumeEach
import java.util.concurrent.ConcurrentHashMap
import kotlinx.coroutines.flow.MutableSharedFlow

// Chat room manager
class ChatRoomManager {
    private val rooms = ConcurrentHashMap<String, MutableSet<WebSocketSession>>()
    
    fun join(roomId: String, session: WebSocketSession) {
        rooms.getOrPut(roomId) { 
            java.util.Collections.synchronizedSet(mutableSetOf()) 
        }.add(session)
    }
    
    fun leave(roomId: String, session: WebSocketSession) {
        rooms[roomId]?.remove(session)
        if (rooms[roomId]?.isEmpty() == true) rooms.remove(roomId)
    }
    
    suspend fun broadcast(roomId: String, message: String, exclude: WebSocketSession? = null) {
        rooms[roomId]?.forEach { session ->
            if (session != exclude) {
                try {
                    session.send(message)
                } catch (e: Exception) {
                    rooms[roomId]?.remove(session)
                }
            }
        }
    }
    
    fun getMemberCount(roomId: String) = rooms[roomId]?.size ?: 0
}

fun Application.configureWebSockets() {
    val chatManager = ChatRoomManager()
    
    routing {
        authenticate("user-jwt") {
            webSocket("/ws/chat/{roomId}") {
                val principal = call.principal<UserPrincipal>()!!
                val roomId = call.parameters["roomId"] ?: run {
                    close(CloseReason(CloseReason.Codes.VIOLATED_POLICY, "Room ID required"))
                    return@webSocket
                }
                
                chatManager.join(roomId, this)
                
                // Announce join
                chatManager.broadcast(
                    roomId = roomId,
                    message = """{"type":"join","userId":"${principal.userId}","members":${chatManager.getMemberCount(roomId)}}""",
                    exclude = this
                )
                
                try {
                    incoming.consumeEach { frame ->
                        when (frame) {
                            is Frame.Text -> {
                                val text = frame.readText()
                                val message = """{"type":"message","userId":"${principal.userId}","content":${kotlinx.serialization.json.Json.encodeToString(text)},"timestamp":${System.currentTimeMillis()}}"""
                                chatManager.broadcast(roomId, message)
                            }
                            is Frame.Binary -> {
                                // Handle binary data
                            }
                            is Frame.Close -> {
                                // Handle close
                            }
                            else -> {}
                        }
                    }
                } finally {
                    chatManager.leave(roomId, this)
                    chatManager.broadcast(
                        roomId = roomId,
                        message = """{"type":"leave","userId":"${principal.userId}"}"""
                    )
                }
            }
        }
    }
}
```

---

## Ktor Client

```kotlin
import io.ktor.client.*
import io.ktor.client.call.*
import io.ktor.client.engine.cio.*
import io.ktor.client.plugins.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.client.request.*
import io.ktor.client.statement.*

// Create HTTP client
val client = HttpClient(CIO) {
    install(ContentNegotiation) {
        json(Json { ignoreUnknownKeys = true })
    }
    
    install(HttpTimeout) {
        connectTimeoutMillis = 5000
        requestTimeoutMillis = 30000
        socketTimeoutMillis = 30000
    }
    
    install(HttpRequestRetry) {
        retryOnServerErrors(maxRetries = 3)
        exponentialDelay()
    }
    
    install(Logging) {
        logger = Logger.DEFAULT
        level = LogLevel.HEADERS
    }
    
    defaultRequest {
        header(HttpHeaders.ContentType, ContentType.Application.Json)
        header("X-Api-Key", System.getenv("API_KEY") ?: "")
    }
}

// Service using Ktor client
class WeatherService(private val client: HttpClient) {
    
    suspend fun getCurrentWeather(city: String): WeatherData {
        return client.get("https://api.weather.example.com/current") {
            parameter("city", city)
            parameter("units", "metric")
        }.body()
    }
    
    suspend fun getForecast(lat: Double, lon: Double, days: Int = 7): List<ForecastDay> {
        return client.get("https://api.weather.example.com/forecast") {
            parameter("lat", lat)
            parameter("lon", lon)
            parameter("days", days)
        }.body()
    }
    
    suspend fun postAlertSubscription(subscription: AlertSubscription): AlertResponse {
        return client.post("https://api.weather.example.com/alerts") {
            setBody(subscription)
        }.body()
    }
}

// Mock response types
@Serializable data class WeatherData(val city: String, val temp: Double, val condition: String)
@Serializable data class ForecastDay(val date: String, val high: Double, val low: Double)
@Serializable data class AlertSubscription(val email: String, val city: String)
@Serializable data class AlertResponse(val id: String, val status: String)
```

---

## Testing Ktor

```kotlin
import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.http.*
import io.ktor.server.testing.*
import kotlin.test.*

class UserRoutesTest {
    
    @Test
    fun `GET users returns list`() = testApplication {
        application {
            configurePlugins()
            configureSecurity()
            configureRoutes()
        }
        
        val token = generateTestToken("user-1", listOf("USER"))
        
        val response = client.get("/api/v1/users") {
            bearerAuth(token)
        }
        
        assertEquals(HttpStatusCode.OK, response.status)
        val users = Json.decodeFromString<List<UserResponse>>(response.bodyAsText())
        assertTrue(users.isNotEmpty())
    }
    
    @Test
    fun `POST users creates user and returns 201`() = testApplication {
        application {
            configurePlugins()
            configureSecurity()
            configureRoutes()
        }
        
        val token = generateTestToken("admin-1", listOf("USER", "ADMIN"))
        
        val response = client.post("/api/v1/users") {
            bearerAuth(token)
            contentType(ContentType.Application.Json)
            setBody("""{"username":"newuser","email":"new@test.com","password":"Test@123"}""")
        }
        
        assertEquals(HttpStatusCode.Created, response.status)
    }
    
    @Test
    fun `GET users without token returns 401`() = testApplication {
        application {
            configurePlugins()
            configureSecurity()
            configureRoutes()
        }
        
        val response = client.get("/api/v1/users")
        assertEquals(HttpStatusCode.Unauthorized, response.status)
    }
    
    @Test
    fun `WebSocket chat sends and receives messages`() = testApplication {
        application {
            configureWebSockets()
        }
        
        val token = generateTestToken("user-1", listOf("USER"))
        
        webSocketSession("/ws/chat/room-1") {
            send(Frame.Text("Hello World"))
            val received = incoming.receive() as Frame.Text
            assertTrue(received.readText().contains("Hello World"))
        }
    }
    
    private fun generateTestToken(userId: String, roles: List<String>): String {
        return JWT.create()
            .withAudience("kotlin-app-users")
            .withIssuer("kotlin-app")
            .withClaim("userId", userId)
            .withClaim("roles", roles)
            .withExpiresAt(java.util.Date(System.currentTimeMillis() + 3600000))
            .sign(Algorithm.HMAC256("dev-secret-key"))
    }
}

// Placeholder services
val userService: Any get() = TODO()
val authService: Any get() = TODO()
val storageService: Any get() = TODO()
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Build a real-time notification system with Ktor WebSocket

// Requirements:
// 1. Users can connect via WebSocket at /ws/notifications
// 2. JWT authentication required
// 3. When order status changes, notify the user
// 4. Support multiple devices per user (multiple WS connections)
// 5. Queue messages if user is offline, deliver when they connect

class NotificationService {
    private val userSessions = ConcurrentHashMap<String, MutableSet<WebSocketSession>>()
    private val messageQueue = ConcurrentHashMap<String, ArrayDeque<Notification>>()
    
    suspend fun connect(userId: String, session: WebSocketSession) {
        userSessions.getOrPut(userId) {
            java.util.Collections.synchronizedSet(mutableSetOf())
        }.add(session)
        
        // Deliver queued messages
        messageQueue[userId]?.let { queue ->
            while (queue.isNotEmpty()) {
                val notification = queue.removeFirst()
                try {
                    session.send(Frame.Text(kotlinx.serialization.json.Json.encodeToString(notification)))
                } catch (e: Exception) {
                    queue.addFirst(notification)  // put back if failed
                    break
                }
            }
        }
    }
    
    fun disconnect(userId: String, session: WebSocketSession) {
        userSessions[userId]?.remove(session)
        if (userSessions[userId]?.isEmpty() == true) {
            userSessions.remove(userId)
        }
    }
    
    suspend fun send(userId: String, notification: Notification) {
        val sessions = userSessions[userId]
        if (sessions.isNullOrEmpty()) {
            // Queue for later delivery
            messageQueue.getOrPut(userId) { ArrayDeque() }.addLast(notification)
        } else {
            val json = kotlinx.serialization.json.Json.encodeToString(notification)
            val deadSessions = mutableListOf<WebSocketSession>()
            sessions.forEach { session ->
                try {
                    session.send(Frame.Text(json))
                } catch (e: Exception) {
                    deadSessions.add(session)
                }
            }
            deadSessions.forEach { disconnect(userId, it) }
        }
    }
}

@Serializable
data class Notification(
    val id: String,
    val type: String,
    val title: String,
    val body: String,
    val data: Map<String, String> = emptyMap(),
    val timestamp: Long = System.currentTimeMillis()
)
```

---

## สรุป Part 40

```
✅ Ktor: lightweight, coroutine-first web framework
✅ Plugins: ContentNegotiation, CORS, Compression, RateLimit
✅ StatusPages: centralized error handling
✅ JWT Auth: stateless authentication
✅ Route groups: organize routes by authentication
✅ WebSockets: real-time bidirectional communication
✅ Ktor Client: HTTP client with retry, timeout, logging
✅ testApplication: test without starting a real server
✅ Multiple auth providers: user vs admin
✅ WebSocket manager: broadcast to room members
✅ Offline queuing: deliver messages when user reconnects
```

---

*Part 40/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
