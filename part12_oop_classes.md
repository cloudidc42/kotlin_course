# Part 12: OOP - Classes และ Objects

## สารบัญ
1. [Class พื้นฐาน](#class-พื้นฐาน)
2. [Properties](#properties)
3. [Constructors](#constructors)
4. [Methods](#methods)
5. [Object Declarations (Singleton)](#object-declarations-singleton)
6. [Companion Object](#companion-object)
7. [Data Class](#data-class)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Class พื้นฐาน

```kotlin
// สร้าง class
class Person {
    // Properties
    var name: String = ""
    var age: Int = 0
    
    // Method
    fun greet() {
        println("สวัสดี ฉันชื่อ $name อายุ $age ปี")
    }
    
    fun isAdult(): Boolean = age >= 18
}

fun main() {
    // สร้าง instance (ไม่มี new ใน Kotlin)
    val p = Person()
    p.name = "สมชาย"
    p.age = 25
    
    p.greet()              // สวัสดี ฉันชื่อ สมชาย อายุ 25 ปี
    println(p.isAdult())   // true
    
    // แก้ไข property
    p.age = 17
    println(p.isAdult())   // false
}
```

### Access Modifiers

```kotlin
class BankAccount {
    var owner: String = ""           // public (default)
    protected var balance: Double = 0.0  // protected
    private var pin: String = ""         // private
    internal var branch: String = ""     // internal (same module)
    
    fun deposit(amount: Double) {
        if (amount > 0) balance += amount
    }
    
    fun getBalance(): Double = balance  // public getter
    
    private fun validatePin(input: String): Boolean = input == pin
    
    fun withdraw(amount: Double, inputPin: String): Boolean {
        if (!validatePin(inputPin)) return false
        if (amount > balance) return false
        balance -= amount
        return true
    }
}
```

---

## Properties

```kotlin
class Rectangle {
    var width: Double = 0.0
        set(value) {
            require(value > 0) { "Width must be positive" }
            field = value
        }
    
    var height: Double = 0.0
        set(value) {
            require(value > 0) { "Height must be positive" }
            field = value
        }
    
    // Computed property (ไม่มี backing field)
    val area: Double
        get() = width * height
    
    val perimeter: Double
        get() = 2 * (width + height)
    
    val diagonal: Double
        get() = Math.sqrt(width * width + height * height)
    
    // Property with custom getter & setter
    var widthCm: Double
        get() = width * 100
        set(value) { width = value / 100 }
}

fun main() {
    val rect = Rectangle()
    rect.width = 5.0
    rect.height = 3.0
    
    println("Area: ${rect.area}")           // 15.0
    println("Perimeter: ${rect.perimeter}") // 16.0
    println("Diagonal: ${"%.2f".format(rect.diagonal)}") // 5.83
    
    rect.widthCm = 500.0  // = 5.0 meters
    println("Width: ${rect.width}")   // 5.0
    
    // Backing field
    class Counter {
        var count: Int = 0
            private set  // setter เป็น private
        
        fun increment() { count++ }
        fun reset() { count = 0 }
    }
    
    val counter = Counter()
    counter.increment()
    counter.increment()
    println(counter.count)  // 2
    // counter.count = 5  // ❌ Error: setter is private
}
```

### Late-initialized Properties

```kotlin
class View {
    // lateinit: ต้องกำหนดก่อนใช้
    lateinit var title: String
    
    fun setup() {
        title = "My View"  // กำหนดทีหลัง
    }
    
    fun render() {
        // ตรวจก่อนใช้
        if (::title.isInitialized) {
            println("Title: $title")
        } else {
            println("Not initialized yet")
        }
    }
}

// lazy initialization (Thread-safe by default)
class ExpensiveObject {
    val heavyData: List<String> by lazy {
        println("Computing heavy data...")
        (1..1000).map { "Item $it" }
    }
    
    val lightData = "Simple data"
}

fun main() {
    val view = View()
    view.render()  // Not initialized yet
    view.setup()
    view.render()  // Title: My View
    
    val obj = ExpensiveObject()
    println(obj.lightData)  // Simple data (ไม่ compute heavyData)
    println(obj.heavyData.size)  // Computing heavy data... \n 1000
    println(obj.heavyData.size)  // 1000 (ใช้ cached value)
}
```

---

## Constructors

### Primary Constructor

```kotlin
// Primary constructor ใน class header
class Person(val name: String, val age: Int) {
    
    // init block: รันหลัง primary constructor
    init {
        require(name.isNotBlank()) { "Name cannot be blank" }
        require(age >= 0) { "Age cannot be negative" }
        println("Creating Person: $name, $age")
    }
    
    fun greet() = "สวัสดี ฉันชื่อ $name"
}

// Primary constructor กับ default values
class Student(
    val name: String,
    val grade: Int = 1,
    val section: String = "A"
) {
    val description get() = "นักเรียน $name ชั้น ม.$grade/$section"
}

fun main() {
    val p1 = Person("สมชาย", 25)
    val p2 = Person(name = "สมหญิง", age = 30)
    
    println(p1.greet())
    
    // Default values
    val s1 = Student("สมชาย")           // grade=1, section=A
    val s2 = Student("สมหญิง", 3)       // section=A
    val s3 = Student("สมศักดิ์", 2, "B")
    
    println(s1.description)
    println(s2.description)
    println(s3.description)
}
```

### Secondary Constructors

```kotlin
class Car {
    val make: String
    val model: String
    val year: Int
    val color: String
    
    // Secondary constructor 1
    constructor(make: String, model: String, year: Int) {
        this.make = make
        this.model = model
        this.year = year
        this.color = "White"  // default
    }
    
    // Secondary constructor 2 - delegates to 1
    constructor(make: String, model: String, year: Int, color: String)
        : this(make, model, year) {
        // this.color = color  // ❌ val ไม่สามารถ reassign
    }
    
    // ❌ ปัญหา: val ไม่สามารถ reassign ใน secondary constructor
    // ✅ วิธีที่ดีกว่า: ใช้ primary constructor + default values
    
    override fun toString() = "$year $color $make $model"
}

// ✅ Better approach
class Car2(
    val make: String,
    val model: String,
    val year: Int,
    val color: String = "White"
) {
    override fun toString() = "$year $color $make $model"
}

fun main() {
    val car1 = Car("Toyota", "Camry", 2024)
    println(car1)  // 2024 White Toyota Camry
    
    val car2 = Car2("Honda", "Civic", 2023, "Red")
    val car3 = Car2("Mazda", "CX-5", 2024)
    println(car2)  // 2023 Red Honda Civic
    println(car3)  // 2024 White Mazda CX-5
}
```

---

## Methods

```kotlin
class BankAccount(
    val owner: String,
    initialBalance: Double = 0.0
) {
    private var _balance: Double = initialBalance
    private val _transactions = mutableListOf<String>()
    
    val balance: Double get() = _balance
    val transactions: List<String> get() = _transactions.toList()
    
    fun deposit(amount: Double): BankAccount {
        require(amount > 0) { "Deposit amount must be positive" }
        _balance += amount
        _transactions.add("ฝาก: +${"%,.2f".format(amount)}")
        return this  // method chaining
    }
    
    fun withdraw(amount: Double): BankAccount {
        require(amount > 0) { "Withdrawal amount must be positive" }
        require(amount <= _balance) { "Insufficient funds" }
        _balance -= amount
        _transactions.add("ถอน: -${"%,.2f".format(amount)}")
        return this  // method chaining
    }
    
    fun transfer(amount: Double, to: BankAccount): BankAccount {
        withdraw(amount)
        to.deposit(amount)
        _transactions.add("โอน: -${"%,.2f".format(amount)} → ${to.owner}")
        to._transactions.add("รับโอน: +${"%,.2f".format(amount)} จาก $owner")
        return this
    }
    
    fun printStatement() {
        println("=== Statement: $owner ===")
        _transactions.forEach { println("  $it") }
        println("  ยอดเงินปัจจุบัน: ${"%,.2f".format(_balance)} บาท")
    }
}

fun main() {
    val account1 = BankAccount("สมชาย", 10_000.0)
    val account2 = BankAccount("สมหญิง")
    
    // Method chaining
    account1
        .deposit(5_000.0)
        .deposit(3_000.0)
        .withdraw(2_000.0)
        .transfer(1_000.0, account2)
    
    account1.printStatement()
    account2.printStatement()
}
```

---

## Object Declarations (Singleton)

```kotlin
// object = Singleton
object DatabaseConfig {
    val host = "localhost"
    val port = 5432
    val dbName = "myapp"
    
    val connectionString: String
        get() = "jdbc:postgresql://$host:$port/$dbName"
    
    fun connect() {
        println("Connecting to $connectionString")
    }
}

// Object สามารถ implement interface
interface Logger {
    fun log(message: String)
}

object ConsoleLogger : Logger {
    private var logCount = 0
    
    override fun log(message: String) {
        logCount++
        println("[$logCount] $message")
    }
    
    fun getLogCount() = logCount
}

fun main() {
    // ใช้ object โดยตรง (ไม่ต้องสร้าง instance)
    println(DatabaseConfig.connectionString)
    DatabaseConfig.connect()
    
    ConsoleLogger.log("Application started")
    ConsoleLogger.log("User logged in")
    ConsoleLogger.log("Data saved")
    println("Total logs: ${ConsoleLogger.getLogCount()}")
    
    // Object expressions (anonymous object)
    val point = object {
        val x = 10
        val y = 20
        override fun toString() = "($x, $y)"
    }
    println(point)  // (10, 20)
    
    // Anonymous object implementing interface
    val listener = object : Logger {
        override fun log(message: String) {
            println("CUSTOM: $message")
        }
    }
    listener.log("Hello")
}
```

---

## Companion Object

```kotlin
class User private constructor(
    val id: Int,
    val name: String,
    val email: String
) {
    companion object Factory {
        private var nextId = 1
        private val users = mutableMapOf<String, User>()
        
        // Factory methods (static-like)
        fun create(name: String, email: String): User? {
            if (users.containsKey(email)) return null  // email ซ้ำ
            val user = User(nextId++, name, email)
            users[email] = user
            return user
        }
        
        fun findByEmail(email: String): User? = users[email]
        
        fun getAllUsers(): List<User> = users.values.toList()
        
        fun count(): Int = users.size
        
        // Constants
        const val MAX_USERS = 1000
        const val DEFAULT_ROLE = "user"
    }
    
    override fun toString() = "User($id, $name, $email)"
}

fun main() {
    // เรียกผ่าน class name (เหมือน static method)
    val u1 = User.create("สมชาย", "somchai@example.com")
    val u2 = User.create("สมหญิง", "somying@example.com")
    val u3 = User.create("สมชาย", "somchai@example.com")  // email ซ้ำ!
    
    println(u1)   // User(1, สมชาย, somchai@example.com)
    println(u2)   // User(2, สมหญิง, somying@example.com)
    println(u3)   // null
    
    println("Total users: ${User.count()}")
    println("Max users: ${User.MAX_USERS}")
    
    val found = User.findByEmail("somying@example.com")
    println("Found: $found")
    
    User.getAllUsers().forEach { println(it) }
}
```

---

## Data Class

```kotlin
// data class: auto-generates equals, hashCode, toString, copy, componentN
data class Point(val x: Double, val y: Double) {
    // สามารถเพิ่ม methods ได้
    fun distanceTo(other: Point): Double {
        val dx = x - other.x
        val dy = y - other.y
        return Math.sqrt(dx * dx + dy * dy)
    }
    
    operator fun plus(other: Point) = Point(x + other.x, y + other.y)
    operator fun minus(other: Point) = Point(x - other.x, y - other.y)
}

data class Address(
    val street: String,
    val city: String,
    val country: String = "Thailand"
)

data class Employee(
    val id: Int,
    val name: String,
    val department: String,
    val salary: Double,
    val address: Address
)

fun main() {
    val p1 = Point(3.0, 4.0)
    val p2 = Point(0.0, 0.0)
    
    // toString (auto-generated)
    println(p1)  // Point(x=3.0, y=4.0)
    
    // equals (auto-generated, structural)
    println(p1 == Point(3.0, 4.0))  // true
    println(p1 == p2)               // false
    
    // copy - สร้าง copy กับ changes
    val p3 = p1.copy(y = 0.0)
    println(p3)  // Point(x=3.0, y=0.0)
    
    // Operations
    println(p1 + p2)  // Point(x=3.0, y=4.0)
    println(p1.distanceTo(p2))  // 5.0
    
    // Destructuring
    val (x, y) = p1
    println("x=$x, y=$y")  // x=3.0, y=4.0
    
    // Deep copy example
    val emp1 = Employee(1, "สมชาย", "IT", 50_000.0, Address("123 Main St", "Bangkok"))
    val emp2 = emp1.copy(
        id = 2,
        name = "สมหญิง",
        address = emp1.address.copy(city = "Chiang Mai")
    )
    
    println(emp1)
    println(emp2)
    
    // hashCode
    val set = setOf(p1, p2, Point(3.0, 4.0))
    println(set.size)  // 2 (p1 และ Point(3.0, 4.0) เท่ากัน)
    
    // Data class as Map key
    val pointMap = mapOf(
        Point(0.0, 0.0) to "origin",
        Point(1.0, 0.0) to "unit-x",
        Point(0.0, 1.0) to "unit-y"
    )
    println(pointMap[Point(0.0, 0.0)])  // origin
}
```

---

## ตัวอย่างโปรแกรม - Library System

```kotlin
import java.time.LocalDate

data class Book(
    val isbn: String,
    val title: String,
    val author: String,
    val year: Int,
    var available: Boolean = true
)

data class Member(
    val id: String,
    val name: String,
    val email: String
)

data class Loan(
    val book: Book,
    val member: Member,
    val loanDate: LocalDate = LocalDate.now(),
    val dueDate: LocalDate = LocalDate.now().plusDays(14)
)

class Library(val name: String) {
    private val books = mutableListOf<Book>()
    private val members = mutableListOf<Member>()
    private val loans = mutableListOf<Loan>()
    
    fun addBook(book: Book) { books.add(book) }
    fun addMember(member: Member) { members.add(member) }
    
    fun borrowBook(isbn: String, memberId: String): Boolean {
        val book = books.find { it.isbn == isbn && it.available } ?: return false
        val member = members.find { it.id == memberId } ?: return false
        
        book.available = false
        loans.add(Loan(book, member))
        println("${member.name} ยืม '${book.title}' สำเร็จ")
        return true
    }
    
    fun returnBook(isbn: String): Boolean {
        val loan = loans.find { it.book.isbn == isbn } ?: return false
        loan.book.available = true
        loans.remove(loan)
        println("คืนหนังสือ '${loan.book.title}' สำเร็จ")
        return true
    }
    
    fun searchBooks(query: String): List<Book> =
        books.filter { 
            it.title.contains(query, ignoreCase = true) ||
            it.author.contains(query, ignoreCase = true)
        }
    
    fun status() {
        println("\n=== $name ===")
        println("หนังสือทั้งหมด: ${books.size}")
        println("หนังสือที่ว่าง: ${books.count { it.available }}")
        println("หนังสือที่ถูกยืม: ${books.count { !it.available }}")
        println("สมาชิก: ${members.size}")
        
        if (loans.isNotEmpty()) {
            println("\nหนังสือที่ถูกยืมอยู่:")
            loans.forEach { l ->
                println("  '${l.book.title}' → ${l.member.name} (ครบ ${l.dueDate})")
            }
        }
    }
}

fun main() {
    val library = Library("ห้องสมุดเมือง")
    
    library.addBook(Book("978-0001", "Kotlin Programming", "JetBrains", 2021))
    library.addBook(Book("978-0002", "Clean Code", "Robert Martin", 2008))
    library.addBook(Book("978-0003", "Design Patterns", "GoF", 1994))
    
    library.addMember(Member("M001", "สมชาย", "somchai@email.com"))
    library.addMember(Member("M002", "สมหญิง", "somying@email.com"))
    
    library.borrowBook("978-0001", "M001")
    library.borrowBook("978-0002", "M002")
    library.borrowBook("978-0001", "M002")  // ไม่ว่าง!
    
    library.status()
    
    println("\nค้นหา 'Kotlin':")
    library.searchBooks("Kotlin").forEach { println("  ${it.title}") }
    
    library.returnBook("978-0001")
    library.status()
}
```

---

## แบบฝึกหัด

### Exercise: Shape Hierarchy Preparation

```kotlin
data class Circle(val radius: Double) {
    val area: Double get() = Math.PI * radius * radius
    val circumference: Double get() = 2 * Math.PI * radius
}

data class Rectangle(val width: Double, val height: Double) {
    val area: Double get() = width * height
    val perimeter: Double get() = 2 * (width + height)
    val diagonal: Double get() = Math.sqrt(width * width + height * height)
}

data class Triangle(val a: Double, val b: Double, val c: Double) {
    val perimeter: Double get() = a + b + c
    val area: Double get() {
        val s = perimeter / 2
        return Math.sqrt(s * (s - a) * (s - b) * (s - c))
    }
    val isValid: Boolean get() = a + b > c && b + c > a && a + c > b
}

fun main() {
    val circle = Circle(5.0)
    val rect = Rectangle(4.0, 6.0)
    val tri = Triangle(3.0, 4.0, 5.0)
    
    println("Circle: area=${"%.2f".format(circle.area)}, circum=${"%.2f".format(circle.circumference)}")
    println("Rectangle: area=${rect.area}, perimeter=${rect.perimeter}")
    println("Triangle: valid=${tri.isValid}, area=${"%.2f".format(tri.area)}")
    
    val shapes = listOf(circle.area, rect.area, tri.area)
    println("Largest area: ${"%.2f".format(shapes.max())}")
}
```

---

## สรุป Part 12

```
✅ class keyword สร้าง class
✅ Properties: val/var ใน class
✅ Custom getter/setter ด้วย get()/set()
✅ field: backing field ใน setter
✅ Primary constructor ใน class header
✅ init block รันหลัง constructor
✅ Secondary constructors ด้วย constructor()
✅ Default parameter values
✅ lateinit var สำหรับ late initialization
✅ by lazy { } สำหรับ lazy initialization
✅ object: Singleton pattern
✅ companion object: static-like members
✅ data class: auto-generates equals/hashCode/toString/copy
✅ Destructuring declaration: val (a, b) = obj
```

---

*Part 12/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
