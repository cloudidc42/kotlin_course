# Part 20: Database กับ Exposed ORM

## สารบัญ
1. [Exposed คืออะไร](#exposed-คืออะไร)
2. [Setup](#setup)
3. [Schema Definition](#schema-definition)
4. [CRUD Operations](#crud-operations)
5. [Queries](#queries)
6. [Relationships](#relationships)
7. [Transactions](#transactions)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Exposed คืออะไร

Exposed คือ Kotlin SQL framework ของ JetBrains มี 2 levels:
- **DSL API**: Type-safe SQL query builder
- **DAO API**: Entity-based (คล้าย JPA/Hibernate)

---

## Setup

### build.gradle.kts

```kotlin
dependencies {
    // Exposed core
    implementation("org.jetbrains.exposed:exposed-core:0.44.1")
    implementation("org.jetbrains.exposed:exposed-dao:0.44.1")
    implementation("org.jetbrains.exposed:exposed-jdbc:0.44.1")
    implementation("org.jetbrains.exposed:exposed-java-time:0.44.1")
    
    // Database drivers
    implementation("org.xerial:sqlite-jdbc:3.44.1.0")          // SQLite
    implementation("org.postgresql:postgresql:42.7.1")          // PostgreSQL
    implementation("com.h2database:h2:2.2.224")                // H2 (in-memory)
    implementation("mysql:mysql-connector-java:8.0.33")         // MySQL
    
    // Connection pool
    implementation("com.zaxxer:HikariCP:5.1.0")
}
```

---

## Schema Definition

### DSL Style (Table objects)

```kotlin
import org.jetbrains.exposed.sql.*
import org.jetbrains.exposed.sql.javatime.*
import java.time.LocalDateTime

// DSL: Table objects
object Users : Table("users") {
    val id = integer("id").autoIncrement()
    val name = varchar("name", 100)
    val email = varchar("email", 200).uniqueIndex()
    val age = integer("age").nullable()
    val role = varchar("role", 20).default("user")
    val createdAt = datetime("created_at").defaultExpression(CurrentDateTime)
    val isActive = bool("is_active").default(true)
    
    override val primaryKey = PrimaryKey(id)
}

object Products : Table("products") {
    val id = integer("id").autoIncrement()
    val name = varchar("name", 200)
    val price = decimal("price", 10, 2)
    val stock = integer("stock").default(0)
    val categoryId = integer("category_id").references(Categories.id)
    
    override val primaryKey = PrimaryKey(id)
}

object Categories : Table("categories") {
    val id = integer("id").autoIncrement()
    val name = varchar("name", 100).uniqueIndex()
    
    override val primaryKey = PrimaryKey(id)
}

object Orders : Table("orders") {
    val id = integer("id").autoIncrement()
    val userId = integer("user_id").references(Users.id)
    val totalAmount = decimal("total_amount", 10, 2)
    val status = varchar("status", 20).default("pending")
    val createdAt = datetime("created_at").defaultExpression(CurrentDateTime)
    
    override val primaryKey = PrimaryKey(id)
}
```

### DAO Style (Entity classes)

```kotlin
import org.jetbrains.exposed.dao.*
import org.jetbrains.exposed.dao.id.*

// DAO: IntIdTable + IntEntity
object UserTable : IntIdTable("users") {
    val name = varchar("name", 100)
    val email = varchar("email", 200).uniqueIndex()
    val age = integer("age").nullable()
    val role = varchar("role", 20).default("user")
    val createdAt = datetime("created_at").defaultExpression(CurrentDateTime)
    val isActive = bool("is_active").default(true)
}

class User(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<User>(UserTable)
    
    var name by UserTable.name
    var email by UserTable.email
    var age by UserTable.age
    var role by UserTable.role
    var createdAt by UserTable.createdAt
    var isActive by UserTable.isActive
    
    override fun toString() = "User($id, $name, $email)"
}

object ProductTable : IntIdTable("products") {
    val name = varchar("name", 200)
    val price = decimal("price", 10, 2)
    val stock = integer("stock").default(0)
    val categoryId = reference("category_id", CategoryTable)
}

class Product(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<Product>(ProductTable)
    
    var name by ProductTable.name
    var price by ProductTable.price
    var stock by ProductTable.stock
    var category by Category referencedOn ProductTable.categoryId
}

object CategoryTable : IntIdTable("categories") {
    val name = varchar("name", 100).uniqueIndex()
}

class Category(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<Category>(CategoryTable)
    
    var name by CategoryTable.name
    val products by Product referrersOn ProductTable.categoryId
}
```

---

## CRUD Operations

### DSL CRUD

```kotlin
import org.jetbrains.exposed.sql.*
import org.jetbrains.exposed.sql.transactions.transaction

fun setupDatabase() {
    Database.connect("jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;", "org.h2.Driver")
    
    transaction {
        SchemaUtils.create(Users, Categories, Products, Orders)
    }
}

fun main() {
    setupDatabase()
    
    transaction {
        // INSERT
        val userId = Users.insert {
            it[name] = "สมชาย"
            it[email] = "somchai@email.com"
            it[age] = 25
        } get Users.id
        
        println("Created user ID: $userId")
        
        // INSERT multiple
        Users.batchInsert(listOf(
            Triple("สมหญิง", "somying@email.com", 30),
            Triple("สมศักดิ์", "somsak@email.com", 28)
        )) { (name, email, age) ->
            this[Users.name] = name
            this[Users.email] = email
            this[Users.age] = age
        }
        
        // SELECT all
        val allUsers = Users.selectAll().map {
            mapOf(
                "id" to it[Users.id],
                "name" to it[Users.name],
                "email" to it[Users.email]
            )
        }
        println("All users: $allUsers")
        
        // SELECT with WHERE
        val user = Users.select { Users.id eq userId }.firstOrNull()
        println("Found: ${user?.get(Users.name)}")
        
        // SELECT specific columns
        val names = Users.slice(Users.name, Users.email)
            .selectAll()
            .map { "${it[Users.name]} <${it[Users.email]}>" }
        println("Names: $names")
        
        // UPDATE
        Users.update({ Users.id eq userId }) {
            it[age] = 26
            it[role] = "admin"
        }
        
        // DELETE
        Users.deleteWhere { Users.email like "%somsak%" }
        
        println("Remaining: ${Users.selectAll().count()}")
    }
}
```

### DAO CRUD

```kotlin
fun daoExample() {
    transaction {
        SchemaUtils.create(CategoryTable, ProductTable, UserTable)
        
        // Create
        val user = User.new {
            name = "สมชาย"
            email = "somchai@email.com"
            age = 25
        }
        println("Created: $user")
        
        // Create category and products
        val electronics = Category.new { name = "Electronics" }
        
        val laptop = Product.new {
            name = "Laptop Pro"
            price = 25000.toBigDecimal()
            stock = 10
            category = electronics
        }
        
        val phone = Product.new {
            name = "Smartphone X"
            price = 15000.toBigDecimal()
            stock = 25
            category = electronics
        }
        
        // Read
        val foundUser = User.findById(user.id)
        println("Found: ${foundUser?.name}")
        
        // Read all
        User.all().forEach { println(it) }
        
        // Update
        user.age = 26
        user.role = "admin"
        // Changes saved automatically in transaction
        
        // Find with condition
        val admins = User.find { UserTable.role eq "admin" }.toList()
        println("Admins: ${admins.map { it.name }}")
        
        // Delete
        User.find { UserTable.email like "%admin%" }.forEach { it.delete() }
        
        // Products in category
        electronics.products.forEach { 
            println("  ${it.name}: ${it.price}")
        }
    }
}
```

---

## Queries

```kotlin
fun advancedQueries() {
    transaction {
        // ORDER BY
        val sorted = Users.selectAll()
            .orderBy(Users.name to SortOrder.ASC)
            .toList()
        
        // LIMIT / OFFSET
        val page1 = Users.selectAll()
            .limit(10, offset = 0)
            .toList()
        
        val page2 = Users.selectAll()
            .limit(10, offset = 10)
            .toList()
        
        // COUNT
        val count = Users.selectAll().count()
        println("Total: $count")
        
        // WHERE conditions
        val activeAdmins = Users.select {
            (Users.isActive eq true) and (Users.role eq "admin")
        }.toList()
        
        val youngOrOld = Users.select {
            (Users.age less 25) or (Users.age greater 60)
        }.toList()
        
        // LIKE / IN
        val byEmail = Users.select { 
            Users.email like "%@gmail.com" 
        }.toList()
        
        val byRole = Users.select {
            Users.role inList listOf("admin", "moderator")
        }.toList()
        
        // IS NULL / IS NOT NULL
        val withAge = Users.select { Users.age.isNotNull() }.toList()
        val withoutAge = Users.select { Users.age.isNull() }.toList()
        
        // BETWEEN
        val middleAged = Users.select {
            Users.age.between(25, 45)
        }.toList()
        
        // Aggregate functions
        val avgAge = Users.slice(Users.age.avg()).selectAll().first()[Users.age.avg()]
        val maxAge = Users.slice(Users.age.max()).selectAll().first()[Users.age.max()]
        val minAge = Users.slice(Users.age.min()).selectAll().first()[Users.age.min()]
        
        println("Avg age: $avgAge, Max: $maxAge, Min: $minAge")
        
        // GROUP BY
        val countByRole = Users
            .slice(Users.role, Users.id.count())
            .selectAll()
            .groupBy(Users.role)
            .map { "${it[Users.role]}: ${it[Users.id.count()]}" }
        
        println("By role: $countByRole")
        
        // JOIN
        val userOrders = (Users innerJoin Orders)
            .slice(Users.name, Orders.totalAmount, Orders.status)
            .selectAll()
            .map { 
                "${it[Users.name]}: ${it[Orders.totalAmount]} (${it[Orders.status]})"
            }
        
        // Subquery
        val usersWithOrders = Users.select {
            Users.id inSubQuery Orders.slice(Orders.userId).selectAll()
        }.toList()
        
        // Raw query (when needed)
        exec("SELECT COUNT(*) FROM users") { rs ->
            while (rs.next()) {
                println("Count: ${rs.getInt(1)}")
            }
        }
    }
}
```

---

## Relationships

```kotlin
// Many-to-Many with junction table
object UserRoles : Table("user_roles") {
    val userId = integer("user_id").references(Users.id)
    val roleId = integer("role_id") // reference to Roles table
    
    override val primaryKey = PrimaryKey(userId, roleId)
}

object Tags : IntIdTable("tags") {
    val name = varchar("name", 50)
}

object ProductTags : Table("product_tags") {
    val productId = reference("product_id", ProductTable)
    val tagId = reference("tag_id", Tags)
    
    override val primaryKey = PrimaryKey(productId, tagId)
}

// DAO with relationships
class ProductWithTags(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<ProductWithTags>(ProductTable)
    
    var name by ProductTable.name
    var price by ProductTable.price
    var stock by ProductTable.stock
    var tags by Tag via ProductTags
}

class Tag(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<Tag>(Tags)
    var name by Tags.name
}

fun relationshipExample() {
    transaction {
        SchemaUtils.create(Tags, ProductTags)
        
        val tag1 = Tag.new { name = "sale" }
        val tag2 = Tag.new { name = "new" }
        val tag3 = Tag.new { name = "featured" }
        
        val product = ProductWithTags.findById(1) ?: return@transaction
        
        // Add tags
        product.tags = SizedCollection(listOf(tag1, tag2))
        
        // Query products with specific tag
        val saleProducts = ProductWithTags.wrapRows(
            ProductTable
                .innerJoin(ProductTags)
                .innerJoin(Tags)
                .select { Tags.name eq "sale" }
        ).toList()
        
        println("Sale products: ${saleProducts.map { it.name }}")
        
        // Eager loading (avoid N+1)
        val allProducts = ProductWithTags.all()
            .with(ProductWithTags::tags)  // eager load
            .toList()
        
        allProducts.forEach { p ->
            println("${p.name}: ${p.tags.map { it.name }}")
        }
    }
}
```

---

## Transactions

```kotlin
import org.jetbrains.exposed.sql.transactions.transaction

fun transactionExample() {
    // Basic transaction
    transaction {
        // All operations in this block are in one transaction
        Users.insert { it[name] = "Test"; it[email] = "test@test.com" }
        
        // If any exception occurs, all changes are rolled back
        throw RuntimeException("Test rollback")
    }
    // Above transaction is rolled back
    
    // Nested transactions
    transaction {
        Users.insert { it[name] = "User A"; it[email] = "a@test.com" }
        
        transaction {
            // Nested = same transaction by default
            Users.insert { it[name] = "User B"; it[email] = "b@test.com" }
        }
    }
    
    // Manual rollback
    transaction {
        Users.insert { it[name] = "Temp"; it[email] = "temp@test.com" }
        rollback()  // explicit rollback
    }
    
    // Exception handling in transaction
    try {
        transaction {
            // This will fail if email is duplicate
            Users.insert { it[name] = "Dup"; it[email] = "existing@test.com" }
        }
    } catch (e: Exception) {
        println("Transaction failed: ${e.message}")
    }
    
    // Coroutine transactions (for Ktor)
    // import org.jetbrains.exposed.sql.transactions.experimental.newSuspendedTransaction
    // 
    // suspend fun createUser(name: String): Int {
    //     return newSuspendedTransaction(Dispatchers.IO) {
    //         Users.insert {
    //             it[Users.name] = name
    //             it[Users.email] = "$name@test.com"
    //         } get Users.id
    //     }
    // }
}
```

---

## แบบฝึกหัด - Blog System

```kotlin
import org.jetbrains.exposed.dao.*
import org.jetbrains.exposed.dao.id.*
import org.jetbrains.exposed.sql.*
import org.jetbrains.exposed.sql.transactions.transaction
import org.jetbrains.exposed.sql.javatime.*

// Schema
object AuthorTable : IntIdTable("authors") {
    val name = varchar("name", 100)
    val email = varchar("email", 200).uniqueIndex()
    val bio = text("bio").nullable()
}

object PostTable : IntIdTable("posts") {
    val title = varchar("title", 200)
    val content = text("content")
    val authorId = reference("author_id", AuthorTable)
    val published = bool("published").default(false)
    val createdAt = datetime("created_at").defaultExpression(CurrentDateTime)
    val viewCount = integer("view_count").default(0)
}

object CommentTable : IntIdTable("comments") {
    val postId = reference("post_id", PostTable)
    val authorName = varchar("author_name", 100)
    val content = text("content")
    val createdAt = datetime("created_at").defaultExpression(CurrentDateTime)
}

// Entities
class Author(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<Author>(AuthorTable)
    var name by AuthorTable.name
    var email by AuthorTable.email
    var bio by AuthorTable.bio
    val posts by Post referrersOn PostTable.authorId
}

class Post(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<Post>(PostTable)
    var title by PostTable.title
    var content by PostTable.content
    var author by Author referencedOn PostTable.authorId
    var published by PostTable.published
    var createdAt by PostTable.createdAt
    var viewCount by PostTable.viewCount
    val comments by Comment referrersOn CommentTable.postId
}

class Comment(id: EntityID<Int>) : IntEntity(id) {
    companion object : IntEntityClass<Comment>(CommentTable)
    var post by Post referencedOn CommentTable.postId
    var authorName by CommentTable.authorName
    var content by CommentTable.content
    var createdAt by CommentTable.createdAt
}

// Blog Service
class BlogService {
    fun createAuthor(name: String, email: String, bio: String? = null): Author =
        transaction {
            Author.new {
                this.name = name
                this.email = email
                this.bio = bio
            }
        }
    
    fun createPost(authorId: Int, title: String, content: String): Post =
        transaction {
            val author = Author.findById(authorId) ?: error("Author not found")
            Post.new {
                this.title = title
                this.content = content
                this.author = author
            }
        }
    
    fun publishPost(postId: Int): Post =
        transaction {
            val post = Post.findById(postId) ?: error("Post not found")
            post.published = true
            post
        }
    
    fun getPublishedPosts(): List<Map<String, Any?>> =
        transaction {
            Post.find { PostTable.published eq true }
                .orderBy(PostTable.createdAt to SortOrder.DESC)
                .map { post ->
                    mapOf(
                        "id" to post.id.value,
                        "title" to post.title,
                        "author" to post.author.name,
                        "comments" to post.comments.count(),
                        "views" to post.viewCount
                    )
                }
        }
    
    fun addComment(postId: Int, authorName: String, content: String): Comment =
        transaction {
            val post = Post.findById(postId) ?: error("Post not found")
            Comment.new {
                this.post = post
                this.authorName = authorName
                this.content = content
            }
        }
    
    fun incrementView(postId: Int) =
        transaction {
            PostTable.update({ PostTable.id eq postId }) {
                with(SqlExpressionBuilder) {
                    it[viewCount] = viewCount + 1
                }
            }
        }
}

fun main() {
    // Setup H2 in-memory database
    Database.connect("jdbc:h2:mem:blog;DB_CLOSE_DELAY=-1", "org.h2.Driver")
    
    transaction {
        SchemaUtils.create(AuthorTable, PostTable, CommentTable)
    }
    
    val service = BlogService()
    
    val author1 = service.createAuthor("สมชาย", "somchai@email.com", "นักเขียนโปรแกรม Kotlin")
    val author2 = service.createAuthor("สมหญิง", "somying@email.com")
    
    val post1 = service.createPost(author1.id.value, "Kotlin Coroutines คืออะไร", "Coroutines เป็น...")
    val post2 = service.createPost(author1.id.value, "Ktor Tutorial", "Ktor เป็น web framework...")
    val post3 = service.createPost(author2.id.value, "Clean Code ใน Kotlin", "หลักการ Clean Code...")
    
    service.publishPost(post1.id.value)
    service.publishPost(post3.id.value)
    
    service.addComment(post1.id.value, "สมศักดิ์", "บทความดีมาก!")
    service.addComment(post1.id.value, "สมพร", "อ่านง่ายเข้าใจได้เลย")
    service.addComment(post3.id.value, "สมเกียรติ", "ขอบคุณครับ")
    
    repeat(10) { service.incrementView(post1.id.value) }
    repeat(5) { service.incrementView(post3.id.value) }
    
    println("=== Published Posts ===")
    service.getPublishedPosts().forEach { post ->
        println("${post["title"]} by ${post["author"]}")
        println("  Comments: ${post["comments"]}, Views: ${post["views"]}")
    }
}
```

---

## สรุป Part 20

```
✅ Exposed: Type-safe Kotlin SQL ORM
✅ DSL API: Table objects, SQL-like queries
✅ DAO API: Entity classes (ORM-style)
✅ Database.connect(): เชื่อมต่อ database
✅ transaction { }: wrap operations
✅ SchemaUtils.create(Table): สร้าง tables
✅ Table.insert { }: insert row
✅ Table.selectAll(): select all
✅ Table.select { condition }: filter
✅ Table.update({ condition }) { }: update
✅ Table.deleteWhere { }: delete
✅ join, orderBy, limit, groupBy
✅ Entity.new { }: DAO create
✅ Entity.findById(): DAO find
✅ SizedCollection: many-to-many
✅ newSuspendedTransaction สำหรับ coroutines
```

---

*Part 20/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
