# Part 04: ตัวดำเนินการ (Operators)

## สารบัญ
1. [Arithmetic Operators](#arithmetic-operators)
2. [Assignment Operators](#assignment-operators)
3. [Comparison Operators](#comparison-operators)
4. [Logical Operators](#logical-operators)
5. [Bitwise Operators](#bitwise-operators)
6. [Range Operators](#range-operators)
7. [Elvis Operator](#elvis-operator)
8. [Safe Call Operator](#safe-call-operator)
9. [Not-Null Assertion](#not-null-assertion)
10. [in และ !in Operator](#in-และ-in-operator)
11. [is และ !is Operator](#is-และ-is-operator)
12. [Operator Precedence](#operator-precedence)
13. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Arithmetic Operators

ตัวดำเนินการทางคณิตศาสตร์พื้นฐาน:

```kotlin
fun main() {
    val a = 17
    val b = 5
    
    // บวก (+)
    println("$a + $b = ${a + b}")    // 22
    
    // ลบ (-)
    println("$a - $b = ${a - b}")    // 12
    
    // คูณ (*)
    println("$a * $b = ${a * b}")    // 85
    
    // หาร (/)
    println("$a / $b = ${a / b}")    // 3 (Integer Division - ตัดทศนิยม!)
    
    // หารเอาเศษ (%)
    println("$a % $b = ${a % b}")    // 2
    
    // ยกกำลัง - ไม่มี ** ต้องใช้ Math.pow()
    println("$a ^ $b = ${Math.pow(a.toDouble(), b.toDouble())}")  // 1419857.0
    
    // Integer Division ระวัง!
    val x = 7
    val y = 2
    println("$x / $y = ${x / y}")         // 3 (ไม่ใช่ 3.5!)
    println("$x / $y = ${x.toDouble() / y}")  // 3.5 (ต้องแปลงเป็น Double)
    
    // Unary Operators
    val c = 10
    println(-c)   // -10 (Negation)
    println(+c)   // 10  (Unary plus - ไม่ค่อยมีประโยชน์)
}
```

### การหารจำนวนเต็ม

```kotlin
fun main() {
    // Integer Division ใน Kotlin ตัดทศนิยมทิ้ง (Truncation ไม่ใช่ Rounding)
    println(7 / 2)     // 3 (ไม่ใช่ 4)
    println(-7 / 2)    // -3 (ไม่ใช่ -4)
    println(7 / -2)    // -3
    println(-7 / -2)   // 3
    
    // Modulo (เศษ) ใช้เครื่องหมายของตัวตั้ง
    println(7 % 3)     // 1
    println(-7 % 3)    // -1 (เครื่องหมายตาม -7)
    println(7 % -3)    // 1  (เครื่องหมายตาม 7)
    println(-7 % -3)   // -1
    
    // ตรวจสอบเลขคู่/คี่
    fun isEven(n: Int) = n % 2 == 0
    fun isOdd(n: Int) = n % 2 != 0
    
    println("4 เลขคู่: ${isEven(4)}")   // true
    println("7 เลขคี่: ${isOdd(7)}")    // true
    println("-3 เลขคี่: ${isOdd(-3)}")  // true
    
    // แปลงวันในสัปดาห์ (Circular)
    val day = 10
    val dayOfWeek = day % 7  // 0=อาทิตย์, 1=จันทร์, ...
    val dayNames = listOf("อาทิตย์", "จันทร์", "อังคาร", "พุธ", "พฤหัส", "ศุกร์", "เสาร์")
    println("วันที่ $day = ${dayNames[dayOfWeek]}")  // วันที่ 10 = พฤหัส
}
```

### Increment และ Decrement

```kotlin
fun main() {
    var x = 5
    
    // Pre-increment (++x): เพิ่มค่าก่อน แล้วใช้
    println(++x)  // 6 (x กลายเป็น 6 แล้วแสดง)
    println(x)    // 6
    
    // Post-increment (x++): ใช้ก่อน แล้วเพิ่มค่า
    println(x++)  // 6 (แสดง 6 แล้ว x กลายเป็น 7)
    println(x)    // 7
    
    // Pre-decrement (--x)
    println(--x)  // 6
    
    // Post-decrement (x--)
    println(x--)  // 6 (แสดง 6 แล้ว x กลายเป็น 5)
    println(x)    // 5
    
    // ตัวอย่างการใช้งาน
    var count = 0
    val list = listOf("a", "b", "c")
    
    for (item in list) {
        count++
    }
    println("นับได้: $count รายการ")  // นับได้: 3 รายการ
}
```

---

## Assignment Operators

```kotlin
fun main() {
    var x = 10
    
    // Simple Assignment
    x = 20
    println(x)  // 20
    
    // Compound Assignment Operators
    x += 5   // x = x + 5
    println(x)  // 25
    
    x -= 3   // x = x - 3
    println(x)  // 22
    
    x *= 2   // x = x * 2
    println(x)  // 44
    
    x /= 4   // x = x / 4
    println(x)  // 11
    
    x %= 3   // x = x % 3
    println(x)  // 2
    
    // String Concatenation Assignment
    var str = "Hello"
    str += " World"
    println(str)  // Hello World
    str += "!"
    println(str)  // Hello World!
    
    // List Assignment
    val numbers = mutableListOf(1, 2, 3)
    numbers += 4     // เพิ่มธาตุ
    numbers += listOf(5, 6)  // เพิ่มหลายธาตุ
    println(numbers)  // [1, 2, 3, 4, 5, 6]
    
    numbers -= 3     // ลบธาตุ
    println(numbers)  // [1, 2, 4, 5, 6]
}
```

---

## Comparison Operators

```kotlin
fun main() {
    val a = 10
    val b = 20
    
    // เปรียบเทียบตัวเลข
    println(a == b)   // false (เท่ากัน)
    println(a != b)   // true  (ไม่เท่ากัน)
    println(a < b)    // true  (น้อยกว่า)
    println(a > b)    // false (มากกว่า)
    println(a <= b)   // true  (น้อยกว่าหรือเท่ากัน)
    println(a >= b)   // false (มากกว่าหรือเท่ากัน)
    
    // เปรียบเทียบ String
    val s1 = "apple"
    val s2 = "apple"
    val s3 = "banana"
    
    println(s1 == s2)  // true  (เนื้อหาเหมือนกัน)
    println(s1 == s3)  // false
    println(s1 < s3)   // true  (alphabetical order)
    
    // Structural Equality (==) vs Referential Equality (===)
    val str1 = String(charArrayOf('A', 'B', 'C'))
    val str2 = String(charArrayOf('A', 'B', 'C'))
    
    println(str1 == str2)   // true  (เนื้อหาเหมือนกัน)
    println(str1 === str2)  // false (เป็นคนละ Object)
    
    // Object Comparison
    data class Point(val x: Int, val y: Int)
    val p1 = Point(1, 2)
    val p2 = Point(1, 2)
    val p3 = p1
    
    println(p1 == p2)   // true  (Data class เปรียบเทียบเนื้อหา)
    println(p1 === p2)  // false (คนละ Object)
    println(p1 === p3)  // true  (อ้างถึง Object เดียวกัน)
    
    // Comparable
    println("abc".compareTo("abd"))   // -1 (น้อยกว่า)
    println("abc".compareTo("abc"))   // 0  (เท่ากัน)
    println("abd".compareTo("abc"))   // 1  (มากกว่า)
}
```

---

## Logical Operators

```kotlin
fun main() {
    val x = 10
    val y = 20
    val z = 15
    
    // AND (&&) - ต้องเป็น true ทั้งคู่
    println(x < y && y > z)   // true && true = true
    println(x > y && y > z)   // false && true = false
    
    // OR (||) - อย่างน้อยหนึ่งอันต้องเป็น true
    println(x > y || y > z)   // false || true = true
    println(x > y || y < z)   // false || false = false
    
    // NOT (!) - กลับค่า
    println(!true)   // false
    println(!false)  // true
    println(!(x > y))  // !(false) = true
    
    // Short-Circuit Evaluation
    fun checkPositive(n: Int): Boolean {
        println("กำลังตรวจสอบ $n")
        return n > 0
    }
    
    println("\n=== AND Short-Circuit ===")
    // ถ้าตัวแรก false จะไม่ประเมินตัวที่สอง
    val result1 = checkPositive(-5) && checkPositive(3)
    // Output: กำลังตรวจสอบ -5 (ไม่มี กำลังตรวจสอบ 3)
    println("Result: $result1")  // false
    
    println("\n=== OR Short-Circuit ===")
    // ถ้าตัวแรก true จะไม่ประเมินตัวที่สอง
    val result2 = checkPositive(5) || checkPositive(-3)
    // Output: กำลังตรวจสอบ 5 (ไม่มี กำลังตรวจสอบ -3)
    println("Result: $result2")  // true
    
    // ใช้ในการป้องกัน NullPointerException
    val list: List<Int>? = null
    
    // ❌ อาจเกิด NullPointerException
    // if (list.size > 0) { ... }
    
    // ✅ ป้องกันด้วย Short-Circuit
    if (list != null && list.size > 0) {
        println("List ไม่ว่าง")
    } else {
        println("List ว่างหรือเป็น null")  // ← จะพิมพ์อันนี้
    }
}
```

---

## Bitwise Operators

Kotlin ใช้ฟังก์ชัน (ไม่ใช่ Symbol) สำหรับ Bitwise Operations:

```kotlin
fun main() {
    val a = 0b1010  // 10 ในฐาน 2
    val b = 0b1100  // 12 ในฐาน 2
    
    // AND (and)
    println("${a.toString(2)} AND ${b.toString(2)} = ${(a and b).toString(2)}")
    println("$a and $b = ${a and b}")  // 1010 AND 1100 = 1000 = 8
    
    // OR (or)
    println("$a or $b = ${a or b}")   // 1010 OR 1100 = 1110 = 14
    
    // XOR (xor)
    println("$a xor $b = ${a xor b}")  // 1010 XOR 1100 = 0110 = 6
    
    // NOT (inv) - Invert bits
    println("inv $a = ${a.inv()}")     // ปลิ้นบิตทุกตัว = -11
    
    // Left Shift (shl)
    println("$a shl 1 = ${a shl 1}")  // 1010 << 1 = 10100 = 20 (คูณ 2)
    println("$a shl 2 = ${a shl 2}")  // 1010 << 2 = 101000 = 40 (คูณ 4)
    
    // Right Shift (shr) - Sign-preserving
    println("$a shr 1 = ${a shr 1}")  // 1010 >> 1 = 0101 = 5 (หาร 2)
    
    // Unsigned Right Shift (ushr) - Zero-fill
    val negative = -1
    println("$negative shr 1 = ${negative shr 1}")   // -1
    println("$negative ushr 1 = ${negative ushr 1}") // 2147483647
    
    // ตัวอย่างการใช้งาน: Flags
    const val READ_PERMISSION    = 0b001  // 1
    const val WRITE_PERMISSION   = 0b010  // 2
    const val EXECUTE_PERMISSION = 0b100  // 4
    
    var userPermissions = 0
    
    // เพิ่ม permission
    userPermissions = userPermissions or READ_PERMISSION
    userPermissions = userPermissions or WRITE_PERMISSION
    
    // ตรวจสอบ permission
    val canRead    = (userPermissions and READ_PERMISSION) != 0
    val canWrite   = (userPermissions and WRITE_PERMISSION) != 0
    val canExecute = (userPermissions and EXECUTE_PERMISSION) != 0
    
    println("\nUser Permissions:")
    println("Read: $canRead")       // true
    println("Write: $canWrite")     // true
    println("Execute: $canExecute") // false
    
    // ลบ permission
    userPermissions = userPermissions and WRITE_PERMISSION.inv()
    println("After removing write:")
    println("Write: ${(userPermissions and WRITE_PERMISSION) != 0}")  // false
}
```

---

## Range Operators

```kotlin
fun main() {
    // .. Range (Inclusive on both ends)
    val range1 = 1..10
    println("1..10 contains 5: ${5 in range1}")   // true
    println("1..10 contains 10: ${10 in range1}")  // true
    println("1..10 contains 11: ${11 in range1}")  // false
    
    // until Range (Exclusive end)
    val range2 = 1 until 10
    println("1 until 10 contains 9: ${9 in range2}")   // true
    println("1 until 10 contains 10: ${10 in range2}")  // false
    
    // downTo (Descending)
    val range3 = 10 downTo 1
    println("10 downTo 1:")
    for (i in range3) print("$i ")  // 10 9 8 7 6 5 4 3 2 1
    println()
    
    // step (กำหนดช่วง)
    for (i in 1..10 step 2) print("$i ")  // 1 3 5 7 9
    println()
    
    for (i in 10 downTo 1 step 3) print("$i ")  // 10 7 4 1
    println()
    
    // Char Range
    for (c in 'A'..'Z') print("$c")  // ABCDEFGHIJKLMNOPQRSTUVWXYZ
    println()
    
    for (c in 'a'..'z' step 2) print("$c")  // acegikmoqsuwy
    println()
    
    // String Range (ใช้ in/contains)
    val fruits = "apple".."mango"
    println("banana in fruits range: ${"banana" in fruits}")  // true
    println("zebra in fruits range: ${"zebra" in fruits}")    // false
    
    // Double Range
    val dRange = 0.0..1.0
    println("0.5 in dRange: ${0.5 in dRange}")  // true
    
    // ใช้ Range กับ when
    val score = 85
    val grade = when (score) {
        in 90..100 -> "A"
        in 80..89  -> "B"
        in 70..79  -> "C"
        in 60..69  -> "D"
        else       -> "F"
    }
    println("คะแนน $score เกรด $grade")  // คะแนน 85 เกรด B
}
```

---

## Elvis Operator

`?:` (Elvis Operator) ใช้กำหนดค่า default เมื่อ expression เป็น null:

```kotlin
fun main() {
    // รูปแบบ: expression ?: defaultValue
    
    var name: String? = null
    
    // แบบปกติ
    val displayName = if (name != null) name else "ไม่ทราบชื่อ"
    
    // แบบ Elvis
    val displayName2 = name ?: "ไม่ทราบชื่อ"
    
    println(displayName2)  // ไม่ทราบชื่อ
    
    name = "สมชาย"
    val displayName3 = name ?: "ไม่ทราบชื่อ"
    println(displayName3)  // สมชาย
    
    // Elvis กับ function call
    fun getUserById(id: Int): String? {
        return if (id == 1) "Admin" else null
    }
    
    val user = getUserById(2) ?: "Guest"
    println("User: $user")  // User: Guest
    
    // Elvis กับ throw
    fun processName(name: String?) {
        val validName = name ?: throw IllegalArgumentException("Name cannot be null")
        println("Processing: $validName")
    }
    
    processName("สมชาย")  // Processing: สมชาย
    // processName(null)  // throws IllegalArgumentException
    
    // Elvis กับ return
    fun getLength(str: String?): Int {
        return str?.length ?: return -1
    }
    
    println(getLength("Hello"))   // 5
    println(getLength(null))      // -1
    
    // Chain Elvis
    val user2: String? = null
    val config: String? = null
    val default = "default_user"
    
    val result = user2 ?: config ?: default
    println("Result: $result")  // default_user
}
```

---

## Safe Call Operator

`?.` (Safe Call Operator) ใช้เรียกใช้ method/property บน nullable object:

```kotlin
fun main() {
    var text: String? = null
    
    // ❌ ไม่ปลอดภัย
    // println(text.length)  // NullPointerException!
    
    // ✅ Safe Call
    println(text?.length)      // null (ไม่ crash)
    println(text?.uppercase()) // null
    
    text = "Hello Kotlin"
    println(text?.length)      // 12
    println(text?.uppercase()) // HELLO KOTLIN
    
    // Safe Call Chain
    data class Address(val city: String?, val country: String)
    data class User(val name: String, val address: Address?)
    
    val user1 = User("สมชาย", null)
    val user2 = User("สมหญิง", Address(null, "Thailand"))
    val user3 = User("สมศักดิ์", Address("Bangkok", "Thailand"))
    
    println(user1.address?.city)   // null
    println(user2.address?.city)   // null
    println(user3.address?.city)   // Bangkok
    
    // Safe Call กับ Elvis
    val city1 = user1.address?.city ?: "ไม่ทราบ"
    val city2 = user2.address?.city ?: "ไม่ทราบ"
    val city3 = user3.address?.city ?: "ไม่ทราบ"
    
    println("$city1, $city2, $city3")  // ไม่ทราบ, ไม่ทราบ, Bangkok
    
    // Safe Call กับ let
    val nullableString: String? = "Hello World"
    nullableString?.let { str ->
        println("ความยาว: ${str.length}")
        println("ตัวพิมพ์ใหญ่: ${str.uppercase()}")
    }
    
    val nullValue: String? = null
    nullValue?.let { 
        println("บรรทัดนี้จะไม่ถูก execute")
    }
}
```

---

## Not-Null Assertion

`!!` (Not-Null Assertion) บอก compiler ว่า "ค่านี้ไม่เป็น null แน่นอน":

```kotlin
fun main() {
    var text: String? = "Hello"
    
    // !! บอกว่า text ไม่เป็น null แน่นอน
    val length = text!!.length  // ✅ ถ้า text ไม่เป็น null
    println(length)  // 5
    
    // ⚠️ อันตราย! ถ้า text เป็น null จะ throw KotlinNullPointerException
    text = null
    // val crash = text!!.length  // 💥 KotlinNullPointerException!
    
    // ใช้ !! เมื่อมั่นใจ 100% ว่าไม่เป็น null
    fun getConfigValue(): String? {
        return System.getenv("APP_ENV") ?: "development"
    }
    
    // กรณีนี้รู้ว่าไม่ null เพราะมี fallback
    val env = getConfigValue()!!
    println("Environment: $env")
    
    // แนะนำ: ใช้ let หรือ ?: แทน !!
    val config: String? = "production"
    
    // แทนที่จะใช้
    // val value = config!!
    
    // ใช้แบบนี้แทน
    val value = config ?: throw IllegalStateException("Config not found")
    println("Config: $value")
}
```

---

## in และ !in Operator

```kotlin
fun main() {
    // in กับ Collection
    val fruits = listOf("apple", "banana", "orange")
    println("apple" in fruits)    // true
    println("grape" in fruits)    // false
    println("grape" !in fruits)   // true
    
    // in กับ Range
    val age = 25
    println(age in 18..65)    // true
    println(age in 0 until 18) // false
    
    // in กับ String (contains)
    val sentence = "Hello World Kotlin"
    println("Kotlin" in sentence)  // true
    println("Java" in sentence)    // false
    
    // in กับ Map (ตรวจสอบ key)
    val scores = mapOf("Math" to 90, "Thai" to 85, "English" to 78)
    println("Math" in scores)     // true
    println("Science" in scores)  // false
    
    // ใช้ใน when
    val grade = when {
        age in 0..12    -> "เด็ก"
        age in 13..17   -> "วัยรุ่น"
        age in 18..59   -> "ผู้ใหญ่"
        else            -> "ผู้สูงอายุ"
    }
    println("$age ปี: $grade")  // 25 ปี: ผู้ใหญ่
    
    // ตัวอย่างใช้งานจริง
    val validCountryCodes = setOf("TH", "US", "JP", "GB", "DE")
    val countryCode = "TH"
    
    if (countryCode in validCountryCodes) {
        println("$countryCode เป็นรหัสประเทศที่ถูกต้อง")
    } else {
        println("$countryCode ไม่พบในระบบ")
    }
}
```

---

## is และ !is Operator

```kotlin
fun main() {
    // is ตรวจสอบ Type
    val value: Any = "Hello Kotlin"
    
    println(value is String)    // true
    println(value is Int)       // false
    println(value !is Int)      // true
    
    // Smart Cast - หลังจาก is ตรวจสอบแล้ว Kotlin จะ cast ให้อัตโนมัติ
    if (value is String) {
        // ภายใน block นี้ value ถูก Smart Cast เป็น String
        println(value.length)       // 12 - ไม่ต้อง cast!
        println(value.uppercase())  // HELLO KOTLIN
    }
    
    // Smart Cast กับ &&
    fun processValue(input: Any?) {
        if (input != null && input is String) {
            // Smart Cast ทำงาน!
            println("String length: ${input.length}")
        }
    }
    
    processValue("Hello")  // String length: 5
    processValue(42)       // ไม่พิมพ์อะไร
    processValue(null)     // ไม่พิมพ์อะไร
    
    // is กับ when (แนะนำมาก!)
    fun describe(obj: Any): String = when (obj) {
        is Int          -> "จำนวนเต็ม: $obj"
        is Double       -> "จำนวนทศนิยม: $obj"
        is String       -> "ข้อความ: '$obj' ยาว ${obj.length} ตัวอักษร"
        is Boolean      -> "Boolean: $obj"
        is List<*>      -> "List มี ${obj.size} รายการ"
        is IntArray     -> "IntArray: ${obj.toList()}"
        else            -> "ไม่ทราบ type: ${obj::class.simpleName}"
    }
    
    println(describe(42))
    println(describe(3.14))
    println(describe("Kotlin"))
    println(describe(true))
    println(describe(listOf(1, 2, 3)))
    println(describe(intArrayOf(4, 5, 6)))
}
```

**Output:**
```
จำนวนเต็ม: 42
จำนวนทศนิยม: 3.14
ข้อความ: 'Kotlin' ยาว 6 ตัวอักษร
Boolean: true
List มี 3 รายการ
IntArray: [4, 5, 6]
```

---

## Operator Precedence

ลำดับความสำคัญของตัวดำเนินการ (สูงสุดก่อน):

```
1. Postfix         : ++, --
2. Prefix          : -, +, ++, --, !
3. Type RHS        : :, as, as?
4. Multiplicative  : *, /, %
5. Additive        : +, -
6. Range           : ..
7. Infix functions : shl, shr, ushr, and, or, xor
8. Elvis           : ?:
9. Named checks    : in, !in, is, !is
10. Comparison     : <, >, <=, >=
11. Equality       : ==, !=
12. Conjunction    : &&
13. Disjunction    : ||
14. Spread         : *
15. Assignment     : =, +=, -=, *=, /=, %=
```

### ตัวอย่าง Precedence

```kotlin
fun main() {
    // คำนวณตามลำดับ Precedence
    println(2 + 3 * 4)     // 14 (ไม่ใช่ 20) - * ก่อน +
    println((2 + 3) * 4)   // 20 - วงเล็บก่อน
    
    println(10 - 4 / 2)    // 8  - / ก่อน -
    println((10 - 4) / 2)  // 3  - วงเล็บก่อน
    
    println(2 + 3 > 4)     // true  - คำนวณ 2+3=5 ก่อน แล้ว 5>4
    println(true || false && false)  // true - && มีลำดับสูงกว่า ||
    
    // ซับซ้อนขึ้น
    val a = 5
    val b = 3
    val c = 2
    
    // a + b * c - a / c
    val result = a + b * c - a / c
    // = 5 + (3*2) - (5/2)
    // = 5 + 6 - 2 (Integer division)
    // = 9
    println(result)  // 9
    
    // แนะนำ: ใช้วงเล็บให้ชัดเจนแม้จะรู้ Precedence
    val clear = a + (b * c) - (a / c)
    println(clear)  // 9 - เหมือนกัน แต่อ่านง่ายกว่า
}
```

---

## ตัวอย่างสรุปรวม

```kotlin
fun main() {
    println("=== ระบบคำนวณภาษี ===\n")
    
    // รับข้อมูล
    val salary = 65_000.0
    val otherIncome = 10_000.0
    val totalIncome = salary + otherIncome
    
    // คำนวณการหักลดหย่อน
    val personalDeduction = 60_000.0
    val socialSecurity = minOf(salary * 0.05, 750.0)  // สูงสุด 750 บาท
    val netIncome = maxOf(totalIncome - personalDeduction - socialSecurity, 0.0)
    
    // คำนวณภาษี (Progressive Tax)
    val tax = when {
        netIncome <= 150_000 -> 0.0
        netIncome <= 300_000 -> (netIncome - 150_000) * 0.05
        netIncome <= 500_000 -> 7_500 + (netIncome - 300_000) * 0.10
        netIncome <= 750_000 -> 27_500 + (netIncome - 500_000) * 0.15
        netIncome <= 1_000_000 -> 65_000 + (netIncome - 750_000) * 0.20
        else -> 115_000 + (netIncome - 1_000_000) * 0.25
    }
    
    // ตรวจสอบช่วงเงินได้
    val incomeLevel = when (totalIncome) {
        in 0.0..15_000.0   -> "ต่ำกว่าเกณฑ์"
        in 15_001.0..30_000.0 -> "รายได้ต่ำ"
        in 30_001.0..60_000.0 -> "รายได้ปานกลาง"
        in 60_001.0..100_000.0 -> "รายได้สูง"
        else -> "รายได้สูงมาก"
    }
    
    // แสดงผล
    println("รายได้จากเงินเดือน:     ${"%,.2f".format(salary)} บาท")
    println("รายได้อื่น:              ${"%,.2f".format(otherIncome)} บาท")
    println("รายได้รวม:              ${"%,.2f".format(totalIncome)} บาท")
    println("ระดับรายได้:            $incomeLevel")
    println()
    println("หักลดหย่อนส่วนตัว:      ${"%,.2f".format(personalDeduction)} บาท")
    println("หักประกันสังคม:         ${"%,.2f".format(socialSecurity)} บาท")
    println("รายได้สุทธิ:            ${"%,.2f".format(netIncome)} บาท")
    println()
    println("ภาษีที่ต้องชำระ:         ${"%,.2f".format(tax)} บาท")
    println("อัตราภาษีแท้จริง:        ${"%.2f".format(tax / totalIncome * 100)}%")
    
    // ตรวจสอบว่าต้องชำระภาษีไหม
    val mustPayTax = tax > 0 && netIncome > 0
    println("\nต้องชำระภาษี: ${if (mustPayTax) "ใช่" else "ไม่ใช่"}")
}
```

---

## แบบฝึกหัด

### Exercise 1: คำนวณ BMI
รับน้ำหนักและส่วนสูง คำนวณ BMI และแสดงสถานะ

```kotlin
fun main() {
    val weight = 75.0
    val heightCm = 172.0
    val heightM = heightCm / 100
    val bmi = weight / (heightM * heightM)
    
    val category = when {
        bmi < 18.5 -> "น้ำหนักน้อย"
        bmi < 25.0 -> "น้ำหนักปกติ"
        bmi < 30.0 -> "น้ำหนักเกิน"
        else -> "อ้วน"
    }
    
    println("BMI: ${"%.2f".format(bmi)} ($category)")
}
```

### Exercise 2: ตรวจสอบปีอธิกสุรทิน

```kotlin
fun isLeapYear(year: Int): Boolean {
    return (year % 4 == 0 && year % 100 != 0) || (year % 400 == 0)
}

fun main() {
    val years = listOf(2000, 1900, 2024, 2023, 2100)
    for (year in years) {
        println("$year: ${if (isLeapYear(year)) "ปีอธิกสุรทิน" else "ปีปกติ"}")
    }
}
```

**Output:**
```
2000: ปีอธิกสุรทิน
1900: ปีปกติ
2024: ปีอธิกสุรทิน
2023: ปีปกติ
2100: ปีปกติ
```

### Exercise 3: ท้าทาย - เข้ารหัส Caesar Cipher

```kotlin
fun caesarEncode(text: String, shift: Int): String {
    return text.map { c ->
        when {
            c in 'A'..'Z' -> ((c.code - 'A'.code + shift) % 26 + 'A'.code).toChar()
            c in 'a'..'z' -> ((c.code - 'a'.code + shift) % 26 + 'a'.code).toChar()
            else -> c
        }
    }.joinToString("")
}

fun caesarDecode(text: String, shift: Int): String = caesarEncode(text, 26 - shift)

fun main() {
    val message = "Hello Kotlin World"
    val shift = 3
    
    val encoded = caesarEncode(message, shift)
    val decoded = caesarDecode(encoded, shift)
    
    println("ต้นฉบับ: $message")
    println("เข้ารหัส (shift=$shift): $encoded")
    println("ถอดรหัส: $decoded")
    println("ถูกต้อง: ${message == decoded}")
}
```

**Output:**
```
ต้นฉบับ: Hello Kotlin World
เข้ารหัส (shift=3): Khoor Nrwolq Zruog
ถอดรหัส: Hello Kotlin World
ถูกต้อง: true
```

---

## สรุป Part 04

```
✅ Arithmetic: +, -, *, /, %, ++, --
✅ Assignment: =, +=, -=, *=, /=, %=
✅ Comparison: ==, !=, <, >, <=, >=, ===, !==
✅ Logical: &&, ||, ! (Short-circuit evaluation)
✅ Bitwise: and, or, xor, inv, shl, shr, ushr
✅ Range: .., until, downTo, step
✅ Elvis (?:): ค่า default เมื่อ null
✅ Safe Call (?.): เรียกใช้บน nullable object
✅ Not-null Assertion (!!): บอกว่าไม่ null
✅ in/!in: ตรวจสอบว่าอยู่ใน Collection/Range
✅ is/!is: ตรวจสอบ Type พร้อม Smart Cast
✅ Operator Precedence: *, / ก่อน +, -
```

---

*Part 04/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
