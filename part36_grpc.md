# Part 36: gRPC ด้วย Kotlin

## สารบัญ
1. [gRPC คืออะไร](#grpc-คืออะไร)
2. [Protocol Buffers](#protocol-buffers)
3. [Unary RPC](#unary-rpc)
4. [Server Streaming](#server-streaming)
5. [Client Streaming](#client-streaming)
6. [Bidirectional Streaming](#bidirectional-streaming)
7. [Interceptors](#interceptors)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## gRPC คืออะไร

gRPC เป็น framework สำหรับ Remote Procedure Call ที่ใช้ HTTP/2 และ Protocol Buffers เหมาะสำหรับ microservices ที่ต้องการ performance สูง

```
REST API:
- Text-based (JSON) -> larger payload
- HTTP/1.1 -> 1 request per connection
- No strict contract

gRPC:
- Binary (Protobuf) -> 3-10x smaller payload
- HTTP/2 -> multiplexing, streaming
- Strict contract (proto files)
- Auto-generated code for all languages
```

---

## Setup

```kotlin
// build.gradle.kts
plugins {
    kotlin("jvm") version "1.9.22"
    id("com.google.protobuf") version "0.9.4"
}

dependencies {
    implementation("io.grpc:grpc-netty-shaded:1.62.2")
    implementation("io.grpc:grpc-protobuf:1.62.2")
    implementation("io.grpc:grpc-kotlin-stub:1.4.1")
    implementation("com.google.protobuf:protobuf-kotlin:3.25.2")
    testImplementation("io.grpc:grpc-testing:1.62.2")
}

protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:3.25.2"
    }
    plugins {
        id("grpc") {
            artifact = "io.grpc:protoc-gen-grpc-java:1.62.2"
        }
        id("grpckt") {
            artifact = "io.grpc:protoc-gen-grpc-kotlin:1.4.1:jdk8@jar"
        }
    }
    generateProtoTasks {
        all().forEach {
            it.plugins {
                id("grpc")
                id("grpckt")
            }
        }
    }
}
```

---

## Protocol Buffers

```protobuf
// src/main/proto/user.proto
syntax = "proto3";

package com.example.grpc;

option java_package = "com.example.grpc.proto";
option java_multiple_files = true;

// Message types
message User {
  string id = 1;
  string username = 2;
  string email = 3;
  string first_name = 4;
  string last_name = 5;
  UserRole role = 6;
  int64 created_at = 7;
  
  enum UserRole {
    USER = 0;
    MODERATOR = 1;
    ADMIN = 2;
  }
}

message CreateUserRequest {
  string username = 1;
  string email = 2;
  string password = 3;
  string first_name = 4;
  string last_name = 5;
}

message GetUserRequest {
  string user_id = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
  string role_filter = 3;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total_count = 2;
  bool has_more = 3;
}

message UpdateUserRequest {
  string user_id = 1;
  optional string first_name = 2;
  optional string last_name = 3;
  optional string email = 4;
}

message DeleteUserRequest {
  string user_id = 1;
}

message DeleteUserResponse {
  bool success = 1;
}

// Service definition
service UserService {
  // Unary
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc GetUser(GetUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
  
  // Server streaming
  rpc ListUsers(ListUsersRequest) returns (stream User);
  
  // Client streaming
  rpc BatchCreateUsers(stream CreateUserRequest) returns (BatchCreateResponse);
  
  // Bidirectional streaming
  rpc SyncUsers(stream SyncRequest) returns (stream SyncResponse);
}

message BatchCreateResponse {
  int32 created_count = 0;
  int32 failed_count = 0;
  repeated string errors = 3;
}

message SyncRequest {
  string user_id = 1;
  SyncAction action = 2;
  User data = 3;
  
  enum SyncAction {
    UPSERT = 0;
    DELETE = 1;
    QUERY = 2;
  }
}

message SyncResponse {
  string user_id = 1;
  bool success = 2;
  string message = 3;
  User user = 4;
}
```

---

## Unary RPC (Server Implementation)

```kotlin
package com.example.grpc.service

import com.example.grpc.proto.*
import io.grpc.Status
import io.grpc.StatusException
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow

class UserGrpcService(
    private val userRepository: UserRepository
) : UserServiceGrpcKt.UserServiceCoroutineImplBase() {
    
    override suspend fun createUser(request: CreateUserRequest): User {
        // Validate input
        if (request.username.isBlank()) {
            throw StatusException(Status.INVALID_ARGUMENT.withDescription("Username is required"))
        }
        if (!request.email.contains("@")) {
            throw StatusException(Status.INVALID_ARGUMENT.withDescription("Invalid email"))
        }
        
        // Check existing
        if (userRepository.existsByEmail(request.email)) {
            throw StatusException(Status.ALREADY_EXISTS.withDescription("Email already in use"))
        }
        
        val user = userRepository.create(
            username = request.username,
            email = request.email,
            password = request.password,
            firstName = request.firstName,
            lastName = request.lastName
        )
        
        return user.toProto()
    }
    
    override suspend fun getUser(request: GetUserRequest): User {
        return userRepository.findById(request.userId)?.toProto()
            ?: throw StatusException(
                Status.NOT_FOUND.withDescription("User not found: ${request.userId}")
            )
    }
    
    override suspend fun updateUser(request: UpdateUserRequest): User {
        val user = userRepository.findById(request.userId)
            ?: throw StatusException(Status.NOT_FOUND.withDescription("User not found"))
        
        val updated = userRepository.update(
            id = request.userId,
            firstName = if (request.hasFirstName()) request.firstName else user.firstName,
            lastName = if (request.hasLastName()) request.lastName else user.lastName,
            email = if (request.hasEmail()) request.email else user.email
        )
        
        return updated.toProto()
    }
    
    override suspend fun deleteUser(request: DeleteUserRequest): DeleteUserResponse {
        if (!userRepository.exists(request.userId)) {
            throw StatusException(Status.NOT_FOUND.withDescription("User not found"))
        }
        
        val success = userRepository.delete(request.userId)
        return deleteUserResponse { this.success = success }
    }
}

// Extension: domain model -> proto
fun UserDomain.toProto(): User = user {
    id = this@toProto.id
    username = this@toProto.username
    email = this@toProto.email
    firstName = this@toProto.firstName
    lastName = this@toProto.lastName
    role = when (this@toProto.role) {
        Role.ADMIN -> User.UserRole.ADMIN
        Role.MODERATOR -> User.UserRole.MODERATOR
        else -> User.UserRole.USER
    }
    createdAt = this@toProto.createdAt.toEpochMilli()
}
```

---

## Server Streaming

```kotlin
// Server streaming: server ส่งข้อมูลหลายชิ้นกลับมาตาม request เดียว

override fun listUsers(request: ListUsersRequest): Flow<User> = flow {
    val pageSize = request.pageSize.takeIf { it > 0 } ?: 20
    var offset = (request.page - 1) * pageSize
    var hasMore = true
    
    while (hasMore) {
        val batch = userRepository.findAll(
            limit = pageSize,
            offset = offset,
            roleFilter = request.roleFilter.takeIf { it.isNotBlank() }
        )
        
        if (batch.isEmpty()) {
            hasMore = false
        } else {
            batch.forEach { user ->
                emit(user.toProto())
            }
            offset += batch.size
            hasMore = batch.size == pageSize
        }
    }
}

// Client code สำหรับ server streaming
suspend fun receiveUsers(stub: UserServiceGrpcKt.UserServiceCoroutineStub) {
    val request = listUsersRequest {
        page = 1
        pageSize = 100
    }
    
    stub.listUsers(request).collect { user ->
        println("Received user: ${user.username}")
    }
}
```

---

## Client Streaming

```kotlin
// Client streaming: client ส่งข้อมูลหลายชิ้นแล้ว server ตอบกลับครั้งเดียว

override suspend fun batchCreateUsers(
    requests: Flow<CreateUserRequest>
): BatchCreateResponse {
    var createdCount = 0
    var failedCount = 0
    val errors = mutableListOf<String>()
    
    requests.collect { request ->
        try {
            userRepository.create(
                username = request.username,
                email = request.email,
                password = request.password,
                firstName = request.firstName,
                lastName = request.lastName
            )
            createdCount++
        } catch (e: Exception) {
            failedCount++
            errors.add("Failed for ${request.email}: ${e.message}")
        }
    }
    
    return batchCreateResponse {
        this.createdCount = createdCount
        this.failedCount = failedCount
        this.errors.addAll(errors)
    }
}

// Client side
suspend fun batchCreateUsers(
    stub: UserServiceGrpcKt.UserServiceCoroutineStub,
    users: List<UserData>
) {
    val requestFlow = flow {
        users.forEach { user ->
            emit(createUserRequest {
                username = user.username
                email = user.email
                password = user.tempPassword
                firstName = user.firstName
                lastName = user.lastName
            })
        }
    }
    
    val response = stub.batchCreateUsers(requestFlow)
    println("Created: ${response.createdCount}, Failed: ${response.failedCount}")
    response.errorsList.forEach { println("Error: $it") }
}
```

---

## Bidirectional Streaming

```kotlin
// Bidirectional: ทั้ง client และ server ส่งข้อมูลได้ตลอดเวลา

override fun syncUsers(requests: Flow<SyncRequest>): Flow<SyncResponse> = flow {
    requests.collect { request ->
        val response = when (request.action) {
            SyncRequest.SyncAction.UPSERT -> {
                try {
                    val user = upsertUser(request.data)
                    syncResponse {
                        userId = request.userId
                        success = true
                        message = "Upserted successfully"
                        this.user = user
                    }
                } catch (e: Exception) {
                    syncResponse {
                        userId = request.userId
                        success = false
                        message = e.message ?: "Unknown error"
                    }
                }
            }
            SyncRequest.SyncAction.DELETE -> {
                val deleted = userRepository.delete(request.userId)
                syncResponse {
                    userId = request.userId
                    success = deleted
                    message = if (deleted) "Deleted" else "Not found"
                }
            }
            SyncRequest.SyncAction.QUERY -> {
                val user = userRepository.findById(request.userId)
                syncResponse {
                    userId = request.userId
                    success = user != null
                    message = if (user != null) "Found" else "Not found"
                    if (user != null) this.user = user.toProto()
                }
            }
            else -> syncResponse {
                userId = request.userId
                success = false
                message = "Unknown action"
            }
        }
        emit(response)
    }
}

private suspend fun upsertUser(data: User): User {
    return if (userRepository.exists(data.id)) {
        userRepository.update(data.id, data.firstName, data.lastName, data.email).toProto()
    } else {
        userRepository.createFromProto(data).toProto()
    }
}
```

---

## Interceptors (Middleware)

```kotlin
import io.grpc.*
import io.grpc.kotlin.GrpcContextElement
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.withContext

// Server-side interceptor
class AuthServerInterceptor(private val jwtService: JwtService) : ServerInterceptor {
    
    companion object {
        val USER_CLAIMS_KEY: Context.Key<UserClaims> = Context.key("user_claims")
    }
    
    override fun <ReqT, RespT> interceptCall(
        call: ServerCall<ReqT, RespT>,
        headers: Metadata,
        next: ServerCallHandler<ReqT, RespT>
    ): ServerCall.Listener<ReqT> {
        val authHeader = headers.get(
            Metadata.Key.of("Authorization", Metadata.ASCII_STRING_MARSHALLER)
        )
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            call.close(
                Status.UNAUTHENTICATED.withDescription("Missing or invalid authorization"),
                Metadata()
            )
            return object : ServerCall.Listener<ReqT>() {}
        }
        
        val token = authHeader.removePrefix("Bearer ")
        val claims = jwtService.validateToken(token).getOrElse {
            call.close(
                Status.UNAUTHENTICATED.withDescription("Invalid token: ${it.message}"),
                Metadata()
            )
            return object : ServerCall.Listener<ReqT>() {}
        }
        
        val ctx = Context.current().withValue(USER_CLAIMS_KEY, claims)
        return Contexts.interceptCall(ctx, call, headers, next)
    }
}

// Logging interceptor
class LoggingServerInterceptor : ServerInterceptor {
    
    override fun <ReqT, RespT> interceptCall(
        call: ServerCall<ReqT, RespT>,
        headers: Metadata,
        next: ServerCallHandler<ReqT, RespT>
    ): ServerCall.Listener<ReqT> {
        val methodName = call.methodDescriptor.fullMethodName
        val startTime = System.currentTimeMillis()
        
        println("gRPC: $methodName started")
        
        val delegate = next.startCall(object : ForwardingServerCall.SimpleForwardingServerCall<ReqT, RespT>(call) {
            override fun close(status: Status, trailers: Metadata) {
                val duration = System.currentTimeMillis() - startTime
                println("gRPC: $methodName completed in ${duration}ms status=${status.code}")
                super.close(status, trailers)
            }
        }, headers)
        
        return delegate
    }
}

// Client-side interceptor
class ClientAuthInterceptor(private val tokenProvider: () -> String) : ClientInterceptor {
    
    override fun <ReqT, RespT> interceptCall(
        method: MethodDescriptor<ReqT, RespT>,
        callOptions: CallOptions,
        next: Channel
    ): ClientCall<ReqT, RespT> {
        return object : ForwardingClientCall.SimpleForwardingClientCall<ReqT, RespT>(
            next.newCall(method, callOptions)
        ) {
            override fun start(responseListener: Listener<RespT>, headers: Metadata) {
                headers.put(
                    Metadata.Key.of("Authorization", Metadata.ASCII_STRING_MARSHALLER),
                    "Bearer ${tokenProvider()}"
                )
                super.start(responseListener, headers)
            }
        }
    }
}

// Using interceptors in server
fun startServer(userService: UserGrpcService, jwtService: JwtService): Server {
    return ServerBuilder.forPort(50051)
        .addService(userService)
        .intercept(AuthServerInterceptor(jwtService))
        .intercept(LoggingServerInterceptor())
        .build()
        .start()
}

// Creating client with interceptor
fun createClient(host: String, port: Int, tokenProvider: () -> String): UserServiceGrpcKt.UserServiceCoroutineStub {
    val channel = ManagedChannelBuilder.forAddress(host, port)
        .usePlaintext()
        .intercept(ClientAuthInterceptor(tokenProvider))
        .build()
    
    return UserServiceGrpcKt.UserServiceCoroutineStub(channel)
}
```

---

## Error Handling

```kotlin
// gRPC Status codes
sealed class GrpcError(val status: Status) : Exception() {
    class NotFound(message: String) : GrpcError(Status.NOT_FOUND.withDescription(message))
    class InvalidArgument(message: String) : GrpcError(Status.INVALID_ARGUMENT.withDescription(message))
    class AlreadyExists(message: String) : GrpcError(Status.ALREADY_EXISTS.withDescription(message))
    class PermissionDenied(message: String) : GrpcError(Status.PERMISSION_DENIED.withDescription(message))
    class Internal(message: String) : GrpcError(Status.INTERNAL.withDescription(message))
}

// Exception handler wrapper
suspend fun <T> handleGrpcExceptions(block: suspend () -> T): T {
    return try {
        block()
    } catch (e: GrpcError) {
        throw StatusException(e.status)
    } catch (e: IllegalArgumentException) {
        throw StatusException(Status.INVALID_ARGUMENT.withDescription(e.message))
    } catch (e: NoSuchElementException) {
        throw StatusException(Status.NOT_FOUND.withDescription(e.message))
    } catch (e: Exception) {
        throw StatusException(Status.INTERNAL.withDescription("Internal error: ${e.message}"))
    }
}

// Use in service
class SafeUserGrpcService(private val userRepository: UserRepository) 
    : UserServiceGrpcKt.UserServiceCoroutineImplBase() {
    
    override suspend fun getUser(request: GetUserRequest): User = 
        handleGrpcExceptions {
            userRepository.findById(request.userId)?.toProto()
                ?: throw GrpcError.NotFound("User ${request.userId} not found")
        }
}

// Client error handling
suspend fun safeGetUser(stub: UserServiceGrpcKt.UserServiceCoroutineStub, userId: String): User? {
    return try {
        stub.getUser(getUserRequest { this.userId = userId })
    } catch (e: StatusException) {
        when (e.status.code) {
            Status.Code.NOT_FOUND -> null
            Status.Code.UNAUTHENTICATED -> throw Exception("Please login first")
            Status.Code.PERMISSION_DENIED -> throw Exception("Access denied")
            else -> throw Exception("RPC error: ${e.status.description}")
        }
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Chat Service with gRPC Streaming

/*
Chat proto (implement this):

service ChatService {
  rpc SendMessage(ChatMessage) returns (SendMessageResponse);
  rpc GetHistory(GetHistoryRequest) returns (stream ChatMessage);
  rpc JoinRoom(stream UserAction) returns (stream ChatMessage);  // bidirectional
}

message ChatMessage {
  string id = 1;
  string room_id = 2;
  string user_id = 3;
  string username = 4;
  string content = 5;
  int64 timestamp = 6;
}

message SendMessageResponse {
  string message_id = 1;
  bool success = 2;
}

message GetHistoryRequest {
  string room_id = 1;
  int32 limit = 2;
  int64 before_timestamp = 3;
}

message UserAction {
  string user_id = 1;
  string username = 2;
  ActionType action = 3;
  string room_id = 4;
  
  enum ActionType {
    JOIN = 0;
    LEAVE = 1;
    TYPING = 2;
    STOP_TYPING = 3;
  }
}
*/

// TODO: Implement ChatGrpcService
class ChatGrpcService(
    private val messageRepository: MessageRepository,
    private val roomManager: RoomManager
) {
    // Hint: use SharedFlow for broadcasting messages to room members
    private val roomFlows = java.util.concurrent.ConcurrentHashMap<String, 
        kotlinx.coroutines.flow.MutableSharedFlow<com.example.grpc.proto.ChatMessage>>()
    
    fun getRoomFlow(roomId: String) = roomFlows.getOrPut(roomId) {
        kotlinx.coroutines.flow.MutableSharedFlow(replay = 50)
    }
    
    // sendMessage: save to repository and broadcast to room
    // getHistory: stream from repository
    // joinRoom: collect user actions, emit room messages
}

interface MessageRepository {
    suspend fun save(message: Any): Any
    suspend fun findByRoom(roomId: String, limit: Int, beforeTimestamp: Long): List<Any>
}

interface RoomManager {
    suspend fun addMember(roomId: String, userId: String, username: String)
    suspend fun removeMember(roomId: String, userId: String)
    suspend fun getMembers(roomId: String): List<String>
}
```

---

## สรุป Part 36

```
✅ gRPC ใช้ HTTP/2 + Protobuf -> faster than REST JSON
✅ .proto files define contract -> auto-generate code
✅ Unary: 1 request, 1 response (เหมือน REST)
✅ Server Streaming: 1 request, หลาย response (flow)
✅ Client Streaming: หลาย request, 1 response
✅ Bidirectional: หลาย request, หลาย response พร้อมกัน
✅ Interceptors: authentication, logging, metrics
✅ StatusException: gRPC-specific error codes
✅ Type-safe with generated Kotlin DSL builders
✅ Best for: microservices, internal APIs, high-performance
```

---

*Part 36/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
