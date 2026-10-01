# Part 89: GraphQL & gRPC APIs

## สารบัญ
1. [GraphQL ด้วย Spring for GraphQL](#graphql-ด้วย-spring-for-graphql)
2. [DataLoader สำหรับ N+1 Problem](#dataloader-สำหรับ-n1-problem)
3. [GraphQL Subscriptions](#graphql-subscriptions)
4. [gRPC ด้วย Kotlin](#grpc-ด้วย-kotlin)
5. [gRPC Streaming](#grpc-streaming)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## GraphQL คืออะไร

```
GraphQL vs REST:

REST:
GET /products/123          → ได้ข้อมูลทั้งหมด (over-fetching)
GET /products/123/reviews  → request แยก (under-fetching/N+1)
GET /products/123/seller

GraphQL: 1 request รับ exactly ข้อมูลที่ต้องการ
query {
  product(id: "123") {
    name
    price
    reviews(limit: 5) { rating comment }
    seller { name rating }
  }
}

Advantages:
✅ Client controls what data it needs
✅ Strongly typed schema
✅ Single endpoint /graphql
✅ Real-time subscriptions
✅ Introspection / self-documenting

Disadvantages:
❌ Complex query caching
❌ N+1 problem (need DataLoader)
❌ File upload tricky
❌ Overkill for simple CRUD

gRPC vs REST:
gRPC: Protocol Buffers (binary), HTTP/2, bidirectional streaming
REST: JSON (text), HTTP/1.1, request-response

gRPC ดีกว่า REST เมื่อ:
✅ Service-to-service communication
✅ Performance-critical (3-10x faster than JSON)
✅ Bidirectional streaming
✅ Strict contract enforcement
```

---

## GraphQL ด้วย Spring for GraphQL

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-graphql")
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-websocket")  // subscriptions
    implementation("com.graphql-java:graphql-java-extended-scalars:21.0")
    testImplementation("org.springframework.graphql:spring-graphql-test")
}

// schema.graphqls (src/main/resources/graphql/)
/*
type Query {
  product(id: ID!): Product
  products(filter: ProductFilter, page: Int = 0, size: Int = 20): ProductPage!
  searchProducts(query: String!, page: Int = 0): ProductPage!
}

type Mutation {
  createProduct(input: CreateProductInput!): Product!
  updateProduct(id: ID!, input: UpdateProductInput!): Product!
  deleteProduct(id: ID!): Boolean!
  placeOrder(input: PlaceOrderInput!): Order!
}

type Subscription {
  orderStatusChanged(orderId: ID!): OrderStatusEvent!
  productStockChanged(productId: ID!): StockEvent!
}

type Product {
  id: ID!
  name: String!
  description: String
  price: Float!
  currency: String!
  stockQuantity: Int!
  category: Category!
  images: [String!]!
  reviews(limit: Int = 10): [Review!]!
  seller: Seller
  isAvailable: Boolean!
  createdAt: DateTime!
}

type Category {
  id: ID!
  name: String!
  parentCategory: Category
  products(page: Int = 0, size: Int = 20): ProductPage!
}

type Review {
  id: ID!
  rating: Int!
  comment: String
  author: User!
  createdAt: DateTime!
}

type Order {
  id: ID!
  status: OrderStatus!
  items: [OrderItem!]!
  totalAmount: Float!
  createdAt: DateTime!
}

type OrderItem {
  product: Product!
  quantity: Int!
  unitPrice: Float!
  subtotal: Float!
}

type User {
  id: ID!
  name: String!
  email: String!
}

type Seller {
  id: ID!
  name: String!
  rating: Float!
}

type ProductPage {
  items: [Product!]!
  totalCount: Int!
  hasNextPage: Boolean!
}

type OrderStatusEvent {
  orderId: ID!
  status: OrderStatus!
  timestamp: DateTime!
}

type StockEvent {
  productId: ID!
  newStock: Int!
  timestamp: DateTime!
}

enum OrderStatus {
  PENDING
  CONFIRMED
  PROCESSING
  SHIPPED
  DELIVERED
  CANCELLED
}

input ProductFilter {
  categoryId: ID
  minPrice: Float
  maxPrice: Float
  inStock: Boolean
  sortBy: ProductSortBy = CREATED_AT_DESC
}

input CreateProductInput {
  name: String!
  description: String
  price: Float!
  currency: String = "THB"
  stockQuantity: Int!
  categoryId: ID!
}

input UpdateProductInput {
  name: String
  description: String
  price: Float
  stockQuantity: Int
}

input PlaceOrderInput {
  items: [OrderItemInput!]!
}

input OrderItemInput {
  productId: ID!
  quantity: Int!
}

enum ProductSortBy {
  PRICE_ASC
  PRICE_DESC
  CREATED_AT_DESC
  NAME_ASC
}

scalar DateTime
*/

// Product GraphQL Controller
@org.springframework.graphql.data.method.annotation.QueryMapping
@org.springframework.graphql.data.method.annotation.MutationMapping
import org.springframework.graphql.data.method.annotation.*
import org.springframework.stereotype.Controller

@Controller
class ProductGraphQLController(
    private val productService: ProductService,
    private val categoryService: CategoryService
) {
    
    @QueryMapping
    suspend fun product(@Argument id: String): ProductDto? {
        return productService.getById(id)?.toDto()
    }
    
    @QueryMapping
    suspend fun products(
        @Argument filter: ProductFilterInput?,
        @Argument page: Int,
        @Argument size: Int
    ): ProductPageDto {
        val result = productService.getProducts(filter?.toDomain(), page, size)
        return ProductPageDto(
            items = result.items.map { it.toDto() },
            totalCount = result.total.toInt(),
            hasNextPage = result.hasMore
        )
    }
    
    @QueryMapping
    suspend fun searchProducts(
        @Argument query: String,
        @Argument page: Int
    ): ProductPageDto {
        val result = productService.search(query, page, 20)
        return ProductPageDto(
            items = result.items.map { it.toDto() },
            totalCount = result.total.toInt(),
            hasNextPage = result.hasMore
        )
    }
    
    @MutationMapping
    @org.springframework.security.access.prepost.PreAuthorize("hasRole('ADMIN')")
    suspend fun createProduct(@Argument input: CreateProductInput): ProductDto {
        return productService.create(input.toCommand()).toDto()
    }
    
    @MutationMapping
    @org.springframework.security.access.prepost.PreAuthorize("hasRole('ADMIN')")
    suspend fun updateProduct(@Argument id: String, @Argument input: UpdateProductInput): ProductDto {
        return productService.update(id, input.toCommand()).toDto()
    }
    
    @MutationMapping
    @org.springframework.security.access.prepost.PreAuthorize("hasRole('ADMIN')")
    suspend fun deleteProduct(@Argument id: String): Boolean {
        productService.delete(id)
        return true
    }
    
    // Batch field resolver using DataLoader
    @SchemaMapping(typeName = "Product", field = "category")
    suspend fun productCategory(
        product: ProductDto,
        loader: org.springframework.graphql.execution.BatchLoaderRegistry
    ): CategoryDto? {
        return categoryService.getById(product.categoryId)?.toDto()
    }
}

// Order controller
@Controller
class OrderGraphQLController(
    private val orderService: OrderApplicationService,
    private val orderStatusPublisher: OrderStatusPublisher
) {
    
    @MutationMapping
    @org.springframework.security.access.prepost.PreAuthorize("isAuthenticated()")
    suspend fun placeOrder(
        @Argument input: PlaceOrderInput,
        authentication: org.springframework.security.core.Authentication
    ): OrderDto {
        val userId = authentication.name
        val command = PlaceOrderCommand(
            customerId = userId,
            items = input.items.map { OrderItemCommand(it.productId, it.quantity) }
        )
        val orderId = orderService.placeOrder(command)
        return orderService.getOrder(orderId.value)!!.toDto()
    }
    
    @SubscriptionMapping
    fun orderStatusChanged(@Argument orderId: String): reactor.core.publisher.Flux<OrderStatusEventDto> {
        return orderStatusPublisher.subscribe(orderId)
            .map { event -> OrderStatusEventDto(orderId, event.status.name, System.currentTimeMillis()) }
    }
}

// DTOs
data class ProductDto(
    val id: String,
    val name: String,
    val description: String?,
    val price: Double,
    val currency: String,
    val stockQuantity: Int,
    val categoryId: String,
    val images: List<String>,
    val isAvailable: Boolean,
    val createdAt: String
)

data class ProductPageDto(val items: List<ProductDto>, val totalCount: Int, val hasNextPage: Boolean)
data class CategoryDto(val id: String, val name: String)
data class OrderDto(val id: String, val status: String, val totalAmount: Double, val createdAt: String)
data class OrderStatusEventDto(val orderId: String, val status: String, val timestamp: Long)

data class ProductFilterInput(val categoryId: String?, val minPrice: Double?, val maxPrice: Double?, val inStock: Boolean?)
data class CreateProductInput(val name: String, val description: String?, val price: Double, val currency: String, val stockQuantity: Int, val categoryId: String)
data class UpdateProductInput(val name: String?, val description: String?, val price: Double?, val stockQuantity: Int?)
data class PlaceOrderInput(val items: List<OrderItemInput>)
data class OrderItemInput(val productId: String, val quantity: Int)

fun Product.toDto(): ProductDto = ProductDto(id.toString(), name, null, price.amount.toDouble(), price.currency, stockQuantity, "", emptyList(), isAvailable = stockQuantity > 0, createdAt = "")
fun Category.toDto(): CategoryDto = CategoryDto("", "")
fun Order.toDto(): OrderDto = OrderDto(id.value, "PENDING", totalAmount.amount.toDouble(), "")
fun ProductFilterInput.toDomain(): ProductFilter = ProductFilter()
fun CreateProductInput.toCommand(): CreateProductRequest = CreateProductRequest(name, price, stockQuantity, categoryId)
fun UpdateProductInput.toCommand(): UpdateProductRequest = UpdateProductRequest(name, price)
```

---

## DataLoader สำหรับ N+1 Problem

```kotlin
// DataLoader: batch-load related entities

@org.springframework.stereotype.Component
class ProductDataLoaderConfig(
    private val categoryService: CategoryService,
    private val reviewService: ReviewService,
    private val sellerService: SellerService
) {
    
    @org.springframework.graphql.execution.BatchLoaderRegistry.RegisteredBatchLoader
    fun categoryBatchLoader(): org.springframework.graphql.execution.BatchLoaderRegistry.MappedBatchLoader<String, CategoryDto> {
        return org.springframework.graphql.execution.BatchLoaderRegistry.MappedBatchLoader { categoryIds ->
            // Single DB call for all categories
            val categories = categoryService.findAllByIds(categoryIds)
            reactor.core.publisher.Mono.just(
                categories.associate { it.id to it.toDto() }
            )
        }
    }
    
    @org.springframework.graphql.execution.BatchLoaderRegistry.RegisteredBatchLoader
    fun reviewBatchLoader(): org.springframework.graphql.execution.BatchLoaderRegistry.BatchLoader<String, List<ReviewDto>> {
        return org.springframework.graphql.execution.BatchLoaderRegistry.BatchLoader { productIds ->
            val reviews = reviewService.findByProductIds(productIds)
            reactor.core.publisher.Mono.just(
                productIds.map { productId ->
                    reviews.filter { it.productId == productId }
                        .map { it.toDto() }
                }
            )
        }
    }
}

// Controller using DataLoader
@Controller
class ProductWithDataLoaderController(
    private val productService: ProductService
) {
    
    @SchemaMapping(typeName = "Product", field = "category")
    fun getCategory(
        product: ProductDto,
        dataLoader: org.dataloader.DataLoader<String, CategoryDto>
    ): java.util.concurrent.CompletableFuture<CategoryDto?> {
        return dataLoader.load(product.categoryId)
    }
    
    @SchemaMapping(typeName = "Product", field = "reviews")
    fun getReviews(
        product: ProductDto,
        @Argument limit: Int,
        dataLoader: org.dataloader.DataLoader<String, List<ReviewDto>>
    ): java.util.concurrent.CompletableFuture<List<ReviewDto>> {
        return dataLoader.load(product.id).thenApply { reviews ->
            reviews.take(limit)
        }
    }
}

data class ReviewDto(val productId: String, val id: String, val rating: Int, val comment: String?)
fun Any.toDto(): ReviewDto = ReviewDto("", "", 0, null)
```

---

## gRPC ด้วย Kotlin

```kotlin
// build.gradle.kts (gRPC)
dependencies {
    implementation("io.grpc:grpc-netty-shaded:1.60.0")
    implementation("io.grpc:grpc-protobuf:1.60.0")
    implementation("io.grpc:grpc-stub:1.60.0")
    implementation("io.grpc:grpc-kotlin-stub:1.4.1")
    implementation("com.google.protobuf:protobuf-kotlin:3.25.1")
    testImplementation("io.grpc:grpc-testing:1.60.0")
}

// proto/product.proto
/*
syntax = "proto3";
package com.ecommerce.product.v1;
option java_package = "com.ecommerce.product.v1";

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

service ProductService {
  rpc GetProduct (GetProductRequest) returns (Product);
  rpc ListProducts (ListProductsRequest) returns (ListProductsResponse);
  rpc CreateProduct (CreateProductRequest) returns (Product);
  rpc UpdateProduct (UpdateProductRequest) returns (Product);
  rpc DeleteProduct (DeleteProductRequest) returns (google.protobuf.Empty);
  
  // Server-side streaming: stream inventory updates
  rpc WatchProductStock (WatchStockRequest) returns (stream StockUpdate);
  
  // Client-side streaming: batch create
  rpc BatchCreateProducts (stream CreateProductRequest) returns (BatchCreateResponse);
  
  // Bidirectional streaming: real-time price negotiation
  rpc NegotiatePrice (stream PriceNegotiationRequest) returns (stream PriceNegotiationResponse);
}

message Product {
  string id = 1;
  string name = 2;
  string description = 3;
  double price = 4;
  string currency = 5;
  int32 stock_quantity = 6;
  string category_id = 7;
  repeated string image_urls = 8;
  google.protobuf.Timestamp created_at = 9;
}

message GetProductRequest {
  string id = 1;
}

message ListProductsRequest {
  string category_id = 1;  // optional filter
  int32 page = 2;
  int32 size = 3;
}

message ListProductsResponse {
  repeated Product products = 1;
  int64 total_count = 2;
  bool has_next_page = 3;
}

message CreateProductRequest {
  string name = 1;
  string description = 2;
  double price = 3;
  int32 stock_quantity = 4;
  string category_id = 5;
}

message UpdateProductRequest {
  string id = 1;
  optional string name = 2;
  optional double price = 3;
  optional int32 stock_quantity = 4;
}

message DeleteProductRequest {
  string id = 1;
}

message WatchStockRequest {
  string product_id = 1;
}

message StockUpdate {
  string product_id = 1;
  int32 new_quantity = 2;
  google.protobuf.Timestamp updated_at = 3;
}

message BatchCreateResponse {
  int32 created_count = 1;
  repeated string created_ids = 2;
  repeated string failed_names = 3;
}

message PriceNegotiationRequest {
  string session_id = 1;
  string product_id = 2;
  double offered_price = 3;
}

message PriceNegotiationResponse {
  string session_id = 1;
  bool accepted = 2;
  double counter_offer = 3;
  string message = 4;
}
*/

// gRPC Service Implementation
import com.ecommerce.product.v1.*
import io.grpc.Status
import io.grpc.StatusException
import kotlinx.coroutines.flow.*

@io.grpc.stub.annotations.GrpcService
class ProductGrpcService(
    private val productService: ProductService
) : ProductServiceCoroutineImplBase() {
    
    override suspend fun getProduct(request: GetProductRequest): Product {
        return productService.getById(request.id)?.toProto()
            ?: throw StatusException(
                Status.NOT_FOUND.withDescription("Product ${request.id} not found")
            )
    }
    
    override suspend fun listProducts(request: ListProductsRequest): ListProductsResponse {
        val result = productService.getProducts(
            filter = if (request.categoryId.isNotBlank()) 
                ProductFilter(categoryId = request.categoryId)
            else null,
            page = request.page,
            size = request.size
        )
        
        return listProductsResponse {
            products.addAll(result.items.map { it.toProto() })
            totalCount = result.total
            hasNextPage = result.hasMore
        }
    }
    
    override suspend fun createProduct(request: CreateProductRequest): Product {
        val product = productService.create(
            CreateProductRequest(
                name = request.name,
                price = request.price,
                stockQuantity = request.stockQuantity,
                categoryId = request.categoryId
            )
        )
        return product.toProto()
    }
    
    // Server-side streaming: continuously send stock updates
    override fun watchProductStock(request: WatchStockRequest): Flow<StockUpdate> {
        return productService.observeStockChanges(request.productId)
            .map { update ->
                stockUpdate {
                    productId = update.productId
                    newQuantity = update.newQuantity
                    updatedAt = com.google.protobuf.timestamp {
                        seconds = update.updatedAt.epochSecond
                        nanos = update.updatedAt.nano
                    }
                }
            }
    }
    
    // Client-side streaming: receive batch and process
    override suspend fun batchCreateProducts(requests: Flow<CreateProductRequest>): BatchCreateResponse {
        val createdIds = mutableListOf<String>()
        val failedNames = mutableListOf<String>()
        
        requests.collect { request ->
            try {
                val product = productService.create(
                    com.ecommerce.CreateProductRequest(request.name, request.price, request.stockQuantity, request.categoryId)
                )
                createdIds.add(product.id.toString())
            } catch (e: Exception) {
                failedNames.add(request.name)
            }
        }
        
        return batchCreateResponse {
            createdCount = createdIds.size
            this.createdIds.addAll(createdIds)
            this.failedNames.addAll(failedNames)
        }
    }
    
    // Bidirectional streaming: negotiate price interactively
    override fun negotiatePrice(requests: Flow<PriceNegotiationRequest>): Flow<PriceNegotiationResponse> {
        return flow {
            requests.collect { request ->
                val product = productService.getById(request.productId)
                    ?: run {
                        emit(priceNegotiationResponse {
                            sessionId = request.sessionId
                            accepted = false
                            message = "Product not found"
                        })
                        return@collect
                    }
                
                val minAcceptablePrice = product.price.amount.toDouble() * 0.85  // 15% max discount
                
                val response = if (request.offeredPrice >= minAcceptablePrice) {
                    priceNegotiationResponse {
                        sessionId = request.sessionId
                        accepted = true
                        counterOffer = request.offeredPrice
                        message = "Price accepted!"
                    }
                } else {
                    priceNegotiationResponse {
                        sessionId = request.sessionId
                        accepted = false
                        counterOffer = minAcceptablePrice
                        message = "Minimum price is ${minAcceptablePrice}"
                    }
                }
                
                emit(response)
            }
        }
    }
}

// gRPC interceptors (authentication, logging)
@io.grpc.stub.annotations.GrpcGlobalServerInterceptor
class AuthInterceptor(private val jwtTokenService: JwtTokenService) : io.grpc.ServerInterceptor {
    
    override fun <ReqT, RespT> interceptCall(
        call: io.grpc.ServerCall<ReqT, RespT>,
        headers: io.grpc.Metadata,
        next: io.grpc.ServerCallHandler<ReqT, RespT>
    ): io.grpc.ServerCall.Listener<ReqT> {
        val token = headers.get(io.grpc.Metadata.Key.of("authorization", io.grpc.Metadata.ASCII_STRING_MARSHALLER))
            ?.removePrefix("Bearer ")
        
        if (token == null) {
            call.close(Status.UNAUTHENTICATED.withDescription("Authorization required"), headers)
            return object : io.grpc.ServerCall.Listener<ReqT>() {}
        }
        
        // Verify token...
        return next.startCall(call, headers)
    }
}

// Helper extensions
fun com.ecommerce.domain.model.Product.toProto(): Product = product {
    id = this@toProto.id.value
    name = this@toProto.name
    price = this@toProto.price.amount.toDouble()
    currency = this@toProto.price.currency
    stockQuantity = this@toProto.stockQuantity
}

data class StockChangeEvent(val productId: String, val newQuantity: Int, val updatedAt: java.time.Instant)

fun ProductService.observeStockChanges(productId: String): kotlinx.coroutines.flow.Flow<StockChangeEvent> = 
    kotlinx.coroutines.flow.emptyFlow()
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง GraphQL Subscription สำหรับ Live Chat Support

// Requirements:
// 1. Mutation: sendMessage(sessionId, content) → Message
// 2. Subscription: messageReceived(sessionId) → Message stream
// 3. รองรับ multiple sessions พร้อมกัน
// 4. Message history ดึงได้ผ่าน Query: chatHistory(sessionId, limit)

// schema additions:
/*
type Message {
  id: ID!
  sessionId: ID!
  content: String!
  fromUser: Boolean!
  createdAt: DateTime!
}

type Query {
  chatHistory(sessionId: ID!, limit: Int = 50): [Message!]!
}

type Mutation {
  sendMessage(sessionId: ID!, content: String!): Message!
}

type Subscription {
  messageReceived(sessionId: ID!): Message!
}
*/

@Controller
class ChatGraphQLController(
    private val chatService: ChatService
) {
    
    @QueryMapping
    suspend fun chatHistory(
        @Argument sessionId: String,
        @Argument limit: Int
    ): List<MessageDto> {
        return chatService.getHistory(sessionId, limit)
    }
    
    @MutationMapping
    @org.springframework.security.access.prepost.PreAuthorize("isAuthenticated()")
    suspend fun sendMessage(
        @Argument sessionId: String,
        @Argument content: String,
        authentication: org.springframework.security.core.Authentication
    ): MessageDto {
        return chatService.sendMessage(sessionId, content, fromUser = true)
    }
    
    @SubscriptionMapping
    fun messageReceived(@Argument sessionId: String): reactor.core.publisher.Flux<MessageDto> {
        return chatService.subscribe(sessionId)
    }
}

interface ChatService {
    suspend fun getHistory(sessionId: String, limit: Int): List<MessageDto>
    suspend fun sendMessage(sessionId: String, content: String, fromUser: Boolean): MessageDto
    fun subscribe(sessionId: String): reactor.core.publisher.Flux<MessageDto>
}

data class MessageDto(val id: String, val sessionId: String, val content: String, val fromUser: Boolean, val createdAt: String)

@org.springframework.stereotype.Service
class ChatServiceImpl(
    private val messageRepository: MessageRepository,
    private val llmService: LlmService
) : ChatService {
    
    private val sinks = java.util.concurrent.ConcurrentHashMap<String, reactor.core.publisher.Sinks.Many<MessageDto>>()
    
    override suspend fun getHistory(sessionId: String, limit: Int): List<MessageDto> {
        return messageRepository.findBySessionId(sessionId, limit)
    }
    
    override suspend fun sendMessage(sessionId: String, content: String, fromUser: Boolean): MessageDto {
        val message = MessageDto(
            id = java.util.UUID.randomUUID().toString(),
            sessionId = sessionId,
            content = content,
            fromUser = fromUser,
            createdAt = java.time.Instant.now().toString()
        )
        
        messageRepository.save(message)
        
        // Broadcast to subscribers
        sinks[sessionId]?.tryEmitNext(message)
        
        // Auto-reply from bot if from user
        if (fromUser) {
            kotlinx.coroutines.GlobalScope.launch {
                val botReply = llmService.chat(
                    systemPrompt = "You are a helpful customer support agent. Reply in Thai.",
                    userMessage = content
                )
                sendMessage(sessionId, botReply, fromUser = false)
            }
        }
        
        return message
    }
    
    override fun subscribe(sessionId: String): reactor.core.publisher.Flux<MessageDto> {
        val sink = sinks.getOrPut(sessionId) {
            reactor.core.publisher.Sinks.many().multicast().onBackpressureBuffer()
        }
        return sink.asFlux()
    }
}

interface MessageRepository {
    fun findBySessionId(sessionId: String, limit: Int): List<MessageDto>
    fun save(message: MessageDto)
}
```

---

## สรุป Part 89

```
✅ GraphQL: single endpoint, client-controlled queries
✅ schema.graphqls: types, queries, mutations, subscriptions, scalars
✅ @QueryMapping, @MutationMapping: map to service methods
✅ @SchemaMapping(typeName): field-level resolvers
✅ @Argument: bind GraphQL arguments
✅ @PreAuthorize: authentication/authorization on mutations
✅ ProductPageDto: paginated response with hasNextPage
✅ DataLoader: batch-load categories/reviews in single DB query
✅ MappedBatchLoader: return Map<K, V> for entity lookups
✅ BatchLoader: return List<List<V>> for one-to-many
✅ Subscription: Flow<T> → WebSocket streaming
✅ reactor.core.publisher.Sinks.Many: hot publisher for events
✅ gRPC: Protocol Buffers, HTTP/2, strongly typed
✅ proto file: service, rpc definitions, message types
✅ ProductServiceCoroutineImplBase: Kotlin coroutine gRPC stubs
✅ Status.NOT_FOUND: proper gRPC status codes
✅ Server streaming: watchProductStock returns Flow<StockUpdate>
✅ Client streaming: batchCreateProducts receives Flow<Request>
✅ Bidirectional streaming: negotiatePrice Flow→Flow
✅ GrpcGlobalServerInterceptor: auth, logging middleware
✅ Proto DSL: product { id = ...; name = ... }
✅ Chat subscription: Sinks.Many multicast Flux
✅ Auto-reply bot: coroutine launch for LLM response
```

---

*Part 89/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
