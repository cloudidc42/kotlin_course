# Part 56: Gradle Kotlin DSL และ Convention Plugins

## สารบัญ
1. [Kotlin DSL Basics](#kotlin-dsl-basics)
2. [Convention Plugins](#convention-plugins)
3. [Version Catalogs](#version-catalogs)
4. [Custom Tasks](#custom-tasks)
5. [Build Optimization](#build-optimization)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Kotlin DSL Basics

Gradle Kotlin DSL ใช้ `.kts` แทน Groovy `.gradle` — type-safe, IDE autocomplete

```kotlin
// settings.gradle.kts
rootProject.name = "my-multimodule-project"

// Plugin Management
pluginManagement {
    repositories {
        gradlePluginPortal()
        google()
        mavenCentral()
    }
}

// Dependency Resolution Management
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
    }
}

// Include modules
include(":app")
include(":core:domain")
include(":core:data")
include(":core:network")
include(":feature:products")
include(":feature:orders")
include(":feature:profile")
```

```kotlin
// build.gradle.kts (root)
plugins {
    alias(libs.plugins.kotlin.jvm) apply false
    alias(libs.plugins.kotlin.android) apply false
    alias(libs.plugins.android.application) apply false
    alias(libs.plugins.android.library) apply false
    alias(libs.plugins.hilt) apply false
    alias(libs.plugins.ksp) apply false
}

// รัน task บน subprojects
tasks.register("clean", Delete::class) {
    delete(rootProject.layout.buildDirectory)
}
```

---

## Version Catalogs

```toml
# gradle/libs.versions.toml

[versions]
kotlin = "2.0.21"
android-gradle-plugin = "8.7.2"
compose-bom = "2024.11.00"
coroutines = "1.8.1"
ktor = "3.0.1"
room = "2.6.1"
hilt = "2.52"
ksp = "2.0.21-1.0.28"
retrofit = "2.11.0"
okhttp = "4.12.0"
coil = "2.7.0"
junit = "5.10.3"
mockk = "1.13.12"
kotest = "5.9.1"
testcontainers = "1.20.3"

[libraries]
# Kotlin
kotlin-stdlib = { group = "org.jetbrains.kotlin", name = "kotlin-stdlib", version.ref = "kotlin" }
kotlin-reflect = { group = "org.jetbrains.kotlin", name = "kotlin-reflect", version.ref = "kotlin" }
kotlinx-coroutines-core = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-core", version.ref = "coroutines" }
kotlinx-coroutines-android = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-android", version.ref = "coroutines" }
kotlinx-coroutines-test = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-test", version.ref = "coroutines" }

# Compose BOM
compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
compose-ui = { group = "androidx.compose.ui", name = "ui" }
compose-material3 = { group = "androidx.compose.material3", name = "material3" }
compose-preview = { group = "androidx.compose.ui", name = "ui-tooling-preview" }
compose-test = { group = "androidx.compose.ui", name = "ui-test-junit4" }

# Room
room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }

# Hilt
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-compiler", version.ref = "hilt" }

# Ktor
ktor-client-core = { group = "io.ktor", name = "ktor-client-core", version.ref = "ktor" }
ktor-client-android = { group = "io.ktor", name = "ktor-client-android", version.ref = "ktor" }
ktor-client-logging = { group = "io.ktor", name = "ktor-client-logging", version.ref = "ktor" }
ktor-serialization-json = { group = "io.ktor", name = "ktor-serialization-kotlinx-json", version.ref = "ktor" }

# Testing
junit-bom = { group = "org.junit", name = "junit-bom", version.ref = "junit" }
junit-jupiter = { group = "org.junit.jupiter", name = "junit-jupiter" }
mockk = { group = "io.mockk", name = "mockk", version.ref = "mockk" }
kotest-runner-junit5 = { group = "io.kotest", name = "kotest-runner-junit5", version.ref = "kotest" }
kotest-assertions = { group = "io.kotest", name = "kotest-assertions-core", version.ref = "kotest" }
testcontainers-bom = { group = "org.testcontainers", name = "testcontainers-bom", version.ref = "testcontainers" }
testcontainers-postgresql = { group = "org.testcontainers", name = "postgresql" }
testcontainers-kafka = { group = "org.testcontainers", name = "kafka" }

[plugins]
kotlin-jvm = { id = "org.jetbrains.kotlin.jvm", version.ref = "kotlin" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-multiplatform = { id = "org.jetbrains.kotlin.multiplatform", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
android-application = { id = "com.android.application", version.ref = "android-gradle-plugin" }
android-library = { id = "com.android.library", version.ref = "android-gradle-plugin" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }

[bundles]
# กลุ่ม dependencies ที่ใช้ร่วมกัน
ktor-client = ["ktor-client-core", "ktor-client-logging", "ktor-serialization-json"]
room = ["room-runtime", "room-ktx"]
testing = ["junit-jupiter", "mockk", "kotest-assertions", "kotlinx-coroutines-test"]
```

```kotlin
// ใช้งาน version catalog:
dependencies {
    implementation(libs.kotlinx.coroutines.core)
    implementation(libs.bundles.ktor.client)
    
    implementation(libs.room.runtime)
    implementation(libs.room.ktx)
    ksp(libs.room.compiler)
    
    implementation(platform(libs.compose.bom))
    implementation(libs.compose.ui)
    implementation(libs.compose.material3)
    
    testImplementation(libs.bundles.testing)
    testImplementation(platform(libs.testcontainers.bom))
    testImplementation(libs.testcontainers.postgresql)
}
```

---

## Convention Plugins

Convention plugins แทน boilerplate ใน build scripts — define once, reuse across modules

```kotlin
// build-logic/build.gradle.kts
plugins {
    `kotlin-dsl`
}

dependencies {
    compileOnly(libs.kotlin.jvm)
    compileOnly(libs.android.gradle.plugin)
}
```

```kotlin
// build-logic/src/main/kotlin/KotlinJvmConventionPlugin.kt

class KotlinJvmConventionPlugin : Plugin<Project> {
    
    override fun apply(target: Project) {
        with(target) {
            with(pluginManager) {
                apply("org.jetbrains.kotlin.jvm")
                apply("io.gitlab.arturbosch.detekt")
                apply("org.jlleitschuh.gradle.ktlint")
            }
            
            extensions.configure<JavaPluginExtension> {
                sourceCompatibility = JavaVersion.VERSION_17
                targetCompatibility = JavaVersion.VERSION_17
            }
            
            tasks.withType<KotlinCompile>().configureEach {
                kotlinOptions {
                    jvmTarget = "17"
                    freeCompilerArgs = freeCompilerArgs + listOf(
                        "-opt-in=kotlin.RequiresOptIn",
                        "-opt-in=kotlinx.coroutines.ExperimentalCoroutinesApi",
                        "-Xcontext-receivers"
                    )
                }
            }
            
            // Standard test config
            tasks.withType<Test>().configureEach {
                useJUnitPlatform()
                testLogging {
                    events("passed", "skipped", "failed")
                    showExceptions = true
                    showCauses = true
                }
                maxParallelForks = (Runtime.getRuntime().availableProcessors() / 2).coerceAtLeast(1)
            }
            
            dependencies {
                "testImplementation"(kotlin("test"))
                "testImplementation"("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")
            }
        }
    }
}

// build-logic/src/main/kotlin/SpringBootConventionPlugin.kt
class SpringBootConventionPlugin : Plugin<Project> {
    
    override fun apply(target: Project) {
        with(target) {
            pluginManager.apply("org.springframework.boot")
            pluginManager.apply("io.spring.dependency-management")
            pluginManager.apply("com.example.kotlin-jvm")  // แล้วก็ apply kotlin-jvm convention
            
            extensions.configure<SpringBootExtension> {
                // Common Spring Boot config
            }
            
            dependencies {
                "implementation"("org.springframework.boot:spring-boot-starter")
                "implementation"("org.springframework.boot:spring-boot-starter-actuator")
                "testImplementation"("org.springframework.boot:spring-boot-starter-test")
            }
        }
    }
}

// settings.gradle.kts
includeBuild("build-logic")

// app/build.gradle.kts (clean!)
plugins {
    id("com.example.kotlin-jvm")
    id("com.example.spring-boot")
}

dependencies {
    implementation(libs.kotlinx.coroutines.core)
    // Only app-specific dependencies here
}
```

---

## Custom Tasks

```kotlin
// build.gradle.kts - custom tasks

// Task สำหรับ generate version info
tasks.register("generateVersionInfo") {
    val outputDir = layout.buildDirectory.dir("generated/version")
    outputs.dir(outputDir)
    
    doLast {
        val gitHash = exec {
            commandLine("git", "rev-parse", "--short", "HEAD")
        }.let { "unknown" }  // เพื่อความปลอดภัย
        
        val version = project.version.toString()
        val buildTime = java.time.Instant.now().toString()
        
        outputDir.get().asFile.mkdirs()
        outputDir.get().file("version.properties").asFile.writeText("""
            version=$version
            gitHash=$gitHash
            buildTime=$buildTime
        """.trimIndent())
    }
}

// Task incremental processing
abstract class GenerateSqlTask : DefaultTask() {
    
    @get:InputDirectory
    abstract val sourceDir: DirectoryProperty
    
    @get:OutputDirectory
    abstract val outputDir: DirectoryProperty
    
    @TaskAction
    fun execute() {
        val source = sourceDir.get().asFile
        val output = outputDir.get().asFile
        output.mkdirs()
        
        source.walkTopDown()
            .filter { it.extension == "kt" }
            .forEach { kotlinFile ->
                // Parse annotations and generate SQL
                val content = kotlinFile.readText()
                if (content.contains("@Table")) {
                    val sqlFile = output.resolve("${kotlinFile.nameWithoutExtension}.sql")
                    sqlFile.writeText(generateSqlFrom(content))
                }
            }
    }
    
    private fun generateSqlFrom(content: String): String {
        // Simplified extraction
        return "-- Generated SQL\n"
    }
}

tasks.register<GenerateSqlTask>("generateSql") {
    sourceDir.set(layout.projectDirectory.dir("src/main/kotlin"))
    outputDir.set(layout.buildDirectory.dir("generated/sql"))
}

// Lifecycle hooks
tasks.named("compileKotlin") {
    dependsOn("generateVersionInfo", "generateSql")
}

// Task สำหรับ Docker build
tasks.register("dockerBuild") {
    dependsOn("build")
    
    doLast {
        val imageName = "${project.name}:${project.version}"
        exec {
            commandLine("docker", "build", "-t", imageName, ".")
        }
        println("Built Docker image: $imageName")
    }
}

// Task สำหรับรัน integration tests แยก
tasks.register<Test>("integrationTest") {
    description = "Run integration tests"
    group = "verification"
    
    testClassesDirs = sourceSets["integrationTest"].output.classesDirs
    classpath = sourceSets["integrationTest"].runtimeClasspath
    
    useJUnitPlatform {
        includeTags("integration")
    }
    
    shouldRunAfter("test")
}

// sourceSets configuration
sourceSets {
    create("integrationTest") {
        kotlin.srcDir("src/integrationTest/kotlin")
        resources.srcDir("src/integrationTest/resources")
        compileClasspath += sourceSets["main"].output + configurations["testRuntimeClasspath"]
        runtimeClasspath += output + compileClasspath
    }
}
```

---

## Build Optimization

```kotlin
// gradle.properties
org.gradle.jvmargs=-Xmx4g -XX:+UseParallelGC
org.gradle.parallel=true
org.gradle.caching=true
org.gradle.configuration-cache=true
kotlin.incremental=true
kotlin.caching.enabled=true

# Android
android.useAndroidX=true
android.enableJetifier=false
```

```kotlin
// build.gradle.kts - optimization techniques

// Avoid resolving dependencies during configuration
configurations.all {
    resolutionStrategy.cacheChangingModulesFor(0, TimeUnit.SECONDS)
    resolutionStrategy.cacheDynamicVersionsFor(1, TimeUnit.DAYS)
}

// Lazy task configuration
tasks.named("test") {
    // Lazy: only configured when actually needed
    doLast { println("Tests done") }
}

// Configuration cache friendly
val myExtension = extensions.create<MyExtension>("myExtension")

abstract class MyExtension {
    abstract val setting: Property<String>
}

// Build cache
tasks.withType<KotlinCompile>().configureEach {
    // Inputs/outputs properly declared for caching
    inputs.property("kotlinVersion", kotlin.coreLibrariesVersion)
}

// Composite builds for faster iteration
// settings.gradle.kts:
// includeBuild("../shared-library") {
//     dependencySubstitution {
//         substitute(module("com.example:shared-library")).using(project(":"))
//     }
// }
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง convention plugin สำหรับ microservice module

// Requirements:
// 1. id("com.example.microservice") 
// 2. Apply: kotlin-jvm, spring-boot, detekt
// 3. Config: JVM 17, coroutines opt-in
// 4. Dependencies: spring-boot-starter-web, actuator, kafka, test
// 5. Task: dockerBuild ที่สร้าง image โดยอัตโนมัติ
// 6. Task: healthCheck ที่ ping /actuator/health

abstract class MicroserviceExtension {
    abstract val serviceName: Property<String>
    abstract val port: Property<Int>
    // TODO: Add more config
}

class MicroserviceConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) {
        val extension = target.extensions.create<MicroserviceExtension>("microservice")
        
        with(target) {
            // TODO: Apply plugins
            // TODO: Configure dependencies
            // TODO: Register dockerBuild task
            // TODO: Register healthCheck task
        }
    }
}

// Usage:
// plugins { id("com.example.microservice") }
// microservice {
//     serviceName.set("order-service")
//     port.set(8081)
// }
```

---

## สรุป Part 56

```
✅ Kotlin DSL: type-safe, IDE autocomplete สำหรับ Gradle
✅ settings.gradle.kts: root configuration, module inclusion
✅ build.gradle.kts: plugins, dependencies, tasks
✅ Version Catalogs: libs.versions.toml, bundles, BOMs
✅ Convention Plugins: reusable build configuration
✅ KotlinJvmConventionPlugin: consistent Kotlin setup
✅ SpringBootConventionPlugin: shared Spring config
✅ includeBuild: composite builds for build-logic
✅ Custom Tasks: @TaskAction, @InputDirectory, @OutputDirectory
✅ Incremental tasks: only rebuild when inputs change
✅ gradle.properties: JVM args, parallel, caching
✅ Configuration cache: reuse task graphs
✅ Build cache: reuse task outputs
✅ Source sets: integrationTest separation
✅ Composite builds: local library iteration
```

---

*Part 56/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
