# Part 05: การควบคุมการทำงาน - if/when (Control Flow)

## สารบัญ
1. [if Expression](#if-expression)
2. [if-else](#if-else)
3. [if-else if-else](#if-else-if-else)
4. [if เป็น Expression](#if-เป็น-expression)
5. [when Expression](#when-expression)
6. [when กับ Multiple Conditions](#when-กับ-multiple-conditions)
7. [when กับ Types](#when-กับ-types)
8. [when โดยไม่มี Argument](#when-โดยไม่มี-argument)
9. [Nested Conditionals](#nested-conditionals)
10. [Guard Clauses / Early Return](#guard-clauses--early-return)
11. [แบบฝึกหัด](#แบบฝึกหัด)

---

## if Expression

### if พื้นฐาน

```kotlin
fun main() {
    val temperature = 35
    
    // if แบบง่าย
    if (temperature > 30) {
        println("อากาศร้อน! ควรดื่มน้ำมากๆ")
    }
    
    // if แบบบรรทัดเดียว (เมื่อมีคำสั่งเดียวใน block)
    if (temperature > 30) println("อากาศร้อนมาก")
    
    // แต่แนะนำให้ใส่ {} เสมอ เพื่อความชัดเจน
    if (temperature > 30) {
        println("อากาศร้อนมาก")
    }
    
    // ตัวอย่างเพิ่มเติม
    val isRaining = true
    
    if (isRaining) {
        println("ฝนตก ควรพกร่ม")
    }
    
    val score = 85
    if (score >= 80) {
        println("ผ่านด้วยคะแนนดี")
    }
}
```

---

## if-else

```kotlin
fun main() {
    val age = 17
    
    // if-else พื้นฐาน
    if (age >= 18) {
        println("คุณเป็นผู้ใหญ่แล้ว")
    } else {
        println("คุณยังเป็นผู้เยาว์")
    }
    
    // ตัวอย่างการตรวจสอบเลขคู่/คี่
    val number = 42
    if (number % 2 == 0) {
        println("$number เป็นเลขคู่")
    } else {
        println("$number เป็นเลขคี่")
    }
    
    // ตัวอย่าง Login
    val username = "admin"
    val password = "secret123"
    val inputUser = "admin"
    val inputPass = "secret123"
    
    if (username == inputUser && password == inputPass) {
        println("เข้าสู่ระบบสำเร็จ!")
    } else {
        println("ชื่อผู้ใช้หรือรหัสผ่านไม่ถูกต้อง")
    }
}
```

---

## if-else if-else

```kotlin
fun main() {
    // ตรวจสอบเกรด
    val score = 72
    
    if (score >= 90) {
        println("เกรด A - ยอดเยี่ยม!")
    } else if (score >= 80) {
        println("เกรด B - ดีมาก!")
    } else if (score >= 70) {
        println("เกรด C - ดี")
    } else if (score >= 60) {
        println("เกรด D - พอใช้")
    } else {
        println("เกรด F - ต้องปรับปรุง")
    }
    
    // ตัวอย่าง: ระดับความดันโลหิต
    val systolic = 125
    val diastolic = 82
    
    if (systolic < 120 && diastolic < 80) {
        println("ความดันปกติ")
    } else if (systolic < 130 && diastolic < 80) {
        println("ความดันสูงขึ้นเล็กน้อย (Elevated)")
    } else if (systolic < 140 || diastolic < 90) {
        println("ความดันสูงระดับ 1 (High Blood Pressure Stage 1)")
    } else if (systolic >= 140 || diastolic >= 90) {
        println("ความดันสูงระดับ 2 (High Blood Pressure Stage 2)")
    } else {
        println("วิกฤต! ควรพบแพทย์ทันที")
    }
    // Output: ความดันสูงระดับ 1
}
```

---

## if เป็น Expression

ใน Kotlin `if` เป็น **Expression** ที่ return ค่าได้ (ต่างจาก Java ที่ if เป็นแค่ Statement):

```kotlin
fun main() {
    val a = 10
    val b = 20
    
    // if เป็น expression - return ค่า
    val max = if (a > b) a else b
    println("Max: $max")  // Max: 20
    
    val min = if (a < b) a else b
    println("Min: $min")  // Min: 10
    
    // ใช้แทน Ternary Operator ของ Java (?:)
    // Java: int max = (a > b) ? a : b;
    // Kotlin: val max = if (a > b) a else b
    
    // if expression กับ multiple statements
    val description = if (a > b) {
        println("a มากกว่า b")  // สามารถมีหลาย statement
        "a is greater"          // ค่าสุดท้ายคือที่ return
    } else {
        println("b มากกว่าหรือเท่ากับ a")
        "b is greater or equal"
    }
    println(description)
    
    // ใน function
    fun classify(n: Int): String {
        return if (n > 0) "บวก" else if (n < 0) "ลบ" else "ศูนย์"
    }
    
    println(classify(5))   // บวก
    println(classify(-3))  // ลบ
    println(classify(0))   // ศูนย์
    
    // if expression กับ null check
    val text: String? = null
    val length = if (text != null) text.length else 0
    // หรือใช้ Elvis ซึ่งกระชับกว่า
    val length2 = text?.length ?: 0
    
    println("Length: $length")   // 0
    println("Length2: $length2") // 0
}
```

---

## when Expression

`when` คือ Kotlin version ของ `switch` ใน Java แต่ทรงพลังกว่ามาก:

### when พื้นฐาน

```kotlin
fun main() {
    val dayNum = 3
    
    val dayName = when (dayNum) {
        1 -> "จันทร์"
        2 -> "อังคาร"
        3 -> "พุธ"
        4 -> "พฤหัสบดี"
        5 -> "ศุกร์"
        6 -> "เสาร์"
        7 -> "อาทิตย์"
        else -> "ไม่รู้จัก"
    }
    println("วันที่ $dayNum คือวัน$dayName")  // วันที่ 3 คือวันพุธ
    
    // when กับ String
    val language = "Kotlin"
    val platform = when (language) {
        "Java", "Kotlin" -> "JVM"     // หลาย value ใน case เดียว
        "Swift"          -> "iOS/macOS"
        "Dart"           -> "Flutter"
        "JavaScript", "TypeScript" -> "Web/Node.js"
        else -> "Unknown"
    }
    println("$language ใช้บน $platform")  // Kotlin ใช้บน JVM
    
    // when เป็น Expression ใน println
    println(when (language) {
        "Kotlin" -> "ภาษาโปรดของผม!"
        "Java"   -> "Classic!"
        else     -> "อีกภาษาหนึ่ง"
    })
}
```

### when กับ Block

```kotlin
fun main() {
    val status = 404
    
    when (status) {
        200 -> {
            println("✅ สำเร็จ")
            println("ดำเนินการต่อ...")
        }
        301, 302 -> {
            println("🔀 Redirect")
            println("ตาม redirect URL...")
        }
        400 -> {
            println("❌ Bad Request")
            println("ตรวจสอบ request parameters")
        }
        401 -> {
            println("🔒 Unauthorized")
            println("กรุณาเข้าสู่ระบบก่อน")
        }
        403 -> {
            println("🚫 Forbidden")
            println("คุณไม่มีสิทธิ์เข้าถึง")
        }
        404 -> {
            println("🔍 Not Found")
            println("ไม่พบ resource ที่ต้องการ")
        }
        500 -> {
            println("💥 Internal Server Error")
            println("มีข้อผิดพลาดในเซิร์ฟเวอร์")
        }
        else -> {
            println("❓ Unknown Status: $status")
        }
    }
}
```

---

## when กับ Multiple Conditions

```kotlin
fun main() {
    // Range ใน when
    val score = 85
    val grade = when (score) {
        in 90..100 -> "A"
        in 80..89  -> "B"
        in 70..79  -> "C"
        in 60..69  -> "D"
        in 0..59   -> "F"
        else       -> "คะแนนไม่ถูกต้อง"
    }
    println("คะแนน $score = เกรด $grade")  // คะแนน 85 = เกรด B
    
    // Condition ใน when
    val number = -15
    val description = when {
        number < -100 -> "ติดลบมาก"
        number < 0    -> "ติดลบ"
        number == 0   -> "ศูนย์"
        number < 100  -> "บวก"
        else          -> "บวกมาก"
    }
    println("$number: $description")  // -15: ติดลบ
    
    // หลาย value (comma-separated)
    val month = 2
    val daysInMonth = when (month) {
        1, 3, 5, 7, 8, 10, 12 -> 31
        4, 6, 9, 11 -> 30
        2 -> 28  // ปกติ (ไม่รวมปีอธิกสุรทิน)
        else -> throw IllegalArgumentException("เดือนไม่ถูกต้อง: $month")
    }
    println("เดือน $month มี $daysInMonth วัน")  // เดือน 2 มี 28 วัน
    
    // Predicate ใน when
    val text = "   "
    val status = when {
        text.isEmpty()      -> "ว่างเปล่า"
        text.isBlank()      -> "มีแต่ space"
        text.length < 10    -> "สั้น"
        text.length < 50    -> "ปานกลาง"
        else                -> "ยาว"
    }
    println("'$text' → $status")  // '   ' → มีแต่ space
}
```

---

## when กับ Types

```kotlin
fun main() {
    fun processValue(value: Any): String {
        return when (value) {
            is Int -> "จำนวนเต็ม: $value (x2 = ${value * 2})"
            is Double -> "ทศนิยม: ${"%.2f".format(value)}"
            is String -> "ข้อความ: '$value' (${value.length} ตัว)"
            is Boolean -> "Boolean: ${if (value) "จริง" else "เท็จ"}"
            is List<*> -> "List: ${value.size} รายการ → $value"
            is IntArray -> "IntArray: ${value.toList()}"
            null -> "เป็น null"
            else -> "ไม่รู้จัก Type: ${value::class.simpleName}"
        }
    }
    
    println(processValue(42))
    println(processValue(3.14159))
    println(processValue("Hello Kotlin"))
    println(processValue(false))
    println(processValue(listOf(1, 2, 3)))
    println(processValue(intArrayOf(4, 5, 6)))
    println(processValue(null))
    println(processValue('A'))
    
    // when กับ Sealed Class (Part 20 จะอธิบายละเอียด)
    sealed class Shape
    data class Circle(val radius: Double) : Shape()
    data class Rectangle(val width: Double, val height: Double) : Shape()
    data class Triangle(val base: Double, val height: Double) : Shape()
    
    fun calculateArea(shape: Shape): Double = when (shape) {
        is Circle    -> Math.PI * shape.radius * shape.radius
        is Rectangle -> shape.width * shape.height
        is Triangle  -> 0.5 * shape.base * shape.height
    }
    
    val shapes = listOf(
        Circle(5.0),
        Rectangle(4.0, 6.0),
        Triangle(3.0, 8.0)
    )
    
    for (shape in shapes) {
        println("$shape → พื้นที่ = ${"%.2f".format(calculateArea(shape))}")
    }
}
```

**Output:**
```
จำนวนเต็ม: 42 (x2 = 84)
ทศนิยม: 3.14
ข้อความ: 'Hello Kotlin' (12 ตัว)
Boolean: เท็จ
List: 3 รายการ → [1, 2, 3]
IntArray: [4, 5, 6]
เป็น null
ไม่รู้จัก Type: Char
Circle(radius=5.0) → พื้นที่ = 78.54
Rectangle(width=4.0, height=6.0) → พื้นที่ = 24.00
Triangle(base=3.0, height=8.0) → พื้นที่ = 12.00
```

---

## when โดยไม่มี Argument

เมื่อไม่ใส่ argument ให้ when มันจะเหมือน if-else chain:

```kotlin
fun main() {
    val x = 15
    val y = 20
    
    // when โดยไม่มี argument - เหมือน if-else if
    val result = when {
        x > y       -> "$x มากกว่า $y"
        x < y       -> "$x น้อยกว่า $y"
        else        -> "$x เท่ากับ $y"
    }
    println(result)
    
    // ใช้งานจริง: ตรวจสอบหลายเงื่อนไขพร้อมกัน
    val temperature = 38
    val humidity = 85
    val isRaining = false
    
    val weatherAdvice = when {
        temperature > 35 && humidity > 80 -> "ร้อนชื้นมาก หลีกเลี่ยงกิจกรรมกลางแจ้ง"
        temperature > 35                  -> "ร้อนมาก ดื่มน้ำให้เยอะ"
        isRaining                         -> "ฝนตก พกร่มด้วย"
        temperature < 15                  -> "หนาว ใส่เสื้อกันหนาว"
        else                              -> "อากาศปกติ เหมาะสำหรับกิจกรรมกลางแจ้ง"
    }
    println("อุณหภูมิ ${temperature}°C ความชื้น ${humidity}%")
    println("คำแนะนำ: $weatherAdvice")
    
    // เปรียบเทียบ: when vs if-else
    // when: อ่านง่ายกว่า โดยเฉพาะเมื่อมีหลาย branch
    // if-else: ยืดหยุ่นกว่า เหมาะกับ complex conditions
}
```

---

## Nested Conditionals

```kotlin
fun main() {
    // ระบบตรวจสอบสิทธิ์
    val isLoggedIn = true
    val userRole = "admin"  // "admin", "user", "guest"
    val requestedResource = "user_management"
    
    if (isLoggedIn) {
        when (userRole) {
            "admin" -> {
                println("ยินดีต้อนรับ Admin!")
                println("คุณมีสิทธิ์เข้าถึงทุกอย่าง")
                
                when (requestedResource) {
                    "user_management" -> println("เปิดหน้า User Management")
                    "reports"         -> println("เปิดรายงาน")
                    "settings"        -> println("เปิด Settings")
                    else              -> println("เปิดหน้า: $requestedResource")
                }
            }
            "user" -> {
                println("ยินดีต้อนรับ User!")
                if (requestedResource in listOf("profile", "dashboard")) {
                    println("เปิดหน้า: $requestedResource")
                } else {
                    println("คุณไม่มีสิทธิ์เข้าถึง $requestedResource")
                }
            }
            "guest" -> {
                println("Guest: เข้าถึงได้เฉพาะหน้า public")
                if (requestedResource == "home") {
                    println("เปิดหน้าหลัก")
                } else {
                    println("กรุณาเข้าสู่ระบบก่อน")
                }
            }
            else -> println("Role ไม่รู้จัก: $userRole")
        }
    } else {
        println("กรุณาเข้าสู่ระบบ")
    }
    
    // ⚠️ ระวัง: Nested มากเกินไป → อ่านยาก (Callback Hell)
    // แนะนำ: ใช้ Early Return แทน (ดูหัวข้อถัดไป)
}
```

---

## Guard Clauses / Early Return

Pattern นี้ช่วยลด Nesting และทำให้โค้ดอ่านง่ายขึ้น:

```kotlin
// ❌ แบบที่ไม่ดี - Deep Nesting
fun processOrderBad(
    orderId: Int?,
    items: List<String>?,
    userId: Int?,
    discount: Double
): String {
    if (orderId != null) {
        if (items != null) {
            if (items.isNotEmpty()) {
                if (userId != null) {
                    if (discount in 0.0..1.0) {
                        // Logic หลักอยู่ลึกมาก 5 ชั้น!
                        return "สำเร็จ: Order #$orderId สำหรับ User #$userId"
                    } else {
                        return "ส่วนลดไม่ถูกต้อง"
                    }
                } else {
                    return "ไม่พบ User"
                }
            } else {
                return "ไม่มีสินค้าในออเดอร์"
            }
        } else {
            return "รายการสินค้าเป็น null"
        }
    } else {
        return "Order ID ไม่ถูกต้อง"
    }
}

// ✅ แบบที่ดี - Guard Clauses (Early Return)
fun processOrder(
    orderId: Int?,
    items: List<String>?,
    userId: Int?,
    discount: Double
): String {
    // Guard Clauses: ตรวจสอบเงื่อนไขที่ fail ก่อน
    if (orderId == null) return "Order ID ไม่ถูกต้อง"
    if (items == null) return "รายการสินค้าเป็น null"
    if (items.isEmpty()) return "ไม่มีสินค้าในออเดอร์"
    if (userId == null) return "ไม่พบ User"
    if (discount !in 0.0..1.0) return "ส่วนลดไม่ถูกต้อง"
    
    // Logic หลักอยู่ระดับเดียว อ่านง่ายมาก!
    val totalItems = items.size
    val orderInfo = "Order #$orderId: $totalItems รายการ"
    return "สำเร็จ: $orderInfo สำหรับ User #$userId (ส่วนลด ${discount * 100}%)"
}

fun main() {
    println(processOrder(null, listOf("a"), 1, 0.1))
    println(processOrder(1, listOf(), 1, 0.1))
    println(processOrder(1, listOf("a", "b"), 1, 0.1))
    println(processOrder(1, listOf("a", "b"), 1, 1.5))
}
```

**Output:**
```
Order ID ไม่ถูกต้อง
ไม่มีสินค้าในออเดอร์
สำเร็จ: Order #1: 2 รายการ สำหรับ User #1 (ส่วนลด 10.0%)
ส่วนลดไม่ถูกต้อง
```

### require() และ check()

```kotlin
fun calculateDiscount(price: Double, discountPercent: Int): Double {
    // require: ตรวจสอบ input argument
    require(price > 0) { "ราคาต้องมากกว่า 0 แต่ได้ $price" }
    require(discountPercent in 0..100) { "ส่วนลดต้อง 0-100% แต่ได้ $discountPercent" }
    
    return price * (1 - discountPercent / 100.0)
}

fun getUserProfile(userId: Int): Map<String, String> {
    val profile = mapOf("name" to "สมชาย", "email" to "somchai@email.com")
    
    // check: ตรวจสอบ internal state
    check(profile.isNotEmpty()) { "Profile ต้องไม่ว่าง" }
    
    return profile
}

fun main() {
    println(calculateDiscount(100.0, 20))  // 80.0
    
    try {
        calculateDiscount(-50.0, 20)
    } catch (e: IllegalArgumentException) {
        println("Error: ${e.message}")
    }
    
    try {
        calculateDiscount(100.0, 150)
    } catch (e: IllegalArgumentException) {
        println("Error: ${e.message}")
    }
}
```

---

## ตัวอย่างโปรแกรมสรุปรวม

### ระบบ ATM อย่างง่าย

```kotlin
fun main() {
    var balance = 15_000.0
    val pin = "1234"
    var attempts = 0
    
    println("╔════════════════════════╗")
    println("║     ยินดีต้อนรับ ATM     ║")
    println("╚════════════════════════╝")
    
    // ตรวจสอบ PIN
    var authenticated = false
    while (attempts < 3) {
        print("กรุณาใส่ PIN (4 หลัก): ")
        val inputPin = readln()
        
        if (inputPin == pin) {
            authenticated = true
            break
        } else {
            attempts++
            val remaining = 3 - attempts
            if (remaining > 0) {
                println("PIN ไม่ถูกต้อง! เหลือ $remaining ครั้ง")
            }
        }
    }
    
    if (!authenticated) {
        println("บัญชีถูกล็อค กรุณาติดต่อธนาคาร")
        return
    }
    
    println("\nเข้าสู่ระบบสำเร็จ!")
    var continueSession = true
    
    while (continueSession) {
        println("\n=== เมนู ===")
        println("1. เช็คยอด")
        println("2. ถอนเงิน")
        println("3. ฝากเงิน")
        println("4. โอนเงิน")
        println("0. ออก")
        print("เลือก: ")
        
        when (readln()) {
            "1" -> {
                println("ยอดเงินคงเหลือ: ${"%,.2f".format(balance)} บาท")
            }
            "2" -> {
                print("จำนวนเงินที่ต้องการถอน: ")
                val amount = readln().toDoubleOrNull() ?: 0.0
                
                when {
                    amount <= 0        -> println("จำนวนเงินไม่ถูกต้อง")
                    amount % 100 != 0.0 -> println("กรุณาถอนเป็นทวีคูณของ 100 บาท")
                    amount > balance   -> println("ยอดเงินไม่เพียงพอ")
                    amount > 50_000    -> println("ถอนได้สูงสุด 50,000 บาทต่อครั้ง")
                    else -> {
                        balance -= amount
                        println("ถอนเงินสำเร็จ: ${"%,.2f".format(amount)} บาท")
                        println("ยอดคงเหลือ: ${"%,.2f".format(balance)} บาท")
                    }
                }
            }
            "3" -> {
                print("จำนวนเงินที่ต้องการฝาก: ")
                val amount = readln().toDoubleOrNull() ?: 0.0
                
                if (amount <= 0) {
                    println("จำนวนเงินไม่ถูกต้อง")
                } else {
                    balance += amount
                    println("ฝากเงินสำเร็จ: ${"%,.2f".format(amount)} บาท")
                    println("ยอดคงเหลือ: ${"%,.2f".format(balance)} บาท")
                }
            }
            "4" -> {
                print("บัญชีปลายทาง: ")
                val destAccount = readln()
                print("จำนวนเงิน: ")
                val amount = readln().toDoubleOrNull() ?: 0.0
                
                when {
                    destAccount.isBlank() -> println("กรุณาระบุบัญชีปลายทาง")
                    amount <= 0           -> println("จำนวนเงินไม่ถูกต้อง")
                    amount > balance      -> println("ยอดเงินไม่เพียงพอ")
                    else -> {
                        balance -= amount
                        println("โอนเงินสำเร็จ!")
                        println("ไปยัง: $destAccount จำนวน: ${"%,.2f".format(amount)} บาท")
                        println("ยอดคงเหลือ: ${"%,.2f".format(balance)} บาท")
                    }
                }
            }
            "0" -> {
                println("ขอบคุณที่ใช้บริการ!")
                continueSession = false
            }
            else -> println("เลือกไม่ถูกต้อง กรุณาลองใหม่")
        }
    }
}
```

---

## แบบฝึกหัด

### Exercise 1: ตรวจสอบอายุ
เขียน when expression แสดงช่วงอายุ:

```kotlin
fun getAgeGroup(age: Int): String = when {
    age < 0    -> "อายุไม่ถูกต้อง"
    age < 1    -> "ทารก"
    age < 3    -> "เด็กเล็ก"
    age < 13   -> "เด็ก"
    age < 18   -> "วัยรุ่น"
    age < 60   -> "ผู้ใหญ่"
    age < 80   -> "ผู้สูงอายุ"
    else       -> "ผู้สูงอายุมาก"
}

fun main() {
    val ages = listOf(-1, 0, 2, 5, 15, 25, 65, 85)
    for (age in ages) {
        println("อายุ $age ปี: ${getAgeGroup(age)}")
    }
}
```

### Exercise 2: ตรวจสอบปีราศี
```kotlin
fun getZodiac(year: Int): String {
    val zodiacs = listOf(
        "ลิง", "ไก่", "หมา", "หมู", "หนู", "วัว",
        "เสือ", "กระต่าย", "มังกร", "งู", "ม้า", "แพะ"
    )
    val index = (year - 4) % 12
    return zodiacs[if (index < 0) index + 12 else index]
}

fun main() {
    val years = listOf(1990, 2000, 2024, 2025)
    for (year in years) {
        println("ปี $year: ปี${getZodiac(year)}")
    }
}
```

### Exercise 3: ท้าทาย - Rock Paper Scissors

```kotlin
import kotlin.random.Random

fun main() {
    val choices = listOf("กรรไกร", "หิน", "กระดาษ")
    
    println("=== เป่าค้อน-กรรไกร-กระดาษ ===")
    println("1=กรรไกร 2=หิน 3=กระดาษ")
    print("เลือก (1-3): ")
    
    val playerChoice = readln().toIntOrNull()?.minus(1)
    if (playerChoice == null || playerChoice !in 0..2) {
        println("เลือกไม่ถูกต้อง!")
        return
    }
    
    val computerChoice = Random.nextInt(3)
    
    println("\nคุณเลือก: ${choices[playerChoice]}")
    println("คอมพิวเตอร์เลือก: ${choices[computerChoice]}")
    
    val result = when {
        playerChoice == computerChoice -> "เสมอ! 🤝"
        (playerChoice == 0 && computerChoice == 2) ||  // กรรไกร > กระดาษ
        (playerChoice == 1 && computerChoice == 0) ||  // หิน > กรรไกร
        (playerChoice == 2 && computerChoice == 1) ->  // กระดาษ > หิน
            "คุณชนะ! 🎉"
        else -> "คอมพิวเตอร์ชนะ! 🤖"
    }
    
    println("\nผลลัพธ์: $result")
}
```

---

## สรุป Part 05

```
✅ if-else: การตรวจสอบเงื่อนไขพื้นฐาน
✅ if เป็น Expression: return ค่าได้ (แทน ternary operator)
✅ when: switch ที่ทรงพลัง รองรับหลาย pattern
✅ when patterns: ค่าเดียว, หลายค่า, range, condition, type
✅ when โดยไม่มี argument: เหมือน if-else chain
✅ Guard Clauses: Early return เพื่อลด nesting
✅ require() / check(): built-in validation
✅ Smart Cast: หลังจาก is check, type ถูก cast อัตโนมัติ
```

---

*Part 05/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
