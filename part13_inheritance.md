# Part 13: Inheritance และ Polymorphism

## สารบัญ
1. [Inheritance พื้นฐาน](#inheritance-พื้นฐาน)
2. [Abstract Classes](#abstract-classes)
3. [Interfaces](#interfaces)
4. [Polymorphism](#polymorphism)
5. [Sealed Classes](#sealed-classes)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Inheritance พื้นฐาน

ใน Kotlin ทุก class เป็น `final` โดย default ต้องใช้ `open` เพื่ออนุญาตให้ inherit

```kotlin
// Base class ต้องมี open
open class Animal(val name: String, val sound: String) {
    
    open fun makeSound() {
        println("$name says: $sound")
    }
    
    open fun describe() {
        println("I am $name")
    }
    
    // final method - ไม่สามารถ override ได้
    fun breathe() {
        println("$name is breathing")
    }
}

// Derived class
class Dog(name: String) : Animal(name, "Woof") {
    
    override fun makeSound() {
        println("$name barks loudly: $sound! $sound!")
    }
    
    fun fetch() {
        println("$name fetches the ball!")
    }
}

class Cat(name: String) : Animal(name, "Meow") {
    override fun describe() {
        super.describe()  // เรียก parent method
        println("...and I am a cat who ignores everyone")
    }
}

fun main() {
    val dog = Dog("Rex")
    val cat = Cat("Whiskers")
    
    dog.makeSound()   // Rex barks loudly: Woof! Woof!
    dog.fetch()       // Rex fetches the ball!
    dog.breathe()     // Rex is breathing
    
    cat.makeSound()   // Whiskers says: Meow
    cat.describe()    // I am Whiskers \n ...and I am a cat...
    
    // Polymorphism
    val animals: List<Animal> = listOf(dog, cat, Animal("Parrot", "Squawk"))
    animals.forEach { it.makeSound() }
    
    // Type checking
    println(dog is Animal)  // true
    println(dog is Dog)     // true
    println(cat is Dog)     // false
    
    // Smart cast
    val animal: Animal = Dog("Buddy")
    if (animal is Dog) {
        animal.fetch()  // Smart cast: animal is automatically Dog here
    }
}
```

### Overriding Rules

```kotlin
open class Shape {
    open val name: String = "Shape"
    open val area: Double get() = 0.0
    
    open fun draw() = println("Drawing $name")
    
    // Preventing further override
    open fun color() = "transparent"
}

open class Circle(val radius: Double) : Shape() {
    override val name = "Circle"
    override val area get() = Math.PI * radius * radius
    
    override fun draw() {
        super.draw()
        println("(with radius $radius)")
    }
    
    // final: subclasses of Circle cannot override color()
    final override fun color() = "red"
}

class FilledCircle(radius: Double, val fillColor: String) : Circle(radius) {
    override val name = "FilledCircle"
    
    // override fun color() = fillColor  // ❌ Error: color() is final
    
    override fun draw() {
        super.draw()
        println("filled with $fillColor")
    }
}

fun main() {
    val c = Circle(5.0)
    val fc = FilledCircle(3.0, "blue")
    
    c.draw()
    println("Area: ${"%.2f".format(c.area)}")
    
    fc.draw()
    println("Color: ${fc.color()}")  // red (from Circle)
}
```

---

## Abstract Classes

```kotlin
// abstract class: ไม่สามารถสร้าง instance โดยตรง
abstract class Vehicle(val brand: String, val model: String) {
    
    // Abstract properties (ต้อง override)
    abstract val fuelType: String
    abstract val maxSpeed: Int
    
    // Concrete property
    var currentSpeed: Int = 0
    
    // Abstract method (ต้อง override)
    abstract fun accelerate(amount: Int)
    abstract fun brake(amount: Int)
    
    // Concrete method
    fun describe() {
        println("$brand $model | Fuel: $fuelType | Max: $maxSpeed km/h | Current: $currentSpeed km/h")
    }
    
    open fun honk() = println("Beep beep!")
    
    // Template method pattern
    fun drive(distance: Int) {
        println("=== Driving $brand $model for $distance km ===")
        accelerate(60)
        println("Cruising at $currentSpeed km/h")
        repeat(distance / 50) {
            accelerate(10)
        }
        brake(currentSpeed)
        println("Arrived! Final speed: $currentSpeed km/h")
    }
}

class ElectricCar(brand: String, model: String, val batteryKwh: Double) 
    : Vehicle(brand, model) {
    
    override val fuelType = "Electric"
    override val maxSpeed = 250
    
    override fun accelerate(amount: Int) {
        currentSpeed = (currentSpeed + amount).coerceAtMost(maxSpeed)
        println("Electric motor: $currentSpeed km/h")
    }
    
    override fun brake(amount: Int) {
        currentSpeed = (currentSpeed - amount).coerceAtLeast(0)
        println("Regenerative braking: $currentSpeed km/h")
    }
}

class GasCar(brand: String, model: String, val engineCC: Int)
    : Vehicle(brand, model) {
    
    override val fuelType = "Gasoline"
    override val maxSpeed = 200
    
    override fun accelerate(amount: Int) {
        currentSpeed = (currentSpeed + amount).coerceAtMost(maxSpeed)
        println("Engine revving: $currentSpeed km/h")
    }
    
    override fun brake(amount: Int) {
        currentSpeed = (currentSpeed - amount).coerceAtLeast(0)
        println("Hydraulic brakes: $currentSpeed km/h")
    }
    
    override fun honk() = println("VROOM VROOM!")
}

fun main() {
    val tesla = ElectricCar("Tesla", "Model 3", 75.0)
    val bmw = GasCar("BMW", "320i", 2000)
    
    tesla.describe()
    bmw.describe()
    
    println("\n--- Tesla Drive ---")
    tesla.drive(100)
    
    println("\n--- BMW Drive ---")
    bmw.drive(100)
    
    // val v = Vehicle("Unknown", "Unknown")  // ❌ Error: Abstract class
}
```

---

## Interfaces

```kotlin
interface Flyable {
    val maxAltitude: Int  // abstract property
    
    fun fly() {
        println("Flying at max altitude $maxAltitude meters")
    }
    
    fun land() = println("Landing...")  // default implementation
}

interface Swimmable {
    val maxDepth: Int
    
    fun swim() {
        println("Swimming at max depth $maxDepth meters")
    }
    
    fun dive() = println("Diving...")
}

interface Describable {
    fun describe(): String
}

// Multiple interface implementation
class Duck(val name: String) : Flyable, Swimmable, Describable {
    override val maxAltitude = 1000
    override val maxDepth = 5
    
    override fun fly() {
        println("$name flaps wings and flies!")
    }
    
    override fun describe() = "$name is a duck that can fly and swim"
}

// Interface with state (via properties)
interface Sensor {
    var isActive: Boolean
    val type: String
    
    fun activate() { isActive = true; println("$type activated") }
    fun deactivate() { isActive = false; println("$type deactivated") }
    fun readValue(): Double
}

class TemperatureSensor : Sensor {
    override var isActive = false
    override val type = "Temperature"
    
    override fun readValue(): Double {
        if (!isActive) throw IllegalStateException("Sensor not active")
        return 23.5 + Math.random() * 5  // simulate reading
    }
}

// Interface inheritance
interface Shape {
    val area: Double
    val perimeter: Double
}

interface Drawable : Shape {
    fun draw()
    fun getColor(): String = "black"
}

class Circle(val radius: Double) : Drawable {
    override val area = Math.PI * radius * radius
    override val perimeter = 2 * Math.PI * radius
    
    override fun draw() = println("Drawing circle with radius $radius")
    override fun getColor() = "red"
}

fun main() {
    val duck = Duck("Donald")
    duck.fly()
    duck.swim()
    duck.land()
    duck.dive()
    println(duck.describe())
    
    // Type checking with interfaces
    println(duck is Flyable)   // true
    println(duck is Swimmable) // true
    
    val sensor = TemperatureSensor()
    sensor.activate()
    println("Temperature: ${"%.1f".format(sensor.readValue())}°C")
    sensor.deactivate()
    
    // Function accepting interface type
    fun printShapeInfo(shape: Drawable) {
        println("Area: ${"%.2f".format(shape.area)}")
        println("Color: ${shape.getColor()}")
        shape.draw()
    }
    
    printShapeInfo(Circle(5.0))
}
```

---

## Polymorphism

```kotlin
abstract class Employee(
    val name: String,
    val baseSalary: Double
) {
    abstract fun calculateSalary(): Double
    abstract fun getRole(): String
    
    fun payslip(): String = buildString {
        appendLine("=== Payslip: $name ===")
        appendLine("Role: ${getRole()}")
        appendLine("Base: ${"%.2f".format(baseSalary)}")
        appendLine("Total: ${"%.2f".format(calculateSalary())}")
    }
}

class FullTimeEmployee(name: String, baseSalary: Double) : Employee(name, baseSalary) {
    override fun calculateSalary() = baseSalary
    override fun getRole() = "Full-Time"
}

class PartTimeEmployee(
    name: String,
    hourlyRate: Double,
    val hoursWorked: Int
) : Employee(name, hourlyRate) {
    override fun calculateSalary() = baseSalary * hoursWorked
    override fun getRole() = "Part-Time"
}

class Contractor(
    name: String,
    dailyRate: Double,
    val daysWorked: Int,
    val bonusRate: Double = 0.1
) : Employee(name, dailyRate) {
    override fun calculateSalary() = baseSalary * daysWorked * (1 + bonusRate)
    override fun getRole() = "Contractor"
}

fun main() {
    // Polymorphic list
    val employees: List<Employee> = listOf(
        FullTimeEmployee("สมชาย", 50_000.0),
        PartTimeEmployee("สมหญิง", 200.0, 80),
        Contractor("สมศักดิ์", 1_500.0, 20, 0.15),
        FullTimeEmployee("สมพร", 45_000.0)
    )
    
    // Polymorphic method calls
    employees.forEach { emp ->
        println(emp.payslip())
    }
    
    // Aggregate
    val totalPayroll = employees.sumOf { it.calculateSalary() }
    println("Total payroll: ${"%.2f".format(totalPayroll)}")
    
    // Group by role
    val byRole = employees.groupBy { it.getRole() }
    byRole.forEach { (role, emps) ->
        println("$role: ${emps.map { it.name }.joinToString()}")
    }
    
    // when with type (smart cast)
    employees.forEach { emp ->
        val extra = when (emp) {
            is Contractor -> "Bonus rate: ${emp.bonusRate * 100}%"
            is PartTimeEmployee -> "Hours: ${emp.hoursWorked}"
            else -> "Fixed salary"
        }
        println("${emp.name}: $extra")
    }
}
```

---

## Sealed Classes

```kotlin
// sealed: จำกัด subclasses ให้อยู่ในไฟล์เดียวกัน
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String, val code: Int = 0) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

sealed class NetworkState {
    object Connected : NetworkState()
    object Disconnected : NetworkState()
    data class Error(val message: String) : NetworkState()
    data class Slow(val speed: Int) : NetworkState() // kbps
}

// เมื่อใช้ when กับ sealed class: exhaustive (ไม่ต้องมี else)
fun handleNetworkState(state: NetworkState): String = when (state) {
    is NetworkState.Connected -> "Connected! ✅"
    is NetworkState.Disconnected -> "No connection ❌"
    is NetworkState.Error -> "Error: ${state.message}"
    is NetworkState.Slow -> "Slow connection: ${state.speed} kbps ⚠️"
}

// Sealed class สำหรับ UI Events
sealed class UIEvent {
    data class ButtonClick(val buttonId: String) : UIEvent()
    data class TextInput(val fieldId: String, val text: String) : UIEvent()
    data class SwipeGesture(val direction: Direction) : UIEvent()
    object BackPress : UIEvent()
}

enum class Direction { UP, DOWN, LEFT, RIGHT }

fun handleEvent(event: UIEvent) = when (event) {
    is UIEvent.ButtonClick -> println("Button clicked: ${event.buttonId}")
    is UIEvent.TextInput -> println("Input in ${event.fieldId}: '${event.text}'")
    is UIEvent.SwipeGesture -> println("Swiped ${event.direction}")
    UIEvent.BackPress -> println("Back pressed")
}

fun main() {
    // Result pattern
    fun divide(a: Int, b: Int): Result<Double> {
        if (b == 0) return Result.Error("Division by zero", 400)
        return Result.Success(a.toDouble() / b)
    }
    
    listOf(divide(10, 2), divide(5, 0)).forEach { result ->
        when (result) {
            is Result.Success -> println("Result: ${result.data}")
            is Result.Error -> println("Error ${result.code}: ${result.message}")
            Result.Loading -> println("Loading...")
        }
    }
    
    // Network states
    val states = listOf(
        NetworkState.Connected,
        NetworkState.Disconnected,
        NetworkState.Error("Timeout"),
        NetworkState.Slow(128)
    )
    states.forEach { println(handleNetworkState(it)) }
    
    // UI Events
    val events = listOf(
        UIEvent.ButtonClick("submit"),
        UIEvent.TextInput("username", "somchai"),
        UIEvent.SwipeGesture(Direction.RIGHT),
        UIEvent.BackPress
    )
    events.forEach { handleEvent(it) }
}
```

---

## แบบฝึกหัด

### Exercise: Animal Kingdom

```kotlin
abstract class Animal(val name: String, val age: Int) {
    abstract fun sound(): String
    abstract fun move(): String
    
    open fun eat() = "$name กำลังกิน"
    
    override fun toString() = "$name (อายุ $age ปี)"
}

interface Domestic {
    fun beOwned(ownerName: String): String
}

class Dog(name: String, age: Int, val breed: String) : Animal(name, age), Domestic {
    override fun sound() = "โฮ่ง!"
    override fun move() = "วิ่งสี่ขา"
    override fun beOwned(ownerName: String) = "$name เป็นสุนัขของ $ownerName"
    override fun toString() = "${super.toString()} - $breed"
}

class Bird(name: String, age: Int, val canFly: Boolean) : Animal(name, age) {
    override fun sound() = "จิ๊บๆ!"
    override fun move() = if (canFly) "บินด้วยปีก" else "เดินสองขา"
}

class Fish(name: String, age: Int, val isDeepSea: Boolean) : Animal(name, age) {
    override fun sound() = "..."
    override fun move() = "ว่ายน้ำ"
    override fun eat() = "$name จับแมลง/ปลาเล็กกิน"
}

fun main() {
    val animals: List<Animal> = listOf(
        Dog("Rex", 3, "German Shepherd"),
        Dog("Bella", 1, "Golden Retriever"),
        Bird("Tweety", 2, true),
        Bird("Tux", 5, false),
        Fish("Nemo", 1, false),
        Fish("Abyss", 3, true)
    )
    
    println("=== Animal Kingdom ===")
    animals.forEach { animal ->
        println("\n$animal")
        println("  เสียง: ${animal.sound()}")
        println("  เคลื่อนที่: ${animal.move()}")
        println("  กิน: ${animal.eat()}")
        
        if (animal is Domestic) {
            println("  ${animal.beOwned("คุณสมชาย")}")
        }
    }
    
    val dogs = animals.filterIsInstance<Dog>()
    println("\n=== สุนัขทั้งหมด (${dogs.size} ตัว) ===")
    dogs.forEach { println("  $it") }
}
```

---

## สรุป Part 13

```
✅ open class อนุญาตให้ inherit
✅ override fun/val สำหรับ override
✅ super.method() เรียก parent implementation
✅ final override ป้องกัน further override
✅ abstract class: ไม่สร้าง instance โดยตรง
✅ abstract fun/val: ต้อง override ใน subclass
✅ interface: multiple implementation
✅ interface default methods
✅ interface inheritance
✅ sealed class: จำกัด subclasses
✅ when กับ sealed class เป็น exhaustive
✅ Polymorphism: List<Base>, เรียก methods ผ่าน base type
✅ Smart cast หลัง is check
```

---

*Part 13/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
