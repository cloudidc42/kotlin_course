# Part 51: Kotlin Native

## สารบัญ
1. [Kotlin Native คืออะไร](#kotlin-native-คืออะไร)
2. [Setup และ Build](#setup-และ-build)
3. [Interop กับ C](#interop-กับ-c)
4. [Memory Model](#memory-model)
5. [Platform-specific Code](#platform-specific-code)
6. [CLI Application](#cli-application)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Kotlin Native คืออะไร

Kotlin Native คือ technology ที่ compile Kotlin code เป็น native binary — ไม่ต้องใช้ JVM runtime

**ข้อดี:**
- Binary เล็ก, startup เร็ว
- Low memory footprint
- ทำงานบน embedded systems, iOS, macOS, Linux, Windows
- สามารถ call C libraries ได้โดยตรง

**Use cases:**
- CLI tools
- iOS mobile apps (via KMP)
- Shared business logic
- System programming
- WebAssembly (WASM)

---

## Setup และ Build

```kotlin
// build.gradle.kts
plugins {
    kotlin("multiplatform") version "2.0.21"
}

kotlin {
    // Native targets
    linuxX64()     // Linux 64-bit
    macosX64()     // macOS Intel
    macosArm64()   // macOS Apple Silicon
    mingwX64()     // Windows 64-bit
    
    sourceSets {
        val nativeMain by creating {
            dependencies {
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.1")
                implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")
            }
        }
        
        val linuxX64Main by getting { dependsOn(nativeMain) }
        val macosX64Main by getting { dependsOn(nativeMain) }
        val macosArm64Main by getting { dependsOn(nativeMain) }
        val mingwX64Main by getting { dependsOn(nativeMain) }
    }
}
```

```kotlin
// src/nativeMain/kotlin/Main.kt
fun main(args: Array<String>) {
    println("Hello from Kotlin Native!")
    println("Platform: ${getPlatformName()}")
    println("Args: ${args.joinToString()}")
}
```

```bash
# Build
./gradlew linkDebugExecutableLinuxX64
./gradlew linkReleaseExecutableLinuxX64

# Run
./build/bin/linuxX64/releaseExecutable/myapp.kexe
```

---

## Interop กับ C

```kotlin
// สร้าง cinterop definition
// src/nativeInterop/cinterop/sqlite.def
headers = sqlite3.h
headerFilter = sqlite3.h
linkerOpts = -lsqlite3
```

```kotlin
// build.gradle.kts
kotlin {
    linuxX64 {
        compilations.getByName("main") {
            cinterops {
                val sqlite by creating {
                    defFile(project.file("src/nativeInterop/cinterop/sqlite.def"))
                    packageName("sqlite3")
                }
            }
        }
    }
}
```

```kotlin
// ใช้งาน SQLite C API จาก Kotlin
import sqlite3.*
import kotlinx.cinterop.*

@OptIn(ExperimentalForeignApi::class)
fun openDatabase(path: String): CPointer<sqlite3>? {
    return memScoped {
        val ppDb = allocPointerTo<sqlite3>()
        val rc = sqlite3_open(path, ppDb.ptr)
        if (rc != SQLITE_OK) {
            sqlite3_close(ppDb.value)
            throw RuntimeException("Cannot open database: ${sqlite3_errmsg(ppDb.value)?.toKString()}")
        }
        ppDb.value
    }
}

@OptIn(ExperimentalForeignApi::class)
fun executeQuery(db: CPointer<sqlite3>, sql: String): List<Map<String, String>> {
    val results = mutableListOf<Map<String, String>>()
    
    memScoped {
        val ppStmt = allocPointerTo<sqlite3_stmt>()
        val rc = sqlite3_prepare_v2(db, sql, -1, ppStmt.ptr, null)
        
        if (rc != SQLITE_OK) {
            throw RuntimeException("SQL error: ${sqlite3_errmsg(db)?.toKString()}")
        }
        
        val stmt = ppStmt.value!!
        
        while (sqlite3_step(stmt) == SQLITE_ROW) {
            val row = mutableMapOf<String, String>()
            val colCount = sqlite3_column_count(stmt)
            
            for (i in 0 until colCount) {
                val colName = sqlite3_column_name(stmt, i)?.toKString() ?: "col$i"
                val colValue = when (sqlite3_column_type(stmt, i)) {
                    SQLITE_INTEGER -> sqlite3_column_int64(stmt, i).toString()
                    SQLITE_FLOAT -> sqlite3_column_double(stmt, i).toString()
                    SQLITE_TEXT -> sqlite3_column_text(stmt, i)?.toKString() ?: "NULL"
                    SQLITE_NULL -> "NULL"
                    else -> "BLOB"
                }
                row[colName] = colValue
            }
            results.add(row)
        }
        
        sqlite3_finalize(stmt)
    }
    
    return results
}

@OptIn(ExperimentalForeignApi::class)
fun main() {
    val db = openDatabase("/tmp/test.db")!!
    
    // Create table
    memScoped {
        val errMsg = allocPointerTo<ByteVar>()
        sqlite3_exec(db, """
            CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY,
                name TEXT NOT NULL,
                email TEXT UNIQUE
            )
        """.trimIndent(), null, null, errMsg.ptr)
    }
    
    // Insert
    memScoped {
        sqlite3_exec(db, "INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com')",
            null, null, null)
    }
    
    // Query
    val users = executeQuery(db, "SELECT * FROM users")
    users.forEach { row ->
        println("User: ${row["name"]} (${row["email"]})")
    }
    
    sqlite3_close(db)
}
```

---

## Memory Model

Kotlin Native มี memory model ที่แตกต่างจาก JVM:

```kotlin
// Kotlin Native ใหม่ (1.7.20+) ใช้ New Memory Manager
// ซึ่งคล้าย JVM มากขึ้น — garbage collected, objects สามารถ share ได้

// Immutable objects (freeze) - สำหรับ share ข้าม coroutines/threads
// ใน new MM ไม่ต้องใช้ freeze แล้ว แต่ยังเข้าใจ concept ได้ดี

// Worker (thread) - ใช้สำหรับ parallel processing
@OptIn(ExperimentalNativeApi::class)
fun parallelProcessing() {
    val worker1 = Worker.start(name = "Worker-1")
    val worker2 = Worker.start(name = "Worker-2")
    
    val future1 = worker1.execute(TransferMode.SAFE, { "Task-1" }) { task ->
        println("Worker processing: $task")
        "Result-1"
    }
    
    val future2 = worker2.execute(TransferMode.SAFE, { "Task-2" }) { task ->
        println("Worker processing: $task")
        "Result-2"
    }
    
    println("Future 1: ${future1.result}")
    println("Future 2: ${future2.result}")
    
    worker1.requestTermination()
    worker2.requestTermination()
}

// AtomicReference - thread-safe shared state
@OptIn(ExperimentalNativeApi::class)
class Counter {
    private val count = AtomicInt(0)
    
    fun increment() = count.addAndGet(1)
    fun get() = count.value
}

// Freeze check (legacy)
fun legacyFreezeExample() {
    data class Config(val host: String, val port: Int)
    
    val config = Config("localhost", 8080)
    // config.freeze()  // Make immutable for sharing (legacy API)
    
    // In new MM, regular objects can be shared without freezing
}
```

---

## Platform-specific Code

```kotlin
// src/commonMain/kotlin/Platform.kt
expect fun getPlatformName(): String
expect fun readLine2(): String?
expect fun getCurrentTimeMillis(): Long

// src/linuxX64Main/kotlin/Platform.kt
import platform.posix.*

actual fun getPlatformName() = "Linux"

actual fun readLine2(): String? {
    return readLine()
}

actual fun getCurrentTimeMillis(): Long {
    val timespec = kotlinx.cinterop.nativeHeap.alloc<timespec>()
    clock_gettime(CLOCK_REALTIME, timespec.ptr)
    return timespec.tv_sec * 1000L + timespec.tv_nsec / 1_000_000L
}

// src/macosX64Main/kotlin/Platform.kt
import platform.Foundation.*

actual fun getPlatformName() = "macOS"

actual fun readLine2(): String? = readLine()

actual fun getCurrentTimeMillis(): Long =
    (NSDate.timeIntervalSinceReferenceDate() * 1000).toLong()

// src/mingwX64Main/kotlin/Platform.kt
import platform.windows.*

actual fun getPlatformName() = "Windows"

actual fun readLine2(): String? = readLine()

actual fun getCurrentTimeMillis(): Long {
    val ft = kotlinx.cinterop.nativeHeap.alloc<FILETIME>()
    GetSystemTimeAsFileTime(ft.ptr)
    val windowsEpoch = ((ft.dwHighDateTime.toLong() shl 32) or ft.dwLowDateTime.toLong())
    return (windowsEpoch - 116444736000000000L) / 10000L
}
```

---

## CLI Application

```kotlin
// src/nativeMain/kotlin/cli/Cli.kt

// Simple argument parser
data class CliArgs(
    val command: String,
    val options: Map<String, String>,
    val flags: Set<String>,
    val positional: List<String>
) {
    fun getOption(name: String, default: String = "") = options[name] ?: default
    fun hasFlag(name: String) = name in flags
    
    companion object {
        fun parse(args: Array<String>): CliArgs {
            val command = args.firstOrNull()?.takeIf { !it.startsWith("-") } ?: ""
            val options = mutableMapOf<String, String>()
            val flags = mutableSetOf<String>()
            val positional = mutableListOf<String>()
            
            var i = if (command.isNotEmpty()) 1 else 0
            while (i < args.size) {
                val arg = args[i]
                when {
                    arg.startsWith("--") && arg.contains("=") -> {
                        val (key, value) = arg.removePrefix("--").split("=", limit = 2)
                        options[key] = value
                    }
                    arg.startsWith("--") -> {
                        val key = arg.removePrefix("--")
                        if (i + 1 < args.size && !args[i + 1].startsWith("-")) {
                            options[key] = args[++i]
                        } else {
                            flags.add(key)
                        }
                    }
                    arg.startsWith("-") -> {
                        arg.removePrefix("-").forEach { flags.add(it.toString()) }
                    }
                    else -> positional.add(arg)
                }
                i++
            }
            
            return CliArgs(command, options, flags, positional)
        }
    }
}

// Terminal colors
object Colors {
    const val RESET = "\u001B[0m"
    const val RED = "\u001B[31m"
    const val GREEN = "\u001B[32m"
    const val YELLOW = "\u001B[33m"
    const val BLUE = "\u001B[34m"
    const val CYAN = "\u001B[36m"
    const val BOLD = "\u001B[1m"
}

fun printSuccess(msg: String) = println("${Colors.GREEN}✓ $msg${Colors.RESET}")
fun printError(msg: String) = println("${Colors.RED}✗ $msg${Colors.RESET}")
fun printWarning(msg: String) = println("${Colors.YELLOW}⚠ $msg${Colors.RESET}")
fun printInfo(msg: String) = println("${Colors.BLUE}ℹ $msg${Colors.RESET}")

// Progress indicator
fun withProgress(message: String, block: () -> Unit) {
    print("${Colors.CYAN}$message...${Colors.RESET} ")
    block()
    println("${Colors.GREEN}done${Colors.RESET}")
}

// File utilities using POSIX
@OptIn(kotlinx.cinterop.ExperimentalForeignApi::class)
fun readFileContents(path: String): String? {
    val file = platform.posix.fopen(path, "r") ?: return null
    
    val sb = StringBuilder()
    val buffer = ByteArray(4096)
    
    while (true) {
        val bytesRead = platform.posix.fread(buffer.refTo(0), 1, buffer.size.toULong(), file)
        if (bytesRead == 0UL) break
        sb.append(buffer.decodeToString(0, bytesRead.toInt()))
    }
    
    platform.posix.fclose(file)
    return sb.toString()
}

@OptIn(kotlinx.cinterop.ExperimentalForeignApi::class)
fun writeFileContents(path: String, content: String): Boolean {
    val file = platform.posix.fopen(path, "w") ?: return false
    platform.posix.fputs(content, file)
    platform.posix.fclose(file)
    return true
}

// Complete CLI app example: a simple task manager
data class Task(val id: Int, val title: String, val done: Boolean = false)

class TaskManager {
    private val tasks = mutableListOf<Task>()
    private var nextId = 1
    
    fun add(title: String): Task {
        val task = Task(nextId++, title)
        tasks.add(task)
        return task
    }
    
    fun complete(id: Int): Boolean {
        val index = tasks.indexOfFirst { it.id == id }
        if (index < 0) return false
        tasks[index] = tasks[index].copy(done = true)
        return true
    }
    
    fun list(showAll: Boolean = true) = 
        if (showAll) tasks.toList()
        else tasks.filter { !it.done }
    
    fun delete(id: Int) = tasks.removeAll { it.id == id }
}

fun main(args: Array<String>) {
    val cli = CliArgs.parse(args)
    val manager = TaskManager()
    
    when (cli.command) {
        "add" -> {
            val title = cli.positional.joinToString(" ")
            if (title.isBlank()) {
                printError("Please provide a task title")
                return
            }
            val task = manager.add(title)
            printSuccess("Added task #${task.id}: ${task.title}")
        }
        
        "list" -> {
            val showAll = !cli.hasFlag("pending")
            val tasks = manager.list(showAll)
            if (tasks.isEmpty()) {
                printInfo("No tasks found")
            } else {
                tasks.forEach { task ->
                    val status = if (task.done) "${Colors.GREEN}✓${Colors.RESET}" else "○"
                    println("  $status [${task.id}] ${task.title}")
                }
            }
        }
        
        "done" -> {
            val id = cli.positional.firstOrNull()?.toIntOrNull()
            if (id == null) {
                printError("Please provide task ID")
                return
            }
            if (manager.complete(id)) {
                printSuccess("Task #$id marked as done")
            } else {
                printError("Task #$id not found")
            }
        }
        
        "delete" -> {
            val id = cli.positional.firstOrNull()?.toIntOrNull()
            if (id == null) {
                printError("Please provide task ID")
                return
            }
            manager.delete(id)
            printSuccess("Task #$id deleted")
        }
        
        "help", "" -> {
            println("${Colors.BOLD}Task Manager CLI${Colors.RESET}")
            println()
            println("Usage: tasks <command> [options]")
            println()
            println("Commands:")
            println("  add <title>     Add a new task")
            println("  list [--pending] List tasks")
            println("  done <id>       Mark task as done")
            println("  delete <id>     Delete task")
            println("  help            Show this help")
        }
        
        else -> {
            printError("Unknown command: ${cli.command}")
            println("Run 'tasks help' for usage")
        }
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง Kotlin Native CLI tool สำหรับ JSON processing

// Requirements:
// 1. Read JSON from file หรือ stdin
// 2. Filter objects ตาม key=value
// 3. Select specific fields
// 4. Output formatted JSON หรือ CSV

// Example:
// $ jq-lite --file data.json --filter "age>18" --select "name,email"

// Hints:
// - ใช้ kotlinx.serialization สำหรับ JSON parsing
// - ใช้ platform.posix.fgets สำหรับอ่าน stdin
// - ใช้ expect/actual สำหรับ platform differences

// Starter:
@OptIn(ExperimentalForeignApi::class)
fun readStdin(): String {
    val sb = StringBuilder()
    val buffer = ByteArray(1024)
    while (true) {
        val line = platform.posix.fgets(buffer.refTo(0), buffer.size, platform.posix.stdin) 
            ?: break
        sb.append(line.toKString())
    }
    return sb.toString()
}
```

---

## สรุป Part 51

```
✅ Kotlin Native: compile to native binary, no JVM required
✅ Target platforms: Linux, macOS, Windows, iOS, WASM
✅ C Interop: call C libraries directly via cinterop
✅ SQLite example: using C API from Kotlin
✅ New Memory Manager: GC-based, similar to JVM
✅ Workers: true parallelism in Kotlin Native
✅ AtomicInt/AtomicReference: thread-safe primitives
✅ expect/actual: platform-specific implementations
✅ Platform APIs: POSIX, Win32, Foundation (macOS)
✅ CLI application: argument parsing, colored output
✅ File I/O: POSIX file operations
✅ Build: linkDebugExecutable, linkReleaseExecutable
✅ Use cases: CLI tools, iOS apps, shared KMP libraries
```

---

*Part 51/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
