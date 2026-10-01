# Part 35: GraphQL ด้วย Kotlin

## สารบัญ
1. [GraphQL คืออะไร](#graphql-คืออะไร)
2. [Schema Definition](#schema-definition)
3. [Resolvers และ Queries](#resolvers-และ-queries)
4. [Mutations](#mutations)
5. [Subscriptions](#subscriptions)
6. [N+1 Problem และ DataLoader](#n1-problem-และ-dataloader)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## GraphQL คืออะไร

GraphQL คือ query language สำหรับ API ที่ให้ client กำหนดเองว่าต้องการข้อมูลอะไรบ้าง แทนที่จะรับข้อมูลทั้งหมดเหมือน REST API

```
REST API:
GET /users/1          -> { id, name, email, address, phone, ... }  // over-fetching
GET /users/1/posts    -> [ {id, title, content, ...} ]            // under-fetching (2 requests)

GraphQL:
query {
  user(id: "1") {
    name          # เลือกเฉพาะที่ต้องการ
    posts {
      title       # nested data ใน 1 request
    }
  }
}
```

---

## Setup กับ Spring Boot + graphql-kotlin

```kotlin
// build.gradle.kts
plugins {
    id("com.expediagroup.graphql") version "7.1.4"
}

dependencies {
    implementation("com.expediagroup:graphql-kotlin-spring-server:7.1.4")
    implementation("com.expediagroup:graphql-kotlin-hooks-provider:7.1.4")
}
```

```yaml
# application.yml
graphql:
  packages:
    - "com.example.graphql"
  playground:
    enabled: true   # /playground endpoint for testing
  sdl:
    enabled: true   # /sdl endpoint for schema
```

---

## Data Classes (Schema Types)

```kotlin
package com.example.graphql.model

import com.expediagroup.graphql.generator.annotations.GraphQLDescription
import com.expediagroup.graphql.generator.annotations.GraphQLIgnore
import com.expediagroup.graphql.generator.scalars.ID

@GraphQLDescription("ผู้ใช้งานในระบบ")
data class User(
    val id: ID,
    val username: String,
    val email: String,
    val firstName: String,
    val lastName: String,
    val role: UserRole = UserRole.USER,
    
    @GraphQLIgnore  // ไม่ expose ใน schema
    val passwordHash: String = ""
) {
    // computed field
    val fullName: String get() = "$firstName $lastName"
}

enum class UserRole {
    USER, MODERATOR, ADMIN
}

@GraphQLDescription("โพสต์บล็อก")
data class Post(
    val id: ID,
    val title: String,
    val content: String,
    val published: Boolean = false,
    val authorId: String,
    val tags: List<String> = emptyList(),
    val createdAt: String,
    val updatedAt: String
)

data class Comment(
    val id: ID,
    val postId: String,
    val authorId: String,
    val content: String,
    val createdAt: String
)

// Input types
data class CreatePostInput(
    val title: String,
    val content: String,
    val tags: List<String> = emptyList(),
    val published: Boolean = false
)

data class UpdatePostInput(
    val id: ID,
    val title: String? = null,
    val content: String? = null,
    val published: Boolean? = null,
    val tags: List<String>? = null
)

data class CreateCommentInput(
    val postId: String,
    val content: String
)

// Pagination
data class PageInfo(
    val hasNextPage: Boolean,
    val hasPreviousPage: Boolean,
    val startCursor: String?,
    val endCursor: String?
)

data class PostConnection(
    val edges: List<PostEdge>,
    val pageInfo: PageInfo,
    val totalCount: Int
)

data class PostEdge(
    val node: Post,
    val cursor: String
)

data class PaginationInput(
    val first: Int? = null,
    val after: String? = null,
    val last: Int? = null,
    val before: String? = null
)
```

---

## Queries

```kotlin
package com.example.graphql.query

import com.expediagroup.graphql.server.operations.Query
import com.expediagroup.graphql.generator.annotations.GraphQLDescription
import com.expediagroup.graphql.generator.scalars.ID
import org.springframework.stereotype.Component
import graphql.schema.DataFetchingEnvironment

@Component
class UserQuery(private val userService: UserService) : Query {
    
    @GraphQLDescription("ดึงข้อมูลผู้ใช้ทั้งหมด")
    fun users(
        role: UserRole? = null,
        limit: Int = 20,
        offset: Int = 0
    ): List<User> {
        return if (role != null) {
            userService.findByRole(role, limit, offset)
        } else {
            userService.findAll(limit, offset)
        }
    }
    
    @GraphQLDescription("ดึงข้อมูลผู้ใช้จาก ID")
    fun user(id: ID): User? {
        return userService.findById(id.value)
    }
    
    @GraphQLDescription("ค้นหาผู้ใช้จาก username")
    fun userByUsername(username: String): User? {
        return userService.findByUsername(username)
    }
    
    @GraphQLDescription("ดึงข้อมูล profile ของตัวเอง")
    fun me(dfe: DataFetchingEnvironment): User? {
        val context = dfe.graphQlContext.get<AuthContext>("auth")
        return context?.userId?.let { userService.findById(it) }
    }
}

@Component
class PostQuery(private val postService: PostService) : Query {
    
    @GraphQLDescription("ดึงโพสต์ทั้งหมด (รองรับ pagination)")
    fun posts(
        pagination: PaginationInput = PaginationInput(first = 20),
        authorId: String? = null,
        tag: String? = null,
        published: Boolean? = null
    ): PostConnection {
        return postService.findPaginated(pagination, authorId, tag, published)
    }
    
    @GraphQLDescription("ดึงโพสต์จาก ID")
    fun post(id: ID): Post? {
        return postService.findById(id.value)
    }
    
    @GraphQLDescription("ค้นหาโพสต์จาก keyword")
    fun searchPosts(keyword: String, limit: Int = 10): List<Post> {
        return postService.search(keyword, limit)
    }
    
    @GraphQLDescription("ดึง tags ยอดนิยม")
    fun popularTags(limit: Int = 20): List<String> {
        return postService.getPopularTags(limit)
    }
}
```

---

## Mutations

```kotlin
package com.example.graphql.mutation

import com.expediagroup.graphql.server.operations.Mutation
import com.expediagroup.graphql.generator.annotations.GraphQLDescription
import com.expediagroup.graphql.generator.scalars.ID
import graphql.schema.DataFetchingEnvironment
import org.springframework.stereotype.Component

// Mutation result types
sealed class PostMutationResult
data class PostSuccess(val post: Post) : PostMutationResult()
data class PostError(val message: String, val code: String) : PostMutationResult()

@Component
class PostMutation(
    private val postService: PostService,
    private val authService: AuthService
) : Mutation {
    
    @GraphQLDescription("สร้างโพสต์ใหม่")
    fun createPost(input: CreatePostInput, dfe: DataFetchingEnvironment): Post {
        val userId = requireAuth(dfe)
        return postService.create(userId, input)
    }
    
    @GraphQLDescription("แก้ไขโพสต์")
    fun updatePost(input: UpdatePostInput, dfe: DataFetchingEnvironment): Post {
        val userId = requireAuth(dfe)
        val post = postService.findById(input.id.value) 
            ?: throw GraphQLException("Post not found")
        
        if (post.authorId != userId) {
            throw GraphQLException("Not authorized to edit this post")
        }
        
        return postService.update(input)
    }
    
    @GraphQLDescription("ลบโพสต์")
    fun deletePost(id: ID, dfe: DataFetchingEnvironment): Boolean {
        val userId = requireAuth(dfe)
        val post = postService.findById(id.value) 
            ?: throw GraphQLException("Post not found")
        
        if (post.authorId != userId) {
            throw GraphQLException("Not authorized to delete this post")
        }
        
        return postService.delete(id.value)
    }
    
    @GraphQLDescription("เผยแพร่โพสต์")
    fun publishPost(id: ID, dfe: DataFetchingEnvironment): Post {
        val userId = requireAuth(dfe)
        return postService.publish(id.value, userId)
    }
    
    @GraphQLDescription("เพิ่ม comment")
    fun addComment(input: CreateCommentInput, dfe: DataFetchingEnvironment): Comment {
        val userId = requireAuth(dfe)
        return postService.addComment(userId, input)
    }
    
    private fun requireAuth(dfe: DataFetchingEnvironment): String {
        val context = dfe.graphQlContext.get<AuthContext>("auth")
        return context?.userId ?: throw GraphQLException("Authentication required")
    }
}

@Component  
class AuthMutation(private val authService: AuthService) : Mutation {
    
    @GraphQLDescription("ลงทะเบียนผู้ใช้ใหม่")
    fun register(username: String, email: String, password: String): AuthPayload {
        return authService.register(username, email, password)
    }
    
    @GraphQLDescription("เข้าสู่ระบบ")
    fun login(email: String, password: String): AuthPayload {
        return authService.login(email, password)
    }
    
    @GraphQLDescription("ออกจากระบบ")
    fun logout(dfe: DataFetchingEnvironment): Boolean {
        val token = dfe.graphQlContext.get<String>("token") ?: return false
        return authService.invalidateToken(token)
    }
    
    @GraphQLDescription("refresh access token")
    fun refreshToken(refreshToken: String): AuthPayload {
        return authService.refreshToken(refreshToken)
    }
}

data class AuthPayload(
    val accessToken: String,
    val refreshToken: String,
    val user: User
)
```

---

## Subscriptions

```kotlin
package com.example.graphql.subscription

import com.expediagroup.graphql.server.operations.Subscription
import com.expediagroup.graphql.generator.annotations.GraphQLDescription
import graphql.schema.DataFetchingEnvironment
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.filter
import kotlinx.coroutines.flow.map
import org.reactivestreams.Publisher
import org.springframework.stereotype.Component
import reactor.core.publisher.Flux

@Component
class PostSubscription(
    private val eventBus: GraphQLEventBus
) : Subscription {
    
    @GraphQLDescription("รับแจ้งเตือนเมื่อมีโพสต์ใหม่")
    fun onPostCreated(authorId: String? = null): Publisher<Post> {
        return if (authorId != null) {
            eventBus.postCreatedEvents
                .filter { it.authorId == authorId }
        } else {
            eventBus.postCreatedEvents
        }
    }
    
    @GraphQLDescription("รับแจ้งเตือนเมื่อมีความคิดเห็นใหม่")
    fun onCommentAdded(postId: String): Publisher<Comment> {
        return eventBus.commentAddedEvents
            .filter { it.postId == postId }
    }
    
    @GraphQLDescription("รับแจ้งเตือนแบบ real-time")
    fun onNotification(dfe: DataFetchingEnvironment): Publisher<Notification> {
        val context = dfe.graphQlContext.get<AuthContext>("auth")
            ?: throw GraphQLException("Authentication required for subscriptions")
        
        return eventBus.userNotifications
            .filter { it.userId == context.userId }
    }
    
    @GraphQLDescription("ติดตามจำนวน live users")
    fun liveUserCount(): Publisher<Int> {
        return Flux.interval(java.time.Duration.ofSeconds(5))
            .map { eventBus.getLiveUserCount() }
    }
}

// Event bus
@Component
class GraphQLEventBus {
    private val postCreatedSink = reactor.core.publisher.Sinks.many().multicast().directBestEffort<Post>()
    private val commentAddedSink = reactor.core.publisher.Sinks.many().multicast().directBestEffort<Comment>()
    private val notificationSink = reactor.core.publisher.Sinks.many().multicast().directBestEffort<Notification>()
    
    val postCreatedEvents: Flux<Post> = postCreatedSink.asFlux()
    val commentAddedEvents: Flux<Comment> = commentAddedSink.asFlux()
    val userNotifications: Flux<Notification> = notificationSink.asFlux()
    
    fun publishPostCreated(post: Post) {
        postCreatedSink.tryEmitNext(post)
    }
    
    fun publishCommentAdded(comment: Comment) {
        commentAddedSink.tryEmitNext(comment)
    }
    
    fun publishNotification(notification: Notification) {
        notificationSink.tryEmitNext(notification)
    }
    
    fun getLiveUserCount(): Int = 42  // demo
}

data class Notification(
    val id: String,
    val userId: String,
    val type: NotificationType,
    val message: String,
    val read: Boolean = false,
    val createdAt: String
)

enum class NotificationType {
    NEW_COMMENT, NEW_FOLLOWER, POST_LIKED, MENTION
}
```

---

## N+1 Problem และ DataLoader

```kotlin
// N+1 Problem: ดึง posts แล้วต้องดึง author แต่ละคน = N+1 queries
// Solution: DataLoader batching

import org.dataloader.BatchLoaderEnvironment
import org.dataloader.MappedBatchLoaderWithContext
import java.util.concurrent.CompletableFuture

// DataLoader สำหรับ User
@Component
class UserDataLoader(private val userService: UserService) 
    : MappedBatchLoaderWithContext<String, User> {
    
    override fun load(
        keys: Set<String>,
        environment: BatchLoaderEnvironment
    ): CompletableFuture<Map<String, User>> {
        return CompletableFuture.supplyAsync {
            // 1 query แทนที่จะเป็น N queries!
            userService.findAllByIds(keys.toList())
                .associateBy { it.id.value }
        }
    }
}

// Register DataLoaders
@Component
class CustomDataLoaderRegistrar(
    private val userDataLoader: UserDataLoader,
    private val postDataLoader: PostDataLoader
) : DataLoaderRegistrar {
    
    override fun registerDataLoaders(
        dataLoaderRegistry: DataLoaderRegistry,
        graphQLContext: GraphQLContext
    ) {
        dataLoaderRegistry.register("userLoader", 
            DataLoaderFactory.newMappedDataLoader(userDataLoader))
        dataLoaderRegistry.register("postLoader",
            DataLoaderFactory.newMappedDataLoader(postDataLoader))
    }
}

// ใช้ DataLoader ใน resolver
@Component
class PostDataFetcher : SchemaGeneratorHooks {
    
    // resolver สำหรับ Post.author field
    fun getAuthor(post: Post, dfe: DataFetchingEnvironment): CompletableFuture<User?> {
        val userLoader = dfe.getDataLoader<String, User>("userLoader")
        return userLoader.load(post.authorId)
    }
    
    // resolver สำหรับ Post.comments field
    fun getComments(post: Post, dfe: DataFetchingEnvironment): CompletableFuture<List<Comment>> {
        val commentLoader = dfe.getDataLoader<String, List<Comment>>("commentLoader")
        return commentLoader.load(post.id.value)
    }
}

// Extended type with DataLoader
@Component
class UserResolver(private val postService: PostService) {
    
    fun posts(user: User, dfe: DataFetchingEnvironment): CompletableFuture<List<Post>> {
        val postLoader = dfe.getDataLoader<String, List<Post>>("userPostsLoader")
        return postLoader.load(user.id.value)
    }
    
    fun postCount(user: User, dfe: DataFetchingEnvironment): CompletableFuture<Int> {
        return posts(user, dfe).thenApply { it.size }
    }
}
```

---

## Custom Scalars

```kotlin
// Custom scalar type (DateTime)
import com.expediagroup.graphql.generator.scalars.ID
import graphql.language.StringValue
import graphql.schema.Coercing
import graphql.schema.CoercingParseLiteralException
import graphql.schema.CoercingParseValueException
import graphql.schema.CoercingSerializeException
import graphql.schema.GraphQLScalarType
import java.time.Instant
import java.time.format.DateTimeFormatter

val DateTimeScalar: GraphQLScalarType = GraphQLScalarType.newScalar()
    .name("DateTime")
    .description("ISO 8601 date-time string")
    .coercing(object : Coercing<Instant, String> {
        
        override fun serialize(dataFetcherResult: Any): String {
            return when (dataFetcherResult) {
                is Instant -> DateTimeFormatter.ISO_INSTANT.format(dataFetcherResult)
                is String -> dataFetcherResult  // pass-through
                else -> throw CoercingSerializeException("Cannot serialize $dataFetcherResult as DateTime")
            }
        }
        
        override fun parseValue(input: Any): Instant {
            return when (input) {
                is String -> try {
                    Instant.parse(input)
                } catch (e: Exception) {
                    throw CoercingParseValueException("Invalid DateTime: $input")
                }
                else -> throw CoercingParseValueException("Cannot parse $input as DateTime")
            }
        }
        
        override fun parseLiteral(input: Any): Instant {
            return when (input) {
                is StringValue -> parseValue(input.value)
                else -> throw CoercingParseLiteralException("Cannot parse literal $input as DateTime")
            }
        }
    })
    .build()

// Register scalar
@Component
class CustomScalarHooks : SchemaGeneratorHooks {
    override fun willGenerateGraphQLType(type: KType): GraphQLType? {
        return when (type.classifier) {
            Instant::class -> DateTimeScalar
            else -> super.willGenerateGraphQLType(type)
        }
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: E-commerce GraphQL API

// Schema types
data class Product(
    val id: ID,
    val name: String,
    val description: String,
    val price: Double,
    val stock: Int,
    val categoryId: String,
    val images: List<String>,
    val rating: Double = 0.0,
    val reviewCount: Int = 0
)

data class Category(
    val id: ID,
    val name: String,
    val parentId: String?
)

data class CartItem(
    val productId: String,
    val quantity: Int,
    val price: Double
)

data class Cart(
    val id: ID,
    val userId: String,
    val items: List<CartItem>,
    val total: Double
)

data class Order(
    val id: ID,
    val userId: String,
    val items: List<OrderItem>,
    val status: OrderStatus,
    val total: Double,
    val createdAt: String
)

data class OrderItem(
    val productId: String,
    val quantity: Int,
    val price: Double
)

enum class OrderStatus { PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED }

// TODO: Implement these queries
// 1. products(categoryId, minPrice, maxPrice, inStock) -> [Product]
// 2. product(id) -> Product
// 3. categories() -> [Category]
// 4. myCart() -> Cart (requires auth)
// 5. myOrders(status) -> [Order] (requires auth)

// TODO: Implement these mutations
// 1. addToCart(productId, quantity) -> Cart
// 2. removeFromCart(productId) -> Cart
// 3. updateCartItem(productId, quantity) -> Cart
// 4. checkout(shippingAddress) -> Order
// 5. cancelOrder(orderId) -> Order

// TODO: Implement subscriptions
// 1. onOrderStatusChanged(orderId) -> Order
// 2. onStockChanged(productId) -> Product

// Hint: Use DataLoader for Product.category
class CategoryDataLoader(private val categoryService: CategoryService) 
    : MappedBatchLoaderWithContext<String, Category> {
    override fun load(
        keys: Set<String>,
        environment: BatchLoaderEnvironment
    ): CompletableFuture<Map<String, Category>> {
        return CompletableFuture.supplyAsync {
            categoryService.findAllByIds(keys.toList()).associateBy { it.id.value }
        }
    }
}

// Placeholder interfaces
interface UserService {
    fun findById(id: String): User?
    fun findByUsername(username: String): User?
    fun findByRole(role: UserRole, limit: Int, offset: Int): List<User>
    fun findAll(limit: Int, offset: Int): List<User>
    fun findAllByIds(ids: List<String>): List<User>
}

interface PostService {
    fun findById(id: String): Post?
    fun findPaginated(pagination: PaginationInput, authorId: String?, tag: String?, published: Boolean?): PostConnection
    fun search(keyword: String, limit: Int): List<Post>
    fun getPopularTags(limit: Int): List<String>
    fun create(userId: String, input: CreatePostInput): Post
    fun update(input: UpdatePostInput): Post
    fun delete(id: String): Boolean
    fun publish(id: String, userId: String): Post
    fun addComment(userId: String, input: CreateCommentInput): Comment
}

interface AuthService {
    fun register(username: String, email: String, password: String): AuthPayload
    fun login(email: String, password: String): AuthPayload
    fun invalidateToken(token: String): Boolean
    fun refreshToken(refreshToken: String): AuthPayload
}

interface CategoryService {
    fun findAllByIds(ids: List<String>): List<Category>
}

data class AuthContext(val userId: String, val roles: List<String>)

class GraphQLException(message: String) : RuntimeException(message)

// Stub imports
typealias DataLoaderRegistry = Any
typealias DataLoaderFactory = Any
typealias DataLoaderRegistrar = Any
typealias SchemaGeneratorHooks = Any
typealias GraphQLContext = Any
typealias DataFetchingEnvironment = Any
typealias KType = Any
typealias GraphQLType = Any
typealias PostDataLoader = Any
```

---

## สรุป Part 35

```
✅ GraphQL แก้ over-fetching และ under-fetching
✅ Schema-first vs code-first approach
✅ Query: ดึงข้อมูล แบบ flexible
✅ Mutation: เปลี่ยนแปลงข้อมูล
✅ Subscription: real-time data ผ่าน WebSocket
✅ DataLoader: แก้ N+1 problem ด้วย batching
✅ Custom Scalars: DateTime, Money ฯลฯ
✅ Authentication: DataFetchingEnvironment context
✅ Pagination: cursor-based ด้วย Connection pattern
✅ graphql-kotlin: Spring Boot integration ง่าย
```

---

*Part 35/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
