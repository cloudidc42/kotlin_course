# Part 23: Delegation และ Property Delegates

## สารบัญ
1. [Interface Delegation](#interface-delegation)
2. [Property Delegates](#property-delegates)
3. [Built-in Delegates](#built-in-delegates)
4. [Custom Property Delegate](#custom-property-delegate)
5. [ReadOnlyProperty และ ReadWriteProperty](#readonlyproperty-และ-readwriteproperty)
6. [Delegate Providers](#delegate-providers)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Interface Delegation

```kotlin
// Delegation: delegate implementation ให้ object อื่น

interface Logger {
    fun log(message: String)
    fun warn(message: String)
    fun error(message: String)
}

class ConsoleLogger : Logger {
    override fun log(message: String)   = println("[LOG] $message")
    override fun warn(message: String)  = println("[WARN] $message")
    override fun error(message: String) = println("[ERROR] $message")
}

class FileLogger(private val filename: String) : Logger {
    override fun log(message: String)   = appendToFile("[$filename][LOG] $message")
    override fun warn(message: String)  = appendToFile("[$filename][WARN] $message")
    override fun error(message: String) = appendToFile("[$filename][ERROR] $message")
    private fun appendToFile(line: String) = println(line)  // simplified
}

// Delegate implementation to another object
class UserService(logger: Logger) : Logger by logger {
    fun createUser(name: String) {
        log("Creating user: $name")  // delegates to logger
        // ... business logic
        log("User $name created")
    }
}

// Override some methods while delegating others
class TimestampLogger(private val delegate: Logger) : Logger by delegate {
    private fun timestamp() = "[${java.time.LocalTime.now()}]"
    
    override fun error(message: String) {
        // Override error to add timestamp, delegate others
        delegate.error("${timestamp()} $message")
    }
}

// Multiple delegation
interface Drawable {
    fun draw(): String
}

interface Resizable {
    fun resize(factor: Double)
}

class CircleDrawer : Drawable {
    override fun draw() = "Drawing circle"
}

class CircleResizer : Resizable {
    var scaleFactor = 1.0
    override fun resize(factor: Double) {
        scaleFactor *= factor
        println("Resized to factor: $scaleFactor")
    }
}

class Circle(
    drawer: Drawable = CircleDrawer(),
    resizer: Resizable = CircleResizer()
) : Drawable by drawer, Resizable by resizer

fun main() {
    val service = UserService(ConsoleLogger())
    service.createUser("สมชาย")
    service.warn("Low memory")
    
    println()
    
    val fileService = UserService(FileLogger("users.log"))
    fileService.createUser("สมหญิง")
    
    println()
    
    val tsLogger = TimestampLogger(ConsoleLogger())
    tsLogger.log("Normal log")        // from ConsoleLogger
    tsLogger.error("Critical error")  // from TimestampLogger
    
    println()
    
    val circle = Circle()
    println(circle.draw())
    circle.resize(1.5)
}
```

---

## Built-in Delegates

```kotlin
import kotlin.properties.Delegates

class UserProfile {
    // lazy: คำนวณครั้งแรกที่ access, thread-safe by default
    val displayName: String by lazy {
        println("Computing displayName...")
        "${firstName} ${lastName}".trim()
    }
    
    var firstName = "สมชาย"
    var lastName = "ใจดี"
    
    // observable: เรียก lambda เมื่อค่าเปลี่ยน
    var age: Int by Delegates.observable(0) { property, old, new ->
        println("${property.name}: $old → $new")
    }
    
    // vetoable: block การเปลี่ยนค่าถ้า condition ไม่ผ่าน
    var email: String by Delegates.vetoable("") { _, _, new ->
        new.contains("@").also { valid ->
            if (!valid) println("Invalid email rejected: $new")
        }
    }
    
    // notNull: initialized lazily, throws if accessed before set
    var config: String by Delegates.notNull()
}

fun main() {
    val profile = UserProfile()
    
    // lazy
    println("Before access")
    println(profile.displayName)  // "Computing displayName..."
    println(profile.displayName)  // cached, no recompute
    
    println()
    
    // observable
    profile.age = 25   // age: 0 → 25
    profile.age = 26   // age: 25 → 26
    
    println()
    
    // vetoable
    profile.email = "valid@email.com"   // accepted
    println("Email: ${profile.email}")
    profile.email = "invalid-email"     // rejected
    println("Email: ${profile.email}")  // unchanged
    
    // lateinit (built-in, not delegate)
    lateinit var data: String
    data = "initialized"
    println(data)
    println(::data.isInitialized)  // true
    
    println()
    
    // Lazy modes
    val syncLazy by lazy(LazyThreadSafetyMode.SYNCHRONIZED) { "thread-safe" }
    val noneThreadLazy by lazy(LazyThreadSafetyMode.NONE) { "not thread-safe" }
    val pubLazy by lazy(LazyThreadSafetyMode.PUBLICATION) { "safe once published" }
    
    println(syncLazy)
}
```

---

## Custom Property Delegate

```kotlin
import kotlin.reflect.KProperty

// Custom delegate: implement getValue (and setValue for var)

// Read-only delegate
class Uppercase {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return "${property.name} (uppercase delegate)"
    }
}

// Read-write delegate
class Clamped(private val min: Int, private val max: Int) {
    private var value: Int = min
    
    operator fun getValue(thisRef: Any?, property: KProperty<*>): Int = value
    
    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: Int) {
        value = newValue.coerceIn(min, max)
    }
}

// Expiring value delegate
class Expiring<T>(private val ttlMs: Long) {
    private var value: T? = null
    private var setAt: Long = 0
    
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T? {
        return if (System.currentTimeMillis() - setAt < ttlMs) value else null
    }
    
    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T?) {
        value = newValue
        setAt = System.currentTimeMillis()
    }
}

// History-tracking delegate
class Tracked<T>(initial: T) {
    private var current = initial
    val history = mutableListOf(initial)
    
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T = current
    
    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {
        if (newValue != current) {
            history.add(newValue)
            current = newValue
        }
    }
}

// Map delegate (useful for serialization)
class Config(map: Map<String, Any?>) {
    val host: String by map
    val port: Int by map
    val debug: Boolean by map
}

class MutableConfig(map: MutableMap<String, Any?>) {
    var host: String by map
    var port: Int by map
    var debug: Boolean by map
}

class Example {
    val text: String by Uppercase()
    
    var volume: Int by Clamped(0, 100)
    
    var token: String? by Expiring(ttlMs = 5000L)  // 5 second TTL
    
    var status: String by Tracked("IDLE")
}

fun main() {
    val ex = Example()
    
    println(ex.text)
    
    ex.volume = 150  // clamped to 100
    println("Volume: ${ex.volume}")
    ex.volume = -10  // clamped to 0
    println("Volume: ${ex.volume}")
    
    ex.token = "abc-123"
    println("Token: ${ex.token}")  // "abc-123"
    // After TTL, token would be null
    
    ex.status = "RUNNING"
    ex.status = "COMPLETED"
    ex.status = "RUNNING"
    
    val delegate = Example::status.getDelegate(ex) as Tracked<*>
    println("Status history: ${delegate.history}")
    
    println()
    
    // Map delegate
    val config = Config(mapOf("host" to "localhost", "port" to 5432, "debug" to true))
    println("${config.host}:${config.port} debug=${config.debug}")
    
    val mutable = MutableConfig(mutableMapOf("host" to "prod.server.com", "port" to 80, "debug" to false))
    println("${mutable.host}:${mutable.port}")
    mutable.port = 443
    println("${mutable.host}:${mutable.port}")
}
```

---

## Delegate Providers

```kotlin
import kotlin.properties.PropertyDelegateProvider
import kotlin.reflect.KProperty

// Delegate Provider: สร้าง delegate ณ เวลา property definition
// ได้รับ property name และ thisRef

class NamedDelegate(val name: String) {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): String {
        return "Property '${property.name}' delegate name: $name"
    }
}

class NameAwareDelegateProvider : PropertyDelegateProvider<Any?, NamedDelegate> {
    override fun provideDelegate(thisRef: Any?, property: KProperty<*>): NamedDelegate {
        println("Creating delegate for property: ${property.name}")
        return NamedDelegate(property.name)
    }
}

// Registry pattern with delegates
class DelegateRegistry {
    val registry = mutableMapOf<String, Any?>()
    
    fun <T> registered(default: T): PropertyDelegateProvider<Any?, RegistryDelegate<T>> {
        return PropertyDelegateProvider { _, property ->
            RegistryDelegate(property.name, default, registry)
        }
    }
}

class RegistryDelegate<T>(
    private val key: String,
    private val default: T,
    private val registry: MutableMap<String, Any?>
) {
    @Suppress("UNCHECKED_CAST")
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T =
        registry.getOrDefault(key, default) as T
    
    operator fun setValue(thisRef: Any?, property: KProperty<*>, value: T) {
        registry[key] = value
    }
}

val delegateRegistry = DelegateRegistry()

class AppSettings {
    var theme: String by delegateRegistry.registered("light")
    var language: String by delegateRegistry.registered("th")
    var fontSize: Int by delegateRegistry.registered(14)
}

fun main() {
    val provider = NameAwareDelegateProvider()
    
    class User {
        val name: String by provider
        val email: String by provider
    }
    
    val user = User()
    println(user.name)
    println(user.email)
    
    println()
    
    val settings = AppSettings()
    println("Theme: ${settings.theme}")
    println("Language: ${settings.language}")
    println("Font: ${settings.fontSize}")
    
    settings.theme = "dark"
    settings.fontSize = 16
    
    println("\nAfter change:")
    println("Theme: ${settings.theme}")
    println("Font: ${settings.fontSize}")
    println("Registry: ${delegateRegistry.registry}")
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง Form Validation ด้วย Delegates

class ValidationException(val errors: Map<String, String>) : Exception(
    "Validation failed: ${errors.entries.joinToString { "${it.key}: ${it.value}" }}"
)

class ValidatedProperty<T>(
    private var value: T,
    private val validators: List<Pair<String, (T) -> Boolean>>
) {
    operator fun getValue(thisRef: Any?, property: KProperty<*>): T = value
    
    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: T) {
        val errors = validators
            .filter { (_, validate) -> !validate(newValue) }
            .map { (message, _) -> message }
        
        if (errors.isNotEmpty()) {
            throw ValidationException(mapOf(property.name to errors.joinToString("; ")))
        }
        value = newValue
    }
}

fun <T> validated(
    initial: T,
    vararg validators: Pair<String, (T) -> Boolean>
): ValidatedProperty<T> = ValidatedProperty(initial, validators.toList())

class UserForm {
    var name: String by validated(
        "",
        "Name is required" to { it.isNotBlank() },
        "Name too short (min 2)" to { it.length >= 2 },
        "Name too long (max 50)" to { it.length <= 50 }
    )
    
    var age: Int by validated(
        0,
        "Age must be 18+" to { it >= 18 },
        "Age too high (max 120)" to { it <= 120 }
    )
    
    var email: String by validated(
        "",
        "Email required" to { it.isNotBlank() },
        "Invalid email format" to { it.contains("@") && it.contains(".") }
    )
    
    var password: String by validated(
        "",
        "Password required" to { it.isNotBlank() },
        "Password too short (min 8)" to { it.length >= 8 },
        "Need uppercase letter" to { it.any { c -> c.isUpperCase() } },
        "Need digit" to { it.any { c -> c.isDigit() } }
    )
}

fun main() {
    val form = UserForm()
    
    // Valid values
    try {
        form.name = "สมชาย ใจดี"
        println("✅ name set")
    } catch (e: ValidationException) {
        println("❌ ${e.message}")
    }
    
    // Invalid age
    try {
        form.age = 15
        println("✅ age set")
    } catch (e: ValidationException) {
        println("❌ ${e.message}")
    }
    
    // Valid age
    try {
        form.age = 25
        println("✅ age set to ${form.age}")
    } catch (e: ValidationException) {
        println("❌ ${e.message}")
    }
    
    // Invalid password
    try {
        form.password = "weak"
        println("✅ password set")
    } catch (e: ValidationException) {
        println("❌ ${e.message}")
    }
    
    // Valid password
    try {
        form.password = "SecurePass1"
        println("✅ password set")
    } catch (e: ValidationException) {
        println("❌ ${e.message}")
    }
}
```

---

## สรุป Part 23

```
✅ Interface delegation: class Foo(bar: Bar) : Bar by bar
✅ Delegation ลด boilerplate code
✅ Override บางส่วนของ delegation ได้
✅ by lazy { }: คำนวณครั้งเดียว, thread-safe
✅ by Delegates.observable(): callback เมื่อค่าเปลี่ยน
✅ by Delegates.vetoable(): block การเปลี่ยนค่าได้
✅ by Delegates.notNull(): ต้องตั้งค่าก่อน access
✅ Custom delegate: operator getValue/setValue
✅ PropertyDelegateProvider: รับ property name ณ definition time
✅ Map delegate: by map / by mutableMap
✅ Delegate pattern เหมาะสำหรับ lazy init, caching, validation
```

---

*Part 23/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
