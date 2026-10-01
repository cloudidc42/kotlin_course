# Part 31: Clean Architecture กับ Kotlin

## สารบัญ
1. [Clean Architecture คืออะไร](#clean-architecture-คืออะไร)
2. [Layers ของ Clean Architecture](#layers-ของ-clean-architecture)
3. [Domain Layer](#domain-layer)
4. [Data Layer](#data-layer)
5. [Presentation Layer](#presentation-layer)
6. [Dependency Injection](#dependency-injection)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Clean Architecture คืออะไร

```
Clean Architecture = วิธีจัด code ที่แยก concerns ชัดเจน

หลักการหลัก:
1. Dependency Rule: dependencies ชี้เข้าหา center เสมอ
2. Independence: Business rules ไม่รู้จัก UI, DB, หรือ Framework
3. Testability: Business rules ทดสอบได้โดยไม่ต้องใช้ UI/DB

วงกลมจากใน → นอก:
Entities → Use Cases → Interface Adapters → Frameworks/Drivers

โครงสร้าง Project:
app/
├── domain/           # Business rules (innermost)
│   ├── model/        # Entities
│   ├── repository/   # Repository interfaces
│   └── usecase/      # Use cases (interactors)
├── data/             # Data sources, implementations
│   ├── repository/   # Repository implementations
│   ├── local/        # Database, SharedPrefs
│   └── remote/       # API calls
└── presentation/     # UI (outermost)
    ├── viewmodel/    # ViewModels
    └── ui/           # Views, Composables
```

---

## Domain Layer

```kotlin
// domain/model/User.kt - Core business entity
data class User(
    val id: UserId,
    val name: UserName,
    val email: Email,
    val role: UserRole,
    val createdAt: Long,
    val isActive: Boolean = true
) {
    companion object {
        fun create(name: String, email: String, role: UserRole = UserRole.REGULAR): User {
            return User(
                id = UserId.generate(),
                name = UserName.of(name),
                email = Email.of(email),
                role = role,
                createdAt = System.currentTimeMillis()
            )
        }
    }
    
    fun activate() = copy(isActive = true)
    fun deactivate() = copy(isActive = false)
    fun changeRole(newRole: UserRole) = copy(role = newRole)
}

// Value Objects: เป็น domain concepts ที่มี validation
@JvmInline
value class UserId(val value: String) {
    companion object {
        fun generate() = UserId(java.util.UUID.randomUUID().toString())
        fun of(value: String) = UserId(value.also {
            require(it.isNotBlank()) { "UserId cannot be blank" }
        })
    }
}

@JvmInline
value class UserName(val value: String) {
    companion object {
        fun of(value: String) = UserName(value.trim().also {
            require(it.length >= 2) { "Name too short (min 2 chars)" }
            require(it.length <= 100) { "Name too long (max 100 chars)" }
        })
    }
}

@JvmInline
value class Email(val value: String) {
    companion object {
        private val EMAIL_REGEX = "^[A-Za-z0-9._%+\\-]+@[A-Za-z0-9.\\-]+\\.[A-Za-z]{2,}$".toRegex()
        
        fun of(value: String) = Email(value.trim().lowercase().also {
            require(it.matches(EMAIL_REGEX)) { "Invalid email: $it" }
        })
    }
}

enum class UserRole { ADMIN, MODERATOR, REGULAR, GUEST }

// domain/repository/UserRepository.kt - Interface (dependency inversion)
interface UserRepository {
    suspend fun findById(id: UserId): User?
    suspend fun findByEmail(email: Email): User?
    suspend fun findAll(page: Int = 0, size: Int = 20): List<User>
    suspend fun save(user: User): User
    suspend fun delete(id: UserId): Boolean
    suspend fun existsByEmail(email: Email): Boolean
    suspend fun count(): Long
}

// domain/usecase/CreateUserUseCase.kt
data class CreateUserCommand(
    val name: String,
    val email: String,
    val role: UserRole = UserRole.REGULAR
)

sealed class CreateUserResult {
    data class Success(val user: User) : CreateUserResult()
    data class EmailAlreadyExists(val email: String) : CreateUserResult()
    data class ValidationError(val message: String) : CreateUserResult()
}

class CreateUserUseCase(private val userRepository: UserRepository) {
    suspend fun execute(command: CreateUserCommand): CreateUserResult {
        return try {
            val email = Email.of(command.email)
            
            if (userRepository.existsByEmail(email)) {
                return CreateUserResult.EmailAlreadyExists(command.email)
            }
            
            val user = User.create(command.name, command.email, command.role)
            val saved = userRepository.save(user)
            
            CreateUserResult.Success(saved)
        } catch (e: IllegalArgumentException) {
            CreateUserResult.ValidationError(e.message ?: "Validation failed")
        }
    }
}

// domain/usecase/GetUserUseCase.kt
class GetUserUseCase(private val userRepository: UserRepository) {
    suspend fun byId(id: UserId): User? = userRepository.findById(id)
    suspend fun byEmail(email: Email): User? = userRepository.findByEmail(email)
    suspend fun all(page: Int = 0, size: Int = 20): List<User> = userRepository.findAll(page, size)
}

// domain/usecase/UpdateUserUseCase.kt
data class UpdateUserCommand(
    val userId: String,
    val name: String?,
    val role: UserRole?
)

sealed class UpdateUserResult {
    data class Success(val user: User) : UpdateUserResult()
    object NotFound : UpdateUserResult()
    data class ValidationError(val message: String) : UpdateUserResult()
}

class UpdateUserUseCase(private val userRepository: UserRepository) {
    suspend fun execute(command: UpdateUserCommand): UpdateUserResult {
        return try {
            val userId = UserId.of(command.userId)
            val user = userRepository.findById(userId)
                ?: return UpdateUserResult.NotFound
            
            var updated = user
            command.name?.let { updated = updated.copy(name = UserName.of(it)) }
            command.role?.let { updated = updated.changeRole(it) }
            
            UpdateUserResult.Success(userRepository.save(updated))
        } catch (e: IllegalArgumentException) {
            UpdateUserResult.ValidationError(e.message ?: "Validation failed")
        }
    }
}
```

---

## Data Layer

```kotlin
// data/local/UserEntity.kt - Database entity (separate from domain model)
import androidx.room.*

@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "name") val name: String,
    @ColumnInfo(name = "email") val email: String,
    @ColumnInfo(name = "role") val role: String,
    @ColumnInfo(name = "is_active") val isActive: Boolean,
    @ColumnInfo(name = "created_at") val createdAt: Long
)

@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    suspend fun findAll(): List<UserEntity>
    
    @Query("SELECT * FROM users LIMIT :size OFFSET :offset")
    suspend fun findPage(size: Int, offset: Int): List<UserEntity>
    
    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun findById(id: String): UserEntity?
    
    @Query("SELECT * FROM users WHERE email = :email")
    suspend fun findByEmail(email: String): UserEntity?
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun save(entity: UserEntity)
    
    @Delete
    suspend fun delete(entity: UserEntity): Int
    
    @Query("SELECT COUNT(*) FROM users WHERE email = :email")
    suspend fun countByEmail(email: String): Int
    
    @Query("SELECT COUNT(*) FROM users")
    suspend fun count(): Long
}

// data/remote/UserApiService.kt - API service
import retrofit2.http.*

data class UserApiModel(
    val id: String,
    val name: String,
    val email: String,
    val role: String,
    val isActive: Boolean,
    val createdAt: Long
)

interface UserApiService {
    @GET("users")
    suspend fun getUsers(@Query("page") page: Int, @Query("size") size: Int): List<UserApiModel>
    
    @GET("users/{id}")
    suspend fun getUserById(@Path("id") id: String): UserApiModel
    
    @POST("users")
    suspend fun createUser(@Body request: CreateUserApiRequest): UserApiModel
    
    @PUT("users/{id}")
    suspend fun updateUser(@Path("id") id: String, @Body request: UpdateUserApiRequest): UserApiModel
    
    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") id: String): retrofit2.Response<Unit>
}

data class CreateUserApiRequest(val name: String, val email: String, val role: String)
data class UpdateUserApiRequest(val name: String?, val role: String?)

// data/repository/UserRepositoryImpl.kt - Implementation
class UserRepositoryImpl(
    private val userDao: UserDao,
    private val apiService: UserApiService,
    private val networkChecker: NetworkChecker
) : UserRepository {
    
    override suspend fun findById(id: UserId): User? {
        return userDao.findById(id.value)?.toDomain()
    }
    
    override suspend fun findByEmail(email: Email): User? {
        return userDao.findByEmail(email.value)?.toDomain()
    }
    
    override suspend fun findAll(page: Int, size: Int): List<User> {
        if (networkChecker.isAvailable()) {
            try {
                val apiUsers = apiService.getUsers(page, size)
                apiUsers.forEach { userDao.save(it.toEntity()) }
            } catch (e: Exception) {
                // Fallback to local
            }
        }
        return userDao.findPage(size, page * size).map { it.toDomain() }
    }
    
    override suspend fun save(user: User): User {
        userDao.save(user.toEntity())
        if (networkChecker.isAvailable()) {
            try {
                apiService.createUser(CreateUserApiRequest(
                    user.name.value,
                    user.email.value,
                    user.role.name
                ))
            } catch (e: Exception) {
                // Log error, local save succeeded
            }
        }
        return user
    }
    
    override suspend fun delete(id: UserId): Boolean {
        val entity = userDao.findById(id.value) ?: return false
        userDao.delete(entity)
        return true
    }
    
    override suspend fun existsByEmail(email: Email): Boolean {
        return userDao.countByEmail(email.value) > 0
    }
    
    override suspend fun count(): Long = userDao.count()
    
    // Mappers
    private fun UserEntity.toDomain() = User(
        id = UserId(id),
        name = UserName(name),
        email = Email(email),
        role = UserRole.valueOf(role),
        createdAt = createdAt,
        isActive = isActive
    )
    
    private fun User.toEntity() = UserEntity(
        id = id.value,
        name = name.value,
        email = email.value,
        role = role.name,
        isActive = isActive,
        createdAt = createdAt
    )
    
    private fun UserApiModel.toEntity() = UserEntity(
        id = id, name = name, email = email,
        role = role, isActive = isActive, createdAt = createdAt
    )
}

interface NetworkChecker {
    fun isAvailable(): Boolean
}
```

---

## Presentation Layer

```kotlin
// presentation/viewmodel/UserViewModel.kt
import kotlinx.coroutines.flow.*

data class UserUiState(
    val users: List<UserUiModel> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
    val createSuccess: Boolean = false
)

data class UserUiModel(
    val id: String,
    val name: String,
    val email: String,
    val role: String,
    val isActive: Boolean
)

class UserViewModel(
    private val createUser: CreateUserUseCase,
    private val getUser: GetUserUseCase,
    private val updateUser: UpdateUserUseCase
) : ViewModel() {
    
    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()
    
    fun loadUsers() {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            
            try {
                val users = getUser.all()
                _uiState.update { state ->
                    state.copy(
                        users = users.map { it.toUiModel() },
                        isLoading = false
                    )
                }
            } catch (e: Exception) {
                _uiState.update { it.copy(isLoading = false, error = e.message) }
            }
        }
    }
    
    fun createUser(name: String, email: String) {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            
            when (val result = createUser.execute(CreateUserCommand(name, email))) {
                is CreateUserResult.Success -> {
                    _uiState.update { state ->
                        state.copy(
                            users = state.users + result.user.toUiModel(),
                            isLoading = false,
                            createSuccess = true
                        )
                    }
                }
                is CreateUserResult.EmailAlreadyExists -> {
                    _uiState.update { it.copy(
                        isLoading = false,
                        error = "Email already exists: ${result.email}"
                    )}
                }
                is CreateUserResult.ValidationError -> {
                    _uiState.update { it.copy(
                        isLoading = false,
                        error = result.message
                    )}
                }
            }
        }
    }
    
    fun dismissCreateSuccess() {
        _uiState.update { it.copy(createSuccess = false) }
    }
    
    private fun User.toUiModel() = UserUiModel(
        id = id.value,
        name = name.value,
        email = email.value,
        role = role.name,
        isActive = isActive
    )
}
```

---

## Dependency Injection

```kotlin
// di/AppModule.kt (using Koin)
import org.koin.dsl.module
import org.koin.androidx.viewmodel.dsl.viewModel

val domainModule = module {
    factory { CreateUserUseCase(get()) }
    factory { GetUserUseCase(get()) }
    factory { UpdateUserUseCase(get()) }
}

val dataModule = module {
    // Room database
    single { 
        Room.databaseBuilder(androidContext(), AppDatabase::class.java, "app_db")
            .build()
    }
    single { get<AppDatabase>().userDao() }
    
    // Retrofit
    single {
        Retrofit.Builder()
            .baseUrl("https://api.example.com/")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(UserApiService::class.java)
    }
    
    single<NetworkChecker> { AndroidNetworkChecker(androidContext()) }
    single<UserRepository> { UserRepositoryImpl(get(), get(), get()) }
}

val presentationModule = module {
    viewModel { UserViewModel(get(), get(), get()) }
}

val appModules = listOf(domainModule, dataModule, presentationModule)

// Application
class MyApp : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@MyApp)
            modules(appModules)
        }
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Task Management Clean Architecture

// Domain
data class TaskId(val value: String) {
    companion object {
        fun generate() = TaskId(java.util.UUID.randomUUID().toString())
    }
}

data class TaskTitle(val value: String) {
    init { require(value.isNotBlank() && value.length <= 200) { "Invalid title" } }
}

enum class TaskPriority { LOW, MEDIUM, HIGH, URGENT }
enum class TaskStatus { TODO, IN_PROGRESS, DONE, CANCELLED }

data class Task(
    val id: TaskId = TaskId.generate(),
    val title: TaskTitle,
    val description: String = "",
    val priority: TaskPriority = TaskPriority.MEDIUM,
    val status: TaskStatus = TaskStatus.TODO,
    val dueDate: Long? = null,
    val createdAt: Long = System.currentTimeMillis()
) {
    fun start() = copy(status = TaskStatus.IN_PROGRESS)
    fun complete() = copy(status = TaskStatus.DONE)
    fun cancel() = copy(status = TaskStatus.CANCELLED)
    
    val isOverdue: Boolean
        get() = dueDate != null && 
                status !in listOf(TaskStatus.DONE, TaskStatus.CANCELLED) &&
                dueDate < System.currentTimeMillis()
}

interface TaskRepository {
    suspend fun findAll(): List<Task>
    suspend fun findById(id: TaskId): Task?
    suspend fun findByStatus(status: TaskStatus): List<Task>
    suspend fun save(task: Task): Task
    suspend fun delete(id: TaskId): Boolean
}

class CreateTaskUseCase(private val repo: TaskRepository) {
    data class Command(val title: String, val description: String = "", 
                       val priority: TaskPriority = TaskPriority.MEDIUM, val dueDate: Long? = null)
    
    suspend fun execute(cmd: Command): Result<Task> = runCatching {
        val task = Task(
            title = TaskTitle(cmd.title),
            description = cmd.description,
            priority = cmd.priority,
            dueDate = cmd.dueDate
        )
        repo.save(task)
    }
}

class GetTasksUseCase(private val repo: TaskRepository) {
    suspend fun all() = repo.findAll()
    suspend fun byStatus(status: TaskStatus) = repo.findByStatus(status)
    suspend fun overdue() = repo.findAll().filter { it.isOverdue }
    suspend fun pending() = repo.findByStatus(TaskStatus.TODO) + repo.findByStatus(TaskStatus.IN_PROGRESS)
}

class UpdateTaskStatusUseCase(private val repo: TaskRepository) {
    sealed class Command {
        data class Start(val id: String) : Command()
        data class Complete(val id: String) : Command()
        data class Cancel(val id: String) : Command()
    }
    
    suspend fun execute(cmd: Command): Task? {
        val (id, transform) = when (cmd) {
            is Command.Start    -> cmd.id to Task::start
            is Command.Complete -> cmd.id to Task::complete
            is Command.Cancel   -> cmd.id to Task::cancel
        }
        val task = repo.findById(TaskId(id)) ?: return null
        return repo.save(transform(task))
    }
}

// In-memory repository for testing
class InMemoryTaskRepository : TaskRepository {
    private val tasks = mutableMapOf<String, Task>()
    
    override suspend fun findAll() = tasks.values.toList()
    override suspend fun findById(id: TaskId) = tasks[id.value]
    override suspend fun findByStatus(status: TaskStatus) = tasks.values.filter { it.status == status }
    override suspend fun save(task: Task) = task.also { tasks[it.id.value] = it }
    override suspend fun delete(id: TaskId) = tasks.remove(id.value) != null
}

suspend fun main() {
    val repo = InMemoryTaskRepository()
    val create = CreateTaskUseCase(repo)
    val get = GetTasksUseCase(repo)
    val update = UpdateTaskStatusUseCase(repo)
    
    // Create tasks
    val t1 = create.execute(CreateTaskUseCase.Command("ออกแบบ Database Schema", priority = TaskPriority.HIGH))
    val t2 = create.execute(CreateTaskUseCase.Command("เขียน Unit Tests", priority = TaskPriority.MEDIUM))
    val t3 = create.execute(CreateTaskUseCase.Command("Deploy to Production", priority = TaskPriority.URGENT))
    
    println("All tasks:")
    get.all().forEach { println("  [${it.status}] ${it.title.value} (${it.priority})") }
    
    // Update status
    t1.getOrNull()?.let { update.execute(UpdateTaskStatusUseCase.Command.Start(it.id.value)) }
    t2.getOrNull()?.let { update.execute(UpdateTaskStatusUseCase.Command.Complete(it.id.value)) }
    
    println("\nAfter updates:")
    get.all().forEach { println("  [${it.status}] ${it.title.value}") }
    
    println("\nPending tasks:")
    get.pending().forEach { println("  ${it.title.value}") }
}
```

---

## สรุป Part 31

```
✅ Clean Architecture: แยก business logic ออกจาก UI/DB
✅ Domain Layer: Entities, Value Objects, Interfaces
✅ Value Objects ด้วย @JvmInline value class (efficient)
✅ Use Cases = single business operation
✅ Data Layer: Repository implementations, mappers
✅ Presentation Layer: ViewModels, UI state
✅ Dependency Inversion: depend on abstractions, not concrete
✅ Mappers: แปลง between layers (Domain ↔ Data ↔ UI)
✅ Koin: dependency injection สำหรับ Kotlin
✅ Sealed classes สำหรับ typed results
✅ StateFlow: observable UI state
```

---

*Part 31/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
