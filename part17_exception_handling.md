# Part 17: Exception Handling

## สารบัญ
1. [Exception Basics](#exception-basics)
2. [Custom Exceptions](#custom-exceptions)
3. [Try-Catch-Finally](#try-catch-finally)
4. [Exception Hierarchy](#exception-hierarchy)
5. [Checked vs Unchecked](#checked-vs-unchecked)
6. [Error Handling Patterns](#error-handling-patterns)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Exception Basics

```kotlin
fun main() {
    // throw exception
    fun divide(a: Int, b: Int): Int {
        if (b == 0) throw ArithmeticException("Cannot divide by zero")
        return a / b
    }
    
    // try-catch
    try {
        println(divide(10, 2))  // 5
        println(divide(10, 0))  // throws
    } catch (e: ArithmeticException) {
        println("Math error: ${e.message}")
    }
    
    // try as expression
    val result = try {
        divide(10, 0)
    } catch (e: ArithmeticException) {
        -1  // default value
    }
    println("Result: $result")  // -1
    
    // Multiple catch blocks
    fun riskyParse(s: String): Int {
        return s.toInt()
    }
    
    try {
        println(riskyParse("123"))   // 123
        println(riskyParse("abc"))   // throws NumberFormatException
    } catch (e: NumberFormatException) {
        println("Not a number: ${e.message}")
    } catch (e: Exception) {
        println("General error: ${e.message}")
    }
    
    // finally
    fun openResource(): String {
        println("Opening resource")
        return "ResourceData"
    }
    
    fun closeResource() {
        println("Closing resource")
    }
    
    try {
        val data = openResource()
        println("Processing: $data")
        throw RuntimeException("Processing failed!")
    } catch (e: RuntimeException) {
        println("Error: ${e.message}")
    } finally {
        closeResource()  // รันเสมอ
    }
    
    // Exception properties
    try {
        throw IllegalArgumentException("Bad input")
    } catch (e: Exception) {
        println("Message: ${e.message}")
        println("Class: ${e::class.simpleName}")
        println("Stack trace (first line): ${e.stackTrace.firstOrNull()}")
    }
}
```

---

## Custom Exceptions

```kotlin
// Custom exception hierarchy
open class AppException(
    message: String,
    cause: Throwable? = null
) : Exception(message, cause)

class ValidationException(
    val field: String,
    message: String
) : AppException("Validation failed for '$field': $message")

class AuthenticationException(
    message: String = "Authentication failed"
) : AppException(message)

class NotFoundException(
    val resourceType: String,
    val id: Any
) : AppException("$resourceType with id=$id not found")

class DatabaseException(
    message: String,
    cause: Throwable? = null
) : AppException(message, cause)

// Data class exception (readable)
data class BusinessException(
    val code: String,
    override val message: String
) : Exception(message)

// Exception with additional data
class ApiException(
    val statusCode: Int,
    val errorBody: String,
    message: String = "API call failed with status $statusCode"
) : AppException(message) {
    override fun toString() = "ApiException($statusCode): $message\nBody: $errorBody"
}

fun main() {
    // Validation
    fun validateAge(age: Int) {
        if (age < 0) throw ValidationException("age", "must be non-negative")
        if (age > 150) throw ValidationException("age", "exceeds maximum (150)")
    }
    
    fun validateEmail(email: String) {
        if (!email.contains("@")) throw ValidationException("email", "must contain @")
    }
    
    try {
        validateAge(-5)
    } catch (e: ValidationException) {
        println("${e.field}: ${e.message}")
    }
    
    // Multiple validations
    data class User(val name: String, val age: Int, val email: String)
    
    fun validateUser(name: String, age: Int, email: String): User {
        val errors = mutableListOf<String>()
        
        if (name.isBlank()) errors.add("name: cannot be blank")
        if (age < 0 || age > 150) errors.add("age: must be 0-150")
        if (!email.contains("@")) errors.add("email: invalid format")
        
        if (errors.isNotEmpty()) {
            throw ValidationException("user", errors.joinToString("; "))
        }
        
        return User(name, age, email)
    }
    
    try {
        validateUser("", -1, "invalid")
    } catch (e: ValidationException) {
        println("Validation errors: ${e.message}")
    }
    
    // Chained exceptions
    fun fetchFromDatabase(id: Int): String {
        try {
            if (id <= 0) throw IllegalArgumentException("Invalid ID")
            return "User #$id"
        } catch (e: IllegalArgumentException) {
            throw DatabaseException("Failed to fetch user", cause = e)
        }
    }
    
    try {
        fetchFromDatabase(-1)
    } catch (e: DatabaseException) {
        println("DB Error: ${e.message}")
        println("Caused by: ${e.cause?.message}")
    }
}
```

---

## Exception Hierarchy

```kotlin
/*
Throwable
├── Error (ปัญหา JVM - ไม่ควร catch)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── VirtualMachineError
└── Exception
    ├── RuntimeException (unchecked)
    │   ├── NullPointerException
    │   ├── IllegalArgumentException
    │   ├── IllegalStateException
    │   ├── IndexOutOfBoundsException
    │   │   └── ArrayIndexOutOfBoundsException
    │   ├── ClassCastException
    │   ├── ArithmeticException
    │   ├── NumberFormatException
    │   ├── UnsupportedOperationException
    │   └── ConcurrentModificationException
    ├── IOException (checked in Java, unchecked in Kotlin)
    │   ├── FileNotFoundException
    │   └── SocketException
    └── Your Custom Exceptions
*/

fun main() {
    // IllegalArgumentException: ข้อมูล argument ไม่ถูกต้อง
    fun setAge(age: Int) = require(age in 0..150) { "Age must be 0-150, got $age" }
    
    // IllegalStateException: สถานะปัจจุบันไม่รองรับ operation
    class Connection {
        var isOpen = false
        fun send(data: String) = check(isOpen) { "Connection is not open" }
    }
    
    // IndexOutOfBoundsException
    val list = listOf(1, 2, 3)
    try {
        list[5]  // throws
    } catch (e: IndexOutOfBoundsException) {
        println("Bad index: ${e.message}")
    }
    
    // NullPointerException (rare in Kotlin due to null safety)
    val s: String? = null
    try {
        s!!.length  // force null → NPE
    } catch (e: NullPointerException) {
        println("NPE: ${e.message}")
    }
    
    // ClassCastException
    val obj: Any = "hello"
    try {
        val num = obj as Int  // throws
    } catch (e: ClassCastException) {
        println("Cast failed: ${e.message}")
    }
    
    // Safe cast (no exception)
    val num = obj as? Int  // null instead of exception
    println("Safe cast: $num")  // null
    
    // require vs check vs error
    fun processAge(age: Int): String {
        require(age >= 0) { "Age cannot be negative: $age" }  // IllegalArgumentException
        
        val category = when {
            age < 18 -> "minor"
            age < 65 -> "adult"
            else -> "senior"
        }
        
        check(category != "unknown") { "Unexpected category" }  // IllegalStateException
        
        return category
    }
    
    try { processAge(-5) } catch (e: IllegalArgumentException) { println(e.message) }
}
```

---

## Error Handling Patterns

### Result Type

```kotlin
sealed class Either<out L, out R> {
    data class Left<L>(val value: L) : Either<L, Nothing>()
    data class Right<R>(val value: R) : Either<Nothing, R>()
}

fun main() {
    // Kotlin's built-in Result
    fun parseInt(s: String): Result<Int> = runCatching { s.toInt() }
    
    val good = parseInt("123")
    val bad = parseInt("abc")
    
    good.onSuccess { println("Parsed: $it") }
    bad.onFailure { println("Failed: ${it.message}") }
    
    // getOrNull, getOrDefault, getOrElse
    println(good.getOrNull())       // 123
    println(bad.getOrNull())        // null
    println(bad.getOrDefault(-1))   // -1
    println(bad.getOrElse { -2 })   // -2
    
    // map/mapCatching
    val doubled = good.map { it * 2 }
    println(doubled)  // Success(246)
    
    // fold
    val result = bad.fold(
        onSuccess = { "Parsed: $it" },
        onFailure = { "Error: ${it.message}" }
    )
    println(result)
    
    // Chain operations
    fun processInput(input: String): Result<String> =
        runCatching { input.trim() }
            .mapCatching { 
                require(it.isNotEmpty()) { "Input is empty" }
                it.toInt()
            }
            .map { "Processed: ${it * 2}" }
    
    println(processInput("  42  "))  // Success(Processed: 84)
    println(processInput(""))        // Failure(...)
    println(processInput("abc"))     // Failure(...)
    
    // Multiple operations
    data class FormData(val name: String, val age: String, val email: String)
    
    fun validate(form: FormData): Result<Triple<String, Int, String>> = runCatching {
        require(form.name.isNotBlank()) { "Name is required" }
        val age = form.age.toInt()
        require(age in 0..150) { "Age must be 0-150" }
        require(form.email.contains("@")) { "Invalid email" }
        Triple(form.name, age, form.email)
    }
    
    val forms = listOf(
        FormData("สมชาย", "25", "test@email.com"),
        FormData("", "25", "test@email.com"),
        FormData("สมหญิง", "abc", "test@email.com"),
        FormData("สมศักดิ์", "25", "invalidemail")
    )
    
    forms.forEach { form ->
        validate(form).fold(
            onSuccess = { (name, age, email) -> println("✅ $name, $age, $email") },
            onFailure = { println("❌ ${it.message}") }
        )
    }
}
```

### Error Accumulation

```kotlin
data class ValidationError(val field: String, val message: String)

sealed class Validated<out T> {
    data class Valid<T>(val value: T) : Validated<T>()
    data class Invalid(val errors: List<ValidationError>) : Validated<Nothing>()
    
    fun isValid() = this is Valid
    
    fun getOrNull() = (this as? Valid)?.value
}

fun <T> validated(value: T): Validated<T> = Validated.Valid(value)

fun <T> T.validate(
    vararg rules: Pair<Boolean, (T) -> ValidationError>
): Validated<T> {
    val errors = rules
        .filter { (condition, _) -> condition }
        .map { (_, error) -> error(this) }
    
    return if (errors.isEmpty()) Validated.Valid(this)
    else Validated.Invalid(errors)
}

fun main() {
    data class RegistrationForm(
        val username: String,
        val password: String,
        val email: String,
        val age: Int
    )
    
    fun validateForm(form: RegistrationForm): Validated<RegistrationForm> {
        val errors = mutableListOf<ValidationError>()
        
        if (form.username.length < 3)
            errors.add(ValidationError("username", "too short (min 3)"))
        if (form.username.contains(" "))
            errors.add(ValidationError("username", "no spaces allowed"))
        if (form.password.length < 8)
            errors.add(ValidationError("password", "too short (min 8)"))
        if (!form.password.any { it.isDigit() })
            errors.add(ValidationError("password", "must contain digit"))
        if (!form.email.contains("@"))
            errors.add(ValidationError("email", "invalid format"))
        if (form.age < 13)
            errors.add(ValidationError("age", "must be 13+"))
        
        return if (errors.isEmpty()) Validated.Valid(form)
        else Validated.Invalid(errors)
    }
    
    val testForms = listOf(
        RegistrationForm("ab", "weak", "bad", 10),
        RegistrationForm("validuser", "SecurePass1", "user@email.com", 25)
    )
    
    testForms.forEach { form ->
        when (val result = validateForm(form)) {
            is Validated.Valid -> println("✅ Form valid: ${result.value.username}")
            is Validated.Invalid -> {
                println("❌ Errors for ${form.username}:")
                result.errors.forEach { (field, msg) ->
                    println("   $field: $msg")
                }
            }
        }
    }
}
```

---

## แบบฝึกหัด

### Exercise: Safe File Parser

```kotlin
import java.io.File

sealed class ParseResult<T> {
    data class Success<T>(val data: T, val warnings: List<String> = emptyList()) : ParseResult<T>()
    data class Failure<T>(val error: String, val line: Int? = null) : ParseResult<T>()
}

data class Record(val id: Int, val name: String, val value: Double)

fun parseCSV(content: String): ParseResult<List<Record>> {
    val lines = content.lines().filter { it.isNotBlank() }
    if (lines.isEmpty()) return ParseResult.Failure("Empty input")
    
    val records = mutableListOf<Record>()
    val warnings = mutableListOf<String>()
    
    lines.forEachIndexed { lineNum, line ->
        val parts = line.split(",").map { it.trim() }
        if (parts.size < 3) {
            warnings.add("Line ${lineNum + 1}: insufficient columns, skipping")
            return@forEachIndexed
        }
        
        val id = parts[0].toIntOrNull()
            ?: run {
                warnings.add("Line ${lineNum + 1}: invalid ID '${parts[0]}', skipping")
                return@forEachIndexed
            }
        
        val value = parts[2].toDoubleOrNull()
            ?: run {
                warnings.add("Line ${lineNum + 1}: invalid value '${parts[2]}', skipping")
                return@forEachIndexed
            }
        
        records.add(Record(id, parts[1], value))
    }
    
    return ParseResult.Success(records, warnings)
}

fun main() {
    val csv = """
        1, Item A, 10.5
        2, Item B, 20.0
        bad, Item C, 15.0
        4, Item D, invalid
        5, Item E, 30.0
        short line
    """.trimIndent()
    
    when (val result = parseCSV(csv)) {
        is ParseResult.Success -> {
            println("Parsed ${result.data.size} records:")
            result.data.forEach { println("  $it") }
            
            if (result.warnings.isNotEmpty()) {
                println("\nWarnings:")
                result.warnings.forEach { println("  ⚠️ $it") }
            }
        }
        is ParseResult.Failure -> println("Parse failed: ${result.error}")
    }
}
```

---

## สรุป Part 17

```
✅ throw ExceptionType("message") ส่ง exception
✅ try-catch-finally จัดการ exception
✅ try เป็น expression - คืนค่าได้
✅ Multiple catch blocks ลำดับ specific ก่อน general
✅ Custom exceptions ด้วย open class ... : Exception()
✅ require(): IllegalArgumentException
✅ check(): IllegalStateException
✅ error(): IllegalStateException ที่ return Nothing
✅ runCatching { }: สร้าง Result<T>
✅ Result.map, onSuccess, onFailure, getOrElse
✅ Error accumulation: validate หลาย rules พร้อมกัน
✅ Either<L, R> pattern สำหรับ functional error handling
✅ Sealed class สำหรับ type-safe error handling
```

---

*Part 17/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
