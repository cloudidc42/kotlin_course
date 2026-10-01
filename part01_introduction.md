# Part 01: บทนำและการติดตั้ง Kotlin

## สารบัญ
1. [Kotlin คืออะไร?](#kotlin-คืออะไร)
2. [ประวัติและความเป็นมา](#ประวัติและความเป็นมา)
3. [ทำไมต้องเรียน Kotlin?](#ทำไมต้องเรียน-kotlin)
4. [Kotlin ทำงานอย่างไร?](#kotlin-ทำงานอย่างไร)
5. [การติดตั้ง JDK](#การติดตั้ง-jdk)
6. [การติดตั้ง IntelliJ IDEA](#การติดตั้ง-intellij-idea)
7. [สร้างโปรเจกต์แรก](#สร้างโปรเจกต์แรก)
8. [โปรแกรม Hello World](#โปรแกรม-hello-world)
9. [โครงสร้างโปรแกรม Kotlin](#โครงสร้างโปรแกรม-kotlin)
10. [การ Compile และ Run โปรแกรม](#การ-compile-และ-run-โปรแกรม)
11. [Kotlin Playground](#kotlin-playground)
12. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Kotlin คืออะไร?

**Kotlin** (อ่านว่า "คอตลิน") คือภาษาโปรแกรมสมัยใหม่แบบ **Statically Typed** ที่พัฒนาโดยบริษัท **JetBrains** ซึ่งเป็นบริษัทที่อยู่เบื้องหลัง IntelliJ IDEA, PyCharm และ IDE อื่นๆ อีกมากมาย

Kotlin ออกแบบมาให้:
- ทำงานบน **JVM (Java Virtual Machine)** ได้
- ทำงานร่วมกับ **Java** ได้อย่างสมบูรณ์ (100% Interoperable)
- คอมไพล์เป็น **JavaScript** ได้
- คอมไพล์เป็น **Native code** ได้ (ผ่าน Kotlin/Native)

### คุณสมบัติหลักของ Kotlin

```
┌─────────────────────────────────────────────────────────────────┐
│                     คุณสมบัติเด่นของ Kotlin                      │
├─────────────────┬───────────────────────────────────────────────┤
│ Concise         │ โค้ดกระชับ ลดบรรทัดโค้ดได้ถึง 40% เทียบกับ Java │
│ Safe            │ ป้องกัน NullPointerException ตั้งแต่ Compile Time│
│ Interoperable   │ ใช้งาน Java Libraries ได้ทั้งหมด               │
│ Tool-friendly   │ รองรับทุก IDE โดย JetBrains                     │
│ Multiplatform   │ เขียนครั้งเดียว ใช้ได้ทุกแพลตฟอร์ม              │
└─────────────────┴───────────────────────────────────────────────┘
```

---

## ประวัติและความเป็นมา

### Timeline ของ Kotlin

```
2010 ── JetBrains เริ่มพัฒนา Kotlin ภายในบริษัท
         เป้าหมาย: ภาษาที่ดีกว่า Java สำหรับพัฒนา IDE เอง

2011 ── JetBrains ประกาศ Kotlin ต่อสาธารณะ
         นำเสนอใน JVM Language Summit

2012 ── Kotlin เป็น Open Source ภายใต้ Apache 2 License
         เริ่มมีชุมชนนักพัฒนาเติบโต

2016 ── Kotlin 1.0 Release อย่างเป็นทางการ (Feb 15, 2016)
         เสถียรและพร้อมใช้งาน Production

2017 ── Google I/O 2017: Google ประกาศ Kotlin เป็น
         "First-class Language" สำหรับ Android Development
         จุดเปลี่ยนครั้งใหญ่ที่ทำให้ Kotlin ดังระเบิด!

2019 ── Google I/O 2019: Google ประกาศ "Kotlin-first"
         แอป Android ใหม่ๆ ควรเขียนด้วย Kotlin เป็นหลัก

2022 ── Kotlin 1.7.0 พร้อม K2 Compiler (Alpha)
         ปรับปรุงประสิทธิภาพการ Compile อย่างมาก

2023 ── Kotlin 1.9.0 และ Kotlin 2.0 Beta
         K2 Compiler ปรับปรุงให้เร็วขึ้น 2x

2024 ── Kotlin 2.0 Stable Release
         ยุคใหม่ของ Kotlin พร้อม Compose Multiplatform
```

### ชื่อ "Kotlin" มาจากไหน?

ชื่อ Kotlin มาจาก **เกาะ Kotlin** (Kotlin Island / Котлин) ในประเทศรัสเซีย ใกล้กับเมือง Saint Petersburg ซึ่งเป็นที่ตั้งสำนักงานใหญ่ของ JetBrains นั่นเอง! (คล้ายกับที่ Java ตั้งชื่อตามเกาะ Java ในอินโดนีเซีย)

---

## ทำไมต้องเรียน Kotlin?

### 1. ตลาดงานมีความต้องการสูง

```
📊 ข้อมูลตลาดงาน Kotlin (2024):
┌─────────────────────────────────────────────────────┐
│ • Android Developer ส่วนใหญ่ต้องการ Kotlin 90%+     │
│ • Backend Developer ที่ใช้ Spring Boot + Kotlin      │
│   เพิ่มขึ้น 300% ใน 3 ปีที่ผ่านมา                  │
│ • เงินเดือนเฉลี่ย Kotlin Developer:                 │
│   - ไทย: 50,000 - 150,000 บาท/เดือน                 │
│   - Global: $80,000 - $150,000/ปี                   │
└─────────────────────────────────────────────────────┘
```

### 2. เรียนรู้ได้ไวกว่า Java

Kotlin มีไวยากรณ์ที่กระชับและสื่อความหมายชัดเจน ผู้ที่เรียน Java มาก่อนจะเรียน Kotlin ได้ภายใน 1-2 สัปดาห์ ส่วนผู้เริ่มต้นก็สามารถเริ่มได้เลยโดยไม่ต้องเรียน Java ก่อน

### 3. ความปลอดภัย (Safety)

หนึ่งในจุดเด่นที่สุดของ Kotlin คือ **Null Safety System** ที่ช่วยป้องกันข้อผิดพลาดที่พบบ่อยที่สุดใน Java:

```kotlin
// Java - อาจเกิด NullPointerException!
String name = null;
System.out.println(name.length()); // 💥 CRASH!

// Kotlin - Error ตั้งแต่ Compile Time
var name: String = null  // ❌ Compile Error: Null can not be a value of a non-null type String
var name: String? = null // ✅ ถูกต้อง - บอกว่า name อาจเป็น null ได้
println(name?.length)    // ✅ Safe call - ถ้า name เป็น null จะ return null แทนการ crash
```

### 4. Multiplatform

```
Kotlin Code
    │
    ├──► Android (JVM/Kotlin)
    ├──► iOS (Kotlin Native)
    ├──► Web (Kotlin/JS + React)
    ├──► Backend (Ktor, Spring Boot)
    ├──► Desktop (Compose Desktop)
    └──► Embedded Systems (Kotlin Native)
```

### 5. Community และ Ecosystem ใหญ่

- มี Library และ Framework ให้ใช้งานมากมาย
- Documentation ครบถ้วนและอัปเดตสม่ำเสมอ
- ชุมชนนักพัฒนาที่ active และ helpful
- JetBrains สนับสนุนอย่างต่อเนื่อง

---

## Kotlin ทำงานอย่างไร?

### กระบวนการ Compile และ Run

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kotlin Compilation Process                    │
└─────────────────────────────────────────────────────────────────┘

1. Kotlin/JVM (แบบที่ใช้บ่อยที่สุด):
   
   Source Code (.kt)
        │
        ▼
   Kotlin Compiler (kotlinc)
        │
        ▼
   Bytecode (.class files)
        │
        ▼
   JVM (Java Virtual Machine)
        │
        ▼
   Program Execution

2. Kotlin/JS:
   
   Source Code (.kt)
        │
        ▼
   Kotlin Compiler
        │
        ▼
   JavaScript Code (.js)
        │
        ▼
   Browser / Node.js

3. Kotlin/Native:
   
   Source Code (.kt)
        │
        ▼
   Kotlin Compiler (LLVM Backend)
        │
        ▼
   Native Binary (iOS, macOS, Linux, Windows)
        │
        ▼
   Direct Execution (No JVM needed!)
```

### JVM คืออะไร?

**JVM (Java Virtual Machine)** คือเครื่องเสมือน (Virtual Machine) ที่ทำหน้าที่รัน Bytecode โดย:
- แปล Bytecode เป็นคำสั่งของ CPU จริงๆ
- จัดการหน่วยความจำ (Garbage Collection)
- ทำให้โปรแกรมรันได้บนทุก OS ที่ติดตั้ง JVM

---

## การติดตั้ง JDK

### JDK คืออะไร?

**JDK (Java Development Kit)** คือชุดเครื่องมือสำหรับพัฒนาโปรแกรม Java/Kotlin ประกอบด้วย:
- **JRE (Java Runtime Environment)**: สำหรับรันโปรแกรม
- **JVM (Java Virtual Machine)**: เครื่องเสมือนสำหรับรัน Bytecode
- **Development Tools**: Compiler, Debugger, etc.

### เวอร์ชันที่แนะนำ

ใช้ **JDK 17** (LTS - Long Term Support) หรือใหม่กว่า

### การติดตั้งบน Windows

**วิธีที่ 1: ใช้ Winget (แนะนำ)**
```powershell
# เปิด PowerShell as Administrator แล้วรัน:
winget install Microsoft.OpenJDK.17

# หรือติดตั้ง Adoptium JDK
winget install EclipseAdoptium.Temurin.17.JDK
```

**วิธีที่ 2: ดาวน์โหลดด้วยตัวเอง**
1. ไปที่ https://adoptium.net/
2. เลือก **Temurin 17 (LTS)**
3. เลือก Windows x64
4. ดาวน์โหลดไฟล์ `.msi` และติดตั้ง

**ตรวจสอบการติดตั้ง:**
```cmd
java -version
javac -version
```

ผลลัพธ์ที่ควรเห็น:
```
openjdk version "17.0.9" 2023-10-17
OpenJDK Runtime Environment Temurin-17.0.9+9 (build 17.0.9+9)
OpenJDK 64-Bit Server VM Temurin-17.0.9+9 (build 17.0.9+9, mixed mode, sharing)
```

### การติดตั้งบน macOS

**วิธีที่ 1: ใช้ Homebrew (แนะนำ)**
```bash
# ติดตั้ง Homebrew ก่อน (ถ้ายังไม่มี)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# ติดตั้ง JDK
brew install --cask temurin@17

# ตรวจสอบ
java -version
```

**วิธีที่ 2: ใช้ SDKMAN (จัดการหลาย JDK version ได้)**
```bash
# ติดตั้ง SDKMAN
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# ติดตั้ง JDK 17
sdk install java 17.0.9-tem

# ดูรายการ JDK ที่ติดตั้ง
sdk list java

# สลับ JDK version
sdk use java 17.0.9-tem
```

### การติดตั้งบน Linux (Ubuntu/Debian)

```bash
# อัปเดต package list
sudo apt update

# ติดตั้ง JDK 17
sudo apt install openjdk-17-jdk

# ตรวจสอบ
java -version
javac -version

# ถ้ามีหลาย JDK และต้องการเลือก
sudo update-alternatives --config java
```

### ตั้งค่า JAVA_HOME (ถ้าจำเป็น)

**Windows:**
```powershell
# เพิ่ม Environment Variable
$env:JAVA_HOME = "C:\Program Files\Eclipse Adoptium\jdk-17.0.9.9-hotspot"
[Environment]::SetEnvironmentVariable("JAVA_HOME", $env:JAVA_HOME, "Machine")
```

**macOS/Linux (เพิ่มใน ~/.bashrc หรือ ~/.zshrc):**
```bash
export JAVA_HOME=$(/usr/libexec/java_home -v 17)
export PATH=$JAVA_HOME/bin:$PATH
```

---

## การติดตั้ง IntelliJ IDEA

### เกี่ยวกับ IntelliJ IDEA

**IntelliJ IDEA** คือ IDE (Integrated Development Environment) ที่ดีที่สุดสำหรับ Kotlin มี 2 เวอร์ชัน:

| | Community Edition | Ultimate Edition |
|---|---|---|
| ราคา | **ฟรี** | $249/ปี (มี Trial 30 วัน) |
| Kotlin | ✅ | ✅ |
| Java | ✅ | ✅ |
| Web Development | ❌ | ✅ |
| Database Tools | ❌ | ✅ |
| Spring Boot | จำกัด | ✅ เต็ม |

สำหรับผู้เริ่มต้น **Community Edition** เพียงพอแล้ว!

### ดาวน์โหลดและติดตั้ง

1. ไปที่ https://www.jetbrains.com/idea/download/
2. เลือก **Community Edition** (ฟรี)
3. เลือก OS ของคุณ
4. ดาวน์โหลดและติดตั้ง

**บน macOS ด้วย Homebrew:**
```bash
brew install --cask intellij-idea-ce
```

**บน Linux ด้วย Snap:**
```bash
sudo snap install intellij-idea-community --classic
```

### การตั้งค่าเบื้องต้น

หลังติดตั้ง IntelliJ IDEA:

1. **เลือก UI Theme**: Light หรือ Dark ตามชอบ
2. **เลือก Keymap**: IntelliJ IDEA (default) หรือ VS Code
3. **ติดตั้ง Plugins เพิ่มเติม** (ถ้าต้องการ):
   - Kotlin (ติดตั้งมาแล้ว)
   - Rainbow Brackets (ช่วยดูโค้ดชัดขึ้น)
   - Material Theme UI (ธีมสวยงาม)

---

## สร้างโปรเจกต์แรก

### ขั้นตอนสร้างโปรเจกต์

1. เปิด IntelliJ IDEA
2. คลิก **New Project**
3. ตั้งค่าดังนี้:

```
Project Type  : Kotlin
Build System  : Gradle (แนะนำ) หรือ IntelliJ
JDK           : เลือก JDK 17
Kotlin DSL    : ✅ (เลือกใช้ Kotlin DSL)
Project Name  : HelloKotlin
Project Location: เลือกที่ต้องการเก็บไฟล์
```

4. คลิก **Create**

### โครงสร้างโปรเจกต์

หลังสร้างโปรเจกต์จะได้โครงสร้างดังนี้:

```
HelloKotlin/
├── .gradle/              ← Gradle cache (ไม่ต้องสนใจ)
├── .idea/                ← IntelliJ settings (ไม่ต้องสนใจ)
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── src/
│   ├── main/
│   │   └── kotlin/
│   │       └── Main.kt   ← ✅ ไฟล์โปรแกรมหลักของเรา
│   └── test/
│       └── kotlin/       ← สำหรับ Unit Tests
├── build.gradle.kts      ← ✅ ไฟล์ Build Configuration
├── gradle.properties
├── gradlew               ← Gradle Wrapper (Linux/Mac)
├── gradlew.bat           ← Gradle Wrapper (Windows)
└── settings.gradle.kts   ← ✅ ชื่อโปรเจกต์
```

### ไฟล์ build.gradle.kts

```kotlin
plugins {
    kotlin("jvm") version "1.9.21"
    application
}

group = "com.example"
version = "1.0-SNAPSHOT"

repositories {
    mavenCentral()
}

dependencies {
    testImplementation(kotlin("test"))
}

tasks.test {
    useJUnitPlatform()
}

kotlin {
    jvmToolchain(17)
}

application {
    mainClass.set("MainKt")
}
```

**คำอธิบาย:**
- `plugins`: กำหนด Plugin ที่ใช้ (Kotlin JVM และ Application)
- `group`: ชื่อกลุ่ม/องค์กร (ใช้ reverse domain เช่น `com.example`)
- `version`: เวอร์ชันของโปรแกรม
- `repositories`: แหล่งดาวน์โหลด Library (Maven Central)
- `dependencies`: Library ที่ต้องใช้
- `application.mainClass`: ระบุ Class หลักของโปรแกรม

---

## โปรแกรม Hello World

สร้างไฟล์ `src/main/kotlin/Main.kt`:

### เวอร์ชันพื้นฐาน

```kotlin
fun main() {
    println("Hello, World!")
}
```

**Output:**
```
Hello, World!
```

### เวอร์ชันพร้อม Comment อธิบาย

```kotlin
// นี่คือ Comment แบบบรรทัดเดียว

/*
   นี่คือ Comment
   แบบหลายบรรทัด
*/

/**
 * นี่คือ KDoc Comment สำหรับ Documentation
 * @author ชื่อผู้เขียน
 */
fun main() {
    // println ย่อมาจาก "print line"
    // ใช้แสดงข้อความพร้อมขึ้นบรรทัดใหม่
    println("Hello, World!")
    
    // print ไม่ขึ้นบรรทัดใหม่
    print("Hello ")
    print("Kotlin!")
    println() // ขึ้นบรรทัดใหม่เปล่าๆ
    
    // แสดงข้อความพร้อม newline โดยตรง
    println("สวัสดีชาว Kotlin!")
}
```

**Output:**
```
Hello, World!
Hello Kotlin!
สวัสดีชาว Kotlin!
```

### เวอร์ชันรับ Arguments

```kotlin
fun main(args: Array<String>) {
    println("สวัสดี! ยินดีต้อนรับสู่ Kotlin")
    
    if (args.isEmpty()) {
        println("ไม่มี argument ที่ส่งมา")
    } else {
        println("Arguments ที่ได้รับ: ${args.size} ตัว")
        for (arg in args) {
            println("  - $arg")
        }
    }
}
```

**รัน:** `java -jar app.jar Kotlin Programming`

**Output:**
```
สวัสดี! ยินดีต้อนรับสู่ Kotlin
Arguments ที่ได้รับ: 2 ตัว
  - Kotlin
  - Programming
```

---

## โครงสร้างโปรแกรม Kotlin

### องค์ประกอบหลักของโปรแกรม Kotlin

```kotlin
// 1. Package Declaration (ไม่จำเป็น แต่ควรมีในโปรเจกต์จริง)
package com.example.hello

// 2. Import Statements
import kotlin.math.sqrt
import kotlin.math.PI

// 3. Top-Level Functions (ฟังก์ชันระดับ Top-level ไม่ต้องอยู่ใน Class)
fun greet(name: String): String {
    return "สวัสดี, $name!"
}

// 4. ฟังก์ชัน main - จุดเริ่มต้นของโปรแกรม
fun main() {
    // 5. การเรียกใช้ฟังก์ชัน
    val message = greet("นักเรียน Kotlin")
    println(message)
    
    // 6. การใช้ Library ที่ Import มา
    val radius = 5.0
    val area = PI * radius * radius
    println("พื้นที่วงกลม radius=$radius = $area")
    
    val hypotenuse = sqrt(3.0 * 3.0 + 4.0 * 4.0)
    println("ด้านตรงข้ามมุมฉาก = $hypotenuse")
}
```

**Output:**
```
สวัสดี, นักเรียน Kotlin!
พื้นที่วงกลม radius=5.0 = 78.53981633974483
ด้านตรงข้ามมุมฉาก = 5.0
```

### ความแตกต่างระหว่าง Kotlin และ Java

```kotlin
// ========== Kotlin ==========
// 1. โปรแกรม Hello World ง่ายกว่ามาก
fun main() {
    println("Hello, World!")
}

// 2. ประกาศตัวแปรง่ายๆ
val name = "Kotlin"  // Type Inference - ไม่ต้องระบุ type
var age = 20

// 3. String Template - แทรกตัวแปรใน String ได้เลย
println("ฉันชื่อ $name อายุ $age ปี")

// 4. Null Safety
var nullableString: String? = null  // ต้องใส่ ? เพื่อบอกว่า nullable
println(nullableString?.length)     // Safe call - ไม่ crash

// 5. Data Class - ไม่ต้องเขียน getter/setter เอง
data class Person(val name: String, val age: Int)
val person = Person("สมชาย", 25)
println(person)  // Person(name=สมชาย, age=25)
```

```java
// ========== Java เทียบกัน ==========
// 1. ต้องมี Class และ method signature ยาวกว่า
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}

// 2. ต้องระบุ Type ทุกครั้ง
String name = "Java";
int age = 20;

// 3. String Concatenation ยุ่งยากกว่า
System.out.println("ฉันชื่อ " + name + " อายุ " + age + " ปี");

// 4. ต้องตรวจสอบ null เอง
if (nullableString != null) {
    System.out.println(nullableString.length());
}

// 5. ต้องเขียน getter/setter เอง หรือใช้ Lombok
public class Person {
    private String name;
    private int age;
    public Person(String name, int age) { this.name = name; this.age = age; }
    public String getName() { return name; }
    public int getAge() { return age; }
    public String toString() { return "Person{name=" + name + ", age=" + age + "}"; }
    // ... equals, hashCode ด้วย!
}
```

---

## การ Compile และ Run โปรแกรม

### วิธีที่ 1: ผ่าน IntelliJ IDEA (ง่ายที่สุด)

1. เปิดไฟล์ `Main.kt`
2. คลิกที่ปุ่ม ▶️ สีเขียวข้างๆ ฟังก์ชัน `main`
3. หรือกด `Shift + F10` (Windows/Linux) หรือ `Ctrl + R` (macOS)
4. ดูผลลัพธ์ที่ **Run** panel ด้านล่าง

### วิธีที่ 2: ผ่าน Command Line ด้วย Gradle

```bash
# รันโปรแกรม
./gradlew run          # macOS/Linux
gradlew.bat run        # Windows

# Build เป็น JAR ไฟล์
./gradlew build

# รัน JAR ไฟล์
java -jar build/libs/HelloKotlin-1.0-SNAPSHOT.jar
```

### วิธีที่ 3: ใช้ Kotlin Compiler โดยตรง

```bash
# ติดตั้ง Kotlin Compiler (ถ้ายังไม่ได้ติดตั้ง)
# macOS:
brew install kotlin

# Linux (SDKMAN):
sdk install kotlin

# Windows (Chocolatey):
choco install kotlinc

# Compile ไฟล์ .kt
kotlinc Main.kt -include-runtime -d main.jar

# รัน
java -jar main.jar
```

### วิธีที่ 4: ใช้ Script Mode (เหมาะสำหรับทดสอบเล็กๆ)

สร้างไฟล์ `hello.kts`:
```kotlin
println("Hello from Kotlin Script!")

val list = listOf(1, 2, 3, 4, 5)
println("Sum: ${list.sum()}")
println("Average: ${list.average()}")
```

รัน:
```bash
kotlinc -script hello.kts
```

**Output:**
```
Hello from Kotlin Script!
Sum: 15
Average: 3.0
```

---

## Kotlin Playground

ถ้าไม่ต้องการติดตั้งอะไรเลย สามารถทดลองเขียน Kotlin ได้ทาง **Kotlin Playground**:

🔗 https://play.kotlinlang.org/

### ฟีเจอร์ของ Kotlin Playground

```
┌─────────────────────────────────────────────────────────────────┐
│                       Kotlin Playground                          │
├─────────────────────────────────────────────────────────────────┤
│ • เขียนและรัน Kotlin ได้ทันทีในเบราว์เซอร์                      │
│ • ไม่ต้องติดตั้งอะไรเลย                                         │
│ • มี Examples ให้ศึกษาหลายร้อยตัวอย่าง                          │
│ • แชร์โค้ดกับคนอื่นได้ผ่าน URL                                  │
│ • รองรับ Kotlin Coroutines                                       │
│ • ใช้ได้บน Mobile ด้วย                                          │
└─────────────────────────────────────────────────────────────────┘
```

ลองเขียนโปรแกรมนี้ใน Playground:

```kotlin
fun main() {
    val languages = listOf("Kotlin", "Java", "Python", "Swift", "Rust")
    
    println("=== ภาษาโปรแกรมที่ฉันรู้จัก ===")
    languages.forEachIndexed { index, language ->
        println("${index + 1}. $language")
    }
    
    println("\nจำนวน: ${languages.size} ภาษา")
    println("ภาษาโปรดของฉัน: ${languages.first()}")
}
```

**Output:**
```
=== ภาษาโปรแกรมที่ฉันรู้จัก ===
1. Kotlin
2. Java
3. Python
4. Swift
5. Rust

จำนวน: 5 ภาษา
ภาษาโปรดของฉัน: Kotlin
```

---

## ตัวอย่างโปรแกรมเพิ่มเติม

### โปรแกรมคำนวณ BMI

```kotlin
fun main() {
    println("=== โปรแกรมคำนวณ BMI ===")
    
    val weight = 70.0  // กิโลกรัม
    val height = 1.75  // เมตร
    
    val bmi = weight / (height * height)
    val bmiRounded = String.format("%.2f", bmi)
    
    println("น้ำหนัก: $weight kg")
    println("ส่วนสูง: $height m")
    println("BMI: $bmiRounded")
    
    val status = when {
        bmi < 18.5 -> "น้ำหนักน้อย (Underweight)"
        bmi < 25.0 -> "น้ำหนักปกติ (Normal)"
        bmi < 30.0 -> "น้ำหนักเกิน (Overweight)"
        else       -> "โรคอ้วน (Obese)"
    }
    
    println("สถานะ: $status")
}
```

**Output:**
```
=== โปรแกรมคำนวณ BMI ===
น้ำหนัก: 70.0 kg
ส่วนสูง: 1.75 m
BMI: 22.86
สถานะ: น้ำหนักปกติ (Normal)
```

### โปรแกรมตรวจสอบจำนวนเฉพาะ

```kotlin
fun isPrime(number: Int): Boolean {
    if (number < 2) return false
    if (number == 2) return true
    if (number % 2 == 0) return false
    
    for (i in 3..Math.sqrt(number.toDouble()).toInt() step 2) {
        if (number % i == 0) return false
    }
    return true
}

fun main() {
    println("=== จำนวนเฉพาะ 1-50 ===")
    
    val primes = (1..50).filter { isPrime(it) }
    println(primes.joinToString(", "))
    println("จำนวนเฉพาะทั้งหมด: ${primes.size} ตัว")
}
```

**Output:**
```
=== จำนวนเฉพาะ 1-50 ===
2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47
จำนวนเฉพาะทั้งหมด: 15 ตัว
```

### โปรแกรมวาดรูปดาว

```kotlin
fun printStar(rows: Int) {
    for (i in 1..rows) {
        // พิมพ์ space
        for (j in 1..(rows - i)) print(" ")
        // พิมพ์ดาว
        for (k in 1..(2 * i - 1)) print("*")
        println()
    }
}

fun main() {
    println("=== สามเหลี่ยมดาว ===")
    printStar(5)
    
    println("\n=== สี่เหลี่ยมดาว ===")
    val size = 5
    for (row in 1..size) {
        println("*".repeat(size))
    }
}
```

**Output:**
```
=== สามเหลี่ยมดาว ===
    *
   ***
  *****
 *******
*********

=== สี่เหลี่ยมดาว ===
*****
*****
*****
*****
*****
```

---

## การตั้งค่า IntelliJ IDEA เพิ่มเติม

### การตั้งค่าที่แนะนำ

**1. เปิด Auto Import**
```
File → Settings → Editor → General → Auto Import
✅ Add unambiguous imports on the fly
✅ Optimize imports on the fly
```

**2. ตั้งค่า Font ให้อ่านง่าย**
```
File → Settings → Editor → Font
Font: JetBrains Mono (แนะนำ)
Size: 14-16
Line height: 1.3
```

**3. เปิด Line Numbers**
```
File → Settings → Editor → General → Appearance
✅ Show line numbers
✅ Show method separators
```

**4. ตั้งค่า Code Style**
```
File → Settings → Editor → Code Style → Kotlin
Indent: 4 spaces
```

### Shortcuts ที่ใช้บ่อย

```
Ctrl + Space          = Code Completion
Ctrl + Shift + Space  = Smart Code Completion
Alt + Enter           = Quick Fix / Suggestion
Ctrl + /              = Toggle Comment
Ctrl + D              = Duplicate Line
Ctrl + Y              = Delete Line
Shift + F6            = Rename
Ctrl + Alt + L        = Reformat Code
Ctrl + B              = Go to Declaration
Ctrl + F              = Find in File
Ctrl + Shift + F      = Find in Project
```

**macOS:**
```
⌘ + Space          = Code Completion
⌥ + Enter          = Quick Fix
⌘ + /              = Toggle Comment
⌘ + D              = Duplicate Line
⌘ + ⌫             = Delete Line
⇧ + F6             = Rename
⌘ + ⌥ + L         = Reformat Code
```

---

## แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Hello ชื่อของคุณ
เขียนโปรแกรมที่แสดงข้อความ `"สวัสดี [ชื่อของคุณ]! ยินดีต้อนรับสู่ Kotlin"`

```kotlin
// Solution
fun main() {
    val myName = "นักเรียน"  // เปลี่ยนเป็นชื่อของคุณ
    println("สวัสดี $myName! ยินดีต้อนรับสู่ Kotlin")
}
```

### แบบฝึกหัดที่ 2: คำนวณพื้นที่สี่เหลี่ยม
เขียนโปรแกรมคำนวณพื้นที่สี่เหลี่ยมผืนผ้า กว้าง 10 หน่วย ยาว 15 หน่วย

```kotlin
// Solution
fun main() {
    val width = 10.0
    val length = 15.0
    val area = width * length
    val perimeter = 2 * (width + length)
    
    println("=== สี่เหลี่ยมผืนผ้า ===")
    println("กว้าง: $width หน่วย")
    println("ยาว: $length หน่วย")
    println("พื้นที่: $area ตารางหน่วย")
    println("เส้นรอบรูป: $perimeter หน่วย")
}
```

**Output:**
```
=== สี่เหลี่ยมผืนผ้า ===
กว้าง: 10.0 หน่วย
ยาว: 15.0 หน่วย
พื้นที่: 150.0 ตารางหน่วย
เส้นรอบรูป: 50.0 หน่วย
```

### แบบฝึกหัดที่ 3: แสดงตาราง
เขียนโปรแกรมแสดงตารางสูตรคูณแม่ 3 ตั้งแต่ 1-10

```kotlin
// Solution
fun main() {
    val n = 3
    println("=== ตารางสูตรคูณแม่ $n ===")
    for (i in 1..10) {
        println("$n x $i = ${n * i}")
    }
}
```

**Output:**
```
=== ตารางสูตรคูณแม่ 3 ===
3 x 1 = 3
3 x 2 = 6
3 x 3 = 9
3 x 4 = 12
3 x 5 = 15
3 x 6 = 18
3 x 7 = 21
3 x 8 = 24
3 x 9 = 27
3 x 10 = 30
```

### แบบฝึกหัดที่ 4: ท้าทาย - Fibonacci
เขียนโปรแกรมแสดงตัวเลข Fibonacci 10 ตัวแรก (0, 1, 1, 2, 3, 5, 8, 13, 21, 34)

```kotlin
// Solution
fun main() {
    val n = 10
    var a = 0
    var b = 1
    
    print("Fibonacci $n ตัวแรก: ")
    repeat(n) {
        print("$a ")
        val temp = a + b
        a = b
        b = temp
    }
    println()
}
```

**Output:**
```
Fibonacci 10 ตัวแรก: 0 1 1 2 3 5 8 13 21 34
```

---

## สรุป Part 01

ในบทนี้คุณได้เรียนรู้:

```
✅ Kotlin คืออะไรและมีประวัติอย่างไร
✅ ทำไม Kotlin ถึงเป็นภาษาที่น่าเรียนและมีตลาดงาน
✅ Kotlin ทำงานอย่างไร (JVM, JS, Native)
✅ การติดตั้ง JDK สำหรับทุก OS
✅ การติดตั้งและตั้งค่า IntelliJ IDEA
✅ การสร้างโปรเจกต์ Kotlin ใหม่
✅ การเขียนโปรแกรม Hello World
✅ โครงสร้างพื้นฐานของโปรแกรม Kotlin
✅ การ Compile และ Run โปรแกรม
✅ การใช้ Kotlin Playground สำหรับทดสอบโค้ด
```

---

## Part ถัดไป

**Part 02: ตัวแปรและชนิดข้อมูล (Variables and Data Types)**
- val vs var
- ชนิดข้อมูลพื้นฐาน: Int, Long, Float, Double, Boolean, Char, String
- Type Inference
- Type Conversion
- การประกาศค่าคงที่

---

*Part 01/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
