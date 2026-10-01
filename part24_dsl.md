# Part 24: DSL (Domain Specific Language)

## สารบัญ
1. [DSL คืออะไร](#dsl-คืออะไร)
2. [Function Literals with Receiver](#function-literals-with-receiver)
3. [สร้าง HTML DSL](#สร้าง-html-dsl)
4. [Builder Pattern DSL](#builder-pattern-dsl)
5. [Configuration DSL](#configuration-dsl)
6. [Test DSL](#test-dsl)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## DSL คืออะไร

```kotlin
// DSL = Domain Specific Language
// ภาษาที่ออกแบบมาสำหรับ domain เฉพาะ
// ใน Kotlin: internal DSL ที่ใช้ syntax ของ Kotlin เอง

// ตัวอย่าง standard code vs DSL code

// Standard code
val html1 = StringBuilder()
html1.append("<html>")
html1.append("<body>")
html1.append("<h1>Hello</h1>")
html1.append("</body>")
html1.append("</html>")

// DSL style (เราจะสร้างสิ่งนี้)
val html2 = html {
    body {
        h1 { +"Hello" }
        p { +"This is a paragraph" }
    }
}

// อีกตัวอย่าง: Gradle DSL
/*
dependencies {
    implementation("org.jetbrains.kotlin:kotlin-stdlib")
    testImplementation("junit:junit:4.13")
}
*/

// Ktor DSL
/*
routing {
    get("/hello") {
        call.respondText("Hello!")
    }
}
*/
```

---

## Function Literals with Receiver

```kotlin
// Extension function type: T.() -> R
// lambda นี้สามารถเรียก members ของ T โดยตรง

// Simple example
fun buildString2(action: StringBuilder.() -> Unit): String {
    val sb = StringBuilder()
    sb.action()  // หรือ action(sb)
    return sb.toString()
}

fun main() {
    val result = buildString2 {
        append("Hello")
        append(", ")
        append("World")
        append("!")
    }
    println(result)
    
    // apply, also, run, with, let เป็น function literals with receiver
    val list = mutableListOf<Int>().apply {
        add(1)
        add(2)
        add(3)
    }
    println(list)
    
    // with: call multiple methods on object
    val formatted = with(StringBuilder()) {
        append("Name: สมชาย")
        append(", Age: 25")
        append(", City: Bangkok")
        toString()
    }
    println(formatted)
    
    // Custom DSL builder
    class HtmlTag(val tag: String) {
        private val children = mutableListOf<String>()
        private var text = ""
        
        fun text(content: String) { text = content }
        
        fun tag(name: String, block: HtmlTag.() -> Unit): HtmlTag {
            val child = HtmlTag(name)
            child.block()
            children.add(child.render())
            return child
        }
        
        fun render(): String {
            val content = if (text.isNotEmpty()) text else children.joinToString("\n")
            return "<$tag>$content</$tag>"
        }
    }
    
    fun html(block: HtmlTag.() -> Unit): String {
        val root = HtmlTag("html")
        root.block()
        return root.render()
    }
    
    val page = html {
        tag("body") {
            tag("h1") { text("สวัสดี") }
            tag("p") { text("ยินดีต้อนรับ") }
        }
    }
    println(page)
}
```

---

## สร้าง HTML DSL

```kotlin
// Type-safe HTML builder

@DslMarker
annotation class HtmlDsl

@HtmlDsl
abstract class HtmlElement {
    protected val children = mutableListOf<HtmlElement>()
    protected val attributes = mutableMapOf<String, String>()
    
    abstract fun render(indent: Int = 0): String
    
    protected fun String.attr() = attributes.entries
        .joinToString(" ") { (k, v) -> "$k=\"$v\"" }
        .let { if (it.isNotEmpty()) " $it" else "" }
    
    protected fun indented(level: Int) = "  ".repeat(level)
}

@HtmlDsl
class TextNode(private val text: String) : HtmlElement() {
    override fun render(indent: Int) = "${"  ".repeat(indent)}$text"
}

@HtmlDsl
open class Tag(private val name: String) : HtmlElement() {
    operator fun String.unaryPlus() {
        children.add(TextNode(this))
    }
    
    fun attr(key: String, value: String) { attributes[key] = value }
    
    override fun render(indent: Int): String {
        val ind = indented(indent)
        val attrStr = attributes.entries.joinToString(" ") { (k, v) -> "$k=\"$v\"" }
        val openTag = if (attrStr.isEmpty()) "<$name>" else "<$name $attrStr>"
        
        return if (children.isEmpty()) {
            "$ind$openTag</$name>"
        } else if (children.all { it is TextNode }) {
            "$ind$openTag${children.joinToString("") { it.render(0) }}</$name>"
        } else {
            "$ind$openTag\n${children.joinToString("\n") { it.render(indent + 1) }}\n$ind</$name>"
        }
    }
    
    protected fun <T : Tag> initTag(tag: T, block: T.() -> Unit): T {
        tag.block()
        children.add(tag)
        return tag
    }
}

@HtmlDsl
class Html : Tag("html") {
    fun head(block: Head.() -> Unit) = initTag(Head(), block)
    fun body(block: Body.() -> Unit) = initTag(Body(), block)
}

@HtmlDsl
class Head : Tag("head") {
    fun title(block: Tag.() -> Unit) = initTag(Tag("title"), block)
    fun meta(name: String, content: String) {
        val tag = Tag("meta")
        tag.attr("name", name)
        tag.attr("content", content)
        children.add(tag)
    }
    fun link(rel: String, href: String) {
        val tag = Tag("link")
        tag.attr("rel", rel)
        tag.attr("href", href)
        children.add(tag)
    }
}

@HtmlDsl
class Body : Tag("body") {
    fun h1(block: Tag.() -> Unit) = initTag(Tag("h1"), block)
    fun h2(block: Tag.() -> Unit) = initTag(Tag("h2"), block)
    fun h3(block: Tag.() -> Unit) = initTag(Tag("h3"), block)
    fun p(block: Tag.() -> Unit) = initTag(Tag("p"), block)
    fun div(id: String? = null, cls: String? = null, block: Div.() -> Unit): Div {
        val div = Div()
        id?.let { div.attr("id", it) }
        cls?.let { div.attr("class", it) }
        div.block()
        children.add(div)
        return div
    }
    fun ul(block: Ul.() -> Unit) = initTag(Ul(), block)
    fun ol(block: Ol.() -> Unit) = initTag(Ol(), block)
    fun a(href: String, block: Tag.() -> Unit): Tag {
        val tag = Tag("a")
        tag.attr("href", href)
        tag.block()
        children.add(tag)
        return tag
    }
    fun img(src: String, alt: String = "") {
        val tag = Tag("img")
        tag.attr("src", src)
        tag.attr("alt", alt)
        children.add(tag)
    }
}

@HtmlDsl
class Div : Tag("div") {
    fun p(block: Tag.() -> Unit) = initTag(Tag("p"), block)
    fun span(block: Tag.() -> Unit) = initTag(Tag("span"), block)
    fun div(block: Div.() -> Unit) = initTag(Div(), block)
    fun strong(block: Tag.() -> Unit) = initTag(Tag("strong"), block)
}

@HtmlDsl
class Ul : Tag("ul") {
    fun li(block: Tag.() -> Unit) = initTag(Tag("li"), block)
}

@HtmlDsl
class Ol : Tag("ol") {
    fun li(block: Tag.() -> Unit) = initTag(Tag("li"), block)
}

fun html(block: Html.() -> Unit): String {
    val html = Html()
    html.block()
    return "<!DOCTYPE html>\n${html.render()}"
}

fun main() {
    val page = html {
        head {
            title { +"Kotlin DSL Demo" }
            meta("viewport", "width=device-width, initial-scale=1")
            link("stylesheet", "style.css")
        }
        body {
            h1 { +"ยินดีต้อนรับสู่ Kotlin DSL" }
            
            div("main", "container") {
                p { +"Kotlin DSL ทำให้ code อ่านง่ายและเขียนได้สะดวก" }
                
                div {
                    strong { +"Features:" }
                    ul {
                        li { +"Type-safe HTML generation" }
                        li { +"Compile-time validation" }
                        li { +"IDE support" }
                        li { +"No reflection needed" }
                    }
                }
                
                a("https://kotlinlang.org") { +"Visit Kotlin website" }
            }
        }
    }
    
    println(page)
}
```

---

## Builder Pattern DSL

```kotlin
// ใช้ DSL สร้าง complex objects

data class EmailAddress(val address: String, val name: String? = null) {
    override fun toString() = if (name != null) "$name <$address>" else address
}

data class Attachment(val filename: String, val content: ByteArray, val mimeType: String)

data class Email(
    val from: EmailAddress,
    val to: List<EmailAddress>,
    val cc: List<EmailAddress>,
    val bcc: List<EmailAddress>,
    val subject: String,
    val body: String,
    val isHtml: Boolean,
    val attachments: List<Attachment>
)

@DslMarker
annotation class EmailDsl

@EmailDsl
class EmailBuilder {
    private var from: EmailAddress? = null
    private val toList = mutableListOf<EmailAddress>()
    private val ccList = mutableListOf<EmailAddress>()
    private val bccList = mutableListOf<EmailAddress>()
    private var subject: String = ""
    private var body: String = ""
    private var isHtml: Boolean = false
    private val attachments = mutableListOf<Attachment>()
    
    fun from(address: String, name: String? = null) {
        from = EmailAddress(address, name)
    }
    
    fun to(address: String, name: String? = null) {
        toList.add(EmailAddress(address, name))
    }
    
    fun to(vararg addresses: String) {
        addresses.forEach { toList.add(EmailAddress(it)) }
    }
    
    fun cc(address: String, name: String? = null) {
        ccList.add(EmailAddress(address, name))
    }
    
    fun bcc(address: String, name: String? = null) {
        bccList.add(EmailAddress(address, name))
    }
    
    fun subject(text: String) { subject = text }
    
    fun textBody(content: String) {
        body = content
        isHtml = false
    }
    
    fun htmlBody(content: String) {
        body = content
        isHtml = true
    }
    
    fun attachment(filename: String, content: ByteArray, mimeType: String = "application/octet-stream") {
        attachments.add(Attachment(filename, content, mimeType))
    }
    
    fun build(): Email {
        requireNotNull(from) { "From address is required" }
        require(toList.isNotEmpty()) { "At least one recipient required" }
        require(subject.isNotBlank()) { "Subject is required" }
        
        return Email(from!!, toList, ccList, bccList, subject, body, isHtml, attachments)
    }
}

fun email(block: EmailBuilder.() -> Unit): Email {
    val builder = EmailBuilder()
    builder.block()
    return builder.build()
}

// SQL Query DSL
@DslMarker
annotation class SqlDsl

@SqlDsl
class SelectBuilder {
    private var table: String = ""
    private val columns = mutableListOf<String>()
    private val conditions = mutableListOf<String>()
    private var orderBy: String? = null
    private var limit: Int? = null
    private var offset: Int? = null
    
    fun from(tableName: String) { table = tableName }
    fun select(vararg cols: String) { columns.addAll(cols) }
    fun select(col: String) { columns.add(col) }
    
    fun where(condition: String) { conditions.add(condition) }
    fun and(condition: String) { conditions.add(condition) }
    
    fun orderBy(col: String, desc: Boolean = false) {
        orderBy = if (desc) "$col DESC" else col
    }
    
    fun limit(n: Int) { limit = n }
    fun offset(n: Int) { offset = n }
    
    fun build(): String {
        val selectCols = if (columns.isEmpty()) "*" else columns.joinToString(", ")
        val sql = StringBuilder("SELECT $selectCols FROM $table")
        
        if (conditions.isNotEmpty()) {
            sql.append(" WHERE ${conditions.joinToString(" AND ")}")
        }
        
        orderBy?.let { sql.append(" ORDER BY $it") }
        limit?.let { sql.append(" LIMIT $it") }
        offset?.let { sql.append(" OFFSET $it") }
        
        return sql.toString()
    }
}

fun query(block: SelectBuilder.() -> Unit): String {
    val builder = SelectBuilder()
    builder.block()
    return builder.build()
}

fun main() {
    // Email DSL
    val mail = email {
        from("sender@company.com", "ระบบแจ้งเตือน")
        to("user@email.com", "สมชาย")
        to("admin@company.com")
        cc("supervisor@company.com", "หัวหน้า")
        subject("แจ้งเตือน: รายการสั่งซื้อใหม่")
        htmlBody("""
            <h2>มีรายการสั่งซื้อใหม่</h2>
            <p>คำสั่งซื้อ #12345 รอการยืนยัน</p>
        """.trimIndent())
    }
    
    println("Email:")
    println("  From: ${mail.from}")
    println("  To: ${mail.to}")
    println("  Subject: ${mail.subject}")
    println("  HTML: ${mail.isHtml}")
    
    println()
    
    // SQL DSL
    val sql1 = query {
        select("id", "name", "email")
        from("users")
        where("age > 18")
        and("active = true")
        orderBy("name")
        limit(10)
    }
    println("SQL: $sql1")
    
    val sql2 = query {
        from("products")
        where("price < 1000")
        orderBy("price", desc = true)
        limit(5)
        offset(10)
    }
    println("SQL: $sql2")
    
    val sql3 = query {
        from("orders")
    }
    println("SQL: $sql3")
}
```

---

## Configuration DSL

```kotlin
// Server configuration DSL

data class SslConfig(
    val enabled: Boolean,
    val keyStorePath: String,
    val keyStorePassword: String,
    val port: Int
)

data class DatabaseConfig(
    val host: String,
    val port: Int,
    val name: String,
    val username: String,
    val password: String,
    val maxConnections: Int,
    val timeout: Long
)

data class ServerConfig(
    val host: String,
    val port: Int,
    val ssl: SslConfig?,
    val database: DatabaseConfig,
    val debug: Boolean,
    val corsOrigins: List<String>
)

@DslMarker
annotation class ConfigDsl

@ConfigDsl
class SslConfigBuilder {
    var enabled = true
    var keyStorePath = ""
    var keyStorePassword = ""
    var port = 443
    
    fun build() = SslConfig(enabled, keyStorePath, keyStorePassword, port)
}

@ConfigDsl
class DatabaseConfigBuilder {
    var host = "localhost"
    var port = 5432
    var name = ""
    var username = ""
    var password = ""
    var maxConnections = 10
    var timeout = 30000L
    
    fun build(): DatabaseConfig {
        require(name.isNotBlank()) { "Database name required" }
        return DatabaseConfig(host, port, name, username, password, maxConnections, timeout)
    }
}

@ConfigDsl
class ServerConfigBuilder {
    var host = "0.0.0.0"
    var port = 8080
    var debug = false
    private var ssl: SslConfig? = null
    private lateinit var database: DatabaseConfig
    private val corsOrigins = mutableListOf<String>()
    
    fun ssl(block: SslConfigBuilder.() -> Unit) {
        ssl = SslConfigBuilder().apply(block).build()
    }
    
    fun database(block: DatabaseConfigBuilder.() -> Unit) {
        database = DatabaseConfigBuilder().apply(block).build()
    }
    
    fun cors(vararg origins: String) {
        corsOrigins.addAll(origins)
    }
    
    fun build(): ServerConfig {
        require(::database.isInitialized) { "Database config required" }
        return ServerConfig(host, port, ssl, database, debug, corsOrigins)
    }
}

fun serverConfig(block: ServerConfigBuilder.() -> Unit): ServerConfig {
    return ServerConfigBuilder().apply(block).build()
}

fun main() {
    val config = serverConfig {
        host = "0.0.0.0"
        port = 8080
        debug = true
        
        ssl {
            enabled = true
            keyStorePath = "/etc/ssl/keystore.jks"
            keyStorePassword = "secret"
            port = 8443
        }
        
        database {
            host = "db.internal"
            port = 5432
            name = "myapp"
            username = "app_user"
            password = "db_secret"
            maxConnections = 20
            timeout = 60000L
        }
        
        cors("https://myapp.com", "https://admin.myapp.com")
    }
    
    println("Server: ${config.host}:${config.port}")
    println("Debug: ${config.debug}")
    println("SSL: ${config.ssl?.enabled} on port ${config.ssl?.port}")
    println("Database: ${config.database.host}:${config.database.port}/${config.database.name}")
    println("CORS: ${config.corsOrigins}")
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง Menu DSL สำหรับ restaurant

data class MenuItem(
    val name: String,
    val description: String,
    val price: Double,
    val category: String,
    val isVegetarian: Boolean = false,
    val allergens: List<String> = emptyList(),
    val spiceLevel: Int = 0  // 0-3
)

data class MenuSection(val name: String, val items: List<MenuItem>)
data class Menu(val restaurantName: String, val sections: List<MenuSection>) {
    val allItems get() = sections.flatMap { it.items }
    val totalItems get() = allItems.size
    
    fun vegetarianOptions() = allItems.filter { it.isVegetarian }
    fun byMaxPrice(maxPrice: Double) = allItems.filter { it.price <= maxPrice }
}

@DslMarker
annotation class MenuDsl

@MenuDsl
class MenuItemBuilder(private val category: String) {
    var name = ""
    var description = ""
    var price = 0.0
    var isVegetarian = false
    var spiceLevel = 0
    private val allergens = mutableListOf<String>()
    
    fun allergen(vararg items: String) { allergens.addAll(items) }
    
    fun build() = MenuItem(name, description, price, category, isVegetarian, allergens, spiceLevel)
}

@MenuDsl
class MenuSectionBuilder(private val sectionName: String) {
    private val items = mutableListOf<MenuItem>()
    
    fun item(block: MenuItemBuilder.() -> Unit) {
        items.add(MenuItemBuilder(sectionName).apply(block).build())
    }
    
    fun build() = MenuSection(sectionName, items)
}

@MenuDsl
class MenuBuilder(private val restaurantName: String) {
    private val sections = mutableListOf<MenuSection>()
    
    fun section(name: String, block: MenuSectionBuilder.() -> Unit) {
        sections.add(MenuSectionBuilder(name).apply(block).build())
    }
    
    fun build() = Menu(restaurantName, sections)
}

fun menu(restaurantName: String, block: MenuBuilder.() -> Unit): Menu {
    return MenuBuilder(restaurantName).apply(block).build()
}

fun main() {
    val restaurantMenu = menu("ร้านอาหารไทยดีใจ") {
        section("อาหารจานหลัก") {
            item {
                name = "ผัดไทยกุ้งสด"
                description = "ผัดไทยกุ้งแม่น้ำสด ใส่ถั่วงอก ต้นหอม"
                price = 120.0
                spiceLevel = 1
                allergen("กุ้ง", "ถั่ว", "ไข่")
            }
            item {
                name = "ต้มยำกุ้ง"
                description = "ต้มยำน้ำข้น รสเผ็ดร้อนจัดจ้าน"
                price = 150.0
                spiceLevel = 3
                allergen("กุ้ง")
            }
            item {
                name = "ผัดกะเพราเต้าหู้"
                description = "เต้าหู้แข็งผัดกะเพราใบสด"
                price = 90.0
                isVegetarian = true
                spiceLevel = 2
                allergen("ถั่วเหลือง")
            }
        }
        
        section("อาหารเรียกน้ำย่อย") {
            item {
                name = "ปอเปี๊ยะทอด"
                description = "ปอเปี๊ยะไส้ผักและหมู กรอบอร่อย"
                price = 60.0
            }
            item {
                name = "ผัก Salad ไทย"
                description = "ผักสดผสมน้ำสลัดงา"
                price = 80.0
                isVegetarian = true
            }
        }
        
        section("เครื่องดื่ม") {
            item {
                name = "น้ำมะนาว"
                description = "มะนาวสดหวานเย็น"
                price = 35.0
                isVegetarian = true
            }
            item {
                name = "ชาไทยเย็น"
                description = "ชาไทยหอมหวาน"
                price = 45.0
                isVegetarian = true
            }
        }
    }
    
    println("=== ${restaurantMenu.restaurantName} ===")
    println("รายการทั้งหมด: ${restaurantMenu.totalItems} รายการ\n")
    
    restaurantMenu.sections.forEach { section ->
        println("--- ${section.name} ---")
        section.items.forEach { item ->
            val veg = if (item.isVegetarian) " 🌱" else ""
            val spice = "🌶️".repeat(item.spiceLevel)
            println("  ${item.name} - ฿${item.price}$veg $spice")
            println("    ${item.description}")
            if (item.allergens.isNotEmpty()) {
                println("    ⚠️ สารก่อภูมิแพ้: ${item.allergens.joinToString(", ")}")
            }
        }
        println()
    }
    
    println("เมนูมังสวิรัติ:")
    restaurantMenu.vegetarianOptions().forEach { println("  - ${it.name}") }
    
    println("\nเมนูราคาไม่เกิน 100 บาท:")
    restaurantMenu.byMaxPrice(100.0).forEach { println("  - ${it.name} (฿${it.price})") }
}
```

---

## สรุป Part 24

```
✅ DSL = ภาษา/syntax พิเศษสำหรับ domain เฉพาะ
✅ Function literal with receiver: T.() -> R
✅ @DslMarker: ป้องกัน accidentally call outer scope
✅ apply/with/run ใช้ receiver ภายใน lambda
✅ initTag pattern: สร้าง tag, apply block, เพิ่มใน children
✅ Builder DSL: สร้าง complex objects แบบ readable
✅ Nested DSL: builders ซ้อนกันได้
✅ Type-safe: compiler ตรวจสอบ validity
✅ IDE support: autocomplete ทำงานดี
✅ DSL = ลด boilerplate, เพิ่ม readability
```

---

*Part 24/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
