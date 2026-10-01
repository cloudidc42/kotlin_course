# Part 28: Spring Boot กับ Kotlin

## สารบัญ
1. [Spring Boot Setup](#spring-boot-setup)
2. [REST Controllers](#rest-controllers)
3. [Services และ Repositories](#services-และ-repositories)
4. [Spring Data JPA](#spring-data-jpa)
5. [Security กับ JWT](#security-กับ-jwt)
6. [Testing Spring Boot](#testing-spring-boot)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Spring Boot Setup

```kotlin
// build.gradle.kts
plugins {
    kotlin("jvm") version "1.9.22"
    kotlin("plugin.spring") version "1.9.22"  // open classes for Spring proxies
    kotlin("plugin.jpa") version "1.9.22"     // no-arg constructors for JPA
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    
    implementation("io.jsonwebtoken:jjwt-api:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.3")
    runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.3")
    
    runtimeOnly("org.postgresql:postgresql")
    
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
}

// allOpen for Spring
allOpen {
    annotation("javax.persistence.Entity")
    annotation("javax.persistence.MappedSuperclass")
    annotation("javax.persistence.Embeddable")
}

// noArg for JPA entities
noArg {
    annotation("javax.persistence.Entity")
}
```

```kotlin
// Main application
@SpringBootApplication
class KotlinSpringApp

fun main(args: Array<String>) {
    runApplication<KotlinSpringApp>(*args)
}
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/myapp
    username: postgres
    password: secret
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true

server:
  port: 8080

app:
  jwt:
    secret: mySecretKey123
    expiration: 86400000  # 24 hours
```

---

## REST Controllers

```kotlin
import org.springframework.web.bind.annotation.*
import org.springframework.http.*
import org.springframework.validation.annotation.Validated
import javax.validation.constraints.*

// DTOs
data class CreateProductRequest(
    @field:NotBlank(message = "Name is required")
    @field:Size(min = 2, max = 100)
    val name: String,
    
    @field:NotBlank
    val description: String,
    
    @field:Positive(message = "Price must be positive")
    val price: Double,
    
    @field:Min(0)
    val stock: Int = 0,
    
    val categoryId: Long
)

data class UpdateProductRequest(
    val name: String?,
    val description: String?,
    @field:Positive
    val price: Double?,
    @field:Min(0)
    val stock: Int?
)

data class ProductResponse(
    val id: Long,
    val name: String,
    val description: String,
    val price: Double,
    val stock: Int,
    val categoryName: String?,
    val createdAt: java.time.Instant
)

data class PagedResponse<T>(
    val data: List<T>,
    val page: Int,
    val pageSize: Int,
    val total: Long,
    val totalPages: Int
)

data class ApiError(
    val message: String,
    val code: String,
    val details: Map<String, String>? = null
)

@RestController
@RequestMapping("/api/v1/products")
@Validated
class ProductController(private val productService: ProductService) {
    
    @GetMapping
    fun getAllProducts(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int,
        @RequestParam(required = false) category: String?,
        @RequestParam(required = false) search: String?,
        @RequestParam(defaultValue = "id") sortBy: String,
        @RequestParam(defaultValue = "asc") sortDir: String
    ): ResponseEntity<PagedResponse<ProductResponse>> {
        val products = productService.findAll(page, size, category, search, sortBy, sortDir)
        return ResponseEntity.ok(products)
    }
    
    @GetMapping("/{id}")
    fun getProduct(@PathVariable id: Long): ResponseEntity<ProductResponse> {
        val product = productService.findById(id)
            ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(product)
    }
    
    @PostMapping
    fun createProduct(
        @RequestBody @Validated request: CreateProductRequest
    ): ResponseEntity<ProductResponse> {
        val product = productService.create(request)
        val location = java.net.URI.create("/api/v1/products/${product.id}")
        return ResponseEntity.created(location).body(product)
    }
    
    @PutMapping("/{id}")
    fun updateProduct(
        @PathVariable id: Long,
        @RequestBody @Validated request: UpdateProductRequest
    ): ResponseEntity<ProductResponse> {
        val product = productService.update(id, request)
            ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(product)
    }
    
    @DeleteMapping("/{id}")
    fun deleteProduct(@PathVariable id: Long): ResponseEntity<Void> {
        return if (productService.delete(id)) {
            ResponseEntity.noContent().build()
        } else {
            ResponseEntity.notFound().build()
        }
    }
    
    @GetMapping("/category/{category}")
    fun getByCategory(@PathVariable category: String): ResponseEntity<List<ProductResponse>> {
        return ResponseEntity.ok(productService.findByCategory(category))
    }
    
    @PatchMapping("/{id}/stock")
    fun updateStock(
        @PathVariable id: Long,
        @RequestParam @Min(0) quantity: Int
    ): ResponseEntity<ProductResponse> {
        val product = productService.updateStock(id, quantity)
            ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(product)
    }
}

// Global exception handler
@RestControllerAdvice
class GlobalExceptionHandler {
    
    @ExceptionHandler(org.springframework.web.bind.MethodArgumentNotValidException::class)
    fun handleValidation(ex: org.springframework.web.bind.MethodArgumentNotValidException): ResponseEntity<ApiError> {
        val details = ex.bindingResult.fieldErrors.associate { 
            it.field to (it.defaultMessage ?: "Invalid value") 
        }
        return ResponseEntity.badRequest().body(
            ApiError("Validation failed", "VALIDATION_ERROR", details)
        )
    }
    
    @ExceptionHandler(NoSuchElementException::class)
    fun handleNotFound(ex: NoSuchElementException): ResponseEntity<ApiError> {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(
            ApiError(ex.message ?: "Not found", "NOT_FOUND")
        )
    }
    
    @ExceptionHandler(Exception::class)
    fun handleGeneral(ex: Exception): ResponseEntity<ApiError> {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(
            ApiError("Internal server error", "INTERNAL_ERROR")
        )
    }
}
```

---

## Spring Data JPA

```kotlin
import org.springframework.data.jpa.repository.*
import org.springframework.data.repository.query.Param
import javax.persistence.*
import java.time.Instant

@Entity
@Table(name = "categories")
data class Category(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(nullable = false, unique = true, length = 100)
    var name: String,
    
    @Column
    var description: String? = null,
    
    @OneToMany(mappedBy = "category", cascade = [CascadeType.ALL], fetch = FetchType.LAZY)
    val products: MutableList<Product> = mutableListOf(),
    
    @Column(name = "created_at", updatable = false)
    val createdAt: Instant = Instant.now()
)

@Entity
@Table(name = "products", indexes = [
    Index(name = "idx_product_category", columnList = "category_id"),
    Index(name = "idx_product_name", columnList = "name")
])
data class Product(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(nullable = false, length = 200)
    var name: String,
    
    @Column(columnDefinition = "TEXT")
    var description: String,
    
    @Column(nullable = false, precision = 10, scale = 2)
    var price: Double,
    
    @Column(nullable = false)
    var stock: Int = 0,
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    var category: Category? = null,
    
    @Column(name = "created_at", updatable = false)
    val createdAt: Instant = Instant.now(),
    
    @Column(name = "updated_at")
    var updatedAt: Instant = Instant.now()
) {
    @PreUpdate
    fun preUpdate() { updatedAt = Instant.now() }
}

// Repository
interface ProductRepository : JpaRepository<Product, Long> {
    fun findByCategory(category: Category): List<Product>
    fun findByCategoryName(name: String): List<Product>
    
    @Query("SELECT p FROM Product p WHERE p.price BETWEEN :min AND :max")
    fun findByPriceRange(
        @Param("min") min: Double,
        @Param("max") max: Double
    ): List<Product>
    
    @Query("SELECT p FROM Product p WHERE LOWER(p.name) LIKE LOWER(CONCAT('%', :search, '%'))")
    fun findByNameContaining(@Param("search") search: String): List<Product>
    
    @Query("SELECT p FROM Product p WHERE p.stock <= :threshold")
    fun findLowStock(@Param("threshold") threshold: Int): List<Product>
    
    // Projection
    interface ProductSummary {
        val id: Long
        val name: String
        val price: Double
        val stock: Int
    }
    
    @Query("SELECT p.id as id, p.name as name, p.price as price, p.stock as stock FROM Product p")
    fun findAllSummaries(): List<ProductSummary>
    
    @Modifying
    @Query("UPDATE Product p SET p.stock = :stock WHERE p.id = :id")
    fun updateStock(@Param("id") id: Long, @Param("stock") stock: Int): Int
}

// Service
@org.springframework.stereotype.Service
@org.springframework.transaction.annotation.Transactional
class ProductService(
    private val productRepo: ProductRepository,
    private val categoryRepo: CategoryRepository
) {
    @org.springframework.transaction.annotation.Transactional(readOnly = true)
    fun findAll(
        page: Int, size: Int,
        category: String?,
        search: String?,
        sortBy: String,
        sortDir: String
    ): PagedResponse<ProductResponse> {
        val sort = org.springframework.data.domain.Sort.by(
            if (sortDir == "desc") org.springframework.data.domain.Sort.Direction.DESC
            else org.springframework.data.domain.Sort.Direction.ASC,
            sortBy
        )
        val pageable = org.springframework.data.domain.PageRequest.of(page, size, sort)
        
        val result = when {
            category != null && search != null -> 
                productRepo.findAll(pageable)  // simplified
            category != null -> 
                productRepo.findByCategoryName(category).let { 
                    org.springframework.data.domain.PageImpl(it, pageable, it.size.toLong())
                }
            search != null ->
                productRepo.findByNameContaining(search).let {
                    org.springframework.data.domain.PageImpl(it, pageable, it.size.toLong())
                }
            else -> productRepo.findAll(pageable)
        }
        
        return PagedResponse(
            data = result.content.map { it.toResponse() },
            page = result.number,
            pageSize = result.size,
            total = result.totalElements,
            totalPages = result.totalPages
        )
    }
    
    @org.springframework.transaction.annotation.Transactional(readOnly = true)
    fun findById(id: Long) = productRepo.findById(id).orElse(null)?.toResponse()
    
    fun create(req: CreateProductRequest): ProductResponse {
        val category = req.categoryId.let { 
            categoryRepo.findById(it).orElseThrow { NoSuchElementException("Category not found") }
        }
        val product = Product(
            name = req.name,
            description = req.description,
            price = req.price,
            stock = req.stock,
            category = category
        )
        return productRepo.save(product).toResponse()
    }
    
    fun update(id: Long, req: UpdateProductRequest): ProductResponse? {
        val product = productRepo.findById(id).orElse(null) ?: return null
        
        req.name?.let { product.name = it }
        req.description?.let { product.description = it }
        req.price?.let { product.price = it }
        req.stock?.let { product.stock = it }
        
        return productRepo.save(product).toResponse()
    }
    
    fun delete(id: Long): Boolean {
        if (!productRepo.existsById(id)) return false
        productRepo.deleteById(id)
        return true
    }
    
    fun findByCategory(category: String) =
        productRepo.findByCategoryName(category).map { it.toResponse() }
    
    fun updateStock(id: Long, quantity: Int): ProductResponse? {
        val updated = productRepo.updateStock(id, quantity)
        if (updated == 0) return null
        return productRepo.findById(id).orElse(null)?.toResponse()
    }
    
    private fun Product.toResponse() = ProductResponse(
        id = id,
        name = name,
        description = description,
        price = price,
        stock = stock,
        categoryName = category?.name,
        createdAt = createdAt
    )
}

interface CategoryRepository : JpaRepository<Category, Long> {
    fun findByName(name: String): Category?
}
```

---

## Testing Spring Boot

```kotlin
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.boot.test.mock.mockito.MockBean
import org.springframework.test.web.servlet.*
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.*
import com.fasterxml.jackson.databind.ObjectMapper
import org.mockito.kotlin.*

@WebMvcTest(ProductController::class)
class ProductControllerTest(
    @Autowired val mockMvc: MockMvc,
    @Autowired val objectMapper: ObjectMapper
) {
    @MockBean
    lateinit var productService: ProductService
    
    @Test
    fun `GET products should return paged results`() {
        val mockPage = PagedResponse(
            data = listOf(
                ProductResponse(1, "Laptop", "Fast laptop", 25000.0, 10, "Electronics", Instant.now())
            ),
            page = 0, pageSize = 20, total = 1, totalPages = 1
        )
        
        whenever(productService.findAll(any(), any(), any(), any(), any(), any()))
            .thenReturn(mockPage)
        
        mockMvc.get("/api/v1/products") {
            accept = MediaType.APPLICATION_JSON
        }.andExpect {
            status { isOk() }
            content { contentType(MediaType.APPLICATION_JSON) }
            jsonPath("$.data[0].name") { value("Laptop") }
            jsonPath("$.total") { value(1) }
        }
    }
    
    @Test
    fun `POST product should create and return 201`() {
        val request = CreateProductRequest(
            name = "New Product",
            description = "Description",
            price = 100.0,
            stock = 5,
            categoryId = 1
        )
        
        val response = ProductResponse(
            id = 1, name = "New Product", description = "Description",
            price = 100.0, stock = 5, categoryName = "Test",
            createdAt = Instant.now()
        )
        
        whenever(productService.create(any())).thenReturn(response)
        
        mockMvc.post("/api/v1/products") {
            contentType = MediaType.APPLICATION_JSON
            content = objectMapper.writeValueAsString(request)
        }.andExpect {
            status { isCreated() }
            header { exists("Location") }
            jsonPath("$.id") { value(1) }
        }
    }
    
    @Test
    fun `POST product with invalid data should return 400`() {
        val invalid = mapOf("name" to "", "price" to -100.0)
        
        mockMvc.post("/api/v1/products") {
            contentType = MediaType.APPLICATION_JSON
            content = objectMapper.writeValueAsString(invalid)
        }.andExpect {
            status { isBadRequest() }
            jsonPath("$.code") { value("VALIDATION_ERROR") }
        }
    }
}

// Integration test
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.ANY)
class ProductIntegrationTest {
    @Autowired
    lateinit var restTemplate: TestRestTemplate
    
    @Autowired
    lateinit var productRepo: ProductRepository
    
    @Autowired
    lateinit var categoryRepo: CategoryRepository
    
    @Test
    fun `full CRUD lifecycle`() {
        val category = categoryRepo.save(Category(name = "Test Category"))
        
        // Create
        val request = CreateProductRequest("Test", "Desc", 100.0, 5, category.id)
        val createResponse = restTemplate.postForEntity(
            "/api/v1/products",
            request,
            ProductResponse::class.java
        )
        assertEquals(HttpStatus.CREATED, createResponse.statusCode)
        val productId = createResponse.body!!.id
        
        // Read
        val getResponse = restTemplate.getForEntity(
            "/api/v1/products/$productId",
            ProductResponse::class.java
        )
        assertEquals(HttpStatus.OK, getResponse.statusCode)
        assertEquals("Test", getResponse.body!!.name)
        
        // Delete
        restTemplate.delete("/api/v1/products/$productId")
        
        val afterDelete = restTemplate.getForEntity(
            "/api/v1/products/$productId",
            Any::class.java
        )
        assertEquals(HttpStatus.NOT_FOUND, afterDelete.statusCode)
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง Task Management API

@Entity
@Table(name = "tasks")
data class Task(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    
    @Column(nullable = false, length = 200)
    var title: String,
    
    @Column(columnDefinition = "TEXT")
    var description: String = "",
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    var status: TaskStatus = TaskStatus.TODO,
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    var priority: Priority = Priority.MEDIUM,
    
    @Column(name = "due_date")
    var dueDate: java.time.LocalDate? = null,
    
    @Column(name = "created_at", updatable = false)
    val createdAt: java.time.Instant = java.time.Instant.now()
) {
    enum class TaskStatus { TODO, IN_PROGRESS, DONE, CANCELLED }
    enum class Priority { LOW, MEDIUM, HIGH, URGENT }
}

interface TaskRepository : JpaRepository<Task, Long> {
    fun findByStatus(status: Task.TaskStatus): List<Task>
    fun findByPriority(priority: Task.Priority): List<Task>
    fun findByDueDateBefore(date: java.time.LocalDate): List<Task>
    
    @Query("SELECT t FROM Task t WHERE t.status != 'DONE' AND t.dueDate < :today")
    fun findOverdue(@Param("today") today: java.time.LocalDate): List<Task>
}

@RestController
@RequestMapping("/api/tasks")
class TaskController(private val repo: TaskRepository) {
    
    @GetMapping
    fun list(
        @RequestParam(required = false) status: Task.TaskStatus?,
        @RequestParam(required = false) priority: Task.Priority?
    ) = when {
        status != null   -> repo.findByStatus(status)
        priority != null -> repo.findByPriority(priority)
        else             -> repo.findAll()
    }
    
    @GetMapping("/{id}")
    fun get(@PathVariable id: Long) =
        repo.findById(id).orElseThrow { NoSuchElementException("Task $id not found") }
    
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun create(@RequestBody @Validated task: Task) = repo.save(task)
    
    @PatchMapping("/{id}/status")
    fun updateStatus(
        @PathVariable id: Long,
        @RequestParam status: Task.TaskStatus
    ): Task {
        val task = repo.findById(id).orElseThrow { NoSuchElementException("Task $id not found") }
        task.status = status
        return repo.save(task)
    }
    
    @GetMapping("/overdue")
    fun getOverdue() = repo.findOverdue(java.time.LocalDate.now())
}
```

---

## สรุป Part 28

```
✅ kotlin.plugin.spring: open classes อัตโนมัติสำหรับ Spring proxies
✅ kotlin.plugin.jpa: no-arg constructors สำหรับ JPA entities
✅ @RestController + @RequestMapping: define REST endpoints
✅ @GetMapping/@PostMapping/@PutMapping/@DeleteMapping/@PatchMapping
✅ @PathVariable, @RequestParam, @RequestBody
✅ ResponseEntity<T> สำหรับ control status/headers
✅ @Validated + javax.validation: automatic request validation
✅ @RestControllerAdvice: global exception handling
✅ JpaRepository: CRUD + custom queries
✅ @Query: JPQL custom queries
✅ @Transactional: transaction management
✅ @WebMvcTest: test controllers without full Spring context
✅ MockBean: mock dependencies in tests
✅ SpringBootTest: full integration tests
```

---

*Part 28/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
