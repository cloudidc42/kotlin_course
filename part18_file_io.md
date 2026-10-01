# Part 18: File I/O

## สารบัญ
1. [File Operations พื้นฐาน](#file-operations-พื้นฐาน)
2. [Reading Files](#reading-files)
3. [Writing Files](#writing-files)
4. [File Paths](#file-paths)
5. [JSON Processing](#json-processing)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## File Operations พื้นฐาน

```kotlin
import java.io.File
import java.nio.file.Files
import java.nio.file.Paths

fun main() {
    val file = File("test.txt")
    val file2 = File("/tmp/test.txt")   // absolute path
    val file3 = File("data", "test.txt") // parent/child
    
    // Check
    println(file.exists())        // false
    println(file.isFile())        // false (doesn't exist)
    println(file.isDirectory())   // false
    println(file.canRead())       // false
    println(file.canWrite())      // false
    
    // Create
    file.createNewFile()
    println(file.exists())  // true
    
    // Properties
    println(file.name)            // test.txt
    println(file.extension)       // txt
    println(file.nameWithoutExtension)  // test
    println(file.parent)          // . or null
    println(file.absolutePath)    // /path/to/test.txt
    println(file.length())        // 0 (empty)
    println(file.lastModified())  // timestamp
    
    // Delete
    file.delete()
    
    // Directory operations
    val dir = File("testDir")
    dir.mkdir()                   // create single directory
    
    val nested = File("a/b/c")
    nested.mkdirs()               // create all parent directories
    
    // List contents
    val files = dir.listFiles()
    files?.forEach { println(it.name) }
    
    // Walk directory tree
    nested.parentFile?.parentFile?.walkTopDown()?.forEach { f ->
        println("${if (f.isDirectory) "D" else "F"}: ${f.path}")
    }
    
    // Cleanup
    nested.parentFile?.parentFile?.deleteRecursively()
    dir.delete()
}
```

---

## Reading Files

```kotlin
import java.io.File
import java.io.BufferedReader

fun main() {
    // สร้างไฟล์ทดสอบก่อน
    val testFile = File("sample.txt")
    testFile.writeText("""
        Line 1: Hello Kotlin
        Line 2: สวัสดี
        Line 3: 12345
        Line 4: File I/O
        Line 5: Last line
    """.trimIndent())
    
    // readText: อ่านทั้งไฟล์เป็น String
    val content = testFile.readText()
    println("=== readText ===")
    println(content)
    
    // readLines: อ่านเป็น List<String>
    val lines = testFile.readLines()
    println("\n=== readLines ===")
    lines.forEachIndexed { i, line ->
        println("${i + 1}: $line")
    }
    println("Total: ${lines.size} lines")
    
    // forEachLine: process line by line (memory efficient)
    println("\n=== forEachLine ===")
    var lineCount = 0
    testFile.forEachLine { line ->
        lineCount++
        if (line.contains("Hello")) println("Found: $line")
    }
    println("Processed $lineCount lines")
    
    // bufferedReader: Java-style
    println("\n=== bufferedReader ===")
    testFile.bufferedReader().use { reader ->
        var line = reader.readLine()
        while (line != null) {
            println(line)
            line = reader.readLine()
        }
    }  // auto-close
    
    // readBytes: อ่านเป็น ByteArray
    val bytes = testFile.readBytes()
    println("\nFile size: ${bytes.size} bytes")
    
    // useLines: Sequence (lazy, closes file automatically)
    testFile.useLines { lines ->
        lines
            .filter { it.contains("Line") }
            .map { it.uppercase() }
            .forEach { println(it) }
    }
    
    // inputStream
    testFile.inputStream().bufferedReader().use { reader ->
        println(reader.readText().lines().count())
    }
    
    testFile.delete()
}
```

---

## Writing Files

```kotlin
import java.io.File

fun main() {
    // writeText: เขียนทับทั้งหมด
    val file1 = File("output1.txt")
    file1.writeText("Hello, Kotlin!\n")
    file1.writeText("Overwritten")  // เขียนทับ
    println(file1.readText())  // Overwritten
    
    // appendText: ต่อท้าย
    val file2 = File("output2.txt")
    file2.writeText("Line 1\n")
    file2.appendText("Line 2\n")
    file2.appendText("Line 3\n")
    println(file2.readText())
    
    // printWriter: formatted output
    val file3 = File("output3.txt")
    file3.printWriter().use { pw ->
        pw.println("Name,Age,Score")
        pw.println("สมชาย,25,85.5")
        pw.println("สมหญิง,30,92.0")
        pw.printf("%-10s %-5d %.2f%n", "สมศักดิ์", 28, 78.3)
    }
    println(file3.readText())
    
    // bufferedWriter: efficient for large files
    val file4 = File("output4.txt")
    file4.bufferedWriter().use { bw ->
        repeat(1000) { i ->
            bw.write("Line ${i + 1}: Data entry #$i")
            bw.newLine()
        }
    }
    println("Lines written: ${file4.readLines().size}")
    
    // writeBytes: binary data
    val file5 = File("output5.bin")
    val bytes = byteArrayOf(72, 101, 108, 108, 111)  // "Hello"
    file5.writeBytes(bytes)
    println("Binary content: ${String(file5.readBytes())}")
    
    // outputStream
    val file6 = File("output6.txt")
    file6.outputStream().use { os ->
        os.write("Data from output stream".toByteArray())
    }
    
    // Cleanup
    listOf(file1, file2, file3, file4, file5, file6).forEach { it.delete() }
}
```

---

## File Paths

```kotlin
import java.io.File
import java.nio.file.Path
import java.nio.file.Paths
import java.nio.file.Files

fun main() {
    // File path operations
    val path = File("data/users/profile.json")
    
    println("Name: ${path.name}")              // profile.json
    println("Ext: ${path.extension}")           // json
    println("No ext: ${path.nameWithoutExtension}") // profile
    println("Parent: ${path.parent}")           // data/users
    println("Absolute: ${path.absolutePath}")   // /pwd/data/users/...
    
    // Path resolution
    val base = File("data")
    val child = File(base, "users/data.csv")
    println("Resolved: ${child.path}")  // data/users/data.csv
    
    // NIO Path
    val nioPath: Path = Paths.get("data", "users", "file.txt")
    println("NIO: $nioPath")
    
    val absolute = nioPath.toAbsolutePath()
    println("Absolute: $absolute")
    
    // Resolve
    val parentPath = Paths.get("project")
    val childPath = parentPath.resolve("src/main.kt")
    println("Resolved: $childPath")
    
    // Normalize
    val ugly = Paths.get("a/../b/./c")
    println("Normalized: ${ugly.normalize()}")  // b/c
    
    // Relativize
    val base2 = Paths.get("/home/user")
    val target = Paths.get("/home/user/documents/file.txt")
    println("Relative: ${base2.relativize(target)}")  // documents/file.txt
    
    // File system operations (NIO)
    val testDir = Paths.get("test_dir")
    Files.createDirectories(testDir)
    
    val testFile = testDir.resolve("test.txt")
    Files.writeString(testFile, "Hello NIO!")
    
    println("Content: ${Files.readString(testFile)}")
    println("Size: ${Files.size(testFile)} bytes")
    println("Is readable: ${Files.isReadable(testFile)}")
    
    // Walk
    Files.walk(testDir).forEach { p ->
        println("  $p")
    }
    
    // Copy
    val copyPath = testDir.resolve("copy.txt")
    Files.copy(testFile, copyPath)
    
    // Delete
    Files.delete(copyPath)
    Files.delete(testFile)
    Files.delete(testDir)
}
```

---

## JSON Processing

```kotlin
// ใช้ kotlinx.serialization
// build.gradle.kts:
// plugins { kotlin("plugin.serialization") version "1.9.0" }
// implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.0")

import kotlinx.serialization.*
import kotlinx.serialization.json.*

@Serializable
data class User(
    val id: Int,
    val name: String,
    val email: String,
    val age: Int,
    val roles: List<String> = listOf("user")
)

@Serializable
data class Config(
    val host: String,
    val port: Int,
    val debug: Boolean = false,
    @SerialName("max_connections")  // map to different JSON key
    val maxConnections: Int = 100
)

fun main() {
    val json = Json { 
        prettyPrint = true
        ignoreUnknownKeys = true
        encodeDefaults = false
    }
    
    // Serialize (Object → JSON)
    val user = User(1, "สมชาย", "somchai@email.com", 25, listOf("user", "admin"))
    val jsonString = json.encodeToString(user)
    println("Serialized:\n$jsonString")
    
    // Deserialize (JSON → Object)
    val userJson = """
        {
            "id": 2,
            "name": "สมหญิง",
            "email": "somying@email.com",
            "age": 30,
            "roles": ["user"],
            "unknown_field": "ignored"
        }
    """.trimIndent()
    
    val parsedUser = json.decodeFromString<User>(userJson)
    println("\nParsed: $parsedUser")
    
    // List
    val users = listOf(user, parsedUser)
    val usersJson = json.encodeToString(users)
    println("\nUsers JSON:\n$usersJson")
    
    val parsedUsers = json.decodeFromString<List<User>>(usersJson)
    println("Parsed ${parsedUsers.size} users")
    
    // Config
    val configJson = """{"host":"localhost","port":8080,"max_connections":50}"""
    val config = json.decodeFromString<Config>(configJson)
    println("\nConfig: $config")
    
    // Save/Load from file
    val file = java.io.File("users.json")
    file.writeText(usersJson)
    
    val loaded = json.decodeFromString<List<User>>(file.readText())
    println("\nLoaded ${loaded.size} users from file")
    file.delete()
    
    // Dynamic JSON with JsonElement
    val dynamic = json.parseToJsonElement("""{"name":"test","value":42,"tags":["a","b"]}""")
    val obj = dynamic.jsonObject
    
    println("name: ${obj["name"]?.jsonPrimitive?.content}")
    println("value: ${obj["value"]?.jsonPrimitive?.int}")
    println("tags: ${obj["tags"]?.jsonArray?.map { it.jsonPrimitive.content }}")
    
    // Build JSON dynamically
    val built = buildJsonObject {
        put("id", 1)
        put("name", "สมชาย")
        putJsonArray("tags") {
            add("kotlin")
            add("developer")
        }
        putJsonObject("address") {
            put("city", "Bangkok")
            put("country", "Thailand")
        }
    }
    println("\nBuilt: ${json.encodeToString(built)}")
}
```

---

## ตัวอย่างโปรแกรม - Contact Book

```kotlin
import java.io.File
import kotlinx.serialization.*
import kotlinx.serialization.json.*

@Serializable
data class Contact(
    val id: Int,
    val name: String,
    val phone: String,
    val email: String,
    val tags: List<String> = emptyList()
)

class ContactBook(private val filePath: String = "contacts.json") {
    private val contacts = mutableListOf<Contact>()
    private var nextId = 1
    private val json = Json { prettyPrint = true }
    
    init { load() }
    
    private fun load() {
        val file = File(filePath)
        if (file.exists()) {
            try {
                contacts.addAll(json.decodeFromString<List<Contact>>(file.readText()))
                nextId = (contacts.maxOfOrNull { it.id } ?: 0) + 1
            } catch (e: Exception) {
                println("Warning: Could not load contacts: ${e.message}")
            }
        }
    }
    
    fun save() {
        File(filePath).writeText(json.encodeToString<List<Contact>>(contacts))
    }
    
    fun add(name: String, phone: String, email: String, vararg tags: String): Contact {
        val contact = Contact(nextId++, name, phone, email, tags.toList())
        contacts.add(contact)
        save()
        return contact
    }
    
    fun findByName(name: String) = contacts.filter { 
        it.name.contains(name, ignoreCase = true) 
    }
    
    fun findByTag(tag: String) = contacts.filter { tag in it.tags }
    
    fun delete(id: Int): Boolean {
        val removed = contacts.removeAll { it.id == id }
        if (removed) save()
        return removed
    }
    
    fun getAll() = contacts.toList()
    
    fun report(): String = buildString {
        appendLine("=== Contact Book ===")
        appendLine("Total: ${contacts.size} contacts")
        
        val byTag = contacts.flatMap { c -> c.tags.map { t -> t to c } }
            .groupBy({ it.first }) { it.second }
        
        if (byTag.isNotEmpty()) {
            appendLine("\nBy tag:")
            byTag.forEach { (tag, list) ->
                appendLine("  $tag: ${list.map { it.name }.joinToString()}")
            }
        }
    }
}

fun main() {
    val book = ContactBook("/tmp/contacts.json")
    
    val c1 = book.add("สมชาย ใจดี", "081-234-5678", "somchai@email.com", "friend", "work")
    val c2 = book.add("สมหญิง วงศ์ดี", "089-876-5432", "somying@email.com", "friend")
    val c3 = book.add("สมศักดิ์ มีทรัพย์", "062-111-2222", "somsak@email.com", "work")
    
    println(book.report())
    
    println("Search 'สม':")
    book.findByName("สม").forEach { 
        println("  ${it.name} - ${it.phone}")
    }
    
    println("\nWork contacts:")
    book.findByTag("work").forEach { 
        println("  ${it.name}")
    }
    
    book.delete(c2.id)
    println("\nAfter delete: ${book.getAll().size} contacts")
    
    File("/tmp/contacts.json").delete()
}
```

---

## แบบฝึกหัด

### Exercise: Log File Analyzer

```kotlin
import java.io.File
import java.time.LocalDateTime
import java.time.format.DateTimeFormatter

data class LogEntry(
    val timestamp: LocalDateTime,
    val level: String,
    val message: String
)

fun parseLogLine(line: String): LogEntry? {
    // Format: 2024-01-15 10:30:45 [INFO] Message here
    val regex = """(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}) \[(\w+)\] (.+)""".toRegex()
    val match = regex.matchEntire(line) ?: return null
    
    val (timestamp, level, message) = match.destructured
    val dt = LocalDateTime.parse(timestamp, DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"))
    return LogEntry(dt, level, message)
}

fun main() {
    // สร้างไฟล์ log ทดสอบ
    val logFile = File("/tmp/app.log")
    logFile.writeText("""
        2024-01-15 10:00:01 [INFO] Application started
        2024-01-15 10:00:05 [DEBUG] Loading config
        2024-01-15 10:05:00 [INFO] User logged in: user123
        2024-01-15 10:06:30 [WARN] Slow query: 2.5s
        2024-01-15 10:07:00 [ERROR] Connection timeout
        2024-01-15 10:08:00 [INFO] Retry successful
        2024-01-15 10:09:00 [ERROR] Database error: connection refused
        2024-01-15 10:10:00 [INFO] Shutdown initiated
    """.trimIndent())
    
    // Parse
    val entries = logFile.readLines()
        .mapNotNull { parseLogLine(it) }
    
    // Analyze
    println("Total entries: ${entries.size}")
    
    val byLevel = entries.groupBy { it.level }
    println("\nBy level:")
    byLevel.forEach { (level, list) ->
        println("  $level: ${list.size}")
    }
    
    println("\nErrors:")
    entries.filter { it.level == "ERROR" }.forEach {
        println("  ${it.timestamp} - ${it.message}")
    }
    
    logFile.delete()
}
```

---

## สรุป Part 18

```
✅ File("path"): สร้าง File object
✅ exists(), isFile(), isDirectory(): check file
✅ readText(): อ่านทั้งไฟล์
✅ readLines(): อ่านเป็น List<String>
✅ forEachLine { }: memory-efficient line processing
✅ writeText(): เขียนทับ
✅ appendText(): ต่อท้าย
✅ bufferedReader/Writer: efficient I/O
✅ use { }: auto-close (Closeable)
✅ mkdir()/mkdirs(): สร้าง directory
✅ deleteRecursively(): ลบ directory
✅ walkTopDown(): traverse directory tree
✅ NIO Paths สำหรับ modern path operations
✅ kotlinx.serialization สำหรับ JSON
✅ Json { prettyPrint = true }
```

---

*Part 18/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
