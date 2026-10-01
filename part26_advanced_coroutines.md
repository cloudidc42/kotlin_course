# Part 26: Advanced Coroutines

## สารบัญ
1. [Channels](#channels)
2. [Actors](#actors)
3. [Select Expression](#select-expression)
4. [Coroutine Context และ CoroutineScope](#coroutine-context-และ-coroutinescope)
5. [Structured Concurrency](#structured-concurrency)
6. [Flow Advanced](#flow-advanced)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Channels

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

fun main() = runBlocking {
    // Channel: communication between coroutines
    // Producer → Channel → Consumer
    
    // Basic Channel
    val channel = Channel<Int>()
    
    launch {
        for (i in 1..5) {
            channel.send(i)
            println("Sent $i")
        }
        channel.close()
    }
    
    launch {
        for (value in channel) {
            println("Received $value")
        }
    }
    
    delay(100)
    println()
    
    // Produce: coroutine builder that creates channel
    val squares = produce {
        for (i in 1..5) {
            send(i * i)
        }
    }
    
    for (sq in squares) {
        println("Square: $sq")
    }
    
    println()
    
    // Fan-out: multiple consumers
    val jobs = produce {
        repeat(10) { send(it) }
    }
    
    val consumer1 = launch {
        for (item in jobs) {
            println("Consumer1: $item")
            delay(10)
        }
    }
    
    val consumer2 = launch {
        for (item in jobs) {
            println("Consumer2: $item")
            delay(10)
        }
    }
    
    consumer1.join()
    consumer2.join()
    
    println()
    
    // Fan-in: merge multiple channels
    fun CoroutineScope.producer(name: String, items: List<String>) = produce {
        items.forEach { item ->
            send("[$name] $item")
            delay(50)
        }
    }
    
    fun CoroutineScope.mergeChannels(vararg channels: ReceiveChannel<String>) = produce {
        channels.forEach { ch ->
            launch {
                for (item in ch) send(item)
            }
        }
    }
    
    val ch1 = producer("A", listOf("apple", "avocado"))
    val ch2 = producer("B", listOf("banana", "blueberry"))
    val ch3 = producer("C", listOf("cherry", "coconut"))
    
    val merged = mergeChannels(ch1, ch2, ch3)
    
    repeat(6) {
        println(merged.receive())
    }
    merged.cancel()
    
    println()
    
    // Buffered channel
    val buffered = Channel<Int>(capacity = 5)
    
    launch {
        repeat(10) { i ->
            buffered.send(i)
            println("Buffered send: $i")
        }
        buffered.close()
    }
    
    delay(50)  // let producer fill buffer
    
    for (v in buffered) {
        println("Buffered recv: $v")
    }
}
```

---

## Actors

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

// Actor = coroutine + mailbox channel
// ใช้สำหรับ thread-safe state management

sealed class CounterMsg {
    object Increment : CounterMsg()
    object Decrement : CounterMsg()
    data class Reset(val value: Int = 0) : CounterMsg()
    class GetValue(val response: CompletableDeferred<Int>) : CounterMsg()
}

fun CoroutineScope.counterActor() = actor<CounterMsg> {
    var counter = 0  // state ภายใน actor
    
    for (msg in channel) {
        when (msg) {
            CounterMsg.Increment       -> counter++
            CounterMsg.Decrement       -> counter--
            is CounterMsg.Reset        -> counter = msg.value
            is CounterMsg.GetValue     -> msg.response.complete(counter)
        }
    }
}

suspend fun CoroutineScope.runCounterDemo() {
    val counter = counterActor()
    
    // Multiple coroutines access safely
    val jobs = (1..1000).map {
        launch {
            counter.send(CounterMsg.Increment)
        }
    }
    
    jobs.forEach { it.join() }
    
    val result = CompletableDeferred<Int>()
    counter.send(CounterMsg.GetValue(result))
    println("Final counter: ${result.await()}")  // 1000
    
    counter.close()
}

// Shopping cart actor (real-world example)
sealed class CartAction {
    data class AddItem(val id: String, val name: String, val price: Double, val qty: Int = 1) : CartAction()
    data class RemoveItem(val id: String) : CartAction()
    data class UpdateQty(val id: String, val qty: Int) : CartAction()
    object Clear : CartAction()
    class GetCart(val response: CompletableDeferred<Map<String, Triple<String, Double, Int>>>) : CartAction()
    class GetTotal(val response: CompletableDeferred<Double>) : CartAction()
}

fun CoroutineScope.cartActor() = actor<CartAction>(capacity = Channel.BUFFERED) {
    val cart = mutableMapOf<String, Triple<String, Double, Int>>()  // id -> (name, price, qty)
    
    for (action in channel) {
        when (action) {
            is CartAction.AddItem -> {
                cart[action.id] = cart[action.id]?.let { (name, price, qty) ->
                    Triple(name, price, qty + action.qty)
                } ?: Triple(action.name, action.price, action.qty)
            }
            is CartAction.RemoveItem -> cart.remove(action.id)
            is CartAction.UpdateQty -> {
                if (action.qty <= 0) cart.remove(action.id)
                else cart[action.id] = cart[action.id]?.copy(third = action.qty)
                    ?: return@actor
            }
            CartAction.Clear -> cart.clear()
            is CartAction.GetCart  -> action.response.complete(cart.toMap())
            is CartAction.GetTotal -> {
                val total = cart.values.sumOf { (_, price, qty) -> price * qty }
                action.response.complete(total)
            }
        }
    }
}

fun main() = runBlocking {
    runCounterDemo()
    
    println()
    
    // Cart actor
    val cart = cartActor()
    
    // Multiple concurrent operations
    launch { cart.send(CartAction.AddItem("P1", "Laptop", 25000.0)) }
    launch { cart.send(CartAction.AddItem("P2", "Mouse", 500.0, 2)) }
    launch { cart.send(CartAction.AddItem("P3", "Keyboard", 1200.0)) }
    
    delay(50)
    
    val cartResult = CompletableDeferred<Map<String, Triple<String, Double, Int>>>()
    cart.send(CartAction.GetCart(cartResult))
    
    val items = cartResult.await()
    println("Cart items:")
    items.forEach { (id, triple) ->
        val (name, price, qty) = triple
        println("  $id: $name x$qty @ ฿$price")
    }
    
    val totalDeferred = CompletableDeferred<Double>()
    cart.send(CartAction.GetTotal(totalDeferred))
    println("Total: ฿${totalDeferred.await()}")
    
    cart.close()
}
```

---

## Select Expression

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*
import kotlinx.coroutines.selects.*

fun main() = runBlocking {
    // select: wait on multiple suspending operations
    
    val ch1 = Channel<String>()
    val ch2 = Channel<String>()
    
    launch {
        delay(100)
        ch1.send("Message from ch1")
    }
    
    launch {
        delay(50)
        ch2.send("Message from ch2")
    }
    
    // Select first that's ready
    val result = select<String> {
        ch1.onReceive { "ch1: $it" }
        ch2.onReceive { "ch2: $it" }
    }
    println("First received: $result")  // ch2 wins (50ms < 100ms)
    
    println()
    
    // Select with timeout
    val slowChannel = Channel<String>()
    
    val timeoutResult = select<String?> {
        slowChannel.onReceive { it }
        onTimeout(100L) { null }
    }
    println("Result with timeout: $timeoutResult")  // null (timeout)
    
    println()
    
    // Select with send
    val outCh1 = Channel<Int>()
    val outCh2 = Channel<Int>()
    
    launch {
        select<Unit> {
            outCh1.onSend(42) { println("Sent to ch1") }
            outCh2.onSend(99) { println("Sent to ch2") }
        }
    }
    
    delay(10)
    println("Receive from ch1: ${outCh1.tryReceive().getOrNull()}")
    println("Receive from ch2: ${outCh2.tryReceive().getOrNull()}")
    
    // Biased select: first clause gets priority
    val biasedCh1 = Channel<Int>()
    val biasedCh2 = Channel<Int>()
    
    biasedCh1.send(1)
    biasedCh2.send(2)
    
    val biased = select<Int> {
        biasedCh1.onReceive { it }  // first: checked first
        biasedCh2.onReceive { it }
    }
    println("Biased result: $biased")  // 1 (ch1 checked first)
    
    ch1.close(); ch2.close(); slowChannel.close()
    outCh1.close(); outCh2.close()
    biasedCh1.close(); biasedCh2.close()
}
```

---

## Structured Concurrency

```kotlin
import kotlinx.coroutines.*

// Structured Concurrency: 
// - Parent coroutine รอ children ทั้งหมดก่อน complete
// - Cancel parent → cancel ทุก children
// - Child failure → propagate to parent (unless SupervisorJob)

fun main() = runBlocking {
    // Basic structured concurrency
    val parentJob = launch {
        println("Parent started")
        
        val child1 = launch {
            println("Child1 started")
            delay(100)
            println("Child1 done")
        }
        
        val child2 = launch {
            println("Child2 started")
            delay(200)
            println("Child2 done")
        }
        
        // Parent waits for both children
        println("Waiting for children...")
    }
    
    parentJob.join()
    println("Parent done")
    
    println()
    
    // Cancel propagation
    val cancelJob = launch {
        val child = launch {
            try {
                delay(1000)
            } catch (e: CancellationException) {
                println("Child cancelled")
                throw e  // must re-throw
            }
        }
        
        delay(100)
        cancel()  // cancel this scope = cancel children too
        println("Parent cancelled itself")
    }
    
    cancelJob.join()
    
    println()
    
    // SupervisorJob: child failure doesn't cancel siblings
    val supervisor = SupervisorJob()
    val scope = CoroutineScope(Dispatchers.Default + supervisor)
    
    val job1 = scope.launch {
        delay(50)
        throw RuntimeException("job1 failed!")
    }
    
    val job2 = scope.launch {
        delay(200)
        println("job2 completed successfully")
    }
    
    delay(300)
    supervisor.cancel()
    
    println()
    
    // supervisorScope block
    try {
        supervisorScope {
            launch {
                delay(50)
                throw RuntimeException("Scoped failure")
            }
            
            launch {
                delay(200)
                println("Still running despite sibling failure")
            }
        }
    } catch (e: Exception) {
        // supervisorScope doesn't catch child exceptions automatically
    }
    
    println()
    
    // coroutineScope vs supervisorScope
    println("coroutineScope: cancels all on first failure")
    try {
        coroutineScope {
            launch {
                delay(50)
                throw RuntimeException("Fatal")
            }
            launch {
                try {
                    delay(200)
                    println("Won't reach here")
                } catch (e: CancellationException) {
                    println("Sibling also cancelled")
                }
            }
        }
    } catch (e: RuntimeException) {
        println("Caught: ${e.message}")
    }
}
```

---

## Flow Advanced

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.*

fun main() = runBlocking {
    // Cold vs Hot flows
    
    // Cold: flow starts fresh for each collector
    val coldFlow = flow {
        println("Flow started")
        emit(1)
        emit(2)
        emit(3)
    }
    
    println("First collection:")
    coldFlow.collect { println(it) }
    println("Second collection:")
    coldFlow.collect { println(it) }
    
    println()
    
    // SharedFlow (hot): broadcast to multiple collectors
    val sharedFlow = MutableSharedFlow<Int>(replay = 2)
    
    sharedFlow.emit(1)
    sharedFlow.emit(2)
    
    val sub1 = launch {
        sharedFlow.collect { println("Sub1: $it") }
    }
    
    val sub2 = launch {
        sharedFlow.collect { println("Sub2: $it") }
    }
    
    delay(50)
    sharedFlow.emit(3)
    sharedFlow.emit(4)
    
    delay(50)
    sub1.cancel()
    sub2.cancel()
    
    println()
    
    // StateFlow: always has a value
    val stateFlow = MutableStateFlow(0)
    
    val observer = launch {
        stateFlow.collect { println("State: $it") }
    }
    
    delay(10)
    stateFlow.value = 1
    delay(10)
    stateFlow.value = 2
    delay(10)
    stateFlow.value = 2  // duplicate, won't emit
    delay(10)
    stateFlow.value = 3
    delay(10)
    
    observer.cancel()
    println()
    
    // Flow operators
    val numbers = flowOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    
    // map, filter, take
    numbers
        .filter { it % 2 == 0 }
        .map { it * it }
        .take(3)
        .collect { println("Even square: $it") }
    
    println()
    
    // flatMapLatest: switch to new flow, cancel previous
    val trigger = flow {
        emit("search1")
        delay(100)
        emit("search2")
    }
    
    trigger
        .flatMapLatest { query ->
            flow {
                println("Searching: $query")
                delay(150)  // simulated search
                emit("Results for $query")
            }
        }
        .collect { println(it) }
    // "search1" may be cancelled by "search2"
    
    println()
    
    // zip: combine two flows
    val names = flowOf("Alice", "Bob", "Charlie")
    val scores = flowOf(90, 85, 95)
    
    names.zip(scores) { name, score -> "$name: $score" }
        .collect { println(it) }
    
    println()
    
    // combine: combine latest from both
    val flow1 = MutableStateFlow("A")
    val flow2 = MutableStateFlow(1)
    
    val combined = combine(flow1, flow2) { letter, number -> "$letter$number" }
    
    val collectJob = launch {
        combined.take(5).collect { println("Combined: $it") }
    }
    
    delay(10)
    flow1.value = "B"
    delay(10)
    flow2.value = 2
    delay(10)
    flow1.value = "C"
    
    collectJob.join()
    
    println()
    
    // catch and onCompletion
    flow {
        emit(1)
        emit(2)
        throw RuntimeException("Flow error")
        emit(3)
    }
    .catch { e -> emit(-1); println("Caught: ${e.message}") }
    .onCompletion { cause ->
        if (cause == null) println("Flow completed normally")
        else println("Flow failed: ${cause.message}")
    }
    .collect { println("Value: $it") }
    
    println()
    
    // buffer, conflate
    val slowFlow = flow {
        repeat(5) { i ->
            emit(i)
            delay(100)
        }
    }
    
    // buffer: producer doesn't wait for slow consumer
    slowFlow
        .buffer(5)
        .collect { value ->
            delay(200)  // slow consumer
            println("Buffered: $value")
        }
    
    println()
    
    // conflate: drop middle values if consumer can't keep up
    val timedFlow = flow {
        repeat(10) { i ->
            emit(i)
            delay(10)
        }
    }
    
    timedFlow
        .conflate()
        .collect { value ->
            delay(50)
            println("Conflated: $value")
        }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Pipeline ด้วย Channels

import kotlinx.coroutines.*
import kotlinx.coroutines.channels.*

data class Record(val id: Int, val raw: String)
data class Processed(val id: Int, val value: Double)
data class Result(val id: Int, val value: Double, val category: String)

fun CoroutineScope.generate(count: Int): ReceiveChannel<Record> = produce {
    repeat(count) { i ->
        val raw = "${(1..100).random()}.${(1..99).random()}"
        send(Record(i, raw))
    }
}

fun CoroutineScope.parse(input: ReceiveChannel<Record>): ReceiveChannel<Processed?> = produce {
    for (record in input) {
        val value = record.raw.toDoubleOrNull()
        send(if (value != null) Processed(record.id, value) else null)
    }
}

fun CoroutineScope.categorize(input: ReceiveChannel<Processed?>): ReceiveChannel<Result> = produce {
    for (item in input) {
        if (item == null) continue
        val category = when {
            item.value < 30  -> "Low"
            item.value < 70  -> "Medium"
            else             -> "High"
        }
        send(Result(item.id, item.value, category))
    }
}

fun main() = runBlocking {
    val generated = generate(20)
    val parsed = parse(generated)
    val categorized = categorize(parsed)
    
    val summary = mutableMapOf("Low" to 0, "Medium" to 0, "High" to 0)
    
    for (result in categorized) {
        summary[result.category] = (summary[result.category] ?: 0) + 1
    }
    
    println("Pipeline summary:")
    summary.forEach { (cat, count) -> println("  $cat: $count") }
}
```

---

## สรุป Part 26

```
✅ Channel: communication conduit between coroutines
✅ produce { }: coroutine builder ที่ return ReceiveChannel
✅ actor { }: coroutine + mailbox สำหรับ thread-safe state
✅ Fan-out: multiple consumers on one channel
✅ Fan-in: merge multiple channels
✅ select { onReceive/onSend/onTimeout }: wait on multiple
✅ Structured Concurrency: parent รอ children
✅ SupervisorJob/supervisorScope: child failure ไม่ cancel siblings
✅ SharedFlow: hot, broadcast to multiple collectors
✅ StateFlow: hot, always has current value
✅ flatMapLatest: cancel previous flow on new emission
✅ combine: combine latest values from multiple flows
✅ buffer/conflate: control backpressure
```

---

*Part 26/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
