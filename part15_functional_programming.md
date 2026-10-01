# Part 15: Functional Programming

## สารบัญ
1. [First-Class Functions](#first-class-functions)
2. [Higher-Order Functions](#higher-order-functions)
3. [Lambda Expressions](#lambda-expressions)
4. [Closures](#closures)
5. [Function Composition](#function-composition)
6. [Currying และ Partial Application](#currying-และ-partial-application)
7. [Functors และ Monads](#functors-และ-monads)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## First-Class Functions

Functions เป็น first-class citizens ใน Kotlin - เก็บใน variable, ส่งเป็น argument, return ได้

```kotlin
fun main() {
    // Function reference
    fun double(x: Int) = x * 2
    val fn: (Int) -> Int = ::double
    
    println(fn(5))  // 10
    
    // Function type
    val add: (Int, Int) -> Int = { a, b -> a + b }
    val greet: (String) -> String = { name -> "Hello, $name!" }
    val print: (Any?) -> Unit = ::println
    
    println(add(3, 4))       // 7
    println(greet("Kotlin")) // Hello, Kotlin!
    print("Hi!")              // Hi!
    
    // Store functions in collections
    val operations: Map<String, (Int, Int) -> Int> = mapOf(
        "+" to { a, b -> a + b },
        "-" to { a, b -> a - b },
        "*" to { a, b -> a * b },
        "/" to { a, b -> if (b != 0) a / b else throw ArithmeticException() }
    )
    
    val op = operations["+"]!!
    println(op(10, 5))  // 15
    
    // Invoke operator
    println(op.invoke(10, 5))  // 15 (explicit)
    
    // Function as return type
    fun multiplier(factor: Int): (Int) -> Int = { x -> x * factor }
    
    val triple = multiplier(3)
    val times10 = multiplier(10)
    
    println(triple(7))   // 21
    println(times10(7))  // 70
}
```

---

## Higher-Order Functions

```kotlin
// รับ function เป็น parameter
fun <T, R> transform(items: List<T>, fn: (T) -> R): List<R> =
    items.map(fn)

// Return function
fun <T> createValidator(predicate: (T) -> Boolean): (T) -> Boolean =
    predicate

// รับ multiple functions
fun <T> List<T>.applyAll(vararg transforms: (T) -> T): List<T> {
    var result = this
    for (transform in transforms) {
        result = result.map(transform)
    }
    return result
}

// Function composition
fun <A, B, C> compose(f: (B) -> C, g: (A) -> B): (A) -> C = { f(g(it)) }

fun main() {
    val numbers = listOf(1, 2, 3, 4, 5)
    
    // Passing functions
    println(transform(numbers, { it * it }))      // [1, 4, 9, 16, 25]
    println(transform(numbers, ::Double::invoke))  // implicit conversion
    
    // applyAll
    val processed = numbers.applyAll(
        { it * 2 },     // double
        { it + 1 },     // increment
        { it * it }     // square
    )
    println(processed)  // [(1*2+1)², (2*2+1)², ...] = [9, 25, 49, 81, 121]
    
    // Compose
    val addOne: (Int) -> Int = { it + 1 }
    val double: (Int) -> Int = { it * 2 }
    val square: (Int) -> Int = { it * it }
    
    val f = compose(square, compose(double, addOne))  // x → ((x+1)*2)²
    println(f(3))  // ((3+1)*2)² = 8² = 64
    
    // Standard library HOF
    val words = listOf("kotlin", "java", "python", "swift")
    
    println(words.filter { it.length > 4 })      // [kotlin, python, swift]
    println(words.map { it.uppercase() })         // [KOTLIN, JAVA, PYTHON, SWIFT]
    println(words.sortedBy { it.length })         // [java, swift, kotlin, python]
    println(words.groupBy { it.length })          // {6=[kotlin, python], 4=[java, swift]}
    println(words.any { it.startsWith("k") })     // true
    println(words.all { it.length > 2 })          // true
    println(words.maxByOrNull { it.length })      // python
}
```

---

## Lambda Expressions

```kotlin
fun main() {
    // Lambda syntax
    val lambda1: (Int) -> Int = { x -> x * 2 }   // explicit parameter
    val lambda2: (Int) -> Int = { it * 2 }        // it shorthand
    
    // Multi-parameter lambda
    val add: (Int, Int) -> Int = { a, b -> a + b }
    
    // Multi-line lambda
    val process: (Int) -> String = { num ->
        val doubled = num * 2
        val formatted = "Number: $doubled"
        formatted  // last expression is return value
    }
    
    // Trailing lambda syntax
    fun operation(x: Int, fn: (Int) -> Int): Int = fn(x)
    
    val result1 = operation(5) { it * 3 }         // 15
    val result2 = operation(5, { it * 3 })         // same
    val result3 = operation(x = 5) { it * 3 }     // named param
    
    println(result1)
    
    // Lambda with receiver
    val buildHtml: StringBuilder.() -> Unit = {
        appendLine("<html>")
        appendLine("<body>Hello</body>")
        appendLine("</html>")
    }
    
    val html = buildString(buildHtml)
    println(html)
    
    // Anonymous function (explicit return)
    val div = fun(a: Int, b: Int): Int {
        if (b == 0) return 0  // explicit return in anonymous function
        return a / b
    }
    println(div(10, 3))  // 3
    println(div(10, 0))  // 0
    
    // Destructuring in lambda
    val pairs = listOf(Pair(1, "one"), Pair(2, "two"), Pair(3, "three"))
    pairs.forEach { (num, name) ->
        println("$num = $name")
    }
    
    // Unused parameters with _
    val withIndex = listOf("a", "b", "c").mapIndexed { _, value ->
        value.uppercase()
    }
    println(withIndex)  // [A, B, C]
}
```

---

## Closures

```kotlin
fun main() {
    // Closure: lambda captures outer variables
    var counter = 0
    
    val increment = { counter++ }
    val reset = { counter = 0 }
    val getCount = { counter }
    
    increment()
    increment()
    increment()
    println(getCount())  // 3
    
    reset()
    println(getCount())  // 0
    
    // Closure capturing mutable state
    fun makeCounter(start: Int = 0, step: Int = 1): () -> Int {
        var current = start
        return {
            val result = current
            current += step
            result
        }
    }
    
    val counter1 = makeCounter()
    val counter2 = makeCounter(10, 5)
    
    repeat(5) { print("${counter1()} ") }  // 0 1 2 3 4
    println()
    repeat(5) { print("${counter2()} ") }  // 10 15 20 25 30
    println()
    
    // Independent closures
    println("Counter1 still: ${counter1()}")  // 5 (independent)
    
    // Memoization with closure
    fun memoize(fn: (Int) -> Int): (Int) -> Int {
        val cache = mutableMapOf<Int, Int>()
        return { n ->
            cache.getOrPut(n) {
                println("Computing for $n...")
                fn(n)
            }
        }
    }
    
    val expensiveSquare = memoize { n ->
        Thread.sleep(1)  // simulate expensive computation
        n * n
    }
    
    println(expensiveSquare(5))   // Computing for 5... \n 25
    println(expensiveSquare(5))   // 25 (cached)
    println(expensiveSquare(10))  // Computing for 10... \n 100
    println(expensiveSquare(5))   // 25 (cached)
}
```

---

## Function Composition

```kotlin
// ฟังก์ชันช่วยสำหรับ composition
infix fun <A, B, C> ((A) -> B).andThen(f: (B) -> C): (A) -> C = { f(this(it)) }
infix fun <A, B, C> ((B) -> C).compose(f: (A) -> B): (A) -> C = { this(f(it)) }

fun main() {
    val addOne: (Int) -> Int = { it + 1 }
    val double: (Int) -> Int = { it * 2 }
    val square: (Int) -> Int = { it * it }
    val toString: (Int) -> String = { it.toString() }
    val addSuffix: (String) -> String = { "$it !" }
    
    // andThen: apply left then right
    val addOneThenDouble = addOne andThen double
    println(addOneThenDouble(3))  // (3+1)*2 = 8
    
    // compose: apply right then left
    val doubleAfterAddOne = double compose addOne
    println(doubleAfterAddOne(3))  // same: (3+1)*2 = 8
    
    // Chain
    val pipeline = addOne andThen double andThen square andThen toString andThen addSuffix
    println(pipeline(3))  // ((3+1)*2)² = 64 → "64 !"
    
    // Real-world pipeline
    data class RawData(val input: String)
    data class CleanData(val text: String)
    data class ProcessedData(val words: List<String>)
    data class Result(val summary: String)
    
    val clean: (RawData) -> CleanData = { raw ->
        CleanData(raw.input.trim().lowercase())
    }
    val tokenize: (CleanData) -> ProcessedData = { clean ->
        ProcessedData(clean.text.split("\\s+".toRegex()))
    }
    val analyze: (ProcessedData) -> Result = { processed ->
        Result("${processed.words.size} words, unique: ${processed.words.toSet().size}")
    }
    
    val processText = clean andThen tokenize andThen analyze
    val rawInput = RawData("  Hello World Hello Kotlin World  ")
    println(processText(rawInput).summary)  // 5 words, unique: 3
    
    // Partial application
    fun <A, B, C> partial(f: (A, B) -> C, a: A): (B) -> C = { b -> f(a, b) }
    
    val add: (Int, Int) -> Int = { a, b -> a + b }
    val add5 = partial(add, 5)
    println(add5(10))  // 15
    println(add5(20))  // 25
    
    // Applied to list
    val numbers = listOf(1, 2, 3, 4, 5)
    println(numbers.map(add5))  // [6, 7, 8, 9, 10]
}
```

---

## Currying และ Partial Application

```kotlin
// Currying: แปลง f(a, b, c) → f(a)(b)(c)
fun <A, B, C> curry(f: (A, B) -> C): (A) -> (B) -> C = { a -> { b -> f(a, b) } }

fun <A, B, C, D> curry3(f: (A, B, C) -> D): (A) -> (B) -> (C) -> D =
    { a -> { b -> { c -> f(a, b, c) } } }

fun main() {
    // Basic currying
    val add: (Int, Int) -> Int = { a, b -> a + b }
    val curriedAdd = curry(add)
    
    val add10 = curriedAdd(10)  // partial application
    println(add10(5))   // 15
    println(add10(20))  // 30
    
    // Apply to list
    val numbers = (1..10).toList()
    println(numbers.map(curriedAdd(100)))  // [101, 102, ..., 110]
    
    // Curried string formatting
    val format: (String, Any) -> String = { template, value -> template.format(value) }
    val curriedFormat = curry(format)
    
    val formatPrice = curriedFormat("ราคา: %.2f บาท")
    val formatScore = curriedFormat("คะแนน: %.1f%%")
    
    println(formatPrice(99.99))   // ราคา: 99.99 บาท
    println(formatScore(87.5))    // คะแนน: 87.5%
    
    // Curry3 example
    val between: (Int, Int, Int) -> Boolean = { min, max, value -> value in min..max }
    val curriedBetween = curry3(between)
    
    val between1and10 = curriedBetween(1)(10)
    println((1..15).filter(between1and10))  // [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
    
    // Practical: Validation factory
    val validLength: (Int) -> (String) -> Boolean = { maxLen -> { s -> s.length <= maxLen } }
    val validPattern: (Regex) -> (String) -> Boolean = { regex -> { s -> regex.matches(s) } }
    
    val maxLength10 = validLength(10)
    val isEmail = validPattern("""^[^@]+@[^@]+\.[^@]+$""".toRegex())
    
    val testStrings = listOf("hello", "this is a very long string", "test@email.com", "invalid")
    
    println("Max 10 chars: ${testStrings.filter(maxLength10)}")
    println("Valid email: ${testStrings.filter(isEmail)}")
}
```

---

## Functors และ Monads

```kotlin
// Functor: type ที่มี map function
// Kotlin's List, Set, Map, Result are all functors

// Option/Maybe Monad (คล้าย Haskell's Maybe)
sealed class Option<out T> {
    object None : Option<Nothing>()
    data class Some<T>(val value: T) : Option<T>()
    
    companion object {
        fun <T> of(value: T?): Option<T> =
            if (value == null) None else Some(value)
    }
}

// Functor: map
fun <T, R> Option<T>.map(fn: (T) -> R): Option<R> = when (this) {
    is Option.None -> Option.None
    is Option.Some -> Option.Some(fn(value))
}

// Monad: flatMap (bind)
fun <T, R> Option<T>.flatMap(fn: (T) -> Option<R>): Option<R> = when (this) {
    is Option.None -> Option.None
    is Option.Some -> fn(value)
}

fun <T> Option<T>.getOrElse(default: T): T = when (this) {
    is Option.None -> default
    is Option.Some -> value
}

fun <T> Option<T>.filter(predicate: (T) -> Boolean): Option<T> = when (this) {
    is Option.None -> Option.None
    is Option.Some -> if (predicate(value)) this else Option.None
}

fun main() {
    // Using Option Monad (null-safe chaining)
    data class User(val name: String, val email: String?)
    data class Email(val domain: String)
    
    fun findUser(id: Int): Option<User> = when (id) {
        1 -> Option.Some(User("สมชาย", "somchai@gmail.com"))
        2 -> Option.Some(User("สมหญิง", null))
        else -> Option.None
    }
    
    fun parseEmail(email: String): Option<Email> {
        val parts = email.split("@")
        return if (parts.size == 2) Option.Some(Email(parts[1]))
        else Option.None
    }
    
    // Chain operations safely
    for (id in 1..3) {
        val domain = findUser(id)
            .flatMap { user -> Option.of(user.email) }
            .flatMap { email -> parseEmail(email) }
            .map { email -> email.domain }
            .getOrElse("unknown")
        
        println("User $id domain: $domain")
    }
    // User 1 domain: gmail.com
    // User 2 domain: unknown
    // User 3 domain: unknown
    
    // Kotlin's built-in monadic operations
    val nullable: String? = "Hello"
    
    // let is like flatMap for nullable
    val result = nullable
        ?.let { it.uppercase() }
        ?.let { if (it.length > 3) it else null }
        ?.let { "$it!" }
        ?: "too short"
    
    println(result)  // HELLO!
    
    // Result Monad (Kotlin Result)
    fun parseInt(s: String): Result<Int> =
        runCatching { s.toInt() }
    
    fun divide(a: Int, b: Int): Result<Double> =
        if (b == 0) Result.failure(ArithmeticException("Div by zero"))
        else Result.success(a.toDouble() / b)
    
    val computation = parseInt("10")
        .mapCatching { a -> divide(a, 2).getOrThrow() }
    
    computation
        .onSuccess { println("Result: $it") }
        .onFailure { println("Error: ${it.message}") }
}
```

---

## แบบฝึกหัด

### Exercise: Functional Pipeline

```kotlin
data class Order(
    val id: String,
    val customer: String,
    val items: List<String>,
    val total: Double,
    val status: String
)

fun main() {
    val orders = listOf(
        Order("O001", "Alice", listOf("Laptop", "Mouse"), 26000.0, "completed"),
        Order("O002", "Bob", listOf("Phone"), 15000.0, "pending"),
        Order("O003", "Alice", listOf("Chair", "Desk"), 7500.0, "completed"),
        Order("O004", "Charlie", listOf("Headphones"), 3000.0, "cancelled"),
        Order("O005", "Bob", listOf("Laptop"), 25000.0, "completed")
    )
    
    // Functional pipeline
    val report = orders
        .filter { it.status == "completed" }
        .groupBy { it.customer }
        .mapValues { (_, orders) ->
            mapOf(
                "count" to orders.size,
                "total" to orders.sumOf { it.total },
                "avgOrder" to orders.map { it.total }.average()
            )
        }
    
    println("=== Sales Report ===")
    report.entries.sortedByDescending { it.value["total"] as Double }.forEach { (customer, stats) ->
        println("$customer:")
        println("  Orders: ${stats["count"]}")
        println("  Total: ${"%,.0f".format(stats["total"])} บาท")
        println("  Avg: ${"%,.0f".format(stats["avgOrder"])} บาท")
    }
    
    // Total revenue
    val totalRevenue = orders
        .filter { it.status == "completed" }
        .sumOf { it.total }
    println("\nTotal Revenue: ${"%,.0f".format(totalRevenue)} บาท")
    
    // Items sold
    val itemsSold = orders
        .filter { it.status == "completed" }
        .flatMap { it.items }
        .groupBy { it }
        .mapValues { it.value.size }
        .entries
        .sortedByDescending { it.value }
    
    println("\nItems sold:")
    itemsSold.forEach { (item, count) -> println("  $item: $count") }
}
```

---

## สรุป Part 15

```
✅ Functions เป็น First-class citizens
✅ Function types: (A, B) -> C
✅ Lambda expressions: { a, b -> a + b }
✅ it shorthand สำหรับ single parameter
✅ Trailing lambda syntax
✅ Higher-order functions: รับ/ส่งคืน functions
✅ Closures: capture ตัวแปรจาก outer scope
✅ Function composition ด้วย andThen/compose
✅ Currying: แปลง multi-param เป็น single-param chain
✅ Partial application: กำหนด argument บางส่วน
✅ Option/Maybe monad สำหรับ null safety
✅ Result monad สำหรับ error handling
✅ Kotlin's let, map, flatMap เป็น monadic operations
```

---

*Part 15/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
