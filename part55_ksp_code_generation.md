# Part 55: KSP - Kotlin Symbol Processing

## สารบัญ
1. [KSP คืออะไร](#ksp-คืออะไร)
2. [Setup KSP Processor](#setup-ksp-processor)
3. [สร้าง Custom Processor](#สร้าง-custom-processor)
4. [Generate Code](#generate-code)
5. [KSP กับ Annotations](#ksp-กับ-annotations)
6. [Testing Processors](#testing-processors)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## KSP คืออะไร

KSP (Kotlin Symbol Processing) คือ API สำหรับ compile-time code generation — เหมือน KAPT แต่เร็วกว่ามาก

```
KSP vs KAPT:
- KSP: Kotlin-first, เข้าใจ Kotlin types โดยตรง, เร็วกว่า 2x
- KAPT: Java annotation processing, แปลง Kotlin เป็น Java stubs ก่อน

Use cases:
- Generate boilerplate code (toDto(), fromDto(), etc.)
- Validate annotations at compile time
- Create type-safe builders
- Generate serialization code
- Route generation (Compose Navigation, Room, etc.)
```

---

## Setup

```kotlin
// processor/build.gradle.kts (KSP processor module)
plugins {
    kotlin("jvm")
}

dependencies {
    implementation("com.google.devtools.ksp:symbol-processing-api:2.0.21-1.0.28")
    implementation("com.squareup:kotlinpoet:1.18.1")
    implementation("com.squareup:kotlinpoet-ksp:1.18.1")
}

// app/build.gradle.kts (consuming module)
plugins {
    kotlin("jvm")
    id("com.google.devtools.ksp") version "2.0.21-1.0.28"
}

dependencies {
    implementation(project(":annotations"))
    ksp(project(":processor"))
}

// annotations/build.gradle.kts (annotation definitions)
plugins { kotlin("jvm") }
// No KSP dependency needed here
```

---

## สร้าง Custom Annotation

```kotlin
// annotations/src/main/kotlin/Annotations.kt

// Annotation สำหรับ generate Builder
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class GenerateBuilder

// Annotation สำหรับ generate DTO mappings
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class GenerateMapper(
    val targetClass: KClass<*>
)

// Annotation สำหรับ validate at compile time
@Target(AnnotationTarget.VALUE_PARAMETER, AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.SOURCE)
annotation class NotEmpty

@Target(AnnotationTarget.VALUE_PARAMETER, AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.SOURCE)
annotation class MinLength(val value: Int)

@Target(AnnotationTarget.VALUE_PARAMETER, AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.SOURCE)
annotation class MaxLength(val value: Int)

// Annotation สำหรับ generate REST client
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class RestClient(val baseUrl: String)

@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.SOURCE)
annotation class Get(val path: String)

@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.SOURCE)
annotation class Post(val path: String)
```

---

## สร้าง Processor

```kotlin
// processor/src/main/kotlin/BuilderProcessor.kt

class BuilderProcessorProvider : SymbolProcessorProvider {
    override fun create(environment: SymbolProcessorEnvironment): SymbolProcessor {
        return BuilderProcessor(environment.codeGenerator, environment.logger)
    }
}

class BuilderProcessor(
    private val codeGenerator: CodeGenerator,
    private val logger: KSPLogger
) : SymbolProcessor {
    
    override fun process(resolver: Resolver): List<KSAnnotated> {
        val symbols = resolver
            .getSymbolsWithAnnotation("com.example.annotations.GenerateBuilder")
            .filterIsInstance<KSClassDeclaration>()
        
        val unableToProcess = symbols.filterNot { it.validate() }.toList()
        
        symbols.filter { it.validate() }.forEach { classDecl ->
            generateBuilder(classDecl)
        }
        
        return unableToProcess
    }
    
    private fun generateBuilder(classDecl: KSClassDeclaration) {
        val packageName = classDecl.packageName.asString()
        val className = classDecl.simpleName.asString()
        val builderName = "${className}Builder"
        
        logger.info("Generating builder for $className")
        
        // Get constructor parameters
        val constructor = classDecl.primaryConstructor
            ?: run {
                logger.error("$className must have a primary constructor", classDecl)
                return
            }
        
        val properties = constructor.parameters.map { param ->
            val name = param.name?.asString() ?: return
            val type = param.type.resolve()
            Triple(name, type, param.hasDefault)
        }
        
        // Generate builder using KotlinPoet
        val fileSpec = FileSpec.builder(packageName, builderName)
            .addType(
                TypeSpec.classBuilder(builderName)
                    .apply {
                        // Add mutable fields
                        properties.forEach { (name, type, hasDefault) ->
                            addProperty(
                                PropertySpec.builder(
                                    name,
                                    type.toTypeName().copy(nullable = !hasDefault)
                                )
                                    .mutable(true)
                                    .initializer(if (hasDefault) "null" else "null")
                                    .build()
                            )
                        }
                        
                        // Add setter methods (fluent API)
                        properties.forEach { (name, type, _) ->
                            addFunction(
                                FunSpec.builder(name)
                                    .addParameter("value", type.toTypeName())
                                    .returns(ClassName(packageName, builderName))
                                    .addStatement("this.$name = value")
                                    .addStatement("return this")
                                    .build()
                            )
                        }
                        
                        // Add build() method
                        addFunction(
                            FunSpec.builder("build")
                                .returns(ClassName(packageName, className))
                                .apply {
                                    properties.forEach { (name, _, _) ->
                                        addStatement(
                                            "val $name = this.$name ?: error(\"$name must be set\")"
                                        )
                                    }
                                    addStatement(
                                        "return %T(${properties.joinToString { (name, _, _) -> name }})",
                                        ClassName(packageName, className)
                                    )
                                }
                                .build()
                        )
                    }
                    .build()
            )
            // Add DSL builder function
            .addFunction(
                FunSpec.builder(className.replaceFirstChar { it.lowercase() })
                    .addParameter(
                        "block",
                        LambdaTypeName.get(
                            receiver = ClassName(packageName, builderName),
                            returnType = UNIT
                        )
                    )
                    .returns(ClassName(packageName, className))
                    .addStatement("return %T().apply(block).build()", ClassName(packageName, builderName))
                    .build()
            )
            .build()
        
        val file = codeGenerator.createNewFile(
            dependencies = Dependencies(false, classDecl.containingFile!!),
            packageName = packageName,
            fileName = builderName
        )
        
        file.writer().use { fileSpec.writeTo(it) }
    }
}

// Register processor in META-INF
// Create: processor/src/main/resources/META-INF/services/com.google.devtools.ksp.processing.SymbolProcessorProvider
// Content: com.example.processor.BuilderProcessorProvider
```

---

## Validator Processor

```kotlin
// processor/src/main/kotlin/ValidatorProcessor.kt

class ValidatorProcessor(
    private val codeGenerator: CodeGenerator,
    private val logger: KSPLogger
) : SymbolProcessor {
    
    override fun process(resolver: Resolver): List<KSAnnotated> {
        // Find all data classes with validation annotations
        val validatedClasses = resolver
            .getAllFiles()
            .flatMap { file -> file.declarations }
            .filterIsInstance<KSClassDeclaration>()
            .filter { classDecl ->
                classDecl.primaryConstructor?.parameters?.any { param ->
                    param.annotations.any { ann ->
                        ann.shortName.asString() in setOf("NotEmpty", "MinLength", "MaxLength")
                    }
                } == true
            }
        
        validatedClasses.forEach { classDecl ->
            generateValidator(classDecl, resolver)
        }
        
        return emptyList()
    }
    
    private fun generateValidator(classDecl: KSClassDeclaration, resolver: Resolver) {
        val packageName = classDecl.packageName.asString()
        val className = classDecl.simpleName.asString()
        val validatorName = "${className}Validator"
        
        val constructor = classDecl.primaryConstructor ?: return
        
        // Collect validation rules
        data class ValidationRule(
            val paramName: String,
            val type: String,
            val rules: List<Pair<String, Any?>>
        )
        
        val rules = constructor.parameters.mapNotNull { param ->
            val paramName = param.name?.asString() ?: return@mapNotNull null
            val paramRules = mutableListOf<Pair<String, Any?>>()
            
            param.annotations.forEach { ann ->
                when (ann.shortName.asString()) {
                    "NotEmpty" -> paramRules.add("notEmpty" to null)
                    "MinLength" -> {
                        val value = ann.arguments.firstOrNull()?.value as? Int ?: return@forEach
                        paramRules.add("minLength" to value)
                    }
                    "MaxLength" -> {
                        val value = ann.arguments.firstOrNull()?.value as? Int ?: return@forEach
                        paramRules.add("maxLength" to value)
                    }
                }
            }
            
            if (paramRules.isNotEmpty()) ValidationRule(paramName, "String", paramRules)
            else null
        }
        
        // Generate validator class
        val sb = buildString {
            appendLine("package $packageName")
            appendLine()
            appendLine("// Generated validator for $className")
            appendLine("object $validatorName {")
            appendLine()
            appendLine("    data class ValidationError(val field: String, val message: String)")
            appendLine()
            appendLine("    fun validate(obj: $className): List<ValidationError> {")
            appendLine("        val errors = mutableListOf<ValidationError>()")
            appendLine()
            
            rules.forEach { rule ->
                rule.rules.forEach { (ruleName, value) ->
                    when (ruleName) {
                        "notEmpty" -> {
                            appendLine("""        if (obj.${rule.paramName}.isBlank()) {""")
                            appendLine("""            errors.add(ValidationError("${rule.paramName}", "${rule.paramName} must not be empty"))""")
                            appendLine("""        }""")
                        }
                        "minLength" -> {
                            appendLine("""        if (obj.${rule.paramName}.length < $value) {""")
                            appendLine("""            errors.add(ValidationError("${rule.paramName}", "${rule.paramName} must be at least $value characters"))""")
                            appendLine("""        }""")
                        }
                        "maxLength" -> {
                            appendLine("""        if (obj.${rule.paramName}.length > $value) {""")
                            appendLine("""            errors.add(ValidationError("${rule.paramName}", "${rule.paramName} must not exceed $value characters"))""")
                            appendLine("""        }""")
                        }
                    }
                }
            }
            
            appendLine()
            appendLine("        return errors")
            appendLine("    }")
            appendLine()
            appendLine("    fun isValid(obj: $className) = validate(obj).isEmpty()")
            appendLine("}")
        }
        
        val file = codeGenerator.createNewFile(
            dependencies = Dependencies(false, classDecl.containingFile!!),
            packageName = packageName,
            fileName = validatorName
        )
        
        file.writer().use { it.write(sb) }
    }
}
```

---

## ตัวอย่างการใช้งาน

```kotlin
// app/src/main/kotlin/models/UserRequest.kt

@GenerateBuilder
data class UserRequest(
    val firstName: String,
    val lastName: String,
    val email: String,
    val age: Int = 0,
    val phone: String? = null
)

// Generated code (UserRequestBuilder.kt):
// class UserRequestBuilder {
//     var firstName: String? = null
//     var lastName: String? = null
//     var email: String? = null
//     var age: Int? = null
//     var phone: String? = null
//
//     fun firstName(value: String) = apply { firstName = value }
//     fun lastName(value: String) = apply { lastName = value }
//     fun email(value: String) = apply { email = value }
//     fun age(value: Int) = apply { age = value }
//     fun phone(value: String?) = apply { phone = value }
//
//     fun build(): UserRequest {
//         val firstName = firstName ?: error("firstName must be set")
//         val lastName = lastName ?: error("lastName must be set")
//         val email = email ?: error("email must be set")
//         val age = age ?: error("age must be set")
//         return UserRequest(firstName, lastName, email, age, phone)
//     }
// }
//
// fun userRequest(block: UserRequestBuilder.() -> Unit) = UserRequestBuilder().apply(block).build()

// ใช้งาน:
fun createUser() {
    // Fluent builder
    val user1 = UserRequestBuilder()
        .firstName("สมชาย")
        .lastName("ใจดี")
        .email("somchai@example.com")
        .age(25)
        .build()
    
    // DSL builder
    val user2 = userRequest {
        firstName = "สมหญิง"
        lastName = "ใจงาม"
        email = "somying@example.com"
        age = 22
    }
}

// Validation:
data class RegisterRequest(
    @NotEmpty val username: String,
    @MinLength(8) @MaxLength(100) val password: String,
    @NotEmpty val email: String
)

// Generated RegisterRequestValidator:
// val errors = RegisterRequestValidator.validate(RegisterRequest("", "123", ""))
// errors.forEach { println("${it.field}: ${it.message}") }
```

---

## Testing

```kotlin
// เทส processor ด้วย KSP testing library
class BuilderProcessorTest {
    
    @Test
    fun `should generate builder for data class`() {
        val source = SourceFile.kotlin("UserRequest.kt", """
            package com.example
            
            import com.example.annotations.GenerateBuilder
            
            @GenerateBuilder
            data class UserRequest(
                val name: String,
                val email: String,
                val age: Int = 18
            )
        """)
        
        val compilation = KotlinCompilation().apply {
            sources = listOf(source)
            symbolProcessorProviders = listOf(BuilderProcessorProvider())
            inheritClassPath = true
        }
        
        val result = compilation.compile()
        
        assertEquals(KotlinCompilation.ExitCode.OK, result.exitCode)
        
        // Check generated file exists
        val generated = result.generatedFiles.find { it.name == "UserRequestBuilder.kt" }
        assertNotNull(generated)
        
        // Check content
        val content = generated!!.readText()
        assertTrue(content.contains("class UserRequestBuilder"))
        assertTrue(content.contains("fun build(): UserRequest"))
        assertTrue(content.contains("fun userRequest(block: UserRequestBuilder.() -> Unit)"))
    }
    
    @Test
    fun `should generate validator with rules`() {
        val source = SourceFile.kotlin("RegisterRequest.kt", """
            package com.example
            
            import com.example.annotations.NotEmpty
            import com.example.annotations.MinLength
            
            data class RegisterRequest(
                @NotEmpty val username: String,
                @MinLength(8) val password: String
            )
        """)
        
        val compilation = KotlinCompilation().apply {
            sources = listOf(source)
            symbolProcessorProviders = listOf(ValidatorProcessorProvider())
            inheritClassPath = true
        }
        
        val result = compilation.compile()
        assertEquals(KotlinCompilation.ExitCode.OK, result.exitCode)
        
        val validator = result.generatedFiles.find { it.name == "RegisterRequestValidator.kt" }
        assertNotNull(validator)
        
        val content = validator!!.readText()
        assertTrue(content.contains("isBlank()"))
        assertTrue(content.contains("length < 8"))
    }
}

// Placeholder types for compilation
class ValidatorProcessorProvider : SymbolProcessorProvider {
    override fun create(environment: SymbolProcessorEnvironment) =
        ValidatorProcessor(environment.codeGenerator, environment.logger)
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง @Table annotation processor

// @Table("users") → generate SQL CREATE TABLE statement

@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.SOURCE)
annotation class Table(val name: String)

@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.SOURCE)
annotation class Column(val name: String = "", val type: String = "", val primaryKey: Boolean = false)

@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.SOURCE)
annotation class Index(val name: String)

// Given:
@Table("users")
data class UserEntity3(
    @Column(primaryKey = true) val id: String,
    @Column("username") val username: String,
    @Column("email") @Index("idx_users_email") val email: String,
    @Column("created_at") val createdAt: Long
)

// Expected generated code:
// object UserEntitySql {
//     const val CREATE_TABLE = """
//         CREATE TABLE IF NOT EXISTS users (
//             id TEXT NOT NULL,
//             username TEXT NOT NULL,
//             email TEXT NOT NULL,
//             created_at INTEGER NOT NULL,
//             PRIMARY KEY (id)
//         );
//         CREATE INDEX IF NOT EXISTS idx_users_email ON users (email);
//     """
// }

// TODO: Implement TableProcessorProvider + TableProcessor
```

---

## สรุป Part 55

```
✅ KSP: Kotlin Symbol Processing API
✅ เร็วกว่า KAPT 2x เพราะไม่ต้อง generate Java stubs
✅ SymbolProcessorProvider: factory สำหรับ processor
✅ SymbolProcessor.process(): entry point
✅ KSClassDeclaration: access class metadata
✅ KSFunctionDeclaration: access function metadata
✅ KSAnnotated: elements with annotations
✅ KotlinPoet: type-safe Kotlin code generation
✅ TypeSpec, FunSpec, PropertySpec: code builders
✅ FileSpec: generate .kt files
✅ CodeGenerator: write to build output
✅ KSPLogger: compile-time logging/errors
✅ Testing: kotlinCompilation + SourceFile
✅ Use cases: Builder, Validator, Mapper, Router
```

---

*Part 55/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
