# Part 21: Testing ด้วย JUnit และ Kotest

## สารบัญ
1. [Testing พื้นฐาน](#testing-พื้นฐาน)
2. [JUnit 5 กับ Kotlin](#junit-5-กับ-kotlin)
3. [Kotest](#kotest)
4. [Mocking ด้วย MockK](#mocking-ด้วย-mockk)
5. [Test Coroutines](#test-coroutines)
6. [TDD - Test Driven Development](#tdd---test-driven-development)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Testing พื้นฐาน

### Setup

```kotlin
// build.gradle.kts
dependencies {
    // JUnit 5
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.1")
    testImplementation(kotlin("test"))
    
    // Kotest
    testImplementation("io.kotest:kotest-runner-junit5:5.8.0")
    testImplementation("io.kotest:kotest-assertions-core:5.8.0")
    testImplementation("io.kotest:kotest-property:5.8.0")
    
    // MockK
    testImplementation("io.mockk:mockk:1.13.9")
    
    // Coroutines testing
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
}

tasks.test {
    useJUnitPlatform()
}
```

---

## JUnit 5 กับ Kotlin

```kotlin
import org.junit.jupiter.api.*
import org.junit.jupiter.api.Assertions.*
import org.junit.jupiter.params.ParameterizedTest
import org.junit.jupiter.params.provider.*

// System under test
class Calculator {
    fun add(a: Int, b: Int) = a + b
    fun subtract(a: Int, b: Int) = a - b
    fun multiply(a: Int, b: Int) = a * b
    fun divide(a: Int, b: Int): Int {
        if (b == 0) throw ArithmeticException("Cannot divide by zero")
        return a / b
    }
    fun factorial(n: Int): Long {
        require(n >= 0) { "Factorial of negative number" }
        return if (n == 0) 1L else n.toLong() * factorial(n - 1)
    }
}

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class CalculatorTest {
    private val calc = Calculator()
    
    // @BeforeAll - รันก่อนทุก test ใน class (ต้องมี @TestInstance)
    @BeforeAll
    fun setup() {
        println("Setting up test suite")
    }
    
    // @BeforeEach - รันก่อนแต่ละ test
    @BeforeEach
    fun beforeEach(testInfo: TestInfo) {
        println("Starting: ${testInfo.displayName}")
    }
    
    @AfterEach
    fun afterEach() {
        println("Test completed")
    }
    
    @AfterAll
    fun teardown() {
        println("Tearing down test suite")
    }
    
    @Test
    @DisplayName("เพิ่มตัวเลขสองตัว")
    fun testAdd() {
        assertEquals(5, calc.add(2, 3))
        assertEquals(-1, calc.add(-3, 2))
        assertEquals(0, calc.add(0, 0))
    }
    
    @Test
    fun testSubtract() {
        assertEquals(1, calc.subtract(3, 2))
        assertEquals(-5, calc.subtract(-3, 2))
    }
    
    @Test
    fun testMultiply() {
        assertEquals(6, calc.multiply(2, 3))
        assertEquals(0, calc.multiply(5, 0))
        assertEquals(-6, calc.multiply(-2, 3))
    }
    
    @Test
    fun testDivide() {
        assertEquals(4, calc.divide(8, 2))
        assertEquals(3, calc.divide(9, 3))
    }
    
    @Test
    fun testDivideByZeroThrows() {
        val exception = assertThrows<ArithmeticException> {
            calc.divide(5, 0)
        }
        assertEquals("Cannot divide by zero", exception.message)
    }
    
    @Test
    fun testMultipleAssertions() {
        assertAll("Calculator operations",
            { assertEquals(5, calc.add(2, 3)) },
            { assertEquals(1, calc.subtract(3, 2)) },
            { assertEquals(6, calc.multiply(2, 3)) }
        )
    }
    
    // Parameterized test
    @ParameterizedTest
    @CsvSource(
        "0, 1",
        "1, 1",
        "2, 2",
        "3, 6",
        "4, 24",
        "5, 120"
    )
    fun testFactorial(n: Int, expected: Long) {
        assertEquals(expected, calc.factorial(n))
    }
    
    @ParameterizedTest
    @ValueSource(ints = [-1, -5, -100])
    fun testFactorialNegativeThrows(n: Int) {
        assertThrows<IllegalArgumentException> {
            calc.factorial(n)
        }
    }
    
    @ParameterizedTest
    @MethodSource("additionProvider")
    fun testAddWithMethod(a: Int, b: Int, expected: Int) {
        assertEquals(expected, calc.add(a, b))
    }
    
    companion object {
        @JvmStatic
        fun additionProvider() = listOf(
            Arguments.of(1, 2, 3),
            Arguments.of(5, 5, 10),
            Arguments.of(-1, 1, 0)
        )
    }
    
    @Test
    @Disabled("Pending implementation")
    fun testSkipped() {
        // ยังไม่ implement
    }
    
    @Test
    @Timeout(5)  // seconds
    fun testWithTimeout() {
        assertEquals(10, calc.add(4, 6))
    }
}
```

---

## Kotest

```kotlin
import io.kotest.core.spec.style.*
import io.kotest.matchers.*
import io.kotest.matchers.collections.*
import io.kotest.matchers.string.*
import io.kotest.matchers.types.*

// StringSpec style
class StringSpecTest : StringSpec({
    "สตริงไม่ควรเป็น null" {
        val s = "Hello"
        s shouldNotBe null
    }
    
    "สตริงมีความยาวถูกต้อง" {
        "Hello".length shouldBe 5
    }
    
    "สตริงประกอบด้วย substring" {
        "Hello World" shouldContain "World"
    }
})

// BehaviorSpec style (BDD)
class UserServiceSpec : BehaviorSpec({
    given("User service ที่มีผู้ใช้หลายคน") {
        val service = UserService()
        service.addUser("Alice", "alice@email.com")
        service.addUser("Bob", "bob@email.com")
        
        `when`("ค้นหาผู้ใช้ที่มีอยู่") {
            val user = service.findByName("Alice")
            
            then("ควรพบผู้ใช้") {
                user shouldNotBe null
                user?.name shouldBe "Alice"
            }
        }
        
        `when`("ค้นหาผู้ใช้ที่ไม่มีอยู่") {
            val user = service.findByName("Charlie")
            
            then("ควร return null") {
                user shouldBe null
            }
        }
    }
})

// DescribeSpec style
class CalculatorSpec : DescribeSpec({
    describe("Calculator") {
        val calc = Calculator()
        
        describe("addition") {
            it("ควรบวกเลขบวกได้ถูกต้อง") {
                calc.add(2, 3) shouldBe 5
            }
            
            it("ควรบวกเลขลบได้ถูกต้อง") {
                calc.add(-2, -3) shouldBe -5
            }
        }
        
        describe("division") {
            it("ควรหารได้ถูกต้อง") {
                calc.divide(10, 2) shouldBe 5
            }
            
            it("ควร throw exception เมื่อหารด้วยศูนย์") {
                shouldThrow<ArithmeticException> {
                    calc.divide(10, 0)
                }
            }
        }
    }
})

// FunSpec with extensive matchers
class MatcherDemo : FunSpec({
    test("Number matchers") {
        val n = 42
        n shouldBe 42
        n shouldNotBe 43
        n shouldBeGreaterThan 40
        n shouldBeLessThan 50
        n shouldBeGreaterThanOrEqualTo 42
        n shouldBeBetween(40, 50)
    }
    
    test("String matchers") {
        val s = "Hello, Kotlin!"
        s shouldStartWith "Hello"
        s shouldEndWith "Kotlin!"
        s shouldContain "Kotlin"
        s shouldHaveLength 14
        s shouldMatch "Hello.*"
        s.shouldBeLowerCase().not()  // not lowercase
        "HELLO".shouldBeUpperCase()
    }
    
    test("Collection matchers") {
        val list = listOf(1, 2, 3, 4, 5)
        list shouldContain 3
        list shouldContainAll listOf(1, 3, 5)
        list shouldHaveSize 5
        list.shouldBeSorted()
        list.shouldNotBeEmpty()
        
        val set = setOf("a", "b", "c")
        set.shouldContainExactlyInAnyOrder("c", "a", "b")
    }
    
    test("Null matchers") {
        val nullable: String? = null
        nullable.shouldBeNull()
        
        val notNull: String? = "hello"
        notNull.shouldNotBeNull()
        notNull shouldBe "hello"
    }
    
    test("Type matchers") {
        val obj: Any = "Hello"
        obj.shouldBeInstanceOf<String>()
        obj.shouldNotBeInstanceOf<Int>()
    }
})
```

---

## Mocking ด้วย MockK

```kotlin
import io.mockk.*
import io.kotest.core.spec.style.FunSpec
import io.kotest.matchers.shouldBe

// Interfaces to mock
interface UserRepository {
    fun findById(id: Int): User?
    fun save(user: User): User
    fun delete(id: Int): Boolean
    fun findAll(): List<User>
}

interface EmailService {
    fun sendWelcomeEmail(email: String, name: String): Boolean
    fun sendNotification(email: String, message: String)
}

data class User(val id: Int, val name: String, val email: String)

class UserService(
    private val userRepo: UserRepository,
    private val emailService: EmailService
) {
    fun createUser(name: String, email: String): User {
        val user = User(System.currentTimeMillis().toInt(), name, email)
        val saved = userRepo.save(user)
        emailService.sendWelcomeEmail(email, name)
        return saved
    }
    
    fun getUserById(id: Int): User {
        return userRepo.findById(id) ?: throw NoSuchElementException("User $id not found")
    }
    
    fun deleteUser(id: Int): Boolean {
        val user = userRepo.findById(id) ?: return false
        return userRepo.delete(id)
    }
    
    fun getAllUsers(): List<User> = userRepo.findAll()
}

class UserServiceTest : FunSpec({
    // Create mocks
    val userRepo = mockk<UserRepository>()
    val emailService = mockk<EmailService>(relaxed = true)  // relaxed = auto-stub void returns
    val service = UserService(userRepo, emailService)
    
    afterTest { clearAllMocks() }
    
    test("createUser ควรบันทึก user และส่ง email") {
        val user = User(1, "สมชาย", "somchai@email.com")
        
        // stub
        every { userRepo.save(any()) } returns user
        every { emailService.sendWelcomeEmail(any(), any()) } returns true
        
        // act
        val result = service.createUser("สมชาย", "somchai@email.com")
        
        // assert
        result.name shouldBe "สมชาย"
        
        // verify
        verify { userRepo.save(any()) }
        verify { emailService.sendWelcomeEmail("somchai@email.com", "สมชาย") }
    }
    
    test("getUserById ควรคืน user ที่มีอยู่") {
        val user = User(1, "สมชาย", "somchai@email.com")
        every { userRepo.findById(1) } returns user
        
        val result = service.getUserById(1)
        result shouldBe user
    }
    
    test("getUserById ควร throw เมื่อไม่พบ user") {
        every { userRepo.findById(999) } returns null
        
        shouldThrow<NoSuchElementException> {
            service.getUserById(999)
        }
    }
    
    test("deleteUser ควร return false เมื่อ user ไม่มีอยู่") {
        every { userRepo.findById(99) } returns null
        
        val result = service.deleteUser(99)
        result shouldBe false
        verify(exactly = 0) { userRepo.delete(any()) }
    }
    
    test("deleteUser ควรลบ user ที่มีอยู่") {
        val user = User(1, "สมชาย", "somchai@email.com")
        every { userRepo.findById(1) } returns user
        every { userRepo.delete(1) } returns true
        
        val result = service.deleteUser(1)
        result shouldBe true
        
        verifyOrder {
            userRepo.findById(1)
            userRepo.delete(1)
        }
    }
    
    test("getAllUsers ด้วย argument capture") {
        val captor = slot<User>()
        every { userRepo.save(capture(captor)) } answers { captor.captured }
        every { emailService.sendWelcomeEmail(any(), any()) } returns true
        
        service.createUser("สมหญิง", "somying@email.com")
        
        captor.captured.name shouldBe "สมหญิง"
        captor.captured.email shouldBe "somying@email.com"
    }
    
    test("spy: stub บางส่วนของ real object") {
        val realCalc = Calculator()
        val spy = spyk(realCalc)
        
        // override add, ใช้ real multiply
        every { spy.add(any(), any()) } returns 999
        
        spy.add(1, 2) shouldBe 999  // stubbed
        spy.multiply(3, 4) shouldBe 12  // real
        
        verify { spy.add(1, 2) }
    }
})
```

---

## Test Coroutines

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.test.*
import io.kotest.core.spec.style.FunSpec
import io.kotest.matchers.shouldBe

class CoroutineService {
    suspend fun fetchData(id: Int): String {
        delay(1000)
        return "Data #$id"
    }
    
    suspend fun processItems(items: List<Int>): List<String> = coroutineScope {
        items.map { id ->
            async { fetchData(id) }
        }.awaitAll()
    }
}

class CoroutineServiceTest : FunSpec({
    test("fetchData ควรคืนข้อมูลถูกต้อง") {
        val service = CoroutineService()
        
        // runTest: replace delay with virtual time
        runTest {
            val result = service.fetchData(42)
            result shouldBe "Data #42"
        }
    }
    
    test("processItems ควรทำงานแบบ concurrent") {
        val service = CoroutineService()
        
        runTest {
            val start = currentTime
            val results = service.processItems(listOf(1, 2, 3))
            val elapsed = currentTime - start
            
            results shouldBe listOf("Data #1", "Data #2", "Data #3")
            // Concurrent: all 3 run together, total ~1000ms not 3000ms
            elapsed shouldBe 1000  // virtual time
        }
    }
    
    test("Flow testing") {
        val service = CoroutineService()
        
        runTest {
            val flow = kotlinx.coroutines.flow.flow {
                repeat(5) { i ->
                    delay(100)
                    emit(i)
                }
            }
            
            val results = mutableListOf<Int>()
            flow.collect { results.add(it) }
            
            results shouldBe listOf(0, 1, 2, 3, 4)
        }
    }
    
    test("TestCoroutineScheduler สำหรับ time control") {
        val scheduler = TestCoroutineScheduler()
        val scope = TestScope(scheduler)
        
        var callCount = 0
        
        scope.launch {
            delay(1000)
            callCount++
        }
        
        callCount shouldBe 0
        scheduler.advanceTimeBy(500)
        callCount shouldBe 0
        scheduler.advanceTimeBy(600)
        callCount shouldBe 1
    }
})
```

---

## TDD - Test Driven Development

```kotlin
// TDD: Red → Green → Refactor

// Step 1: Write failing test (RED)
class BankAccountTest : FunSpec({
    test("ยอดเงินเริ่มต้นควรเป็น 0") {
        val account = BankAccount()
        account.balance shouldBe 0.0
    }
    
    test("ฝากเงิน 100 ควรทำให้ยอดเป็น 100") {
        val account = BankAccount()
        account.deposit(100.0)
        account.balance shouldBe 100.0
    }
    
    test("ถอนเงินถ้ามียอดพอ ควรลดยอด") {
        val account = BankAccount(initialBalance = 500.0)
        account.withdraw(200.0)
        account.balance shouldBe 300.0
    }
    
    test("ถอนเกินยอด ควร throw InsufficientFundsException") {
        val account = BankAccount(initialBalance = 100.0)
        shouldThrow<InsufficientFundsException> {
            account.withdraw(200.0)
        }
    }
    
    test("ฝากเงินติดลบ ควร throw IllegalArgumentException") {
        val account = BankAccount()
        shouldThrow<IllegalArgumentException> {
            account.deposit(-50.0)
        }
    }
    
    test("โอนเงินระหว่าง accounts") {
        val from = BankAccount(initialBalance = 1000.0)
        val to = BankAccount(initialBalance = 0.0)
        
        from.transfer(300.0, to)
        
        from.balance shouldBe 700.0
        to.balance shouldBe 300.0
    }
})

// Step 2: Implement to make tests pass (GREEN)
class InsufficientFundsException(amount: Double, balance: Double) 
    : Exception("Cannot withdraw ${amount}, only ${balance} available")

class BankAccount(initialBalance: Double = 0.0) {
    var balance: Double = initialBalance
        private set
    
    fun deposit(amount: Double) {
        require(amount > 0) { "Deposit amount must be positive" }
        balance += amount
    }
    
    fun withdraw(amount: Double) {
        require(amount > 0) { "Withdrawal amount must be positive" }
        if (amount > balance) throw InsufficientFundsException(amount, balance)
        balance -= amount
    }
    
    fun transfer(amount: Double, to: BankAccount) {
        withdraw(amount)
        to.deposit(amount)
    }
}

// Step 3: Refactor (REFACTOR) - code is clean already
// Run tests again to verify nothing broke
```

---

## Property-based Testing

```kotlin
import io.kotest.property.*
import io.kotest.property.arbitrary.*

class PropertyBasedTest : FunSpec({
    test("บวกเลขใดๆ กับ 0 ควรได้เลขเดิม") {
        forAll(Arb.int()) { n ->
            n + 0 == n
        }
    }
    
    test("การบวกควรเป็น commutative (a+b = b+a)") {
        forAll(Arb.int(-1000..1000), Arb.int(-1000..1000)) { a, b ->
            a + b == b + a
        }
    }
    
    test("reverse สองครั้งควรได้ string เดิม") {
        forAll(Arb.string()) { s ->
            s.reversed().reversed() == s
        }
    }
    
    test("sort ควร idempotent") {
        forAll(Arb.list(Arb.int())) { list ->
            list.sorted() == list.sorted().sorted()
        }
    }
    
    test("factorial ควรเป็น positive สำหรับ non-negative numbers") {
        val calc = Calculator()
        forAll(Arb.int(0..15)) { n ->
            calc.factorial(n) > 0
        }
    }
    
    // Custom arbitrary
    val positiveInt = Arb.int(1..Int.MAX_VALUE)
    val nonEmptyString = Arb.string(1..100).filter { it.isNotBlank() }
    
    test("ความยาว string หลัง uppercase = ก่อน uppercase") {
        forAll(nonEmptyString) { s ->
            s.uppercase().length == s.length
        }
    }
})
```

---

## แบบฝึกหัด

### Exercise: Testing a Shopping Cart

```kotlin
// ออกแบบ tests สำหรับ ShoppingCart
data class CartItem(val productId: Int, val name: String, val price: Double, val qty: Int)

class ShoppingCart {
    private val items = mutableMapOf<Int, CartItem>()
    
    fun addItem(productId: Int, name: String, price: Double, qty: Int = 1) {
        require(price > 0) { "Price must be positive" }
        require(qty > 0) { "Quantity must be positive" }
        
        items[productId] = items[productId]?.copy(qty = items[productId]!!.qty + qty)
            ?: CartItem(productId, name, price, qty)
    }
    
    fun removeItem(productId: Int) = items.remove(productId) != null
    
    fun updateQty(productId: Int, qty: Int) {
        require(qty >= 0) { "Quantity cannot be negative" }
        if (qty == 0) removeItem(productId)
        else items[productId] = items[productId]?.copy(qty = qty)
            ?: throw NoSuchElementException("Item $productId not in cart")
    }
    
    fun total() = items.values.sumOf { it.price * it.qty }
    fun itemCount() = items.values.sumOf { it.qty }
    fun isEmpty() = items.isEmpty()
    fun clear() = items.clear()
    fun getItems() = items.values.toList()
}

class ShoppingCartTest : FunSpec({
    lateinit var cart: ShoppingCart
    
    beforeTest { cart = ShoppingCart() }
    
    test("cart ใหม่ควรว่างเปล่า") {
        cart.isEmpty() shouldBe true
        cart.total() shouldBe 0.0
        cart.itemCount() shouldBe 0
    }
    
    test("เพิ่มสินค้า 1 รายการ") {
        cart.addItem(1, "Laptop", 25000.0)
        
        cart.isEmpty() shouldBe false
        cart.itemCount() shouldBe 1
        cart.total() shouldBe 25000.0
    }
    
    test("เพิ่มสินค้าเดิมควรบวก qty") {
        cart.addItem(1, "Laptop", 25000.0, 2)
        cart.addItem(1, "Laptop", 25000.0, 1)
        
        cart.itemCount() shouldBe 3
        cart.total() shouldBe 75000.0
    }
    
    test("ลบสินค้าออก") {
        cart.addItem(1, "Laptop", 25000.0)
        cart.addItem(2, "Phone", 15000.0)
        
        cart.removeItem(1) shouldBe true
        cart.itemCount() shouldBe 1
        cart.total() shouldBe 15000.0
    }
    
    test("ลบสินค้าที่ไม่มีควร return false") {
        cart.removeItem(999) shouldBe false
    }
    
    test("เพิ่มราคาติดลบควร throw exception") {
        shouldThrow<IllegalArgumentException> {
            cart.addItem(1, "Test", -100.0)
        }
    }
    
    test("คำนวณ total หลายรายการ") {
        cart.addItem(1, "Item A", 100.0, 3)
        cart.addItem(2, "Item B", 200.0, 2)
        cart.addItem(3, "Item C", 50.0, 5)
        
        cart.total() shouldBe 1050.0  // 300 + 400 + 250
    }
})
```

---

## สรุป Part 21

```
✅ JUnit 5: @Test, @BeforeEach, @AfterEach, @Disabled
✅ Assertions: assertEquals, assertThrows, assertAll
✅ Parameterized: @ParameterizedTest, @CsvSource, @ValueSource
✅ Kotest: StringSpec, BehaviorSpec, DescribeSpec, FunSpec
✅ Kotest Matchers: shouldBe, shouldContain, shouldThrow
✅ MockK: mockk(), every { }, verify { }, slot { }
✅ spyk(): spy real objects with some stubs
✅ relaxed = true: auto-stub void/Unit returns
✅ runTest: test coroutines with virtual time
✅ advanceTimeBy(): control virtual time
✅ Property-based testing: forAll { }
✅ Arb.int(), Arb.string(): random generators
✅ TDD: Red → Green → Refactor cycle
```

---

*Part 21/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
