# Part 02: ตัวแปรและชนิดข้อมูล (Variables and Data Types)

## สารบัญ
1. [val vs var](#val-vs-var)
2. [Type Inference](#type-inference)
3. [ชนิดข้อมูลตัวเลข](#ชนิดข้อมูลตัวเลข)
4. [Boolean](#boolean)
5. [Char](#char)
6. [String](#string)
7. [Type Conversion](#type-conversion)
8. [ค่าคงที่ (Constants)](#ค่าคงที่-constants)
9. [Nullable Types เบื้องต้น](#nullable-types-เบื้องต้น)
10. [การตั้งชื่อตัวแปร](#การตั้งชื่อตัวแปร)
11. [แบบฝึกหัด](#แบบฝึกหัด)

---

## val vs var

ใน Kotlin มีคำสำคัญ 2 คำสำหรับประกาศตัวแปร:

### val (Value) - ค่าที่เปลี่ยนไม่ได้

```kotlin
val pi = 3.14159
val greeting = "สวัสดี"
val maxSize = 100

// ❌ ไม่สามารถเปลี่ยนค่าได้หลังจากกำหนดแล้ว
pi = 3.14  // Error: Val cannot be reassigned
```

`val` คล้ายกับ `final` ใน Java - กำหนดค่าได้ครั้งเดียว

### var (Variable) - ค่าที่เปลี่ยนได้

```kotlin
var score = 0
var playerName = "ผู้เล่น"
var isAlive = true

// ✅ สามารถเปลี่ยนค่าได้
score = 100
playerName = "สมชาย"
isAlive = false

println("คะแนน: $score")       // คะแนน: 100
println("ผู้เล่น: $playerName") // ผู้เล่น: สมชาย
println("มีชีวิต: $isAlive")    // มีชีวิต: false
```

### เมื่อไหรใช้ val และ var?

```kotlin
// ✅ ใช้ val เมื่อค่าไม่ต้องการเปลี่ยน (แนะนำ ใช้บ่อยกว่า)
val userName = "admin"
val baseUrl = "https://api.example.com"
val maxRetry = 3

// ✅ ใช้ var เมื่อต้องการเปลี่ยนค่า
var currentPage = 1
var isLoading = false
var errorMessage = ""

// กฎทั่วไป: ใช้ val เป็นค่าเริ่มต้น เปลี่ยนเป็น var เฉพาะเมื่อจำเป็น
```

**ทำไมต้องใช้ val เป็นหลัก?**
- ป้องกัน Bug จากการเปลี่ยนค่าโดยไม่ตั้งใจ
- ทำให้โค้ดอ่านและ Debug ง่ายขึ้น
- เหมาะกับ Functional Programming
- Thread-safe โดยธรรมชาติ

---

## Type Inference

Kotlin สามารถ **อนุมาน (Infer)** ชนิดข้อมูลได้เองจากค่าที่กำหนด ทำให้ไม่ต้องระบุ Type เสมอไป:

```kotlin
// แบบที่ 1: Type Inference (ปล่อยให้ Kotlin อนุมานเอง)
val name = "Kotlin"     // String
val age = 25            // Int
val height = 1.75       // Double
val isStudent = true    // Boolean

// แบบที่ 2: ระบุ Type ชัดเจน (Explicit Type)
val name: String = "Kotlin"
val age: Int = 25
val height: Double = 1.75
val isStudent: Boolean = true
```

**เมื่อไหรควรระบุ Type ชัดเจน?**

```kotlin
// 1. เมื่อประกาศตัวแปรแต่ยังไม่กำหนดค่า
var score: Int  // ต้องระบุ Type เพราะยังไม่มีค่าให้ Infer
// ... 
score = 100

// 2. เมื่อต้องการ Type ที่ต่างจาก Default
val number = 42        // Default: Int
val bigNumber: Long = 42  // ต้องการ Long
val smallNumber: Byte = 42  // ต้องการ Byte

// 3. เมื่อต้องการความชัดเจนในโค้ด API/Library
fun processUser(userId: Long): String { ... }
val id: Long = 12345L  // ชัดเจนว่าเป็น Long

// 4. เมื่อ Type เป็น Interface หรือ Supertype
val list: List<String> = mutableListOf("a", "b", "c")  // ใช้ List ไม่ใช่ MutableList
```

### ตรวจสอบ Type ของตัวแปร

```kotlin
val x = 42
val y = 3.14
val z = "Hello"

println(x::class.simpleName)  // Int
println(y::class.simpleName)  // Double
println(z::class.simpleName)  // String

// ใช้ is operator ตรวจสอบ type
println(x is Int)     // true
println(y is Double)  // true
println(z is String)  // true
println(x is Long)    // false
```

---

## ชนิดข้อมูลตัวเลข

### จำนวนเต็ม (Integer Types)

```
┌──────────┬────────┬───────────────────────────────────────────────────────────┐
│ Type     │ Size   │ Range                                                     │
├──────────┼────────┼───────────────────────────────────────────────────────────┤
│ Byte     │ 8 bit  │ -128 ถึง 127                                              │
│ Short    │ 16 bit │ -32,768 ถึง 32,767                                        │
│ Int      │ 32 bit │ -2,147,483,648 ถึง 2,147,483,647 (~2 พันล้าน)            │
│ Long     │ 64 bit │ -9,223,372,036,854,775,808 ถึง 9,223,372,036,854,775,807 │
└──────────┴────────┴───────────────────────────────────────────────────────────┘
```

```kotlin
// Byte
val byteValue: Byte = 127
val minByte: Byte = -128
println("Byte max: ${Byte.MAX_VALUE}")  // 127
println("Byte min: ${Byte.MIN_VALUE}")  // -128

// Short
val shortValue: Short = 32767
println("Short max: ${Short.MAX_VALUE}")  // 32767

// Int (ใช้บ่อยที่สุด)
val age = 25              // Type Inference: Int
val population = 70_000_000  // ใช้ _ เพื่อความอ่านง่าย
val hexValue = 0xFF           // Hexadecimal = 255
val binaryValue = 0b1010_1010 // Binary = 170
val octalValue = 0o17         // ❌ ไม่มี Octal ใน Kotlin

println("Int max: ${Int.MAX_VALUE}")    // 2147483647
println("population: $population")      // 70000000
println("hex 0xFF = $hexValue")         // hex 0xFF = 255
println("binary 0b10101010 = $binaryValue") // 170

// Long - ใช้ L suffix
val longValue = 9_000_000_000L  // ต้องใส่ L
val worldPopulation: Long = 8_000_000_000L
println("Long max: ${Long.MAX_VALUE}")  // 9223372036854775807
println("worldPopulation: $worldPopulation")  // 8000000000
```

### จำนวนทศนิยม (Floating-Point Types)

```
┌──────────┬────────┬────────────────────────────────────────────────────────┐
│ Type     │ Size   │ Precision                                              │
├──────────┼────────┼────────────────────────────────────────────────────────┤
│ Float    │ 32 bit │ ~6-7 decimal digits (ไม่แม่นยำมาก ไม่แนะนำ)            │
│ Double   │ 64 bit │ ~15-16 decimal digits (แนะนำ - ใช้เป็น default)        │
└──────────┴────────┴────────────────────────────────────────────────────────┘
```

```kotlin
// Double (default สำหรับทศนิยม)
val pi = 3.14159265358979    // Type Inference: Double
val temperature = -15.5
val price = 1999.99

println("pi = $pi")          // pi = 3.14159265358979
println("temp = $temperature") // temp = -15.5

// Float - ต้องใส่ F suffix
val weight: Float = 65.5f    // หรือ 65.5F
val floatPi = 3.14f
println("Float pi = $floatPi")  // Float pi = 3.14

// Scientific Notation
val lightSpeed = 3.0e8    // 3.0 × 10^8 = 300,000,000
val electronMass = 9.1e-31
println("Speed of light: $lightSpeed")  // 3.0E8
println("Electron mass: $electronMass") // 9.1E-31

// ข้อควรระวัง: Floating Point Precision
val a = 0.1 + 0.2
println(a)          // 0.30000000000000004 (ไม่ใช่ 0.3!)
println(a == 0.3)   // false!

// วิธีแก้: ใช้ BigDecimal สำหรับการเงิน
import java.math.BigDecimal
val x = BigDecimal("0.1")
val y = BigDecimal("0.2")
println(x + y)  // 0.3 ✅
```

### การแสดงผลตัวเลข

```kotlin
val number = 1234567.89

// ใช้ String.format
println(String.format("%,f", number))     // 1,234,567.890000
println(String.format("%.2f", number))    // 1234567.89
println(String.format("%,.2f", number))   // 1,234,567.89
println(String.format("%10.2f", number))  //  1234567.89 (pad ด้วย space)
println(String.format("%010.2f", number)) // 01234567.89 (pad ด้วย 0)

// ใช้ formatString extension
val formattedPrice = "%.2f".format(number)
println("ราคา: $formattedPrice บาท")  // ราคา: 1234567.89 บาท

// Kotlin ไม่มี built-in number formatting ที่ง่ายเหมือน Python
// ใช้ java.text.NumberFormat
import java.text.NumberFormat
import java.util.Locale

val formatter = NumberFormat.getNumberInstance(Locale("th", "TH"))
println(formatter.format(1234567.89))  // 1,234,567.89
```

---

## Boolean

```kotlin
// ประกาศ Boolean
val isKotlinFun = true
val isBoring = false

println(isKotlinFun)  // true
println(isBoring)     // false

// Boolean Operations
val a = true
val b = false

println(a && b)   // AND: false
println(a || b)   // OR: true
println(!a)       // NOT: false
println(a xor b)  // XOR: true (ต่างกัน = true)

// Short-circuit evaluation
val x = 10
val y = 0

// ถ้า x > 5 เป็น false จะไม่คำนวณ x / y (ป้องกัน Division by Zero)
if (x > 5 && x / y > 1) {  // ❌ อาจเกิด ArithmeticException
    println("condition met")
}

// แก้ไข: ตรวจสอบ y != 0 ก่อน
if (y != 0 && x / y > 1) {  // ✅ ถ้า y == 0 จะไม่คำนวณ x / y
    println("condition met")
}

// Boolean กับ when
val score = 85
val grade = when {
    score >= 90 -> "A"
    score >= 80 -> "B"
    score >= 70 -> "C"
    score >= 60 -> "D"
    else        -> "F"
}
println("เกรด: $grade")  // เกรด: B
```

---

## Char

`Char` แทนอักขระเดี่ยว ใช้ single quote `'`:

```kotlin
// ประกาศ Char
val letter = 'A'
val digit = '9'
val thaiChar = 'ก'
val space = ' '

println(letter)    // A
println(digit)     // 9
println(thaiChar)  // ก

// Escape Sequences
val newline = '\n'    // ขึ้นบรรทัดใหม่
val tab = '\t'        // Tab
val backslash = '\\'  // \
val singleQuote = '\'' // '
val unicodeA = '\u0041' // Unicode: A

println("Hello\tWorld\nNew Line")
// Hello   World
// New Line

// Char Operations
val ch = 'A'
println(ch.code)           // 65 (ASCII/Unicode value)
println(ch + 1)            // 66 (Int)
println((ch + 1).toChar()) // B
println(ch.lowercaseChar()) // a
println(ch.isLetter())     // true
println(ch.isDigit())      // false
println(ch.isUpperCase())  // true

// วนซ้ำตัวอักษร
for (c in 'A'..'Z') {
    print("$c ")
}
// A B C D E F G H I J K L M N O P Q R S T U V W X Y Z

println()

// ตรวจสอบตัวอักษร
val input = 'k'
when {
    input.isUpperCase() -> println("$input คือตัวพิมพ์ใหญ่")
    input.isLowerCase() -> println("$input คือตัวพิมพ์เล็ก")
    input.isDigit()     -> println("$input คือตัวเลข")
    else                -> println("$input เป็นอักขระพิเศษ")
}
// k คือตัวพิมพ์เล็ก
```

---

## String

`String` ใช้เก็บข้อความ ใช้ double quote `"`:

### การสร้าง String

```kotlin
// String ทั่วไป
val str1 = "Hello, Kotlin!"
val str2 = "สวัสดี ชาว Kotlin"

// Raw String (Multiline) ใช้ triple-quote """
val address = """
    123 ถนนสุขุมวิท
    แขวงคลองเตย
    เขตคลองเตย
    กรุงเทพฯ 10110
""".trimIndent()  // ตัด indent นำหน้า

println(address)
// 123 ถนนสุขุมวิท
// แขวงคลองเตย
// เขตคลองเตย
// กรุงเทพฯ 10110

// Raw String ไม่ต้อง escape
val json = """
    {
        "name": "สมชาย",
        "age": 25,
        "path": "C:\Users\somchai"
    }
""".trimIndent()
println(json)
```

### String Template

```kotlin
val name = "สมชาย"
val age = 25
val salary = 50000.0

// Simple Template: $variableName
println("ชื่อ: $name")          // ชื่อ: สมชาย
println("อายุ: $age ปี")        // อายุ: 25 ปี

// Expression Template: ${expression}
println("ปีเกิด: ${2024 - age}")  // ปีเกิด: 1999
println("เงินเดือน: ${salary.toInt()} บาท")  // เงินเดือน: 50000 บาท
println("ชื่อ มี ${name.length} ตัวอักษร")   // ชื่อ มี 6 ตัวอักษร

// Template ใน Raw String
val report = """
    ===== รายงาน =====
    ชื่อ: $name
    อายุ: $age ปี
    เงินเดือน: ${String.format("%,.0f", salary)} บาท
    สถานะ: ${if (age >= 18) "ผู้ใหญ่" else "ผู้เยาว์"}
""".trimIndent()
println(report)
```

**Output:**
```
===== รายงาน =====
ชื่อ: สมชาย
อายุ: 25 ปี
เงินเดือน: 50,000 บาท
สถานะ: ผู้ใหญ่
```

### String Operations

```kotlin
val str = "Hello, Kotlin World!"

// ความยาว
println(str.length)          // 20

// การเข้าถึงตัวอักษร
println(str[0])              // H
println(str[7])              // K
println(str.first())         // H
println(str.last())          // !

// Substring
println(str.substring(7))        // Kotlin World!
println(str.substring(7, 13))    // Kotlin
println(str.take(5))             // Hello
println(str.takeLast(6))         // World!
println(str.drop(7))             // Kotlin World!
println(str.dropLast(7))         // Hello, Kotlin

// Case Conversion
println(str.uppercase())     // HELLO, KOTLIN WORLD!
println(str.lowercase())     // hello, kotlin world!
println("hello".capitalize())  // Hello (Deprecated ใน Kotlin 1.5)
println("hello WORLD".replaceFirstChar { it.uppercase() })  // Hello WORLD

// Trim Whitespace
val padded = "   Hello Kotlin   "
println(padded.trim())       // "Hello Kotlin"
println(padded.trimStart())  // "Hello Kotlin   "
println(padded.trimEnd())    // "   Hello Kotlin"

// Search
println(str.contains("Kotlin"))    // true
println(str.contains("Python"))    // false
println(str.startsWith("Hello"))   // true
println(str.endsWith("!"))         // true
println(str.indexOf("o"))          // 4
println(str.lastIndexOf("o"))      // 17

// Replace
println(str.replace("World", "Universe"))  // Hello, Kotlin Universe!
println(str.replace("l", "L"))             // HeLLo, KotLin WorLd!
println(str.replaceFirst("l", "L"))        // HeLlo, Kotlin World!

// Split
val csv = "แอปเปิ้ล,กล้วย,ส้ม,มะม่วง"
val fruits = csv.split(",")
println(fruits)              // [แอปเปิ้ล, กล้วย, ส้ม, มะม่วง]
println(fruits.size)         // 4
println(fruits[2])           // ส้ม

// Join
val joined = fruits.joinToString(" | ")
println(joined)              // แอปเปิ้ล | กล้วย | ส้ม | มะม่วง

// Check Empty/Blank
val empty = ""
val blank = "   "
println(empty.isEmpty())    // true
println(empty.isBlank())    // true
println(blank.isEmpty())    // false
println(blank.isBlank())    // true ← (มีแต่ space ถือว่า blank)

// Comparison
val s1 = "apple"
val s2 = "Apple"
println(s1 == s2)                    // false (case sensitive)
println(s1.equals(s2, ignoreCase = true))  // true
println(s1.compareTo(s2))            // 32 (ต่างกัน 32 ใน ASCII)
```

### StringBuilder - สร้าง String ประสิทธิภาพสูง

```kotlin
// ❌ ไม่ดี: สร้าง String ใหม่ทุกครั้ง (Slow สำหรับ String ใหญ่)
var result = ""
for (i in 1..1000) {
    result += i.toString()  // สร้าง String ใหม่ 1000 ครั้ง!
}

// ✅ ดีกว่า: ใช้ StringBuilder
val sb = StringBuilder()
for (i in 1..1000) {
    sb.append(i)
}
val result2 = sb.toString()

// ตัวอย่างการใช้ buildString (Kotlin idiomatic)
val message = buildString {
    append("สวัสดี")
    append(" ")
    append("Kotlin")
    appendLine()
    append("ยินดีต้อนรับ!")
}
println(message)
// สวัสดี Kotlin
// ยินดีต้อนรับ!

// สร้าง HTML
val html = buildString {
    appendLine("<html>")
    appendLine("  <body>")
    appendLine("    <h1>Hello Kotlin</h1>")
    appendLine("  </body>")
    append("</html>")
}
println(html)
```

---

## Type Conversion

ใน Kotlin ไม่มี Implicit Type Conversion (การแปลง Type อัตโนมัติ) ต้องแปลงเองทั้งหมด:

### Numeric Conversion

```kotlin
// ❌ Java อนุญาตให้ทำได้ (Implicit Widening)
// int x = 5;
// long y = x;  ← Java อนุญาต!

// ❌ Kotlin ไม่อนุญาต!
val intVal: Int = 42
// val longVal: Long = intVal  // Error: Type mismatch

// ✅ ต้องแปลงเองด้วย toXxx()
val intVal2: Int = 42
val longVal: Long = intVal2.toLong()
val doubleVal: Double = intVal2.toDouble()
val floatVal: Float = intVal2.toFloat()
val byteVal: Byte = intVal2.toByte()
val shortVal: Short = intVal2.toShort()

println("Int: $intVal2")        // 42
println("Long: $longVal")       // 42
println("Double: $doubleVal")   // 42.0
println("Float: $floatVal")     // 42.0

// แปลง Double เป็น Int (ตัดทศนิยม ไม่ปัดเศษ)
val pi = 3.14159
val piInt = pi.toInt()
println("pi as Int: $piInt")   // 3 (ตัดทศนิยม)

// ปัดเศษต้องใช้ roundToInt()
import kotlin.math.roundToInt
val rounded = pi.roundToInt()
println("pi rounded: $rounded")  // 3

val almost4 = 3.7
println(almost4.toInt())            // 3 (ตัดทศนิยม)
println(almost4.roundToInt())       // 4 (ปัดเศษ)
println(Math.round(almost4))        // 4

// การแปลงที่อาจสูญเสียข้อมูล
val bigLong: Long = 9_000_000_000L
val toInt = bigLong.toInt()
println("$bigLong as Int: $toInt")  // 410065408 (Overflow!)
```

### String Conversion

```kotlin
// แปลงเป็น String
val num = 42
val dbl = 3.14
val bool = true

println(num.toString())    // "42"
println(dbl.toString())    // "3.14"
println(bool.toString())   // "true"

// แปลงจาก String (อาจ throw NumberFormatException)
val strNum = "123"
val strDbl = "3.14"
val strBool = "true"

val parsedInt = strNum.toInt()          // 123
val parsedDouble = strDbl.toDouble()    // 3.14
val parsedBool = strBool.toBoolean()    // true

println(parsedInt + 1)     // 124
println(parsedDouble + 1)  // 4.140000000000001
println(!parsedBool)       // false

// Safe Parsing - ใช้ toIntOrNull() ป้องกัน Exception
val valid = "456"
val invalid = "abc"

val result1 = valid.toIntOrNull()    // 456
val result2 = invalid.toIntOrNull()  // null (ไม่ throw exception)

println(result1)  // 456
println(result2)  // null

// ใช้ Elvis operator กำหนดค่า default
val number = invalid.toIntOrNull() ?: 0
println("number: $number")  // number: 0

// แปลงเลขฐาน
val hex = "FF"
val fromHex = hex.toInt(16)   // แปลง hex string เป็น Int
println("FF hex = $fromHex")  // FF hex = 255

val binary = "1010"
val fromBinary = binary.toInt(2)  // แปลง binary string เป็น Int
println("1010 binary = $fromBinary")  // 1010 binary = 10
```

---

## ค่าคงที่ (Constants)

### const val

```kotlin
// Top-level constants (นิยมใช้)
const val MAX_SIZE = 1000
const val PI = 3.14159265358979
const val APP_NAME = "MyApp"
const val VERSION = "1.0.0"

// ใน Object
object AppConstants {
    const val BASE_URL = "https://api.example.com"
    const val TIMEOUT_MS = 30_000
    const val MAX_RETRY = 3
    const val API_VERSION = "v1"
}

fun main() {
    println("App: $APP_NAME v$VERSION")
    println("Max: $MAX_SIZE")
    println("URL: ${AppConstants.BASE_URL}")
    println("Timeout: ${AppConstants.TIMEOUT_MS}ms")
}
```

### ความแตกต่างระหว่าง const val และ val

```kotlin
// const val: Compile-time constant
// - ต้องเป็น primitive type หรือ String
// - ค่าต้องรู้ตอน Compile
// - เร็วกว่า (ไม่มี getter overhead)
const val GRAVITY = 9.81    // ✅

// val: Runtime constant
// - ใช้ได้กับทุก Type
// - ค่าอาจรู้ตอน Runtime
// - มี getter overhead เล็กน้อย
val currentTime = System.currentTimeMillis()  // ✅ - รู้ตอน Runtime
val items = listOf(1, 2, 3)                  // ✅ - List ไม่ใช่ primitive

// const val ไม่สามารถใช้กับ:
// const val items = listOf(1, 2, 3)  // ❌ Error
// const val now = System.currentTimeMillis()  // ❌ Error - ไม่รู้ค่าตอน Compile
```

---

## Nullable Types เบื้องต้น

หนึ่งในฟีเจอร์เด่นที่สุดของ Kotlin คือ **Null Safety**:

```kotlin
// Non-nullable (ค่าเริ่มต้น - ห้ามเป็น null)
var name: String = "สมชาย"
name = null  // ❌ Error: Null can not be a value of a non-null type String

// Nullable (ต้องใส่ ?)
var nickname: String? = "ชาย"
nickname = null  // ✅ อนุญาต

// การใช้งาน Nullable
var email: String? = null

// ❌ ต้องตรวจสอบก่อนใช้
// println(email.length)  // Error: Only safe (?.) or non-null asserted (!!.) calls are allowed

// ✅ วิธีที่ 1: Safe Call (?.)
println(email?.length)  // null (ถ้า email เป็น null จะ return null)

// ✅ วิธีที่ 2: Elvis Operator (?:)
val len = email?.length ?: 0  // ถ้า email เป็น null ใช้ 0 แทน
println("ความยาว: $len")      // ความยาว: 0

// ✅ วิธีที่ 3: if-null check
if (email != null) {
    println(email.length)  // Smart cast: email ถูก cast เป็น String อัตโนมัติ
}

// ✅ วิธีที่ 4: let
email?.let { 
    println("Email: $it")  // จะรันเฉพาะถ้า email ไม่ใช่ null
}

// ⚠️ วิธีที่ 5: Non-null Assertion (!!) - ระวัง!
email = "test@example.com"
val length = email!!.length  // ถ้า email เป็น null จะ throw NullPointerException
println("Length: $length")   // Length: 16
```

---

## การตั้งชื่อตัวแปร

### กฎการตั้งชื่อ (Naming Conventions)

```kotlin
// ✅ ถูกต้อง - ใช้ camelCase สำหรับตัวแปรและฟังก์ชัน
val userName = "สมชาย"
val maxRetryCount = 3
var isAuthenticated = false
fun calculateTotal(): Double { ... }

// ✅ ถูกต้อง - ใช้ PascalCase สำหรับ Class
class UserProfile { ... }
class DatabaseHelper { ... }

// ✅ ถูกต้อง - ใช้ SCREAMING_SNAKE_CASE สำหรับ Constants
const val MAX_SIZE = 100
const val BASE_URL = "https://api.example.com"

// ✅ ถูกต้อง - เริ่มด้วยตัวอักษรหรือ _
val _privateField = "private"  // ไม่นิยม แต่ valid
val name2 = "name"             // ตามด้วยตัวเลขได้

// ❌ ผิด
// val 2name = "name"    // ขึ้นต้นด้วยตัวเลขไม่ได้
// val my-name = "name"  // มี - ไม่ได้
// val class = "name"    // ใช้ keyword ไม่ได้

// ⚠️ Backtick Escape - ใช้ keyword เป็นชื่อได้ (ไม่แนะนำ)
val `class` = "Math"  // ใช้ backtick ครอบ
val `my-name` = "Test"
println(`class`)  // Math
```

### ชื่อที่สื่อความหมาย

```kotlin
// ❌ ไม่ดี - ชื่อไม่สื่อความหมาย
val a = 30
val b = 1.5
val c = a * b

// ✅ ดีกว่า - ชื่อสื่อความหมาย
val workingDays = 30
val dailyRate = 1500.0
val monthlySalary = workingDays * dailyRate

// ❌ ไม่ดี - ย่อเกินไป
val usr = getUser()
val msg = "Error"

// ✅ ดีกว่า
val currentUser = getUser()
val errorMessage = "Error"

// ❌ ไม่ดี - ซ้ำซาก
val userObject = User()      // Object ไม่จำเป็น
val listOfNames = listOf()   // OfNames ไม่จำเป็น

// ✅ ดีกว่า
val user = User()
val names = listOf<String>()
```

---

## ตัวอย่างสรุปรวม

```kotlin
fun main() {
    // ===== ตัวแปรพื้นฐาน =====
    val appName: String = "Student Management System"
    val version: Double = 1.0
    var totalStudents: Int = 0
    var isRunning: Boolean = true
    
    println("=".repeat(50))
    println("$appName v$version")
    println("=".repeat(50))
    
    // ===== ข้อมูลนักเรียน =====
    val studentName = "สมหญิง รักเรียน"
    val studentId = "STD2024001"
    var gpa = 3.85f
    val isScholarship = gpa >= 3.5
    
    totalStudents++
    
    println("\nข้อมูลนักเรียน:")
    println("  ชื่อ: $studentName")
    println("  รหัส: $studentId")
    println("  GPA: ${"%.2f".format(gpa)}")
    println("  ทุนการศึกษา: ${if (isScholarship) "ได้รับ ✅" else "ไม่ได้รับ ❌"}")
    
    // ===== การคำนวณ =====
    val tuitionFee = 25_000.0
    val dormFee = 5_000.0
    val scholarshipAmount = if (isScholarship) tuitionFee * 0.5 else 0.0
    val totalFee = tuitionFee + dormFee - scholarshipAmount
    
    println("\nค่าใช้จ่าย:")
    println("  ค่าเล่าเรียน: ${String.format("%,.0f", tuitionFee)} บาท")
    println("  ค่าหอพัก: ${String.format("%,.0f", dormFee)} บาท")
    println("  ส่วนลดทุน: -${String.format("%,.0f", scholarshipAmount)} บาท")
    println("  รวมสุทธิ: ${String.format("%,.0f", totalFee)} บาท")
    
    // ===== Nullable =====
    var phoneNumber: String? = null
    var email: String? = "somying@example.com"
    
    println("\nข้อมูลติดต่อ:")
    println("  โทรศัพท์: ${phoneNumber ?: "ไม่ระบุ"}")
    println("  อีเมล: ${email ?: "ไม่ระบุ"}")
    println("  อีเมลความยาว: ${email?.length ?: 0} ตัวอักษร")
    
    println("\nจำนวนนักเรียนทั้งหมด: $totalStudents คน")
    println("สถานะระบบ: ${if (isRunning) "กำลังทำงาน" else "หยุดทำงาน"}")
}
```

**Output:**
```
==================================================
Student Management System v1.0
==================================================

ข้อมูลนักเรียน:
  ชื่อ: สมหญิง รักเรียน
  รหัส: STD2024001
  GPA: 3.85
  ทุนการศึกษา: ได้รับ ✅

ค่าใช้จ่าย:
  ค่าเล่าเรียน: 25,000 บาท
  ค่าหอพัก: 5,000 บาท
  ส่วนลดทุน: -12,500 บาท
  รวมสุทธิ: 17,500 บาท

ข้อมูลติดต่อ:
  โทรศัพท์: ไม่ระบุ
  อีเมล: somying@example.com
  อีเมลความยาว: 20 ตัวอักษร

จำนวนนักเรียนทั้งหมด: 1 คน
สถานะระบบ: กำลังทำงาน
```

---

## แบบฝึกหัด

### Exercise 1: ประกาศตัวแปร
สร้างตัวแปรต่อไปนี้แล้วแสดงผล:
- ชื่อของคุณ (String)
- อายุของคุณ (Int)
- ส่วนสูงเป็นเมตร (Double)
- เกิดวันอาทิตย์ไหม (Boolean)

```kotlin
// Solution
fun main() {
    val myName = "สมชาย"
    val myAge = 25
    val myHeight = 1.75
    val bornOnSunday = false
    
    println("ชื่อ: $myName")
    println("อายุ: $myAge ปี")
    println("ส่วนสูง: $myHeight เมตร")
    println("เกิดวันอาทิตย์: $bornOnSunday")
}
```

### Exercise 2: Type Conversion
รับตัวเลข String แล้วแปลงเป็น Double คำนวณและแสดงผล

```kotlin
// Solution
fun main() {
    val priceStr = "299.99"
    val quantityStr = "5"
    
    val price = priceStr.toDouble()
    val quantity = quantityStr.toInt()
    val total = price * quantity
    val discount = if (total > 1000) total * 0.1 else 0.0
    val finalPrice = total - discount
    
    println("ราคาต่อชิ้น: %.2f บาท".format(price))
    println("จำนวน: $quantity ชิ้น")
    println("รวม: %.2f บาท".format(total))
    println("ส่วนลด: %.2f บาท".format(discount))
    println("สุทธิ: %.2f บาท".format(finalPrice))
}
```

**Output:**
```
ราคาต่อชิ้น: 299.99 บาท
จำนวน: 5 ชิ้น
รวม: 1499.95 บาท
ส่วนลด: 149.99 บาท
สุทธิ: 1349.95 บาท
```

### Exercise 3: String Operations
รับชื่อเต็มแล้วแยกเป็นชื่อและนามสกุล แสดงผลต่างๆ

```kotlin
// Solution
fun main() {
    val fullName = "สมชาย ใจดี"
    val parts = fullName.split(" ")
    val firstName = parts[0]
    val lastName = parts[1]
    val initials = "${firstName.first()}.${lastName.first()}."
    
    println("ชื่อเต็ม: $fullName")
    println("ชื่อ: $firstName")
    println("นามสกุล: $lastName")
    println("อักษรย่อ: $initials")
    println("จำนวนตัวอักษร: ${fullName.replace(" ", "").length}")
    println("ตัวพิมพ์ใหญ่: ${fullName.uppercase()}")
}
```

### Exercise 4: ท้าทาย - Calculator
เขียน Simple Calculator แสดงผลการคำนวณ +(บวก) -(ลบ) *(คูณ) /(หาร) %(หารเอาเศษ)

```kotlin
// Solution
fun main() {
    val a = 17.0
    val b = 5.0
    
    println("=== Calculator ===")
    println("$a + $b = ${a + b}")
    println("$a - $b = ${a - b}")
    println("$a * $b = ${a * b}")
    println("$a / $b = ${"%.4f".format(a / b)}")
    println("$a % $b = ${a % b}")
    println("$a ^ $b = ${Math.pow(a, b)}")
    println("√$a = ${"%.4f".format(Math.sqrt(a))}")
}
```

**Output:**
```
=== Calculator ===
17.0 + 5.0 = 22.0
17.0 - 5.0 = 12.0
17.0 * 5.0 = 85.0
17.0 / 5.0 = 3.4000
17.0 % 5.0 = 2.0
17.0 ^ 5.0 = 1419857.0
√17.0 = 4.1231
```

---

## สรุป Part 02

```
✅ val = ค่าที่เปลี่ยนไม่ได้ (ใช้เป็น default)
✅ var = ค่าที่เปลี่ยนได้
✅ Type Inference - Kotlin อนุมาน Type ให้เองได้
✅ Integer Types: Byte, Short, Int, Long
✅ Floating Types: Float, Double
✅ Boolean: true/false
✅ Char: ตัวอักษรเดี่ยว ใช้ single quote
✅ String: ข้อความ ใช้ double quote, String Template ด้วย $
✅ Type Conversion ต้องทำเองด้วย toXxx()
✅ const val สำหรับ Compile-time Constants
✅ Nullable Types ใช้ ? บอกว่า null ได้
✅ Naming Convention: camelCase (ตัวแปร/ฟังก์ชัน), PascalCase (คลาส), SCREAMING_SNAKE_CASE (constants)
```

---

## Part ถัดไป

**Part 03: การรับและแสดงผลข้อมูล (Input/Output)**
- readLine() รับ Input จากผู้ใช้
- Scanner
- การ Format Output ด้วย printf style
- Console, File Output

---

*Part 02/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
