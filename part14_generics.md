# Part 14: Generics

## สารบัญ
1. [Generic Functions](#generic-functions)
2. [Generic Classes](#generic-classes)
3. [Type Constraints](#type-constraints)
4. [Variance (in/out)](#variance-inout)
5. [Reified Type Parameters](#reified-type-parameters)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Generic Functions

```kotlin
// T คือ type parameter - placeholder สำหรับ type จริง
fun <T> identity(value: T): T = value

fun <T> swap(pair: Pair<T, T>): Pair<T, T> = Pair(pair.second, pair.first)

fun <T> List<T>.secondOrNull(): T? = if (size >= 2) get(1) else null

fun main() {
    // Type inference - Kotlin infer T จาก argument
    println(identity(42))           // 42 (T = Int)
    println(identity("Hello"))      // Hello (T = String)
    println(identity(3.14))         // 3.14 (T = Double)
    
    // Explicit type argument
    println(identity<String>("World"))
    
    val swapped = swap(Pair("Hello", "World"))
    println(swapped)  // (World, Hello)
    
    val swappedNums = swap(1 to 2)
    println(swappedNums)  // (2, 1)
    
    println(listOf(1, 2, 3).secondOrNull())   // 2
    println(listOf(1).secondOrNull())          // null
    println(listOf<String>().secondOrNull())   // null
    
    // Multiple type parameters
    fun <K, V> Map<K, V>.getOrDefault(key: K, default: V): V =
        get(key) ?: default
    
    val map = mapOf("a" to 1, "b" to 2)
    println(map.getOrDefault("a", 0))  // 1
    println(map.getOrDefault("z", -1)) // -1
}
```

---

## Generic Classes

```kotlin
// Generic Stack
class Stack<T> {
    private val items = mutableListOf<T>()
    
    fun push(item: T) = items.add(item)
    
    fun pop(): T {
        if (isEmpty()) throw NoSuchElementException("Stack is empty")
        return items.removeLast()
    }
    
    fun peek(): T {
        if (isEmpty()) throw NoSuchElementException("Stack is empty")
        return items.last()
    }
    
    fun isEmpty(): Boolean = items.isEmpty()
    val size: Int get() = items.size
    
    override fun toString() = items.toString()
}

// Generic Queue
class Queue<T> {
    private val items = ArrayDeque<T>()
    
    fun enqueue(item: T) = items.addLast(item)
    fun dequeue(): T = items.removeFirst()
    fun peek(): T = items.first()
    fun isEmpty() = items.isEmpty()
    val size get() = items.size
    
    override fun toString() = items.toString()
}

// Generic Pair with operations
data class Pair2<A, B>(val first: A, val second: B) {
    fun swap(): Pair2<B, A> = Pair2(second, first)
    fun toList(): List<Any?> = listOf(first, second)
    fun <C> map(transform: (A, B) -> C): C = transform(first, second)
}

// Generic Result type
sealed class Either<out L, out R> {
    data class Left<L>(val value: L) : Either<L, Nothing>()
    data class Right<R>(val value: R) : Either<Nothing, R>()
    
    fun isLeft() = this is Left
    fun isRight() = this is Right
    
    fun <T> fold(onLeft: (L) -> T, onRight: (R) -> T): T = when (this) {
        is Left -> onLeft(value)
        is Right -> onRight(value)
    }
}

fun main() {
    // Stack usage
    val stack = Stack<Int>()
    stack.push(1)
    stack.push(2)
    stack.push(3)
    println("Stack: $stack")     // [1, 2, 3]
    println("Pop: ${stack.pop()}") // 3
    println("Peek: ${stack.peek()}") // 2
    
    // Queue usage
    val queue = Queue<String>()
    queue.enqueue("first")
    queue.enqueue("second")
    queue.enqueue("third")
    println("Queue: $queue")
    println("Dequeue: ${queue.dequeue()}")  // first
    
    // Either usage
    fun safeDivide(a: Int, b: Int): Either<String, Double> =
        if (b == 0) Either.Left("Division by zero")
        else Either.Right(a.toDouble() / b)
    
    val result = safeDivide(10, 2)
    println(result.fold(
        onLeft = { "Error: $it" },
        onRight = { "Result: $it" }
    ))
    
    // Generic function using the class
    fun <T : Comparable<T>> Stack<T>.pushSorted(vararg items: T) {
        items.sorted().forEach { push(it) }
    }
    
    val sortedStack = Stack<Int>()
    sortedStack.pushSorted(5, 2, 8, 1, 9, 3)
    println("Sorted stack: $sortedStack")
}
```

---

## Type Constraints

```kotlin
// Upper bound constraint
fun <T : Comparable<T>> max(a: T, b: T): T = if (a >= b) a else b

fun <T : Comparable<T>> List<T>.sortedAndFiltered(
    minValue: T,
    maxValue: T
): List<T> = filter { it in minValue..maxValue }.sorted()

// Multiple constraints
interface Printable {
    fun print()
}

interface Serializable {
    fun serialize(): String
}

// where clause สำหรับ multiple constraints
fun <T> process(item: T) where T : Printable, T : Serializable {
    item.print()
    println("Serialized: ${item.serialize()}")
}

// Constraint ใน class
class SortedList<T : Comparable<T>> {
    private val items = mutableListOf<T>()
    
    fun add(item: T) {
        val index = items.binarySearch(item).let { 
            if (it < 0) -it - 1 else it
        }
        items.add(index, item)
    }
    
    fun toList() = items.toList()
    override fun toString() = items.toString()
}

// Number constraint
fun <T : Number> sum(list: List<T>): Double =
    list.sumOf { it.toDouble() }

fun <T : Number> average(list: List<T>): Double =
    if (list.isEmpty()) 0.0 else sum(list) / list.size

fun main() {
    println(max(3, 7))           // 7
    println(max("apple", "zoo")) // zoo
    println(max(3.14, 2.71))     // 3.14
    
    val nums = listOf(7, 2, 9, 1, 5, 8, 3, 6, 4)
    println(nums.sortedAndFiltered(3, 7))  // [3, 4, 5, 6, 7]
    
    val sorted = SortedList<Int>()
    sorted.add(5); sorted.add(2); sorted.add(8); sorted.add(1)
    println(sorted)  // [1, 2, 5, 8]
    
    println(sum(listOf(1, 2, 3, 4, 5)))        // 15.0
    println(average(listOf(10.0, 20.0, 30.0))) // 20.0
    println(average(listOf<Int>()))            // 0.0
}
```

---

## Variance (in/out)

```kotlin
// Covariance (out): Producer - อ่านอย่างเดียว
// ถ้า Foo<Dog> เป็น subtype ของ Foo<Animal> → covariant
interface Producer<out T> {
    fun produce(): T
}

class DogProducer : Producer<Dog> {
    override fun produce() = Dog("Buddy")
}

class Dog(val name: String)
class Animal(val name: String)

// ✅ Covariant: Producer<Dog> ใช้แทน Producer<Animal> ได้
fun feedAnimal(producer: Producer<Animal>) {
    val animal = producer.produce()
    println("Feeding ${animal.name}")
}

// Contravariance (in): Consumer - เขียนอย่างเดียว
// ถ้า Foo<Animal> เป็น subtype ของ Foo<Dog> → contravariant
interface Consumer<in T> {
    fun consume(item: T)
}

class AnimalConsumer : Consumer<Animal> {
    override fun consume(item: Animal) {
        println("Consuming animal: ${item.name}")
    }
}

// ✅ Contravariant: Consumer<Animal> ใช้แทน Consumer<Dog> ได้
fun feedDog(consumer: Consumer<Dog>) {
    consumer.consume(Dog("Rex"))
}

// Invariant: ทำได้ทั้งอ่านและเขียน
class Box<T>(var value: T)
// Box<Dog> ไม่ใช่ subtype ของ Box<Animal>

// Use-site variance (Type Projection)
fun copy(from: Array<out Animal>, to: Array<Animal>) {
    for (i in from.indices) {
        to[i] = from[i]
    }
}

// Practical example: Collections
fun main() {
    // List<T> is covariant (List<out T> in source)
    val dogs: List<Dog> = listOf(Dog("Rex"), Dog("Buddy"))
    val animals: List<Animal> = dogs  // ✅ Works because List is covariant
    
    // MutableList<T> is invariant
    val mutableDogs: MutableList<Dog> = mutableListOf(Dog("Rex"))
    // val mutableAnimals: MutableList<Animal> = mutableDogs  // ❌ Error!
    
    // Star Projection
    fun printList(list: List<*>) {  // List<out Any?>
        list.forEach { println(it) }
    }
    
    printList(dogs)
    printList(listOf(1, 2, 3))
    
    // Practical: Generic function with out
    fun <T> copyList(source: List<out T>, dest: MutableList<T>) {
        dest.addAll(source)
    }
    
    val src = listOf("a", "b", "c")
    val dst = mutableListOf<String>()
    copyList(src, dst)
    println(dst)  // [a, b, c]
}
```

---

## Reified Type Parameters

```kotlin
// reified: เข้าถึง type ได้ใน inline function
inline fun <reified T> Any.isInstance(): Boolean = this is T

inline fun <reified T> Any.castOrNull(): T? = this as? T

inline fun <reified T> List<*>.filterType(): List<T> =
    filterIsInstance<T>()

// JSON-like parsing
inline fun <reified T> parseConfig(config: Map<String, Any>, key: String): T? {
    val value = config[key] ?: return null
    return value as? T
}

// Type-safe event system
class EventBus {
    private val listeners = mutableMapOf<String, MutableList<(Any) -> Unit>>()
    
    inline fun <reified T : Any> subscribe(noinline listener: (T) -> Unit) {
        val key = T::class.simpleName ?: return
        listeners.getOrPut(key) { mutableListOf() }.add { event ->
            if (event is T) listener(event)
        }
    }
    
    fun publish(event: Any) {
        val key = event::class.simpleName ?: return
        listeners[key]?.forEach { it(event) }
    }
}

data class UserLoggedIn(val userId: String)
data class OrderPlaced(val orderId: String, val amount: Double)

fun main() {
    // isInstance
    println(42.isInstance<Int>())      // true
    println(42.isInstance<String>())   // false
    println("hello".isInstance<String>()) // true
    
    // castOrNull
    val obj: Any = "Hello"
    val str: String? = obj.castOrNull<String>()
    val num: Int? = obj.castOrNull<Int>()
    println(str)  // Hello
    println(num)  // null
    
    // filterType
    val mixed: List<Any> = listOf(1, "hello", 2, "world", 3.0, true)
    println(mixed.filterType<String>())  // [hello, world]
    println(mixed.filterType<Int>())     // [1, 2]
    
    // parseConfig
    val config = mapOf<String, Any>(
        "port" to 8080,
        "host" to "localhost",
        "debug" to true,
        "timeout" to 30.0
    )
    
    val port: Int? = parseConfig<Int>(config, "port")
    val host: String? = parseConfig<String>(config, "host")
    val debug: Boolean? = parseConfig<Boolean>(config, "debug")
    
    println("port=$port, host=$host, debug=$debug")
    
    // EventBus
    val bus = EventBus()
    
    bus.subscribe<UserLoggedIn> { event ->
        println("User logged in: ${event.userId}")
    }
    
    bus.subscribe<OrderPlaced> { event ->
        println("Order placed: ${event.orderId} = $${event.amount}")
    }
    
    bus.publish(UserLoggedIn("user123"))
    bus.publish(OrderPlaced("ORD-001", 299.99))
    bus.publish(UserLoggedIn("user456"))
}
```

---

## ตัวอย่างโปรแกรม - Generic Repository Pattern

```kotlin
interface Entity {
    val id: Int
}

interface Repository<T : Entity> {
    fun findById(id: Int): T?
    fun findAll(): List<T>
    fun save(entity: T): T
    fun delete(id: Int): Boolean
    fun count(): Int
}

abstract class InMemoryRepository<T : Entity> : Repository<T> {
    protected val storage = mutableMapOf<Int, T>()
    
    override fun findById(id: Int): T? = storage[id]
    override fun findAll(): List<T> = storage.values.toList()
    override fun save(entity: T): T {
        storage[entity.id] = entity
        return entity
    }
    override fun delete(id: Int): Boolean = storage.remove(id) != null
    override fun count(): Int = storage.size
    
    // Generic search
    fun findBy(predicate: (T) -> Boolean): List<T> =
        storage.values.filter(predicate)
    
    fun findFirst(predicate: (T) -> Boolean): T? =
        storage.values.find(predicate)
}

data class User(
    override val id: Int,
    val name: String,
    val email: String,
    val role: String = "user"
) : Entity

data class Product(
    override val id: Int,
    val name: String,
    val price: Double,
    val category: String
) : Entity

class UserRepository : InMemoryRepository<User>() {
    fun findByEmail(email: String) = findFirst { it.email == email }
    fun findByRole(role: String) = findBy { it.role == role }
}

class ProductRepository : InMemoryRepository<Product>() {
    fun findByCategory(category: String) = findBy { it.category == category }
    fun findByPriceRange(min: Double, max: Double) = findBy { it.price in min..max }
    fun findCheapest(n: Int) = findAll().sortedBy { it.price }.take(n)
}

fun main() {
    val userRepo = UserRepository()
    val productRepo = ProductRepository()
    
    // Add users
    userRepo.save(User(1, "สมชาย", "somchai@email.com", "admin"))
    userRepo.save(User(2, "สมหญิง", "somying@email.com"))
    userRepo.save(User(3, "สมศักดิ์", "somsak@email.com"))
    
    // Add products
    productRepo.save(Product(1, "Laptop", 25000.0, "Electronics"))
    productRepo.save(Product(2, "Phone", 15000.0, "Electronics"))
    productRepo.save(Product(3, "Desk", 5000.0, "Furniture"))
    productRepo.save(Product(4, "Chair", 2500.0, "Furniture"))
    
    println("=== Users ===")
    println("All: ${userRepo.count()}")
    println("Admin: ${userRepo.findByRole("admin").map { it.name }}")
    println("By email: ${userRepo.findByEmail("somying@email.com")?.name}")
    
    println("\n=== Products ===")
    println("Electronics: ${productRepo.findByCategory("Electronics").map { it.name }}")
    println("Under 10k: ${productRepo.findByPriceRange(0.0, 10000.0).map { it.name }}")
    println("3 cheapest: ${productRepo.findCheapest(3).map { "${it.name} (${it.price})" }}")
    
    // CRUD
    userRepo.delete(3)
    println("\nAfter delete: ${userRepo.count()} users")
}
```

---

## แบบฝึกหัด

### Exercise: Generic Cache

```kotlin
class Cache<K, V>(private val maxSize: Int = 10) {
    private val data = LinkedHashMap<K, V>(maxSize, 0.75f, true)
    
    fun put(key: K, value: V) {
        if (data.size >= maxSize) {
            val oldestKey = data.keys.first()
            data.remove(oldestKey)
        }
        data[key] = value
    }
    
    fun get(key: K): V? = data[key]
    
    fun contains(key: K) = data.containsKey(key)
    
    fun size() = data.size
    
    fun getOrCompute(key: K, compute: () -> V): V {
        return get(key) ?: compute().also { put(key, it) }
    }
}

fun main() {
    val cache = Cache<String, Int>(maxSize = 3)
    
    cache.put("a", 1)
    cache.put("b", 2)
    cache.put("c", 3)
    println("Size: ${cache.size()}")  // 3
    
    cache.put("d", 4)  // evicts "a"
    println("Contains 'a': ${cache.contains("a")}")  // false
    println("Contains 'd': ${cache.contains("d")}")  // true
    
    // getOrCompute
    var computeCount = 0
    val result1 = cache.getOrCompute("expensive") {
        computeCount++
        42
    }
    val result2 = cache.getOrCompute("expensive") {
        computeCount++
        42
    }
    println("Result: $result1, $result2, computed $computeCount times")
    // Result: 42, 42, computed 1 times
}
```

---

## สรุป Part 14

```
✅ <T> type parameter สำหรับ functions และ classes
✅ Type inference: Kotlin infer T จาก context
✅ Upper bound: <T : SomeClass>
✅ Multiple bounds: where T : A, T : B
✅ Covariance (out T): Producer, อ่านได้อย่างเดียว
✅ Contravariance (in T): Consumer, เขียนได้อย่างเดียว
✅ Invariance: ทำได้ทั้ง read/write (default)
✅ Star projection: <*> = <out Any?>
✅ reified: เข้าถึง type ใน inline function
✅ Generic patterns: Stack, Queue, Repository
```

---

*Part 14/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
