# Part 53: Jetpack Compose

## สารบัญ
1. [Compose Fundamentals](#compose-fundamentals)
2. [State Management](#state-management)
3. [Layouts](#layouts)
4. [Navigation](#navigation)
5. [Animations](#animations)
6. [Testing Compose](#testing-compose)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Compose Fundamentals

Jetpack Compose คือ modern, declarative UI toolkit สำหรับ Android

```kotlin
// Composable function
@Composable
fun ProductCard(
    product: ProductUiModel,
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Card(
        onClick = onClick,
        modifier = modifier.fillMaxWidth(),
        elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            AsyncImage(
                model = product.imageUrl,
                contentDescription = product.name,
                modifier = Modifier
                    .fillMaxWidth()
                    .height(180.dp)
                    .clip(RoundedCornerShape(8.dp)),
                contentScale = ContentScale.Crop,
                placeholder = painterResource(R.drawable.placeholder)
            )
            
            Spacer(modifier = Modifier.height(12.dp))
            
            Text(
                text = product.name,
                style = MaterialTheme.typography.titleMedium,
                maxLines = 2,
                overflow = TextOverflow.Ellipsis
            )
            
            Spacer(modifier = Modifier.height(4.dp))
            
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.SpaceBetween,
                verticalAlignment = Alignment.CenterVertically
            ) {
                Text(
                    text = product.price,
                    style = MaterialTheme.typography.titleLarge,
                    color = MaterialTheme.colorScheme.primary
                )
                
                if (!product.isInStock) {
                    Badge {
                        Text("หมด")
                    }
                }
            }
        }
    }
}

// Preview
@Preview(showBackground = true)
@Composable
fun ProductCardPreview() {
    MaterialTheme {
        ProductCard(
            product = ProductUiModel(
                id = "1",
                name = "iPhone 15 Pro Max 256GB",
                price = "฿52,900",
                imageUrl = null,
                isInStock = true
            ),
            onClick = {}
        )
    }
}
```

---

## State Management

```kotlin
// ===== Local State =====
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    
    Column(horizontalAlignment = Alignment.CenterHorizontally) {
        Text(
            text = count.toString(),
            style = MaterialTheme.typography.displayMedium
        )
        
        Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(onClick = { count-- }) { Text("-") }
            Button(onClick = { count++ }) { Text("+") }
        }
    }
}

// ===== Hoisted State =====
@Composable
fun SearchBar(
    query: String,
    onQueryChange: (String) -> Unit,
    modifier: Modifier = Modifier
) {
    OutlinedTextField(
        value = query,
        onValueChange = onQueryChange,
        modifier = modifier.fillMaxWidth(),
        placeholder = { Text("ค้นหาสินค้า...") },
        leadingIcon = { Icon(Icons.Default.Search, contentDescription = "Search") },
        trailingIcon = {
            if (query.isNotEmpty()) {
                IconButton(onClick = { onQueryChange("") }) {
                    Icon(Icons.Default.Clear, contentDescription = "Clear")
                }
            }
        },
        singleLine = true,
        keyboardOptions = KeyboardOptions(imeAction = ImeAction.Search)
    )
}

// ===== ViewModel State =====
@Composable
fun ProductListScreen(
    viewModel: ProductViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    ProductListContent(
        uiState = uiState,
        onProductClick = viewModel::onProductClicked,
        onSearchChange = viewModel::onSearchQueryChanged,
        onRefresh = viewModel::loadProducts,
        onRetry = viewModel::loadProducts
    )
}

@Composable
fun ProductListContent(
    uiState: ProductUiState,
    onProductClick: (String) -> Unit,
    onSearchChange: (String) -> Unit,
    onRefresh: () -> Unit,
    onRetry: () -> Unit
) {
    Column(modifier = Modifier.fillMaxSize()) {
        // Search
        SearchBar(
            query = uiState.searchQuery,
            onQueryChange = onSearchChange,
            modifier = Modifier.padding(horizontal = 16.dp, vertical = 8.dp)
        )
        
        // Content
        when {
            uiState.isLoading -> {
                Box(modifier = Modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
                    CircularProgressIndicator()
                }
            }
            
            uiState.error != null -> {
                ErrorState(
                    message = uiState.error,
                    onRetry = onRetry
                )
            }
            
            uiState.products.isEmpty() -> {
                EmptyState(message = "ไม่พบสินค้า")
            }
            
            else -> {
                PullToRefreshLazyColumn(
                    isRefreshing = uiState.isLoading,
                    onRefresh = onRefresh
                ) {
                    LazyVerticalGrid(
                        columns = GridCells.Fixed(2),
                        contentPadding = PaddingValues(16.dp),
                        horizontalArrangement = Arrangement.spacedBy(8.dp),
                        verticalArrangement = Arrangement.spacedBy(8.dp)
                    ) {
                        items(
                            items = uiState.products,
                            key = { it.id }
                        ) { product ->
                            ProductCard(
                                product = product,
                                onClick = { onProductClick(product.id) }
                            )
                        }
                    }
                }
            }
        }
    }
}

// Reusable composables
@Composable
fun ErrorState(message: String, onRetry: () -> Unit) {
    Column(
        modifier = Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Icon(
            imageVector = Icons.Default.Warning,
            contentDescription = null,
            modifier = Modifier.size(64.dp),
            tint = MaterialTheme.colorScheme.error
        )
        Spacer(Modifier.height(16.dp))
        Text(text = message, textAlign = TextAlign.Center)
        Spacer(Modifier.height(16.dp))
        Button(onClick = onRetry) { Text("ลองอีกครั้ง") }
    }
}

@Composable
fun EmptyState(message: String, modifier: Modifier = Modifier) {
    Box(modifier = modifier.fillMaxSize(), contentAlignment = Alignment.Center) {
        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            Icon(Icons.Default.ShoppingCart, contentDescription = null, modifier = Modifier.size(64.dp))
            Spacer(Modifier.height(8.dp))
            Text(message, style = MaterialTheme.typography.bodyLarge)
        }
    }
}

// Placeholder composable
@Composable
fun PullToRefreshLazyColumn(
    isRefreshing: Boolean,
    onRefresh: () -> Unit,
    content: @Composable () -> Unit
) {
    // Implementation using PullRefreshIndicator or SwipeRefresh
    content()
}
```

---

## Layouts

```kotlin
// ===== Adaptive Layout =====
@Composable
fun ProductDetailScreen(
    productId: String,
    viewModel: ProductDetailViewModel = hiltViewModel()
) {
    val product by viewModel.product.collectAsStateWithLifecycle()
    val windowSizeClass = calculateWindowSizeClass(LocalActivity.current)
    
    if (windowSizeClass.widthSizeClass == WindowWidthSizeClass.Expanded) {
        // Tablet layout: side by side
        Row(modifier = Modifier.fillMaxSize()) {
            ProductImages(
                images = product?.images ?: emptyList(),
                modifier = Modifier.weight(1f)
            )
            ProductInfo(
                product = product,
                modifier = Modifier.weight(1f)
            )
        }
    } else {
        // Phone layout: scrollable column
        LazyColumn(modifier = Modifier.fillMaxSize()) {
            item {
                ProductImages(images = product?.images ?: emptyList())
            }
            item {
                ProductInfo(product = product)
            }
        }
    }
}

@Composable
fun ProductImages(images: List<String>, modifier: Modifier = Modifier) {
    var selectedIndex by remember { mutableStateOf(0) }
    
    Column(modifier = modifier) {
        // Main image
        AsyncImage(
            model = images.getOrNull(selectedIndex),
            contentDescription = null,
            modifier = Modifier
                .fillMaxWidth()
                .height(300.dp),
            contentScale = ContentScale.Fit
        )
        
        // Thumbnails
        if (images.size > 1) {
            LazyRow(
                horizontalArrangement = Arrangement.spacedBy(8.dp),
                contentPadding = PaddingValues(horizontal = 16.dp, vertical = 8.dp)
            ) {
                itemsIndexed(images) { index, imageUrl ->
                    AsyncImage(
                        model = imageUrl,
                        contentDescription = null,
                        modifier = Modifier
                            .size(60.dp)
                            .clip(RoundedCornerShape(4.dp))
                            .border(
                                width = if (index == selectedIndex) 2.dp else 0.dp,
                                color = MaterialTheme.colorScheme.primary,
                                shape = RoundedCornerShape(4.dp)
                            )
                            .clickable { selectedIndex = index }
                    )
                }
            }
        }
    }
}

@Composable
fun ProductInfo(product: ProductDetailUiModel?, modifier: Modifier = Modifier) {
    Column(
        modifier = modifier.padding(16.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        if (product == null) {
            repeat(5) {
                Box(
                    modifier = Modifier
                        .fillMaxWidth()
                        .height(20.dp)
                        .shimmer()
                )
            }
            return
        }
        
        Text(product.name, style = MaterialTheme.typography.headlineSmall)
        Text(
            text = product.price,
            style = MaterialTheme.typography.headlineMedium,
            color = MaterialTheme.colorScheme.primary
        )
        
        HorizontalDivider()
        
        Text("รายละเอียด", style = MaterialTheme.typography.titleMedium)
        Text(product.description, style = MaterialTheme.typography.bodyMedium)
        
        Spacer(Modifier.weight(1f))
        
        Button(
            onClick = { /* add to cart */ },
            modifier = Modifier.fillMaxWidth(),
            enabled = product.isInStock
        ) {
            Icon(Icons.Default.AddShoppingCart, contentDescription = null)
            Spacer(Modifier.width(8.dp))
            Text(if (product.isInStock) "เพิ่มลงตะกร้า" else "สินค้าหมด")
        }
    }
}

data class ProductDetailUiModel(
    val id: String, val name: String, val price: String,
    val description: String, val images: List<String>, val isInStock: Boolean
)
interface ProductDetailViewModel {
    val product: StateFlow<ProductDetailUiModel?>
}

// Shimmer loading effect
fun Modifier.shimmer(): Modifier = this.background(
    brush = Brush.linearGradient(
        colors = listOf(Color.LightGray.copy(0.6f), Color.LightGray.copy(0.2f), Color.LightGray.copy(0.6f))
    )
)
```

---

## Navigation with Compose

```kotlin
// Navigation setup
@Composable
fun AppNavigation() {
    val navController = rememberNavController()
    
    NavHost(
        navController = navController,
        startDestination = "product_list"
    ) {
        composable("product_list") {
            ProductListScreen(
                onProductClick = { id -> navController.navigate("product_detail/$id") },
                onCartClick = { navController.navigate("cart") }
            )
        }
        
        composable(
            route = "product_detail/{productId}",
            arguments = listOf(navArgument("productId") { type = NavType.StringType })
        ) { backStackEntry ->
            val productId = backStackEntry.arguments?.getString("productId") ?: return@composable
            ProductDetailScreen(
                productId = productId,
                onBackClick = navController::popBackStack,
                onAddToCart = { navController.navigate("cart") }
            )
        }
        
        composable("cart") {
            CartScreen(
                onCheckout = { navController.navigate("checkout") },
                onBackClick = navController::popBackStack
            )
        }
        
        composable("checkout") {
            CheckoutScreen(
                onOrderSuccess = { orderId ->
                    navController.navigate("order_success/$orderId") {
                        popUpTo("product_list")  // Clear backstack after order
                    }
                }
            )
        }
    }
}

// Placeholder composables
@Composable fun ProductListScreen(onProductClick: (String)->Unit, onCartClick: ()->Unit) {}
@Composable fun ProductDetailScreen(productId: String, onBackClick: ()->Unit, onAddToCart: ()->Unit) {}
@Composable fun CartScreen(onCheckout: ()->Unit, onBackClick: ()->Unit) {}
@Composable fun CheckoutScreen(onOrderSuccess: (String)->Unit) {}
```

---

## Animations

```kotlin
// Animated visibility
@Composable
fun AnimatedCart(itemCount: Int) {
    var visible by remember { mutableStateOf(true) }
    
    AnimatedVisibility(
        visible = visible,
        enter = slideInVertically(initialOffsetY = { -it }) + fadeIn(),
        exit = slideOutVertically(targetOffsetY = { -it }) + fadeOut()
    ) {
        Card(modifier = Modifier.padding(8.dp)) {
            Row(modifier = Modifier.padding(16.dp), verticalAlignment = Alignment.CenterVertically) {
                Icon(Icons.Default.ShoppingCart, contentDescription = null)
                Spacer(Modifier.width(8.dp))
                Text("$itemCount รายการ")
            }
        }
    }
}

// Animated counter
@Composable
fun AnimatedCounter(count: Int) {
    val animatedCount by animateIntAsState(
        targetValue = count,
        animationSpec = spring(stiffness = Spring.StiffnessMedium),
        label = "counter"
    )
    
    Text(
        text = animatedCount.toString(),
        style = MaterialTheme.typography.displayLarge
    )
}

// Swipe to dismiss
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun SwipeToDismissCartItem(
    item: CartItemUiModel,
    onRemove: (String) -> Unit,
    content: @Composable () -> Unit
) {
    val dismissState = rememberSwipeToDismissBoxState(
        confirmValueChange = { dismissValue ->
            if (dismissValue == SwipeToDismissBoxValue.StartToEnd ||
                dismissValue == SwipeToDismissBoxValue.EndToStart) {
                onRemove(item.id)
                true
            } else false
        }
    )
    
    SwipeToDismissBox(
        state = dismissState,
        backgroundContent = {
            val color by animateColorAsState(
                targetValue = when (dismissState.currentValue) {
                    SwipeToDismissBoxValue.Settled -> Color.Transparent
                    else -> Color.Red.copy(alpha = 0.7f)
                },
                label = "swipe_color"
            )
            
            Box(
                modifier = Modifier
                    .fillMaxSize()
                    .background(color)
                    .padding(horizontal = 20.dp),
                contentAlignment = Alignment.CenterEnd
            ) {
                Icon(
                    imageVector = Icons.Default.Delete,
                    contentDescription = "Delete",
                    tint = Color.White
                )
            }
        }
    ) {
        content()
    }
}

data class CartItemUiModel(val id: String, val name: String, val price: String, val quantity: Int)
```

---

## Testing Compose

```kotlin
@RunWith(AndroidJUnit4::class)
class ProductCardTest {
    
    @get:Rule
    val composeTestRule = createComposeRule()
    
    @Test
    fun productCard_displaysNameAndPrice() {
        val product = ProductUiModel(
            id = "1",
            name = "Samsung Galaxy S24",
            price = "฿29,900",
            imageUrl = null,
            isInStock = true
        )
        
        composeTestRule.setContent {
            MaterialTheme {
                ProductCard(product = product, onClick = {})
            }
        }
        
        composeTestRule.onNodeWithText("Samsung Galaxy S24").assertIsDisplayed()
        composeTestRule.onNodeWithText("฿29,900").assertIsDisplayed()
    }
    
    @Test
    fun productCard_showsOutOfStockBadge_whenNotInStock() {
        val product = ProductUiModel("1", "Widget", "฿100", null, isInStock = false)
        
        composeTestRule.setContent {
            MaterialTheme {
                ProductCard(product = product, onClick = {})
            }
        }
        
        composeTestRule.onNodeWithText("หมด").assertIsDisplayed()
    }
    
    @Test
    fun productCard_callsOnClick_whenClicked() {
        var clicked = false
        val product = ProductUiModel("1", "Widget", "฿100", null, isInStock = true)
        
        composeTestRule.setContent {
            MaterialTheme {
                ProductCard(product = product, onClick = { clicked = true })
            }
        }
        
        composeTestRule.onNodeWithText("Widget").performClick()
        
        assertTrue(clicked)
    }
    
    @Test
    fun searchBar_updatesText() {
        var searchQuery = ""
        
        composeTestRule.setContent {
            MaterialTheme {
                SearchBar(
                    query = searchQuery,
                    onQueryChange = { searchQuery = it }
                )
            }
        }
        
        composeTestRule.onNode(hasSetTextAction()).performTextInput("laptop")
        
        assertEquals("laptop", searchQuery)
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง Chat UI ด้วย Compose

// Requirements:
// 1. MessageBubble composable สำหรับ sent/received messages
// 2. Message input ด้านล่าง
// 3. LazyColumn แสดง messages
// 4. Auto-scroll เมื่อมี message ใหม่
// 5. Animation เมื่อ message ปรากฏ

@Composable
fun ChatScreen(
    messages: List<ChatMessage>,
    onSendMessage: (String) -> Unit
) {
    var inputText by remember { mutableStateOf("") }
    val listState = rememberLazyListState()
    
    // TODO: Auto-scroll to bottom on new message
    LaunchedEffect(messages.size) {
        // listState.animateScrollToItem(...)
    }
    
    Column(modifier = Modifier.fillMaxSize()) {
        // Messages
        LazyColumn(
            state = listState,
            modifier = Modifier.weight(1f),
            contentPadding = PaddingValues(16.dp),
            verticalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items(messages, key = { it.id }) { message ->
                AnimatedVisibility(
                    visible = true,
                    enter = slideInVertically { it } + fadeIn()
                ) {
                    MessageBubble(message = message)
                }
            }
        }
        
        // Input
        Row(
            modifier = Modifier.padding(8.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            TextField(
                value = inputText,
                onValueChange = { inputText = it },
                modifier = Modifier.weight(1f),
                placeholder = { Text("พิมพ์ข้อความ...") }
            )
            IconButton(
                onClick = {
                    if (inputText.isNotBlank()) {
                        onSendMessage(inputText)
                        inputText = ""
                    }
                }
            ) {
                Icon(Icons.Default.Send, "Send")
            }
        }
    }
}

@Composable
fun MessageBubble(message: ChatMessage) {
    // TODO: Different alignment for sent vs received
    // TODO: Show timestamp
    // TODO: Different colors for sent vs received
    TODO("Implement MessageBubble")
}

data class ChatMessage(
    val id: String,
    val text: String,
    val isSent: Boolean,
    val timestamp: Long
)
```

---

## สรุป Part 53

```
✅ @Composable: pure function, recompose on state change
✅ remember + mutableStateOf: local state
✅ State hoisting: lift state up for reuse
✅ collectAsStateWithLifecycle: safe Flow collection
✅ LazyColumn + LazyVerticalGrid: efficient lists
✅ Card, Column, Row, Box: layout composables
✅ Modifier chain: padding, fillMaxWidth, clip, etc.
✅ MaterialTheme: colors, typography, shapes
✅ AnimatedVisibility: show/hide with animation
✅ animateIntAsState: smooth number transitions
✅ SwipeToDismiss: gesture-based deletion
✅ NavHost + composable: navigation routes
✅ Hilt + hiltViewModel(): DI in Compose
✅ ComposeTestRule: UI testing
✅ Preview: instant design feedback
```

---

*Part 53/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
