# Part 07: ฟังก์ชัน (Functions)

## สารบัญ
1. [ฟังก์ชันพื้นฐาน](#ฟังก์ชันพื้นฐาน)
2. [Single-Expression Functions](#single-expression-functions)
3. [Default Parameters](#default-parameters)
4. [Named Arguments](#named-arguments)
5. [Vararg Parameters](#vararg-parameters)
6. [Return Types และ Unit](#return-types-และ-unit)
7. [Local Functions](#local-functions)
8. [Extension Functions](#extension-functions)
9. [Infix Functions](#infix-functions)
10. [Tail Recursive Functions](#tail-recursive-functions)
11. [Inline Functions (เบื้องต้น)](#inline-functions-เบื้องต้น)
12. [แบบฝึกหัด](#แบบฝึกหัด)

---

## ฟังก์ชันพื้นฐาน

### syntax ของฟังก์ชัน

```kotlin
// รูปแบบ:
// fun functionName(param1: Type1, param2: Type2): ReturnType {
//     // body
//     return value
// }

fun greet(name: String): String {
    return "สวัสดี, $name!"
}

fun add(a: Int, b: Int): Int {
    return a + b
}

fun printDivider() {  // ไม่มี return value
    println("=".repeat(40))
}

fun main() {
    println(greet("สมชาย"))          // สวัสดี, สมชาย!
    println(add(10, 20))              // 30
    printDivider()                    // ========================================
    
    // ฟังก์ชันที่ return หลายค่า ด้วย Pair/Triple
    fun divmod(a: Int, b: Int): Pair<Int, Int> {
        return Pair(a / b, a % b)
    }
    
    val (quotient, remainder) = divmod(17, 5)
    println("17 ÷ 5 = $quotient เศษ $remainder")  // 17 ÷ 5 = 3 เศษ 2
    
    // ฟังก์ชันที่ return ข้อมูลหลายชนิด ด้วย data class
    data class Stats(val min: Int, val max: Int, val avg: Double, val sum: Int)
    
    fun calculateStats(numbers: List<Int>): Stats {
        return Stats(
            min = numbers.min(),
            max = numbers.max(),
            avg = numbers.average(),
            sum = numbers.sum()
        )
    }
    
    val stats = calculateStats(listOf(5, 3, 8, 1, 9, 4, 7))
    println("Min: ${stats.min}, Max: ${stats.max}, Avg: ${"%.2f".format(stats.avg)}, Sum: ${stats.sum}")
}
```

### ฟังก์ชัน Top-Level vs Member Function

```kotlin
// Top-Level Function - ไม่ต้องอยู่ใน Class
fun topLevelFunction() {
    println("ฉันอยู่ที่ระดับ Top-Level")
}

// ใช้ได้จากทุกที่ใน package
class MyClass {
    // Member Function (Method)
    fun memberFunction() {
        println("ฉันอยู่ใน Class")
        topLevelFunction()  // เรียก top-level ได้
    }
}

fun main() {
    topLevelFunction()  // เรียกตรงได้เลย
    
    val obj = MyClass()
    obj.memberFunction()
}
```

---

## Single-Expression Functions

เมื่อ function body มีแค่ expression เดียว:

```kotlin
// รูปแบบยาว
fun double(x: Int): Int {
    return x * 2
}

// Single-Expression รูปแบบสั้น
fun double2(x: Int) = x * 2  // Type Inferred

fun triple(x: Int): Int = x * 3  // ระบุ Type ชัดเจน

fun isEven(n: Int) = n % 2 == 0
fun isPositive(n: Int) = n > 0

fun max(a: Int, b: Int) = if (a > b) a else b

fun classify(n: Int) = when {
    n > 0  -> "บวก"
    n < 0  -> "ลบ"
    else   -> "ศูนย์"
}

fun main() {
    println(double2(5))      // 10
    println(triple(4))       // 12
    println(isEven(8))       // true
    println(max(3, 7))       // 7
    println(classify(-5))    // ลบ
    
    // Single-expression ที่ซับซ้อนขึ้น
    fun celsiusToFahrenheit(c: Double) = c * 9 / 5 + 32
    fun areaOfCircle(radius: Double) = Math.PI * radius * radius
    
    println("${25}°C = ${celsiusToFahrenheit(25.0)}°F")
    println("พื้นที่วงกลม r=5: ${"%.2f".format(areaOfCircle(5.0))}")
}
```

---

## Default Parameters

กำหนดค่า default ให้ parameter:

```kotlin
// ไม่ต้องสร้าง overloaded functions หลายตัว!
fun createUser(
    name: String,
    age: Int = 0,
    email: String = "",
    role: String = "user",
    isActive: Boolean = true
): String {
    return buildString {
        appendLine("=== User Created ===")
        appendLine("Name: $name")
        if (age > 0) appendLine("Age: $age")
        if (email.isNotEmpty()) appendLine("Email: $email")
        appendLine("Role: $role")
        appendLine("Active: $isActive")
    }
}

fun main() {
    // ใช้ค่า default ทั้งหมด
    println(createUser("สมชาย"))
    
    // ระบุบางค่า
    println(createUser("สมหญิง", age = 25, email = "somying@example.com"))
    
    // ระบุทุกค่า
    println(createUser("Admin", 30, "admin@example.com", "admin", true))
    
    // Default กับ Collection
    fun makeList(
        vararg items: String,
        separator: String = ", ",
        prefix: String = "[",
        suffix: String = "]"
    ): String {
        return items.joinToString(separator, prefix, suffix)
    }
    
    println(makeList("a", "b", "c"))                              // [a, b, c]
    println(makeList("x", "y", "z", separator = " | "))          // [x | y | z]
    println(makeList("1", "2", "3", prefix = "(", suffix = ")")) // (1, 2, 3)
    
    // Default กับ Nullable
    fun log(message: String, level: String = "INFO", tag: String? = null) {
        val tagStr = tag?.let { "[$it] " } ?: ""
        println("[$level] $tagStr$message")
    }
    
    log("เริ่มต้นโปรแกรม")
    log("ข้อผิดพลาด!", "ERROR")
    log("Debug info", "DEBUG", "NetworkLayer")
    // [INFO] เริ่มต้นโปรแกรม
    // [ERROR] ข้อผิดพลาด!
    // [DEBUG] [NetworkLayer] Debug info
}
```

---

## Named Arguments

เรียกใช้ function โดยระบุชื่อ parameter:

```kotlin
fun drawRect(width: Int, height: Int, char: Char = '*', filled: Boolean = false) {
    for (row in 0 until height) {
        for (col in 0 until width) {
            if (filled || row == 0 || row == height - 1 || col == 0 || col == width - 1) {
                print(char)
            } else {
                print(" ")
            }
        }
        println()
    }
}

fun main() {
    // โดยปกติ (Positional)
    drawRect(5, 3)
    
    // Named Arguments - อ่านชัดเจนขึ้น
    drawRect(width = 5, height = 3)
    
    // Named Arguments - เปลี่ยนลำดับได้
    drawRect(height = 3, width = 5, char = '#')
    
    // ข้าม optional arguments
    drawRect(width = 6, height = 4, filled = true)
    
    // ประโยชน์: อ่านโค้ดที่ซับซ้อนได้ง่ายขึ้น
    fun sendEmail(
        to: String,
        from: String,
        subject: String,
        body: String,
        cc: String = "",
        bcc: String = "",
        isHtml: Boolean = false
    ) {
        println("From: $from")
        println("To: $to")
        if (cc.isNotEmpty()) println("CC: $cc")
        println("Subject: $subject")
        println("HTML: $isHtml")
        println("Body: $body")
    }
    
    // Named Arguments ทำให้ชัดเจน
    sendEmail(
        to = "user@example.com",
        from = "admin@example.com",
        subject = "ยืนยันการสมัครสมาชิก",
        body = "<h1>ยินดีต้อนรับ!</h1>",
        isHtml = true
    )
}
```

---

## Vararg Parameters

รับ argument จำนวนไม่กำหนด:

```kotlin
// vararg: รับหลาย argument เป็น Array
fun sum(vararg numbers: Int): Int {
    return numbers.sum()
}

fun printAll(vararg items: Any?) {
    items.forEach { println(it) }
}

fun <T> listOf(vararg elements: T): List<T> {
    return elements.toList()
}

fun main() {
    println(sum(1, 2, 3))         // 6
    println(sum(1, 2, 3, 4, 5))  // 15
    println(sum())                 // 0
    
    printAll("Hello", 42, true, null, 3.14)
    
    // ใช้ Spread Operator (*) ส่ง Array เข้า vararg
    val numbers = intArrayOf(1, 2, 3, 4, 5)
    println(sum(*numbers))  // 15
    
    val moreNumbers = intArrayOf(6, 7, 8)
    println(sum(*numbers, *moreNumbers))  // 36
    
    // vararg กับ parameter อื่น (vararg ต้องอยู่สุดท้าย)
    fun format(template: String, vararg args: Any?): String {
        var result = template
        args.forEachIndexed { i, arg ->
            result = result.replace("{$i}", arg.toString())
        }
        return result
    }
    
    println(format("สวัสดี {0}! คุณอายุ {1} ปี", "สมชาย", 25))
    // สวัสดี สมชาย! คุณอายุ 25 ปี
    
    // ตัวอย่างจริง: logging
    fun log(level: String, message: String, vararg context: Pair<String, Any?>) {
        println("[$level] $message")
        context.forEach { (key, value) ->
            println("  $key: $value")
        }
    }
    
    log("ERROR", "Database connection failed",
        "host" to "localhost",
        "port" to 5432,
        "database" to "mydb",
        "attempts" to 3
    )
}
```

---

## Return Types และ Unit

```kotlin
// Unit = ไม่ return ค่า (เหมือน void ใน Java)
fun printHello(): Unit {
    println("Hello!")
}

// ไม่ต้องใส่ : Unit ก็ได้ (default)
fun printHello2() {
    println("Hello!")
}

// Nothing = ฟังก์ชันไม่ return เลย (throw หรือ loop ไม่สิ้นสุด)
fun fail(message: String): Nothing {
    throw IllegalStateException(message)
}

fun infiniteLoop(): Nothing {
    while (true) {
        // ไม่มีทางออก
    }
}

fun main() {
    // Unit
    val result: Unit = printHello()
    println(result)  // kotlin.Unit
    
    // Nothing ใช้ใน type system
    fun getUser(id: Int): String {
        if (id <= 0) fail("Invalid ID: $id")  // type: Nothing
        return "User #$id"  // type: String
        // Compiler รู้ว่า fail() ไม่ return
        // ดังนั้น function นี้ return String เสมอ
    }
    
    println(getUser(1))  // User #1
    
    try {
        getUser(-1)
    } catch (e: IllegalStateException) {
        println("Error: ${e.message}")  // Error: Invalid ID: -1
    }
    
    // ใช้ Nothing กับ Elvis
    fun requirePositive(n: Int): Int {
        return if (n > 0) n else throw IllegalArgumentException("Must be positive: $n")
    }
    
    println(requirePositive(5))  // 5
}
```

---

## Local Functions

ฟังก์ชันที่ประกาศภายในฟังก์ชันอื่น:

```kotlin
fun processData(data: List<Int>): List<String> {
    // Local function - เข้าถึงได้เฉพาะใน processData
    fun validate(n: Int): Boolean {
        return n > 0 && n < 1000
    }
    
    fun format(n: Int): String {
        return when {
            n < 10   -> "00$n"
            n < 100  -> "0$n"
            else     -> "$n"
        }
    }
    
    return data
        .filter { validate(it) }
        .map { format(it) }
}

fun main() {
    val result = processData(listOf(-5, 3, 42, 1001, 7, 123, 0))
    println(result)  // [003, 042, 007, 123]
    
    // Local function เข้าถึง outer scope ได้ (Closure)
    fun makeCounter(start: Int = 0): () -> Int {
        var count = start
        
        fun increment(): Int {
            count++
            return count
        }
        
        return ::increment
    }
    
    val counter1 = makeCounter()
    val counter2 = makeCounter(100)
    
    println(counter1())  // 1
    println(counter1())  // 2
    println(counter2())  // 101
    println(counter1())  // 3
    println(counter2())  // 102
    
    // Local function สำหรับ Recursive จำนวนมาก
    fun factorial(n: Int): Long {
        fun factHelper(n: Int, acc: Long): Long {
            return if (n <= 1) acc else factHelper(n - 1, acc * n)
        }
        return factHelper(n, 1)
    }
    
    println("10! = ${factorial(10)}")  // 3628800
    println("15! = ${factorial(15)}")  // 1307674368000
}
```

---

## Extension Functions

เพิ่ม function ให้ class ที่มีอยู่แล้วโดยไม่ต้อง inherit:

```kotlin
// เพิ่ม function ให้ String
fun String.isPalindrome(): Boolean {
    val clean = this.lowercase().filter { it.isLetterOrDigit() }
    return clean == clean.reversed()
}

fun String.toTitleCase(): String {
    return this.split(" ")
        .joinToString(" ") { word ->
            word.replaceFirstChar { it.uppercase() }
        }
}

fun String.truncate(maxLength: Int, suffix: String = "..."): String {
    return if (this.length <= maxLength) this
    else this.take(maxLength - suffix.length) + suffix
}

// เพิ่ม function ให้ Int
fun Int.isPrime(): Boolean {
    if (this < 2) return false
    if (this == 2) return true
    if (this % 2 == 0) return false
    for (i in 3..Math.sqrt(this.toDouble()).toInt() step 2) {
        if (this % i == 0) return false
    }
    return true
}

fun Int.factorial(): Long {
    require(this >= 0) { "Factorial ต้องเป็นจำนวนบวก" }
    return if (this <= 1) 1L else this.toLong() * (this - 1).factorial()
}

// เพิ่ม function ให้ List<Int>
fun List<Int>.median(): Double {
    val sorted = this.sorted()
    val mid = sorted.size / 2
    return if (sorted.size % 2 == 0) {
        (sorted[mid - 1] + sorted[mid]) / 2.0
    } else {
        sorted[mid].toDouble()
    }
}

fun main() {
    // String extensions
    println("racecar".isPalindrome())          // true
    println("A man a plan a canal Panama".isPalindrome())  // true
    println("hello world".toTitleCase())        // Hello World
    println("This is a very long text".truncate(15))  // This is a ve...
    
    // Int extensions
    println(17.isPrime())   // true
    println(18.isPrime())   // false
    println(5.factorial())  // 120
    println(10.factorial()) // 3628800
    
    // List extensions
    val nums = listOf(5, 2, 8, 1, 9, 3, 7, 4, 6)
    println("Median: ${nums.median()}")  // Median: 5.0
    
    // Extension บน Nullable
    fun String?.orDefault(default: String = "N/A"): String {
        return this ?: default
    }
    
    val nullStr: String? = null
    println(nullStr.orDefault())         // N/A
    println(nullStr.orDefault("Unknown")) // Unknown
    println("Hello".orDefault())          // Hello
}
```

---

## Infix Functions

ฟังก์ชันที่เรียกได้แบบ infix (ไม่ต้องใส่ dot และวงเล็บ):

```kotlin
// infix function ต้องเป็น member หรือ extension
// มี parameter เดียว

infix fun Int.plus(other: Int): Int = this + other

data class Point(val x: Int, val y: Int) {
    infix fun to(other: Point): String {
        return "($x, $y) → (${other.x}, ${other.y})"
    }
    
    infix fun distanceTo(other: Point): Double {
        val dx = (this.x - other.x).toDouble()
        val dy = (this.y - other.y).toDouble()
        return Math.sqrt(dx * dx + dy * dy)
    }
}

// String extension infix
infix fun String.containsWord(word: String): Boolean {
    return this.split("\\s+".toRegex()).any { it == word }
}

// Bitwise infix
infix fun Int.bitwiseOr(other: Int) = this or other
infix fun Int.bitwiseAnd(other: Int) = this and other

fun main() {
    // ใช้แบบ infix
    println(5 plus 3)  // 8 (เหมือน 5.plus(3))
    
    val p1 = Point(0, 0)
    val p2 = Point(3, 4)
    
    println(p1 to p2)             // (0, 0) → (3, 4)
    println(p1 distanceTo p2)     // 5.0
    
    val sentence = "The quick brown fox"
    println(sentence containsWord "fox")    // true
    println(sentence containsWord "dog")    // false
    
    // สร้าง DSL-like API
    infix fun String.shouldBe(expected: String): Boolean {
        val result = this == expected
        if (!result) println("Expected: '$expected' but was: '$this'")
        return result
    }
    
    "Hello" shouldBe "Hello"  // true
    "Hello" shouldBe "World"  // Expected: 'World' but was: 'Hello'
    
    // Standard Library ใช้ infix ด้วย
    val map = mapOf(1 to "หนึ่ง", 2 to "สอง", 3 to "สาม")  // to เป็น infix!
    println(map)  // {1=หนึ่ง, 2=สอง, 3=สาม}
}
```

---

## Tail Recursive Functions

Kotlin รองรับ `tailrec` modifier สำหรับ Tail Recursion Optimization:

```kotlin
// tailrec: Compiler แปลง recursion เป็น loop อัตโนมัติ
// ป้องกัน StackOverflowError สำหรับ recursion ลึก

// ❌ recursion ปกติ - อาจเกิด StackOverflow
fun factorialNormal(n: Int): Long {
    return if (n <= 1) 1L else n.toLong() * factorialNormal(n - 1)
}

// ✅ tail recursion
tailrec fun factorialTail(n: Int, acc: Long = 1L): Long {
    return if (n <= 1) acc else factorialTail(n - 1, acc * n)
}

// หา Fibonacci แบบ tail recursive
tailrec fun fibTail(n: Int, a: Long = 0L, b: Long = 1L): Long {
    return when (n) {
        0 -> a
        1 -> b
        else -> fibTail(n - 1, b, a + b)
    }
}

// ค้นหาแบบ binary search
tailrec fun binarySearch(
    arr: List<Int>,
    target: Int,
    low: Int = 0,
    high: Int = arr.lastIndex
): Int {
    if (low > high) return -1
    
    val mid = (low + high) / 2
    return when {
        arr[mid] == target -> mid
        arr[mid] < target  -> binarySearch(arr, target, mid + 1, high)
        else               -> binarySearch(arr, target, low, mid - 1)
    }
}

fun main() {
    println(factorialTail(20))   // 2432902008176640000
    println(fibTail(50))         // 12586269025
    
    val sorted = listOf(1, 3, 5, 7, 9, 11, 13, 15, 17, 19)
    println(binarySearch(sorted, 7))   // 3 (index)
    println(binarySearch(sorted, 12))  // -1 (ไม่พบ)
    
    // tail recursion สามารถรับ depth ลึกมากได้
    tailrec fun countDown(n: Int) {
        if (n <= 0) {
            println("🚀 Launch!")
            return
        }
        print("$n... ")
        countDown(n - 1)
    }
    
    countDown(10)  // 10... 9... 8... 7... 6... 5... 4... 3... 2... 1... 🚀 Launch!
}
```

---

## Inline Functions (เบื้องต้น)

```kotlin
// inline: Compiler copy-pastes body ของ function แทนที่จะ call
// ลด overhead ของ function call โดยเฉพาะ higher-order functions

inline fun measureTime(block: () -> Unit): Long {
    val start = System.currentTimeMillis()
    block()
    return System.currentTimeMillis() - start
}

inline fun <T> withLogging(tag: String, block: () -> T): T {
    println("[$tag] Started")
    val result = block()
    println("[$tag] Completed → $result")
    return result
}

// noinline: ป้องกัน inline สำหรับ parameter บางตัว
inline fun doSomething(
    inlineBlock: () -> Unit,
    noinline noInlineBlock: () -> Unit
) {
    inlineBlock()
    noSomethingElse(noInlineBlock)  // ส่ง function reference ได้
}

fun noSomethingElse(block: () -> Unit) = block()

fun main() {
    val time = measureTime {
        Thread.sleep(100)
        println("Processing...")
    }
    println("ใช้เวลา: $time ms")
    
    val result = withLogging("Calculator") {
        (1..100).sum()
    }
    println("Result: $result")
    
    // crossinline: ห้าม non-local return
    inline fun runTasks(crossinline task: () -> Unit) {
        val thread = Thread { task() }
        thread.start()
        thread.join()
    }
    
    runTasks {
        println("Running in thread!")
        // return ที่นี่จะ return จาก lambda เท่านั้น ไม่ใช่ runTasks
    }
}
```

---

## ตัวอย่างโปรแกรมสรุปรวม

### Library ฟังก์ชันคณิตศาสตร์

```kotlin
import kotlin.math.*

// ======= MATH FUNCTIONS LIBRARY =======

// ฟังก์ชันพื้นฐาน
fun factorial(n: Int): Long {
    require(n >= 0) { "n ต้องเป็น 0 หรือมากกว่า" }
    return if (n <= 1) 1L else n.toLong() * factorial(n - 1)
}

fun fibonacci(n: Int): Long {
    require(n >= 0) { "n ต้องเป็น 0 หรือมากกว่า" }
    tailrec fun fib(n: Int, a: Long = 0L, b: Long = 1L): Long =
        if (n == 0) a else fib(n - 1, b, a + b)
    return fib(n)
}

fun gcd(a: Int, b: Int): Int = if (b == 0) a else gcd(b, a % b)
fun lcm(a: Int, b: Int): Long = a.toLong() * b / gcd(a, b)

// ฟังก์ชัน Extension
fun Int.isPrime(): Boolean {
    if (this < 2) return false
    if (this == 2) return true
    if (this % 2 == 0) return false
    return (3..sqrt(this.toDouble()).toInt() step 2).none { this % it == 0 }
}

fun Int.digitSum(): Int = this.toString().sumOf { it.digitToInt() }
fun Int.digits(): List<Int> = this.toString().map { it.digitToInt() }
fun Int.isPerfect(): Boolean = (1 until this).filter { this % it == 0 }.sum() == this

// ฟังก์ชันทางสถิติ
fun List<Double>.mean() = average()
fun List<Double>.variance(): Double {
    val m = mean()
    return map { (it - m).pow(2) }.average()
}
fun List<Double>.stdDev() = sqrt(variance())
fun List<Double>.median(): Double {
    val sorted = sorted()
    val mid = size / 2
    return if (size % 2 == 0) (sorted[mid-1] + sorted[mid]) / 2.0 else sorted[mid]
}

fun main() {
    println("=== Math Library Demo ===\n")
    
    // Factorial
    println("Factorials:")
    (0..10).forEach { n -> print("$n!=${factorial(n)} ") }
    println("\n")
    
    // Fibonacci
    println("Fibonacci:")
    (0..15).forEach { n -> print("${fibonacci(n)} ") }
    println("\n")
    
    // GCD & LCM
    println("GCD(48, 18) = ${gcd(48, 18)}")
    println("LCM(4, 6) = ${lcm(4, 6)}")
    println()
    
    // Prime Numbers
    val primes = (1..50).filter { it.isPrime() }
    println("Primes 1-50: $primes")
    println()
    
    // Digit Operations
    println("1234 digit sum: ${1234.digitSum()}")    // 10
    println("9999 digits: ${9999.digits()}")          // [9, 9, 9, 9]
    println("6 is perfect: ${6.isPerfect()}")         // true (1+2+3=6)
    println("28 is perfect: ${28.isPerfect()}")       // true (1+2+4+7+14=28)
    println()
    
    // Statistics
    val data = listOf(2.0, 4.0, 4.0, 4.0, 5.0, 5.0, 7.0, 9.0)
    println("Data: $data")
    println("Mean: ${data.mean()}")
    println("Variance: ${"%.4f".format(data.variance())}")
    println("StdDev: ${"%.4f".format(data.stdDev())}")
    println("Median: ${data.median()}")
}
```

---

## แบบฝึกหัด

### Exercise 1: เครื่องคิดเลขสถิติ

```kotlin
fun statistics(vararg numbers: Double): Map<String, Double> {
    val list = numbers.toList()
    return mapOf(
        "count" to list.size.toDouble(),
        "sum" to list.sum(),
        "min" to list.min(),
        "max" to list.max(),
        "mean" to list.average(),
        "range" to list.max() - list.min()
    )
}

fun main() {
    val stats = statistics(5.0, 3.0, 8.0, 1.0, 9.0, 4.0, 7.0, 2.0, 6.0)
    println("Statistics:")
    stats.forEach { (key, value) ->
        println("  $key: ${"%.2f".format(value)}")
    }
}
```

### Exercise 2: String Utilities

```kotlin
fun String.wordCount(): Map<String, Int> {
    return this.lowercase()
        .replace("[^a-zA-Z0-9ก-ฮ\\s]".toRegex(), "")
        .split("\\s+".toRegex())
        .filter { it.isNotEmpty() }
        .groupBy { it }
        .mapValues { it.value.size }
}

fun String.mostFrequentWord(): String? {
    return wordCount().maxByOrNull { it.value }?.key
}

fun main() {
    val text = "the quick brown fox jumps over the lazy dog the fox"
    val wordCounts = text.wordCount()
    
    println("Word counts:")
    wordCounts.entries.sortedByDescending { it.value }.forEach { (word, count) ->
        println("  '$word': $count")
    }
    println("Most frequent: '${text.mostFrequentWord()}'")
}
```

### Exercise 3: ท้าทาย - Function Composition

```kotlin
// Function composition - รวม functions เข้าด้วยกัน
fun <A, B, C> compose(f: (B) -> C, g: (A) -> B): (A) -> C = { f(g(it)) }

infix fun <A, B, C> ((B) -> C).after(g: (A) -> B): (A) -> C = { this(g(it)) }

fun main() {
    val double = { x: Int -> x * 2 }
    val addOne = { x: Int -> x + 1 }
    val square = { x: Int -> x * x }
    
    val doubleThenAddOne = compose(addOne, double)  // addOne(double(x))
    val addOneThenDouble = compose(double, addOne)  // double(addOne(x))
    
    println(doubleThenAddOne(5))  // (5*2)+1 = 11
    println(addOneThenDouble(5))  // (5+1)*2 = 12
    
    // ใช้ infix after
    val transform = square after addOne after double  // square(addOne(double(x)))
    println(transform(3))  // square(addOne(3*2)) = square(7) = 49
    
    // Pipeline
    val process = listOf<(Int) -> Int>(double, addOne, square)
    val pipeline = process.reduce { acc, fn -> compose(fn, acc) }
    
    println(pipeline(3))  // square(addOne(double(3))) = square(addOne(6)) = square(7) = 49
}
```

---

## สรุป Part 07

```
✅ fun keyword สำหรับประกาศฟังก์ชัน
✅ Single-expression function ด้วย =
✅ Default parameters ลด overloading
✅ Named arguments ทำให้โค้ดอ่านง่าย
✅ vararg รับ arguments ไม่จำกัด
✅ Unit = void, Nothing = ไม่ return
✅ Local functions ภายใน function อื่น
✅ Extension functions เพิ่ม method ให้ class
✅ infix functions สำหรับ DSL-like syntax
✅ tailrec สำหรับ Tail Recursion Optimization
✅ inline functions ลด overhead
```

---

*Part 07/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
