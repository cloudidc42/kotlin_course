# Part 10: Arrays

## สารบัญ
1. [Array พื้นฐาน](#array-พื้นฐาน)
2. [สร้าง Array](#สร้าง-array)
3. [Array Operations](#array-operations)
4. [Multidimensional Arrays](#multidimensional-arrays)
5. [Typed Arrays](#typed-arrays)
6. [Array vs List](#array-vs-list)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Array พื้นฐาน

Array คือโครงสร้างข้อมูลที่เก็บข้อมูลหลายๆ ตัวที่มี **ขนาดคงที่** (fixed size) และ **เปลี่ยนค่าได้** (mutable)

```
Index:  [0]  [1]  [2]  [3]  [4]
Array:  [10] [20] [30] [40] [50]
```

```kotlin
fun main() {
    // สร้าง array ด้วย arrayOf()
    val nums = arrayOf(10, 20, 30, 40, 50)
    
    // เข้าถึงด้วย index
    println(nums[0])    // 10
    println(nums[4])    // 50
    println(nums.last())  // 50
    println(nums.first()) // 10
    
    // แก้ไขค่า (mutable)
    nums[2] = 999
    println(nums[2])    // 999
    
    // ขนาด
    println(nums.size)  // 5
    
    // วนลูป
    for (n in nums) print("$n ")  // 10 20 999 40 50
    println()
    
    // วนด้วย index
    for (i in nums.indices) {
        println("nums[$i] = ${nums[i]}")
    }
    
    // withIndex
    for ((index, value) in nums.withIndex()) {
        println("[$index] = $value")
    }
}
```

---

## สร้าง Array

```kotlin
fun main() {
    // arrayOf - Generic array
    val strings = arrayOf("Hello", "World", "Kotlin")
    val mixed = arrayOf(1, "two", 3.0, true)  // Array<Any>
    
    // emptyArray
    val empty = emptyArray<String>()
    println(empty.size)  // 0
    
    // arrayOfNulls
    val nulls = arrayOfNulls<String>(5)  // Array<String?> ขนาด 5
    println(nulls.contentToString())     // [null, null, null, null, null]
    
    // Array constructor
    val squares = Array(10) { i -> i * i }  // [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
    println(squares.contentToString())
    
    // Array ด้วย formula
    val evens = Array(5) { i -> (i + 1) * 2 }  // [2, 4, 6, 8, 10]
    println(evens.contentToString())
    
    // Array ตัวอักษร
    val letters = Array(26) { i -> ('A' + i) }
    println(letters.contentToString())
    // [A, B, C, D, E, F, G, H, I, J, K, L, M, N, O, P, Q, R, S, T, U, V, W, X, Y, Z]
    
    // Copy
    val original = arrayOf(1, 2, 3, 4, 5)
    val copy = original.copyOf()          // copy ทั้งหมด
    val partial = original.copyOf(3)      // copy 3 ตัวแรก: [1, 2, 3]
    val range = original.copyOfRange(1, 4) // copy index 1-3: [2, 3, 4]
    
    println(copy.contentToString())    // [1, 2, 3, 4, 5]
    println(partial.contentToString()) // [1, 2, 3]
    println(range.contentToString())   // [2, 3, 4]
    
    // fill
    val arr = IntArray(5)  // [0, 0, 0, 0, 0]
    arr.fill(7)
    println(arr.contentToString())  // [7, 7, 7, 7, 7]
    arr.fill(0, 1, 4)  // fill 0 ในตำแหน่ง 1-3
    println(arr.contentToString())  // [7, 0, 0, 0, 7]
}
```

---

## Array Operations

```kotlin
fun main() {
    val nums = arrayOf(3, 1, 4, 1, 5, 9, 2, 6, 5, 3)
    
    // Sorting
    val sorted = nums.sorted()          // List<Int> - ไม่แก้ original
    val sortedDesc = nums.sortedDescending()
    println(sorted)      // [1, 1, 2, 3, 3, 4, 5, 5, 6, 9]
    println(sortedDesc)  // [9, 6, 5, 5, 4, 3, 3, 2, 1, 1]
    
    // sort in place (แก้ array โดยตรง)
    val mutableArr = intArrayOf(3, 1, 4, 1, 5, 9)
    mutableArr.sort()
    println(mutableArr.contentToString())  // [1, 1, 3, 4, 5, 9]
    
    // sort partial
    val partial = intArrayOf(5, 3, 1, 4, 2)
    partial.sort(1, 4)  // sort index 1-3 เท่านั้น
    println(partial.contentToString())  // [5, 1, 3, 4, 2]
    
    // Search
    val data = arrayOf(10, 20, 30, 40, 50)
    println(data.contains(30))           // true
    println(30 in data)                  // true (operator)
    println(data.indexOf(30))            // 2
    println(data.lastIndexOf(30))        // 2
    
    // binarySearch (ต้อง sorted ก่อน)
    val sortedData = intArrayOf(10, 20, 30, 40, 50)
    println(sortedData.binarySearch(30))  // 2
    println(sortedData.binarySearch(25))  // ค่าลบ (ไม่พบ)
    
    // Aggregates
    println(nums.sum())           // 39
    println(nums.average())       // 3.9
    println(nums.min())           // 1
    println(nums.max())           // 9
    println(nums.minOrNull())     // 1 (null-safe)
    println(nums.maxOrNull())     // 9
    
    // Filter, Map (return List ไม่ใช่ Array)
    val filtered = nums.filter { it > 3 }        // List<Int>
    val doubled = nums.map { it * 2 }            // List<Int>
    val sumOfSquares = nums.sumOf { it.toLong() * it }
    
    println(filtered)       // [4, 5, 9, 6, 5]
    println(doubled)        // [6, 2, 8, 2, 10, 18, 4, 12, 10, 6]
    println(sumOfSquares)   // 197
    
    // toList / toSet
    val list = nums.toList()
    val set = nums.toSet()  // ลบ duplicate
    println(set)  // [3, 1, 4, 5, 9, 2, 6]
    
    // Slice
    val slice = nums.slice(2..5)  // List [4, 1, 5, 9]
    println(slice)
    
    // Reverse
    val reversed = nums.reversed()  // List
    println(reversed)  // [3, 5, 6, 2, 9, 5, 1, 4, 1, 3]
    
    // Print array เหมือน Java
    println(nums.contentToString())  // [3, 1, 4, 1, 5, 9, 2, 6, 5, 3]
    // java.util.Arrays.toString(nums) ก็ได้เหมือนกัน
}
```

### Array Equality

```kotlin
fun main() {
    val a = arrayOf(1, 2, 3)
    val b = arrayOf(1, 2, 3)
    val c = a
    
    // == เปรียบเทียบ reference (เหมือน Java)
    println(a == b)   // false (คนละ object)
    println(a == c)   // true (reference เดียวกัน)
    
    // contentEquals เปรียบเทียบเนื้อหา
    println(a.contentEquals(b))   // true ✅
    println(a.contentDeepEquals(b))  // true (nested arrays)
    
    // contentHashCode
    println(a.contentHashCode())  // เท่ากับ b
    println(b.contentHashCode())  // เท่ากัน
}
```

---

## Multidimensional Arrays

```kotlin
fun main() {
    // 2D Array (Matrix)
    val matrix = Array(3) { row ->
        Array(3) { col -> row * 3 + col + 1 }
    }
    
    // Print matrix
    for (row in matrix) {
        println(row.contentToString())
    }
    // [1, 2, 3]
    // [4, 5, 6]
    // [7, 8, 9]
    
    // เข้าถึง element
    println(matrix[1][2])  // 6 (row 1, col 2)
    
    // contentDeepToString สำหรับ nested arrays
    println(matrix.contentDeepToString())
    // [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
    
    // แก้ไขค่า
    matrix[0][0] = 99
    println(matrix[0].contentToString())  // [99, 2, 3]
    
    // 3D Array
    val cube = Array(2) { z ->
        Array(2) { y ->
            Array(2) { x -> z * 4 + y * 2 + x }
        }
    }
    println(cube.contentDeepToString())
    // [[[0, 1], [2, 3]], [[4, 5], [6, 7]]]
    
    // Matrix multiplication
    fun matMul(a: Array<IntArray>, b: Array<IntArray>): Array<IntArray> {
        val n = a.size
        val m = b[0].size
        val k = b.size
        return Array(n) { i ->
            IntArray(m) { j ->
                (0 until k).sumOf { p -> a[i][p] * b[p][j] }
            }
        }
    }
    
    val A = arrayOf(
        intArrayOf(1, 2),
        intArrayOf(3, 4)
    )
    val B = arrayOf(
        intArrayOf(5, 6),
        intArrayOf(7, 8)
    )
    val C = matMul(A, B)
    C.forEach { println(it.contentToString()) }
    // [19, 22]
    // [43, 50]
}
```

---

## Typed Arrays

Kotlin มี primitive-type arrays เพื่อประสิทธิภาพสูง (ไม่ boxing)

```kotlin
fun main() {
    // Primitive Type Arrays (ไม่มี boxing overhead)
    val intArr = intArrayOf(1, 2, 3, 4, 5)          // IntArray
    val longArr = longArrayOf(1L, 2L, 3L)             // LongArray
    val doubleArr = doubleArrayOf(1.0, 2.5, 3.14)     // DoubleArray
    val floatArr = floatArrayOf(1.0f, 2.5f)           // FloatArray
    val boolArr = booleanArrayOf(true, false, true)   // BooleanArray
    val charArr = charArrayOf('a', 'b', 'c')          // CharArray
    val byteArr = byteArrayOf(1, 2, 127)              // ByteArray
    val shortArr = shortArrayOf(100, 200, 300)         // ShortArray
    
    println(intArr.contentToString())    // [1, 2, 3, 4, 5]
    println(doubleArr.contentToString()) // [1.0, 2.5, 3.14]
    println(charArr.contentToString())   // [a, b, c]
    
    // IntArray constructor
    val squares = IntArray(5) { i -> i * i }  // [0, 1, 4, 9, 16]
    println(squares.contentToString())
    
    // ใช้กับ Java APIs
    val byteData = byteArrayOf(72, 101, 108, 108, 111)
    println(String(byteData))  // Hello
    
    // Conversion
    val boxed: Array<Int> = intArr.toTypedArray()    // IntArray → Array<Int>
    val primitive: IntArray = boxed.toIntArray()      // Array<Int> → IntArray
    
    println(boxed.contentToString())     // [1, 2, 3, 4, 5]
    println(primitive.contentToString()) // [1, 2, 3, 4, 5]
    
    // Performance comparison
    // IntArray: stores as int[], เร็วกว่า Array<Int> ซึ่งเป็น Integer[]
    
    // สำหรับ Byte operations (เช่น network, file I/O)
    val buffer = ByteArray(1024) { 0 }  // 1KB buffer
    println("Buffer size: ${buffer.size} bytes")
    
    // Image pixel data
    fun createGrayscaleGradient(width: Int): ByteArray {
        return ByteArray(width) { i -> (i * 255 / width).toByte() }
    }
    val gradient = createGrayscaleGradient(10)
    println(gradient.map { it.toInt() and 0xFF })  // [0, 28, 56, 84, 113, 141, 169, 198, 226, 254]
}
```

---

## Array vs List

```kotlin
fun main() {
    // Array: fixed size, mutable values, ใช้ [][] syntax
    val array = arrayOf(1, 2, 3)
    array[0] = 10        // ✅ แก้ค่าได้
    // array เพิ่ม/ลบ element ไม่ได้โดยตรง
    
    // List: immutable size, ใช้ index function
    val list = listOf(1, 2, 3)
    // list[0] = 10     // ❌ List is read-only
    
    // MutableList
    val mutableList = mutableListOf(1, 2, 3)
    mutableList[0] = 10  // ✅
    mutableList.add(4)   // ✅ เพิ่ม element
    mutableList.remove(2) // ✅ ลบ element
    
    // เมื่อไหร่ใช้ Array:
    // 1. ต้องการ primitive performance (IntArray)
    // 2. ต้องการ fixed size
    // 3. Java interop ที่ต้องการ int[]
    // 4. vararg parameters
    
    // เมื่อไหร่ใช้ List:
    // 1. Functional operations (filter, map, etc.)
    // 2. เพิ่ม/ลบ elements บ่อย
    // 3. Idiomatic Kotlin code
    
    // Array → List
    val arr = arrayOf(1, 2, 3)
    val lst = arr.toList()      // immutable List
    val mut = arr.toMutableList() // mutable List
    
    // List → Array
    val backToArray = lst.toTypedArray()   // Array<Int>
    val intArray = lst.toIntArray()        // IntArray (faster)
    
    // Spread operator * (Array → vararg)
    fun sum(vararg nums: Int) = nums.sum()
    
    val args = intArrayOf(1, 2, 3, 4, 5)
    println(sum(*args))  // 15 (spread IntArray)
    
    val args2 = arrayOf(1, 2, 3, 4, 5)
    println(sum(*args2.toIntArray()))  // 15 (spread Array<Int>)
}
```

---

## ตัวอย่างโปรแกรม - เกมส์จำลอง

### Board Game Grid

```kotlin
fun main() {
    val EMPTY = '.'
    val PLAYER = 'P'
    val ENEMY = 'E'
    val WALL = '#'
    
    val gridSize = 7
    val grid = Array(gridSize) { CharArray(gridSize) { EMPTY } }
    
    // วาง Walls
    for (i in 0 until gridSize) {
        grid[0][i] = WALL
        grid[gridSize - 1][i] = WALL
        grid[i][0] = WALL
        grid[i][gridSize - 1] = WALL
    }
    
    // วาง Player และ Enemy
    grid[1][1] = PLAYER
    grid[5][5] = ENEMY
    grid[3][3] = WALL
    grid[3][4] = WALL
    
    fun printGrid() {
        for (row in grid) {
            println(row.concatToString())
        }
    }
    
    fun findPosition(c: Char): Pair<Int, Int>? {
        for (r in grid.indices) {
            val col = grid[r].indexOf(c)
            if (col >= 0) return Pair(r, col)
        }
        return null
    }
    
    fun movePlayer(dr: Int, dc: Int): Boolean {
        val pos = findPosition(PLAYER) ?: return false
        val (r, c) = pos
        val nr = r + dr
        val nc = c + dc
        
        if (nr !in grid.indices || nc !in grid[0].indices) return false
        if (grid[nr][nc] == WALL) return false
        
        val hitEnemy = grid[nr][nc] == ENEMY
        grid[r][c] = EMPTY
        grid[nr][nc] = PLAYER
        return hitEnemy
    }
    
    println("=== Board Game ===")
    printGrid()
    
    val moves = listOf(
        Pair(1, 0), Pair(1, 0), Pair(1, 0),
        Pair(0, 1), Pair(0, 1), Pair(0, 1), Pair(0, 1)
    )
    
    for ((dr, dc) in moves) {
        val hitEnemy = movePlayer(dr, dc)
        if (hitEnemy) {
            println("\nPlayer hit enemy!")
            break
        }
    }
    
    println("\nAfter moves:")
    printGrid()
}
```

---

## แบบฝึกหัด

### Exercise 1: Rotation

```kotlin
fun rotateRight(arr: IntArray, k: Int): IntArray {
    val n = arr.size
    val shift = k % n
    return IntArray(n) { i -> arr[(i - shift + n) % n] }
}

fun main() {
    val arr = intArrayOf(1, 2, 3, 4, 5)
    println(arr.contentToString())                // [1, 2, 3, 4, 5]
    println(rotateRight(arr, 2).contentToString()) // [4, 5, 1, 2, 3]
    println(rotateRight(arr, 7).contentToString()) // [4, 5, 1, 2, 3]
}
```

### Exercise 2: Statistics

```kotlin
fun statistics(data: DoubleArray): Map<String, Double> {
    val sorted = data.sorted()
    val n = data.size
    val mean = data.average()
    val variance = data.map { (it - mean) * (it - mean) }.average()
    val stdDev = Math.sqrt(variance)
    val median = if (n % 2 == 0)
        (sorted[n / 2 - 1] + sorted[n / 2]) / 2.0
    else
        sorted[n / 2]
    
    return mapOf(
        "mean" to mean,
        "median" to median,
        "stdDev" to stdDev,
        "min" to sorted.first(),
        "max" to sorted.last()
    )
}

fun main() {
    val scores = doubleArrayOf(85.0, 92.0, 78.5, 96.0, 88.0, 74.5, 91.0)
    val stats = statistics(scores)
    
    println("=== Statistics ===")
    stats.forEach { (key, value) ->
        println("$key: ${"%.2f".format(value)}")
    }
}
```

---

## สรุป Part 10

```
✅ arrayOf() สร้าง Array<T>
✅ Array(size) { init } - constructor with lambda
✅ arrayOfNulls<T>(size) - nullable array
✅ IntArray, DoubleArray ฯลฯ - primitive arrays (ไม่มี boxing)
✅ intArrayOf() - primitive array factory
✅ contentToString() / contentDeepToString() - print arrays
✅ contentEquals() - compare arrays
✅ sort(), sorted(), sortedDescending()
✅ binarySearch() - O(log n) search
✅ filter, map, sum ส่งคืน List
✅ toList(), toTypedArray(), toIntArray() - convert
✅ Spread operator *args สำหรับ vararg
✅ Multidimensional arrays ด้วย nested Array
```

---

*Part 10/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
