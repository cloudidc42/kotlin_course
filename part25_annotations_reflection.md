# Part 25: Annotations และ Reflection

## สารบัญ
1. [Annotations พื้นฐาน](#annotations-พื้นฐาน)
2. [สร้าง Custom Annotation](#สร้าง-custom-annotation)
3. [Reflection API](#reflection-api)
4. [Annotation Processing](#annotation-processing)
5. [Practical Examples](#practical-examples)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Annotations พื้นฐาน

```kotlin
import kotlin.reflect.*
import kotlin.reflect.full.*

// Built-in annotations
@Deprecated("Use newFunction() instead", ReplaceWith("newFunction()"))
fun oldFunction() = "old"

fun newFunction() = "new"

@Suppress("UNCHECKED_CAST")
fun unsafeCast(obj: Any) = obj as List<String>

@JvmStatic  // ใช้ใน companion object เพื่อให้ Java เรียกได้
class MyClass {
    companion object {
        @JvmStatic
        fun staticMethod() = "called from Java as MyClass.staticMethod()"
    }
}

@JvmField  // expose field โดยตรง ไม่ต้องผ่าน getter
class Config {
    @JvmField
    val version = "1.0"
}

// Annotation targets
@Target(AnnotationTarget.CLASS)
annotation class ClassOnly

@Target(AnnotationTarget.FUNCTION)
annotation class FunctionOnly

@Target(AnnotationTarget.PROPERTY)
annotation class PropertyOnly

@Target(
    AnnotationTarget.CLASS,
    AnnotationTarget.FUNCTION,
    AnnotationTarget.PROPERTY,
    AnnotationTarget.FIELD,
    AnnotationTarget.VALUE_PARAMETER
)
annotation class Anywhere

// Retention
@Retention(AnnotationRetention.SOURCE)   // อยู่แค่ source code
annotation class CompileHint

@Retention(AnnotationRetention.BINARY)   // อยู่ใน bytecode แต่ reflection ไม่เห็น (default)
annotation class BinaryAnnotation

@Retention(AnnotationRetention.RUNTIME)  // อยู่ตอน runtime, reflection เห็นได้
annotation class RuntimeAnnotation

fun main() {
    @Suppress("DEPRECATION")
    println(oldFunction())
    println(newFunction())
}
```

---

## สร้าง Custom Annotation

```kotlin
import kotlin.reflect.full.*

// Custom annotation with parameters
@Target(AnnotationTarget.CLASS, AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class Api(
    val version: String = "v1",
    val path: String = "",
    val description: String = ""
)

@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class Route(
    val method: HttpMethod = HttpMethod.GET,
    val path: String
) {
    enum class HttpMethod { GET, POST, PUT, DELETE, PATCH }
}

@Target(AnnotationTarget.VALUE_PARAMETER, AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.RUNTIME)
annotation class Validate(
    val notBlank: Boolean = false,
    val minLength: Int = 0,
    val maxLength: Int = Int.MAX_VALUE,
    val pattern: String = "",
    val message: String = ""
)

@Target(AnnotationTarget.PROPERTY, AnnotationTarget.FIELD)
@Retention(AnnotationRetention.RUNTIME)
annotation class Column(
    val name: String = "",
    val nullable: Boolean = true,
    val unique: Boolean = false,
    val length: Int = 255
)

@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
annotation class Table(val name: String)

// Using annotations
@Api(version = "v2", path = "/users", description = "User management API")
class UserController {
    @Route(method = Route.HttpMethod.GET, path = "/")
    fun getAllUsers(): List<String> = listOf("Alice", "Bob")
    
    @Route(method = Route.HttpMethod.GET, path = "/{id}")
    fun getUser(): String = "User details"
    
    @Route(method = Route.HttpMethod.POST, path = "/")
    fun createUser(): String = "Created"
}

@Table("products")
data class ProductEntity(
    @Column(name = "product_id", nullable = false, unique = true)
    val id: Int,
    
    @Column(name = "product_name", nullable = false, length = 200)
    val name: String,
    
    @Column(name = "unit_price")
    val price: Double,
    
    @Column(name = "in_stock", nullable = false)
    val inStock: Boolean
)

fun main() {
    // Read annotation at runtime
    val apiAnnotation = UserController::class.findAnnotation<Api>()
    println("API version: ${apiAnnotation?.version}")
    println("API path: ${apiAnnotation?.path}")
    println("API description: ${apiAnnotation?.description}")
    
    println()
    
    // Read method annotations
    UserController::class.memberFunctions.forEach { func ->
        val route = func.findAnnotation<Route>()
        if (route != null) {
            println("${route.method} ${apiAnnotation?.path}${route.path} → ${func.name}()")
        }
    }
    
    println()
    
    // Table/Column annotations
    val tableAnn = ProductEntity::class.findAnnotation<Table>()
    println("Table: ${tableAnn?.name}")
    
    ProductEntity::class.memberProperties.forEach { prop ->
        val column = prop.findAnnotation<Column>()
        if (column != null) {
            val colName = column.name.ifEmpty { prop.name }
            val constraints = buildList {
                if (!column.nullable) add("NOT NULL")
                if (column.unique) add("UNIQUE")
                if (column.length != 255) add("LENGTH(${column.length})")
            }
            println("  ${prop.name} → $colName ${constraints.joinToString(" ")}")
        }
    }
}
```

---

## Reflection API

```kotlin
import kotlin.reflect.*
import kotlin.reflect.full.*
import kotlin.reflect.jvm.*

data class Person(
    val name: String,
    val age: Int,
    var email: String
) {
    fun greet() = "สวัสดี ฉันชื่อ $name อายุ $age ปี"
    fun updateEmail(newEmail: String) { email = newEmail }
    
    companion object {
        fun create(name: String) = Person(name, 0, "")
    }
}

fun main() {
    val person = Person("สมชาย", 25, "somchai@email.com")
    val kClass = person::class  // หรือ Person::class
    
    // Class info
    println("Class name: ${kClass.simpleName}")
    println("Qualified: ${kClass.qualifiedName}")
    println("Is data class: ${kClass.isData}")
    println("Is abstract: ${kClass.isAbstract}")
    println("Is sealed: ${kClass.isSealed}")
    println("Is final: ${kClass.isFinal}")
    
    println()
    
    // Properties
    println("Properties:")
    kClass.memberProperties.forEach { prop ->
        val value = prop.get(person)
        val mutable = if (prop is KMutableProperty<*>) "var" else "val"
        println("  $mutable ${prop.name}: ${prop.returnType} = $value")
    }
    
    println()
    
    // Functions
    println("Functions:")
    kClass.memberFunctions.forEach { func ->
        val params = func.parameters.drop(1).joinToString(", ") { "${it.name}: ${it.type}" }
        println("  fun ${func.name}($params): ${func.returnType}")
    }
    
    println()
    
    // Constructors
    println("Constructors:")
    kClass.constructors.forEach { ctor ->
        val params = ctor.parameters.joinToString(", ") { "${it.name}: ${it.type}" }
        println("  constructor($params)")
    }
    
    println()
    
    // Call function by name
    val greetFn = kClass.memberFunctions.first { it.name == "greet" }
    val greeting = greetFn.call(person)
    println("Greeting: $greeting")
    
    // Modify property by name
    val emailProp = kClass.memberProperties.first { it.name == "email" }
    if (emailProp is KMutableProperty<*>) {
        emailProp.setter.call(person, "new@email.com")
        println("New email: ${person.email}")
    }
    
    println()
    
    // Create instance via reflection
    val ctor = kClass.primaryConstructor!!
    val newPerson = ctor.callBy(mapOf(
        ctor.parameters.first { it.name == "name" } to "สมหญิง",
        ctor.parameters.first { it.name == "age" } to 30,
        ctor.parameters.first { it.name == "email" } to "somying@email.com"
    ))
    println("Created: $newPerson")
    
    // Type info
    val nameType = Person::name.returnType
    println("\nname type: $nameType")
    println("Is nullable: ${nameType.isMarkedNullable}")
    println("Classifier: ${nameType.classifier}")
    
    // Superclasses
    println("\nSuperclasses of Person:")
    kClass.supertypes.forEach { println("  $it") }
}
```

---

## Practical Examples

```kotlin
// Dependency Injection container (simplified)
import kotlin.reflect.full.*

@Target(AnnotationTarget.CONSTRUCTOR, AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
annotation class Injectable

@Target(AnnotationTarget.VALUE_PARAMETER)
@Retention(AnnotationRetention.RUNTIME)
annotation class Inject

class Container {
    private val registry = mutableMapOf<String, () -> Any>()
    private val singletons = mutableMapOf<String, Any>()
    
    fun <T : Any> register(klass: kotlin.reflect.KClass<T>, factory: () -> T) {
        registry[klass.qualifiedName!!] = factory
    }
    
    inline fun <reified T : Any> register(noinline factory: () -> T) {
        register(T::class, factory)
    }
    
    @Suppress("UNCHECKED_CAST")
    fun <T : Any> resolve(klass: kotlin.reflect.KClass<T>): T {
        val key = klass.qualifiedName!!
        return singletons.getOrPut(key) {
            registry[key]?.invoke() ?: autoResolve(klass)
        } as T
    }
    
    inline fun <reified T : Any> resolve(): T = resolve(T::class)
    
    private fun <T : Any> autoResolve(klass: kotlin.reflect.KClass<T>): T {
        val ctor = klass.primaryConstructor
            ?: throw IllegalArgumentException("No primary constructor for ${klass.simpleName}")
        
        val args = ctor.parameters.map { param ->
            val paramClass = param.type.classifier as? kotlin.reflect.KClass<*>
                ?: throw IllegalArgumentException("Cannot resolve parameter ${param.name}")
            resolve(paramClass)
        }
        
        return ctor.call(*args.toTypedArray())
    }
}

// Object mapper (JSON-like)
class ObjectMapper {
    fun <T : Any> toMap(obj: T): Map<String, Any?> {
        val kClass = obj::class
        return kClass.memberProperties.associate { prop ->
            prop.name to prop.get(obj)
        }
    }
    
    @Suppress("UNCHECKED_CAST")
    fun <T : Any> fromMap(klass: kotlin.reflect.KClass<T>, map: Map<String, Any?>): T {
        val ctor = klass.primaryConstructor
            ?: throw IllegalArgumentException("No primary constructor")
        
        val args = ctor.parameters.associate { param ->
            param to map[param.name]
        }
        
        return ctor.callBy(args)
    }
    
    inline fun <reified T : Any> fromMap(map: Map<String, Any?>): T = fromMap(T::class, map)
}

data class Address(val street: String, val city: String, val country: String)
data class Customer(val id: Int, val name: String, val email: String)

fun main() {
    // Object Mapper
    val mapper = ObjectMapper()
    
    val customer = Customer(1, "สมชาย ใจดี", "somchai@email.com")
    val map = mapper.toMap(customer)
    println("To map: $map")
    
    val restored = mapper.fromMap<Customer>(map)
    println("From map: $restored")
    println("Equal: ${customer == restored}")
    
    println()
    
    // Validation via annotation
    @Target(AnnotationTarget.PROPERTY)
    @Retention(AnnotationRetention.RUNTIME)
    annotation class NotEmpty(val message: String = "must not be empty")
    
    @Target(AnnotationTarget.PROPERTY)
    @Retention(AnnotationRetention.RUNTIME)
    annotation class Min(val value: Int, val message: String = "too small")
    
    data class UserDto(
        @NotEmpty val name: String,
        @Min(18) val age: Int,
        @NotEmpty val email: String
    )
    
    fun validate(obj: Any): List<String> {
        val errors = mutableListOf<String>()
        val kClass = obj::class
        
        kClass.memberProperties.forEach { prop ->
            val value = prop.get(obj)
            
            prop.findAnnotation<NotEmpty>()?.let { ann ->
                if (value is String && value.isEmpty()) {
                    errors.add("${prop.name}: ${ann.message}")
                }
            }
            
            prop.findAnnotation<Min>()?.let { ann ->
                if (value is Int && value < ann.value) {
                    errors.add("${prop.name}: ${ann.message} (min=${ann.value})")
                }
            }
        }
        
        return errors
    }
    
    val valid = UserDto("สมชาย", 25, "somchai@email.com")
    val invalid = UserDto("", 15, "")
    
    println("Valid errors: ${validate(valid)}")
    println("Invalid errors: ${validate(invalid)}")
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Mini ORM using Reflection + Annotations

@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
annotation class Entity(val table: String = "")

@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.RUNTIME)
annotation class Id

@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.RUNTIME)
annotation class Col(
    val name: String = "",
    val nullable: Boolean = true
)

@Entity("employees")
data class Employee(
    @Id @Col("emp_id", nullable = false) val id: Int,
    @Col("full_name", nullable = false) val name: String,
    @Col("department") val department: String?,
    @Col("salary") val salary: Double,
    val notes: String = ""  // no @Col, should be ignored
)

class MiniOrm {
    fun generateCreateTable(klass: kotlin.reflect.KClass<*>): String {
        val entityAnn = klass.findAnnotation<Entity>()
            ?: throw IllegalArgumentException("Not an @Entity")
        val tableName = entityAnn.table.ifEmpty { klass.simpleName!!.lowercase() }
        
        val columns = klass.memberProperties
            .mapNotNull { prop ->
                val col = prop.findAnnotation<Col>() ?: return@mapNotNull null
                val colName = col.name.ifEmpty { prop.name }
                val type = when (prop.returnType.classifier) {
                    Int::class -> "INTEGER"
                    Long::class -> "BIGINT"
                    Double::class -> "DECIMAL(10,2)"
                    String::class -> "VARCHAR(255)"
                    Boolean::class -> "BOOLEAN"
                    else -> "TEXT"
                }
                val constraints = buildList {
                    if (!col.nullable) add("NOT NULL")
                    if (prop.findAnnotation<Id>() != null) add("PRIMARY KEY")
                }
                "$colName $type ${constraints.joinToString(" ")}".trim()
            }
        
        return "CREATE TABLE $tableName (\n  ${columns.joinToString(",\n  ")}\n);"
    }
    
    fun <T : Any> generateInsert(obj: T): String {
        val kClass = obj::class
        val entityAnn = kClass.findAnnotation<Entity>()
            ?: throw IllegalArgumentException("Not an @Entity")
        val tableName = entityAnn.table.ifEmpty { kClass.simpleName!!.lowercase() }
        
        val colValues = kClass.memberProperties
            .mapNotNull { prop ->
                val col = prop.findAnnotation<Col>() ?: return@mapNotNull null
                val colName = col.name.ifEmpty { prop.name }
                val value = prop.get(obj)
                val sqlValue = when (value) {
                    null -> "NULL"
                    is String -> "'${value.replace("'", "''")}'"
                    is Boolean -> if (value) "TRUE" else "FALSE"
                    else -> value.toString()
                }
                colName to sqlValue
            }
        
        val cols = colValues.map { it.first }.joinToString(", ")
        val vals = colValues.map { it.second }.joinToString(", ")
        
        return "INSERT INTO $tableName ($cols) VALUES ($vals);"
    }
}

fun main() {
    val orm = MiniOrm()
    
    println(orm.generateCreateTable(Employee::class))
    println()
    
    val emp = Employee(
        id = 1,
        name = "สมชาย ใจดี",
        department = "Engineering",
        salary = 50000.0
    )
    println(orm.generateInsert(emp))
    
    val emp2 = Employee(
        id = 2,
        name = "สมหญิง ดีใจ",
        department = null,
        salary = 45000.0,
        notes = "ไม่มีแผนก"
    )
    println(orm.generateInsert(emp2))
}
```

---

## สรุป Part 25

```
✅ @Annotation สร้างด้วย annotation class
✅ @Target: กำหนดว่า annotate ได้ที่ไหน
✅ @Retention: SOURCE, BINARY, RUNTIME
✅ Annotation มี parameters (val) ได้
✅ kClass.findAnnotation<T>(): ดึง annotation
✅ kClass.memberProperties: ดึงทุก properties
✅ kClass.memberFunctions: ดึงทุก functions
✅ kClass.primaryConstructor: ดึง constructor หลัก
✅ prop.get(obj): อ่านค่า property
✅ prop.setter.call(obj, value): set ค่า property
✅ ctor.callBy(mapOf(param to value)): สร้าง instance
✅ Reflection ช้ากว่าการเรียกปกติ ใช้เฉพาะเมื่อจำเป็น
```

---

*Part 25/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
