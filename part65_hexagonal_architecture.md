# Part 65: Hexagonal Architecture (Ports and Adapters)

## สารบัญ
1. [หลักการ Hexagonal Architecture](#หลักการ-hexagonal-architecture)
2. [Domain Layer](#domain-layer)
3. [Application Layer (Use Cases)](#application-layer-use-cases)
4. [Ports: Inbound และ Outbound](#ports-inbound-และ-outbound)
5. [Adapters: REST, Database, Messaging](#adapters-rest-database-messaging)
6. [Dependency Injection](#dependency-injection)
7. [Testing Hexagonal Architecture](#testing-hexagonal-architecture)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## หลักการ Hexagonal Architecture

```
Hexagonal Architecture (เรียกอีกชื่อว่า Ports and Adapters) มีหลักการดังนี้:

1. Domain (Core) อยู่ตรงกลาง — ไม่รู้จัก frameworks, databases, HTTP
2. Ports: interface ที่ domain กำหนด
   - Inbound Ports (Driving): วิธีการ interact กับ domain (Use Cases)
   - Outbound Ports (Driven): สิ่งที่ domain ต้องการจากภายนอก
3. Adapters: implementation ของ Ports
   - Inbound Adapters: REST Controllers, gRPC, CLI, Tests
   - Outbound Adapters: JPA Repositories, Kafka Producers, Email Service

โครงสร้างโปรเจค:
src/
  ├── domain/
  │   ├── model/          - Value Objects, Entities, Aggregates
  │   ├── port/
  │   │   ├── inbound/    - Use Case interfaces
  │   │   └── outbound/   - Repository/Service interfaces
  │   └── service/        - Domain Services
  ├── application/
  │   └── usecase/        - Use Case implementations
  └── adapter/
      ├── inbound/
      │   ├── rest/        - REST Controllers
      │   └── messaging/   - Kafka Consumers
      └── outbound/
          ├── persistence/ - JPA Repositories
          └── messaging/   - Kafka Producers
```

---

## Domain Layer

```kotlin
package com.example.banking.domain.model

// Value Objects
@JvmInline
value class AccountId(val value: String) {
    init { require(value.isNotBlank()) { "AccountId cannot be blank" } }
    override fun toString() = value
}

@JvmInline
value class CustomerId(val value: String)

data class Money(val amount: java.math.BigDecimal, val currency: Currency) {
    
    constructor(amount: Double, currency: Currency) : this(amount.toBigDecimal(), currency)
    
    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Cannot add different currencies: $currency + ${other.currency}" }
        return Money(amount + other.amount, currency)
    }
    
    operator fun minus(other: Money): Money {
        require(currency == other.currency) { "Cannot subtract different currencies" }
        return Money(amount - other.amount, currency)
    }
    
    operator fun compareTo(other: Money): Int {
        require(currency == other.currency) { "Cannot compare different currencies" }
        return amount.compareTo(other.amount)
    }
    
    fun isPositive() = amount > java.math.BigDecimal.ZERO
    fun isNegative() = amount < java.math.BigDecimal.ZERO
    
    override fun toString() = "$amount $currency"
    
    companion object {
        fun zero(currency: Currency) = Money(java.math.BigDecimal.ZERO, currency)
    }
}

enum class Currency { THB, USD, EUR }

data class AccountHolder(
    val customerId: CustomerId,
    val name: String,
    val email: String
)

// Domain Events
sealed class AccountEvent {
    data class AccountOpened(
        val accountId: AccountId,
        val holder: AccountHolder,
        val initialBalance: Money,
        val occurredAt: java.time.Instant = java.time.Instant.now()
    ) : AccountEvent()
    
    data class MoneyDeposited(
        val accountId: AccountId,
        val amount: Money,
        val newBalance: Money,
        val occurredAt: java.time.Instant = java.time.Instant.now()
    ) : AccountEvent()
    
    data class MoneyWithdrawn(
        val accountId: AccountId,
        val amount: Money,
        val newBalance: Money,
        val occurredAt: java.time.Instant = java.time.Instant.now()
    ) : AccountEvent()
    
    data class MoneyTransferred(
        val fromAccountId: AccountId,
        val toAccountId: AccountId,
        val amount: Money,
        val occurredAt: java.time.Instant = java.time.Instant.now()
    ) : AccountEvent()
}

// Domain Exceptions
sealed class AccountDomainException(message: String) : Exception(message) {
    class InsufficientFunds(accountId: AccountId, required: Money, available: Money) :
        AccountDomainException("Account $accountId has insufficient funds: required $required, available $available")
    
    class AccountNotActive(accountId: AccountId) :
        AccountDomainException("Account $accountId is not active")
    
    class InvalidAmount(reason: String) :
        AccountDomainException("Invalid amount: $reason")
}

// Account Aggregate — บริสุทธิ์ ไม่มีการอ้างอิง frameworks ใดๆ
class Account private constructor(
    val id: AccountId,
    val holder: AccountHolder,
    private var balance: Money,
    private var status: AccountStatus,
    private val _domainEvents: MutableList<AccountEvent> = mutableListOf()
) {
    val domainEvents: List<AccountEvent> get() = _domainEvents.toList()
    
    fun currentBalance(): Money = balance
    
    fun deposit(amount: Money): Account {
        require(amount.isPositive()) { throw AccountDomainException.InvalidAmount("Deposit amount must be positive") }
        requireActive()
        
        balance = balance + amount
        _domainEvents.add(AccountEvent.MoneyDeposited(id, amount, balance))
        return this
    }
    
    fun withdraw(amount: Money): Account {
        require(amount.isPositive()) { throw AccountDomainException.InvalidAmount("Withdrawal amount must be positive") }
        requireActive()
        
        if (balance < amount) throw AccountDomainException.InsufficientFunds(id, amount, balance)
        
        balance = balance - amount
        _domainEvents.add(AccountEvent.MoneyWithdrawn(id, amount, balance))
        return this
    }
    
    fun clearDomainEvents(): Account {
        _domainEvents.clear()
        return this
    }
    
    private fun requireActive() {
        if (status != AccountStatus.ACTIVE) throw AccountDomainException.AccountNotActive(id)
    }
    
    companion object {
        fun open(holder: AccountHolder, initialDeposit: Money): Account {
            require(initialDeposit.isPositive()) { 
                throw AccountDomainException.InvalidAmount("Initial deposit must be positive") 
            }
            
            val id = AccountId(java.util.UUID.randomUUID().toString())
            val account = Account(id, holder, initialDeposit, AccountStatus.ACTIVE)
            
            account._domainEvents.add(
                AccountEvent.AccountOpened(id, holder, initialDeposit)
            )
            
            return account
        }
        
        fun reconstitute(
            id: AccountId,
            holder: AccountHolder,
            balance: Money,
            status: AccountStatus
        ) = Account(id, holder, balance, status)
    }
}

enum class AccountStatus { ACTIVE, SUSPENDED, CLOSED }
```

---

## Application Layer (Use Cases)

```kotlin
package com.example.banking.application.usecase

// Inbound Port: what the application can do
interface OpenAccountUseCase {
    fun openAccount(command: OpenAccountCommand): OpenAccountResult
}

interface TransferMoneyUseCase {
    fun transferMoney(command: TransferMoneyCommand): TransferMoneyResult
}

interface GetAccountUseCase {
    fun getAccount(query: GetAccountQuery): GetAccountResult
}

// Commands and Results (Application Layer DTOs)
data class OpenAccountCommand(
    val holderName: String,
    val holderEmail: String,
    val initialDeposit: java.math.BigDecimal,
    val currency: Currency
)

sealed class OpenAccountResult {
    data class Success(val accountId: AccountId, val balance: Money) : OpenAccountResult()
    data class Failure(val reason: String) : OpenAccountResult()
}

data class TransferMoneyCommand(
    val fromAccountId: AccountId,
    val toAccountId: AccountId,
    val amount: java.math.BigDecimal,
    val currency: Currency
)

sealed class TransferMoneyResult {
    data class Success(
        val fromNewBalance: Money,
        val toNewBalance: Money
    ) : TransferMoneyResult()
    data class Failure(val reason: String) : TransferMoneyResult()
}

data class GetAccountQuery(val accountId: AccountId)

sealed class GetAccountResult {
    data class Found(
        val accountId: AccountId,
        val holderName: String,
        val balance: Money,
        val status: AccountStatus
    ) : GetAccountResult()
    object NotFound : GetAccountResult()
}

// Use Case Implementation
@Service
class TransferMoneyService(
    private val loadAccountPort: LoadAccountPort,       // Outbound Port
    private val saveAccountPort: SaveAccountPort,       // Outbound Port
    private val publishEventPort: PublishDomainEventPort // Outbound Port
) : TransferMoneyUseCase {
    
    @Transactional
    override fun transferMoney(command: TransferMoneyCommand): TransferMoneyResult {
        val fromAccount = loadAccountPort.loadAccount(command.fromAccountId)
            ?: return TransferMoneyResult.Failure("Source account not found")
        
        val toAccount = loadAccountPort.loadAccount(command.toAccountId)
            ?: return TransferMoneyResult.Failure("Destination account not found")
        
        val transferAmount = Money(command.amount, command.currency)
        
        return try {
            fromAccount.withdraw(transferAmount)
            toAccount.deposit(transferAmount)
            
            saveAccountPort.saveAccount(fromAccount)
            saveAccountPort.saveAccount(toAccount)
            
            // Publish domain events
            fromAccount.domainEvents.forEach { publishEventPort.publish(it) }
            toAccount.domainEvents.forEach { publishEventPort.publish(it) }
            
            fromAccount.clearDomainEvents()
            toAccount.clearDomainEvents()
            
            TransferMoneyResult.Success(
                fromNewBalance = fromAccount.currentBalance(),
                toNewBalance = toAccount.currentBalance()
            )
        } catch (ex: AccountDomainException) {
            TransferMoneyResult.Failure(ex.message ?: "Transfer failed")
        }
    }
}

@Service
class OpenAccountService(
    private val saveAccountPort: SaveAccountPort,
    private val publishEventPort: PublishDomainEventPort
) : OpenAccountUseCase {
    
    override fun openAccount(command: OpenAccountCommand): OpenAccountResult {
        return try {
            val holder = AccountHolder(
                customerId = CustomerId(java.util.UUID.randomUUID().toString()),
                name = command.holderName,
                email = command.holderEmail
            )
            
            val initialDeposit = Money(command.initialDeposit, command.currency)
            val account = Account.open(holder, initialDeposit)
            
            saveAccountPort.saveAccount(account)
            account.domainEvents.forEach { publishEventPort.publish(it) }
            
            OpenAccountResult.Success(account.id, account.currentBalance())
        } catch (ex: AccountDomainException) {
            OpenAccountResult.Failure(ex.message ?: "Failed to open account")
        }
    }
}

typealias Service = org.springframework.stereotype.Service
typealias Transactional = org.springframework.transaction.annotation.Transactional
```

---

## Ports: Inbound และ Outbound

```kotlin
package com.example.banking.domain.port

// Outbound Ports (Driven Ports) — defined in domain, implemented in adapters

interface LoadAccountPort {
    fun loadAccount(accountId: AccountId): Account?
    fun loadAccountByCustomer(customerId: CustomerId): List<Account>
}

interface SaveAccountPort {
    fun saveAccount(account: Account): Account
}

interface PublishDomainEventPort {
    fun publish(event: AccountEvent)
}

interface SendNotificationPort {
    fun sendTransactionNotification(email: String, transaction: TransactionInfo)
}

data class TransactionInfo(
    val type: String,
    val amount: Money,
    val newBalance: Money,
    val timestamp: java.time.Instant
)
```

---

## Adapters: REST, Database, Messaging

```kotlin
package com.example.banking.adapter.inbound.rest

// REST Adapter (Inbound)
@RestController
@RequestMapping("/api/accounts")
class AccountRestAdapter(
    private val openAccountUseCase: OpenAccountUseCase,
    private val transferMoneyUseCase: TransferMoneyUseCase,
    private val getAccountUseCase: GetAccountUseCase
) {
    
    @PostMapping
    fun openAccount(@RequestBody request: OpenAccountRequest): ResponseEntity<Any> {
        val command = OpenAccountCommand(
            holderName = request.holderName,
            holderEmail = request.holderEmail,
            initialDeposit = request.initialDeposit,
            currency = Currency.valueOf(request.currency)
        )
        
        return when (val result = openAccountUseCase.openAccount(command)) {
            is OpenAccountResult.Success -> ResponseEntity
                .created(java.net.URI.create("/api/accounts/${result.accountId}"))
                .body(OpenAccountResponse(result.accountId.value, result.balance.toString()))
            is OpenAccountResult.Failure -> ResponseEntity
                .badRequest()
                .body(mapOf("error" to result.reason))
        }
    }
    
    @PostMapping("/transfer")
    fun transferMoney(@RequestBody request: TransferRequest): ResponseEntity<Any> {
        val command = TransferMoneyCommand(
            fromAccountId = AccountId(request.fromAccountId),
            toAccountId = AccountId(request.toAccountId),
            amount = request.amount,
            currency = Currency.valueOf(request.currency)
        )
        
        return when (val result = transferMoneyUseCase.transferMoney(command)) {
            is TransferMoneyResult.Success -> ResponseEntity.ok(mapOf(
                "fromBalance" to result.fromNewBalance.toString(),
                "toBalance" to result.toNewBalance.toString()
            ))
            is TransferMoneyResult.Failure -> ResponseEntity
                .status(422)
                .body(mapOf("error" to result.reason))
        }
    }
    
    @GetMapping("/{accountId}")
    fun getAccount(@PathVariable accountId: String): ResponseEntity<Any> {
        return when (val result = getAccountUseCase.getAccount(GetAccountQuery(AccountId(accountId)))) {
            is GetAccountResult.Found -> ResponseEntity.ok(result)
            is GetAccountResult.NotFound -> ResponseEntity.notFound().build()
        }
    }
}

data class OpenAccountRequest(val holderName: String, val holderEmail: String, val initialDeposit: java.math.BigDecimal, val currency: String)
data class OpenAccountResponse(val accountId: String, val balance: String)
data class TransferRequest(val fromAccountId: String, val toAccountId: String, val amount: java.math.BigDecimal, val currency: String)

typealias RestController = org.springframework.web.bind.annotation.RestController
typealias RequestMapping = org.springframework.web.bind.annotation.RequestMapping
typealias PostMapping = org.springframework.web.bind.annotation.PostMapping
typealias GetMapping = org.springframework.web.bind.annotation.GetMapping
typealias RequestBody = org.springframework.web.bind.annotation.RequestBody
typealias PathVariable = org.springframework.web.bind.annotation.PathVariable
typealias ResponseEntity = org.springframework.http.ResponseEntity<*>

// JPA Adapter (Outbound)
package com.example.banking.adapter.outbound.persistence

@Entity
@Table(name = "accounts")
data class AccountJpaEntity(
    @Id val id: String,
    val customerId: String,
    val holderName: String,
    val holderEmail: String,
    val balanceAmount: java.math.BigDecimal,
    val balanceCurrency: String,
    val status: String,
    val createdAt: java.time.Instant = java.time.Instant.now()
)

interface AccountJpaRepository : JpaRepository<AccountJpaEntity, String>

@Repository
class AccountPersistenceAdapter(
    private val jpaRepository: AccountJpaRepository
) : LoadAccountPort, SaveAccountPort {
    
    override fun loadAccount(accountId: AccountId): Account? {
        return jpaRepository.findById(accountId.value)
            .map { it.toDomain() }
            .orElse(null)
    }
    
    override fun loadAccountByCustomer(customerId: CustomerId): List<Account> {
        return jpaRepository.findAll()
            .filter { it.customerId == customerId.value }
            .map { it.toDomain() }
    }
    
    override fun saveAccount(account: Account): Account {
        val entity = account.toJpa()
        jpaRepository.save(entity)
        return account
    }
    
    private fun AccountJpaEntity.toDomain(): Account {
        return Account.reconstitute(
            id = AccountId(id),
            holder = AccountHolder(
                customerId = CustomerId(customerId),
                name = holderName,
                email = holderEmail
            ),
            balance = Money(balanceAmount, Currency.valueOf(balanceCurrency)),
            status = AccountStatus.valueOf(status)
        )
    }
    
    private fun Account.toJpa(): AccountJpaEntity {
        return AccountJpaEntity(
            id = id.value,
            customerId = holder.customerId.value,
            holderName = holder.name,
            holderEmail = holder.email,
            balanceAmount = currentBalance().amount,
            balanceCurrency = currentBalance().currency.name,
            status = AccountStatus.ACTIVE.name  // simplified
        )
    }
}

typealias JpaRepository<T, ID> = org.springframework.data.jpa.repository.JpaRepository<T, ID>
typealias Repository = org.springframework.stereotype.Repository
typealias Entity = jakarta.persistence.Entity
typealias Table = jakarta.persistence.Table
typealias Id = jakarta.persistence.Id

// Kafka Adapter (Outbound)
@Component
class AccountEventKafkaAdapter(
    private val kafkaTemplate: KafkaTemplate<String, String>,
    private val objectMapper: ObjectMapper
) : PublishDomainEventPort {
    
    override fun publish(event: AccountEvent) {
        val (topic, key, payload) = when (event) {
            is AccountEvent.MoneyTransferred -> Triple(
                "account.transfers",
                event.fromAccountId.value,
                event
            )
            is AccountEvent.MoneyDeposited -> Triple(
                "account.transactions",
                event.accountId.value,
                event
            )
            is AccountEvent.MoneyWithdrawn -> Triple(
                "account.transactions",
                event.accountId.value,
                event
            )
            is AccountEvent.AccountOpened -> Triple(
                "account.lifecycle",
                event.accountId.value,
                event
            )
        }
        
        val json = objectMapper.writeValueAsString(payload)
        kafkaTemplate.send(topic, key, json)
    }
}

typealias KafkaTemplate<K, V> = org.springframework.kafka.core.KafkaTemplate<K, V>
typealias ObjectMapper = com.fasterxml.jackson.databind.ObjectMapper
typealias Component = org.springframework.stereotype.Component
```

---

## Testing Hexagonal Architecture

```kotlin
// Domain test (no Spring context needed)
class AccountTest {
    
    @Test
    fun `should deposit money and emit event`() {
        val account = Account.open(
            holder = AccountHolder(CustomerId("cust-1"), "Alice", "alice@example.com"),
            initialDeposit = Money(1000.0, Currency.THB)
        )
        account.clearDomainEvents()
        
        account.deposit(Money(500.0, Currency.THB))
        
        assertEquals(Money(1500.0, Currency.THB), account.currentBalance())
        assertEquals(1, account.domainEvents.size)
        
        val event = account.domainEvents[0] as AccountEvent.MoneyDeposited
        assertEquals(Money(500.0, Currency.THB), event.amount)
        assertEquals(Money(1500.0, Currency.THB), event.newBalance)
    }
    
    @Test
    fun `should throw InsufficientFunds on overdraft`() {
        val account = Account.open(
            holder = AccountHolder(CustomerId("cust-1"), "Bob", "bob@example.com"),
            initialDeposit = Money(100.0, Currency.THB)
        )
        
        assertThrows<AccountDomainException.InsufficientFunds> {
            account.withdraw(Money(200.0, Currency.THB))
        }
        assertEquals(Money(100.0, Currency.THB), account.currentBalance())
    }
}

// Use Case test with mock adapters
class TransferMoneyServiceTest {
    
    private val loadAccountPort = mockk<LoadAccountPort>()
    private val saveAccountPort = mockk<SaveAccountPort>()
    private val publishEventPort = mockk<PublishDomainEventPort>(relaxed = true)
    
    private val sut = TransferMoneyService(loadAccountPort, saveAccountPort, publishEventPort)
    
    @Test
    fun `should transfer money between accounts`() {
        val fromAccount = Account.open(
            AccountHolder(CustomerId("c1"), "Alice", "alice@ex.com"),
            Money(1000.0, Currency.THB)
        )
        val toAccount = Account.open(
            AccountHolder(CustomerId("c2"), "Bob", "bob@ex.com"),
            Money(500.0, Currency.THB)
        )
        
        every { loadAccountPort.loadAccount(fromAccount.id) } returns fromAccount
        every { loadAccountPort.loadAccount(toAccount.id) } returns toAccount
        every { saveAccountPort.saveAccount(any()) } answers { firstArg() }
        
        val result = sut.transferMoney(
            TransferMoneyCommand(
                fromAccountId = fromAccount.id,
                toAccountId = toAccount.id,
                amount = 300.toBigDecimal(),
                currency = Currency.THB
            )
        )
        
        assertTrue(result is TransferMoneyResult.Success)
        val success = result as TransferMoneyResult.Success
        assertEquals(Money(700.0, Currency.THB), success.fromNewBalance)
        assertEquals(Money(800.0, Currency.THB), success.toNewBalance)
        
        verify(exactly = 2) { saveAccountPort.saveAccount(any()) }
        verify(atLeast = 1) { publishEventPort.publish(any()) }
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: เพิ่ม "SuspendAccount" feature ใน Hexagonal Architecture

// 1. Domain: เพิ่ม suspend() method ใน Account aggregate
//    - ต้องเป็น ACTIVE ก่อน suspend
//    - emit AccountSuspended event
//    - ไม่สามารถ deposit/withdraw ในสถานะ SUSPENDED

// 2. Port: SuspendAccountUseCase interface
interface SuspendAccountUseCase {
    fun suspendAccount(command: SuspendAccountCommand): SuspendAccountResult
}

data class SuspendAccountCommand(val accountId: AccountId, val reason: String)

sealed class SuspendAccountResult {
    data class Success(val accountId: AccountId) : SuspendAccountResult()
    data class AccountNotFound(val accountId: AccountId) : SuspendAccountResult()
    data class AlreadySuspended(val accountId: AccountId) : SuspendAccountResult()
}

// 3. Application: SuspendAccountService implement SuspendAccountUseCase

// 4. Adapter: REST endpoint POST /api/accounts/{id}/suspend

// 5. Test: domain test + use case test with mocks
```

---

## สรุป Part 65

```
✅ Hexagonal Architecture: domain at center, adapters at edges
✅ Ports: interfaces defined by domain
✅ Inbound Ports: use case interfaces (what application can do)
✅ Outbound Ports: dependency interfaces (what domain needs)
✅ Inbound Adapters: REST, gRPC, CLI, Tests
✅ Outbound Adapters: JPA, Kafka, Email, HTTP Clients
✅ Domain: pure Kotlin, no framework dependencies
✅ Value Objects: AccountId, Money, Currency
✅ Domain Events: AccountOpened, MoneyDeposited, MoneyTransferred
✅ Domain Exceptions: sealed class hierarchy
✅ Aggregate: Account with private constructor, companion object factory
✅ Application Layer: use case implementations
✅ Commands/Results: application layer DTOs
✅ AccountPersistenceAdapter: maps JPA entity ↔ domain model
✅ AccountEventKafkaAdapter: publishes domain events to Kafka
✅ Domain tests: pure unit tests without Spring
✅ Use case tests: mock outbound ports with MockK
✅ Clean separation: domain knows nothing about infrastructure
```

---

*Part 65/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
