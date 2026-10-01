# Part 54: Compose Multiplatform

## สารบัญ
1. [Compose Multiplatform คืออะไร](#compose-multiplatform-คืออะไร)
2. [Project Setup](#project-setup)
3. [Shared UI Code](#shared-ui-code)
4. [Platform Specifics](#platform-specifics)
5. [Navigation Multiplatform](#navigation-multiplatform)
6. [ViewModel แบบ Multiplatform](#viewmodel-แบบ-multiplatform)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Compose Multiplatform คืออะไร

Compose Multiplatform ช่วยให้เขียน UI code เดียว แล้ว run บนหลาย platforms:

```
Target Platforms:
├── Android        (Production-ready)
├── iOS            (Beta/Stable)
├── Desktop        (JVM: macOS, Linux, Windows)
└── Web            (via Kotlin/WASM - Alpha)

Code Sharing:
commonMain/     ← Business logic + UI ที่ share ได้
androidMain/    ← Android specific code
iosMain/        ← iOS specific code
desktopMain/    ← Desktop specific code
```

---

## Project Setup

```kotlin
// build.gradle.kts (root)
plugins {
    kotlin("multiplatform") version "2.0.21" apply false
    id("com.android.application") version "8.7.2" apply false
    id("org.jetbrains.compose") version "1.7.1" apply false
    id("org.jetbrains.kotlin.plugin.compose") version "2.0.21" apply false
}

// composeApp/build.gradle.kts
plugins {
    kotlin("multiplatform")
    id("com.android.application")
    id("org.jetbrains.compose")
    id("org.jetbrains.kotlin.plugin.compose")
}

kotlin {
    androidTarget {
        compilations.all {
            kotlinOptions { jvmTarget = "17" }
        }
    }
    
    listOf(
        iosX64(),
        iosArm64(),
        iosSimulatorArm64()
    ).forEach { iosTarget ->
        iosTarget.binaries.framework {
            baseName = "ComposeApp"
            isStatic = true
        }
    }
    
    jvm("desktop")
    
    sourceSets {
        val commonMain by getting {
            dependencies {
                implementation(compose.runtime)
                implementation(compose.foundation)
                implementation(compose.material3)
                implementation(compose.ui)
                implementation(compose.components.resources)
                
                // Multiplatform libraries
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.8.1")
                implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")
                implementation("io.ktor:ktor-client-core:3.0.1")
                implementation("io.ktor:ktor-client-content-negotiation:3.0.1")
                implementation("io.ktor:ktor-serialization-kotlinx-json:3.0.1")
                
                // Navigation
                implementation("org.jetbrains.androidx.navigation:navigation-compose:2.8.0-alpha10")
                
                // Settings/Preferences
                implementation("com.russhwolf:multiplatform-settings:1.2.0")
            }
        }
        
        val androidMain by getting {
            dependencies {
                implementation("io.ktor:ktor-client-android:3.0.1")
                implementation("androidx.activity:activity-compose:1.9.3")
                implementation("com.google.android.material:material:1.12.0")
            }
        }
        
        val iosMain by creating {
            dependsOn(commonMain)
            dependencies {
                implementation("io.ktor:ktor-client-darwin:3.0.1")
            }
        }
        
        val desktopMain by getting {
            dependencies {
                implementation(compose.desktop.currentOs)
                implementation("io.ktor:ktor-client-java:3.0.1")
                implementation("org.jetbrains.kotlinx:kotlinx-coroutines-swing:1.8.1")
            }
        }
    }
}
```

---

## Shared UI Code

```kotlin
// commonMain/kotlin/App.kt
@Composable
fun App() {
    AppTheme {
        val navController = rememberNavController()
        
        NavHost(navController = navController, startDestination = "home") {
            composable("home") {
                HomeScreen(
                    onNavigateToDetail = { id -> navController.navigate("detail/$id") }
                )
            }
            composable("detail/{id}") { back ->
                DetailScreen(
                    id = back.arguments?.getString("id") ?: "",
                    onBack = navController::popBackStack
                )
            }
            composable("settings") {
                SettingsScreen()
            }
        }
    }
}

// commonMain/kotlin/screens/HomeScreen.kt
@Composable
fun HomeScreen(
    onNavigateToDetail: (String) -> Unit,
    viewModel: HomeViewModel = remember { HomeViewModel() }
) {
    val state by viewModel.state.collectAsState()
    
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("KMP App") },
                actions = {
                    IconButton(onClick = { /* settings */ }) {
                        Icon(Icons.Default.Settings, "Settings")
                    }
                }
            )
        }
    ) { padding ->
        Box(modifier = Modifier.padding(padding)) {
            when {
                state.isLoading -> LoadingView()
                state.error != null -> ErrorView(state.error!!)
                else -> ItemList(items = state.items, onItemClick = onNavigateToDetail)
            }
        }
    }
}

@Composable
fun ItemList(items: List<ItemModel>, onItemClick: (String) -> Unit) {
    LazyColumn {
        items(items = items, key = { it.id }) { item ->
            ListItem(
                headlineContent = { Text(item.title) },
                supportingContent = { Text(item.description) },
                trailingContent = {
                    Icon(Icons.Default.ChevronRight, contentDescription = null)
                },
                modifier = Modifier.clickable { onItemClick(item.id) }
            )
            HorizontalDivider()
        }
    }
}

@Composable
fun LoadingView() {
    Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
        CircularProgressIndicator()
    }
}

@Composable
fun ErrorView(message: String) {
    Box(Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            Icon(Icons.Default.Error, contentDescription = null, modifier = Modifier.size(48.dp))
            Spacer(Modifier.height(8.dp))
            Text(message, textAlign = TextAlign.Center)
        }
    }
}

// Theme
@Composable
fun AppTheme(content: @Composable () -> Unit) {
    MaterialTheme(
        colorScheme = lightColorScheme(
            primary = Color(0xFF6200EE),
            secondary = Color(0xFF03DAC6)
        ),
        content = content
    )
}

data class ItemModel(val id: String, val title: String, val description: String)
```

---

## ViewModel แบบ Multiplatform

```kotlin
// commonMain/kotlin/viewmodel/HomeViewModel.kt
data class HomeState(
    val items: List<ItemModel> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null
)

class HomeViewModel {
    
    private val _state = MutableStateFlow(HomeState())
    val state: StateFlow<HomeState> = _state.asStateFlow()
    
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Main)
    
    init {
        loadItems()
    }
    
    fun loadItems() {
        scope.launch {
            _state.update { it.copy(isLoading = true, error = null) }
            
            try {
                val items = ApiClient.getItems()
                _state.update { it.copy(items = items, isLoading = false) }
            } catch (e: Exception) {
                _state.update { it.copy(isLoading = false, error = e.message) }
            }
        }
    }
    
    fun onCleared() {
        scope.cancel()
    }
}

// commonMain/kotlin/network/ApiClient.kt
object ApiClient {
    
    private val client = HttpClient {
        install(ContentNegotiation) {
            json(Json {
                ignoreUnknownKeys = true
                isLenient = true
            })
        }
        install(HttpTimeout) {
            requestTimeoutMillis = 30_000
        }
    }
    
    suspend fun getItems(): List<ItemModel> {
        return client.get("https://api.example.com/items").body()
    }
    
    suspend fun getItemDetail(id: String): ItemModel {
        return client.get("https://api.example.com/items/$id").body()
    }
}

// Platform expect/actual สำหรับ JSON serialization
@Serializable
data class ItemDto(
    @SerialName("id") val id: String,
    @SerialName("title") val title: String,
    @SerialName("description") val description: String
)
```

---

## Platform Specifics

```kotlin
// commonMain: expect declarations
expect fun getPlatform(): Platform
expect fun openUrl(url: String)
expect fun shareText(text: String)

data class Platform(val name: String, val version: String)

// androidMain: actual implementations
actual fun getPlatform() = Platform(
    name = "Android",
    version = android.os.Build.VERSION.RELEASE
)

actual fun openUrl(url: String) {
    val intent = android.content.Intent(
        android.content.Intent.ACTION_VIEW,
        android.net.Uri.parse(url)
    )
    // Need context - use a static reference or pass it
}

actual fun shareText(text: String) {
    val intent = android.content.Intent(android.content.Intent.ACTION_SEND).apply {
        type = "text/plain"
        putExtra(android.content.Intent.EXTRA_TEXT, text)
    }
    // startActivity(intent)
}

// iosMain: actual implementations
actual fun getPlatform() = Platform(
    name = "iOS",
    version = platform.UIKit.UIDevice.currentDevice.systemVersion
)

actual fun openUrl(url: String) {
    val nsUrl = platform.Foundation.NSURL(string = url) ?: return
    platform.UIKit.UIApplication.sharedApplication.openURL(nsUrl)
}

actual fun shareText(text: String) {
    val items = listOf(text)
    val activityController = platform.UIKit.UIActivityViewController(
        activityItems = items,
        applicationActivities = null
    )
    // Present controller
}

// desktopMain: actual implementations
actual fun getPlatform(): Platform {
    val os = System.getProperty("os.name") ?: "Unknown"
    val version = System.getProperty("os.version") ?: ""
    return Platform(os, version)
}

actual fun openUrl(url: String) {
    java.awt.Desktop.getDesktop().browse(java.net.URI(url))
}

actual fun shareText(text: String) {
    val clipboard = java.awt.Toolkit.getDefaultToolkit().systemClipboard
    clipboard.setContents(java.awt.datatransfer.StringSelection(text), null)
}
```

---

## Desktop Entry Point

```kotlin
// desktopMain/kotlin/main.kt
import androidx.compose.ui.window.Window
import androidx.compose.ui.window.application

fun main() = application {
    Window(
        onCloseRequest = ::exitApplication,
        title = "My KMP App"
    ) {
        App()
    }
}

// desktopMain: Menu bar
fun main2() = application {
    val trayState = rememberTrayState()
    
    Tray(
        state = trayState,
        icon = painterResource("icon.png"),
        menu = {
            Item("Open") { /* navigate */ }
            Item("Quit") { exitApplication() }
        }
    )
    
    Window(
        onCloseRequest = ::exitApplication,
        title = "My App",
        state = rememberWindowState(
            placement = WindowPlacement.Floating,
            width = 1200.dp,
            height = 800.dp
        )
    ) {
        MenuBar {
            Menu("File") {
                Item("New", shortcut = KeyShortcut(Key.N, meta = true)) { /* new */ }
                Item("Open...", shortcut = KeyShortcut(Key.O, meta = true)) { /* open */ }
                Separator()
                Item("Quit", shortcut = KeyShortcut(Key.Q, meta = true)) { exitApplication() }
            }
            Menu("Edit") {
                Item("Copy", shortcut = KeyShortcut(Key.C, meta = true)) { /* copy */ }
                Item("Paste", shortcut = KeyShortcut(Key.V, meta = true)) { /* paste */ }
            }
        }
        App()
    }
}
```

---

## iOS Entry Point

```kotlin
// iosMain/kotlin/MainViewController.kt
import androidx.compose.ui.window.ComposeUIViewController

fun MainViewController() = ComposeUIViewController { App() }

// Swift side (AppDelegate.swift):
// import ComposeApp
// func application(...) {
//     window = UIWindow()
//     window?.rootViewController = MainViewController()
//     window?.makeKeyAndVisible()
// }
```

---

## Shared Resources

```kotlin
// commonMain/composeResources/values/strings.xml
// <resources>
//   <string name="app_name">My KMP App</string>
//   <string name="loading">กำลังโหลด...</string>
//   <string name="error_generic">เกิดข้อผิดพลาด กรุณาลองใหม่</string>
// </resources>

// ใช้งาน resources:
@Composable
fun MyScreen() {
    Text(text = stringResource(Res.string.app_name))
    
    Image(
        painter = painterResource(Res.drawable.logo),
        contentDescription = null
    )
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง Note-taking app ด้วย KMP

// Shared code (commonMain):
// 1. Note data class: id, title, content, createdAt, updatedAt
// 2. NoteRepository interface
// 3. NoteViewModel: CRUD operations
// 4. Shared UI: NoteListScreen, NoteDetailScreen, NoteEditorScreen

// Platform specific:
// - Android: Room for persistence
// - iOS: SQLite (via SQLDelight)
// - Desktop: local file storage

// SQLDelight (cross-platform SQL):
// dependencies {
//     implementation("app.cash.sqldelight:runtime:2.0.2")
//     androidMain: implementation("app.cash.sqldelight:android-driver:2.0.2")
//     iosMain: implementation("app.cash.sqldelight:native-driver:2.0.2")
//     desktopMain: implementation("app.cash.sqldelight:sqlite-driver:2.0.2")
// }

data class Note(
    val id: String = java.util.UUID.randomUUID().toString(),
    val title: String,
    val content: String,
    val createdAt: Long = System.currentTimeMillis(),
    val updatedAt: Long = System.currentTimeMillis()
)

interface NoteRepository {
    fun observeAll(): kotlinx.coroutines.flow.Flow<List<Note>>
    suspend fun getById(id: String): Note?
    suspend fun save(note: Note)
    suspend fun delete(id: String)
}
```

---

## สรุป Part 54

```
✅ KMP: write once, run on Android/iOS/Desktop/Web
✅ Compose Multiplatform: shared UI across platforms
✅ commonMain: business logic + UI shared
✅ expect/actual: platform-specific implementations
✅ Ktor Client: multiplatform HTTP client
✅ kotlinx.serialization: multiplatform JSON
✅ Multiplatform ViewModel: coroutine-based state
✅ Navigation Multiplatform: shared routes
✅ Resources: shared strings, images, fonts
✅ Desktop: Window, MenuBar, Tray, keyboard shortcuts
✅ iOS: ComposeUIViewController, Swift interop
✅ Platform APIs: openUrl, shareText, clipboard
✅ SQLDelight: cross-platform SQL database
```

---

*Part 54/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
