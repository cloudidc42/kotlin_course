# Part 06: การวนซ้ำ (Loops)

## สารบัญ
1. [for Loop](#for-loop)
2. [while Loop](#while-loop)
3. [do-while Loop](#do-while-loop)
4. [break และ continue](#break-และ-continue)
5. [Labels](#labels)
6. [repeat()](#repeat)
7. [forEach และ Collection Iteration](#foreach-และ-collection-iteration)
8. [Sequence และ Lazy Evaluation](#sequence-และ-lazy-evaluation)
9. [แบบฝึกหัด](#แบบฝึกหัด)

---

## for Loop

### for กับ Range

```kotlin
fun main() {
    // for วนจาก 1 ถึง 5 (inclusive)
    for (i in 1..5) {
        print("$i ")
    }
    println()  // 1 2 3 4 5
    
    // for วนจาก 1 ถึง 4 (ไม่รวม 5) ด้วย until
    for (i in 1 until 5) {
        print("$i ")
    }
    println()  // 1 2 3 4
    
    // for ถอยหลัง ด้วย downTo
    for (i in 5 downTo 1) {
        print("$i ")
    }
    println()  // 5 4 3 2 1
    
    // for กับ step (กำหนดช่วง)
    for (i in 0..20 step 5) {
        print("$i ")
    }
    println()  // 0 5 10 15 20
    
    // for ถอยหลังทีละ 2
    for (i in 10 downTo 1 step 2) {
        print("$i ")
    }
    println()  // 10 8 6 4 2
    
    // for กับตัวอักษร
    for (c in 'A'..'F') {
        print("$c ")
    }
    println()  // A B C D E F
}
```

### for กับ Collection

```kotlin
fun main() {
    // List
    val fruits = listOf("แอปเปิ้ล", "กล้วย", "ส้ม", "มะม่วง")
    
    for (fruit in fruits) {
        println("ผลไม้: $fruit")
    }
    
    // for กับ index ด้วย indices
    for (i in fruits.indices) {
        println("${i + 1}. ${fruits[i]}")
    }
    
    // forEachIndexed (แนะนำมากกว่า)
    fruits.forEachIndexed { index, fruit ->
        println("${index + 1}. $fruit")
    }
    
    // Map
    val ages = mapOf("สมชาย" to 25, "สมหญิง" to 30, "สมศักดิ์" to 28)
    
    for ((name, age) in ages) {
        println("$name อายุ $age ปี")
    }
    
    // Array
    val numbers = arrayOf(10, 20, 30, 40, 50)
    for (n in numbers) {
        print("$n ")
    }
    println()
    
    // IntArray (Primitive Array)
    val primes = intArrayOf(2, 3, 5, 7, 11)
    for (p in primes) {
        print("$p ")
    }
    println()
}
```

### for กับ withIndex()

```kotlin
fun main() {
    val colors = listOf("แดง", "เขียว", "น้ำเงิน", "เหลือง")
    
    // withIndex() ให้ pair ของ (index, value)
    for ((index, color) in colors.withIndex()) {
        println("[$index] $color")
    }
    
    // Output:
    // [0] แดง
    // [1] เขียว
    // [2] น้ำเงิน
    // [3] เหลือง
    
    // ค้นหาด้วย forEachIndexed
    val target = "น้ำเงิน"
    colors.forEachIndexed { i, color ->
        if (color == target) {
            println("พบ '$target' ที่ index $i")
        }
    }
}
```

---

## while Loop

### while พื้นฐาน

```kotlin
fun main() {
    // while: ตรวจสอบเงื่อนไขก่อน วน
    var i = 1
    while (i <= 5) {
        print("$i ")
        i++
    }
    println()  // 1 2 3 4 5
    
    // ถ้าเงื่อนไขเป็น false ตั้งแต่ต้น จะไม่วนเลย
    var j = 10
    while (j < 5) {
        println("บรรทัดนี้จะไม่ถูก execute")
        j++
    }
    println("j = $j")  // j = 10
    
    // while กับ Collection
    val queue = ArrayDeque(listOf("งานที่ 1", "งานที่ 2", "งานที่ 3"))
    
    while (queue.isNotEmpty()) {
        val task = queue.removeFirst()
        println("กำลังทำ: $task")
    }
    println("ทำงานเสร็จทั้งหมด!")
}
```

### while วนซ้ำจนกว่าจะได้ input ที่ถูกต้อง

```kotlin
fun main() {
    var validInput = false
    var number = 0
    
    while (!validInput) {
        print("ใส่ตัวเลข 1-10: ")
        val input = readln()
        val parsed = input.toIntOrNull()
        
        if (parsed != null && parsed in 1..10) {
            number = parsed
            validInput = true
        } else {
            println("ข้อมูลไม่ถูกต้อง! กรุณาใส่ตัวเลข 1-10")
        }
    }
    
    println("คุณใส่: $number")
    
    // เขียนสั้นกว่าด้วย while loop
    var input2: Int? = null
    while (input2 == null || input2 !in 1..10) {
        print("ใส่ตัวเลข 1-10: ")
        input2 = readln().toIntOrNull()
        if (input2 == null || input2 !in 1..10) {
            println("กรุณาใส่ตัวเลข 1-10 เท่านั้น")
        }
    }
    println("คุณใส่: $input2")
}
```

### while กับอัลกอริทึม

```kotlin
fun main() {
    // หา GCD ด้วย Euclidean Algorithm
    fun gcd(a: Int, b: Int): Int {
        var x = a
        var y = b
        while (y != 0) {
            val temp = y
            y = x % y
            x = temp
        }
        return x
    }
    
    println("GCD(48, 18) = ${gcd(48, 18)}")   // 6
    println("GCD(100, 75) = ${gcd(100, 75)}") // 25
    
    // Newton's Method หา Square Root
    fun sqrt(n: Double): Double {
        var guess = n / 2
        while (Math.abs(guess * guess - n) > 0.0001) {
            guess = (guess + n / guess) / 2
        }
        return guess
    }
    
    println("√25 = ${sqrt(25.0)}")    // ~5.0
    println("√2 = ${"%.6f".format(sqrt(2.0))}")   // ~1.414214
    
    // Collatz Conjecture
    fun collatzSteps(n: Int): Int {
        var num = n
        var steps = 0
        while (num != 1) {
            num = if (num % 2 == 0) num / 2 else 3 * num + 1
            steps++
        }
        return steps
    }
    
    for (n in listOf(6, 27, 100)) {
        println("Collatz($n) ใช้ ${collatzSteps(n)} ขั้นตอน")
    }
}
```

---

## do-while Loop

`do-while` ทำงานเหมือน `while` แต่รัน body ก่อนแล้วค่อยตรวจสอบเงื่อนไข (รันอย่างน้อย 1 ครั้ง):

```kotlin
fun main() {
    // do-while: รัน body ก่อน แล้วตรวจเงื่อนไข
    var i = 1
    do {
        print("$i ")
        i++
    } while (i <= 5)
    println()  // 1 2 3 4 5
    
    // รันอย่างน้อย 1 ครั้ง แม้เงื่อนไขเป็น false
    var j = 10
    do {
        println("จะรัน 1 ครั้ง แม้ j = $j")
        j++
    } while (j < 5)
    // Output: จะรัน 1 ครั้ง แม้ j = 10
    
    // ใช้บ่อยสำหรับ Menu
    var choice: String
    do {
        println("\n=== เมนู ===")
        println("1. ตัวเลือกที่ 1")
        println("2. ตัวเลือกที่ 2")
        println("0. ออก")
        print("เลือก: ")
        choice = readln()
        
        when (choice) {
            "1" -> println("เลือกตัวเลือกที่ 1")
            "2" -> println("เลือกตัวเลือกที่ 2")
            "0" -> println("ลาก่อน!")
            else -> println("ไม่ถูกต้อง กรุณาเลือกใหม่")
        }
    } while (choice != "0")
}
```

### เปรียบเทียบ while vs do-while

```kotlin
fun main() {
    // while: ถ้า list ว่าง จะไม่เข้า loop
    val emptyList = mutableListOf<Int>()
    var whileCount = 0
    var listIndex = 0
    
    while (listIndex < emptyList.size) {
        whileCount++
        listIndex++
    }
    println("while รัน $whileCount ครั้ง")  // 0 ครั้ง
    
    // do-while: รันอย่างน้อย 1 ครั้ง
    var doWhileCount = 0
    var listIndex2 = 0
    do {
        doWhileCount++
        listIndex2++
    } while (listIndex2 < emptyList.size)
    println("do-while รัน $doWhileCount ครั้ง")  // 1 ครั้ง
    
    // ใช้ while เมื่อ: อาจไม่ต้องการรันเลย (เช่น iterate ข้อมูล)
    // ใช้ do-while เมื่อ: ต้องการรันอย่างน้อยหนึ่งครั้ง (เช่น input validation)
}
```

---

## break และ continue

### break

```kotlin
fun main() {
    // break: หยุด loop ทันที
    for (i in 1..10) {
        if (i == 6) break
        print("$i ")
    }
    println()  // 1 2 3 4 5
    
    // ใช้ break ค้นหาข้อมูล
    val numbers = listOf(3, 7, 12, 5, 19, 8, 4)
    var found = -1
    
    for ((index, num) in numbers.withIndex()) {
        if (num > 15) {
            found = index
            break
        }
    }
    
    if (found != -1) {
        println("พบตัวเลขมากกว่า 15 ที่ index $found: ${numbers[found]}")
    } else {
        println("ไม่พบตัวเลขมากกว่า 15")
    }
    
    // break ใน while
    var count = 0
    while (true) {  // Infinite loop
        count++
        if (count >= 5) break
    }
    println("count = $count")  // 5
}
```

### continue

```kotlin
fun main() {
    // continue: ข้ามรอบนี้ไปรอบถัดไป
    for (i in 1..10) {
        if (i % 2 == 0) continue  // ข้ามเลขคู่
        print("$i ")
    }
    println()  // 1 3 5 7 9
    
    // ข้าม null values
    val data = listOf("apple", null, "banana", null, "orange")
    for (item in data) {
        if (item == null) continue
        println(item.uppercase())
    }
    // APPLE
    // BANANA
    // ORANGE
    
    // กรองด้วย continue
    val scores = listOf(45, 78, 92, 55, 88, 43, 71, 60)
    println("\nคะแนนที่ผ่าน (>= 60):")
    var passCount = 0
    for (score in scores) {
        if (score < 60) continue  // ข้ามคะแนนที่ไม่ผ่าน
        println("  $score ✅")
        passCount++
    }
    println("ผ่าน $passCount คน จาก ${scores.size} คน")
}
```

---

## Labels

Labels ใช้ควบคุม loop ที่ซ้อนกัน:

```kotlin
fun main() {
    // break loop ด้านนอก ด้วย Label
    outer@ for (i in 1..5) {
        for (j in 1..5) {
            if (i + j == 7) break@outer  // break จาก outer loop
            print("($i,$j) ")
        }
        println()
    }
    println("เสร็จสิ้น")
    
    // Output:
    // (1,1) (1,2) (1,3) (1,4) (1,5)
    // (2,1) (2,2) (2,3) (2,4) (2,5)
    // (3,1) (3,2) (3,3)
    // เสร็จสิ้น
    
    println()
    
    // continue loop ด้านนอก ด้วย Label
    loop@ for (i in 1..4) {
        for (j in 1..4) {
            if (j == 3) continue@loop  // continue ไปรอบถัดไปของ outer loop
            print("($i,$j) ")
        }
    }
    println()
    // Output: (1,1) (1,2) (2,1) (2,2) (3,1) (3,2) (4,1) (4,2)
    
    // ตัวอย่างการใช้งานจริง: ค้นหาใน Matrix
    val matrix = arrayOf(
        intArrayOf(1, 2, 3),
        intArrayOf(4, 5, 6),
        intArrayOf(7, 8, 9)
    )
    val target = 5
    
    search@ for (row in matrix.indices) {
        for (col in matrix[row].indices) {
            if (matrix[row][col] == target) {
                println("พบ $target ที่ row=$row, col=$col")
                break@search
            }
        }
    }
}
```

---

## repeat()

`repeat()` เป็น stdlib function สำหรับการวนซ้ำจำนวนครั้งที่กำหนด:

```kotlin
fun main() {
    // repeat n ครั้ง
    repeat(5) {
        print("★ ")
    }
    println()  // ★ ★ ★ ★ ★
    
    // repeat พร้อม index
    repeat(5) { index ->
        println("รอบที่ ${index + 1}")
    }
    
    // เทียบกับ for
    // for (i in 0 until 5) { ... }
    // repeat(5) { i -> ... }
    
    // ตัวอย่างการใช้งาน
    val password = "mySecret"
    var loginSuccess = false
    var attempts = 0
    
    repeat(3) {
        if (!loginSuccess) {
            attempts++
            print("ใส่รหัสผ่าน (ครั้งที่ $attempts/3): ")
            val input = readln()
            if (input == password) {
                println("✅ เข้าสู่ระบบสำเร็จ!")
                loginSuccess = true
            } else {
                println("❌ รหัสผ่านผิด")
            }
        }
    }
    
    if (!loginSuccess) {
        println("บัญชีถูกล็อค!")
    }
}
```

---

## forEach และ Collection Iteration

Kotlin มี Higher-Order Functions สำหรับ iterate collection ที่ Idiomatic กว่า:

```kotlin
fun main() {
    val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    
    // forEach - วนทุกธาตุ
    numbers.forEach { n ->
        print("$n ")
    }
    println()  // 1 2 3 4 5 6 7 8 9 10
    
    // it ใน forEach (ถ้ามี parameter เดียว)
    numbers.forEach { print("$it ") }
    println()
    
    // forEachIndexed - มี index ด้วย
    numbers.forEachIndexed { index, value ->
        print("[$index]=$value ")
    }
    println()
    
    // filter - กรองข้อมูล
    val evens = numbers.filter { it % 2 == 0 }
    println("เลขคู่: $evens")  // [2, 4, 6, 8, 10]
    
    // map - แปลงข้อมูล
    val doubled = numbers.map { it * 2 }
    println("คูณ 2: $doubled")  // [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
    
    // reduce - รวมข้อมูล
    val sum = numbers.reduce { acc, n -> acc + n }
    println("ผลรวม: $sum")  // 55
    
    // fold - เหมือน reduce แต่มีค่าเริ่มต้น
    val sumWithStart = numbers.fold(100) { acc, n -> acc + n }
    println("ผลรวมเริ่มที่ 100: $sumWithStart")  // 155
    
    // any - มีธาตุที่ตรงเงื่อนไขอย่างน้อยหนึ่งตัว?
    println("มีเลข > 8: ${numbers.any { it > 8 }}")  // true
    
    // all - ทุกธาตุตรงเงื่อนไข?
    println("ทุกตัว > 0: ${numbers.all { it > 0 }}")  // true
    
    // none - ไม่มีธาตุใดตรงเงื่อนไข?
    println("ไม่มีตัว < 0: ${numbers.none { it < 0 }}")  // true
    
    // count - นับธาตุที่ตรงเงื่อนไข
    println("จำนวนเลขคู่: ${numbers.count { it % 2 == 0 }}")  // 5
    
    // groupBy - จัดกลุ่ม
    val grouped = numbers.groupBy { if (it % 2 == 0) "คู่" else "คี่" }
    println("จัดกลุ่ม: $grouped")
    // {คี่=[1, 3, 5, 7, 9], คู่=[2, 4, 6, 8, 10]}
    
    // sorted
    val unsorted = listOf(5, 2, 8, 1, 9, 3)
    println("เรียงลำดับ: ${unsorted.sorted()}")             // [1, 2, 3, 5, 8, 9]
    println("เรียงลำดับกลับ: ${unsorted.sortedDescending()}")  // [9, 8, 5, 3, 2, 1]
    
    // partition - แบ่งออกเป็น 2 กลุ่ม
    val (pass, fail) = numbers.partition { it >= 5 }
    println("≥5: $pass")  // [5, 6, 7, 8, 9, 10]
    println("<5: $fail")  // [1, 2, 3, 4]
}
```

### Chaining Operations

```kotlin
fun main() {
    val students = listOf(
        mapOf("name" to "สมชาย", "score" to 85, "subject" to "Math"),
        mapOf("name" to "สมหญิง", "score" to 92, "subject" to "Math"),
        mapOf("name" to "สมศักดิ์", "score" to 65, "subject" to "Science"),
        mapOf("name" to "สมพร", "score" to 78, "subject" to "Math"),
        mapOf("name" to "สมเกียรติ", "score" to 55, "subject" to "Science"),
    )
    
    // หานักเรียนวิชา Math ที่ได้คะแนน >= 80 เรียงตามคะแนน
    val topMathStudents = students
        .filter { it["subject"] == "Math" }
        .filter { (it["score"] as Int) >= 80 }
        .sortedByDescending { it["score"] as Int }
    
    println("=== Top Math Students (Score ≥ 80) ===")
    topMathStudents.forEachIndexed { i, s ->
        println("${i + 1}. ${s["name"]}: ${s["score"]} คะแนน")
    }
    
    // หาคะแนนเฉลี่ยของแต่ละวิชา
    val avgBySubject = students
        .groupBy { it["subject"] as String }
        .mapValues { (_, students) ->
            students.map { it["score"] as Int }.average()
        }
    
    println("\n=== คะแนนเฉลี่ยแต่ละวิชา ===")
    avgBySubject.forEach { (subject, avg) ->
        println("$subject: ${"%.1f".format(avg)}")
    }
}
```

---

## Sequence และ Lazy Evaluation

`Sequence` ใน Kotlin คือ Lazy Collection - ประมวลผลเมื่อต้องการเท่านั้น:

```kotlin
fun main() {
    // Collection vs Sequence
    
    // ❌ Collection (Eager) - ประมวลผลทันทีทุก step
    val result1 = (1..1_000_000)
        .filter { it % 2 == 0 }    // สร้าง List ใหม่ 500,000 ธาตุ
        .map { it * it }             // สร้าง List ใหม่ 500,000 ธาตุ
        .first()                     // เอาแค่ธาตุแรก
    println("Eager result: $result1")  // 4
    
    // ✅ Sequence (Lazy) - ประมวลผลเมื่อต้องการ ไม่สร้าง intermediate list
    val result2 = (1..1_000_000)
        .asSequence()                // แปลงเป็น Sequence
        .filter { it % 2 == 0 }     // ยังไม่ประมวลผล
        .map { it * it }             // ยังไม่ประมวลผล
        .first()                     // ประมวลผลจนได้ธาตุแรก แล้วหยุด
    println("Lazy result: $result2")   // 4
    
    // generateSequence - สร้าง Sequence ไม่จำกัด
    val fibonacci = generateSequence(Pair(0, 1)) { (a, b) -> Pair(b, a + b) }
        .map { it.first }
        .take(15)
        .toList()
    println("Fibonacci 15 ตัว: $fibonacci")
    // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377]
    
    // generateSequence กับ null (หยุดเมื่อ return null)
    val powersOf2 = generateSequence(1) { n ->
        if (n < 1024) n * 2 else null  // หยุดเมื่อ n >= 1024
    }.toList()
    println("Powers of 2: $powersOf2")
    // [1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024]
    
    // sequence builder
    val evenNumbers = sequence {
        var n = 0
        while (true) {
            yield(n)    // ส่งค่าออก
            n += 2
        }
    }
    println("เลขคู่ 10 ตัวแรก: ${evenNumbers.take(10).toList()}")
    // [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
}
```

---

## ตัวอย่างโปรแกรมสรุปรวม

### โปรแกรมตาราง Multiplication

```kotlin
fun main() {
    val size = 9
    
    // Header
    print("  × |")
    for (i in 1..size) {
        print("%4d".format(i))
    }
    println()
    println("-" * 4 + "+" + "-" * (size * 4))
    
    // Rows
    for (i in 1..size) {
        print("%3d |".format(i))
        for (j in 1..size) {
            print("%4d".format(i * j))
        }
        println()
    }
}

// Extension function
operator fun String.times(n: Int): String = repeat(n)
```

**Output:**
```
  × |   1   2   3   4   5   6   7   8   9
----+------------------------------------
  1 |   1   2   3   4   5   6   7   8   9
  2 |   2   4   6   8  10  12  14  16  18
  3 |   3   6   9  12  15  18  21  24  27
  4 |   4   8  12  16  20  24  28  32  36
  5 |   5  10  15  20  25  30  35  40  45
  6 |   6  12  18  24  30  36  42  48  54
  7 |   7  14  21  28  35  42  49  56  63
  8 |   8  16  24  32  40  48  56  64  72
  9 |   9  18  27  36  45  54  63  72  81
```

### โปรแกรม Number Guessing Game

```kotlin
import kotlin.random.Random

fun main() {
    val secretNumber = Random.nextInt(1, 101)
    var attempts = 0
    val maxAttempts = 7
    var won = false
    
    println("╔══════════════════════════════╗")
    println("║    เกมทายตัวเลข 1-100          ║")
    println("║    คุณมี $maxAttempts ครั้ง              ║")
    println("╚══════════════════════════════╝")
    
    while (attempts < maxAttempts && !won) {
        attempts++
        val remaining = maxAttempts - attempts
        
        print("\nครั้งที่ $attempts/$maxAttempts: ")
        val guess = readln().toIntOrNull()
        
        if (guess == null || guess !in 1..100) {
            println("กรุณาใส่ตัวเลข 1-100")
            attempts--  // ไม่นับครั้ง
            continue
        }
        
        when {
            guess == secretNumber -> {
                won = true
                println("🎉 ถูกต้อง! คำตอบคือ $secretNumber")
                println("คุณใช้ไป $attempts ครั้ง")
            }
            guess < secretNumber -> {
                val diff = secretNumber - guess
                val hint = when {
                    diff > 30 -> "สูงกว่ามาก! ↑↑↑"
                    diff > 10 -> "สูงกว่า ↑↑"
                    else      -> "สูงกว่าเล็กน้อย ↑"
                }
                println("น้อยเกินไป! $hint (เหลือ $remaining ครั้ง)")
            }
            else -> {
                val diff = guess - secretNumber
                val hint = when {
                    diff > 30 -> "ต่ำกว่ามาก! ↓↓↓"
                    diff > 10 -> "ต่ำกว่า ↓↓"
                    else      -> "ต่ำกว่าเล็กน้อย ↓"
                }
                println("มากเกินไป! $hint (เหลือ $remaining ครั้ง)")
            }
        }
    }
    
    if (!won) {
        println("\n😢 หมดครั้งแล้ว! คำตอบคือ $secretNumber")
    }
}
```

### Bubble Sort

```kotlin
fun bubbleSort(arr: IntArray): IntArray {
    val n = arr.size
    val result = arr.copyOf()
    
    for (i in 0 until n - 1) {
        var swapped = false
        for (j in 0 until n - i - 1) {
            if (result[j] > result[j + 1]) {
                // swap
                val temp = result[j]
                result[j] = result[j + 1]
                result[j + 1] = temp
                swapped = true
            }
        }
        // ถ้าไม่มีการ swap แสดงว่าเรียงแล้ว
        if (!swapped) break
    }
    return result
}

fun main() {
    val arr = intArrayOf(64, 34, 25, 12, 22, 11, 90)
    println("ก่อนเรียง: ${arr.toList()}")
    
    val sorted = bubbleSort(arr)
    println("หลังเรียง: ${sorted.toList()}")
}
```

---

## แบบฝึกหัด

### Exercise 1: สูตรคูณทุกแม่ 1-12

```kotlin
fun main() {
    for (i in 1..12) {
        println("=== แม่ $i ===")
        for (j in 1..12) {
            print("$i×$j=${i*j}".padEnd(8))
            if (j % 4 == 0) println()
        }
        println()
    }
}
```

### Exercise 2: Pattern วาดรูป

```kotlin
fun main() {
    val n = 5
    
    // สี่เหลี่ยมกลวง
    println("=== สี่เหลี่ยมกลวง ===")
    for (i in 1..n) {
        for (j in 1..n) {
            if (i == 1 || i == n || j == 1 || j == n) {
                print("* ")
            } else {
                print("  ")
            }
        }
        println()
    }
    
    // สามเหลี่ยมฟลอยด์
    println("\n=== Floyd's Triangle ===")
    var num = 1
    for (i in 1..5) {
        for (j in 1..i) {
            print("%3d".format(num++))
        }
        println()
    }
}
```

**Output:**
```
=== สี่เหลี่ยมกลวง ===
* * * * *
*       *
*       *
*       *
* * * * *

=== Floyd's Triangle ===
  1
  2  3
  4  5  6
  7  8  9 10
 11 12 13 14 15
```

### Exercise 3: ท้าทาย - FizzBuzz

```kotlin
fun main() {
    println("=== FizzBuzz 1-100 ===")
    
    for (i in 1..100) {
        val result = when {
            i % 15 == 0 -> "FizzBuzz"
            i % 3 == 0  -> "Fizz"
            i % 5 == 0  -> "Buzz"
            else        -> i.toString()
        }
        print("$result ")
        if (i % 10 == 0) println()
    }
    
    // สถิติ
    val fizzCount = (1..100).count { it % 3 == 0 && it % 5 != 0 }
    val buzzCount = (1..100).count { it % 5 == 0 && it % 3 != 0 }
    val fizzBuzzCount = (1..100).count { it % 15 == 0 }
    
    println("\nFizz: $fizzCount, Buzz: $buzzCount, FizzBuzz: $fizzBuzzCount")
}
```

---

## สรุป Part 06

```
✅ for loop กับ Range: 1..10, 1 until 10, 10 downTo 1, step
✅ for loop กับ Collection: List, Map, Array
✅ for กับ withIndex(), forEachIndexed()
✅ while: ตรวจเงื่อนไขก่อน
✅ do-while: รันก่อน ตรวจทีหลัง (อย่างน้อย 1 ครั้ง)
✅ break: หยุด loop
✅ continue: ข้ามรอบนี้
✅ Labels: ควบคุม nested loops
✅ repeat(n): วนซ้ำ n ครั้ง
✅ forEach, filter, map, reduce, fold: Higher-Order Functions
✅ Sequence: Lazy evaluation สำหรับ Big Data
```

---

*Part 06/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
