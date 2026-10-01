# Part 08: Null Safety - ระบบป้องกัน Null

## สารบัญ
1. [ปัญหาของ Null](#ปัญหาของ-null)
2. [Nullable Types](#nullable-types)
3. [Safe Call Operator ?.](#safe-call-operator-)
4. [Elvis Operator ?:](#elvis-operator-)
5. [Non-null Assertion !!](#non-null-assertion-)
6. [Smart Casts](#smart-casts)
7. [let, run, also, apply กับ Null](#let-run-also-apply-กับ-null)
8. [Nullable Collections](#nullable-collections)
9. [Java Interoperability และ Platform Types](#java-interoperability-และ-platform-types)
10. [Best Practices](#best-practices)
11. [แบบฝึกหัด](#แบบฝึกหัด)

---

## ปัญหาของ Null

**NullPointerException (NPE)** เป็น Exception ที่พบบ่อยที่สุดใน Java:

```java
// Java - ปัญหา NullPointerException
public class JavaExample {
    public static void main(String[] args) {
        String name = null;
        System.out.println(name.length());  // 💥 NullPointerException!
        
        // ต้องตรวจสอบ null ทุกที่ = โค้ดยุ่งเหยิง
        if (name != null) {
            System.out.println(name.length());
        }
        
        // ลืมตรวจสอบ = CRASH!
        processUser(null);
    }
    
    static void processUser(User user) {
        // developer อาจลืมตรวจสอบ
        System.out.println(user.getName());  // 💥 ถ้า user เป็น null
    }
}
```

Tony Hoare ผู้คิดค้น null ในปี 1965 เรียกมันว่า **"Billion Dollar Mistake"** เพราะทำให้เสียหายทางเศรษฐกิจหลายพันล้านดอลลาร์

### Kotlin แก้ปัญหาอย่างไร?

```kotlin
// Kotlin - Null Safety ตั้งแต่ระดับ Type System
var name: String = null   // ❌ Compile Error!
var name: String? = null  // ✅ ต้องประกาศว่า nullable ด้วย ?

println(name.length)     // ❌ Compile Error! ต้องตรวจสอบก่อน
println(name?.length)    // ✅ Safe call - return null ถ้า null
```

---

## Nullable Types

### ประกาศ Nullable Type

```kotlin
fun main() {
    // Non-nullable (ค่า default)
    val name: String = "สมชาย"       // ✅ ต้องมีค่าเสมอ
    val age: Int = 25                  // ✅
    val score: Double = 98.5           // ✅
    
    // Nullable - ใส่ ? หลัง Type
    val nickname: String? = null       // ✅ อาจเป็น null ได้
    var email: String? = null          // ✅
    var phone: String? = "081-xxx-xxxx" // ✅ มีค่าหรือ null ก็ได้
    
    // Nullable Primitives
    val optionalAge: Int? = null
    val optionalScore: Double? = null
    
    // Nullable Reference Types
    val optionalUser: Any? = null
    val optionalList: List<String>? = null
    
    // ตรวจสอบว่าเป็น null ด้วย ==
    println(nickname == null)   // true
    println(email == null)      // true
    println(phone == null)      // false
    println(name == null)       // false (non-nullable)
    
    // ตรวจสอบว่ามีค่าด้วย != null
    if (phone != null) {
        println("โทรศัพท์: $phone")
    }
    
    println("มี email: ${email != null}")  // false
    println("มี phone: ${phone != null}")  // true
}
```

### ความแตกต่างระหว่าง T และ T?

```kotlin
fun main() {
    // String ≠ String?
    var a: String = "Hello"
    var b: String? = "World"
    
    // a รับค่าได้จาก a และ b (ถ้าไม่ null)
    // a = b  // ❌ Type mismatch: String? cannot be assigned to String
    a = b ?: ""  // ✅ ต้องจัดการ null case ก่อน
    
    // b รับค่าได้จาก a
    b = a   // ✅ String เป็น subtype ของ String?
    b = null // ✅
    
    // Kotlin Type Hierarchy:
    // Any
    //  ├── String (Non-nullable)
    //  └── String? (Nullable - รวม null ด้วย)
    //
    // String เป็น subtype ของ String?
    // ดังนั้น String ใช้ได้ทุกที่ที่ String? ต้องการ
    
    fun printLength(str: String?) {
        println("Length: ${str?.length ?: 0}")
    }
    
    printLength("Hello")  // Length: 5 - ส่ง String ให้ String? ได้
    printLength(null)     // Length: 0
}
```

---

## Safe Call Operator ?.

ใช้เรียก method/property บน nullable object:

```kotlin
data class Address(
    val street: String,
    val city: String?,
    val country: String
)

data class User(
    val name: String,
    val age: Int,
    val email: String?,
    val address: Address?
)

fun main() {
    val user1 = User("สมชาย", 25, null, null)
    val user2 = User(
        "สมหญิง", 30,
        "somying@example.com",
        Address("123 ถนนสุขุมวิท", "กรุงเทพฯ", "Thailand")
    )
    val user3 = User(
        "สมศักดิ์", 28,
        "somsak@example.com",
        Address("456 ถนนสีลม", null, "Thailand")
    )
    
    // Safe call ชั้นเดียว
    println(user1.email?.length)    // null
    println(user2.email?.length)    // 20
    println(user2.email?.uppercase()) // SOMYING@EXAMPLE.COM
    
    // Safe call chain (Deep access)
    println(user1.address?.city)    // null
    println(user2.address?.city)    // กรุงเทพฯ
    println(user3.address?.city)    // null
    
    // chain ลึก
    println(user1.address?.city?.length)    // null
    println(user2.address?.city?.length)    // 9
    
    // Safe call กับ function call
    val users = listOf(user1, user2, user3)
    
    users.forEach { user ->
        val cityInfo = user.address?.city?.let { "เมือง: $it" } ?: "ไม่ทราบเมือง"
        println("${user.name}: $cityInfo")
    }
    // สมชาย: ไม่ทราบเมือง
    // สมหญิง: เมือง: กรุงเทพฯ
    // สมศักดิ์: ไม่ทราบเมือง
    
    // Safe call กับ Collection operations
    val emailLengths = users.mapNotNull { it.email?.length }
    println("Email lengths: $emailLengths")  // [20, 18]
    
    // Safe call กับ index access
    val nullableList: List<String>? = listOf("a", "b", "c")
    println(nullableList?.get(1))   // b
    println(nullableList?.size)     // 3
    
    val emptyList: List<String>? = null
    println(emptyList?.get(1))  // null (ไม่ crash)
    println(emptyList?.size)    // null
}
```

---

## Elvis Operator ?:

กำหนดค่า default เมื่อ expression เป็น null:

```kotlin
data class Config(
    val host: String?,
    val port: Int?,
    val timeout: Int?,
    val maxConnections: Int?
)

fun main() {
    val config = Config(
        host = null,
        port = null,
        timeout = 30,
        maxConnections = null
    )
    
    // Elvis ?: ให้ค่า default
    val host = config.host ?: "localhost"
    val port = config.port ?: 8080
    val timeout = config.timeout ?: 60
    val maxConn = config.maxConnections ?: 100
    
    println("Host: $host")           // localhost
    println("Port: $port")           // 8080
    println("Timeout: $timeout")     // 30 (มีค่าอยู่แล้ว)
    println("Max Connections: $maxConn")  // 100
    
    // Elvis กับ chain
    val primaryEmail: String? = null
    val backupEmail: String? = null
    val defaultEmail = "noreply@example.com"
    
    val email = primaryEmail ?: backupEmail ?: defaultEmail
    println("Email: $email")  // noreply@example.com
    
    // Elvis กับ throw
    fun getUser(id: Int): User? {
        // จำลองการ query database
        return if (id == 1) User("Admin", 30, "admin@example.com", null)
               else null
    }
    
    fun processUser(id: Int) {
        val user = getUser(id) ?: throw NoSuchElementException("User #$id ไม่พบ")
        println("กำลัง process ${user.name}")
    }
    
    processUser(1)  // กำลัง process Admin
    
    try {
        processUser(999)  // throws NoSuchElementException
    } catch (e: NoSuchElementException) {
        println("Error: ${e.message}")  // Error: User #999 ไม่พบ
    }
    
    // Elvis กับ return (early return)
    fun sendWelcomeEmail(userId: Int) {
        val user = getUser(userId) ?: return  // ออกจาก function ถ้าไม่พบ user
        val email = user.email ?: return      // ออกถ้าไม่มี email
        println("ส่ง welcome email ไปที่ $email")
    }
    
    sendWelcomeEmail(1)    // ส่ง welcome email ไปที่ admin@example.com
    sendWelcomeEmail(999)  // ไม่ทำอะไร (user ไม่พบ)
    
    // Elvis กับ Elvis chain
    val result: Int = null?.let { 42 } ?: null?.let { 100 } ?: -1
    println("Result: $result")  // -1
}

data class User(val name: String, val age: Int, val email: String?, val address: Any?)
```

---

## Non-null Assertion !!

```kotlin
fun main() {
    // !! บอก compiler ว่า "ฉันรับประกันว่าค่านี้ไม่ null"
    
    var text: String? = "Hello Kotlin"
    val length = text!!.length  // ✅ ถ้า text ไม่เป็น null
    println(length)  // 12
    
    // ⚠️ อันตราย! ถ้าค่าเป็น null จะ throw KotlinNullPointerException
    text = null
    // text!!.length  // 💥 KotlinNullPointerException!
    
    // ควรใช้ !! เฉพาะเมื่อ:
    // 1. รู้แน่ชัดว่าค่าไม่เป็น null ในเวลานั้น
    // 2. Logic ของ program การันตีว่าไม่ null
    
    // ตัวอย่างที่ !! ใช้ได้สมเหตุสมผล
    val lines = "Line 1\nLine 2\nLine 3".lines()
    
    // เราแน่ใจว่า lines ไม่ว่าง เพราะ string ไม่ว่าง
    val firstLine = lines.firstOrNull()!!  // ยืนยันว่ามีค่า
    println(firstLine)  // Line 1
    
    // ตัวอย่างในการทดสอบ
    // ใน unit tests การใช้ !! บางทีสมเหตุสมผล
    // เพราะถ้า null แสดงว่า test fail
    fun findUser(id: Int): String? {
        return if (id > 0) "User #$id" else null
    }
    
    val user = findUser(1)!!  // ใน test context - ยืนยันว่าต้องพบ
    println(user)  // User #1
    
    // แนะนำ: ใช้ let หรือ require แทน !!
    
    // แทน: someNullable!!.process()
    // ใช้: someNullable?.let { it.process() }
    
    // แทน: val value = someNullable!!
    // ใช้: val value = someNullable ?: error("Should not be null")
    
    val nullableValue: String? = null
    // val safe = nullableValue ?: error("Value must not be null")  // throw IllegalStateException
    
    // !! cascade - อย่าทำ!
    // val bad = a!!.b!!.c!!.d  // ถ้า a, b, หรือ c เป็น null จะ crash พร้อมข้อความที่ไม่ชัดเจน
    
    // ดีกว่า:
    // val good = a?.b?.c?.d ?: defaultValue
}
```

---

## Smart Casts

Kotlin ทำการ cast ให้อัตโนมัติหลังจาก null check:

```kotlin
fun main() {
    var text: String? = "Hello"
    
    // ❌ ไม่มี smart cast - ต้อง cast เอง
    // val length = (text as String).length  // อันตราย ถ้า null
    
    // ✅ Smart Cast หลัง null check
    if (text != null) {
        // ภายใน block นี้ text เป็น String (non-nullable) อัตโนมัติ!
        println(text.length)       // ไม่ต้อง text?.length
        println(text.uppercase())  // ไม่ต้อง cast!
    }
    
    // Smart Cast กับ && 
    val nullable: String? = "World"
    if (nullable != null && nullable.length > 3) {  // Smart cast ทำงาน
        println(nullable.uppercase())  // WORLD
    }
    
    // Smart Cast ไม่ทำงาน กับ var ที่ "อาจ" เปลี่ยนค่า
    var mutable: String? = "Hello"
    if (mutable != null) {
        // mutable.length  // ❌ อาจไม่ทำงาน เพราะ mutable อาจเปลี่ยนค่าจาก thread อื่น
        println(mutable!!.length)  // ต้องใช้ !!
    }
    
    // หรือใช้ val local variable
    val local = mutable
    if (local != null) {
        println(local.length)  // ✅ Smart cast ทำงาน
    }
    
    // Smart Cast กับ is
    fun processAny(obj: Any) {
        when (obj) {
            is String  -> println("String: ${obj.uppercase()}")   // obj เป็น String
            is Int     -> println("Int: ${obj * 2}")               // obj เป็น Int
            is Double  -> println("Double: ${"%.2f".format(obj)}") // obj เป็น Double
            is List<*> -> println("List size: ${obj.size}")         // obj เป็น List
            else       -> println("Unknown: $obj")
        }
    }
    
    processAny("hello")           // String: HELLO
    processAny(42)                // Int: 84
    processAny(3.14)              // Double: 3.14
    processAny(listOf(1, 2, 3))  // List size: 3
    
    // Smart Cast กับ early return
    fun processText(text: String?): Int {
        text ?: return -1  // ถ้า text เป็น null ให้ return -1
        // หลังบรรทัดนี้ text เป็น String (non-nullable) แน่นอน
        return text.length
    }
    
    println(processText(null))    // -1
    println(processText("Hello")) // 5
}
```

---

## let, run, also, apply กับ Null

```kotlin
data class User(
    val name: String,
    var email: String?,
    var phone: String?
)

fun main() {
    val user: User? = User("สมชาย", "somchai@example.com", null)
    
    // let: ใช้บ่อยมากกับ null check
    // ถ้า user ไม่เป็น null จะ execute block และ return ผลลัพธ์
    user?.let { u ->
        println("ชื่อ: ${u.name}")
        println("อีเมล: ${u.email ?: "ไม่มี"}")
        println("โทร: ${u.phone ?: "ไม่มี"}")
    }
    
    // let + Elvis สำหรับ default value
    val greeting = user?.let { "สวัสดี ${it.name}!" } ?: "กรุณาเข้าสู่ระบบ"
    println(greeting)
    
    // run: เหมือน let แต่ this แทน it
    user?.run {
        // this = user
        println("ผู้ใช้: $name")
        email?.let { println("อีเมล: $it") }
    }
    
    // also: ใช้สำหรับ side effects (logging, debugging)
    val processedUser = user?.also { u ->
        println("[LOG] กำลัง process user: ${u.name}")
    }
    println("Processed: ${processedUser?.name}")
    
    // apply: ใช้สำหรับ configure object
    user?.apply {
        // this = user
        email = email ?: "default@example.com"  // set default email
        phone = phone ?: "N/A"
        println("Updated user: $this")
    }
    println("Final user: $user")
    
    // Nested let
    val nullableUser: User? = User("สมหญิง", "somying@example.com", "081-xxx-xxxx")
    
    nullableUser?.let { u ->
        u.email?.let { email ->
            println("Email ของ ${u.name}: $email")
        }
        u.phone?.let { phone ->
            println("Phone ของ ${u.name}: $phone")
        }
    }
    
    // หรือใช้ chain
    nullableUser
        ?.also { println("Processing ${it.name}") }
        ?.let { u ->
            mapOf(
                "name" to u.name,
                "email" to (u.email ?: "N/A"),
                "phone" to (u.phone ?: "N/A")
            )
        }
        ?.also { println("Result: $it") }
}
```

---

## Nullable Collections

```kotlin
fun main() {
    // List<String> vs List<String?>
    val list1: List<String> = listOf("a", "b", "c")       // ไม่มี null ใน list
    val list2: List<String?> = listOf("a", null, "c")      // อาจมี null
    val list3: List<String>? = null                         // list เองอาจ null
    val list4: List<String?>? = null                        // ทั้งคู่อาจ null
    
    // กรอง null ออกด้วย filterNotNull()
    val withNulls: List<String?> = listOf("apple", null, "banana", null, "cherry")
    val withoutNulls: List<String> = withNulls.filterNotNull()
    println(withoutNulls)  // [apple, banana, cherry]
    
    // mapNotNull - map + filter null ในครั้งเดียว
    val strings = listOf("1", "2", "abc", "3", "def", "4")
    val numbers = strings.mapNotNull { it.toIntOrNull() }
    println(numbers)  // [1, 2, 3, 4]
    
    // orEmpty() สำหรับ nullable collection
    val nullableList: List<String>? = null
    val safeList = nullableList.orEmpty()  // return empty list ถ้า null
    println(safeList.size)  // 0
    println(safeList.isEmpty())  // true
    
    // ?.orEmpty() pattern
    fun getUsers(): List<String>? = null
    val users = getUsers()?.orEmpty() ?: emptyList()
    println("Users: ${users.size}")  // Users: 0
    
    // Map กับ null values
    val map1: Map<String, String?> = mapOf(
        "name" to "สมชาย",
        "email" to null,
        "phone" to "081-xxx-xxxx"
    )
    
    // กรองเฉพาะที่มีค่า
    val nonNullEntries = map1.filterValues { it != null }
    println(nonNullEntries)  // {name=สมชาย, phone=081-xxx-xxxx}
    
    // mapValues กับ null
    val processedMap = map1.mapValues { (_, v) -> v ?: "N/A" }
    println(processedMap)  // {name=สมชาย, email=N/A, phone=081-xxx-xxxx}
    
    // Null-safe collection operations
    fun processUsers(users: List<String>?) {
        val count = users?.size ?: 0
        val firstUser = users?.firstOrNull() ?: "ไม่มีผู้ใช้"
        val emails = users?.map { "$it@example.com" } ?: emptyList()
        
        println("จำนวน: $count")
        println("คนแรก: $firstUser")
        println("Emails: $emails")
    }
    
    processUsers(listOf("Admin", "User1", "User2"))
    processUsers(null)
    processUsers(emptyList())
}
```

---

## Java Interoperability และ Platform Types

เมื่อใช้ Java API ใน Kotlin จะได้ **Platform Types** (T!):

```kotlin
import java.util.Date

fun main() {
    // Java method ที่อาจ return null
    // เช่น System.getenv() return String? (platform type)
    
    val javaPath = System.getenv("PATH")  // String! (platform type)
    // ทั้ง String และ String? ก็ได้
    
    // ❌ อาจ crash ถ้า getenv return null
    // println(javaPath.length)
    
    // ✅ ปลอดภัยกว่า
    println(javaPath?.length ?: "PATH not found")
    
    // Java @Nullable annotation → Kotlin รู้ว่าเป็น Nullable
    // Java @NotNull annotation → Kotlin รู้ว่าเป็น Non-nullable
    
    // ตัวอย่าง: Date
    val date = Date()
    val toStr = date.toString()  // String! (อาจ null ในทาง theory)
    println(toStr)
    
    // Best practice: treat platform types as nullable
    fun safePlatformType(str: String?): String {
        return str ?: "default"
    }
    
    val env = System.getenv("MY_VAR")
    val safeEnv = safePlatformType(env)
    println("MY_VAR: $safeEnv")
    
    // @JvmField, @JvmStatic - สำหรับ Java interop
    // (จะอธิบายละเอียดใน Part Java Interop)
}
```

---

## Best Practices

### 1. ใช้ val กับ Non-nullable เสมอเมื่อทำได้

```kotlin
// ❌ ไม่ดี - nullable โดยไม่จำเป็น
var name: String? = "สมชาย"
println(name?.length)  // ต้องใช้ safe call ทุกครั้ง

// ✅ ดีกว่า - non-nullable
val name = "สมชาย"
println(name.length)   // ใช้โดยตรง
```

### 2. ใช้ Elvis สำหรับ Default Values

```kotlin
fun displayUserName(name: String?): String {
    // ✅ ดี - กระชับ
    return name ?: "ไม่ทราบชื่อ"
    
    // ❌ ยาวกว่าโดยไม่จำเป็น
    // return if (name != null) name else "ไม่ทราบชื่อ"
}
```

### 3. ใช้ let สำหรับ null-safe block

```kotlin
val user: String? = "สมชาย"

// ✅ ใช้ let
user?.let { name ->
    println("ชื่อ: $name")
    println("ความยาว: ${name.length}")
}

// ❌ ยาวกว่า
if (user != null) {
    println("ชื่อ: $user")
    println("ความยาว: ${user.length}")
}
```

### 4. ใช้ filterNotNull และ mapNotNull

```kotlin
val names: List<String?> = listOf("Alice", null, "Bob", null, "Charlie")

// ✅ กระชับ
val validNames = names.filterNotNull()
println(validNames)  // [Alice, Bob, Charlie]

// ✅ map + filter ในครั้งเดียว
val data = listOf("1", "abc", "2", "xyz", "3")
val numbers = data.mapNotNull { it.toIntOrNull() }
println(numbers)  // [1, 2, 3]
```

### 5. หลีกเลี่ยง !! ให้มากที่สุด

```kotlin
// ❌ อันตราย
fun bad(str: String?) = str!!.length

// ✅ ปลอดภัย
fun good(str: String?) = str?.length ?: 0

// ✅ ถ้าต้องการ throw error
fun better(str: String?): Int {
    val nonNull = str ?: throw IllegalArgumentException("String ต้องไม่เป็น null")
    return nonNull.length
}
```

### 6. ใช้ require/check สำหรับ Validation

```kotlin
fun processAge(age: Int?) {
    requireNotNull(age) { "Age must not be null" }
    require(age in 0..150) { "Age must be between 0 and 150" }
    // หลังจากนี้ age ≠ null
    println("Processing age: $age")
}

fun getUser(id: Int?): String {
    checkNotNull(id) { "User ID must not be null" }
    check(id > 0) { "User ID must be positive" }
    return "User #$id"
}
```

---

## ตัวอย่างสรุปรวม: ระบบจัดการผู้ใช้

```kotlin
data class Address(val street: String, val city: String?, val zip: String?)
data class User(
    val id: Int,
    val name: String,
    val email: String?,
    val phone: String?,
    val address: Address?
)

class UserService {
    private val users = mutableListOf<User>()
    
    fun addUser(user: User) = users.add(user)
    
    fun findById(id: Int): User? = users.find { it.id == id }
    
    fun getEmail(userId: Int): String {
        return findById(userId)?.email ?: "ไม่มีอีเมล"
    }
    
    fun getCity(userId: Int): String {
        return findById(userId)?.address?.city ?: "ไม่ทราบเมือง"
    }
    
    fun sendNotification(userId: Int, message: String): Boolean {
        val user = findById(userId) ?: run {
            println("ไม่พบ user #$userId")
            return false
        }
        
        val contact = user.email ?: user.phone ?: run {
            println("ไม่มีช่องทางติดต่อสำหรับ ${user.name}")
            return false
        }
        
        println("ส่ง '$message' ไปยัง ${user.name} ที่ $contact")
        return true
    }
    
    fun generateReport(): String {
        return buildString {
            appendLine("=== รายงานผู้ใช้ ===")
            appendLine("จำนวนทั้งหมด: ${users.size} คน")
            
            val withEmail = users.count { it.email != null }
            val withPhone = users.count { it.phone != null }
            val withAddress = users.count { it.address != null }
            val withCity = users.count { it.address?.city != null }
            
            appendLine("มีอีเมล: $withEmail คน")
            appendLine("มีโทรศัพท์: $withPhone คน")
            appendLine("มีที่อยู่: $withAddress คน")
            appendLine("มีชื่อเมือง: $withCity คน")
            
            appendLine("\n=== รายละเอียด ===")
            users.forEach { user ->
                appendLine("\nUser #${user.id}: ${user.name}")
                appendLine("  Email: ${user.email ?: "N/A"}")
                appendLine("  Phone: ${user.phone ?: "N/A"}")
                user.address?.let { addr ->
                    appendLine("  ที่อยู่: ${addr.street}")
                    appendLine("  เมือง: ${addr.city ?: "N/A"}")
                    appendLine("  ZIP: ${addr.zip ?: "N/A"}")
                } ?: appendLine("  ที่อยู่: ไม่ระบุ")
            }
        }
    }
}

fun main() {
    val service = UserService()
    
    service.addUser(User(1, "สมชาย", "somchai@example.com", "081-111-1111",
        Address("123 ถ.สุขุมวิท", "กรุงเทพฯ", "10110")))
    
    service.addUser(User(2, "สมหญิง", null, "082-222-2222",
        Address("456 ถ.สีลม", null, null)))
    
    service.addUser(User(3, "สมศักดิ์", "somsak@example.com", null, null))
    
    service.addUser(User(4, "สมพร", null, null,
        Address("789 ถ.รัชดา", "กรุงเทพฯ", "10400")))
    
    println("Email ของ user 1: ${service.getEmail(1)}")
    println("Email ของ user 2: ${service.getEmail(2)}")
    println("Email ของ user 99: ${service.getEmail(99)}")
    
    println("\nCity ของ user 1: ${service.getCity(1)}")
    println("City ของ user 2: ${service.getCity(2)}")
    println("City ของ user 3: ${service.getCity(3)}")
    
    println()
    service.sendNotification(1, "ยินดีต้อนรับ!")
    service.sendNotification(2, "อัปเดตรหัสผ่าน")
    service.sendNotification(4, "ส่วนลดพิเศษ")
    service.sendNotification(5, "Test")
    
    println()
    println(service.generateReport())
}
```

---

## แบบฝึกหัด

### Exercise 1: Safe Navigation
```kotlin
data class Company(val name: String, val ceo: Employee?)
data class Employee(val name: String, val department: Department?)
data class Department(val name: String, val budget: Double?)

fun main() {
    val company = Company("TechCorp",
        Employee("สมชาย",
            Department("IT", 1_000_000.0)
        )
    )
    
    // ดึง CEO name, Department, Budget แบบ null-safe
    val ceoName = company.ceo?.name ?: "ไม่มี CEO"
    val deptName = company.ceo?.department?.name ?: "ไม่มีแผนก"
    val budget = company.ceo?.department?.budget?.let {
        "${String.format("%,.0f", it)} บาท"
    } ?: "ไม่ทราบ"
    
    println("CEO: $ceoName")
    println("แผนก: $deptName")
    println("งบประมาณ: $budget")
}
```

### Exercise 2: Null-safe Data Processing

```kotlin
fun processScores(scores: List<Int?>): Map<String, Any> {
    val nonNull = scores.filterNotNull()
    
    return mapOf(
        "total" to scores.size,
        "valid" to nonNull.size,
        "invalid" to scores.count { it == null },
        "average" to if (nonNull.isEmpty()) "N/A" else "%.2f".format(nonNull.average()),
        "max" to (nonNull.maxOrNull()?.toString() ?: "N/A"),
        "min" to (nonNull.minOrNull()?.toString() ?: "N/A")
    )
}

fun main() {
    val scores = listOf(85, null, 92, null, 78, 95, null, 88)
    val result = processScores(scores)
    result.forEach { (key, value) -> println("$key: $value") }
}
```

---

## สรุป Part 08

```
✅ T ≠ T?: Non-nullable vs Nullable types
✅ ?. Safe Call: ไม่ crash ถ้าเป็น null
✅ ?: Elvis Operator: กำหนดค่า default
✅ !! Non-null Assertion: ระวัง! อาจ crash
✅ Smart Cast: auto-cast หลัง null check
✅ let: null-safe block execution
✅ filterNotNull(), mapNotNull(): กรอง null จาก collection
✅ orEmpty(): list ว่างแทน null
✅ requireNotNull(), checkNotNull(): validation
✅ Best practice: ใช้ val, ?, let, Elvis มากกว่า !!
```

---

*Part 08/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
