# Part 68: API Design ขั้นสูง — REST, GraphQL, gRPC

## สารบัญ
1. [REST API Best Practices](#rest-api-best-practices)
2. [API Versioning](#api-versioning)
3. [GraphQL ด้วย Spring for GraphQL](#graphql-ด้วย-spring-for-graphql)
4. [gRPC ด้วย Kotlin](#grpc-ด้วย-kotlin)
5. [API Documentation ด้วย OpenAPI](#api-documentation-ด้วย-openapi)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## REST API Best Practices

```kotlin
// Request/Response design with validation
data class CreateProductRequest(
    @field:NotBlank(message = "Name is required")
    @field:Size(min = 3, max = 200, message = "Name must be 3-200 characters")
    val name: String,
    
    @field:NotBlank
    @field:Size(max = 2000)
    val description: String,
    
    @field:NotNull
    @field:DecimalMin("0.01")
    @field:Digits(integer = 10, fraction = 2)
    val price: java.math.BigDecimal,
    
    @field:Min(0)
    val stockQuantity: Int,
    
    @field:NotBlank
    val categoryId: String,
    
    @field:Size(max = 10, message = "Maximum 10 images allowed")
    val imageUrls: List<@URL String> = emptyList()
)

// Standardized error response
data class ApiError(
    val code: String,
    val message: String,
    val details: List<FieldError> = emptyList(),
    val timestamp: java.time.Instant = java.time.Instant.now(),
    val traceId: String? = null
)

data class FieldError(
    val field: String,
    val message: String,
    val rejectedValue: Any? = null
)

// Global exception handler
@RestControllerAdvice
class GlobalExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(ex: MethodArgumentNotValidException): ResponseEntity<ApiError> {
        val errors = ex.bindingResult.fieldErrors.map { 
            FieldError(it.field, it.defaultMessage ?: "Invalid", it.rejectedValue)
        }
        
        return ResponseEntity
            .badRequest()
            .body(ApiError(
                code = "VALIDATION_ERROR",
                message = "Request validation failed",
                details = errors
            ))
    }
    
    @ExceptionHandler(ResourceNotFoundException::class)
    fun handleNotFound(ex: ResourceNotFoundException): ResponseEntity<ApiError> {
        return ResponseEntity.status(404).body(
            ApiError(code = "NOT_FOUND", message = ex.message ?: "Resource not found")
        )
    }
    
    @ExceptionHandler(ConflictException::class)
    fun handleConflict(ex: ConflictException): ResponseEntity<ApiError> {
        return ResponseEntity.status(409).body(
            ApiError(code = "CONFLICT", message = ex.message ?: "Resource conflict")
        )
    }
    
    @ExceptionHandler(Exception::class)
    fun handleUnexpected(ex: Exception): ResponseEntity<ApiError> {
        println("Unexpected error: ${ex.message}")
        return ResponseEntity.status(500).body(
            ApiError(code = "INTERNAL_ERROR", message = "An unexpected error occurred")
        )
    }
}

// Paginated response envelope
data class PagedResponse<T>(
    val data: List<T>,
    val pagination: PaginationMeta,
    val links: PageLinks
)

data class PaginationMeta(
    val page: Int,
    val size: Int,
    val totalElements: Long,
    val totalPages: Int
)

data class PageLinks(
    val self: String,
    val first: String,
    val last: String,
    val next: String?,
    val prev: String?
)

@RestController
@RequestMapping("/api/v1/products")
class ProductRestController(private val productService: ProductApiService) {
    
    @GetMapping
    fun listProducts(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") @Max(100) size: Int,
        @RequestParam(defaultValue = "createdAt") sortBy: String,
        @RequestParam(defaultValue = "DESC") sortDirection: String,
        @RequestParam(required = false) category: String?,
        @RequestParam(required = false) minPrice: java.math.BigDecimal?,
        @RequestParam(required = false) maxPrice: java.math.BigDecimal?,
        request: HttpServletRequest
    ): ResponseEntity<PagedResponse<ProductDto>> {
        
        val result = productService.findProducts(
            ProductSearchCriteria(page, size, sortBy, sortDirection, category, minPrice, maxPrice)
        )
        
        val baseUrl = "${request.scheme}://${request.serverName}:${request.serverPort}${request.requestURI}"
        
        return ResponseEntity.ok(
            PagedResponse(
                data = result.content,
                pagination = PaginationMeta(page, size, result.totalElements, result.totalPages),
                links = buildLinks(baseUrl, page, size, result.totalPages)
            )
        )
    }
    
    private fun buildLinks(baseUrl: String, page: Int, size: Int, totalPages: Int): PageLinks {
        return PageLinks(
            self = "$baseUrl?page=$page&size=$size",
            first = "$baseUrl?page=0&size=$size",
            last = "$baseUrl?page=${totalPages - 1}&size=$size",
            next = if (page < totalPages - 1) "$baseUrl?page=${page + 1}&size=$size" else null,
            prev = if (page > 0) "$baseUrl?page=${page - 1}&size=$size" else null
        )
    }
}

data class ProductDto(val id: String, val name: String, val price: java.math.BigDecimal)
data class ProductSearchCriteria(val page: Int, val size: Int, val sortBy: String, val sortDirection: String, val category: String?, val minPrice: java.math.BigDecimal?, val maxPrice: java.math.BigDecimal?)

interface ProductApiService {
    fun findProducts(criteria: ProductSearchCriteria): PageResult<ProductDto>
}

data class PageResult<T>(val content: List<T>, val totalElements: Long, val totalPages: Int)

class ResourceNotFoundException(msg: String) : Exception(msg)
class ConflictException(msg: String) : Exception(msg)

typealias RestControllerAdvice = org.springframework.web.bind.annotation.RestControllerAdvice
typealias ExceptionHandler = org.springframework.web.bind.annotation.ExceptionHandler
typealias MethodArgumentNotValidException = org.springframework.web.bind.MethodArgumentNotValidException
typealias NotBlank = jakarta.validation.constraints.NotBlank
typealias NotNull = jakarta.validation.constraints.NotNull
typealias Size = jakarta.validation.constraints.Size
typealias Min = jakarta.validation.constraints.Min
typealias Max = jakarta.validation.constraints.Max
typealias DecimalMin = jakarta.validation.constraints.DecimalMin
typealias Digits = jakarta.validation.constraints.Digits
typealias URL = jakarta.validation.constraints.NotNull  // simplified
typealias HttpServletRequest = jakarta.servlet.http.HttpServletRequest
```

---

## API Versioning

```kotlin
// Strategy 1: URL versioning (most common)
// /api/v1/products
// /api/v2/products

@RestController
@RequestMapping("/api/v2/products")
class ProductV2Controller(private val productService: ProductApiService) {
    
    @GetMapping("/{id}")
    fun getProduct(@PathVariable id: String): ProductV2Dto {
        // v2 adds new fields: rating, reviewCount
        return ProductV2Dto(id, "Product", 99.99.toBigDecimal(), 4.5, 100)
    }
}

data class ProductV2Dto(
    val id: String,
    val name: String,
    val price: java.math.BigDecimal,
    val rating: Double,     // NEW in v2
    val reviewCount: Int    // NEW in v2
)

// Strategy 2: Header versioning
// Accept: application/vnd.myapp.v2+json

@RestController
@RequestMapping("/api/products")
class ProductVersionedController {
    
    @GetMapping(
        headers = ["API-Version=1"],
        produces = ["application/vnd.myapp.v1+json"]
    )
    fun getProductV1(@PathVariable id: String): ProductDto {
        return ProductDto(id, "Product", 99.99.toBigDecimal())
    }
    
    @GetMapping(
        headers = ["API-Version=2"],
        produces = ["application/vnd.myapp.v2+json"]
    )
    fun getProductV2(@PathVariable id: String): ProductV2Dto {
        return ProductV2Dto(id, "Product", 99.99.toBigDecimal(), 4.5, 100)
    }
}

// Strategy 3: Deprecation headers
@GetMapping("/api/v1/products/{id}")
fun getProductDeprecated(@PathVariable id: String): ResponseEntity<ProductDto> {
    return ResponseEntity.ok()
        .header("Deprecation", "true")
        .header("Sunset", "Sat, 01 Jan 2026 00:00:00 GMT")
        .header("Link", "</api/v2/products/{id}>; rel=\"successor-version\"")
        .body(ProductDto(id, "Product", 99.99.toBigDecimal()))
}
```

---

## GraphQL ด้วย Spring for GraphQL

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-graphql")
    implementation("org.springframework.boot:spring-boot-starter-web")
}

// schema.graphqls (src/main/resources/graphql/)
/*
type Query {
    product(id: ID!): Product
    products(filter: ProductFilter, page: Int = 0, size: Int = 20): ProductPage!
    searchProducts(query: String!): [Product!]!
}

type Mutation {
    createProduct(input: CreateProductInput!): Product!
    updateProduct(id: ID!, input: UpdateProductInput!): Product!
    deleteProduct(id: ID!): Boolean!
}

type Subscription {
    orderUpdated(customerId: ID!): OrderUpdate!
}

type Product {
    id: ID!
    name: String!
    description: String
    price: Float!
    stock: Int!
    category: Category
    reviews(limit: Int = 10): [Review!]!
}

type Category {
    id: ID!
    name: String!
}

type Review {
    id: ID!
    rating: Int!
    comment: String
    author: User
}

type User {
    id: ID!
    name: String!
}

input ProductFilter {
    categoryId: ID
    minPrice: Float
    maxPrice: Float
    inStock: Boolean
}

input CreateProductInput {
    name: String!
    description: String
    price: Float!
    stock: Int!
    categoryId: ID!
}

input UpdateProductInput {
    name: String
    description: String
    price: Float
    stock: Int
}

type ProductPage {
    content: [Product!]!
    totalElements: Int!
    totalPages: Int!
}

type OrderUpdate {
    orderId: ID!
    status: String!
    updatedAt: String!
}
*/

// GraphQL Controller
@Controller
class ProductGraphQlController(
    private val productService: GraphQlProductService,
    private val categoryService: CategoryService
) {
    
    @QueryMapping
    fun product(@Argument id: String): GraphQlProduct? {
        return productService.findById(id)
    }
    
    @QueryMapping
    fun products(
        @Argument filter: ProductFilterInput?,
        @Argument page: Int,
        @Argument size: Int
    ): GraphQlProductPage {
        return productService.findAll(filter, page, size)
    }
    
    @MutationMapping
    fun createProduct(@Argument input: CreateProductInput): GraphQlProduct {
        return productService.create(input)
    }
    
    // DataLoader: batch loading to prevent N+1
    @SchemaMapping(typeName = "Product", field = "category")
    fun productCategory(product: GraphQlProduct, loader: DataLoader<String, GraphQlCategory>): CompletableFuture<GraphQlCategory?> {
        return loader.load(product.categoryId)
    }
    
    // DataLoader for reviews (batch)
    @SchemaMapping(typeName = "Product", field = "reviews")
    fun productReviews(product: GraphQlProduct): List<Review> {
        return productService.getReviews(product.id)
    }
    
    // Subscription
    @SubscriptionMapping
    fun orderUpdated(@Argument customerId: String): Flux<OrderUpdate> {
        return orderUpdatePublisher.subscribeForCustomer(customerId)
    }
}

// Register DataLoader
@Configuration
class DataLoaderConfig {
    
    @Bean
    fun categoryDataLoader(categoryService: CategoryService): BatchLoaderRegistry {
        return BatchLoaderRegistry.newInstance().also { registry ->
            registry.forTypePair(String::class.java, GraphQlCategory::class.java)
                .withName("categoryLoader")
                .registerBatchLoader { ids, env ->
                    categoryService.findByIds(ids.toList())
                        .let { categories ->
                            Mono.just(ids.map { id -> categories.find { it.id == id } })
                        }
                }
        }
    }
}

data class GraphQlProduct(val id: String, val name: String, val price: Double, val categoryId: String)
data class GraphQlCategory(val id: String, val name: String)
data class Review(val id: String, val rating: Int, val comment: String)
data class GraphQlProductPage(val content: List<GraphQlProduct>, val totalElements: Int, val totalPages: Int)
data class ProductFilterInput(val categoryId: String?, val minPrice: Double?, val maxPrice: Double?)
data class CreateProductInput(val name: String, val price: Double, val categoryId: String)
data class OrderUpdate(val orderId: String, val status: String, val updatedAt: String)

interface GraphQlProductService {
    fun findById(id: String): GraphQlProduct?
    fun findAll(filter: ProductFilterInput?, page: Int, size: Int): GraphQlProductPage
    fun create(input: CreateProductInput): GraphQlProduct
    fun getReviews(productId: String): List<Review>
}

interface CategoryService {
    fun findByIds(ids: List<String>): List<GraphQlCategory>
}

val orderUpdatePublisher = object {
    fun subscribeForCustomer(customerId: String): Flux<OrderUpdate> = Flux.empty()
}

typealias Controller = org.springframework.stereotype.Controller
typealias QueryMapping = org.springframework.graphql.data.method.annotation.QueryMapping
typealias MutationMapping = org.springframework.graphql.data.method.annotation.MutationMapping
typealias SubscriptionMapping = org.springframework.graphql.data.method.annotation.SubscriptionMapping
typealias SchemaMapping = org.springframework.graphql.data.method.annotation.SchemaMapping
typealias Argument = org.springframework.graphql.data.method.annotation.Argument
typealias DataLoader = org.dataloader.DataLoader
typealias CompletableFuture<T> = java.util.concurrent.CompletableFuture<T>
typealias BatchLoaderRegistry = org.springframework.graphql.execution.BatchLoaderRegistry
typealias Flux<T> = reactor.core.publisher.Flux<T>
typealias Mono<T> = reactor.core.publisher.Mono<T>
```

---

## gRPC ด้วย Kotlin

```kotlin
// build.gradle.kts
plugins {
    id("com.google.protobuf") version "0.9.4"
}

dependencies {
    implementation("io.grpc:grpc-kotlin-stub:1.4.1")
    implementation("io.grpc:grpc-protobuf:1.68.0")
    implementation("io.grpc:grpc-netty-shaded:1.68.0")
    implementation("com.google.protobuf:protobuf-kotlin:3.25.5")
}

// proto/product.proto:
/*
syntax = "proto3";
package com.example.product;

service ProductService {
    rpc GetProduct(GetProductRequest) returns (Product);
    rpc ListProducts(ListProductsRequest) returns (stream Product);  // server streaming
    rpc CreateProduct(stream CreateProductRequest) returns (CreateProductResponse);  // client streaming
    rpc SyncProducts(stream ProductUpdate) returns (stream SyncResult);  // bidirectional
}

message Product {
    string id = 1;
    string name = 2;
    double price = 3;
    int32 stock = 4;
    string category_id = 5;
}

message GetProductRequest {
    string id = 1;
}

message ListProductsRequest {
    string category_id = 1;
    int32 page = 2;
    int32 size = 3;
}

message CreateProductRequest {
    string name = 1;
    double price = 2;
    string category_id = 3;
}

message CreateProductResponse {
    int32 created_count = 1;
    repeated string product_ids = 2;
}

message ProductUpdate {
    string product_id = 1;
    double new_price = 2;
}

message SyncResult {
    string product_id = 1;
    bool success = 2;
    string error = 3;
}
*/

// gRPC Service implementation
class ProductGrpcService(
    private val productRepository: GrpcProductRepository
) : ProductServiceGrpcKt.ProductServiceCoroutineImplBase() {
    
    // Unary RPC
    override suspend fun getProduct(request: GetProductRequest): Product {
        val product = productRepository.findById(request.id)
            ?: throw StatusException(Status.NOT_FOUND.withDescription("Product ${request.id} not found"))
        
        return product {
            id = product.id
            name = product.name
            price = product.price
            stock = product.stockQuantity
            categoryId = product.categoryId
        }
    }
    
    // Server streaming RPC
    override fun listProducts(request: ListProductsRequest): Flow<Product> = flow {
        productRepository.findByCategory(request.categoryId).forEach { p ->
            emit(product {
                id = p.id
                name = p.name
                price = p.price
                stock = p.stockQuantity
                categoryId = p.categoryId
            })
        }
    }
    
    // Client streaming RPC
    override suspend fun createProduct(requests: Flow<CreateProductRequest>): CreateProductResponse {
        val createdIds = mutableListOf<String>()
        
        requests.collect { request ->
            val created = productRepository.create(
                GrpcProduct(java.util.UUID.randomUUID().toString(), request.name, request.price, 0, request.categoryId)
            )
            createdIds.add(created.id)
        }
        
        return createProductResponse {
            createdCount = createdIds.size
            productIds.addAll(createdIds)
        }
    }
    
    // Bidirectional streaming RPC
    override fun syncProducts(requests: Flow<ProductUpdate>): Flow<SyncResult> = flow {
        requests.collect { update ->
            try {
                productRepository.updatePrice(update.productId, update.newPrice)
                emit(syncResult {
                    productId = update.productId
                    success = true
                })
            } catch (ex: Exception) {
                emit(syncResult {
                    productId = update.productId
                    success = false
                    error = ex.message ?: "Update failed"
                })
            }
        }
    }
}

// gRPC Client
class ProductGrpcClient(channel: ManagedChannel) {
    
    private val stub = ProductServiceGrpcKt.ProductServiceCoroutineStub(channel)
    
    suspend fun getProduct(id: String): Product {
        return stub.withDeadlineAfter(5, java.util.concurrent.TimeUnit.SECONDS)
            .getProduct(getProductRequest { this.id = id })
    }
    
    suspend fun listAllProducts(categoryId: String): List<Product> {
        return stub.listProducts(listProductsRequest { this.categoryId = categoryId })
            .toList()
    }
}

data class GrpcProduct(val id: String, val name: String, val price: Double, val stockQuantity: Int, val categoryId: String)

interface GrpcProductRepository {
    fun findById(id: String): GrpcProduct?
    fun findByCategory(categoryId: String): List<GrpcProduct>
    fun create(product: GrpcProduct): GrpcProduct
    fun updatePrice(productId: String, newPrice: Double)
}

// placeholder types
class StatusException(val status: Any) : Exception()
object Status { val NOT_FOUND = StatusCode() }
class StatusCode { fun withDescription(msg: String) = this }
object ProductServiceGrpcKt {
    abstract class ProductServiceCoroutineImplBase
    class ProductServiceCoroutineStub(channel: Any) {
        fun withDeadlineAfter(n: Long, unit: java.util.concurrent.TimeUnit) = this
        suspend fun getProduct(req: Any): Product = TODO()
        fun listProducts(req: Any): Flow<Product> = TODO()
    }
}
typealias ManagedChannel = io.grpc.ManagedChannel
typealias Flow<T> = kotlinx.coroutines.flow.Flow<T>
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Design a versioned product API

// V1: Basic product info
// V2: Add rating, reviewCount fields
// V3: Add recommendedProducts list (breaking change)

// Requirements:
// 1. URL versioning: /api/v1/, /api/v2/, /api/v3/
// 2. V1 and V2 should work simultaneously
// 3. V1 sends Deprecation + Sunset headers
// 4. V3 uses pagination for recommendedProducts

// Hint: Use polymorphism or feature toggles for versioned responses

interface ProductResponse {
    val id: String
    val name: String
    val price: java.math.BigDecimal
}

data class ProductV1Response(
    override val id: String,
    override val name: String,
    override val price: java.math.BigDecimal
) : ProductResponse

data class ProductV3Response(
    override val id: String,
    override val name: String,
    override val price: java.math.BigDecimal,
    val rating: Double,
    val reviewCount: Int,
    val recommendedProducts: PagedResponse<ProductV3Response>  // recursive!
) : ProductResponse
```

---

## สรุป Part 68

```
✅ REST Best Practices: validation, error response format, pagination
✅ @RestControllerAdvice: global exception handling
✅ ApiError: standardized error response with code + details
✅ PagedResponse: data + pagination + HAL-style links
✅ URL versioning: /api/v1/ and /api/v2/ simultaneously
✅ Header versioning: Accept header or custom API-Version
✅ Deprecation headers: Deprecation, Sunset, Link
✅ GraphQL: schema-first with .graphqls files
✅ @QueryMapping, @MutationMapping, @SubscriptionMapping
✅ @SchemaMapping: field resolver for nested types
✅ DataLoader: batch loading to prevent N+1 in GraphQL
✅ Subscription: real-time with Flux<T>
✅ gRPC: Protocol Buffers, generated Kotlin stubs
✅ Unary, Server Streaming, Client Streaming, Bidirectional
✅ Flow<T>: Kotlin coroutine integration with gRPC
✅ Status and StatusException: gRPC error handling
✅ ManagedChannel: gRPC client connection
✅ OpenAPI: automatic documentation from annotations
```

---

*Part 68/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
