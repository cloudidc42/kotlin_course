# Part 09: String และการจัดการข้อความ

## สารบัญ
1. [String พื้นฐาน](#string-พื้นฐาน)
2. [String Templates](#string-templates)
3. [Raw Strings](#raw-strings)
4. [String Operations](#string-operations)
5. [String Functions ที่ใช้บ่อย](#string-functions-ที่ใช้บ่อย)
6. [Regular Expressions](#regular-expressions)
7. [StringBuilder และ buildString](#stringbuilder-และ-buildstring)
8. [String Comparison](#string-comparison)
9. [Char Operations](#char-operations)
10. [แบบฝึกหัด](#แบบฝึกหัด)

---

## String พื้นฐาน

```kotlin
fun main() {
    // String literals
    val s1 = "Hello, Kotlin!"
    val s2 = "สวัสดี ชาว Kotlin"
    val empty = ""
    
    // String เป็น immutable - ไม่สามารถเปลี่ยน char ได้
    val str = "Hello"
    // str[0] = 'h'  // ❌ Error: Strings are immutable
    
    // String เป็น sequence of chars
    for (c in "Hello") {
        print("$c ")  // H e l l o
    }
    println()
    
    // ความยาว
    println(s1.length)  // 14
    println(empty.length)  // 0
    
    // เข้าถึง char
    println(s1[0])         // H
    println(s1.first())    // H
    println(s1.last())     // !
    println(s1[7])         // K
    
    // String เป็น null-safe
    val nullable: String? = null
    println(nullable?.length ?: 0)  // 0
    println(nullable.isNullOrEmpty())  // true
    println(nullable.isNullOrBlank())  // true
}
```

---

## String Templates

```kotlin
fun main() {
    val name = "สมชาย"
    val age = 25
    val salary = 45_000.0
    
    // Simple template: $variable
    println("ชื่อ: $name")
    println("อายุ: $age ปี")
    
    // Expression template: ${expression}
    println("ปีเกิด: ${2024 - age}")
    println("ชื่อมี ${name.length} ตัวอักษร")
    println("เงินเดือน: ${"%,.0f".format(salary)} บาท")
    println("เฉลี่ยวัน: ${(salary / 30).toInt()} บาท/วัน")
    
    // Nested
    data class User(val firstName: String, val lastName: String)
    val user = User("สมชาย", "ใจดี")
    println("ชื่อเต็ม: ${user.firstName} ${user.lastName}")
    println("อักษรย่อ: ${user.firstName.first()}.${user.lastName.first()}.")
    
    // Template ใน Raw String
    val report = """
        === รายงาน ===
        ชื่อ: $name
        อายุ: $age ปี
        เงินเดือน: ${"%,.2f".format(salary)} บาท
        ระดับ: ${when {
            salary >= 60_000 -> "สูง"
            salary >= 40_000 -> "กลาง"
            else -> "ต่ำ"
        }}
    """.trimIndent()
    
    println(report)
    
    // Escape $ ใน String Template
    println("ราคา: \$${salary}")  // ราคา: $45000.0
    println("ตัวแปร: \$name = $name")  // ตัวแปร: $name = สมชาย
}
```

---

## Raw Strings

```kotlin
fun main() {
    // Raw string: triple quote """..."""
    // ไม่ต้อง escape characters
    
    val path = """C:\Users\สมชาย\Documents\project"""
    println(path)
    // C:\Users\สมชาย\Documents\project
    
    val regex = """\d+\.\d{2}"""  // ไม่ต้อง escape \
    println(regex)  // \d+\.\d{2}
    
    // Multiline
    val poem = """
        กวีสี่บท
        แรกเลิศล้วนแล้ว
        สองโสภาแท้
        สามงามเยี่ยม
        สี่เลิศล้วน
    """.trimIndent()
    println(poem)
    
    // trimIndent(): ลบ indent นำหน้าทุกบรรทัด
    // trimMargin(): ลบ margin ที่กำหนด
    
    val code = """
        |fun hello() {
        |    println("Hello!")
        |}
    """.trimMargin()  // ลบทุกอย่างก่อน | และ | เอง
    println(code)
    
    // Raw string กับ Template
    val x = 42
    val y = 100
    val calculation = """
        ผลลัพธ์:
        $x + $y = ${x + y}
        $x * $y = ${x * y}
        $x / $y = ${"%.4f".format(x.toDouble() / y)}
    """.trimIndent()
    println(calculation)
    
    // JSON ใน Raw String (ไม่ต้อง escape quotes)
    val json = """
        {
            "name": "สมชาย",
            "age": 25,
            "hobbies": ["Kotlin", "Coffee", "Reading"]
        }
    """.trimIndent()
    println(json)
    
    // HTML ใน Raw String
    val html = """
        <!DOCTYPE html>
        <html>
        <head><title>Hello Kotlin</title></head>
        <body>
            <h1>สวัสดี Kotlin!</h1>
            <p>ยินดีต้อนรับ</p>
        </body>
        </html>
    """.trimIndent()
    println(html)
}
```

---

## String Operations

### Substring และ Slicing

```kotlin
fun main() {
    val str = "Hello, Kotlin World!"
    
    // Substring
    println(str.substring(7))         // Kotlin World!
    println(str.substring(7, 13))     // Kotlin
    println(str.substring(0, 5))      // Hello
    
    // take/drop
    println(str.take(5))              // Hello (เอา 5 ตัวแรก)
    println(str.takeLast(6))          // World! (เอา 6 ตัวสุดท้าย)
    println(str.drop(7))              // Kotlin World! (ทิ้ง 7 ตัวแรก)
    println(str.dropLast(7))          // Hello, Kotlin (ทิ้ง 7 ตัวสุดท้าย)
    
    // takeWhile / dropWhile
    val numbers = "12345abc678"
    println(numbers.takeWhile { it.isDigit() })   // 12345
    println(numbers.dropWhile { it.isDigit() })   // abc678
    
    // slice
    println(str.slice(0..4))          // Hello
    println(str.slice(7..12))         // Kotlin
    println(str.slice(listOf(0, 2, 4, 6)))  // Hlo,
    
    // subSequence (ไม่สร้าง String ใหม่ - เหมาะกับ performance)
    val seq: CharSequence = str.subSequence(7, 13)
    println(seq)  // Kotlin
    
    // chunked (แบ่งเป็น chunks)
    val lorem = "ABCDEFGHIJ"
    println(lorem.chunked(3))  // [ABC, DEF, GHI, J]
    println(lorem.chunked(3) { it.lowercase() })  // [abc, def, ghi, j]
    
    // windowed (sliding window)
    println("Hello".windowed(3))  // [Hel, ell, llo]
    println("Hello".windowed(3, step = 1))  // [Hel, ell, llo]
    println("Hello".windowed(3, step = 2))  // [Hel, lo]
}
```

### Search Operations

```kotlin
fun main() {
    val text = "The quick brown fox jumps over the lazy dog"
    
    // contains
    println(text.contains("fox"))          // true
    println(text.contains("cat"))          // false
    println(text.contains("FOX"))          // false
    println(text.contains("FOX", ignoreCase = true))  // true
    
    // startsWith / endsWith
    println(text.startsWith("The"))        // true
    println(text.startsWith("the"))        // false
    println(text.startsWith("the", ignoreCase = true))  // true
    println(text.endsWith("dog"))          // true
    
    // indexOf / lastIndexOf
    println(text.indexOf("o"))             // 12
    println(text.lastIndexOf("o"))         // 41
    println(text.indexOf("the"))           // 31 (lowercase the)
    println(text.indexOf("the", ignoreCase = true))  // 0 (The at start)
    println(text.indexOf("xyz"))           // -1 (ไม่พบ)
    
    // count
    println(text.count { it == 'o' })     // 4 (จำนวน 'o' ทั้งหมด)
    
    // find / findLast
    val firstDigit = "abc123def456".find { it.isDigit() }
    println(firstDigit)   // 1
    
    val lastDigit = "abc123def456".findLast { it.isDigit() }
    println(lastDigit)    // 6
    
    // indexOfFirst / indexOfLast
    val str2 = "Hello World"
    println(str2.indexOfFirst { it.isUpperCase() })  // 0 (H)
    println(str2.indexOfLast { it.isUpperCase() })   // 6 (W)
}
```

### Transform Operations

```kotlin
fun main() {
    val str = "  Hello, Kotlin World!  "
    
    // Case conversion
    println(str.uppercase())          // HELLO, KOTLIN WORLD!
    println(str.lowercase())          // hello, kotlin world!
    println(str.trim().capitalize())  // Hello, Kotlin World! (deprecated)
    println(str.trim().replaceFirstChar { it.uppercase() })  // แนะนำแทน capitalize
    
    // Trim
    println(str.trim())               // Hello, Kotlin World!
    println(str.trimStart())          // Hello, Kotlin World!  (เฉพาะหน้า)
    println(str.trimEnd())            //   Hello, Kotlin World! (เฉพาะหลัง)
    println(str.trim { it == ' ' || it == '!' })  // custom trim
    
    // Replace
    println(str.trim().replace("Kotlin", "Kotlin 🎉"))
    println(str.trim().replace("l", "L"))      // replaceAll
    println(str.trim().replaceFirst("l", "L")) // replaceFirst
    
    // Replace ด้วย Regex
    val mixed = "Hello123World456"
    println(mixed.replace("[0-9]".toRegex(), "#"))  // Hello###World###
    println(mixed.replace("[A-Z]".toRegex(), "_"))  // _ello123_orld456
    
    // Replace ด้วย transform function
    val result = mixed.replace("[A-Z]+".toRegex()) { match ->
        "[${match.value}]"
    }
    println(result)  // [H]ello123[W]orld456
    
    // Pad
    println("42".padStart(6))           // "    42"
    println("42".padStart(6, '0'))      // "000042"
    println("42".padEnd(6))             // "42    "
    println("42".padEnd(6, '-'))        // "42----"
    println("Hello".padStart(10, '='))  // =====Hello
    
    // Reverse
    println("Hello".reversed())  // olleH
    
    // Repeat
    println("Ha".repeat(3))       // HaHaHa
    println("-".repeat(20))       // --------------------
    println("🎉".repeat(5))       // 🎉🎉🎉🎉🎉
}
```

---

## String Functions ที่ใช้บ่อย

### Split และ Join

```kotlin
fun main() {
    // split
    val csv = "แดง,เขียว,น้ำเงิน,เหลือง"
    val colors = csv.split(",")
    println(colors)        // [แดง, เขียว, น้ำเงิน, เหลือง]
    println(colors.size)   // 4
    
    // split ด้วย Regex
    val sentence = "Hello   World   Kotlin"
    val words = sentence.split("\\s+".toRegex())
    println(words)  // [Hello, World, Kotlin]
    
    // split จำกัดจำนวน
    val limited = "a:b:c:d:e".split(":", limit = 3)
    println(limited)  // [a, b, c:d:e]
    
    // lines() - แยกตาม newline
    val multiline = "Line 1\nLine 2\nLine 3"
    val lines = multiline.lines()
    println(lines)   // [Line 1, Line 2, Line 3]
    
    // join
    val items = listOf("แอปเปิ้ล", "กล้วย", "ส้ม")
    println(items.joinToString())               // แอปเปิ้ล, กล้วย, ส้ม
    println(items.joinToString(" | "))          // แอปเปิ้ล | กล้วย | ส้ม
    println(items.joinToString(", ", "[", "]")) // [แอปเปิ้ล, กล้วย, ส้ม]
    println(items.joinToString { it.uppercase() })  // แอปเปิ้ล, กล้วย, ส้ม (uppercase)
    
    // joinToString กับ limit
    val longList = (1..100).toList()
    println(longList.joinToString(limit = 5, truncated = "... (${longList.size - 5} more)"))
    // 1, 2, 3, 4, 5, ... (95 more)
    
    // String concatenation
    val parts = listOf("Hello", " ", "World", "!")
    val joined = parts.joinToString("")  // Hello World!
    println(joined)
}
```

### Conversion Functions

```kotlin
fun main() {
    // String → Number
    println("123".toInt())           // 123
    println("3.14".toDouble())       // 3.14
    println("true".toBoolean())      // true
    println("TRUE".toBoolean())      // true
    println("abc".toBoolean())       // false (ไม่ใช่ "true")
    
    // Safe conversion
    println("123".toIntOrNull())     // 123
    println("abc".toIntOrNull())     // null
    println("3.14".toDoubleOrNull()) // 3.14
    println("xyz".toDoubleOrNull())  // null
    
    // Number → String
    println(42.toString())           // "42"
    println(3.14.toString())         // "3.14"
    println(true.toString())         // "true"
    
    // ฐานต่างๆ
    println(255.toString(16))        // ff (hex)
    println(255.toString(2))         // 11111111 (binary)
    println(255.toString(8))         // 377 (octal)
    
    // String → Char Array
    val chars: CharArray = "Hello".toCharArray()
    println(chars.toList())  // [H, e, l, l, o]
    
    // Char Array → String
    val str = String(chars)
    println(str)  // Hello
    
    // toList() - String เป็น List<Char>
    val charList: List<Char> = "Hello".toList()
    println(charList)  // [H, e, l, l, o]
    
    // filter + joinToString
    val onlyLetters = "H3ll0 W0rld!".filter { it.isLetter() }
    println(onlyLetters)  // HllWrld
    
    val onlyDigits = "abc123def456".filter { it.isDigit() }
    println(onlyDigits)  // 123456
}
```

### Check Functions

```kotlin
fun main() {
    // isEmpty / isNotEmpty
    println("".isEmpty())           // true
    println("Hello".isEmpty())      // false
    println("".isNotEmpty())        // false
    println("Hello".isNotEmpty())   // true
    
    // isBlank / isNotBlank
    println("   ".isBlank())        // true (มีแต่ whitespace)
    println("Hello".isBlank())      // false
    println("   ".isNotBlank())     // false
    
    // Null-safe checks
    val nullable: String? = null
    println(nullable.isNullOrEmpty())   // true
    println(nullable.isNullOrBlank())   // true
    
    val blank: String? = "   "
    println(blank.isNullOrEmpty())      // false (มีตัวอักษร space)
    println(blank.isNullOrBlank())      // true (เป็น blank)
    
    // Char checks
    "Hello World 123!".forEach { c ->
        when {
            c.isLetter()    -> print("L")  // Letter
            c.isDigit()     -> print("D")  // Digit
            c.isWhitespace()-> print("W")  // Whitespace
            else            -> print("S")  // Symbol
        }
    }
    println()  // LLLLLWLLLLLWDDDDS
    
    // all, any, none
    println("12345".all { it.isDigit() })    // true
    println("123a5".all { it.isDigit() })    // false
    println("abc".any { it == 'b' })         // true
    println("abc".none { it.isDigit() })     // true
}
```

---

## Regular Expressions

```kotlin
import kotlin.text.Regex

fun main() {
    // สร้าง Regex
    val pattern1 = Regex("\\d+")          // ตัวเลขหนึ่งตัวขึ้นไป
    val pattern2 = "\\d+".toRegex()        // เหมือนกัน
    val pattern3 = """\d+""".toRegex()     // Raw string - ไม่ต้อง escape
    
    // matches (ทั้ง string ต้องตรง)
    println(Regex("\\d+").matches("12345"))   // true
    println(Regex("\\d+").matches("12345a"))  // false
    println(Regex("[A-Z]+").matches("HELLO")) // true
    
    // containsMatchIn (บางส่วนตรง)
    val text = "ราคา 250 บาท"
    println("""\d+""".toRegex().containsMatchIn(text))  // true
    
    // find (ค้นหา match แรก)
    val match = """\d+""".toRegex().find(text)
    println(match?.value)    // 250
    println(match?.range)    // 5..7
    
    // findAll (ค้นหาทุก match)
    val mixedText = "I have 3 cats and 2 dogs and 10 birds"
    val numbers = """\d+""".toRegex().findAll(mixedText)
    numbers.forEach { println(it.value) }
    // 3, 2, 10
    
    // replace
    val dirty = "Hello   World   Kotlin"
    println("""\s+""".toRegex().replace(dirty, " "))  // Hello World Kotlin
    
    // replace ด้วย function
    val result = """\d+""".toRegex().replace("Price: 100, Tax: 7, Total: 107") { match ->
        "[${match.value}]"
    }
    println(result)  // Price: [100], Tax: [7], Total: [107]
    
    // split
    val csv = "a, b,  c,d  ,e"
    val items = """\s*,\s*""".toRegex().split(csv)
    println(items)  // [a, b, c, d, e]
    
    // Groups
    val datePattern = """(\d{2})/(\d{2})/(\d{4})""".toRegex()
    val dateMatch = datePattern.find("วันที่ 15/01/2024 คือวันจันทร์")
    
    dateMatch?.let { m ->
        println("Full match: ${m.value}")   // 15/01/2024
        println("Day: ${m.groupValues[1]}") // 15
        println("Month: ${m.groupValues[2]}") // 01
        println("Year: ${m.groupValues[3]}") // 2024
    }
    
    // Named Groups
    val namePattern = """(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})""".toRegex()
    val isoMatch = namePattern.find("2024-01-15")
    
    isoMatch?.let { m ->
        println("Year: ${m.groups["year"]?.value}")   // 2024
        println("Month: ${m.groups["month"]?.value}") // 01
        println("Day: ${m.groups["day"]?.value}")     // 15
    }
}
```

### Email และ Phone Validation

```kotlin
fun main() {
    // Email Validation
    val emailRegex = """^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$""".toRegex()
    
    val emails = listOf(
        "valid@example.com",
        "user.name+tag@example.co.th",
        "invalid-email",
        "@missing-local.com",
        "missing@.com"
    )
    
    emails.forEach { email ->
        val valid = emailRegex.matches(email)
        println("$email → ${if (valid) "✅" else "❌"}")
    }
    
    // Phone Validation (Thai Format)
    val phoneRegex = """^0[6-9]\d-\d{3}-\d{4}$""".toRegex()
    val phones = listOf("081-234-5678", "062-999-1234", "123-456-7890", "090-123-456")
    phones.forEach { phone ->
        println("$phone → ${if (phoneRegex.matches(phone)) "✅" else "❌"}")
    }
    
    // Thai ID Card (13 digits)
    val idRegex = """\d{13}""".toRegex()
    
    fun validateThaiId(id: String): Boolean {
        if (!idRegex.matches(id)) return false
        
        // checksum algorithm
        var sum = 0
        for (i in 0..11) {
            sum += id[i].digitToInt() * (13 - i)
        }
        val checkDigit = (11 - sum % 11) % 10
        return checkDigit == id[12].digitToInt()
    }
    
    println("\nThai ID validation:")
    println("1234567890123 → ${validateThaiId("1234567890123")}")
}
```

---

## StringBuilder และ buildString

```kotlin
fun main() {
    // StringBuilder - efficient string building
    val sb = StringBuilder()
    
    sb.append("Hello")
    sb.append(", ")
    sb.append("Kotlin")
    sb.append("!")
    
    println(sb.toString())  // Hello, Kotlin!
    println(sb.length)      // 13
    
    // Operations
    sb.insert(5, " World")   // แทรกที่ index 5
    println(sb)              // Hello World, Kotlin!
    
    sb.delete(5, 11)         // ลบ index 5-10
    println(sb)              // Hello, Kotlin!
    
    sb.replace(7, 13, "World")  // แทนที่ index 7-12
    println(sb)               // Hello, World!
    
    sb.reverse()              // กลับลำดับ
    println(sb)               // !dlroW ,olleH
    
    sb.reverse()              // กลับมาเหมือนเดิม
    
    // buildString DSL (Kotlin idiomatic)
    val html = buildString {
        appendLine("<!DOCTYPE html>")
        appendLine("<html>")
        append("  <head>")
        appendLine("<title>Kotlin</title></head>")
        appendLine("  <body>")
        for (i in 1..3) {
            appendLine("    <p>Paragraph $i</p>")
        }
        appendLine("  </body>")
        append("</html>")
    }
    println(html)
    
    // StringBuilder กับ performance
    val n = 10_000
    
    // ❌ ช้า: String concatenation สร้าง String ใหม่ทุกครั้ง O(n²)
    var slow = ""
    repeat(100) { slow += "x" }  // ใช้งานได้สำหรับจำนวนน้อยๆ
    
    // ✅ เร็ว: StringBuilder O(n)
    val fast = buildString {
        repeat(n) { append("x") }
    }
    println("Length: ${fast.length}")  // 10000
    
    // สร้าง Table
    fun createTable(headers: List<String>, rows: List<List<String>>): String {
        val colWidths = headers.mapIndexed { i, h ->
            maxOf(h.length, rows.maxOfOrNull { row -> row.getOrElse(i) { "" }.length } ?: 0)
        }
        
        return buildString {
            // Header
            append(headers.mapIndexed { i, h -> h.padEnd(colWidths[i]) }.joinToString(" | "))
            appendLine()
            append(colWidths.joinToString("-+-") { "-".repeat(it) })
            appendLine()
            // Rows
            rows.forEach { row ->
                appendLine(row.mapIndexed { i, cell ->
                    cell.padEnd(colWidths[i])
                }.joinToString(" | "))
            }
        }
    }
    
    val table = createTable(
        listOf("ชื่อ", "อายุ", "เงินเดือน"),
        listOf(
            listOf("สมชาย", "25", "45,000"),
            listOf("สมหญิง", "30", "60,000"),
            listOf("สมศักดิ์", "28", "52,000")
        )
    )
    println(table)
}
```

---

## String Comparison

```kotlin
fun main() {
    val s1 = "Hello"
    val s2 = "hello"
    val s3 = "Hello"
    
    // == (structural equality) - เปรียบเทียบเนื้อหา
    println(s1 == s2)   // false (case-sensitive)
    println(s1 == s3)   // true
    
    // === (referential equality) - เป็น object เดียวกัน?
    println(s1 === s3)  // true/false ขึ้นอยู่กับ String interning
    
    // equalsIgnoreCase
    println(s1.equals(s2, ignoreCase = true))   // true
    
    // compareTo (alphabetical order)
    println("apple".compareTo("banana"))   // ค่าลบ (apple < banana)
    println("zebra".compareTo("apple"))    // ค่าบวก (zebra > apple)
    println("hello".compareTo("hello"))    // 0 (เท่ากัน)
    
    // compareToIgnoreCase
    println("ABC".compareTo("abc", ignoreCase = true))  // 0
    
    // Sorting
    val words = listOf("Banana", "apple", "Cherry", "date")
    
    // Default: case-sensitive
    println(words.sorted())
    // [Banana, Cherry, apple, date]
    
    // Case-insensitive
    println(words.sortedWith(compareBy { it.lowercase() }))
    // [apple, Banana, Cherry, date]
    
    // Natural order comparison
    val versions = listOf("v1.10", "v1.2", "v1.1", "v2.0")
    println(versions.sorted())
    // [v1.1, v1.10, v1.2, v2.0] (lexicographic - v1.10 ก่อน v1.2!)
    
    // Custom sort
    println(versions.sortedWith(compareBy { 
        it.removePrefix("v").split(".").map { n -> n.toInt() }
    }))
    // [v1.1, v1.2, v1.10, v2.0] (natural version sort)
}
```

---

## Char Operations

```kotlin
fun main() {
    val c = 'A'
    
    // Properties
    println(c.code)             // 65 (Unicode code point)
    println('a'.code)           // 97
    println('0'.code)           // 48
    
    // Conversion
    println(c.lowercaseChar())  // a
    println('a'.uppercaseChar())// A
    println(65.toChar())        // A
    println('A'.digitToInt())   // ❌ exception (ไม่ใช่ digit)
    println('5'.digitToInt())   // 5
    
    // Checks
    println(c.isLetter())       // true
    println(c.isDigit())        // false
    println(c.isLetterOrDigit())// true
    println(c.isUpperCase())    // true
    println(c.isLowerCase())    // false
    println(' '.isWhitespace()) // true
    println('\t'.isWhitespace())// true
    println(c.isAlphabetic())   // true
    
    // Char arithmetic
    println('A' + 1)            // 66 (Int)
    println(('A' + 1).toChar()) // B
    println('Z' - 'A')          // 25 (Int - number of letters - 1)
    
    // สร้าง alphabet
    val alphabet = ('a'..'z').joinToString("")
    println(alphabet)  // abcdefghijklmnopqrstuvwxyz
    
    // ROT13 Cipher
    fun rot13(c: Char): Char = when {
        c in 'A'..'Z' -> 'A' + (c - 'A' + 13) % 26
        c in 'a'..'z' -> 'a' + (c - 'a' + 13) % 26
        else -> c
    }
    
    val encoded = "Hello, World!".map { rot13(it) }.joinToString("")
    println(encoded)   // Uryyb, Jbeyq!
    val decoded = encoded.map { rot13(it) }.joinToString("")
    println(decoded)   // Hello, World!
    
    // Count char occurrences
    val text = "Hello World"
    val vowels = text.count { it in "aeiouAEIOU" }
    println("Vowels: $vowels")  // 3
    
    // Frequency map
    val freq = text.lowercase()
        .filter { it.isLetter() }
        .groupBy { it }
        .mapValues { it.value.size }
        .entries
        .sortedByDescending { it.value }
    
    println("Character frequency:")
    freq.forEach { (c, n) -> println("  '$c': $n") }
}
```

---

## ตัวอย่างโปรแกรมสรุปรวม

### Text Processor

```kotlin
class TextProcessor(private val text: String) {
    
    fun wordCount(): Int = 
        text.split("\\s+".toRegex()).count { it.isNotEmpty() }
    
    fun characterCount(includeSpaces: Boolean = true): Int =
        if (includeSpaces) text.length
        else text.count { !it.isWhitespace() }
    
    fun sentenceCount(): Int =
        text.split("[.!?]".toRegex()).count { it.isNotBlank() }
    
    fun averageWordLength(): Double {
        val words = text.split("\\s+".toRegex())
            .filter { it.isNotEmpty() }
            .map { it.filter { c -> c.isLetter() } }
            .filter { it.isNotEmpty() }
        return if (words.isEmpty()) 0.0 else words.map { it.length }.average()
    }
    
    fun mostCommonWords(n: Int = 10): List<Pair<String, Int>> =
        text.lowercase()
            .replace("[^a-zA-Z0-9\\s]".toRegex(), "")
            .split("\\s+".toRegex())
            .filter { it.isNotEmpty() }
            .groupBy { it }
            .mapValues { it.value.size }
            .entries
            .sortedByDescending { it.value }
            .take(n)
            .map { Pair(it.key, it.value) }
    
    fun highlight(keyword: String): String =
        text.replace(keyword, "**$keyword**", ignoreCase = true)
    
    fun summary(): String = buildString {
        appendLine("=== Text Analysis ===")
        appendLine("Characters (with spaces): ${characterCount()}")
        appendLine("Characters (no spaces): ${characterCount(false)}")
        appendLine("Words: ${wordCount()}")
        appendLine("Sentences: ${sentenceCount()}")
        appendLine("Average word length: ${"%.1f".format(averageWordLength())}")
        appendLine("\nTop 5 words:")
        mostCommonWords(5).forEach { (word, count) ->
            appendLine("  '$word': $count times")
        }
    }
}

fun main() {
    val text = """
        The quick brown fox jumps over the lazy dog.
        The dog barked at the fox. The fox ran away quickly.
        Quick brown foxes are rare in the wild.
    """.trimIndent()
    
    val processor = TextProcessor(text)
    println(processor.summary())
    println(processor.highlight("fox"))
}
```

---

## แบบฝึกหัด

### Exercise 1: Palindrome Checker

```kotlin
fun isPalindrome(str: String): Boolean {
    val clean = str.lowercase().filter { it.isLetterOrDigit() }
    return clean == clean.reversed()
}

fun main() {
    val tests = listOf(
        "racecar",
        "A man a plan a canal Panama",
        "Hello World",
        "Was it a car or a cat I saw",
        "Kotlin"
    )
    
    tests.forEach { str ->
        println("'$str' → ${if (isPalindrome(str)) "✅ Palindrome" else "❌ Not palindrome"}")
    }
}
```

### Exercise 2: String Encryption

```kotlin
fun encrypt(text: String, key: Int): String {
    return text.map { c ->
        when {
            c in 'A'..'Z' -> (((c - 'A' + key) % 26) + 'A'.code).toChar()
            c in 'a'..'z' -> (((c - 'a' + key) % 26) + 'a'.code).toChar()
            else -> c
        }
    }.joinToString("")
}

fun decrypt(text: String, key: Int): String = encrypt(text, 26 - key)

fun main() {
    val original = "Hello Kotlin World!"
    val key = 7
    
    val encrypted = encrypt(original, key)
    val decrypted = decrypt(encrypted, key)
    
    println("Original:  $original")
    println("Encrypted: $encrypted")
    println("Decrypted: $decrypted")
    println("Match: ${original == decrypted}")
}
```

### Exercise 3: ท้าทาย - CSV Parser

```kotlin
data class Student(val name: String, val grade: String, val score: Double)

fun parseCSV(csv: String): List<Student> {
    return csv.lines()
        .drop(1)  // ข้าม header
        .filter { it.isNotBlank() }
        .mapNotNull { line ->
            val parts = line.split(",").map { it.trim() }
            if (parts.size >= 3) {
                val score = parts[2].toDoubleOrNull() ?: return@mapNotNull null
                Student(parts[0], parts[1], score)
            } else null
        }
}

fun main() {
    val csv = """
        Name, Grade, Score
        สมชาย, 10, 85.5
        สมหญิง, 11, 92.0
        สมศักดิ์, 10, 78.3
        สมพร, 12, 88.7
        invalid row
        สมเกียรติ, 11, 95.1
    """.trimIndent()
    
    val students = parseCSV(csv)
    
    println("=== ผลการเรียน ===")
    students.forEach { s ->
        println("${s.name} (ม.${s.grade}): ${s.score}")
    }
    
    println("\nเฉลี่ย: ${"%.2f".format(students.map { it.score }.average())}")
    println("สูงสุด: ${students.maxByOrNull { it.score }?.name}")
    println("ต่ำสุด: ${students.minByOrNull { it.score }?.name}")
    
    // จัดกลุ่มตามเกรด
    val byGrade = students.groupBy { it.grade }
    println("\n=== แยกตามชั้น ===")
    byGrade.forEach { (grade, list) ->
        println("ม.$grade: ${list.map { it.name }.joinToString(", ")}")
    }
}
```

---

## สรุป Part 09

```
✅ String literals: "", raw string """"""
✅ String templates: $variable, ${expression}
✅ trimIndent(), trimMargin() สำหรับ multiline
✅ substring, take, drop, slice สำหรับ slicing
✅ contains, startsWith, endsWith, indexOf สำหรับ search
✅ uppercase, lowercase, replace, trim สำหรับ transform
✅ split, joinToString สำหรับ split/join
✅ Regex: matches, find, findAll, replace
✅ StringBuilder, buildString สำหรับ performance
✅ Char operations: isLetter, isDigit, code, toChar
✅ filterNotNull, mapNotNull, filter สำหรับ processing
```

---

*Part 09/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
