# Part 11: Collections - List, Set, Map

## สารบัญ
1. [Collections Overview](#collections-overview)
2. [List](#list)
3. [Set](#set)
4. [Map](#map)
5. [Collection Operations](#collection-operations)
6. [Destructuring](#destructuring)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Collections Overview

```
Kotlin Collections
├── Collection<T>
│   ├── List<T>         → ordered, allows duplicates
│   │   ├── listOf()    → immutable
│   │   └── mutableListOf() / ArrayList
│   └── Set<T>          → unordered, unique values
│       ├── setOf()     → immutable
│       ├── mutableSetOf() / HashSet
│       ├── linkedSetOf() / LinkedHashSet (ordered)
│       └── sortedSetOf() / TreeSet (sorted)
└── Map<K, V>           → key-value pairs
    ├── mapOf()         → immutable
    ├── mutableMapOf() / HashMap
    ├── linkedMapOf() / LinkedHashMap (insertion order)
    └── sortedMapOf() / TreeMap (sorted keys)
```

---

## List

### Immutable List

```kotlin
fun main() {
    // listOf() - immutable
    val fruits = listOf("Apple", "Banana", "Cherry", "Date")
    
    println(fruits)           // [Apple, Banana, Cherry, Date]
    println(fruits.size)      // 4
    println(fruits[0])        // Apple
    println(fruits.first())   // Apple
    println(fruits.last())    // Date
    println(fruits.get(2))    // Cherry
    
    // getOrNull - safe access
    println(fruits.getOrNull(10))  // null (ไม่ throw exception)
    println(fruits.getOrElse(10) { "Unknown" })  // Unknown
    
    // Contains
    println(fruits.contains("Apple"))  // true
    println("Mango" in fruits)         // false
    
    // Index operations
    println(fruits.indexOf("Cherry"))     // 2
    println(fruits.lastIndexOf("Apple"))  // 0
    println(fruits.indexOfFirst { it.startsWith("B") })  // 1
    println(fruits.indexOfLast { it.length > 5 })  // 2 (Cherry)
    
    // Sublist
    println(fruits.subList(1, 3))  // [Banana, Cherry]
    
    // emptyList
    val empty = emptyList<String>()
    println(empty.isEmpty())  // true
    
    // listOfNotNull - กรอง null ออก
    val nullable = listOfNotNull("a", null, "b", null, "c")
    println(nullable)  // [a, b, c]
}
```

### Mutable List

```kotlin
fun main() {
    // mutableListOf()
    val tasks = mutableListOf("Task A", "Task B", "Task C")
    
    // Add
    tasks.add("Task D")           // ท้าย
    tasks.add(1, "Task X")        // เพิ่มที่ index 1
    tasks.addAll(listOf("Task E", "Task F"))
    println(tasks)
    // [Task A, Task X, Task B, Task C, Task D, Task E, Task F]
    
    // Remove
    tasks.remove("Task X")        // ลบตาม value
    tasks.removeAt(0)             // ลบตาม index
    tasks.removeAll { it.contains("E") }  // ลบด้วย predicate
    println(tasks)  // [Task B, Task C, Task D, Task F]
    
    // Update
    tasks[0] = "Updated Task B"
    tasks.set(1, "Updated Task C")
    println(tasks)
    
    // Sort
    val numbers = mutableListOf(3, 1, 4, 1, 5, 9, 2, 6)
    numbers.sort()          // ascending in-place
    println(numbers)        // [1, 1, 2, 3, 4, 5, 6, 9]
    
    numbers.sortDescending()
    println(numbers)        // [9, 6, 5, 4, 3, 2, 1, 1]
    
    numbers.shuffle()       // random order
    println(numbers)
    
    numbers.reverse()       // in-place reverse
    
    // Retain
    val items = mutableListOf(1, 2, 3, 4, 5, 6)
    items.retainAll { it % 2 == 0 }  // เก็บเฉพาะที่เป็นจริง
    println(items)  // [2, 4, 6]
    
    // Clear
    items.clear()
    println(items)  // []
    
    // ArrayList (Java-style)
    val list = ArrayList<String>()
    list.add("Hello")
    list.add("World")
    println(list)
}
```

---

## Set

```kotlin
fun main() {
    // Set: ไม่มี duplicate
    val set = setOf(1, 2, 3, 2, 1, 4)
    println(set)        // [1, 2, 3, 4] - duplicate ถูกลบออก
    println(set.size)   // 4
    
    // HashSet - unordered (เร็วสุด)
    val hashSet = hashSetOf("apple", "banana", "apple", "cherry")
    println(hashSet)  // [banana, apple, cherry] order ไม่แน่นอน
    
    // LinkedHashSet - maintains insertion order
    val linked = linkedSetOf("c", "a", "b", "a", "d")
    println(linked)   // [c, a, b, d]
    
    // TreeSet - sorted order
    val sorted = sortedSetOf(5, 3, 1, 4, 2)
    println(sorted)   // [1, 2, 3, 4, 5]
    
    // Contains - O(1) for HashSet
    println("apple" in hashSet)  // true
    println("mango" in hashSet)  // false
    
    // Set operations
    val a = setOf(1, 2, 3, 4, 5)
    val b = setOf(3, 4, 5, 6, 7)
    
    println(a union b)        // [1, 2, 3, 4, 5, 6, 7]
    println(a intersect b)    // [3, 4, 5]
    println(a subtract b)     // [1, 2]
    println(b subtract a)     // [6, 7]
    
    // Mutable Set
    val mSet = mutableSetOf("x", "y", "z")
    mSet.add("w")
    mSet.add("x")  // ไม่เพิ่ม (duplicate)
    println(mSet)  // [x, y, z, w]
    
    mSet.remove("y")
    println(mSet)  // [x, z, w]
    
    // ใช้ Set ตรวจ unique
    val words = listOf("apple", "banana", "apple", "cherry", "banana")
    val unique = words.toSet()
    val duplicates = words.groupBy { it }.filter { it.value.size > 1 }.keys
    
    println("Unique: $unique")        // [apple, banana, cherry]
    println("Duplicates: $duplicates") // [apple, banana]
    
    // Set เปรียบเทียบ
    println(setOf(1, 2, 3) == setOf(3, 2, 1))  // true (same elements)
    println(setOf(1, 2, 3) == setOf(1, 2, 4))  // false
}
```

---

## Map

### Immutable Map

```kotlin
fun main() {
    // mapOf() - key-value pairs
    val capitals = mapOf(
        "Thailand" to "Bangkok",
        "Japan" to "Tokyo",
        "UK" to "London",
        "USA" to "Washington D.C."
    )
    
    println(capitals)
    println(capitals.size)  // 4
    
    // Access
    println(capitals["Japan"])           // Tokyo
    println(capitals["Germany"])         // null (ไม่มี key)
    println(capitals.get("Japan"))       // Tokyo
    println(capitals.getOrDefault("Germany", "Unknown"))  // Unknown
    println(capitals.getOrElse("Germany") { "Country not found" })
    
    // Safe access
    val value: String? = capitals["Germany"]
    println(value ?: "Not found")  // Not found
    
    // Keys, Values
    println(capitals.keys)    // [Thailand, Japan, UK, USA]
    println(capitals.values)  // [Bangkok, Tokyo, London, Washington D.C.]
    println(capitals.entries) // [Thailand=Bangkok, ...]
    
    // Iterate
    for ((country, capital) in capitals) {
        println("$country → $capital")
    }
    
    // contains
    println("Japan" in capitals)           // true (check key)
    println(capitals.containsKey("Japan")) // true
    println(capitals.containsValue("Tokyo")) // true
    
    // Map of pairs
    val pairs = listOf("a" to 1, "b" to 2, "c" to 3)
    val fromPairs = pairs.toMap()
    println(fromPairs)  // {a=1, b=2, c=3}
    
    // emptyMap
    val empty = emptyMap<String, Int>()
}
```

### Mutable Map

```kotlin
fun main() {
    val scores = mutableMapOf(
        "Alice" to 85,
        "Bob" to 92,
        "Charlie" to 78
    )
    
    // Add / Update
    scores["Dave"] = 88             // add new
    scores["Alice"] = 90            // update existing
    scores.put("Eve", 95)
    scores.putIfAbsent("Bob", 100)  // ไม่ update ถ้ามีอยู่แล้ว
    
    println(scores)
    
    // Remove
    scores.remove("Charlie")
    scores.remove("Bob", 92)  // ลบเฉพาะถ้า value ตรง
    
    println(scores)
    
    // Merge
    scores.merge("Alice", 5) { old, new -> old + new }  // Alice: 90 + 5 = 95
    println(scores["Alice"])  // 95
    
    // Compute
    scores.compute("Frank") { key, value ->
        (value ?: 0) + 50  // ถ้าไม่มีให้เริ่มที่ 0 แล้วบวก 50
    }
    println(scores["Frank"])  // 50
    
    // putAll
    scores.putAll(mapOf("Grace" to 87, "Henry" to 93))
    
    // forEach
    scores.forEach { (name, score) ->
        println("$name: $score ${if (score >= 90) "A" else "B"}")
    }
    
    // getOrPut (add if absent)
    val wordCount = mutableMapOf<String, Int>()
    val text = "the cat sat on the mat the cat"
    
    for (word in text.split(" ")) {
        wordCount[word] = wordCount.getOrDefault(word, 0) + 1
        // หรือ:
        // wordCount[word] = (wordCount[word] ?: 0) + 1
    }
    println(wordCount)  // {the=3, cat=2, sat=1, on=1, mat=1}
    
    // Sorted Map
    val sorted = sortedMapOf("b" to 2, "a" to 1, "c" to 3)
    println(sorted)  // {a=1, b=2, c=3}
}
```

---

## Collection Operations

### Transformations

```kotlin
fun main() {
    val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    
    // map - transform each element
    val doubled = numbers.map { it * 2 }
    println(doubled)  // [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
    
    // mapIndexed - transform with index
    val indexed = numbers.mapIndexed { i, n -> "$i:$n" }
    println(indexed)  // [0:1, 1:2, 2:3, ...]
    
    // mapNotNull - transform + filter null
    val strings = listOf("1", "two", "3", "four", "5")
    val parsed = strings.mapNotNull { it.toIntOrNull() }
    println(parsed)  // [1, 3, 5]
    
    // flatMap - flatten nested
    val nested = listOf(listOf(1, 2), listOf(3, 4), listOf(5, 6))
    val flat = nested.flatten()
    println(flat)  // [1, 2, 3, 4, 5, 6]
    
    val words = listOf("Hello World", "Kotlin")
    val letters = words.flatMap { it.split(" ") }
    println(letters)  // [Hello, World, Kotlin]
    
    // zip - combine two lists
    val names = listOf("Alice", "Bob", "Charlie")
    val ages = listOf(25, 30, 28)
    
    val zipped = names.zip(ages)
    println(zipped)  // [(Alice, 25), (Bob, 30), (Charlie, 28)]
    
    val withTransform = names.zip(ages) { name, age -> "$name is $age" }
    println(withTransform)  // [Alice is 25, Bob is 30, Charlie is 28]
    
    // unzip
    val (nameList, ageList) = zipped.unzip()
    println(nameList)  // [Alice, Bob, Charlie]
    println(ageList)   // [25, 30, 28]
    
    // associate - List → Map
    val nameToLength = names.associateWith { it.length }
    println(nameToLength)  // {Alice=5, Bob=3, Charlie=7}
    
    val lengthToName = names.associateBy { it.length }
    println(lengthToName)  // {5=Alice, 3=Bob, 7=Charlie}
    
    // groupBy - List → Map<K, List<V>>
    val grouped = numbers.groupBy { if (it % 2 == 0) "even" else "odd" }
    println(grouped)  // {odd=[1, 3, 5, 7, 9], even=[2, 4, 6, 8, 10]}
}
```

### Filtering

```kotlin
fun main() {
    val data = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    
    // filter
    val evens = data.filter { it % 2 == 0 }
    println(evens)  // [2, 4, 6, 8, 10]
    
    // filterNot
    val odds = data.filterNot { it % 2 == 0 }
    println(odds)  // [1, 3, 5, 7, 9]
    
    // filterNotNull
    val withNulls = listOf(1, null, 3, null, 5)
    val withoutNulls: List<Int> = withNulls.filterNotNull()
    println(withoutNulls)  // [1, 3, 5]
    
    // filterIsInstance - filter by type
    val mixed: List<Any> = listOf(1, "hello", 2.0, "world", 3, true)
    val strings: List<String> = mixed.filterIsInstance<String>()
    val ints: List<Int> = mixed.filterIsInstance<Int>()
    println(strings)  // [hello, world]
    println(ints)     // [1, 3]
    
    // partition - split into two lists
    val (even, odd) = data.partition { it % 2 == 0 }
    println("Even: $even")  // Even: [2, 4, 6, 8, 10]
    println("Odd: $odd")    // Odd: [1, 3, 5, 7, 9]
    
    // take / drop
    println(data.take(3))           // [1, 2, 3]
    println(data.takeLast(3))       // [8, 9, 10]
    println(data.drop(3))           // [4, 5, 6, 7, 8, 9, 10]
    println(data.dropLast(3))       // [1, 2, 3, 4, 5, 6, 7]
    println(data.takeWhile { it < 5 })  // [1, 2, 3, 4]
    println(data.dropWhile { it < 5 })  // [5, 6, 7, 8, 9, 10]
    
    // distinct / distinctBy
    val dup = listOf(1, 2, 2, 3, 3, 3, 4)
    println(dup.distinct())  // [1, 2, 3, 4]
    
    data class Person(val name: String, val dept: String)
    val people = listOf(
        Person("Alice", "IT"),
        Person("Bob", "HR"),
        Person("Charlie", "IT"),
        Person("Dave", "Finance")
    )
    val uniqueDepts = people.distinctBy { it.dept }
    println(uniqueDepts.map { it.name })  // [Alice, Bob, Dave]
}
```

### Aggregation

```kotlin
fun main() {
    val nums = listOf(1, 2, 3, 4, 5)
    
    // Basic
    println(nums.sum())        // 15
    println(nums.average())    // 3.0
    println(nums.count())      // 5
    println(nums.min())        // 1
    println(nums.max())        // 5
    
    // sumOf - transform then sum
    data class Product(val name: String, val price: Double, val qty: Int)
    val cart = listOf(
        Product("Apple", 10.0, 3),
        Product("Banana", 5.0, 5),
        Product("Cherry", 25.0, 2)
    )
    val total = cart.sumOf { it.price * it.qty }
    println("Total: $total")  // 105.0
    
    // minBy / maxBy
    val cheapest = cart.minByOrNull { it.price }
    val priciest = cart.maxByOrNull { it.price }
    println("Cheapest: ${cheapest?.name}")  // Banana
    println("Priciest: ${priciest?.name}")  // Cherry
    
    // reduce - เหมือน fold แต่ไม่มี initial value
    val product = nums.reduce { acc, n -> acc * n }  // 1*2*3*4*5 = 120
    println(product)
    
    // fold - พร้อม initial value
    val sumWithStart = nums.fold(100) { acc, n -> acc + n }  // 100+15 = 115
    println(sumWithStart)
    
    // runningFold - intermediate results
    val running = nums.runningFold(0) { acc, n -> acc + n }
    println(running)  // [0, 1, 3, 6, 10, 15]
    
    // any / all / none
    println(nums.any { it > 3 })   // true
    println(nums.all { it > 0 })   // true
    println(nums.none { it < 0 })  // true
    
    // count with predicate
    println(nums.count { it % 2 == 0 })  // 2
    
    // joinToString
    println(nums.joinToString(", "))            // 1, 2, 3, 4, 5
    println(nums.joinToString(" + ") { "$it" }) // 1 + 2 + 3 + 4 + 5
}
```

---

## Destructuring

```kotlin
fun main() {
    // Destructuring Pairs
    val pair = Pair("Kotlin", 2011)
    val (language, year) = pair
    println("$language was created in $year")
    
    // Destructuring data class
    data class Point(val x: Int, val y: Int)
    val point = Point(10, 20)
    val (x, y) = point
    println("x=$x, y=$y")
    
    // Destructuring in forEach
    val map = mapOf("a" to 1, "b" to 2, "c" to 3)
    map.forEach { (key, value) ->
        println("$key = $value")
    }
    
    // Destructuring in for
    val pairs = listOf(Pair("one", 1), Pair("two", 2), Pair("three", 3))
    for ((name, num) in pairs) {
        println("$name → $num")
    }
    
    // Skip components with _
    data class RGB(val r: Int, val g: Int, val b: Int)
    val color = RGB(255, 128, 0)
    val (r, _, b) = color  // skip g
    println("R=$r, B=$b")
    
    // Destructuring return value
    fun getMinMax(list: List<Int>): Pair<Int, Int> =
        Pair(list.min(), list.max())
    
    val (min, max) = getMinMax(listOf(3, 1, 4, 1, 5, 9))
    println("min=$min, max=$max")  // min=1, max=9
    
    // componentN functions
    val triple = Triple(1, "hello", true)
    println(triple.first)   // 1
    println(triple.second)  // hello
    println(triple.third)   // true
    
    val (a, b, c) = triple
    println("$a, $b, $c")  // 1, hello, true
}
```

---

## ตัวอย่างโปรแกรม - Student Grade Management

```kotlin
data class Student(
    val id: String,
    val name: String,
    val scores: Map<String, Double>
)

fun main() {
    val students = listOf(
        Student("001", "สมชาย", mapOf("Math" to 85.0, "Science" to 92.0, "Thai" to 78.0)),
        Student("002", "สมหญิง", mapOf("Math" to 95.0, "Science" to 88.0, "Thai" to 91.0)),
        Student("003", "สมศักดิ์", mapOf("Math" to 72.0, "Science" to 79.0, "Thai" to 85.0)),
        Student("004", "สมพร", mapOf("Math" to 88.0, "Science" to 94.0, "Thai" to 80.0)),
        Student("005", "สมเกียรติ", mapOf("Math" to 65.0, "Science" to 70.0, "Thai" to 75.0))
    )
    
    // Average per student
    val avgPerStudent = students.associate { s ->
        s.name to s.scores.values.average()
    }
    
    println("=== คะแนนเฉลี่ยต่อคน ===")
    avgPerStudent.entries.sortedByDescending { it.value }.forEach { (name, avg) ->
        println("$name: ${"%.1f".format(avg)}")
    }
    
    // Average per subject
    val subjects = students.first().scores.keys
    println("\n=== คะแนนเฉลี่ยต่อวิชา ===")
    subjects.forEach { subject ->
        val avg = students.map { it.scores[subject]!! }.average()
        println("$subject: ${"%.1f".format(avg)}")
    }
    
    // Grade classification
    fun grade(score: Double) = when {
        score >= 90 -> "A"
        score >= 80 -> "B"
        score >= 70 -> "C"
        score >= 60 -> "D"
        else -> "F"
    }
    
    println("\n=== เกรด ===")
    students.forEach { s ->
        val avg = s.scores.values.average()
        println("${s.name}: ${grade(avg)} (${"%.1f".format(avg)})")
    }
    
    // Top student per subject
    println("\n=== อันดับ 1 ต่อวิชา ===")
    subjects.forEach { subject ->
        val top = students.maxByOrNull { it.scores[subject]!! }
        println("$subject: ${top?.name} (${top?.scores?.get(subject)})")
    }
    
    // Students needing improvement (any score < 70)
    val needHelp = students.filter { s ->
        s.scores.values.any { it < 70 }
    }
    
    println("\n=== ต้องปรับปรุง ===")
    if (needHelp.isEmpty()) {
        println("ทุกคนผ่านหมด!")
    } else {
        needHelp.forEach { s ->
            val lowSubjects = s.scores.filter { it.value < 70 }.keys
            println("${s.name}: ต้องปรับปรุง ${lowSubjects.joinToString(", ")}")
        }
    }
    
    // Distribution
    val distribution = students
        .map { s -> grade(s.scores.values.average()) }
        .groupBy { it }
        .mapValues { it.value.size }
        .entries
        .sortedBy { it.key }
    
    println("\n=== การกระจายเกรด ===")
    distribution.forEach { (g, count) ->
        println("$g: ${"|".repeat(count)} ($count คน)")
    }
}
```

---

## แบบฝึกหัด

### Exercise 1: Inventory System

```kotlin
data class Item(val name: String, val category: String, val price: Double, val stock: Int)

fun main() {
    val inventory = listOf(
        Item("Laptop", "Electronics", 25000.0, 10),
        Item("Phone", "Electronics", 15000.0, 25),
        Item("Desk", "Furniture", 5000.0, 5),
        Item("Chair", "Furniture", 2500.0, 15),
        Item("Headphones", "Electronics", 3000.0, 30),
        Item("Bookshelf", "Furniture", 4000.0, 8)
    )
    
    // Total value per category
    println("=== มูลค่าต่อหมวดหมู่ ===")
    inventory
        .groupBy { it.category }
        .mapValues { (_, items) -> items.sumOf { it.price * it.stock } }
        .forEach { (cat, value) ->
            println("$cat: ${"%,.0f".format(value)} บาท")
        }
    
    // Low stock alert
    println("\n=== แจ้งเตือนสินค้าใกล้หมด (น้อยกว่า 10) ===")
    inventory
        .filter { it.stock < 10 }
        .sortedBy { it.stock }
        .forEach { println("${it.name}: ${it.stock} ชิ้น") }
    
    // Most expensive per category
    println("\n=== สินค้าราคาสูงสุดต่อหมวด ===")
    inventory
        .groupBy { it.category }
        .mapValues { (_, items) -> items.maxByOrNull { it.price }!! }
        .forEach { (cat, item) ->
            println("$cat: ${item.name} (${"%,.0f".format(item.price)} บาท)")
        }
}
```

### Exercise 2: Word Frequency Analysis

```kotlin
fun analyzeText(text: String): Map<String, Int> {
    return text.lowercase()
        .replace("[^a-zA-Z\\s]".toRegex(), "")
        .split("\\s+".toRegex())
        .filter { it.isNotEmpty() }
        .groupBy { it }
        .mapValues { it.value.size }
        .entries
        .sortedByDescending { it.value }
        .associate { it.key to it.value }
}

fun main() {
    val text = """
        To be or not to be that is the question
        Whether tis nobler in the mind to suffer
        The slings and arrows of outrageous fortune
        Or to take arms against a sea of troubles
    """.trimIndent()
    
    val freq = analyzeText(text)
    
    println("Top 10 words:")
    freq.entries.take(10).forEachIndexed { i, (word, count) ->
        println("${i + 1}. '$word' → $count times")
    }
}
```

---

## สรุป Part 11

```
✅ listOf() / mutableListOf() - ordered, allows duplicates
✅ setOf() / mutableSetOf() - unique elements
✅ mapOf() / mutableMapOf() - key-value pairs
✅ linkedSetOf() / sortedSetOf() - special Set variants
✅ linkedMapOf() / sortedMapOf() - special Map variants
✅ map, filter, flatMap, groupBy - transformations
✅ reduce, fold, sum, average - aggregations
✅ zip, unzip, associate, associateWith - combining
✅ partition, distinct, take, drop - utility
✅ union, intersect, subtract - Set operations
✅ Destructuring: val (a, b) = pair
```

---

*Part 11/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
