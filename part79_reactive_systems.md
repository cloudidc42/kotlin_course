# Part 79: Reactive Systems และ Event Sourcing

## สารบัญ
1. [Reactive Manifesto](#reactive-manifesto)
2. [Event Sourcing Pattern](#event-sourcing-pattern)
3. [CQRS ด้วย Kotlin](#cqrs-ด้วย-kotlin)
4. [Aggregate ใน Event Sourcing](#aggregate-ใน-event-sourcing)
5. [Projections](#projections)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Reactive Manifesto

```
Reactive Systems มีคุณสมบัติ 4 ข้อ:

1. Responsive: ตอบสนองทันเวลา
   - Bounded latency: guarantee response time
   - Detect and fix problems quickly

2. Resilient: ทนต่อความล้มเหลว
   - Component isolation
   - Replication and fail-over
   - Circuit breaker pattern

3. Elastic: ขยายตัวได้ตามโหลด
   - Scale out/in automatically
   - No bottlenecks or synchronization points

4. Message-Driven: communicate ด้วย async messages
   - Location transparency
   - Back-pressure support
   - Non-blocking communication

Event-Driven vs Message-Driven:
- Event: "something happened" (broadcast, multiple consumers)
- Message: "do this for me" (targeted, one consumer)

Event Sourcing ตอบโจทย์ Reactive ได้:
- Audit log: complete history of all changes
- Time travel: rebuild state at any point in time
- Eventual consistency: projections can be rebuilt
- Event-driven: downstream systems react to events
```

---

## Event Sourcing Pattern

```kotlin
// Event: immutable fact that happened in the past
sealed class BankAccountEvent {
    abstract val aggregateId: String
    abstract val version: Long
    abstract val occurredAt: java.time.Instant
    
    data class AccountOpened(
        override val aggregateId: String,
        override val version: Long,
        override val occurredAt: java.time.Instant = java.time.Instant.now(),
        val ownerId: String,
        val initialBalance: java.math.BigDecimal,
        val currency: String
    ) : BankAccountEvent()
    
    data class MoneyDeposited(
        override val aggregateId: String,
        override val version: Long,
        override val occurredAt: java.time.Instant = java.time.Instant.now(),
        val amount: java.math.BigDecimal,
        val description: String
    ) : BankAccountEvent()
    
    data class MoneyWithdrawn(
        override val aggregateId: String,
        override val version: Long,
        override val occurredAt: java.time.Instant = java.time.Instant.now(),
        val amount: java.math.BigDecimal,
        val description: String
    ) : BankAccountEvent()
    
    data class AccountFrozen(
        override val aggregateId: String,
        override val version: Long,
        override val occurredAt: java.time.Instant = java.time.Instant.now(),
        val reason: String
    ) : BankAccountEvent()
    
    data class MoneyTransferred(
        override val aggregateId: String,
        override val version: Long,
        override val occurredAt: java.time.Instant = java.time.Instant.now(),
        val amount: java.math.BigDecimal,
        val toAccountId: String
    ) : BankAccountEvent()
}

// Event Store: append-only storage for events
interface EventStore {
    suspend fun append(aggregateId: String, events: List<BankAccountEvent>, expectedVersion: Long)
    suspend fun load(aggregateId: String): List<BankAccountEvent>
    suspend fun loadSince(aggregateId: String, version: Long): List<BankAccountEvent>
    suspend fun subscribe(consumer: suspend (BankAccountEvent) -> Unit)
}

// PostgreSQL implementation
class PostgresEventStore(
    private val jdbcTemplate: org.springframework.jdbc.core.JdbcTemplate
) : EventStore {
    
    override suspend fun append(
        aggregateId: String,
        events: List<BankAccountEvent>,
        expectedVersion: Long
    ) = kotlinx.coroutines.withContext(kotlinx.coroutines.Dispatchers.IO) {
        // Optimistic concurrency check
        val currentVersion = jdbcTemplate.queryForObject(
            "SELECT COALESCE(MAX(version), 0) FROM bank_account_events WHERE aggregate_id = ?",
            Long::class.java,
            aggregateId
        ) ?: 0L
        
        if (currentVersion != expectedVersion) {
            throw ConcurrencyException(
                "Expected version $expectedVersion but found $currentVersion"
            )
        }
        
        // Append all events in one transaction
        val batchArgs = events.map { event ->
            arrayOf(
                event.aggregateId,
                event.version,
                event.javaClass.simpleName,
                objectMapper.writeValueAsString(event),
                event.occurredAt
            )
        }
        
        jdbcTemplate.batchUpdate(
            """
            INSERT INTO bank_account_events 
            (aggregate_id, version, event_type, event_data, occurred_at)
            VALUES (?, ?, ?, ?::jsonb, ?)
            """,
            batchArgs
        )
    }
    
    override suspend fun load(aggregateId: String): List<BankAccountEvent> =
        kotlinx.coroutines.withContext(kotlinx.coroutines.Dispatchers.IO) {
            jdbcTemplate.query(
                "SELECT event_type, event_data FROM bank_account_events WHERE aggregate_id = ? ORDER BY version",
                { rs, _ -> deserialize(rs.getString("event_type"), rs.getString("event_data")) },
                aggregateId
            )
        }
    
    override suspend fun loadSince(aggregateId: String, version: Long): List<BankAccountEvent> =
        kotlinx.coroutines.withContext(kotlinx.coroutines.Dispatchers.IO) {
            jdbcTemplate.query(
                """
                SELECT event_type, event_data 
                FROM bank_account_events 
                WHERE aggregate_id = ? AND version > ? 
                ORDER BY version
                """,
                { rs, _ -> deserialize(rs.getString("event_type"), rs.getString("event_data")) },
                aggregateId, version
            )
        }
    
    override suspend fun subscribe(consumer: suspend (BankAccountEvent) -> Unit) {
        // Use PostgreSQL LISTEN/NOTIFY for real-time event streaming
        // Or use polling with last seen position
    }
    
    private fun deserialize(type: String, data: String): BankAccountEvent {
        return when (type) {
            "AccountOpened" -> objectMapper.readValue(data, BankAccountEvent.AccountOpened::class.java)
            "MoneyDeposited" -> objectMapper.readValue(data, BankAccountEvent.MoneyDeposited::class.java)
            "MoneyWithdrawn" -> objectMapper.readValue(data, BankAccountEvent.MoneyWithdrawn::class.java)
            "AccountFrozen" -> objectMapper.readValue(data, BankAccountEvent.AccountFrozen::class.java)
            "MoneyTransferred" -> objectMapper.readValue(data, BankAccountEvent.MoneyTransferred::class.java)
            else -> throw IllegalArgumentException("Unknown event type: $type")
        }
    }
    
    private val objectMapper = com.fasterxml.jackson.databind.ObjectMapper()
        .apply { findAndRegisterModules() }
}

class ConcurrencyException(message: String) : RuntimeException(message)
```

---

## Aggregate ใน Event Sourcing

```kotlin
// Aggregate: rebuilds state from events
class BankAccount private constructor() {
    
    // Current state (rebuilt from events)
    var id: String = ""
        private set
    var ownerId: String = ""
        private set
    var balance: java.math.BigDecimal = java.math.BigDecimal.ZERO
        private set
    var currency: String = "THB"
        private set
    var status: AccountStatus = AccountStatus.ACTIVE
        private set
    var version: Long = 0L
        private set
    
    // Uncommitted events (to be persisted)
    private val _uncommittedEvents = mutableListOf<BankAccountEvent>()
    val uncommittedEvents: List<BankAccountEvent> = _uncommittedEvents
    
    enum class AccountStatus { ACTIVE, FROZEN, CLOSED }
    
    companion object {
        // Factory: open new account
        fun open(
            accountId: String,
            ownerId: String,
            initialBalance: java.math.BigDecimal,
            currency: String = "THB"
        ): BankAccount {
            require(initialBalance >= java.math.BigDecimal.ZERO) { "Initial balance cannot be negative" }
            
            val account = BankAccount()
            account.apply(
                BankAccountEvent.AccountOpened(
                    aggregateId = accountId,
                    version = 1L,
                    ownerId = ownerId,
                    initialBalance = initialBalance,
                    currency = currency
                )
            )
            return account
        }
        
        // Reconstitute from events
        fun reconstitute(events: List<BankAccountEvent>): BankAccount {
            require(events.isNotEmpty()) { "Cannot reconstitute from empty events" }
            
            val account = BankAccount()
            events.forEach { account.applyEvent(event = it, isNew = false) }
            return account
        }
    }
    
    // Commands (business logic)
    fun deposit(amount: java.math.BigDecimal, description: String) {
        require(status == AccountStatus.ACTIVE) { "Cannot deposit to ${status} account" }
        require(amount > java.math.BigDecimal.ZERO) { "Deposit amount must be positive" }
        
        apply(
            BankAccountEvent.MoneyDeposited(
                aggregateId = id,
                version = version + 1,
                amount = amount,
                description = description
            )
        )
    }
    
    fun withdraw(amount: java.math.BigDecimal, description: String) {
        require(status == AccountStatus.ACTIVE) { "Cannot withdraw from ${status} account" }
        require(amount > java.math.BigDecimal.ZERO) { "Withdrawal amount must be positive" }
        require(balance >= amount) { "Insufficient funds: balance=$balance, requested=$amount" }
        
        apply(
            BankAccountEvent.MoneyWithdrawn(
                aggregateId = id,
                version = version + 1,
                amount = amount,
                description = description
            )
        )
    }
    
    fun transfer(amount: java.math.BigDecimal, toAccountId: String) {
        require(status == AccountStatus.ACTIVE) { "Cannot transfer from ${status} account" }
        require(balance >= amount) { "Insufficient funds" }
        
        apply(
            BankAccountEvent.MoneyTransferred(
                aggregateId = id,
                version = version + 1,
                amount = amount,
                toAccountId = toAccountId
            )
        )
    }
    
    fun freeze(reason: String) {
        require(status == AccountStatus.ACTIVE) { "Account is already ${status}" }
        
        apply(
            BankAccountEvent.AccountFrozen(
                aggregateId = id,
                version = version + 1,
                reason = reason
            )
        )
    }
    
    // Apply event (updates in-memory state)
    private fun apply(event: BankAccountEvent) = applyEvent(event, isNew = true)
    
    private fun applyEvent(event: BankAccountEvent, isNew: Boolean) {
        // Update state based on event type (no validation here!)
        when (event) {
            is BankAccountEvent.AccountOpened -> {
                id = event.aggregateId
                ownerId = event.ownerId
                balance = event.initialBalance
                currency = event.currency
                status = AccountStatus.ACTIVE
            }
            is BankAccountEvent.MoneyDeposited -> {
                balance += event.amount
            }
            is BankAccountEvent.MoneyWithdrawn -> {
                balance -= event.amount
            }
            is BankAccountEvent.MoneyTransferred -> {
                balance -= event.amount
            }
            is BankAccountEvent.AccountFrozen -> {
                status = AccountStatus.FROZEN
            }
        }
        
        version = event.version
        
        if (isNew) _uncommittedEvents.add(event)
    }
    
    fun markEventsAsCommitted() {
        _uncommittedEvents.clear()
    }
}

// Repository: load/save aggregates via event store
class BankAccountRepository(private val eventStore: EventStore) {
    
    // Cache for snapshots (optimization for long event streams)
    private val snapshots = java.util.concurrent.ConcurrentHashMap<String, Pair<Long, BankAccount>>()
    
    suspend fun load(accountId: String): BankAccount {
        // Check snapshot cache
        val snapshot = snapshots[accountId]
        
        val events = if (snapshot != null) {
            // Load only events after snapshot
            eventStore.loadSince(accountId, snapshot.first)
        } else {
            eventStore.load(accountId)
        }
        
        return if (snapshot != null && events.isEmpty()) {
            snapshot.second
        } else {
            val allEvents = if (snapshot != null) {
                // Merge snapshot with new events by reconstituting
                eventStore.load(accountId)
            } else {
                events
            }
            BankAccount.reconstitute(allEvents)
        }
    }
    
    suspend fun save(account: BankAccount) {
        val newEvents = account.uncommittedEvents
        if (newEvents.isEmpty()) return
        
        val expectedVersion = newEvents.first().version - 1
        
        eventStore.append(account.id, newEvents, expectedVersion)
        account.markEventsAsCommitted()
        
        // Update snapshot every 50 events
        if (account.version % 50 == 0L) {
            snapshots[account.id] = account.version to account
        }
    }
}
```

---

## CQRS ด้วย Kotlin

```kotlin
// CQRS: Command Query Responsibility Segregation
// Write side (Commands) → Event Store
// Read side (Queries) → Projections/Read Models

// Command side
sealed class BankAccountCommand {
    data class OpenAccount(val ownerId: String, val initialBalance: java.math.BigDecimal) : BankAccountCommand()
    data class Deposit(val accountId: String, val amount: java.math.BigDecimal, val description: String) : BankAccountCommand()
    data class Withdraw(val accountId: String, val amount: java.math.BigDecimal, val description: String) : BankAccountCommand()
    data class Transfer(val fromAccountId: String, val toAccountId: String, val amount: java.math.BigDecimal) : BankAccountCommand()
}

// Command handlers
@Service
class BankAccountCommandHandler(
    private val repository: BankAccountRepository,
    private val eventBus: EventBus
) {
    
    @Transactional
    suspend fun handle(command: BankAccountCommand): String {
        return when (command) {
            is BankAccountCommand.OpenAccount -> {
                val accountId = java.util.UUID.randomUUID().toString()
                val account = BankAccount.open(
                    accountId = accountId,
                    ownerId = command.ownerId,
                    initialBalance = command.initialBalance
                )
                repository.save(account)
                eventBus.publish(account.uncommittedEvents)
                accountId
            }
            
            is BankAccountCommand.Deposit -> {
                val account = repository.load(command.accountId)
                account.deposit(command.amount, command.description)
                repository.save(account)
                eventBus.publish(account.uncommittedEvents)
                account.id
            }
            
            is BankAccountCommand.Withdraw -> {
                val account = repository.load(command.accountId)
                account.withdraw(command.amount, command.description)
                repository.save(account)
                eventBus.publish(account.uncommittedEvents)
                account.id
            }
            
            is BankAccountCommand.Transfer -> {
                // Two-phase: debit source, credit destination
                val source = repository.load(command.fromAccountId)
                val destination = repository.load(command.toAccountId)
                
                source.transfer(command.amount, command.toAccountId)
                destination.deposit(command.amount, "Transfer from ${command.fromAccountId}")
                
                repository.save(source)
                repository.save(destination)
                
                eventBus.publish(source.uncommittedEvents + destination.uncommittedEvents)
                
                source.id
            }
        }
    }
}

interface EventBus {
    suspend fun publish(events: List<BankAccountEvent>)
}

typealias Service = org.springframework.stereotype.Service
typealias Transactional = org.springframework.transaction.annotation.Transactional
```

---

## Projections

```kotlin
// Read-side: projections that rebuild read models from events

// Read model for account queries
data class AccountReadModel(
    val accountId: String,
    val ownerId: String,
    val balance: java.math.BigDecimal,
    val currency: String,
    val status: String,
    val transactionCount: Int,
    val lastTransactionAt: java.time.Instant?
)

// Transaction history read model
data class TransactionReadModel(
    val id: String,
    val accountId: String,
    val type: String,
    val amount: java.math.BigDecimal,
    val description: String,
    val balanceAfter: java.math.BigDecimal,
    val occurredAt: java.time.Instant
)

// Projection: handles events and updates read models
@Service
class AccountProjection(
    private val readModelRepository: AccountReadModelRepository
) {
    
    // Process events (called by event consumer)
    suspend fun on(event: BankAccountEvent) {
        when (event) {
            is BankAccountEvent.AccountOpened -> {
                readModelRepository.save(
                    AccountReadModel(
                        accountId = event.aggregateId,
                        ownerId = event.ownerId,
                        balance = event.initialBalance,
                        currency = event.currency,
                        status = "ACTIVE",
                        transactionCount = 0,
                        lastTransactionAt = null
                    )
                )
            }
            
            is BankAccountEvent.MoneyDeposited -> {
                readModelRepository.updateBalance(
                    accountId = event.aggregateId,
                    delta = event.amount,
                    transactionType = "DEPOSIT",
                    description = event.description,
                    occurredAt = event.occurredAt
                )
            }
            
            is BankAccountEvent.MoneyWithdrawn -> {
                readModelRepository.updateBalance(
                    accountId = event.aggregateId,
                    delta = -event.amount,
                    transactionType = "WITHDRAWAL",
                    description = event.description,
                    occurredAt = event.occurredAt
                )
            }
            
            is BankAccountEvent.AccountFrozen -> {
                readModelRepository.updateStatus(event.aggregateId, "FROZEN")
            }
            
            is BankAccountEvent.MoneyTransferred -> {
                readModelRepository.updateBalance(
                    accountId = event.aggregateId,
                    delta = -event.amount,
                    transactionType = "TRANSFER_OUT",
                    description = "Transfer to ${event.toAccountId}",
                    occurredAt = event.occurredAt
                )
            }
        }
    }
    
    // Rebuild entire projection from scratch
    suspend fun rebuild(eventStore: EventStore) {
        readModelRepository.deleteAll()
        
        // Load all events and replay
        // In production: stream events page by page
        var position = 0L
        var hasMore = true
        
        while (hasMore) {
            val batch = eventStore.loadBatch(position, 1000)
            batch.forEach { on(it) }
            position += batch.size
            hasMore = batch.size == 1000
        }
    }
}

interface AccountReadModelRepository {
    suspend fun save(model: AccountReadModel)
    suspend fun updateBalance(accountId: String, delta: java.math.BigDecimal, transactionType: String, description: String, occurredAt: java.time.Instant)
    suspend fun updateStatus(accountId: String, status: String)
    suspend fun deleteAll()
    suspend fun findById(accountId: String): AccountReadModel?
}

fun EventStore.loadBatch(position: Long, batchSize: Int): List<BankAccountEvent> {
    return emptyList() // TODO: implement
}

// Query side: fast reads from projection
@Service
class AccountQueryService(
    private val readModelRepository: AccountReadModelRepository
) {
    
    suspend fun getAccount(accountId: String): AccountReadModel {
        return readModelRepository.findById(accountId)
            ?: throw AccountNotFoundException("Account $accountId not found")
    }
    
    suspend fun getTransactions(
        accountId: String,
        from: java.time.Instant? = null,
        to: java.time.Instant? = null
    ): List<TransactionReadModel> {
        // Query from read model (fast, no aggregate reconstitution needed)
        return emptyList()  // TODO: implement
    }
}

class AccountNotFoundException(msg: String) : RuntimeException(msg)
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement Shopping Cart ด้วย Event Sourcing

sealed class CartEvent {
    abstract val cartId: String
    abstract val version: Long
    
    data class CartCreated(
        override val cartId: String,
        override val version: Long,
        val userId: String
    ) : CartEvent()
    
    data class ItemAdded(
        override val cartId: String,
        override val version: Long,
        val productId: String,
        val name: String,
        val price: java.math.BigDecimal,
        val quantity: Int
    ) : CartEvent()
    
    data class ItemQuantityChanged(
        override val cartId: String,
        override val version: Long,
        val productId: String,
        val newQuantity: Int
    ) : CartEvent()
    
    data class ItemRemoved(
        override val cartId: String,
        override val version: Long,
        val productId: String
    ) : CartEvent()
    
    data class CartCheckedOut(
        override val cartId: String,
        override val version: Long,
        val orderId: String
    ) : CartEvent()
    
    data class CartAbandoned(
        override val cartId: String,
        override val version: Long,
        val reason: String
    ) : CartEvent()
}

// TODO: Implement ShoppingCart aggregate:
// 1. CartItem data class
// 2. CartStatus enum
// 3. addItem() command
// 4. removeItem() command
// 5. changeQuantity() command
// 6. checkout() command
// 7. applyEvent() for each event type
// 8. Calculate total() from current state

data class CartItem(
    val productId: String,
    val name: String,
    val price: java.math.BigDecimal,
    val quantity: Int
) {
    val subtotal: java.math.BigDecimal get() = price * quantity.toBigDecimal()
}

class ShoppingCart private constructor() {
    var cartId: String = ""
        private set
    var userId: String = ""
        private set
    var items: Map<String, CartItem> = emptyMap()
        private set
    var status: CartStatus = CartStatus.ACTIVE
        private set
    var version: Long = 0L
        private set
    
    private val _uncommittedEvents = mutableListOf<CartEvent>()
    val uncommittedEvents: List<CartEvent> = _uncommittedEvents
    
    val total: java.math.BigDecimal
        get() = items.values.sumOf { it.subtotal }
    
    enum class CartStatus { ACTIVE, CHECKED_OUT, ABANDONED }
    
    companion object {
        fun create(cartId: String, userId: String): ShoppingCart {
            val cart = ShoppingCart()
            cart.applyEvent(CartEvent.CartCreated(cartId, 1L, userId), isNew = true)
            return cart
        }
        
        fun reconstitute(events: List<CartEvent>): ShoppingCart {
            val cart = ShoppingCart()
            events.forEach { cart.applyEvent(it, isNew = false) }
            return cart
        }
    }
    
    fun addItem(productId: String, name: String, price: java.math.BigDecimal, quantity: Int) {
        require(status == CartStatus.ACTIVE) { "Cannot modify ${status} cart" }
        require(quantity > 0) { "Quantity must be positive" }
        
        val existing = items[productId]
        if (existing != null) {
            applyEvent(CartEvent.ItemQuantityChanged(cartId, version + 1, productId, existing.quantity + quantity), isNew = true)
        } else {
            applyEvent(CartEvent.ItemAdded(cartId, version + 1, productId, name, price, quantity), isNew = true)
        }
    }
    
    private fun applyEvent(event: CartEvent, isNew: Boolean) {
        when (event) {
            is CartEvent.CartCreated -> {
                cartId = event.cartId
                userId = event.userId
            }
            is CartEvent.ItemAdded -> {
                items = items + (event.productId to CartItem(event.productId, event.name, event.price, event.quantity))
            }
            is CartEvent.ItemQuantityChanged -> {
                val item = items[event.productId] ?: return
                items = items + (event.productId to item.copy(quantity = event.newQuantity))
            }
            is CartEvent.ItemRemoved -> {
                items = items - event.productId
            }
            is CartEvent.CartCheckedOut -> status = CartStatus.CHECKED_OUT
            is CartEvent.CartAbandoned -> status = CartStatus.ABANDONED
        }
        version = event.version
        if (isNew) _uncommittedEvents.add(event)
    }
}
```

---

## สรุป Part 79

```
✅ Reactive Manifesto: responsive, resilient, elastic, message-driven
✅ Event Sourcing: append-only event log as source of truth
✅ Immutable events: sealed class hierarchy
✅ EventStore: append with optimistic concurrency check
✅ Aggregate reconstitute: replay events to rebuild state
✅ apply() vs applyEvent(): validation vs pure state update
✅ uncommittedEvents: track new events before persistence
✅ markEventsAsCommitted(): clear after successful save
✅ Snapshot: cache aggregate state every N events
✅ ConcurrencyException: version mismatch detection
✅ CQRS: separate command and query models
✅ CommandHandler: processes commands, calls aggregate
✅ EventBus: publish events for downstream consumers
✅ Projection: rebuild read model from events
✅ AccountReadModel: denormalized for fast queries
✅ Projection rebuild: replay all events from scratch
✅ QueryService: read from projection (no aggregate load)
✅ ShoppingCart: event-sourced aggregate exercise
✅ CartStatus: ACTIVE/CHECKED_OUT/ABANDONED
✅ CartItem subtotal: computed property
```

---

*Part 79/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
