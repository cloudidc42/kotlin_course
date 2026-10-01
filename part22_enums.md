# Part 22: Enums และ Sealed Classes

## สารบัญ
1. [Enum Class พื้นฐาน](#enum-class-พื้นฐาน)
2. [Enum กับ Properties และ Methods](#enum-กับ-properties-และ-methods)
3. [Enum กับ Abstract Methods](#enum-กับ-abstract-methods)
4. [Enum Utilities](#enum-utilities)
5. [Sealed Class แบบ Advanced](#sealed-class-แบบ-advanced)
6. [Enum vs Sealed Class](#enum-vs-sealed-class)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Enum Class พื้นฐาน

```kotlin
// Enum คือชุดของค่าคงที่ที่มีจำนวนจำกัด
enum class Direction {
    NORTH, SOUTH, EAST, WEST
}

enum class DayOfWeek {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}

enum class HttpMethod {
    GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS
}

fun main() {
    val dir = Direction.NORTH
    println(dir)           // NORTH
    println(dir.name)      // NORTH (String)
    println(dir.ordinal)   // 0 (index)
    
    // เปรียบเทียบ
    println(dir == Direction.NORTH)   // true
    println(dir == Direction.SOUTH)   // false
    
    // when expression
    val message = when (dir) {
        Direction.NORTH -> "ไปทางเหนือ"
        Direction.SOUTH -> "ไปทางใต้"
        Direction.EAST  -> "ไปทางตะวันออก"
        Direction.WEST  -> "ไปทางตะวันตก"
    }
    println(message)
    
    // values() - ดึง array ทั้งหมด
    println("All directions:")
    Direction.values().forEach { println("  ${it.ordinal}: ${it.name}") }
    
    // entries - modern way (Kotlin 1.9+)
    Direction.entries.forEach { println(it) }
    
    // valueOf - string to enum
    val e = Direction.valueOf("EAST")
    println(e)  // EAST
    
    // enumValueOf<T>() - generic version
    val w = enumValueOf<Direction>("WEST")
    println(w)  // WEST
    
    // enumValues<T>() - generic array
    enumValues<Direction>().forEach { println(it) }
    
    // แปลงจาก String อย่างปลอดภัย
    fun String.toDirectionOrNull(): Direction? =
        Direction.entries.firstOrNull { it.name == this.uppercase() }
    
    println("north".toDirectionOrNull())   // NORTH
    println("invalid".toDirectionOrNull()) // null
}
```

---

## Enum กับ Properties และ Methods

```kotlin
enum class Planet(val massKg: Double, val radiusM: Double) {
    MERCURY(3.303e+23, 2.4397e6),
    VENUS(4.869e+24, 6.0518e6),
    EARTH(5.976e+24, 6.37814e6),
    MARS(6.421e+23, 3.3972e6),
    JUPITER(1.9e+27, 7.1492e7),
    SATURN(5.688e+26, 6.0268e7),
    URANUS(8.686e+25, 2.5559e7),
    NEPTUNE(1.024e+26, 2.4746e7);
    
    companion object {
        const val G = 6.67300E-11  // gravitational constant
    }
    
    // Computed property
    val surfaceGravity: Double
        get() = G * massKg / (radiusM * radiusM)
    
    // Method
    fun surfaceWeight(otherMass: Double): Double = otherMass * surfaceGravity
}

enum class Status(val code: Int, val description: String) {
    PENDING(1, "รอดำเนินการ"),
    PROCESSING(2, "กำลังดำเนินการ"),
    COMPLETED(3, "เสร็จสิ้น"),
    FAILED(4, "ล้มเหลว"),
    CANCELLED(5, "ยกเลิก");
    
    val isTerminal: Boolean
        get() = this == COMPLETED || this == FAILED || this == CANCELLED
    
    val isActive: Boolean
        get() = this == PENDING || this == PROCESSING
    
    fun canTransitionTo(next: Status): Boolean = when (this) {
        PENDING     -> next == PROCESSING || next == CANCELLED
        PROCESSING  -> next == COMPLETED || next == FAILED || next == CANCELLED
        else        -> false
    }
}

enum class Color(val r: Int, val g: Int, val b: Int) {
    RED(255, 0, 0),
    GREEN(0, 255, 0),
    BLUE(0, 0, 255),
    WHITE(255, 255, 255),
    BLACK(0, 0, 0),
    YELLOW(255, 255, 0),
    CYAN(0, 255, 255);
    
    val hex: String
        get() = "#%02X%02X%02X".format(r, g, b)
    
    val brightness: Int
        get() = (r + g + b) / 3
    
    fun mix(other: Color): Color? {
        val mixR = (this.r + other.r) / 2
        val mixG = (this.g + other.g) / 2
        val mixB = (this.b + other.b) / 2
        return entries.firstOrNull { it.r == mixR && it.g == mixG && it.b == mixB }
    }
    
    companion object {
        fun fromHex(hex: String): Color? {
            val clean = hex.removePrefix("#")
            val r = clean.substring(0, 2).toIntOrNull(16) ?: return null
            val g = clean.substring(2, 4).toIntOrNull(16) ?: return null
            val b = clean.substring(4, 6).toIntOrNull(16) ?: return null
            return entries.firstOrNull { it.r == r && it.g == g && it.b == b }
        }
    }
}

fun main() {
    // Planet weight calculator
    val earthWeight = 75.0
    val mass = earthWeight / Planet.EARTH.surfaceGravity
    
    Planet.entries.forEach { planet ->
        val weight = planet.surfaceWeight(mass)
        println("Weight on ${planet.name}: ${"%.2f".format(weight)} N")
    }
    
    println()
    
    // Status transitions
    var status = Status.PENDING
    println("Current: ${status.name} - ${status.description}")
    println("Can go to PROCESSING: ${status.canTransitionTo(Status.PROCESSING)}")
    println("Can go to COMPLETED: ${status.canTransitionTo(Status.COMPLETED)}")
    
    if (status.canTransitionTo(Status.PROCESSING)) {
        status = Status.PROCESSING
    }
    println("New status: ${status.name}")
    println("Is active: ${status.isActive}")
    println("Is terminal: ${status.isTerminal}")
    
    println()
    
    // Color
    println("RED hex: ${Color.RED.hex}")
    println("BLUE hex: ${Color.BLUE.hex}")
    println("Brightness of WHITE: ${Color.WHITE.brightness}")
    
    Color.fromHex("#FF0000")?.let { println("From hex: $it") }
}
```

---

## Enum กับ Abstract Methods

```kotlin
enum class Operation(val symbol: String) {
    PLUS("+") {
        override fun apply(x: Double, y: Double) = x + y
    },
    MINUS("-") {
        override fun apply(x: Double, y: Double) = x - y
    },
    TIMES("*") {
        override fun apply(x: Double, y: Double) = x * y
    },
    DIVIDE("/") {
        override fun apply(x: Double, y: Double) {
            require(y != 0.0) { "Cannot divide by zero" }
            return x / y
        }
    };
    
    abstract fun apply(x: Double, y: Double): Double
    
    override fun toString() = symbol
}

enum class Shape {
    CIRCLE {
        override fun area(params: DoubleArray): Double {
            val (r) = params
            return Math.PI * r * r
        }
        override fun perimeter(params: DoubleArray): Double {
            val (r) = params
            return 2 * Math.PI * r
        }
        override val paramNames = listOf("radius")
    },
    RECTANGLE {
        override fun area(params: DoubleArray): Double {
            val (w, h) = params
            return w * h
        }
        override fun perimeter(params: DoubleArray): Double {
            val (w, h) = params
            return 2 * (w + h)
        }
        override val paramNames = listOf("width", "height")
    },
    TRIANGLE {
        override fun area(params: DoubleArray): Double {
            val (base, height) = params
            return 0.5 * base * height
        }
        override fun perimeter(params: DoubleArray): Double {
            val (a, b, c) = params
            return a + b + c
        }
        override val paramNames = listOf("base", "height")
    };
    
    abstract fun area(params: DoubleArray): Double
    abstract fun perimeter(params: DoubleArray): Double
    abstract val paramNames: List<String>
}

fun main() {
    // Arithmetic operations
    val x = 10.0
    val y = 3.0
    
    Operation.entries.forEach { op ->
        if (op == Operation.DIVIDE && y == 0.0) return@forEach
        println("${x} ${op.symbol} ${y} = ${op.apply(x, y)}")
    }
    
    println()
    
    // Parse and evaluate
    fun evaluate(expr: String): Double {
        val tokens = expr.trim().split("\\s+".toRegex())
        val a = tokens[0].toDouble()
        val op = Operation.entries.first { it.symbol == tokens[1] }
        val b = tokens[2].toDouble()
        return op.apply(a, b)
    }
    
    println("10 + 5 = ${evaluate("10 + 5")}")
    println("20 - 8 = ${evaluate("20 - 8")}")
    println("6 * 7 = ${evaluate("6 * 7")}")
    println("15 / 4 = ${evaluate("15 / 4")}")
    
    println()
    
    // Shapes
    println("Circle (r=5): area=${Shape.CIRCLE.area(doubleArrayOf(5.0)).format()}")
    println("Rectangle (4x6): area=${Shape.RECTANGLE.area(doubleArrayOf(4.0, 6.0)).format()}")
    println("Triangle (b=6, h=4): area=${Shape.TRIANGLE.area(doubleArrayOf(6.0, 4.0)).format()}")
}

fun Double.format(decimals: Int = 2) = "%.${decimals}f".format(this)
```

---

## Sealed Class แบบ Advanced

```kotlin
// Sealed class: รู้จักทุก subclass ณ compile time
sealed class NetworkResult<out T> {
    data class Success<T>(val data: T, val statusCode: Int = 200) : NetworkResult<T>()
    data class Error(val code: Int, val message: String) : NetworkResult<Nothing>()
    data class Loading(val progress: Int = 0) : NetworkResult<Nothing>()
    object Empty : NetworkResult<Nothing>()
    
    val isSuccess: Boolean get() = this is Success
    val isError: Boolean get() = this is Error
    
    fun getOrNull(): T? = (this as? Success)?.data
    
    fun <R> map(transform: (T) -> R): NetworkResult<R> = when (this) {
        is Success -> Success(transform(data), statusCode)
        is Error   -> this
        is Loading -> this
        is Empty   -> this
    }
}

sealed class Event {
    data class Click(val x: Int, val y: Int) : Event()
    data class KeyPress(val key: String, val modifiers: Set<String> = emptySet()) : Event()
    data class Scroll(val delta: Int, val direction: ScrollDirection) : Event()
    object AppStart : Event()
    object AppPause : Event()
    object AppStop : Event()
    
    enum class ScrollDirection { UP, DOWN, LEFT, RIGHT }
}

sealed class UiState<out T> {
    object Idle : UiState<Nothing>()
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String, val retry: Boolean = true) : UiState<Nothing>()
    
    fun <R> map(f: (T) -> R): UiState<R> = when (this) {
        is Idle    -> Idle
        is Loading -> Loading
        is Success -> Success(f(data))
        is Error   -> this
    }
}

// Sealed hierarchy (nested sealed)
sealed class AuthState {
    object Unauthenticated : AuthState()
    object Authenticating : AuthState()
    
    sealed class Authenticated : AuthState() {
        abstract val userId: String
        abstract val email: String
        
        data class Regular(
            override val userId: String,
            override val email: String,
            val name: String
        ) : Authenticated()
        
        data class Admin(
            override val userId: String,
            override val email: String,
            val permissions: Set<String>
        ) : Authenticated()
    }
    
    data class Error(val reason: String) : AuthState()
    
    val isAuthenticated: Boolean
        get() = this is Authenticated
    
    fun currentUserId(): String? = (this as? Authenticated)?.userId
}

fun handleEvent(event: Event) = when (event) {
    is Event.Click     -> "Clicked at (${event.x}, ${event.y})"
    is Event.KeyPress  -> "Key: ${event.key}" + if (event.modifiers.isNotEmpty()) 
                             " [${event.modifiers.joinToString("+")}]" else ""
    is Event.Scroll    -> "Scrolled ${event.direction} by ${event.delta}"
    Event.AppStart     -> "App started"
    Event.AppPause     -> "App paused"
    Event.AppStop      -> "App stopped"
}

fun main() {
    // NetworkResult
    val results: List<NetworkResult<String>> = listOf(
        NetworkResult.Success("Hello, World!", 200),
        NetworkResult.Error(404, "Not Found"),
        NetworkResult.Loading(50),
        NetworkResult.Empty
    )
    
    results.forEach { result ->
        val message = when (result) {
            is NetworkResult.Success -> "✅ ${result.data} (${result.statusCode})"
            is NetworkResult.Error   -> "❌ ${result.code}: ${result.message}"
            is NetworkResult.Loading -> "⏳ Loading ${result.progress}%"
            NetworkResult.Empty      -> "📭 No data"
        }
        println(message)
    }
    
    println()
    
    // map transformation
    val numbers = NetworkResult.Success(listOf(1, 2, 3))
    val doubled = numbers.map { it.map { n -> n * 2 } }
    println(doubled)  // Success(data=[2, 4, 6])
    
    // Events
    val events = listOf(
        Event.Click(100, 200),
        Event.KeyPress("Ctrl", setOf("Ctrl", "Alt")),
        Event.Scroll(3, Event.ScrollDirection.DOWN),
        Event.AppStart
    )
    
    events.forEach { println(handleEvent(it)) }
    
    println()
    
    // AuthState
    val states: List<AuthState> = listOf(
        AuthState.Unauthenticated,
        AuthState.Authenticating,
        AuthState.Authenticated.Regular("user-1", "user@email.com", "สมชาย"),
        AuthState.Authenticated.Admin("admin-1", "admin@email.com", setOf("READ", "WRITE", "DELETE")),
        AuthState.Error("Token expired")
    )
    
    states.forEach { state ->
        val info = when (state) {
            AuthState.Unauthenticated -> "Not logged in"
            AuthState.Authenticating  -> "Logging in..."
            is AuthState.Authenticated.Regular -> "User: ${state.name} (${state.email})"
            is AuthState.Authenticated.Admin   -> "Admin: ${state.email} permissions: ${state.permissions}"
            is AuthState.Error -> "Auth error: ${state.reason}"
        }
        println(info)
    }
}
```

---

## Enum vs Sealed Class

```kotlin
/*
 * Enum: ใช้เมื่อ
 * - ต้องการค่า constant จำนวนจำกัดที่ไม่ต้องการ state ต่างกัน
 * - ต้องการ ordinal/name/values() built-in
 * - ต้องการใช้ใน @Annotation
 * - ต้องการ when exhaustive ง่ายๆ
 *
 * Sealed: ใช้เมื่อ
 * - แต่ละ variant มี data ต่างกัน
 * - ต้องการ subclass hierarchy
 * - ต้องการ generic type parameters
 * - ต้องการความยืดหยุ่นสูงกว่า
 */

// Enum เหมาะกว่า
enum class Weekday { MON, TUE, WED, THU, FRI, SAT, SUN }

// Sealed เหมาะกว่า (แต่ละ state มี data ต่างกัน)
sealed class LoadingState {
    object Idle : LoadingState()
    data class Loading(val message: String, val progress: Int) : LoadingState()
    data class Success<T>(val result: T) : LoadingState()
    data class Error(val code: Int, val message: String, val retryable: Boolean) : LoadingState()
}

// Enum กับ interface
interface Printable {
    fun print(): String
}

enum class LogLevel(val priority: Int) : Printable {
    VERBOSE(0),
    DEBUG(1),
    INFO(2),
    WARN(3),
    ERROR(4),
    FATAL(5);
    
    override fun print() = "[${name.padStart(7)}]"
    
    fun isAtLeast(level: LogLevel) = this.priority >= level.priority
}

fun main() {
    LogLevel.entries.forEach { level ->
        if (level.isAtLeast(LogLevel.INFO)) {
            println("${level.print()} This level is INFO or above")
        }
    }
    
    // Enum in annotation
    // (annotations require enum, not sealed)
    println(LogLevel.valueOf("WARN"))
    println(LogLevel.entries.sortedByDescending { it.priority })
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง State Machine สำหรับ Order

enum class OrderStatus {
    CREATED, PAYMENT_PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED, REFUNDED;
    
    fun allowedTransitions(): Set<OrderStatus> = when (this) {
        CREATED          -> setOf(PAYMENT_PENDING, CANCELLED)
        PAYMENT_PENDING  -> setOf(PAID, CANCELLED)
        PAID             -> setOf(PROCESSING, REFUNDED)
        PROCESSING       -> setOf(SHIPPED)
        SHIPPED          -> setOf(DELIVERED)
        DELIVERED        -> setOf(REFUNDED)
        CANCELLED        -> emptySet()
        REFUNDED         -> emptySet()
    }
    
    fun canTransitionTo(next: OrderStatus): Boolean = next in allowedTransitions()
    
    val isTerminal: Boolean
        get() = this == CANCELLED || this == REFUNDED || this == DELIVERED
    
    val displayName: String
        get() = name.replace("_", " ").lowercase()
            .replaceFirstChar { it.uppercase() }
}

data class Order(
    val id: String,
    var status: OrderStatus = OrderStatus.CREATED,
    val history: MutableList<Pair<OrderStatus, String>> = mutableListOf()
) {
    fun transition(newStatus: OrderStatus, reason: String = ""): Boolean {
        if (!status.canTransitionTo(newStatus)) {
            println("❌ Cannot transition from $status to $newStatus")
            return false
        }
        history.add(status to reason)
        status = newStatus
        println("✅ Order $id: ${history.last().first} → $status")
        return true
    }
    
    fun printHistory() {
        println("Order $id history:")
        history.forEachIndexed { i, (s, reason) ->
            val note = if (reason.isNotEmpty()) " ($reason)" else ""
            println("  ${i+1}. ${s.displayName}$note")
        }
        println("  Current: ${status.displayName}")
    }
}

fun main() {
    val order = Order("ORD-001")
    
    order.transition(OrderStatus.PAYMENT_PENDING)
    order.transition(OrderStatus.PAID)
    order.transition(OrderStatus.PROCESSING)
    order.transition(OrderStatus.SHIPPED)
    order.transition(OrderStatus.DELIVERED)
    
    // Invalid transition
    order.transition(OrderStatus.PROCESSING)  // ❌ already delivered
    
    println()
    order.printHistory()
    
    println("\nTerminal: ${order.status.isTerminal}")
    
    println("\n--- Another order (cancelled) ---")
    val order2 = Order("ORD-002")
    order2.transition(OrderStatus.PAYMENT_PENDING)
    order2.transition(OrderStatus.CANCELLED, "Customer request")
    order2.transition(OrderStatus.PAID)  // ❌ already cancelled
    order2.printHistory()
}
```

---

## สรุป Part 22

```
✅ enum class คือชุด constants ที่มี name/ordinal built-in
✅ values() และ entries (Kotlin 1.9+) ดึงทุก enum values
✅ valueOf("NAME") แปลง String เป็น enum
✅ Enum กับ constructor parameters: สร้าง properties ใน enum
✅ Enum กับ abstract fun: แต่ละ value implement ต่างกัน
✅ Enum implements interface ได้
✅ Sealed class: hierarchy ที่รู้จักทุก subclass ณ compile time
✅ Sealed subclass มี data ต่างกันได้ (แตกต่างจาก enum)
✅ when กับ sealed เป็น exhaustive โดยอัตโนมัติ
✅ Nested sealed: sealed ซ้อนใน sealed ได้
✅ Enum ใช้เมื่อต้องการ constant ง่ายๆ
✅ Sealed ใช้เมื่อ variant มี data หรือ type parameter ต่างกัน
```

---

*Part 22/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
