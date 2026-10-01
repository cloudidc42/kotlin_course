# Part 59: Functional Programming ขั้นสูงด้วย Arrow Kt

## สารบัญ
1. [Arrow Kt คืออะไร](#arrow-kt-คืออะไร)
2. [Either และ Option](#either-และ-option)
3. [Validated](#validated)
4. [IO Monad](#io-monad)
5. [Optics](#optics)
6. [Function Composition](#function-composition)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Arrow Kt คืออะไร

Arrow เป็น functional programming library สำหรับ Kotlin — Monads, Type classes, Optics

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.arrow-kt:arrow-core:1.2.4")
    implementation("io.arrow-kt:arrow-fx-coroutines:1.2.4")
    implementation("io.arrow-kt:arrow-optics:1.2.4")
    ksp("io.arrow-kt:arrow-optics-ksp-plugin:1.2.4")
}
```

---

## Either: Typed Error Handling

```kotlin
import arrow.core.*

// Either<E, A>: แทน operation ที่อาจ succeed (Right) หรือ fail (Left)

sealed class DomainError {
    data class NotFound(val id: String) : DomainError()
    data class ValidationError(val field: String, val message: String) : DomainError()
    data class Unauthorized(val reason: String) : DomainError()
    data class ExternalServiceError(val service: String, val message: String) : DomainError()
}

// ฟังก์ชันที่คืน Either แทน Exception
fun findUser(id: String): Either<DomainError, User3> {
    if (id.isBlank()) return DomainError.ValidationError("id", "ID cannot be blank").left()
    
    val user = userDatabase[id] ?: return DomainError.NotFound(id).left()
    return user.right()
}

fun validateEmail(email: String): Either<DomainError, String> {
    return if (email.contains("@")) email.right()
    else DomainError.ValidationError("email", "Invalid email format").left()
}

// Chaining operations
fun getUserEmail(userId: String): Either<DomainError, String> {
    return findUser(userId)
        .flatMap { user ->
            validateEmail(user.email)
        }
        .map { email ->
            email.lowercase()
        }
}

// map, flatMap, fold
fun processUser(userId: String): String {
    return findUser(userId)
        .map { user -> "Hello, ${user.name}!" }
        .getOrElse { error -> "Error: $error" }
}

// fold: handle both cases
fun handleResult(userId: String): Response {
    return findUser(userId).fold(
        ifLeft = { error ->
            when (error) {
                is DomainError.NotFound -> Response.notFound("User ${error.id} not found")
                is DomainError.ValidationError -> Response.badRequest("${error.field}: ${error.message}")
                else -> Response.serverError("Unexpected error")
            }
        },
        ifRight = { user ->
            Response.ok(user)
        }
    )
}

// Either.catch: wrap exceptions
fun parseJson(json: String): Either<DomainError, Map<String, Any>> {
    return Either.catch {
        com.fasterxml.jackson.databind.ObjectMapper().readValue(json, Map::class.java) as Map<String, Any>
    }.mapLeft { DomainError.ValidationError("json", it.message ?: "Invalid JSON") }
}

// zip: combine multiple Either results
fun createOrder(userId: String, productId: String): Either<DomainError, OrderDto2> {
    return Either.zipOrAccumulate(
        findUser(userId),
        findProduct(productId)
    ) { user, product ->
        OrderDto2(
            orderId = java.util.UUID.randomUUID().toString(),
            userId = user.id,
            productId = product.id,
            price = product.price
        )
    }
}

data class User3(val id: String, val name: String, val email: String)
data class Product3(val id: String, val name: String, val price: Double)
data class OrderDto2(val orderId: String, val userId: String, val productId: String, val price: Double)
data class Response(val status: Int, val body: Any?) {
    companion object {
        fun ok(body: Any?) = Response(200, body)
        fun badRequest(msg: String) = Response(400, msg)
        fun notFound(msg: String) = Response(404, msg)
        fun serverError(msg: String) = Response(500, msg)
    }
}

val userDatabase = mapOf("user-1" to User3("user-1", "Alice", "alice@example.com"))
fun findProduct(id: String): Either<DomainError, Product3> = 
    if (id.startsWith("prod")) Product3(id, "Product $id", 99.99).right()
    else DomainError.NotFound(id).left()
```

---

## Option: Nullable ที่ Explicit

```kotlin
import arrow.core.*

// Option<A>: แทน value ที่อาจมีหรือไม่มี — ชัดเจนกว่า nullable

fun findUserById(id: String): Option<User3> {
    return userDatabase[id].toOption()
}

// Map, flatMap เหมือน Either
fun getUserName(id: String): Option<String> {
    return findUserById(id).map { it.name }
}

// getOrElse
fun getUserNameOrDefault(id: String, default: String): String {
    return findUserById(id)
        .map { it.name }
        .getOrElse { default }
}

// None และ Some
val maybeUser: Option<User3> = if (true) User3("1", "Bob", "bob@ex.com").some() else None

// Filter
fun findActiveUser(id: String): Option<User3> {
    return findUserById(id).filter { it.name.isNotBlank() }
}

// Traverse: List<Option<A>> → Option<List<A>>
fun findAllUsers(ids: List<String>): Option<List<User3>> {
    return ids.traverse { id -> findUserById(id) }
}

// toEither
fun findUserOrError(id: String): Either<DomainError, User3> {
    return findUserById(id).toEither { DomainError.NotFound(id) }
}
```

---

## Validated: Accumulate Multiple Errors

```kotlin
import arrow.core.*

// Validated แตกต่างจาก Either: ไม่หยุดที่ error แรก สะสม errors ทั้งหมด

data class UserInput(
    val username: String,
    val email: String,
    val age: Int,
    val password: String
)

data class ValidatedUser(
    val username: String,
    val email: String,
    val age: Int,
    val password: String
)

object UserValidator {
    
    fun validateUsername(username: String): ValidatedNel<String, String> {
        return when {
            username.isBlank() -> "Username cannot be blank".invalidNel()
            username.length < 3 -> "Username must be at least 3 characters".invalidNel()
            username.length > 20 -> "Username must not exceed 20 characters".invalidNel()
            !username.matches(Regex("[a-zA-Z0-9_]+")) -> "Username can only contain letters, numbers and underscores".invalidNel()
            else -> username.valid()
        }
    }
    
    fun validateEmail(email: String): ValidatedNel<String, String> {
        return if (email.matches(Regex(".+@.+\\..+"))) email.valid()
        else "Invalid email format".invalidNel()
    }
    
    fun validateAge(age: Int): ValidatedNel<String, Int> {
        return when {
            age < 13 -> "Must be at least 13 years old".invalidNel()
            age > 120 -> "Invalid age".invalidNel()
            else -> age.valid()
        }
    }
    
    fun validatePassword(password: String): ValidatedNel<String, String> {
        val errors = mutableListOf<String>()
        if (password.length < 8) errors.add("Password must be at least 8 characters")
        if (!password.any { it.isUpperCase() }) errors.add("Password must contain uppercase letter")
        if (!password.any { it.isLowerCase() }) errors.add("Password must contain lowercase letter")
        if (!password.any { it.isDigit() }) errors.add("Password must contain digit")
        
        return if (errors.isEmpty()) password.valid()
        else errors.toNonEmptyListOrNull()!!.invalid()
    }
    
    fun validate(input: UserInput): ValidatedNel<String, ValidatedUser> {
        return Validated.zipOrAccumulate(
            validateUsername(input.username),
            validateEmail(input.email),
            validateAge(input.age),
            validatePassword(input.password)
        ) { username, email, age, password ->
            ValidatedUser(username, email, age, password)
        }
    }
}

// ใช้งาน
fun registerUser(input: UserInput): Either<List<String>, ValidatedUser> {
    return UserValidator.validate(input)
        .toEither()
        .mapLeft { it.toList() }
}

fun handleRegistration(input: UserInput) {
    when (val result = registerUser(input)) {
        is Either.Right -> println("Registered: ${result.value}")
        is Either.Left -> {
            println("Validation errors:")
            result.value.forEach { println("  - $it") }
        }
    }
}
```

---

## Function Composition

```kotlin
import arrow.core.*

// Function composition เป็นหัวใจของ FP

// andThen / compose
val addOne: (Int) -> Int = { it + 1 }
val double: (Int) -> Int = { it * 2 }
val square: (Int) -> Int = { it * it }

val addOneThenDouble = addOne andThen double  // double(addOne(x))
val doubleThendAddOne = double andThen addOne  // addOne(double(x))

fun main() {
    println(addOneThenDouble(3))   // (3+1)*2 = 8
    println(doubleThendAddOne(3))  // (3*2)+1 = 7
}

// Partial Application
fun add(x: Int, y: Int) = x + y
fun multiply(x: Int, y: Int) = x * y

val addFive = { y: Int -> add(5, y) }
val triple = { y: Int -> multiply(3, y) }

// Currying
fun <A, B, C> curry(f: (A, B) -> C): (A) -> (B) -> C = { a -> { b -> f(a, b) } }

val curriedAdd = curry(::add)
val addTen = curriedAdd(10)
println(addTen(5))   // 15
println(addTen(20))  // 30

// Memoization
fun <A, B> memoize(f: (A) -> B): (A) -> B {
    val cache = mutableMapOf<A, B>()
    return { a -> cache.getOrPut(a) { f(a) } }
}

val fib: (Int) -> Long = memoize { n ->
    when (n) {
        0 -> 0L
        1 -> 1L
        else -> fib(n - 1) + fib(n - 2)
    }
}

// Trampolining: ป้องกัน stack overflow ใน recursive functions
sealed class Trampoline<A> {
    data class Done<A>(val result: A) : Trampoline<A>()
    data class Bounce<A>(val thunk: () -> Trampoline<A>) : Trampoline<A>()
    
    tailrec fun run(): A = when (this) {
        is Done -> result
        is Bounce -> thunk().run()
    }
}

fun factorialTrampoline(n: Long, acc: Long = 1L): Trampoline<Long> {
    return if (n <= 1) Trampoline.Done(acc)
    else Trampoline.Bounce { factorialTrampoline(n - 1, n * acc) }
}

val factorial1000 = factorialTrampoline(1000L).run()

// Arrow Fx: functional effects
import arrow.fx.coroutines.*

// Resource: safe acquisition and release
suspend fun withDatabase(block: suspend (Any) -> Unit) {
    Resource.fromAutoCloseable { 
        java.io.Closeable { println("Closing DB") }
    }.use { db ->
        block(db)
    }
}

// Parallel operations with Resource safety
suspend fun parallelExample() {
    parZip(
        { fetchUser("user-1") },
        { fetchProduct("prod-1") },
        { fetchInventory("prod-1") }
    ) { user, product, inventory ->
        Triple(user, product, inventory)
    }
}

suspend fun fetchUser(id: String) = User3(id, "Alice", "alice@ex.com")
suspend fun fetchProduct(id: String) = Product3(id, "Widget", 99.99)
suspend fun fetchInventory(id: String) = 42
```

---

## Optics

```kotlin
// Arrow Optics: ทำงานกับ nested immutable data structures

import arrow.optics.*
import arrow.optics.dsl.*

@optics
data class Address(val street: String, val city: String, val country: String) {
    companion object
}

@optics
data class Person(val name: String, val age: Int, val address: Address) {
    companion object
}

@optics
data class Company(val name: String, val employees: List<Person>) {
    companion object
}

// ใช้งาน Optics
fun updatePersonCity(person: Person, newCity: String): Person {
    return Person.address.city.set(person, newCity)
}

fun incrementAllAges(company: Company): Company {
    return Company.employees.every(Every.list()).age.modify(company) { it + 1 }
}

// Lens: focus on single field
val addressLens = Person.address
val cityLens = Address.city

// Compose lenses
val personCityLens = addressLens compose cityLens

fun modifyCity(person: Person, transform: (String) -> String): Person {
    return personCityLens.modify(person, transform)
}

// Prism: focus on sum types
@optics
sealed class Shape {
    companion object
    data class Circle(val radius: Double) : Shape()
    data class Rectangle(val width: Double, val height: Double) : Shape()
}

fun scaleCircle(shape: Shape, factor: Double): Shape {
    return Shape.circle.radius.modify(shape) { it * factor }
}

// ISO: bijection between types
val stringToList = Iso<String, List<Char>>(
    get = { it.toList() },
    reverseGet = { it.joinToString("") }
)
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Build a complete validation pipeline using Either and Validated

// Requirements:
// 1. Validate CreateProductRequest using Validated (accumulate errors)
// 2. Check if category exists using Either (fail fast)
// 3. Check for duplicate SKU using Either
// 4. Return Either<List<ValidationError>, Product3>

data class CreateProductRequest2(
    val name: String,
    val description: String,
    val price: Double,
    val stockQuantity: Int,
    val categoryId: String,
    val sku: String
)

data class ValidationError2(val field: String, val message: String)

interface CategoryRepository2 {
    fun exists(id: String): Boolean
}

interface ProductRepository2 {
    fun findBySku(sku: String): Product3?
}

class ProductCreationService(
    private val categoryRepo: CategoryRepository2,
    private val productRepo: ProductRepository2
) {
    
    fun create(request: CreateProductRequest2): Either<List<ValidationError2>, Product3> {
        // Step 1: Validate request fields (collect all errors)
        val validation = validateRequest(request)
        
        // Step 2: If valid, check business rules (fail fast)
        return validation.toEither().mapLeft { it.toList() }.flatMap { validRequest ->
            checkCategoryExists(validRequest.categoryId)
                .flatMap { checkSkuUnique(validRequest.sku) }
                .map { Product3(java.util.UUID.randomUUID().toString(), validRequest.name, validRequest.price) }
        }
    }
    
    private fun validateRequest(request: CreateProductRequest2): ValidatedNel<ValidationError2, CreateProductRequest2> {
        // TODO: Implement validation
        TODO("Validate name, price > 0, stockQuantity >= 0, sku not blank")
    }
    
    private fun checkCategoryExists(categoryId: String): Either<List<ValidationError2>, String> {
        return if (categoryRepo.exists(categoryId)) categoryId.right()
        else listOf(ValidationError2("categoryId", "Category $categoryId not found")).left()
    }
    
    private fun checkSkuUnique(sku: String): Either<List<ValidationError2>, String> {
        return if (productRepo.findBySku(sku) == null) sku.right()
        else listOf(ValidationError2("sku", "SKU $sku already exists")).left()
    }
}
```

---

## สรุป Part 59

```
✅ Arrow Kt: functional programming library for Kotlin
✅ Either<E, A>: typed error handling, no exceptions
✅ map/flatMap: chain Either operations
✅ fold: handle both success and failure
✅ Either.catch: wrap exceptions in Either
✅ zipOrAccumulate: combine multiple Either results
✅ Option<A>: explicit nullable, clearer than null
✅ toOption/fromOption: interop with nullable
✅ Validated<E, A>: accumulate multiple errors
✅ ValidatedNel: use NonEmptyList for error accumulation
✅ Function composition: andThen, compose
✅ Currying: partial application
✅ Memoization: cache function results
✅ Trampolining: stack-safe recursion
✅ Arrow Optics: modify nested immutable data
✅ Lens, Prism, ISO: optics hierarchy
✅ @optics annotation: generate optics for data classes
```

---

*Part 59/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
