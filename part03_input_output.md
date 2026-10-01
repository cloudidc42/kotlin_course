# Part 03: การรับและแสดงผลข้อมูล (Input/Output)

## สารบัญ
1. [การแสดงผล (Output)](#การแสดงผล-output)
2. [การรับข้อมูล (Input)](#การรับข้อมูล-input)
3. [readLine() และ readln()](#readline-และ-readln)
4. [การแปลงข้อมูล Input](#การแปลงข้อมูล-input)
5. [Scanner](#scanner)
6. [การ Format Output](#การ-format-output)
7. [โปรแกรมตัวอย่าง](#โปรแกรมตัวอย่าง)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## การแสดงผล (Output)

### ฟังก์ชันแสดงผลพื้นฐาน

```kotlin
// println - แสดงผลพร้อมขึ้นบรรทัดใหม่
println("Hello, World!")
println(42)
println(3.14)
println(true)
println()  // แสดงบรรทัดว่าง

// print - แสดงผลโดยไม่ขึ้นบรรทัดใหม่
print("Hello ")
print("Kotlin")
print("!")
println()  // ต้องใส่เองถ้าต้องการขึ้นบรรทัดใหม่

// Output: Hello Kotlin!

// printlnหลายค่า
println("ชื่อ: สมชาย, อายุ: 25")
val name = "สมชาย"
val age = 25
println("ชื่อ: $name, อายุ: $age")  // String Template
```

### print vs println

```kotlin
// print ไม่ขึ้นบรรทัดใหม่ - ใช้สำหรับแสดงในบรรทัดเดียวกัน
for (i in 1..5) {
    print("$i ")
}
println()  // ขึ้นบรรทัดใหม่ตอนท้าย
// Output: 1 2 3 4 5

// println แต่ละครั้งขึ้นบรรทัดใหม่
for (i in 1..5) {
    println(i)
}
// Output:
// 1
// 2
// 3
// 4
// 5
```

### System.out.printf (Java Style)

```kotlin
// ใช้ printf แบบ Java ได้
System.out.printf("ชื่อ: %s, อายุ: %d ปี%n", "สมชาย", 25)
System.out.printf("ราคา: %.2f บาท%n", 1234.5)
System.out.printf("เลขฐาน 16: %x%n", 255)
System.out.printf("เลขฐาน 8: %o%n", 255)
System.out.printf("เลขฐาน 2: %s%n", Integer.toBinaryString(255))

// Output:
// ชื่อ: สมชาย, อายุ: 25 ปี
// ราคา: 1234.50 บาท
// เลขฐาน 16: ff
// เลขฐาน 8: 377
// เลขฐาน 2: 11111111
```

### String Format

```kotlin
// String.format
val formatted = String.format("ชื่อ: %-10s อายุ: %3d", "สมชาย", 25)
println(formatted)  // ชื่อ: สมชาย     อายุ:  25

// Format Specifiers ที่ใช้บ่อย:
// %d - Integer
// %f - Float/Double
// %s - String
// %c - Char
// %b - Boolean
// %n - Newline (platform-specific)
// %x - Hexadecimal
// %o - Octal
// %e - Scientific notation

// Width และ Precision
println("%10d".format(42))       // "        42" (pad ด้านซ้าย)
println("%-10d".format(42))      // "42        " (pad ด้านขวา)
println("%010d".format(42))      // "0000000042" (pad ด้วย 0)
println("%+d".format(42))        // "+42"
println("%+d".format(-42))       // "-42"
println("%.4f".format(3.14159))  // "3.1416"
println("%10.2f".format(1234.5)) // "   1234.50"

// Kotlin Extension
val price = 1234567.89
val formatted2 = "ราคา: %,.2f บาท".format(price)
println(formatted2)  // ราคา: 1,234,567.89 บาท
```

---

## การรับข้อมูล (Input)

### วิธีรับข้อมูลใน Kotlin

Kotlin มีหลายวิธีรับข้อมูลจาก Standard Input (Keyboard):

```
1. readLine() - ฟังก์ชันมาตรฐานของ Kotlin (return String?)
2. readln() - Kotlin 1.6+ (return String ไม่ใช่ nullable)
3. Scanner - Java Scanner Class
4. BufferedReader - สำหรับ I/O ที่ต้องการประสิทธิภาพสูง
```

---

## readLine() และ readln()

### readLine()

```kotlin
fun main() {
    print("กรุณาใส่ชื่อของคุณ: ")
    val name = readLine()  // return String? (Nullable)
    
    println("สวัสดี, $name!")
    
    // ต้องจัดการ null (ถ้าผู้ใช้กด Ctrl+D/EOF)
    val nameNonNull = readLine() ?: "ไม่ทราบชื่อ"
    println("สวัสดี, $nameNonNull!")
}
```

### readln() (Kotlin 1.6+)

```kotlin
fun main() {
    print("กรุณาใส่ชื่อของคุณ: ")
    val name = readln()  // return String (ไม่ nullable)
    // ถ้า EOF จะ throw RuntimeException
    
    println("สวัสดี, $name!")
}
```

### ตัวอย่างการรับข้อมูลหลายบรรทัด

```kotlin
fun main() {
    println("=== แบบฟอร์มข้อมูลส่วนตัว ===")
    
    print("ชื่อ: ")
    val firstName = readln()
    
    print("นามสกุล: ")
    val lastName = readln()
    
    print("อายุ: ")
    val ageStr = readln()
    val age = ageStr.toIntOrNull() ?: 0
    
    print("อีเมล: ")
    val email = readln()
    
    println("\n=== ข้อมูลที่รับได้ ===")
    println("ชื่อเต็ม: $firstName $lastName")
    println("อายุ: $age ปี")
    println("อีเมล: $email")
    println("เป็นผู้ใหญ่: ${if (age >= 18) "ใช่" else "ไม่ใช่"}")
}
```

**ตัวอย่าง Interaction:**
```
=== แบบฟอร์มข้อมูลส่วนตัว ===
ชื่อ: สมชาย
นามสกุล: ใจดี
อายุ: 25
อีเมล: somchai@example.com

=== ข้อมูลที่รับได้ ===
ชื่อเต็ม: สมชาย ใจดี
อายุ: 25 ปี
อีเมล: somchai@example.com
เป็นผู้ใหญ่: ใช่
```

---

## การแปลงข้อมูล Input

เนื่องจาก readLine()/readln() return `String` เสมอ จึงต้องแปลง Type ก่อนใช้งาน:

```kotlin
fun main() {
    // รับและแปลงเป็น Int
    print("ใส่ตัวเลข: ")
    val numStr = readln()
    
    // วิธีที่ 1: toInt() - throw exception ถ้าแปลงไม่ได้
    val num1 = numStr.toInt()
    
    // วิธีที่ 2: toIntOrNull() - return null ถ้าแปลงไม่ได้ (แนะนำ)
    val num2 = numStr.toIntOrNull()
    if (num2 == null) {
        println("ข้อมูลไม่ถูกต้อง!")
        return
    }
    
    // วิธีที่ 3: try-catch
    val num3 = try {
        numStr.toInt()
    } catch (e: NumberFormatException) {
        println("รูปแบบตัวเลขไม่ถูกต้อง: ${e.message}")
        0  // ค่า default
    }
    
    println("ผลลัพธ์: $num3")
}
```

### ฟังก์ชัน Helper สำหรับรับข้อมูล

```kotlin
// ฟังก์ชัน helper รับ Int
fun readInt(prompt: String, errorMsg: String = "ข้อมูลไม่ถูกต้อง"): Int {
    while (true) {
        print(prompt)
        val input = readln()
        val num = input.toIntOrNull()
        if (num != null) return num
        println(errorMsg)
    }
}

// ฟังก์ชัน helper รับ Double
fun readDouble(prompt: String): Double {
    while (true) {
        print(prompt)
        val input = readln()
        val num = input.toDoubleOrNull()
        if (num != null) return num
        println("กรุณาใส่ตัวเลขทศนิยม")
    }
}

// ฟังก์ชัน helper รับ String ที่ไม่ว่าง
fun readNonEmpty(prompt: String): String {
    while (true) {
        print(prompt)
        val input = readln().trim()
        if (input.isNotEmpty()) return input
        println("กรุณากรอกข้อมูล")
    }
}

fun main() {
    val name = readNonEmpty("ชื่อ: ")
    val age = readInt("อายุ: ", "กรุณาใส่อายุเป็นตัวเลข")
    val salary = readDouble("เงินเดือน: ")
    
    println("\n=== ข้อมูลพนักงาน ===")
    println("ชื่อ: $name")
    println("อายุ: $age ปี")
    println("เงินเดือน: ${String.format("%,.2f", salary)} บาท")
}
```

---

## Scanner

`Scanner` เป็น Java class ที่ให้ความสามารถในการอ่านข้อมูลแบบต่างๆ:

```kotlin
import java.util.Scanner

fun main() {
    val scanner = Scanner(System.`in`)
    
    // อ่าน String
    print("ชื่อ: ")
    val name = scanner.nextLine()
    
    // อ่าน Int
    print("อายุ: ")
    val age = scanner.nextInt()
    scanner.nextLine()  // consume newline ที่เหลือ
    
    // อ่าน Double
    print("เงินเดือน: ")
    val salary = scanner.nextDouble()
    scanner.nextLine()  // consume newline
    
    println("\nข้อมูล: $name, $age ปี, ${salary} บาท")
    
    scanner.close()  // ปิด Scanner เสมอ
}
```

### Scanner กับประเภทข้อมูลต่างๆ

```kotlin
import java.util.Scanner

fun main() {
    val scanner = Scanner(System.`in`)
    
    println("=== ทดสอบ Scanner ===")
    
    // nextLine() - อ่านทั้งบรรทัด
    print("Enter full line: ")
    val line = scanner.nextLine()
    println("Line: '$line'")
    
    // next() - อ่านคำถัดไป (แบ่งด้วย whitespace)
    print("Enter two words: ")
    val word1 = scanner.next()
    val word2 = scanner.next()
    scanner.nextLine()  // consume remaining newline
    println("Words: '$word1', '$word2'")
    
    // nextInt(), nextLong(), nextDouble(), nextFloat()
    print("Enter integer: ")
    val num = scanner.nextInt()
    scanner.nextLine()
    println("Int: $num")
    
    // hasNextInt() - ตรวจสอบก่อนอ่าน
    print("Enter numbers (non-number to stop): ")
    val numbers = mutableListOf<Int>()
    while (scanner.hasNextInt()) {
        numbers.add(scanner.nextInt())
    }
    println("Numbers: $numbers")
    
    scanner.close()
}
```

### Scanner อ่านจาก String

```kotlin
import java.util.Scanner

fun main() {
    // Scanner สามารถอ่านจาก String ได้
    val input = "สมชาย 25 50000.0 true"
    val scanner = Scanner(input)
    
    val name = scanner.next()
    val age = scanner.nextInt()
    val salary = scanner.nextDouble()
    val isEmployee = scanner.nextBoolean()
    
    println("ชื่อ: $name")
    println("อายุ: $age")
    println("เงินเดือน: $salary")
    println("เป็นพนักงาน: $isEmployee")
    
    scanner.close()
}
```

---

## การ Format Output

### ตาราง (Table Formatting)

```kotlin
fun main() {
    val employees = listOf(
        Triple("สมชาย ใจดี", 25, 45000.0),
        Triple("สมหญิง รักดี", 30, 65000.0),
        Triple("สมศักดิ์ มีสุข", 28, 55000.0)
    )
    
    // หัวตาราง
    println("=".repeat(52))
    println("%-20s %5s %12s".format("ชื่อ", "อายุ", "เงินเดือน"))
    println("=".repeat(52))
    
    // ข้อมูล
    for ((name, age, salary) in employees) {
        println("%-20s %5d %,12.2f".format(name, age, salary))
    }
    
    // สรุป
    println("-".repeat(52))
    val total = employees.sumOf { it.third }
    val avg = total / employees.size
    println("%-20s %5s %,12.2f".format("รวมเงินเดือน", "", total))
    println("%-20s %5s %,12.2f".format("เฉลี่ย", "", avg))
    println("=".repeat(52))
}
```

**Output:**
```
====================================================
ชื่อ                 อายุ    เงินเดือน
====================================================
สมชาย ใจดี           25    45,000.00
สมหญิง รักดี          30    65,000.00
สมศักดิ์ มีสุข        28    55,000.00
----------------------------------------------------
รวมเงินเดือน                165,000.00
เฉลี่ย                       55,000.00
====================================================
```

### NumberFormat

```kotlin
import java.text.NumberFormat
import java.util.Locale

fun main() {
    val amount = 1234567.89
    
    // Thai Locale
    val thaiFormatter = NumberFormat.getCurrencyInstance(Locale("th", "TH"))
    println(thaiFormatter.format(amount))  // ฿1,234,567.89
    
    // US Locale
    val usFormatter = NumberFormat.getCurrencyInstance(Locale.US)
    println(usFormatter.format(amount))  // $1,234,567.89
    
    // Number Format
    val numFormatter = NumberFormat.getNumberInstance()
    numFormatter.maximumFractionDigits = 2
    println(numFormatter.format(amount))  // 1,234,567.89
    
    // Percentage
    val pctFormatter = NumberFormat.getPercentInstance()
    pctFormatter.maximumFractionDigits = 1
    println(pctFormatter.format(0.7523))  // 75.2%
}
```

### DateTimeFormatter

```kotlin
import java.time.LocalDateTime
import java.time.format.DateTimeFormatter

fun main() {
    val now = LocalDateTime.now()
    
    // Patterns
    val patterns = mapOf(
        "วันที่" to "dd/MM/yyyy",
        "เวลา" to "HH:mm:ss",
        "วันที่และเวลา" to "dd/MM/yyyy HH:mm:ss",
        "Thai Style" to "d MMMM yyyy HH:mm",
        "ISO" to "yyyy-MM-dd'T'HH:mm:ss"
    )
    
    for ((label, pattern) in patterns) {
        val formatter = DateTimeFormatter.ofPattern(pattern)
        println("$label: ${now.format(formatter)}")
    }
}
```

**Output (ตัวอย่าง):**
```
วันที่: 15/01/2024
เวลา: 14:30:25
วันที่และเวลา: 15/01/2024 14:30:25
Thai Style: 15 January 2024 14:30
ISO: 2024-01-15T14:30:25
```

---

## โปรแกรมตัวอย่าง

### โปรแกรมคำนวณคะแนนนักเรียน

```kotlin
fun main() {
    println("╔══════════════════════════════════════╗")
    println("║      ระบบคำนวณคะแนนนักเรียน           ║")
    println("╚══════════════════════════════════════╝")
    
    print("\nชื่อนักเรียน: ")
    val name = readln()
    
    println("\nใส่คะแนน 5 วิชา (แต่ละวิชา 0-100):")
    val subjects = listOf("คณิตศาสตร์", "ภาษาไทย", "ภาษาอังกฤษ", "วิทยาศาสตร์", "สังคมศึกษา")
    val scores = mutableListOf<Double>()
    
    for (subject in subjects) {
        while (true) {
            print("  $subject: ")
            val scoreStr = readln()
            val score = scoreStr.toDoubleOrNull()
            if (score != null && score in 0.0..100.0) {
                scores.add(score)
                break
            }
            println("  ❌ กรุณาใส่คะแนน 0-100")
        }
    }
    
    val total = scores.sum()
    val average = total / scores.size
    val maxScore = scores.max()
    val minScore = scores.min()
    
    val grade = when {
        average >= 80 -> "A"
        average >= 70 -> "B"
        average >= 60 -> "C"
        average >= 50 -> "D"
        else -> "F"
    }
    
    println("\n╔══════════════════════════════════════╗")
    println("║              ผลการเรียน               ║")
    println("╠══════════════════════════════════════╣")
    println("║ นักเรียน: %-28s║".format(name))
    println("╠══════════════════════════════════════╣")
    
    for (i in subjects.indices) {
        println("║ %-14s : %5.1f คะแนน            ║".format(subjects[i], scores[i]))
    }
    
    println("╠══════════════════════════════════════╣")
    println("║ รวม         : %5.1f คะแนน            ║".format(total))
    println("║ เฉลี่ย       : %5.1f คะแนน            ║".format(average))
    println("║ สูงสุด       : %5.1f คะแนน            ║".format(maxScore))
    println("║ ต่ำสุด       : %5.1f คะแนน            ║".format(minScore))
    println("║ เกรด        : %-28s║".format(grade))
    println("╚══════════════════════════════════════╝")
}
```

**ตัวอย่าง Output:**
```
╔══════════════════════════════════════╗
║      ระบบคำนวณคะแนนนักเรียน           ║
╚══════════════════════════════════════╝

ชื่อนักเรียน: สมชาย ใจดี

ใส่คะแนน 5 วิชา (แต่ละวิชา 0-100):
  คณิตศาสตร์: 85
  ภาษาไทย: 78
  ภาษาอังกฤษ: 92
  วิทยาศาสตร์: 88
  สังคมศึกษา: 75

╔══════════════════════════════════════╗
║              ผลการเรียน               ║
╠══════════════════════════════════════╣
║ นักเรียน: สมชาย ใจดี                  ║
╠══════════════════════════════════════╣
║ คณิตศาสตร์    :  85.0 คะแนน            ║
║ ภาษาไทย       :  78.0 คะแนน            ║
║ ภาษาอังกฤษ    :  92.0 คะแนน            ║
║ วิทยาศาสตร์   :  88.0 คะแนน            ║
║ สังคมศึกษา    :  75.0 คะแนน            ║
╠══════════════════════════════════════╣
║ รวม         : 418.0 คะแนน            ║
║ เฉลี่ย       :  83.6 คะแนน            ║
║ สูงสุด       :  92.0 คะแนน            ║
║ ต่ำสุด       :  75.0 คะแนน            ║
║ เกรด        : A                       ║
╚══════════════════════════════════════╝
```

### โปรแกรมแปลงหน่วย

```kotlin
fun main() {
    println("=== โปรแกรมแปลงหน่วย ===")
    println("1. เมตร → ฟุต")
    println("2. กิโลกรัม → ปอนด์")
    println("3. เซลเซียส → ฟาเรนไฮต์")
    println("4. บาท → ดอลลาร์")
    print("\nเลือกการแปลง (1-4): ")
    
    val choice = readln().toIntOrNull() ?: 0
    
    when (choice) {
        1 -> {
            print("ใส่ค่าเป็นเมตร: ")
            val meters = readln().toDoubleOrNull() ?: 0.0
            val feet = meters * 3.28084
            println("$meters เมตร = %.4f ฟุต".format(feet))
        }
        2 -> {
            print("ใส่ค่าเป็นกิโลกรัม: ")
            val kg = readln().toDoubleOrNull() ?: 0.0
            val lbs = kg * 2.20462
            println("$kg กิโลกรัม = %.4f ปอนด์".format(lbs))
        }
        3 -> {
            print("ใส่อุณหภูมิเซลเซียส: ")
            val celsius = readln().toDoubleOrNull() ?: 0.0
            val fahrenheit = celsius * 9.0 / 5.0 + 32
            println("$celsius°C = %.2f°F".format(fahrenheit))
        }
        4 -> {
            print("ใส่จำนวนบาท: ")
            val thb = readln().toDoubleOrNull() ?: 0.0
            val exchangeRate = 35.5  // อัตราแลกเปลี่ยนสมมติ
            val usd = thb / exchangeRate
            println("${String.format("%,.2f", thb)} บาท = ${String.format("%.2f", usd)} USD")
            println("(อัตราแลกเปลี่ยน: 1 USD = $exchangeRate THB)")
        }
        else -> println("ตัวเลือกไม่ถูกต้อง!")
    }
}
```

---

## แบบฝึกหัด

### Exercise 1: รับข้อมูลพื้นฐาน
เขียนโปรแกรมรับชื่อ อายุ และวิชาโปรด แล้วแสดงผล:

```kotlin
// Solution
fun main() {
    print("ชื่อของคุณ: ")
    val name = readln()
    
    print("อายุของคุณ: ")
    val age = readln().toIntOrNull() ?: 0
    
    print("วิชาโปรด: ")
    val subject = readln()
    
    val birthYear = 2024 - age
    
    println("\n=== ข้อมูลของคุณ ===")
    println("ชื่อ: $name")
    println("อายุ: $age ปี (เกิดปี $birthYear)")
    println("วิชาโปรด: $subject")
    println("ยินดีต้อนรับ $name! ขอให้สนุกกับการเรียน $subject นะครับ/ค่ะ")
}
```

### Exercise 2: เครื่องคิดเลข
เขียน Calculator ที่รับตัวเลข 2 ตัวและเครื่องหมาย:

```kotlin
// Solution
fun main() {
    println("=== เครื่องคิดเลข ===")
    
    print("ตัวเลขที่ 1: ")
    val num1 = readln().toDoubleOrNull() ?: run {
        println("ข้อมูลไม่ถูกต้อง")
        return
    }
    
    print("เครื่องหมาย (+, -, *, /): ")
    val operator = readln()
    
    print("ตัวเลขที่ 2: ")
    val num2 = readln().toDoubleOrNull() ?: run {
        println("ข้อมูลไม่ถูกต้อง")
        return
    }
    
    val result = when (operator) {
        "+" -> num1 + num2
        "-" -> num1 - num2
        "*" -> num1 * num2
        "/" -> {
            if (num2 == 0.0) {
                println("ไม่สามารถหารด้วยศูนย์ได้!")
                return
            }
            num1 / num2
        }
        else -> {
            println("เครื่องหมายไม่รู้จัก!")
            return
        }
    }
    
    println("\n$num1 $operator $num2 = $result")
}
```

### Exercise 3: ท้าทาย - เมนูวนซ้ำ

```kotlin
// Solution
fun main() {
    var running = true
    
    while (running) {
        println("\n=== เมนูหลัก ===")
        println("1. แสดง Hello")
        println("2. คำนวณพื้นที่วงกลม")
        println("3. แปลงอุณหภูมิ")
        println("0. ออก")
        print("เลือก: ")
        
        when (readln()) {
            "1" -> {
                print("ชื่อของคุณ: ")
                val name = readln()
                println("สวัสดี, $name! ยินดีต้อนรับ!")
            }
            "2" -> {
                print("รัศมี: ")
                val r = readln().toDoubleOrNull() ?: 0.0
                val area = Math.PI * r * r
                println("พื้นที่ = %.4f".format(area))
            }
            "3" -> {
                print("อุณหภูมิ (°C): ")
                val c = readln().toDoubleOrNull() ?: 0.0
                val f = c * 9.0 / 5.0 + 32
                println("$c°C = %.2f°F".format(f))
            }
            "0" -> {
                println("ลาก่อน!")
                running = false
            }
            else -> println("ตัวเลือกไม่ถูกต้อง!")
        }
    }
}
```

---

## สรุป Part 03

```
✅ println() แสดงผลพร้อมขึ้นบรรทัดใหม่
✅ print() แสดงผลโดยไม่ขึ้นบรรทัดใหม่
✅ String.format() / printf สำหรับ format output
✅ readLine() รับข้อมูล String? (nullable)
✅ readln() รับข้อมูล String (Kotlin 1.6+)
✅ toIntOrNull(), toDoubleOrNull() แปลงอย่างปลอดภัย
✅ Scanner สำหรับรับข้อมูลหลาย Type
✅ NumberFormat, DateTimeFormatter สำหรับ format ขั้นสูง
```

---

*Part 03/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
