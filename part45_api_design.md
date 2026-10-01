# Part 45: API Design Best Practices

## สารบัญ
1. [RESTful API Design Principles](#restful-api-design-principles)
2. [API Versioning](#api-versioning)
3. [Pagination และ Filtering](#pagination-และ-filtering)
4. [Error Handling Standards](#error-handling-standards)
5. [OpenAPI Documentation](#openapi-documentation)
6. [API Gateway Pattern](#api-gateway-pattern)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## RESTful API Design Principles

```kotlin
// ✅ GOOD: Resource-based URLs, proper HTTP methods

// Resources (nouns, not verbs):
// GET    /users              - list users
// POST   /users              - create user
// GET    /users/{id}         - get user
// PUT    /users/{id}         - full update
// PATCH  /users/{id}         - partial update
// DELETE /users/{id}         - delete user

// Nested resources:
// GET    /users/{id}/orders  - orders of a user
// POST   /users/{id}/orders  - create order for user

// Actions (when CRUD doesn't fit):
// POST   /orders/{id}/cancel  - cancel order
// POST   /users/{id}/activate - activate user

// ❌ BAD: Verb-based URLs
// GET    /getUsers
// POST   /createUser
// GET    /getUserById?id=1

@RestController
@RequestMapping("/api/v1")
class UserController(private val userService: UserApiService) {
    
    @GetMapping("/users")
    fun listUsers(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20", name = "per_page") perPage: Int,
        @RequestParam(required = false) role: String?,
        @RequestParam(required = false) search: String?,
        @RequestParam(defaultValue = "created_at") sortBy: String,
        @RequestParam(defaultValue = "desc") order: String
    ): ResponseEntity<PagedResponse<UserDto>> {
        val filter = UserFilter(role = role, search = search)
        val sort = SortConfig(field = sortBy, direction = order)
        val result = userService.findAll(page, perPage, filter, sort)
        
        return ResponseEntity.ok()
            .header("X-Total-Count", result.total.toString())
            .header("X-Page", page.toString())
            .header("X-Per-Page", perPage.toString())
            .header("Link", buildLinkHeader(result, "/api/v1/users", page, perPage))
            .body(result)
    }
    
    @GetMapping("/users/{id}")
    fun getUser(@PathVariable id: String): UserDto {
        return userService.findById(id)
            ?: throw ResourceNotFoundException("users", id)
    }
    
    @PostMapping("/users")
    fun createUser(
        @Valid @RequestBody request: CreateUserRequest
    ): ResponseEntity<UserDto> {
        val user = userService.create(request)
        val location = URI.create("/api/v1/users/${user.id}")
        return ResponseEntity.created(location).body(user)
    }
    
    @PatchMapping("/users/{id}")
    fun patchUser(
        @PathVariable id: String,
        @RequestBody patch: Map<String, Any?>  // JSON Merge Patch
    ): UserDto {
        return userService.patch(id, patch)
    }
    
    @DeleteMapping("/users/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    fun deleteUser(@PathVariable id: String) {
        userService.delete(id)
    }
    
    @PostMapping("/users/{id}/deactivate")
    fun deactivateUser(@PathVariable id: String): UserDto {
        return userService.deactivate(id)
    }
    
    private fun buildLinkHeader(
        result: PagedResponse<*>,
        baseUrl: String,
        currentPage: Int,
        perPage: Int
    ): String {
        val links = mutableListOf<String>()
        if (currentPage > 0) links.add("<$baseUrl?page=${currentPage-1}&per_page=$perPage>; rel=\"prev\"")
        if (result.hasNextPage) links.add("<$baseUrl?page=${currentPage+1}&per_page=$perPage>; rel=\"next\"")
        links.add("<$baseUrl?page=0&per_page=$perPage>; rel=\"first\"")
        links.add("<$baseUrl?page=${result.lastPage}&per_page=$perPage>; rel=\"last\"")
        return links.joinToString(", ")
    }
}
```

---

## API Versioning

```kotlin
// Strategy 1: URL versioning (most common)
// GET /api/v1/users -> old endpoint
// GET /api/v2/users -> new endpoint

// Strategy 2: Header versioning
// GET /api/users + Header: Accept: application/vnd.myapp.v2+json

// Strategy 3: Query param
// GET /api/users?version=2

// Implementation with URL versioning
@Configuration
class ApiVersioningConfig {
    
    @Bean
    fun requestMappingHandlerMapping(): RequestMappingHandlerMapping {
        return RequestMappingHandlerMapping()
    }
}

// V1 API
@RestController
@RequestMapping("/api/v1/users")
class UserControllerV1(private val userServiceV1: UserServiceV1) {
    
    @GetMapping("/{id}")
    fun getUser(@PathVariable id: String): UserDtoV1 {
        return userServiceV1.findById(id)
    }
}

// V2 API: added fields, changed structure
@RestController
@RequestMapping("/api/v2/users")
class UserControllerV2(private val userServiceV2: UserServiceV2) {
    
    @GetMapping("/{id}")
    fun getUser(@PathVariable id: String): UserDtoV2 {
        return userServiceV2.findById(id)
    }
}

// DTOs showing versioned changes
data class UserDtoV1(
    val id: String,
    val name: String,  // full name as one field
    val email: String
)

data class UserDtoV2(
    val id: String,
    val firstName: String,  // split into first/last
    val lastName: String,
    val email: String,
    val roles: List<String>,  // new field
    val createdAt: String,    // new field
    val _links: HateoasLinks? = null  // HATEOAS
)

data class HateoasLinks(
    val self: Link,
    val orders: Link? = null,
    val profile: Link? = null
)

data class Link(val href: String, val method: String = "GET")
```

---

## Pagination และ Filtering

```kotlin
// Cursor-based pagination (better for large datasets)
data class CursorPage<T>(
    val data: List<T>,
    val nextCursor: String?,
    val prevCursor: String?,
    val hasMore: Boolean
)

// Offset-based pagination
data class PagedResponse<T>(
    val data: List<T>,
    val page: Int,
    val perPage: Int,
    val total: Long,
    val hasNextPage: Boolean = false,
    val lastPage: Int = 0
)

// Filter DSL
data class UserFilter(
    val role: String? = null,
    val search: String? = null,
    val active: Boolean? = null,
    val createdAfter: String? = null,
    val createdBefore: String? = null
)

data class SortConfig(
    val field: String = "created_at",
    val direction: String = "desc"
) {
    val isAscending get() = direction.lowercase() == "asc"
    
    fun toSort(): Sort = if (isAscending) {
        Sort.by(Sort.Direction.ASC, field)
    } else {
        Sort.by(Sort.Direction.DESC, field)
    }
    
    companion object {
        val ALLOWED_FIELDS = setOf("created_at", "updated_at", "name", "email")
        
        fun validated(field: String, direction: String): SortConfig {
            require(field in ALLOWED_FIELDS) { "Invalid sort field: $field" }
            require(direction.lowercase() in setOf("asc", "desc")) { "Invalid sort direction" }
            return SortConfig(field, direction)
        }
    }
}

// Repository with dynamic filtering
@Repository
interface UserRepository : JpaRepository<UserEntity, String> {
    
    fun findAll(spec: Specification<UserEntity>, pageable: Pageable): Page<UserEntity>
}

@Component
class UserSpecifications {
    
    fun build(filter: UserFilter): Specification<UserEntity> {
        return Specification.where(byRole(filter.role))
            .and(bySearch(filter.search))
            .and(byActive(filter.active))
            .and(createdAfter(filter.createdAfter))
            .and(createdBefore(filter.createdBefore))
    }
    
    private fun byRole(role: String?): Specification<UserEntity>? {
        if (role == null) return null
        return Specification { root, _, cb -> cb.equal(root.get<String>("role"), role) }
    }
    
    private fun bySearch(search: String?): Specification<UserEntity>? {
        if (search.isNullOrBlank()) return null
        val pattern = "%${search.lowercase()}%"
        return Specification { root, _, cb ->
            cb.or(
                cb.like(cb.lower(root.get("username")), pattern),
                cb.like(cb.lower(root.get("email")), pattern),
                cb.like(cb.lower(root.get("firstName")), pattern)
            )
        }
    }
    
    private fun byActive(active: Boolean?): Specification<UserEntity>? {
        if (active == null) return null
        return Specification { root, _, cb -> cb.equal(root.get<Boolean>("active"), active) }
    }
    
    private fun createdAfter(date: String?): Specification<UserEntity>? {
        if (date == null) return null
        val instant = java.time.Instant.parse(date)
        return Specification { root, _, cb -> 
            cb.greaterThanOrEqualTo(root.get("createdAt"), instant)
        }
    }
    
    private fun createdBefore(date: String?): Specification<UserEntity>? {
        if (date == null) return null
        val instant = java.time.Instant.parse(date)
        return Specification { root, _, cb -> 
            cb.lessThanOrEqualTo(root.get("createdAt"), instant)
        }
    }
}
```

---

## Error Handling Standards

```kotlin
// RFC 7807: Problem Details for HTTP APIs
data class ProblemDetail(
    val type: String = "about:blank",
    val title: String,
    val status: Int,
    val detail: String? = null,
    val instance: String? = null,
    val extensions: Map<String, Any> = emptyMap()
)

@RestControllerAdvice
class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException::class)
    fun handleNotFound(
        e: ResourceNotFoundException,
        request: HttpServletRequest
    ): ResponseEntity<ProblemDetail> {
        val problem = ProblemDetail(
            type = "https://api.myapp.com/errors/not-found",
            title = "Resource Not Found",
            status = 404,
            detail = e.message,
            instance = request.requestURI
        )
        return ResponseEntity.status(404).body(problem)
    }
    
    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(
        e: MethodArgumentNotValidException,
        request: HttpServletRequest
    ): ResponseEntity<ProblemDetail> {
        val errors = e.bindingResult.fieldErrors
            .groupBy { it.field }
            .mapValues { (_, errors) -> errors.map { it.defaultMessage } }
        
        val problem = ProblemDetail(
            type = "https://api.myapp.com/errors/validation",
            title = "Validation Failed",
            status = 400,
            detail = "One or more fields have validation errors",
            instance = request.requestURI,
            extensions = mapOf("errors" to errors)
        )
        return ResponseEntity.badRequest().body(problem)
    }
    
    @ExceptionHandler(Exception::class)
    fun handleGeneral(
        e: Exception,
        request: HttpServletRequest
    ): ResponseEntity<ProblemDetail> {
        log.error("Unhandled exception at ${request.requestURI}", e)
        val problem = ProblemDetail(
            type = "https://api.myapp.com/errors/internal",
            title = "Internal Server Error",
            status = 500,
            detail = "An unexpected error occurred",
            instance = request.requestURI
        )
        return ResponseEntity.internalServerError().body(problem)
    }
    
    companion object {
        private val log = LoggerFactory.getLogger(GlobalExceptionHandler::class.java)
    }
}

class ResourceNotFoundException(resource: String, id: String) 
    : RuntimeException("$resource not found: $id")
```

---

## OpenAPI Documentation

```kotlin
// build.gradle.kts
// implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0")

// API documentation annotations
@RestController
@RequestMapping("/api/v1/products")
@Tag(name = "Products", description = "Product management API")
class DocumentedProductController {
    
    @Operation(
        summary = "List all products",
        description = "Returns a paginated list of products with optional filtering"
    )
    @ApiResponses(
        ApiResponse(responseCode = "200", description = "Successful operation",
            content = [Content(schema = Schema(implementation = ProductListResponse::class))]),
        ApiResponse(responseCode = "400", description = "Invalid parameters",
            content = [Content(schema = Schema(implementation = ProblemDetail::class))])
    )
    @GetMapping
    fun listProducts(
        @Parameter(description = "Page number (0-based)") @RequestParam(defaultValue = "0") page: Int,
        @Parameter(description = "Items per page") @RequestParam(defaultValue = "20") perPage: Int,
        @Parameter(description = "Filter by category") @RequestParam(required = false) category: String?
    ): ProductListResponse {
        TODO()
    }
    
    @Operation(summary = "Create a new product")
    @ApiResponse(responseCode = "201", description = "Product created",
        headers = [Header(name = "Location", description = "URL of created product")])
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun createProduct(@RequestBody @Valid request: CreateProductApiRequest): ProductDto {
        TODO()
    }
}

@Schema(description = "Product data")
data class ProductDto(
    @Schema(description = "Unique identifier", example = "prod-123") val id: String,
    @Schema(description = "Product name", example = "iPhone 15 Pro") val name: String,
    @Schema(description = "Price in THB", example = "49900.00") val price: Double,
    @Schema(description = "Available stock", example = "100") val stock: Int
)

@Schema(description = "Create product request")
data class CreateProductApiRequest(
    @Schema(required = true, example = "iPhone 15 Pro") 
    @field:NotBlank val name: String,
    
    @Schema(required = true, example = "49900.00") 
    @field:Positive val price: Double,
    
    @Schema(required = true, example = "100") 
    @field:PositiveOrZero val stock: Int
)

data class ProductListResponse(
    val data: List<ProductDto>,
    val total: Long,
    val page: Int,
    val perPage: Int
)

// OpenAPI config
@Configuration
class OpenApiConfig {
    
    @Bean
    fun openApiDocs(): OpenAPI {
        return OpenAPI()
            .info(Info()
                .title("My App API")
                .version("1.0.0")
                .description("Complete API documentation")
                .contact(Contact().name("Team").email("api@myapp.com"))
                .license(License().name("MIT")))
            .addSecurityItem(SecurityRequirement().addList("bearerAuth"))
            .components(Components()
                .addSecuritySchemes("bearerAuth", 
                    SecurityScheme()
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")))
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Design a complete Blog API

// Requirements:
// - POST   /api/v1/posts           create post
// - GET    /api/v1/posts           list posts (filter: author, tag, published)
// - GET    /api/v1/posts/{slug}    get post by slug
// - PATCH  /api/v1/posts/{id}      update post
// - DELETE /api/v1/posts/{id}      delete post
// - POST   /api/v1/posts/{id}/publish  publish post
// - GET    /api/v1/posts/{id}/comments list comments
// - POST   /api/v1/posts/{id}/comments add comment
// - GET    /api/v1/tags             list tags with post count
// - GET    /api/v1/authors/{id}/posts posts by author

// Design your DTOs and request/response objects:

data class PostDto(
    val id: String,
    val slug: String,
    val title: String,
    val excerpt: String,
    val content: String,
    val published: Boolean,
    val authorId: String,
    val authorName: String,
    val tags: List<String>,
    val createdAt: String,
    val updatedAt: String,
    val _links: PostLinks? = null
)

data class PostLinks(
    val self: Link,
    val author: Link,
    val comments: Link
)

data class CreatePostRequest(
    @field:NotBlank val title: String,
    @field:NotBlank val content: String,
    val tags: List<String> = emptyList(),
    val published: Boolean = false
)

data class PostFilter(
    val authorId: String? = null,
    val tag: String? = null,
    val published: Boolean? = null,
    val search: String? = null
)

// TODO: Implement PostController with all endpoints
// TODO: Add proper error handling
// TODO: Add OpenAPI documentation
// TODO: Add unit tests for each endpoint

// Placeholder imports
typealias Sort = org.springframework.data.domain.Sort
typealias Pageable = org.springframework.data.domain.Pageable
typealias Page<T> = org.springframework.data.domain.Page<T>
typealias Specification<T> = org.springframework.data.jpa.domain.Specification<T>
typealias JpaRepository<T, ID> = org.springframework.data.jpa.repository.JpaRepository<T, ID>
typealias HttpServletRequest = javax.servlet.http.HttpServletRequest
typealias URI = java.net.URI

interface UserApiService {
    fun findAll(page: Int, perPage: Int, filter: UserFilter, sort: SortConfig): PagedResponse<UserDto>
    fun findById(id: String): UserDto?
    fun create(request: CreateUserRequest): UserDto
    fun patch(id: String, patch: Map<String, Any?>): UserDto
    fun delete(id: String)
    fun deactivate(id: String): UserDto
}

interface UserServiceV1 { fun findById(id: String): UserDtoV1 }
interface UserServiceV2 { fun findById(id: String): UserDtoV2 }

data class UserDto(val id: String, val name: String, val email: String, val role: String)
data class CreateUserRequest(val name: String, val email: String, val password: String)
data class UserEntity(val id: String, val username: String, val email: String, val firstName: String, val role: String, val active: Boolean, val createdAt: java.time.Instant)

annotation class Tag(val name: String, val description: String)
annotation class Operation(val summary: String, val description: String = "")
annotation class ApiResponses(vararg val value: ApiResponse)
annotation class ApiResponse(val responseCode: String, val description: String, val content: Array<Content> = [], val headers: Array<Header> = [])
annotation class Content(val schema: Schema = Schema(implementation = Any::class))
annotation class Parameter(val description: String)
annotation class Header(val name: String, val description: String)
annotation class Schema(val description: String = "", val example: String = "", val required: Boolean = false, val implementation: kotlin.reflect.KClass<*> = Any::class)
annotation class SecurityRequirement(val name: String = "")
```

---

## สรุป Part 45

```
✅ Resource-based URLs: /users, /users/{id}/orders
✅ HTTP methods: GET/POST/PUT/PATCH/DELETE ตามความหมาย
✅ Status codes: 201 Created, 204 No Content, 404 Not Found
✅ API versioning: URL path strategy (/v1/, /v2/)
✅ Pagination: page/per_page, Link headers, cursor-based
✅ Filtering: query params, Specification pattern
✅ Sorting: sortBy, order params with validation
✅ RFC 7807: Problem Details standard error format
✅ OpenAPI/Swagger: auto-generated documentation
✅ HATEOAS: hypermedia links in responses
✅ Location header: URI of created resource
✅ Content negotiation: Accept header versioning
```

---

*Part 45/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
