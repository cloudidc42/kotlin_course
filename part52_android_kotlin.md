# Part 52: Android Development ด้วย Kotlin

## สารบัญ
1. [Android Kotlin Fundamentals](#android-kotlin-fundamentals)
2. [Architecture Components](#architecture-components)
3. [Coroutines ใน Android](#coroutines-ใน-android)
4. [Hilt Dependency Injection](#hilt-dependency-injection)
5. [Room Database](#room-database)
6. [Network Layer](#network-layer)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Android Kotlin Fundamentals

```kotlin
// build.gradle.kts (app level)
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("com.google.dagger.hilt.android")
    id("com.google.devtools.ksp")
}

android {
    compileSdk = 35
    
    defaultConfig {
        applicationId = "com.example.myapp"
        minSdk = 26
        targetSdk = 35
        versionCode = 1
        versionName = "1.0.0"
    }
    
    buildFeatures {
        viewBinding = true
        compose = true  // สำหรับ Compose
    }
    
    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.15"
    }
}

dependencies {
    // Android core
    implementation("androidx.core:core-ktx:1.13.1")
    implementation("androidx.appcompat:appcompat:1.7.0")
    
    // Architecture Components
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.8.7")
    implementation("androidx.lifecycle:lifecycle-livedata-ktx:2.8.7")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.7")
    implementation("androidx.activity:activity-ktx:1.9.3")
    implementation("androidx.fragment:fragment-ktx:1.8.5")
    
    // Navigation
    implementation("androidx.navigation:navigation-fragment-ktx:2.8.4")
    implementation("androidx.navigation:navigation-ui-ktx:2.8.4")
    
    // Room
    implementation("androidx.room:room-runtime:2.6.1")
    implementation("androidx.room:room-ktx:2.6.1")
    ksp("androidx.room:room-compiler:2.6.1")
    
    // Hilt
    implementation("com.google.dagger:hilt-android:2.52")
    ksp("com.google.dagger:hilt-compiler:2.52")
    
    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.8.1")
    
    // Retrofit + OkHttp
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.squareup.retrofit2:converter-gson:2.11.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
    
    // Coil (image loading)
    implementation("io.coil-kt:coil:2.7.0")
}
```

---

## Architecture Components

### ViewModel + StateFlow

```kotlin
// data class สำหรับ UI state
data class ProductUiState(
    val products: List<ProductUiModel> = emptyList(),
    val isLoading: Boolean = false,
    val error: String? = null,
    val searchQuery: String = ""
)

data class ProductUiModel(
    val id: String,
    val name: String,
    val price: String,
    val imageUrl: String?,
    val isInStock: Boolean
)

// ViewModel
@HiltViewModel
class ProductViewModel @Inject constructor(
    private val productRepository: ProductRepo
) : ViewModel() {
    
    private val _uiState = MutableStateFlow(ProductUiState())
    val uiState: StateFlow<ProductUiState> = _uiState.asStateFlow()
    
    private val _navigationEvent = MutableSharedFlow<NavigationEvent>()
    val navigationEvent: SharedFlow<NavigationEvent> = _navigationEvent.asSharedFlow()
    
    init {
        loadProducts()
    }
    
    fun loadProducts(query: String = "") {
        viewModelScope.launch {
            _uiState.update { it.copy(isLoading = true, error = null) }
            
            productRepository.searchProducts(query)
                .onSuccess { products ->
                    _uiState.update { state ->
                        state.copy(
                            products = products.map { it.toUiModel() },
                            isLoading = false,
                            searchQuery = query
                        )
                    }
                }
                .onFailure { error ->
                    _uiState.update { state ->
                        state.copy(isLoading = false, error = error.message)
                    }
                }
        }
    }
    
    fun onProductClicked(productId: String) {
        viewModelScope.launch {
            _navigationEvent.emit(NavigationEvent.ToProductDetail(productId))
        }
    }
    
    fun onSearchQueryChanged(query: String) {
        viewModelScope.launch {
            // Debounce ป้องกันการค้นหาบ่อยเกินไป
            kotlinx.coroutines.delay(300)
            loadProducts(query)
        }
    }
}

sealed class NavigationEvent {
    data class ToProductDetail(val productId: String) : NavigationEvent()
    object ToCart : NavigationEvent()
}
```

### Fragment + ViewBinding

```kotlin
@AndroidEntryPoint
class ProductListFragment : Fragment(R.layout.fragment_product_list) {
    
    private var _binding: FragmentProductListBinding? = null
    private val binding get() = _binding!!
    
    private val viewModel: ProductViewModel by viewModels()
    private lateinit var adapter: ProductAdapter
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        _binding = FragmentProductListBinding.bind(view)
        
        setupRecyclerView()
        setupSearch()
        observeViewModel()
    }
    
    private fun setupRecyclerView() {
        adapter = ProductAdapter(
            onProductClick = viewModel::onProductClicked
        )
        binding.recyclerView.apply {
            this.adapter = this@ProductListFragment.adapter
            layoutManager = GridLayoutManager(context, 2)
            addItemDecoration(GridSpacingDecoration(16))
        }
    }
    
    private fun setupSearch() {
        binding.searchEditText.doAfterTextChanged { text ->
            viewModel.onSearchQueryChanged(text?.toString() ?: "")
        }
    }
    
    private fun observeViewModel() {
        // Collect StateFlow ด้วย lifecycle-aware collection
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                launch {
                    viewModel.uiState.collect { state ->
                        renderState(state)
                    }
                }
                
                launch {
                    viewModel.navigationEvent.collect { event ->
                        handleNavigation(event)
                    }
                }
            }
        }
    }
    
    private fun renderState(state: ProductUiState) {
        binding.progressBar.isVisible = state.isLoading
        binding.errorText.isVisible = state.error != null
        binding.errorText.text = state.error
        binding.recyclerView.isVisible = !state.isLoading && state.error == null
        
        adapter.submitList(state.products)
    }
    
    private fun handleNavigation(event: NavigationEvent) {
        when (event) {
            is NavigationEvent.ToProductDetail -> {
                val action = ProductListFragmentDirections
                    .actionProductListToDetail(event.productId)
                findNavController().navigate(action)
            }
            NavigationEvent.ToCart -> {
                findNavController().navigate(R.id.cartFragment)
            }
        }
    }
    
    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null  // ป้องกัน memory leak
    }
}
```

### RecyclerView Adapter

```kotlin
class ProductAdapter(
    private val onProductClick: (String) -> Unit
) : ListAdapter<ProductUiModel, ProductAdapter.ProductViewHolder>(ProductDiffCallback()) {
    
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ProductViewHolder {
        val binding = ItemProductBinding.inflate(
            LayoutInflater.from(parent.context), parent, false
        )
        return ProductViewHolder(binding)
    }
    
    override fun onBindViewHolder(holder: ProductViewHolder, position: Int) {
        holder.bind(getItem(position))
    }
    
    inner class ProductViewHolder(
        private val binding: ItemProductBinding
    ) : RecyclerView.ViewHolder(binding.root) {
        
        init {
            binding.root.setOnClickListener {
                val item = getItem(adapterPosition)
                onProductClick(item.id)
            }
        }
        
        fun bind(product: ProductUiModel) {
            binding.apply {
                productName.text = product.name
                productPrice.text = product.price
                outOfStockBadge.isVisible = !product.isInStock
                
                productImage.load(product.imageUrl) {
                    crossfade(true)
                    placeholder(R.drawable.placeholder_product)
                    error(R.drawable.error_product)
                }
            }
        }
    }
    
    class ProductDiffCallback : DiffUtil.ItemCallback<ProductUiModel>() {
        override fun areItemsTheSame(old: ProductUiModel, new: ProductUiModel) = old.id == new.id
        override fun areContentsTheSame(old: ProductUiModel, new: ProductUiModel) = old == new
    }
}
```

---

## Room Database

```kotlin
// Entity
@Entity(
    tableName = "products",
    indices = [
        Index("category_id"),
        Index("sku", unique = true)
    ]
)
data class ProductEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "name") val name: String,
    @ColumnInfo(name = "description") val description: String,
    @ColumnInfo(name = "price") val price: Double,
    @ColumnInfo(name = "stock_quantity") val stockQuantity: Int,
    @ColumnInfo(name = "category_id") val categoryId: String?,
    @ColumnInfo(name = "sku") val sku: String?,
    @ColumnInfo(name = "image_url") val imageUrl: String?,
    @ColumnInfo(name = "is_active") val isActive: Boolean = true,
    @ColumnInfo(name = "created_at") val createdAt: Long = System.currentTimeMillis()
)

// DAO
@Dao
interface ProductDao {
    
    @Query("SELECT * FROM products WHERE is_active = 1 ORDER BY name")
    fun getAllProducts(): Flow<List<ProductEntity>>
    
    @Query("""
        SELECT * FROM products 
        WHERE is_active = 1 
        AND (:query = '' OR name LIKE '%' || :query || '%')
        AND (:categoryId IS NULL OR category_id = :categoryId)
        AND price BETWEEN :minPrice AND :maxPrice
        ORDER BY 
            CASE WHEN :sortBy = 'PRICE_ASC' THEN price END ASC,
            CASE WHEN :sortBy = 'PRICE_DESC' THEN price END DESC,
            CASE WHEN :sortBy = 'NAME' THEN name END ASC
    """)
    fun searchProducts(
        query: String,
        categoryId: String?,
        minPrice: Double,
        maxPrice: Double,
        sortBy: String = "NAME"
    ): Flow<List<ProductEntity>>
    
    @Query("SELECT * FROM products WHERE id = :id")
    suspend fun findById(id: String): ProductEntity?
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(products: List<ProductEntity>)
    
    @Update
    suspend fun update(product: ProductEntity)
    
    @Delete
    suspend fun delete(product: ProductEntity)
    
    @Query("DELETE FROM products WHERE id = :id")
    suspend fun deleteById(id: String)
    
    @Query("SELECT COUNT(*) FROM products WHERE is_active = 1")
    suspend fun getActiveProductCount(): Int
    
    // Transaction example
    @Transaction
    suspend fun replaceAll(products: List<ProductEntity>) {
        deleteAll()
        insertAll(products)
    }
    
    @Query("DELETE FROM products")
    suspend fun deleteAll()
}

// Type Converters
class Converters {
    @TypeConverter
    fun fromTimestamp(value: Long?): java.util.Date? =
        value?.let { java.util.Date(it) }
    
    @TypeConverter
    fun dateToTimestamp(date: java.util.Date?): Long? = date?.time
    
    @TypeConverter
    fun fromString(value: String?): List<String> =
        value?.split(",")?.filter { it.isNotBlank() } ?: emptyList()
    
    @TypeConverter
    fun fromList(list: List<String>?): String? = list?.joinToString(",")
}

// Database
@Database(
    entities = [ProductEntity::class, CartItemEntity::class, OrderEntity::class],
    version = 3,
    exportSchema = true
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    
    abstract fun productDao(): ProductDao
    abstract fun cartDao(): CartDao
    abstract fun orderDao(): OrderDao
    
    companion object {
        const val DATABASE_NAME = "myapp.db"
        
        fun create(context: android.content.Context): AppDatabase {
            return Room.databaseBuilder(context, AppDatabase::class.java, DATABASE_NAME)
                .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
                .build()
        }
        
        val MIGRATION_1_2 = object : Migration(1, 2) {
            override fun migrate(db: SupportSQLiteDatabase) {
                db.execSQL("ALTER TABLE products ADD COLUMN image_url TEXT")
            }
        }
        
        val MIGRATION_2_3 = object : Migration(2, 3) {
            override fun migrate(db: SupportSQLiteDatabase) {
                db.execSQL("""
                    CREATE TABLE cart_items (
                        id TEXT NOT NULL PRIMARY KEY,
                        product_id TEXT NOT NULL,
                        quantity INTEGER NOT NULL DEFAULT 1,
                        added_at INTEGER NOT NULL
                    )
                """)
            }
        }
    }
}

// Placeholder entities
@Entity(tableName = "cart_items")
data class CartItemEntity(@PrimaryKey val id: String, val productId: String, val quantity: Int, val addedAt: Long)

@Entity(tableName = "orders")
data class OrderEntity(@PrimaryKey val id: String, val totalAmount: Double, val status: String, val createdAt: Long)

@Dao interface CartDao
@Dao interface OrderDao
```

---

## Hilt Dependency Injection

```kotlin
// Application class
@HiltAndroidApp
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        // Global init
    }
}

// App Module
@Module
@InstallIn(SingletonComponent::class)
object AppModule {
    
    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: android.content.Context): AppDatabase =
        AppDatabase.create(context)
    
    @Provides
    fun provideProductDao(db: AppDatabase): ProductDao = db.productDao()
    
    @Provides
    @Singleton
    fun provideOkHttpClient(): okhttp3.OkHttpClient =
        okhttp3.OkHttpClient.Builder()
            .addInterceptor(okhttp3.logging.HttpLoggingInterceptor().apply {
                level = okhttp3.logging.HttpLoggingInterceptor.Level.BODY
            })
            .connectTimeout(30, java.util.concurrent.TimeUnit.SECONDS)
            .readTimeout(30, java.util.concurrent.TimeUnit.SECONDS)
            .build()
    
    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: okhttp3.OkHttpClient): retrofit2.Retrofit =
        retrofit2.Retrofit.Builder()
            .baseUrl("https://api.myapp.com/")
            .client(okHttpClient)
            .addConverterFactory(retrofit2.converter.gson.GsonConverterFactory.create())
            .build()
    
    @Provides
    @Singleton
    fun provideProductApiService(retrofit: retrofit2.Retrofit): ProductApiService =
        retrofit.create(ProductApiService::class.java)
    
    @Provides
    @Singleton
    fun provideProductRepository(
        api: ProductApiService,
        dao: ProductDao
    ): ProductRepo = ProductRepositoryImpl(api, dao)
}

// Repository interface and implementation
interface ProductRepo {
    suspend fun searchProducts(query: String): Result<List<ProductItem>>
    fun observeProducts(): Flow<List<ProductItem>>
}

data class ProductItem(val id: String, val name: String, val price: Double, val imageUrl: String?, val stockQuantity: Int)

fun ProductItem.toUiModel() = ProductUiModel(
    id = id, name = name,
    price = "฿${String.format("%.2f", price)}",
    imageUrl = imageUrl,
    isInStock = stockQuantity > 0
)

class ProductRepositoryImpl(
    private val api: ProductApiService,
    private val dao: ProductDao
) : ProductRepo {
    
    override suspend fun searchProducts(query: String): Result<List<ProductItem>> {
        return runCatching {
            val response = api.searchProducts(query)
            if (response.isSuccessful) {
                val items = response.body()?.map { it.toItem() } ?: emptyList()
                dao.insertAll(items.map { it.toEntity() })
                items
            } else {
                // Fallback to cache
                dao.searchProducts(query, null, 0.0, Double.MAX_VALUE).first()
                    .map { it.toItem() }
            }
        }
    }
    
    override fun observeProducts(): Flow<List<ProductItem>> =
        dao.getAllProducts().map { entities -> entities.map { it.toItem() } }
}

// Extensions
fun ProductEntity.toItem() = ProductItem(id, name, price, imageUrl, stockQuantity)
fun ProductItem.toEntity() = ProductEntity(id, name, "", price, stockQuantity, null, null, imageUrl)

// API Service
interface ProductApiService {
    @retrofit2.http.GET("products")
    suspend fun searchProducts(@retrofit2.http.Query("q") query: String): retrofit2.Response<List<ProductApiDto>>
}

data class ProductApiDto(val id: String, val name: String, val price: Double, val imageUrl: String?, val stockQuantity: Int) {
    fun toItem() = ProductItem(id, name, price, imageUrl, stockQuantity)
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement offline-first cart feature

// 1. CartEntity เก็บใน Room
// 2. CartViewModel manage cart state
// 3. Sync ไปยัง server เมื่อ online
// 4. Show total price แบบ real-time

@HiltViewModel
class CartViewModel @Inject constructor(
    private val cartRepository: CartRepository
) : ViewModel() {
    
    // TODO: Expose cart items as StateFlow<List<CartItemUiModel>>
    // TODO: Expose total price as StateFlow<Double>
    // TODO: Expose item count as StateFlow<Int>
    
    fun addToCart(productId: String, quantity: Int = 1) {
        viewModelScope.launch {
            // TODO: Implement add to cart
        }
    }
    
    fun removeFromCart(cartItemId: String) {
        viewModelScope.launch {
            // TODO: Implement remove from cart
        }
    }
    
    fun updateQuantity(cartItemId: String, newQuantity: Int) {
        viewModelScope.launch {
            // TODO: Implement quantity update, remove if quantity <= 0
        }
    }
    
    fun checkout() {
        viewModelScope.launch {
            // TODO: Implement checkout, clear cart after success
        }
    }
}

interface CartRepository {
    fun observeCart(): Flow<List<CartItemEntity>>
    suspend fun addItem(productId: String, quantity: Int)
    suspend fun updateQuantity(id: String, quantity: Int)
    suspend fun removeItem(id: String)
    suspend fun clearCart()
}
```

---

## สรุป Part 52

```
✅ Android build: plugins, dependencies, buildFeatures
✅ ViewModel: StateFlow + viewModelScope
✅ Fragment: ViewBinding, lifecycleScope, repeatOnLifecycle
✅ RecyclerView: ListAdapter + DiffUtil
✅ Navigation: Safe Args, NavController
✅ Room: Entity, DAO, Database, Migrations
✅ Type Converters: date, list to string
✅ Transaction: @Transaction annotation
✅ Hilt: Application, Module, @Inject, @Provides
✅ Retrofit: API service, interceptors, converters
✅ Repository pattern: offline-first with Room cache
✅ Coroutines: Flow, collect with lifecycle awareness
✅ Coil: async image loading
✅ Error handling: Result<T>, onSuccess/onFailure
```

---

*Part 52/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
