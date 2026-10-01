# Part 84: Data Engineering — ETL Pipelines & Batch Processing

## สารบัญ
1. [Data Engineering Concepts](#data-engineering-concepts)
2. [Spring Batch Framework](#spring-batch-framework)
3. [ETL Pipeline](#etl-pipeline)
4. [Data Transformation](#data-transformation)
5. [Scheduler & Monitoring](#scheduler--monitoring)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Data Engineering Concepts

```
Data Engineering Pipeline Types:

1. Batch Processing: ประมวลผลข้อมูลเป็นก้อนใหญ่
   - เหมาะสำหรับ: รายงานประจำวัน, data warehouse loading
   - เครื่องมือ: Spring Batch, Apache Spark, dbt

2. Stream Processing: ประมวลผลข้อมูลแบบ real-time
   - เหมาะสำหรับ: fraud detection, live analytics
   - เครื่องมือ: Kafka Streams, Apache Flink

3. Micro-batch: ผสมระหว่าง batch และ stream
   - ประมวลผลทุก 1-5 นาที
   - เครื่องมือ: Apache Spark Structured Streaming

ETL vs ELT:
ETL: Extract → Transform → Load (transform ก่อน load)
ELT: Extract → Load → Transform (load raw data ก่อน แล้ว transform ใน warehouse)

Data Lakehouse:
Raw Zone → Cleaned Zone → Curated Zone → Serving Zone
(Bronze)    (Silver)        (Gold)          (Platinum)
```

---

## Spring Batch Framework

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-batch")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("com.opencsv:opencsv:5.8")
    implementation("org.apache.poi:poi-ooxml:5.2.5")  // Excel
    implementation("io.arrow-kt:arrow-core:1.2.0")    // Functional error handling
    runtimeOnly("org.postgresql:postgresql")
    runtimeOnly("com.h2database:h2")  // H2 for batch metadata
}

// BatchConfig.kt
import org.springframework.batch.core.*
import org.springframework.batch.core.configuration.annotation.EnableBatchProcessing
import org.springframework.batch.core.configuration.annotation.StepScope
import org.springframework.batch.core.job.builder.JobBuilder
import org.springframework.batch.core.repository.JobRepository
import org.springframework.batch.core.step.builder.StepBuilder
import org.springframework.batch.item.*
import org.springframework.batch.item.database.*
import org.springframework.batch.item.file.*
import org.springframework.batch.item.file.mapping.*
import org.springframework.batch.item.file.transform.*
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.transaction.PlatformTransactionManager

@Configuration
@EnableBatchProcessing
class BatchConfig {
    
    // Sales Report Job: read orders → calculate → write to report table
    @Bean
    fun dailySalesReportJob(
        jobRepository: JobRepository,
        dailySalesStep: Step
    ): Job {
        return JobBuilder("dailySalesReportJob", jobRepository)
            .incrementer(org.springframework.batch.core.launch.support.RunIdIncrementer())
            .start(dailySalesStep)
            .listener(JobExecutionListener())
            .build()
    }
    
    @Bean
    fun dailySalesStep(
        jobRepository: JobRepository,
        transactionManager: PlatformTransactionManager,
        orderReader: ItemReader<OrderEntity>,
        salesProcessor: ItemProcessor<OrderEntity, DailySalesRecord>,
        salesWriter: ItemWriter<DailySalesRecord>
    ): Step {
        return StepBuilder("dailySalesStep", jobRepository)
            .chunk<OrderEntity, DailySalesRecord>(100, transactionManager)
            .reader(orderReader)
            .processor(salesProcessor)
            .writer(salesWriter)
            .faultTolerant()
            .retry(java.net.ConnectException::class.java)
            .retryLimit(3)
            .skip(DataIntegrityException::class.java)
            .skipLimit(10)
            .listener(StepExecutionListener())
            .build()
    }
}

// Reader: JpaPagingItemReader
@Configuration
class ReaderConfig {
    
    @Bean
    @StepScope
    fun orderReader(
        entityManagerFactory: jakarta.persistence.EntityManagerFactory,
        @org.springframework.batch.core.configuration.annotation.Value("#{jobParameters['reportDate']}") reportDate: String?
    ): JpaPagingItemReader<OrderEntity> {
        return JpaPagingItemReaderBuilder<OrderEntity>()
            .name("orderReader")
            .entityManagerFactory(entityManagerFactory)
            .queryString("""
                SELECT o FROM OrderEntity o 
                WHERE o.status = 'DELIVERED' 
                AND CAST(o.deliveredAt AS DATE) = :reportDate
                ORDER BY o.id
            """.trimIndent())
            .parameterValues(mapOf("reportDate" to (reportDate ?: java.time.LocalDate.now().toString())))
            .pageSize(100)
            .build()
    }
    
    // CSV reader for import jobs
    @Bean
    @StepScope
    fun productCsvReader(
        @org.springframework.batch.core.configuration.annotation.Value("#{jobParameters['inputFile']}") inputFile: String?
    ): FlatFileItemReader<ProductImportRow> {
        return FlatFileItemReaderBuilder<ProductImportRow>()
            .name("productCsvReader")
            .resource(org.springframework.core.io.FileSystemResource(inputFile ?: "products.csv"))
            .delimited()
            .names("sku", "name", "price", "stockQuantity", "categoryId", "description")
            .fieldSetMapper(BeanWrapperFieldSetMapper<ProductImportRow>().apply {
                setTargetType(ProductImportRow::class.java)
            })
            .linesToSkip(1)  // Skip header
            .build()
    }
}

// Processor: Transform and validate
@org.springframework.stereotype.Component
class SalesProcessor : ItemProcessor<OrderEntity, DailySalesRecord> {
    
    override fun process(order: OrderEntity): DailySalesRecord? {
        // Return null to skip item
        if (order.totalAmount <= 0) return null
        
        return DailySalesRecord(
            orderId = order.id,
            customerId = order.customerId,
            totalAmount = order.totalAmount,
            itemCount = order.items.size,
            categoryBreakdown = order.items.groupBy { it.categoryId }
                .mapValues { (_, items) -> items.sumOf { it.price * it.quantity } },
            reportDate = java.time.LocalDate.now()
        )
    }
}

@org.springframework.stereotype.Component
class ProductImportProcessor(
    private val categoryRepository: CategoryRepository,
    private val validator: jakarta.validation.Validator
) : ItemProcessor<ProductImportRow, Product> {
    
    private val processedSkus = mutableSetOf<String>()
    
    override fun process(row: ProductImportRow): Product? {
        // Deduplicate within batch
        if (!processedSkus.add(row.sku)) {
            return null  // Skip duplicate SKU
        }
        
        val violations = validator.validate(row)
        if (violations.isNotEmpty()) {
            throw ValidationException("Invalid product: ${violations.map { it.message }}")
        }
        
        val category = categoryRepository.findByCode(row.categoryId)
            ?: throw DataIntegrityException("Category not found: ${row.categoryId}")
        
        return Product(
            id = java.util.UUID.randomUUID().toString(),
            sku = row.sku,
            name = row.name,
            price = row.price.toBigDecimal(),
            stockQuantity = row.stockQuantity,
            category = category,
            description = row.description
        )
    }
}

// Writer: JpaItemWriter
@Configuration
class WriterConfig {
    
    @Bean
    fun salesWriter(entityManagerFactory: jakarta.persistence.EntityManagerFactory): JpaItemWriter<DailySalesRecord> {
        return JpaItemWriterBuilder<DailySalesRecord>()
            .entityManagerFactory(entityManagerFactory)
            .build()
    }
    
    // Composite writer: write to multiple destinations
    @Bean
    fun productCompositeWriter(
        entityManagerFactory: jakarta.persistence.EntityManagerFactory,
        kafkaTemplate: org.springframework.kafka.core.KafkaTemplate<String, String>
    ): org.springframework.batch.item.support.CompositeItemWriter<Product> {
        val dbWriter = JpaItemWriterBuilder<Product>()
            .entityManagerFactory(entityManagerFactory)
            .build()
        
        val kafkaWriter = ItemWriter<Product> { products ->
            products.forEach { product ->
                kafkaTemplate.send(
                    "product.created",
                    product.id,
                    com.fasterxml.jackson.databind.ObjectMapper().writeValueAsString(product)
                )
            }
        }
        
        return org.springframework.batch.item.support.CompositeItemWriterBuilder<Product>()
            .delegates(listOf(dbWriter, kafkaWriter))
            .build()
    }
    
    // CSV writer for export jobs
    @Bean
    @StepScope
    fun reportCsvWriter(
        @org.springframework.batch.core.configuration.annotation.Value("#{jobParameters['outputFile']}") outputFile: String?
    ): FlatFileItemWriter<DailySalesRecord> {
        return FlatFileItemWriterBuilder<DailySalesRecord>()
            .name("reportCsvWriter")
            .resource(org.springframework.core.io.FileSystemResource(outputFile ?: "report.csv"))
            .headerCallback { writer -> writer.write("orderId,customerId,totalAmount,itemCount,reportDate") }
            .delimited()
            .delimiter(",")
            .names("orderId", "customerId", "totalAmount", "itemCount", "reportDate")
            .build()
    }
}

// Job Listeners
class JobExecutionListener : JobExecutionListenerSupport() {
    private val logger = org.slf4j.LoggerFactory.getLogger(javaClass)
    private var startTime: Long = 0
    
    override fun beforeJob(jobExecution: JobExecution) {
        startTime = System.currentTimeMillis()
        logger.info("Starting job: ${jobExecution.jobInstance.jobName}")
    }
    
    override fun afterJob(jobExecution: JobExecution) {
        val duration = System.currentTimeMillis() - startTime
        val readCount = jobExecution.stepExecutions.sumOf { it.readCount }
        val writeCount = jobExecution.stepExecutions.sumOf { it.writeCount }
        val skipCount = jobExecution.stepExecutions.sumOf { it.skipCount }
        
        logger.info("""
            Job ${jobExecution.jobInstance.jobName} completed: ${jobExecution.status}
            Duration: ${duration}ms
            Read: $readCount, Write: $writeCount, Skip: $skipCount
        """.trimIndent())
        
        if (jobExecution.status == BatchStatus.FAILED) {
            // Send alert
            jobExecution.allFailureExceptions.forEach { e ->
                logger.error("Job failed", e)
            }
        }
    }
}

class StepExecutionListener : StepExecutionListenerSupport() {
    private val logger = org.slf4j.LoggerFactory.getLogger(javaClass)
    
    override fun beforeStep(stepExecution: StepExecution) {
        logger.info("Starting step: ${stepExecution.stepName}")
    }
    
    override fun afterStep(stepExecution: StepExecution): ExitStatus? {
        logger.info("""
            Step ${stepExecution.stepName}: ${stepExecution.status}
            Read: ${stepExecution.readCount}
            Written: ${stepExecution.writeCount}
            Skipped: ${stepExecution.skipCount}
            Filtered: ${stepExecution.filterCount}
        """.trimIndent())
        return null
    }
}

// Data models
data class ProductImportRow(
    var sku: String = "",
    var name: String = "",
    var price: String = "",
    var stockQuantity: Int = 0,
    var categoryId: String = "",
    var description: String = ""
)

data class DailySalesRecord(
    val orderId: String,
    val customerId: String,
    val totalAmount: java.math.BigDecimal,
    val itemCount: Int,
    val categoryBreakdown: Map<String, java.math.BigDecimal>,
    val reportDate: java.time.LocalDate
) : java.io.Serializable

class DataIntegrityException(message: String) : RuntimeException(message)
class ValidationException(message: String) : RuntimeException(message)
```

---

## ETL Pipeline

```kotlin
// Multi-step ETL: Extract from source, Transform, Load to warehouse

@Configuration
class EtlPipelineConfig {
    
    @Bean
    fun customerDataSyncJob(
        jobRepository: JobRepository,
        extractStep: Step,
        transformStep: Step,
        loadStep: Step,
        cleanupStep: Step
    ): Job {
        return JobBuilder("customerDataSyncJob", jobRepository)
            .start(extractStep)
            .next(transformStep)
            .next(loadStep)
            .on("FAILED").to(cleanupStep)
            .from(loadStep).on("COMPLETED").end()
            .end()
            .build()
    }
    
    // Partitioned step for parallel processing
    @Bean
    fun partitionedProductStep(
        jobRepository: JobRepository,
        transactionManager: PlatformTransactionManager,
        partitioner: ProductPartitioner,
        workerStep: Step
    ): Step {
        return StepBuilder("partitionedProductStep", jobRepository)
            .partitioner("workerStep", partitioner)
            .step(workerStep)
            .gridSize(4)  // 4 parallel threads
            .taskExecutor(org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor().apply {
                corePoolSize = 4
                maxPoolSize = 8
                initialize()
            })
            .build()
    }
    
    @Bean
    fun workerStep(
        jobRepository: JobRepository,
        transactionManager: PlatformTransactionManager
    ): Step {
        return StepBuilder("workerStep", jobRepository)
            .chunk<ProductEntity, ProductWarehouseRecord>(50, transactionManager)
            .reader(partitionedProductReader(null))
            .processor(productWarehouseProcessor())
            .writer(warehouseWriter())
            .build()
    }
    
    @Bean
    @StepScope
    fun partitionedProductReader(
        @org.springframework.batch.core.configuration.annotation.Value("#{stepExecutionContext['minId']}") minId: Long?
    ): JpaPagingItemReader<ProductEntity> {
        return JpaPagingItemReaderBuilder<ProductEntity>()
            .name("partitionedProductReader")
            .entityManagerFactory(TODO("inject EntityManagerFactory"))
            .queryString("SELECT p FROM ProductEntity p WHERE p.id >= :minId AND p.id <= :maxId")
            .parameterValues(mapOf("minId" to (minId ?: 0L), "maxId" to (minId ?: 0L) + 1000L))
            .pageSize(50)
            .build()
    }
    
    private fun productWarehouseProcessor(): ItemProcessor<ProductEntity, ProductWarehouseRecord> =
        ItemProcessor { product ->
            ProductWarehouseRecord(
                productKey = product.id,
                sku = product.sku,
                productName = product.name,
                category = product.category.name,
                currentPrice = product.price,
                stockQuantity = product.stockQuantity,
                isActive = product.status == "ACTIVE",
                lastUpdated = java.time.LocalDateTime.now()
            )
        }
    
    private fun warehouseWriter(): ItemWriter<ProductWarehouseRecord> = TODO("implement warehouse writer")
}

// Partitioner: divides data into chunks for parallel processing
@org.springframework.stereotype.Component
class ProductPartitioner(
    private val productRepository: ProductRepository
) : org.springframework.batch.core.partition.support.Partitioner {
    
    override fun partition(gridSize: Int): Map<String, org.springframework.batch.item.ExecutionContext> {
        val totalCount = productRepository.count()
        val partitionSize = (totalCount / gridSize).coerceAtLeast(1)
        
        return (0 until gridSize).associate { partition ->
            val minId = partition * partitionSize
            val maxId = if (partition == gridSize - 1) totalCount else (partition + 1) * partitionSize - 1
            
            "partition$partition" to org.springframework.batch.item.ExecutionContext().apply {
                putLong("minId", minId)
                putLong("maxId", maxId)
                putInt("partitionNumber", partition)
            }
        }
    }
}

data class ProductWarehouseRecord(
    val productKey: String,
    val sku: String,
    val productName: String,
    val category: String,
    val currentPrice: java.math.BigDecimal,
    val stockQuantity: Int,
    val isActive: Boolean,
    val lastUpdated: java.time.LocalDateTime
)
```

---

## Data Transformation

```kotlin
// Functional ETL with Arrow
import arrow.core.*

data class RawOrderData(
    val orderId: String,
    val customerId: String,
    val amount: String,  // string from CSV
    val date: String,
    val items: String    // JSON string
)

data class CleanOrderData(
    val orderId: String,
    val customerId: String,
    val amount: java.math.BigDecimal,
    val date: java.time.LocalDate,
    val items: List<OrderItem>
)

data class OrderItem(val productId: String, val quantity: Int, val price: java.math.BigDecimal)

object OrderTransformer {
    
    fun transform(raw: RawOrderData): Either<List<String>, CleanOrderData> {
        val errors = mutableListOf<String>()
        
        val amount = raw.amount.toBigDecimalOrNull()
        if (amount == null) errors.add("Invalid amount: ${raw.amount}")
        
        val date = runCatching { java.time.LocalDate.parse(raw.date) }.getOrNull()
        if (date == null) errors.add("Invalid date: ${raw.date}")
        
        val items = runCatching {
            com.fasterxml.jackson.module.kotlin.jacksonObjectMapper()
                .readValue(raw.items, object : com.fasterxml.jackson.core.type.TypeReference<List<OrderItem>>() {})
        }.getOrNull()
        if (items == null) errors.add("Invalid items JSON: ${raw.items}")
        
        return if (errors.isEmpty()) {
            Either.Right(CleanOrderData(
                orderId = raw.orderId,
                customerId = raw.customerId,
                amount = amount!!,
                date = date!!,
                items = items!!
            ))
        } else {
            Either.Left(errors)
        }
    }
    
    fun transformBatch(raws: List<RawOrderData>): Pair<List<CleanOrderData>, List<Pair<RawOrderData, List<String>>>> {
        val successes = mutableListOf<CleanOrderData>()
        val failures = mutableListOf<Pair<RawOrderData, List<String>>>()
        
        raws.forEach { raw ->
            when (val result = transform(raw)) {
                is Either.Right -> successes.add(result.value)
                is Either.Left -> failures.add(raw to result.value)
            }
        }
        
        return successes to failures
    }
}

// Data Quality Checks
class DataQualityChecker {
    
    data class QualityReport(
        val totalRecords: Int,
        val validRecords: Int,
        val invalidRecords: Int,
        val nullCountByField: Map<String, Int>,
        val duplicateCount: Int,
        val outlierCount: Int,
        val qualityScore: Double
    ) {
        val passedThreshold: Boolean get() = qualityScore >= 0.95
    }
    
    fun check(records: List<CleanOrderData>): QualityReport {
        val total = records.size
        var invalid = 0
        val nullCounts = mutableMapOf<String, Int>()
        
        // Null checks
        val nullAmounts = records.count { it.amount <= java.math.BigDecimal.ZERO }
        if (nullAmounts > 0) nullCounts["amount"] = nullAmounts
        
        // Duplicate check
        val duplicates = records.size - records.distinctBy { it.orderId }.size
        
        // Outlier check (amount > 3 standard deviations)
        val amounts = records.map { it.amount.toDouble() }
        val mean = amounts.average()
        val stdDev = Math.sqrt(amounts.sumOf { (it - mean).pow(2.0) } / amounts.size)
        val outliers = amounts.count { Math.abs(it - mean) > 3 * stdDev }
        
        invalid += nullAmounts + duplicates
        
        val valid = total - invalid
        val qualityScore = if (total > 0) valid.toDouble() / total else 1.0
        
        return QualityReport(
            totalRecords = total,
            validRecords = valid,
            invalidRecords = invalid,
            nullCountByField = nullCounts,
            duplicateCount = duplicates,
            outlierCount = outliers,
            qualityScore = qualityScore
        )
    }
}

fun Double.pow(n: Double): Double = Math.pow(this, n)

// Aggregate calculations
class SalesAggregator {
    
    data class SalesSummary(
        val date: java.time.LocalDate,
        val totalRevenue: java.math.BigDecimal,
        val orderCount: Int,
        val avgOrderValue: java.math.BigDecimal,
        val topProducts: List<ProductSales>,
        val revenueByCategory: Map<String, java.math.BigDecimal>
    )
    
    data class ProductSales(val productId: String, val revenue: java.math.BigDecimal, val quantity: Int)
    
    fun aggregate(orders: List<CleanOrderData>, date: java.time.LocalDate): SalesSummary {
        val filteredOrders = orders.filter { it.date == date }
        
        val totalRevenue = filteredOrders.sumOf { it.amount }
        val orderCount = filteredOrders.size
        val avgOrderValue = if (orderCount > 0) 
            totalRevenue.divide(orderCount.toBigDecimal(), 2, java.math.RoundingMode.HALF_UP)
        else java.math.BigDecimal.ZERO
        
        val productSales = filteredOrders
            .flatMap { it.items }
            .groupBy { it.productId }
            .map { (productId, items) ->
                ProductSales(
                    productId = productId,
                    revenue = items.sumOf { it.price * it.quantity.toBigDecimal() },
                    quantity = items.sumOf { it.quantity }
                )
            }
            .sortedByDescending { it.revenue }
        
        return SalesSummary(
            date = date,
            totalRevenue = totalRevenue,
            orderCount = orderCount,
            avgOrderValue = avgOrderValue,
            topProducts = productSales.take(10),
            revenueByCategory = emptyMap()  // populate from product category lookup
        )
    }
}
```

---

## Scheduler & Monitoring

```kotlin
// BatchScheduler.kt
import org.springframework.batch.core.JobParameters
import org.springframework.batch.core.JobParametersBuilder
import org.springframework.batch.core.launch.JobLauncher
import org.springframework.scheduling.annotation.EnableScheduling
import org.springframework.scheduling.annotation.Scheduled

@org.springframework.stereotype.Component
@EnableScheduling
class BatchScheduler(
    private val jobLauncher: JobLauncher,
    private val dailySalesReportJob: Job,
    private val customerDataSyncJob: Job,
    private val meterRegistry: io.micrometer.core.instrument.MeterRegistry
) {
    private val logger = org.slf4j.LoggerFactory.getLogger(javaClass)
    
    // Run daily at 2 AM
    @Scheduled(cron = "0 0 2 * * *", zone = "Asia/Bangkok")
    fun runDailySalesReport() {
        val reportDate = java.time.LocalDate.now().minusDays(1).toString()
        runJob(dailySalesReportJob, mapOf(
            "reportDate" to reportDate,
            "runTime" to System.currentTimeMillis().toString()
        ))
    }
    
    // Run every 4 hours
    @Scheduled(fixedDelay = 4 * 60 * 60 * 1000)
    fun runCustomerDataSync() {
        runJob(customerDataSyncJob, mapOf(
            "syncTime" to System.currentTimeMillis().toString()
        ))
    }
    
    private fun runJob(job: Job, params: Map<String, String>) {
        val timer = meterRegistry.timer("batch.job.duration", "job", job.name)
        
        timer.record {
            try {
                val jobParams = JobParametersBuilder()
                    .apply { params.forEach { (k, v) -> addString(k, v) } }
                    .toJobParameters()
                
                val execution = jobLauncher.run(job, jobParams)
                
                meterRegistry.counter(
                    "batch.job.executions",
                    "job", job.name,
                    "status", execution.status.name
                ).increment()
                
                logger.info("Job ${job.name} completed: ${execution.status}")
            } catch (e: Exception) {
                logger.error("Failed to start job ${job.name}", e)
                meterRegistry.counter("batch.job.errors", "job", job.name).increment()
            }
        }
    }
}

// Job monitoring API
@org.springframework.web.bind.annotation.RestController
@org.springframework.web.bind.annotation.RequestMapping("/api/v1/admin/batch")
@org.springframework.security.access.prepost.PreAuthorize("hasRole('ADMIN')")
class BatchMonitoringController(
    private val jobExplorer: org.springframework.batch.core.explore.JobExplorer,
    private val jobOperator: org.springframework.batch.core.launch.JobOperator,
    private val jobLauncher: JobLauncher,
    private val dailySalesReportJob: Job
) {
    
    @org.springframework.web.bind.annotation.GetMapping("/jobs")
    fun listJobs(): List<String> {
        return jobExplorer.jobNames
    }
    
    @org.springframework.web.bind.annotation.GetMapping("/jobs/{name}/executions")
    fun getJobExecutions(
        @org.springframework.web.bind.annotation.PathVariable name: String
    ): List<JobExecutionSummary> {
        return jobExplorer.getJobInstances(name, 0, 10)
            .flatMap { instance -> jobExplorer.getJobExecutions(instance) }
            .map { execution ->
                JobExecutionSummary(
                    id = execution.id ?: 0L,
                    jobName = name,
                    status = execution.status.name,
                    startTime = execution.startTime?.toString() ?: "",
                    endTime = execution.endTime?.toString() ?: "",
                    readCount = execution.stepExecutions.sumOf { it.readCount },
                    writeCount = execution.stepExecutions.sumOf { it.writeCount },
                    skipCount = execution.stepExecutions.sumOf { it.skipCount },
                    failureExceptions = execution.allFailureExceptions.map { it.message ?: "" }
                )
            }
    }
    
    @org.springframework.web.bind.annotation.PostMapping("/jobs/{name}/restart")
    fun restartJob(@org.springframework.web.bind.annotation.PathVariable name: String): Map<String, Any> {
        val lastExecution = jobExplorer.getJobInstances(name, 0, 1)
            .firstOrNull()
            ?.let { jobExplorer.getLastJobExecution(it) }
            ?: return mapOf("error" to "No previous execution found")
        
        return if (lastExecution.status == BatchStatus.FAILED) {
            val executionId = jobOperator.restart(lastExecution.id!!)
            mapOf("restarted" to true, "executionId" to executionId)
        } else {
            mapOf("error" to "Job is not in FAILED state: ${lastExecution.status}")
        }
    }
    
    @org.springframework.web.bind.annotation.PostMapping("/jobs/daily-sales/trigger")
    fun triggerDailySalesReport(
        @org.springframework.web.bind.annotation.RequestParam date: String
    ): Map<String, Any> {
        val params = JobParametersBuilder()
            .addString("reportDate", date)
            .addLong("runTime", System.currentTimeMillis())
            .toJobParameters()
        
        val execution = jobLauncher.run(dailySalesReportJob, params)
        return mapOf(
            "executionId" to (execution.id ?: 0L),
            "status" to execution.status.name
        )
    }
}

data class JobExecutionSummary(
    val id: Long,
    val jobName: String,
    val status: String,
    val startTime: String,
    val endTime: String,
    val readCount: Int,
    val writeCount: Int,
    val skipCount: Int,
    val failureExceptions: List<String>
)
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง Product Recommendation ETL Pipeline

// Requirements:
// 1. Read order history จาก PostgreSQL (last 30 days)
// 2. Calculate co-purchase matrix: products often bought together
// 3. Calculate user similarity matrix (collaborative filtering)
// 4. Write recommendations table: user_id, product_id, score, reason
// 5. Schedule ทุกคืน เวลา 01:00
// 6. Skip users ที่มี < 3 orders
// 7. Report: จำนวน users, products, recommendations generated

@org.springframework.stereotype.Component
class RecommendationProcessor(
    private val orderRepository: OrderRepository
) : ItemProcessor<UserOrderHistory, List<Recommendation>> {
    
    override fun process(userHistory: UserOrderHistory): List<Recommendation>? {
        if (userHistory.orders.size < 3) return null
        
        val purchasedProducts = userHistory.orders.flatMap { it.productIds }.toSet()
        val coPurchases = calculateCoPurchases(userHistory.orders)
        
        // สินค้าที่ถูก co-purchase กับสินค้าที่ user ซื้อแล้ว
        val candidateProducts = coPurchases
            .filter { (productPair, _) -> productPair.first in purchasedProducts }
            .map { (productPair, count) -> productPair.second to count }
            .filter { (productId, _) -> productId !in purchasedProducts }
            .groupBy { (productId, _) -> productId }
            .mapValues { (_, scores) -> scores.sumOf { (_, count) -> count } }
            .entries.sortedByDescending { it.value }
            .take(10)
        
        return candidateProducts.map { (productId, score) ->
            Recommendation(
                userId = userHistory.userId,
                productId = productId,
                score = score.toDouble() / 100.0,
                reason = "Co-purchase"
            )
        }
    }
    
    private fun calculateCoPurchases(orders: List<OrderHistory>): Map<Pair<String, String>, Int> {
        return orders.flatMap { order ->
            val products = order.productIds
            products.flatMap { p1 ->
                products.filter { it != p1 }.map { p2 -> (p1 to p2) }
            }
        }.groupingBy { it }.eachCount()
    }
}

data class UserOrderHistory(
    val userId: String,
    val orders: List<OrderHistory>
)

data class OrderHistory(val orderId: String, val productIds: List<String>, val date: java.time.LocalDate)

data class Recommendation(
    val userId: String,
    val productId: String,
    val score: Double,
    val reason: String
)
```

---

## สรุป Part 84

```
✅ Spring Batch: Job, Step, Reader, Processor, Writer
✅ @EnableBatchProcessing: auto-configure batch infrastructure
✅ JobBuilder: build jobs with steps, conditions (on/to)
✅ StepBuilder.chunk(): read-process-write with commit interval
✅ faultTolerant(): retry, skip with limits
✅ JpaPagingItemReader: paginated JPA query reader
✅ FlatFileItemReader: CSV with header skip
✅ @StepScope + @Value("#{jobParameters[...]}"): runtime parameter injection
✅ ItemProcessor: return null to skip item
✅ Deduplication: processedSkus set within processor
✅ JpaItemWriter: persist entities to database
✅ CompositeItemWriter: write to multiple destinations (DB + Kafka)
✅ FlatFileItemWriter: CSV export with headerCallback
✅ JobExecutionListener: timing, alerting, metrics
✅ StepExecutionListener: per-step read/write/skip counts
✅ Partitioner: divide data for parallel step execution
✅ gridSize + ThreadPoolTaskExecutor: parallelism
✅ Either (Arrow): functional error handling in transformation
✅ DataQualityChecker: null, duplicate, outlier detection
✅ qualityScore: reject batch if quality < threshold
✅ SalesAggregator: revenue, count, avg, top products
✅ @Scheduled(cron): timezone-aware cron scheduling
✅ Micrometer metrics: batch job duration/executions/errors
✅ BatchMonitoringController: list, restart, trigger jobs
✅ Co-purchase matrix: collaborative filtering foundation
```

---

*Part 84/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
