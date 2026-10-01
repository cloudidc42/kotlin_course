# Part 19: Ktor - Web Framework

## สารบัญ
1. [Ktor คืออะไร](#ktor-คืออะไร)
2. [Setup และ Project Structure](#setup-และ-project-structure)
3. [Routing](#routing)
4. [Request Handling](#request-handling)
5. [Response](#response)
6. [Authentication](#authentication)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Ktor คืออะไร

Ktor คือ web framework สำหรับ Kotlin ที่สร้างโดย JetBrains รองรับ:
- HTTP Server
- HTTP Client
- WebSocket
- Built with Coroutines
- Modular architecture

---

## Setup และ Project Structure

### build.gradle.kts

```kotlin
plugins {
    kotlin("jvm") version "1.9.22"
    kotlin("plugin.serialization") version "1.9.22"
    id("io.ktor.plugin") version "2.3.7"
}

application {
    mainClass.set("com.example.ApplicationKt")
}

dependencies {
    // Core
    implementation("io.ktor:ktor-server-core-jvm")
    implementation("io.ktor:ktor-server-netty-jvm")
    
    // Features
    implementation("io.ktor:ktor-server-content-negotiation-jvm")
    implementation("io.ktor:ktor-serialization-kotlinx-json-jvm")
    implementation("io.ktor:ktor-server-auth-jvm")
    implementation("io.ktor:ktor-server-auth-jwt-jvm")
    implementation("io.ktor:ktor-server-request-validation")
    implementation("io.ktor:ktor-server-status-pages")
    implementation("io.ktor:ktor-server-cors")
    implementation("io.ktor:ktor-server-call-logging")
    
    // Logging
    implementation("ch.qos.logback:logback-classic:1.4.11")
    
    // Serialization
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.0")
    
    // Testing
    testImplementation("io.ktor:ktor-server-tests-jvm")
    testImplementation(kotlin("test-junit"))
}
```

---

## Basic Server

```kotlin
// src/main/kotlin/Application.kt
package com.example

import io.ktor.server.application.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.response.*
import io.ktor.server.routing.*

fun main() {
    embeddedServer(Netty, port = 8080, host = "0.0.0.0") {
        routing {
            get("/") {
                call.respondText("Hello, Ktor!")
            }
        }
    }.start(wait = true)
}
```

### Configured Application

```kotlin
// Application.kt
package com.example

import io.ktor.server.application.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*

fun main() {
    embeddedServer(Netty, port = 8080) {
        configureRouting()
        configureSerialization()
        configureMonitoring()
        configureSecurity()
    }.start(wait = true)
}

// Application extension functions
fun Application.configureRouting() {
    routing {
        get("/") {
            call.respondText("Welcome to Ktor!")
        }
    }
}

fun Application.configureSerialization() {
    install(ContentNegotiation) {
        json(Json {
            prettyPrint = true
            isLenient = true
        })
    }
}

fun Application.configureMonitoring() {
    install(CallLogging) {
        level = Level.INFO
    }
}

fun Application.configureSecurity() {
    install(CORS) {
        anyHost()
        allowHeader(HttpHeaders.ContentType)
        allowMethod(HttpMethod.Options)
        allowMethod(HttpMethod.Put)
        allowMethod(HttpMethod.Delete)
    }
}
```

---

## Routing

```kotlin
package com.example

import io.ktor.server.application.*
import io.ktor.server.request.*
import io.ktor.server.response.*
import io.ktor.server.routing.*
import io.ktor.http.*
import kotlinx.serialization.Serializable

@Serializable
data class User(
    val id: Int,
    val name: String,
    val email: String
)

// In-memory "database"
val users = mutableListOf(
    User(1, "สมชาย", "somchai@email.com"),
    User(2, "สมหญิง", "somying@email.com"),
    User(3, "สมศักดิ์", "somsak@email.com")
)
var nextId = 4

fun Application.configureRouting() {
    routing {
        // Root
        get("/") {
            call.respondText("Ktor API Server v1.0")
        }
        
        // Route grouping
        route("/api/v1") {
            // Users CRUD
            route("/users") {
                
                // GET /api/v1/users
                get {
                    call.respond(users)
                }
                
                // GET /api/v1/users/{id}
                get("/{id}") {
                    val id = call.parameters["id"]?.toIntOrNull()
                        ?: return@get call.respond(HttpStatusCode.BadRequest, "Invalid ID")
                    
                    val user = users.find { it.id == id }
                        ?: return@get call.respond(HttpStatusCode.NotFound, "User not found")
                    
                    call.respond(user)
                }
                
                // POST /api/v1/users
                post {
                    val userInput = call.receive<NewUserRequest>()
                    
                    // Validation
                    if (userInput.name.isBlank()) {
                        return@post call.respond(HttpStatusCode.BadRequest, "Name is required")
                    }
                    
                    val user = User(nextId++, userInput.name, userInput.email)
                    users.add(user)
                    call.respond(HttpStatusCode.Created, user)
                }
                
                // PUT /api/v1/users/{id}
                put("/{id}") {
                    val id = call.parameters["id"]?.toIntOrNull()
                        ?: return@put call.respond(HttpStatusCode.BadRequest, "Invalid ID")
                    
                    val updateData = call.receive<UpdateUserRequest>()
                    val index = users.indexOfFirst { it.id == id }
                    
                    if (index == -1) {
                        return@put call.respond(HttpStatusCode.NotFound, "User not found")
                    }
                    
                    val updatedUser = users[index].copy(
                        name = updateData.name ?: users[index].name,
                        email = updateData.email ?: users[index].email
                    )
                    users[index] = updatedUser
                    call.respond(updatedUser)
                }
                
                // DELETE /api/v1/users/{id}
                delete("/{id}") {
                    val id = call.parameters["id"]?.toIntOrNull()
                        ?: return@delete call.respond(HttpStatusCode.BadRequest, "Invalid ID")
                    
                    val removed = users.removeAll { it.id == id }
                    if (!removed) {
                        return@delete call.respond(HttpStatusCode.NotFound, "User not found")
                    }
                    
                    call.respond(HttpStatusCode.NoContent)
                }
            }
        }
        
        // Query parameters
        get("/search") {
            val query = call.request.queryParameters["q"]
            val limit = call.request.queryParameters["limit"]?.toIntOrNull() ?: 10
            
            if (query.isNullOrBlank()) {
                return@get call.respond(HttpStatusCode.BadRequest, "Query parameter 'q' is required")
            }
            
            val results = users
                .filter { it.name.contains(query, ignoreCase = true) }
                .take(limit)
            
            call.respond(results)
        }
    }
}

@Serializable
data class NewUserRequest(val name: String, val email: String)

@Serializable
data class UpdateUserRequest(val name: String? = null, val email: String? = null)
```

---

## Request Handling

```kotlin
package com.example

import io.ktor.server.application.*
import io.ktor.server.request.*
import io.ktor.server.response.*
import io.ktor.server.routing.*
import io.ktor.http.*

fun Application.configureRequestHandling() {
    routing {
        // Path parameters
        get("/items/{id}/details") {
            val id = call.parameters["id"]
            val section = call.parameters.getOrFail("section")  // throws if missing
            call.respondText("Item $id, section: $section")
        }
        
        // Optional parameters
        get("/books") {
            val page = call.request.queryParameters["page"]?.toIntOrNull() ?: 1
            val size = call.request.queryParameters["size"]?.toIntOrNull() ?: 20
            val sort = call.request.queryParameters["sort"] ?: "title"
            
            call.respondText("Page $page, size $size, sort: $sort")
        }
        
        // Request body
        post("/data") {
            val contentType = call.request.contentType()
            
            when {
                contentType.match(ContentType.Application.Json) -> {
                    val body = call.receiveText()
                    call.respondText("Got JSON: $body")
                }
                contentType.match(ContentType.Application.FormUrlEncoded) -> {
                    val params = call.receiveParameters()
                    val name = params["name"]
                    call.respondText("Form name: $name")
                }
                else -> {
                    call.respond(HttpStatusCode.UnsupportedMediaType)
                }
            }
        }
        
        // Multipart (file upload)
        post("/upload") {
            val multipart = call.receiveMultipart()
            val fileData = mutableListOf<String>()
            
            multipart.forEachPart { part ->
                when (part) {
                    is PartData.FormItem -> {
                        fileData.add("${part.name}: ${part.value}")
                    }
                    is PartData.FileItem -> {
                        val fileName = part.originalFileName ?: "unknown"
                        val bytes = part.streamProvider().readBytes()
                        fileData.add("File: $fileName (${bytes.size} bytes)")
                    }
                    else -> Unit
                }
                part.dispose()
            }
            
            call.respond(fileData)
        }
        
        // Headers
        get("/headers") {
            val userAgent = call.request.headers["User-Agent"]
            val auth = call.request.headers["Authorization"]
            val accept = call.request.accept()
            
            call.respondText("""
                User-Agent: $userAgent
                Authorization: ${auth?.take(20)}...
                Accept: $accept
            """.trimIndent())
        }
    }
}
```

---

## Response

```kotlin
package com.example

import io.ktor.server.application.*
import io.ktor.server.response.*
import io.ktor.server.routing.*
import io.ktor.http.*
import io.ktor.server.plugins.statuspages.*
import kotlinx.serialization.Serializable
import java.io.File

@Serializable
data class ApiResponse<T>(
    val success: Boolean,
    val data: T? = null,
    val error: String? = null,
    val meta: Meta? = null
)

@Serializable
data class Meta(
    val total: Int,
    val page: Int,
    val perPage: Int
)

fun Application.configureResponses() {
    // Error handling
    install(StatusPages) {
        exception<IllegalArgumentException> { call, cause ->
            call.respond(
                HttpStatusCode.BadRequest,
                ApiResponse<Nothing>(false, error = cause.message)
            )
        }
        
        exception<NoSuchElementException> { call, cause ->
            call.respond(
                HttpStatusCode.NotFound,
                ApiResponse<Nothing>(false, error = "Resource not found")
            )
        }
        
        exception<Throwable> { call, cause ->
            call.application.environment.log.error("Unhandled exception", cause)
            call.respond(
                HttpStatusCode.InternalServerError,
                ApiResponse<Nothing>(false, error = "Internal server error")
            )
        }
        
        status(HttpStatusCode.NotFound) { call, status ->
            call.respond(
                status,
                ApiResponse<Nothing>(false, error = "Route not found: ${call.request.uri}")
            )
        }
    }
    
    routing {
        // JSON response
        get("/api/products") {
            val products = listOf(
                mapOf("id" to 1, "name" to "Laptop", "price" to 25000),
                mapOf("id" to 2, "name" to "Phone", "price" to 15000)
            )
            
            call.respond(
                ApiResponse(
                    success = true,
                    data = products,
                    meta = Meta(total = 2, page = 1, perPage = 20)
                )
            )
        }
        
        // Custom status codes
        post("/api/items") {
            call.respond(HttpStatusCode.Created, ApiResponse(true, data = "Item created"))
        }
        
        delete("/api/items/{id}") {
            call.respond(HttpStatusCode.NoContent)
        }
        
        // Redirect
        get("/old-path") {
            call.respondRedirect("/new-path", permanent = true)
        }
        
        // File response
        get("/download/{filename}") {
            val filename = call.parameters["filename"] ?: "file"
            val file = File("uploads/$filename")
            
            if (!file.exists()) {
                call.respond(HttpStatusCode.NotFound, "File not found")
                return@get
            }
            
            call.response.header(
                HttpHeaders.ContentDisposition,
                ContentDisposition.Attachment.withParameter(
                    ContentDisposition.Parameters.FileName, filename
                ).toString()
            )
            call.respondFile(file)
        }
        
        // Stream response
        get("/stream") {
            call.respondTextWriter(ContentType.Text.EventStream) {
                repeat(5) { i ->
                    write("data: Event $i\n\n")
                    flush()
                    kotlinx.coroutines.delay(1000)
                }
            }
        }
        
        // HTML response
        get("/html") {
            call.respondText(
                """
                <!DOCTYPE html>
                <html>
                <body>
                    <h1>Hello from Ktor!</h1>
                </body>
                </html>
                """.trimIndent(),
                ContentType.Text.Html
            )
        }
    }
}
```

---

## Authentication

```kotlin
package com.example

import io.ktor.server.application.*
import io.ktor.server.auth.*
import io.ktor.server.auth.jwt.*
import io.ktor.server.response.*
import io.ktor.server.routing.*
import io.ktor.http.*
import com.auth0.jwt.JWT
import com.auth0.jwt.algorithms.Algorithm
import java.util.*

const val JWT_SECRET = "your-secret-key-change-in-production"
const val JWT_ISSUER = "http://localhost:8080"
const val JWT_AUDIENCE = "http://localhost:8080/api"
const val JWT_REALM = "Access to API"

data class UserCredentials(val username: String, val password: String)

fun generateToken(userId: Int, username: String): String {
    return JWT.create()
        .withAudience(JWT_AUDIENCE)
        .withIssuer(JWT_ISSUER)
        .withClaim("userId", userId)
        .withClaim("username", username)
        .withExpiresAt(Date(System.currentTimeMillis() + 3_600_000))  // 1 hour
        .sign(Algorithm.HMAC256(JWT_SECRET))
}

fun Application.configureSecurity() {
    // Basic Auth
    install(Authentication) {
        basic("basic-auth") {
            realm = "API"
            validate { credentials ->
                if (credentials.name == "admin" && credentials.password == "password") {
                    UserIdPrincipal(credentials.name)
                } else null
            }
        }
        
        // JWT Auth
        jwt("jwt-auth") {
            realm = JWT_REALM
            verifier(
                JWT.require(Algorithm.HMAC256(JWT_SECRET))
                    .withAudience(JWT_AUDIENCE)
                    .withIssuer(JWT_ISSUER)
                    .build()
            )
            validate { credential ->
                if (credential.payload.getClaim("userId").asInt() != null) {
                    JWTPrincipal(credential.payload)
                } else null
            }
            challenge { _, _ ->
                call.respond(HttpStatusCode.Unauthorized, "Token is not valid or expired")
            }
        }
    }
    
    routing {
        // Login endpoint
        post("/auth/login") {
            val credentials = call.receive<UserCredentials>()
            
            // Validate (in real app, check database)
            if (credentials.username == "admin" && credentials.password == "password") {
                val token = generateToken(1, credentials.username)
                call.respond(mapOf("token" to token))
            } else {
                call.respond(HttpStatusCode.Unauthorized, "Invalid credentials")
            }
        }
        
        // Protected with Basic Auth
        authenticate("basic-auth") {
            get("/admin") {
                val principal = call.principal<UserIdPrincipal>()
                call.respondText("Hello, ${principal?.name}!")
            }
        }
        
        // Protected with JWT
        authenticate("jwt-auth") {
            route("/api/protected") {
                get("/profile") {
                    val principal = call.principal<JWTPrincipal>()
                    val userId = principal?.payload?.getClaim("userId")?.asInt()
                    val username = principal?.payload?.getClaim("username")?.asString()
                    call.respond(mapOf(
                        "userId" to userId,
                        "username" to username
                    ))
                }
                
                get("/data") {
                    call.respondText("This is protected data!")
                }
            }
        }
    }
}
```

---

## ตัวอย่างโปรแกรม - REST API สมบูรณ์

```kotlin
// Complete Todo API
package com.example.todo

import io.ktor.server.application.*
import io.ktor.server.engine.*
import io.ktor.server.netty.*
import io.ktor.server.response.*
import io.ktor.server.request.*
import io.ktor.server.routing.*
import io.ktor.server.plugins.contentnegotiation.*
import io.ktor.serialization.kotlinx.json.*
import io.ktor.http.*
import io.ktor.server.plugins.statuspages.*
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json

@Serializable
data class Todo(
    val id: Int,
    val title: String,
    val description: String = "",
    val completed: Boolean = false,
    val priority: Int = 1  // 1=low, 2=medium, 3=high
)

@Serializable
data class CreateTodoRequest(
    val title: String,
    val description: String = "",
    val priority: Int = 1
)

@Serializable
data class UpdateTodoRequest(
    val title: String? = null,
    val description: String? = null,
    val completed: Boolean? = null,
    val priority: Int? = null
)

// Todo Service
class TodoService {
    private val todos = mutableListOf(
        Todo(1, "ศึกษา Kotlin", "เรียน coroutines และ ktor", false, 3),
        Todo(2, "ออกกำลังกาย", "วิ่ง 30 นาที", false, 2),
        Todo(3, "อ่านหนังสือ", "Clean Code", true, 1)
    )
    private var nextId = 4
    
    fun getAll(completed: Boolean? = null): List<Todo> =
        if (completed == null) todos.toList()
        else todos.filter { it.completed == completed }
    
    fun getById(id: Int): Todo? = todos.find { it.id == id }
    
    fun create(req: CreateTodoRequest): Todo {
        require(req.title.isNotBlank()) { "Title is required" }
        require(req.priority in 1..3) { "Priority must be 1-3" }
        val todo = Todo(nextId++, req.title, req.description, priority = req.priority)
        todos.add(todo)
        return todo
    }
    
    fun update(id: Int, req: UpdateTodoRequest): Todo? {
        val index = todos.indexOfFirst { it.id == id }
        if (index == -1) return null
        
        val updated = todos[index].copy(
            title = req.title ?: todos[index].title,
            description = req.description ?: todos[index].description,
            completed = req.completed ?: todos[index].completed,
            priority = req.priority ?: todos[index].priority
        )
        todos[index] = updated
        return updated
    }
    
    fun delete(id: Int): Boolean = todos.removeAll { it.id == id }
    
    fun stats() = mapOf(
        "total" to todos.size,
        "completed" to todos.count { it.completed },
        "pending" to todos.count { !it.completed },
        "high_priority" to todos.count { it.priority == 3 }
    )
}

fun main() {
    val todoService = TodoService()
    
    embeddedServer(Netty, port = 8080) {
        install(ContentNegotiation) {
            json(Json { prettyPrint = true })
        }
        
        install(StatusPages) {
            exception<IllegalArgumentException> { call, cause ->
                call.respond(HttpStatusCode.BadRequest, mapOf("error" to cause.message))
            }
        }
        
        routing {
            route("/api/todos") {
                get {
                    val completed = call.request.queryParameters["completed"]?.toBoolean()
                    call.respond(todoService.getAll(completed))
                }
                
                get("/{id}") {
                    val id = call.parameters["id"]?.toIntOrNull()
                        ?: return@get call.respond(HttpStatusCode.BadRequest, "Invalid ID")
                    val todo = todoService.getById(id)
                        ?: return@get call.respond(HttpStatusCode.NotFound, "Todo not found")
                    call.respond(todo)
                }
                
                post {
                    val req = call.receive<CreateTodoRequest>()
                    val todo = todoService.create(req)
                    call.respond(HttpStatusCode.Created, todo)
                }
                
                patch("/{id}") {
                    val id = call.parameters["id"]?.toIntOrNull()
                        ?: return@patch call.respond(HttpStatusCode.BadRequest, "Invalid ID")
                    val req = call.receive<UpdateTodoRequest>()
                    val todo = todoService.update(id, req)
                        ?: return@patch call.respond(HttpStatusCode.NotFound, "Todo not found")
                    call.respond(todo)
                }
                
                delete("/{id}") {
                    val id = call.parameters["id"]?.toIntOrNull()
                        ?: return@delete call.respond(HttpStatusCode.BadRequest, "Invalid ID")
                    if (!todoService.delete(id)) {
                        return@delete call.respond(HttpStatusCode.NotFound, "Todo not found")
                    }
                    call.respond(HttpStatusCode.NoContent)
                }
            }
            
            get("/api/todos/stats") {
                call.respond(todoService.stats())
            }
        }
    }.start(wait = true)
}
```

---

## Testing Ktor

```kotlin
import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.http.*
import io.ktor.server.testing.*
import kotlin.test.*

class TodoApiTest {
    @Test
    fun testGetAllTodos() = testApplication {
        application {
            // configure your application
        }
        
        val response = client.get("/api/todos")
        assertEquals(HttpStatusCode.OK, response.status)
        
        val body = response.bodyAsText()
        assertContains(body, "Kotlin")
    }
    
    @Test
    fun testCreateTodo() = testApplication {
        val response = client.post("/api/todos") {
            contentType(ContentType.Application.Json)
            setBody("""{"title":"New Task","priority":2}""")
        }
        assertEquals(HttpStatusCode.Created, response.status)
    }
    
    @Test
    fun testGetNotFound() = testApplication {
        val response = client.get("/api/todos/9999")
        assertEquals(HttpStatusCode.NotFound, response.status)
    }
}
```

---

## สรุป Part 19

```
✅ Ktor: Kotlin-native web framework
✅ embeddedServer(Netty, port) { ... }.start()
✅ routing { get/post/put/delete/patch { } }
✅ route("/prefix") { } สำหรับ grouping
✅ call.parameters["id"]: path parameters
✅ call.request.queryParameters["q"]: query params
✅ call.receive<T>(): deserialize request body
✅ call.respond(data): serialize response
✅ HttpStatusCode.Created, NotFound, etc.
✅ install(ContentNegotiation) { json() }
✅ install(StatusPages) { exception<T> { } }
✅ install(Authentication) { jwt/basic }
✅ authenticate("name") { } สำหรับ protected routes
✅ testApplication { } สำหรับ testing
```

---

*Part 19/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
