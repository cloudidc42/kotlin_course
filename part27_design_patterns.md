# Part 27: Design Patterns ใน Kotlin

## สารบัญ
1. [Creational Patterns](#creational-patterns)
2. [Structural Patterns](#structural-patterns)
3. [Behavioral Patterns](#behavioral-patterns)
4. [Kotlin-specific Patterns](#kotlin-specific-patterns)
5. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Creational Patterns

### Singleton

```kotlin
// Kotlin object = singleton
object AppConfig {
    var debug = false
    var version = "1.0.0"
    var apiUrl = "https://api.example.com"
    
    fun printConfig() {
        println("Config: debug=$debug, version=$version, apiUrl=$apiUrl")
    }
}

// Thread-safe lazy singleton
class Database private constructor(val url: String) {
    companion object {
        @Volatile
        private var instance: Database? = null
        
        fun getInstance(url: String): Database {
            return instance ?: synchronized(this) {
                instance ?: Database(url).also { instance = it }
            }
        }
    }
    
    fun query(sql: String) = "Result of: $sql"
}
```

### Factory Method

```kotlin
interface Button {
    fun render(): String
    fun onClick(): String
}

class WindowsButton : Button {
    override fun render() = "[Windows Button]"
    override fun onClick() = "Windows click!"
}

class MacButton : Button {
    override fun render() = "(Mac Button)"
    override fun onClick() = "Mac click!"
}

class LinuxButton : Button {
    override fun render() = "<Linux Button>"
    override fun onClick() = "Linux click!"
}

abstract class Dialog {
    abstract fun createButton(): Button
    
    fun renderDialog(): String {
        val button = createButton()
        return "Dialog with ${button.render()}"
    }
}

class WindowsDialog : Dialog() {
    override fun createButton() = WindowsButton()
}

class MacDialog : Dialog() {
    override fun createButton() = MacButton()
}

// Factory function (Kotlin style)
fun createButton(os: String): Button = when (os.lowercase()) {
    "windows" -> WindowsButton()
    "mac"     -> MacButton()
    "linux"   -> LinuxButton()
    else      -> throw IllegalArgumentException("Unknown OS: $os")
}
```

### Abstract Factory

```kotlin
interface CheckBox {
    fun check(): String
    fun uncheck(): String
}

interface TextField {
    fun input(text: String): String
    fun getValue(): String
}

// Families of related objects
interface UIFactory {
    fun createButton(): Button
    fun createCheckBox(): CheckBox
    fun createTextField(): TextField
}

class WindowsCheckBox : CheckBox {
    private var checked = false
    override fun check() = "☑ Checked (Win)"
    override fun uncheck() = "☐ Unchecked (Win)"
}

class MacCheckBox : CheckBox {
    override fun check() = "✓ Checked (Mac)"
    override fun uncheck() = "○ Unchecked (Mac)"
}

class WindowsTextField : TextField {
    private var value = ""
    override fun input(text: String) = also { value = text }.let { "Win TextField: $text" }
    override fun getValue() = value
}

class MacTextField : TextField {
    private var value = ""
    override fun input(text: String) = also { value = text }.let { "Mac TextField: $text" }
    override fun getValue() = value
}

class WindowsFactory : UIFactory {
    override fun createButton() = WindowsButton()
    override fun createCheckBox() = WindowsCheckBox()
    override fun createTextField() = WindowsTextField()
}

class MacFactory : UIFactory {
    override fun createButton() = MacButton()
    override fun createCheckBox() = MacCheckBox()
    override fun createTextField() = MacTextField()
}
```

### Builder

```kotlin
data class Pizza(
    val size: String,
    val crust: String,
    val toppings: List<String>,
    val sauce: String,
    val cheese: String,
    val extraCheese: Boolean
) {
    class Builder {
        private var size = "medium"
        private var crust = "thin"
        private val toppings = mutableListOf<String>()
        private var sauce = "tomato"
        private var cheese = "mozzarella"
        private var extraCheese = false
        
        fun size(size: String) = apply { this.size = size }
        fun crust(crust: String) = apply { this.crust = crust }
        fun topping(topping: String) = apply { toppings.add(topping) }
        fun toppings(vararg toppings: String) = apply { this.toppings.addAll(toppings) }
        fun sauce(sauce: String) = apply { this.sauce = sauce }
        fun cheese(cheese: String) = apply { this.cheese = cheese }
        fun extraCheese() = apply { extraCheese = true }
        
        fun build() = Pizza(size, crust, toppings.toList(), sauce, cheese, extraCheese)
    }
}

// Kotlin DSL version
fun pizza(block: Pizza.Builder.() -> Unit) = Pizza.Builder().apply(block).build()
```

---

## Structural Patterns

### Adapter

```kotlin
// Adapt old interface to new one
interface NewLogger {
    fun logInfo(msg: String)
    fun logError(msg: String, ex: Throwable? = null)
    fun logDebug(msg: String)
}

class LegacyLogger {
    fun log(level: String, message: String) {
        println("[$level] $message")
    }
}

// Adapter wraps legacy, implements new interface
class LegacyLoggerAdapter(private val legacy: LegacyLogger) : NewLogger {
    override fun logInfo(msg: String) = legacy.log("INFO", msg)
    override fun logError(msg: String, ex: Throwable?) {
        legacy.log("ERROR", msg + (ex?.let { " | ${it.message}" } ?: ""))
    }
    override fun logDebug(msg: String) = legacy.log("DEBUG", msg)
}

// Extension function as adapter (Kotlin style)
fun LegacyLogger.asNewLogger(): NewLogger = LegacyLoggerAdapter(this)
```

### Decorator

```kotlin
interface TextProcessor {
    fun process(text: String): String
}

class PlainTextProcessor : TextProcessor {
    override fun process(text: String) = text
}

// Decorators add behavior
class UpperCaseDecorator(private val wrapped: TextProcessor) : TextProcessor {
    override fun process(text: String) = wrapped.process(text).uppercase()
}

class TrimDecorator(private val wrapped: TextProcessor) : TextProcessor {
    override fun process(text: String) = wrapped.process(text).trim()
}

class PrefixDecorator(
    private val wrapped: TextProcessor,
    private val prefix: String
) : TextProcessor {
    override fun process(text: String) = "$prefix${wrapped.process(text)}"
}

class SuffixDecorator(
    private val wrapped: TextProcessor,
    private val suffix: String
) : TextProcessor {
    override fun process(text: String) = "${wrapped.process(text)}$suffix"
}

// DSL for decorating
fun TextProcessor.uppercase() = UpperCaseDecorator(this)
fun TextProcessor.trim() = TrimDecorator(this)
fun TextProcessor.prefix(p: String) = PrefixDecorator(this, p)
fun TextProcessor.suffix(s: String) = SuffixDecorator(this, s)
```

### Facade

```kotlin
// Facade: simple interface to complex subsystem
class VideoDecoder {
    fun decode(file: String) = "Decoded: $file"
}

class AudioDecoder {
    fun decode(file: String) = "Audio decoded: $file"
}

class VideoConverter {
    fun convert(data: String, format: String) = "Converted to $format"
}

class AudioMixer {
    fun mix(video: String, audio: String) = "Mixed: video+audio"
}

class FileWriter {
    fun write(data: String, output: String) = "Written to $output"
}

// Facade: hides complexity
class VideoConversionFacade {
    private val videoDecoder = VideoDecoder()
    private val audioDecoder = AudioDecoder()
    private val converter = VideoConverter()
    private val mixer = AudioMixer()
    private val writer = FileWriter()
    
    fun convertVideo(inputFile: String, outputFile: String, format: String): String {
        val video = videoDecoder.decode(inputFile)
        val audio = audioDecoder.decode(inputFile)
        val converted = converter.convert(video, format)
        val mixed = mixer.mix(converted, audio)
        return writer.write(mixed, outputFile)
    }
}
```

---

## Behavioral Patterns

### Observer / Event System

```kotlin
typealias EventHandler<T> = (T) -> Unit

class EventBus<T> {
    private val handlers = mutableListOf<EventHandler<T>>()
    
    fun subscribe(handler: EventHandler<T>) {
        handlers.add(handler)
    }
    
    fun unsubscribe(handler: EventHandler<T>) {
        handlers.remove(handler)
    }
    
    fun emit(event: T) {
        handlers.forEach { it(event) }
    }
}

// Typed event system
sealed class AppEvent {
    data class UserLogin(val userId: String, val timestamp: Long) : AppEvent()
    data class UserLogout(val userId: String) : AppEvent()
    data class OrderCreated(val orderId: String, val total: Double) : AppEvent()
    data class PaymentReceived(val orderId: String, val amount: Double) : AppEvent()
}

class EventSystem {
    private val buses = mutableMapOf<String, EventBus<*>>()
    
    @Suppress("UNCHECKED_CAST")
    fun <T : AppEvent> on(eventClass: kotlin.reflect.KClass<T>): EventBus<T> {
        return buses.getOrPut(eventClass.simpleName!!) { EventBus<T>() } as EventBus<T>
    }
    
    inline fun <reified T : AppEvent> on(): EventBus<T> = on(T::class)
    
    @Suppress("UNCHECKED_CAST")
    fun <T : AppEvent> emit(event: T) {
        val bus = buses[event::class.simpleName] as? EventBus<T>
        bus?.emit(event)
    }
}
```

### Strategy

```kotlin
// Strategy: algorithm family, encapsulated and interchangeable
interface SortStrategy<T : Comparable<T>> {
    fun sort(list: MutableList<T>)
    val name: String
}

class BubbleSort<T : Comparable<T>> : SortStrategy<T> {
    override val name = "Bubble Sort"
    
    override fun sort(list: MutableList<T>) {
        for (i in list.indices) {
            for (j in 0 until list.size - i - 1) {
                if (list[j] > list[j + 1]) {
                    val temp = list[j]
                    list[j] = list[j + 1]
                    list[j + 1] = temp
                }
            }
        }
    }
}

class QuickSort<T : Comparable<T>> : SortStrategy<T> {
    override val name = "Quick Sort"
    
    override fun sort(list: MutableList<T>) {
        quickSort(list, 0, list.size - 1)
    }
    
    private fun quickSort(list: MutableList<T>, low: Int, high: Int) {
        if (low < high) {
            val pivot = partition(list, low, high)
            quickSort(list, low, pivot - 1)
            quickSort(list, pivot + 1, high)
        }
    }
    
    private fun partition(list: MutableList<T>, low: Int, high: Int): Int {
        val pivot = list[high]
        var i = low - 1
        for (j in low until high) {
            if (list[j] <= pivot) {
                i++
                val temp = list[i]; list[i] = list[j]; list[j] = temp
            }
        }
        val temp = list[i + 1]; list[i + 1] = list[high]; list[high] = temp
        return i + 1
    }
}

class Sorter<T : Comparable<T>>(private var strategy: SortStrategy<T>) {
    fun setStrategy(strategy: SortStrategy<T>) { this.strategy = strategy }
    
    fun sort(list: MutableList<T>): MutableList<T> {
        val start = System.nanoTime()
        strategy.sort(list)
        val elapsed = System.nanoTime() - start
        println("${strategy.name}: ${elapsed / 1000}μs for ${list.size} items")
        return list
    }
}
```

### Command

```kotlin
// Command: encapsulate actions as objects
interface Command {
    fun execute()
    fun undo()
}

class TextEditor {
    private val text = StringBuilder()
    private val history = ArrayDeque<Command>()
    
    fun executeCommand(cmd: Command) {
        cmd.execute()
        history.addLast(cmd)
    }
    
    fun undo() {
        history.removeLastOrNull()?.undo()
    }
    
    fun getText() = text.toString()
    
    inner class InsertCommand(val pos: Int, val text: String) : Command {
        override fun execute() {
            this@TextEditor.text.insert(pos, text)
        }
        override fun undo() {
            this@TextEditor.text.delete(pos, pos + text.length)
        }
    }
    
    inner class DeleteCommand(val pos: Int, val length: Int) : Command {
        private lateinit var deletedText: String
        
        override fun execute() {
            deletedText = text.substring(pos, pos + length)
            text.delete(pos, pos + length)
        }
        override fun undo() {
            text.insert(pos, deletedText)
        }
    }
}
```

### Chain of Responsibility

```kotlin
// Chain of Responsibility: pass request along a chain
abstract class Handler<T, R> {
    protected var next: Handler<T, R>? = null
    
    fun setNext(handler: Handler<T, R>): Handler<T, R> {
        next = handler
        return handler
    }
    
    abstract fun handle(request: T): R?
    
    protected fun passToNext(request: T): R? = next?.handle(request)
}

// Middleware chain
data class HttpRequest(
    val path: String,
    val method: String,
    val headers: Map<String, String> = emptyMap(),
    val body: String? = null
)

data class HttpResponse(val status: Int, val body: String)

abstract class Middleware : Handler<HttpRequest, HttpResponse>()

class AuthMiddleware(private val validToken: String) : Middleware() {
    override fun handle(request: HttpRequest): HttpResponse? {
        val token = request.headers["Authorization"]?.removePrefix("Bearer ")
        return if (token == validToken) {
            println("[Auth] Authorized")
            passToNext(request)
        } else {
            println("[Auth] Unauthorized")
            HttpResponse(401, "Unauthorized")
        }
    }
}

class RateLimitMiddleware(private val maxRpm: Int) : Middleware() {
    private val requestTimes = mutableListOf<Long>()
    
    override fun handle(request: HttpRequest): HttpResponse? {
        val now = System.currentTimeMillis()
        requestTimes.removeAll { it < now - 60000 }
        
        return if (requestTimes.size < maxRpm) {
            requestTimes.add(now)
            println("[RateLimit] OK (${requestTimes.size}/$maxRpm rpm)")
            passToNext(request)
        } else {
            println("[RateLimit] Too many requests")
            HttpResponse(429, "Too Many Requests")
        }
    }
}

class LoggingMiddleware : Middleware() {
    override fun handle(request: HttpRequest): HttpResponse? {
        println("[Log] ${request.method} ${request.path}")
        val response = passToNext(request)
        println("[Log] Response: ${response?.status}")
        return response
    }
}

class RouterMiddleware(private val routes: Map<String, (HttpRequest) -> HttpResponse>) : Middleware() {
    override fun handle(request: HttpRequest): HttpResponse? {
        val handler = routes["${request.method}:${request.path}"]
        return handler?.invoke(request) ?: HttpResponse(404, "Not Found")
    }
}
```

---

## Kotlin-specific Patterns

```kotlin
// Kotlin-idiomatic patterns

// 1. Extension functions as decorator
fun String.validated(minLen: Int = 1, maxLen: Int = Int.MAX_VALUE): String {
    require(length >= minLen) { "Too short (min $minLen)" }
    require(length <= maxLen) { "Too long (max $maxLen)" }
    return this
}

fun String.trimmedOrNull() = trim().takeIf { it.isNotEmpty() }

// 2. Sealed class as Result/Either
sealed class Outcome<out S, out F> {
    data class Success<S>(val value: S) : Outcome<S, Nothing>()
    data class Failure<F>(val error: F) : Outcome<Nothing, F>()
    
    inline fun <T> fold(
        onSuccess: (S) -> T,
        onFailure: (F) -> T
    ): T = when (this) {
        is Success -> onSuccess(value)
        is Failure -> onFailure(error)
    }
    
    fun <T> map(transform: (S) -> T): Outcome<T, F> = when (this) {
        is Success -> Success(transform(value))
        is Failure -> this
    }
}

// 3. Scope functions as patterns
data class UserConfig(var theme: String = "light", var fontSize: Int = 14) {
    companion object {
        fun create(block: UserConfig.() -> Unit) = UserConfig().apply(block)
    }
}

// 4. Typesafe heterogeneous containers
class TypedMap {
    private val map = HashMap<Class<*>, Any>()
    
    @Suppress("UNCHECKED_CAST")
    fun <T : Any> get(clazz: Class<T>): T? = map[clazz] as T?
    
    fun <T : Any> put(value: T) { map[value.javaClass] = value }
    
    inline fun <reified T : Any> get() = get(T::class.java)
    inline fun <reified T : Any> put(value: T) = put(value as Any)
}

fun main() {
    // Pizza builder
    val pizza1 = Pizza.Builder()
        .size("large")
        .crust("thick")
        .toppings("pepperoni", "mushrooms", "olives")
        .extraCheese()
        .build()
    println(pizza1)
    
    val pizza2 = pizza {
        size("medium")
        sauce("pesto")
        topping("chicken")
        topping("sun-dried tomatoes")
    }
    println(pizza2)
    
    println()
    
    // Sorter with strategy
    val sorter = Sorter(BubbleSort<Int>())
    sorter.sort(mutableListOf(5, 3, 8, 1, 9, 2))
    
    sorter.setStrategy(QuickSort())
    sorter.sort(mutableListOf(5, 3, 8, 1, 9, 2))
    
    println()
    
    // Text editor with command pattern
    val editor = TextEditor()
    editor.executeCommand(editor.InsertCommand(0, "Hello"))
    editor.executeCommand(editor.InsertCommand(5, " World"))
    println("Text: ${editor.getText()}")
    editor.undo()
    println("After undo: ${editor.getText()}")
    editor.undo()
    println("After undo again: ${editor.getText()}")
    
    println()
    
    // Middleware chain
    val router = RouterMiddleware(mapOf(
        "GET:/users" to { _ -> HttpResponse(200, "Users list") },
        "POST:/users" to { req -> HttpResponse(201, "Created") }
    ))
    
    val logging = LoggingMiddleware()
    val rateLimit = RateLimitMiddleware(100)
    val auth = AuthMiddleware("valid-token-123")
    
    logging.setNext(auth).setNext(rateLimit).setNext(router)
    
    val req1 = HttpRequest("/users", "GET", mapOf("Authorization" to "Bearer valid-token-123"))
    val resp1 = logging.handle(req1)
    println("Response: ${resp1?.status} ${resp1?.body}")
    
    println()
    
    val req2 = HttpRequest("/users", "GET", mapOf("Authorization" to "Bearer invalid"))
    val resp2 = logging.handle(req2)
    println("Response: ${resp2?.status} ${resp2?.body}")
    
    println()
    
    // Event system
    val events = EventSystem()
    
    events.on<AppEvent.UserLogin>().subscribe { event ->
        println("User logged in: ${event.userId}")
    }
    
    events.on<AppEvent.OrderCreated>().subscribe { event ->
        println("Order created: #${event.orderId} for ฿${event.total}")
    }
    
    events.emit(AppEvent.UserLogin("user-123", System.currentTimeMillis()))
    events.emit(AppEvent.OrderCreated("ORD-456", 1500.0))
    events.emit(AppEvent.PaymentReceived("ORD-456", 1500.0))  // no handler
    
    println()
    
    // TypedMap
    val container = TypedMap()
    container.put("Hello String")
    container.put(42)
    container.put(3.14)
    
    println(container.get<String>())
    println(container.get<Int>())
    println(container.get<Double>())
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Plugin System using Abstract Factory + Observer

interface Plugin {
    val name: String
    val version: String
    fun onLoad()
    fun onUnload()
}

sealed class PluginEvent {
    data class Loaded(val plugin: Plugin) : PluginEvent()
    data class Unloaded(val plugin: Plugin) : PluginEvent()
    data class Error(val plugin: Plugin, val error: Throwable) : PluginEvent()
}

class PluginManager {
    private val plugins = mutableMapOf<String, Plugin>()
    private val eventHandlers = mutableListOf<(PluginEvent) -> Unit>()
    
    fun onEvent(handler: (PluginEvent) -> Unit) {
        eventHandlers.add(handler)
    }
    
    private fun emit(event: PluginEvent) {
        eventHandlers.forEach { it(event) }
    }
    
    fun load(plugin: Plugin): Boolean {
        return try {
            plugin.onLoad()
            plugins[plugin.name] = plugin
            emit(PluginEvent.Loaded(plugin))
            true
        } catch (e: Exception) {
            emit(PluginEvent.Error(plugin, e))
            false
        }
    }
    
    fun unload(name: String): Boolean {
        val plugin = plugins.remove(name) ?: return false
        plugin.onUnload()
        emit(PluginEvent.Unloaded(plugin))
        return true
    }
    
    fun getPlugin(name: String): Plugin? = plugins[name]
    fun listPlugins() = plugins.values.toList()
}

// Sample plugins
class LoggingPlugin : Plugin {
    override val name = "logging"
    override val version = "1.0.0"
    override fun onLoad() = println("Logging plugin loaded")
    override fun onUnload() = println("Logging plugin unloaded")
}

class MetricsPlugin : Plugin {
    override val name = "metrics"
    override val version = "2.1.0"
    override fun onLoad() = println("Metrics plugin loaded")
    override fun onUnload() = println("Metrics plugin unloaded")
}

fun main() {
    val manager = PluginManager()
    
    manager.onEvent { event ->
        when (event) {
            is PluginEvent.Loaded   -> println("✅ Plugin loaded: ${event.plugin.name} v${event.plugin.version}")
            is PluginEvent.Unloaded -> println("🔴 Plugin unloaded: ${event.plugin.name}")
            is PluginEvent.Error    -> println("❌ Plugin error: ${event.plugin.name}: ${event.error.message}")
        }
    }
    
    manager.load(LoggingPlugin())
    manager.load(MetricsPlugin())
    
    println("\nLoaded plugins:")
    manager.listPlugins().forEach { println("  - ${it.name} v${it.version}") }
    
    manager.unload("logging")
    
    println("\nLoaded plugins after unload:")
    manager.listPlugins().forEach { println("  - ${it.name} v${it.version}") }
}
```

---

## สรุป Part 27

```
✅ Singleton: object declaration ใน Kotlin
✅ Factory Method: สร้าง objects โดยไม่ระบุ exact class
✅ Abstract Factory: สร้าง family of related objects
✅ Builder: สร้าง complex objects step by step
✅ Adapter: แปลง interface เก่าเป็น interface ใหม่
✅ Decorator: เพิ่ม behavior โดยไม่แก้ original class
✅ Facade: ซ่อน complexity ของ subsystem
✅ Observer/EventBus: notify หลาย objects เมื่อ event เกิดขึ้น
✅ Strategy: กลุ่ม algorithms ที่สลับกันได้
✅ Command: encapsulate actions เป็น objects (รองรับ undo)
✅ Chain of Responsibility: ส่ง request ตามลำดับ handlers
✅ Kotlin-idiomatic patterns ใช้ extension functions, sealed classes
```

---

*Part 27/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
