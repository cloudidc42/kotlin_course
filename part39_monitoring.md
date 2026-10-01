# Part 39: Monitoring และ Observability ด้วย Kotlin

## สารบัญ
1. [Metrics ด้วย Micrometer](#metrics-ด้วย-micrometer)
2. [Distributed Tracing](#distributed-tracing)
3. [Structured Logging](#structured-logging)
4. [Alerting](#alerting)
5. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Metrics ด้วย Micrometer

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.micrometer:micrometer-core")
    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("org.springframework.boot:spring-boot-starter-actuator")
}
```

```kotlin
import io.micrometer.core.instrument.*
import io.micrometer.core.instrument.binder.MeterBinder
import org.springframework.stereotype.Component
import java.util.concurrent.atomic.AtomicInteger
import java.util.concurrent.atomic.AtomicLong

// Custom metrics for business operations
@Component
class OrderMetrics(private val registry: MeterRegistry) : MeterBinder {
    
    private val orderCounter: Counter by lazy {
        Counter.builder("orders.created")
            .description("Total orders created")
            .tag("app", "kotlin-shop")
            .register(registry)
    }
    
    private val orderErrorCounter: Counter by lazy {
        Counter.builder("orders.errors")
            .description("Total order creation errors")
            .register(registry)
    }
    
    private val activeOrders: AtomicInteger = AtomicInteger(0)
    
    private val orderRevenue: DistributionSummary by lazy {
        DistributionSummary.builder("orders.revenue")
            .description("Order revenue distribution")
            .baseUnit("baht")
            .publishPercentiles(0.5, 0.95, 0.99)
            .publishPercentileHistogram()
            .register(registry)
    }
    
    override fun bindTo(registry: MeterRegistry) {
        Gauge.builder("orders.active", activeOrders, AtomicInteger::get)
            .description("Currently active (unfulfilled) orders")
            .register(registry)
    }
    
    fun recordOrderCreated(amount: Double) {
        orderCounter.increment()
        orderRevenue.record(amount)
        activeOrders.incrementAndGet()
    }
    
    fun recordOrderError(errorType: String) {
        registry.counter("orders.errors", "type", errorType).increment()
    }
    
    fun recordOrderFulfilled() {
        activeOrders.decrementAndGet()
    }
}

// Timer for measuring operation duration
@Component
class TimedOperationService(
    private val registry: MeterRegistry,
    private val orderRepository: OrderRepository
) {
    
    fun findOrderById(id: String) = registry.timer("db.query.orders")
        .record<Order?> {
            orderRepository.findById(id)
        }
    
    // Using @Timed annotation (Spring AOP)
    // @Timed(value = "db.query.orders", percentiles = [0.5, 0.95, 0.99])
    
    fun processOrder(order: Order) {
        val timer = Timer.start(registry)
        try {
            // process...
            timer.stop(registry.timer("orders.processing.success"))
        } catch (e: Exception) {
            timer.stop(registry.timer("orders.processing.error", 
                "exception", e.javaClass.simpleName))
            throw e
        }
    }
}

// Custom metrics endpoint
@Component
class SystemMetrics(
    private val registry: MeterRegistry,
    private val dataSource: javax.sql.DataSource
) : MeterBinder {
    
    override fun bindTo(registry: MeterRegistry) {
        // Connection pool metrics
        Gauge.builder("db.pool.active") {
            try {
                (dataSource as com.zaxxer.hikari.HikariDataSource)
                    .hikariPoolMXBean?.activeConnections?.toDouble() ?: 0.0
            } catch (e: Exception) { 0.0 }
        }.register(registry)
        
        Gauge.builder("db.pool.idle") {
            try {
                (dataSource as com.zaxxer.hikari.HikariDataSource)
                    .hikariPoolMXBean?.idleConnections?.toDouble() ?: 0.0
            } catch (e: Exception) { 0.0 }
        }.register(registry)
        
        // JVM metrics are added automatically by MicrometerAutoConfiguration
        // - jvm.memory.used, jvm.gc.pause, jvm.threads.live etc.
    }
}
```

---

## Distributed Tracing ด้วย OpenTelemetry

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.micrometer:micrometer-tracing-bridge-otel")
    implementation("io.opentelemetry:opentelemetry-exporter-otlp")
}

// application.yml
/*
management:
  tracing:
    sampling:
      probability: 1.0  # 100% in dev, 0.1 in prod
  otlp:
    tracing:
      endpoint: http://jaeger:4317
*/
```

```kotlin
import io.micrometer.tracing.Tracer
import io.micrometer.tracing.annotation.NewSpan
import io.micrometer.tracing.annotation.SpanTag
import org.springframework.stereotype.Service

@Service
class OrderService(
    private val tracer: Tracer,
    private val orderRepository: OrderRepository,
    private val paymentService: PaymentService,
    private val inventoryService: InventoryService
) {
    
    // Automatic span creation with annotation
    @NewSpan("create-order")
    fun createOrder(
        @SpanTag("customer.id") customerId: String,
        items: List<OrderItem>
    ): Order {
        val span = tracer.currentSpan()
        span?.tag("items.count", items.size.toString())
        
        try {
            val order = orderRepository.save(Order(customerId = customerId, items = items))
            span?.tag("order.id", order.id)
            return order
        } catch (e: Exception) {
            span?.error(e)
            throw e
        }
    }
    
    // Manual span creation for fine-grained tracing
    fun processOrder(orderId: String): ProcessResult {
        val span = tracer.nextSpan()
            .name("process-order")
            .tag("order.id", orderId)
            .start()
        
        return tracer.withSpan(span).use {
            try {
                // Child span for payment
                val paymentSpan = tracer.nextSpan().name("process-payment").start()
                val paymentResult = tracer.withSpan(paymentSpan).use {
                    paymentService.charge(orderId)
                }
                
                // Child span for inventory
                val inventorySpan = tracer.nextSpan().name("update-inventory").start()
                val inventoryResult = tracer.withSpan(inventorySpan).use {
                    inventoryService.reserve(orderId)
                }
                
                span.tag("payment.status", paymentResult.status.name)
                span.tag("inventory.status", inventoryResult.status.name)
                
                ProcessResult(payment = paymentResult, inventory = inventoryResult)
            } catch (e: Exception) {
                span.error(e)
                throw e
            } finally {
                span.end()
            }
        }
    }
}

// Propagate trace context in async operations
@Service
class AsyncOrderService(
    private val tracer: Tracer
) {
    fun processAsync(order: Order) {
        val traceContext = tracer.currentTraceContext().context()
        
        kotlinx.coroutines.GlobalScope.launch {
            // Restore trace context in coroutine
            tracer.currentTraceContext().newScope(traceContext).use {
                processInBackground(order)
            }
        }
    }
    
    private suspend fun processInBackground(order: Order) {
        val span = tracer.nextSpan()
            .name("background-processing")
            .tag("order.id", order.id)
            .start()
        
        try {
            // process...
        } finally {
            span.end()
        }
    }
}
```

---

## Structured Logging

```kotlin
// build.gradle.kts
dependencies {
    implementation("net.logstash.logback:logstash-logback-encoder:7.4")
}

// logback-spring.xml
/*
<configuration>
    <springProfile name="production">
        <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
        </appender>
        <root level="INFO">
            <appender-ref ref="JSON"/>
        </root>
    </springProfile>
    
    <springProfile name="!production">
        <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>
</configuration>
*/
```

```kotlin
import org.slf4j.LoggerFactory
import org.slf4j.MDC
import net.logstash.logback.argument.StructuredArguments.kv

// Structured logging with context
@Component
class OrderLogger {
    private val log = LoggerFactory.getLogger(OrderLogger::class.java)
    
    fun logOrderCreated(order: Order) {
        log.info("Order created",
            kv("orderId", order.id),
            kv("customerId", order.customerId),
            kv("amount", order.total),
            kv("itemCount", order.items.size),
            kv("event", "ORDER_CREATED")
        )
    }
    
    fun logOrderFailed(orderId: String, reason: String, exception: Exception? = null) {
        if (exception != null) {
            log.error("Order processing failed",
                kv("orderId", orderId),
                kv("reason", reason),
                kv("event", "ORDER_FAILED"),
                exception
            )
        } else {
            log.warn("Order processing failed",
                kv("orderId", orderId),
                kv("reason", reason),
                kv("event", "ORDER_FAILED")
            )
        }
    }
}

// MDC (Mapped Diagnostic Context) for request context
@Component
class RequestLoggingFilter : javax.servlet.Filter {
    
    override fun doFilter(
        request: javax.servlet.ServletRequest,
        response: javax.servlet.ServletResponse,
        chain: javax.servlet.FilterChain
    ) {
        val httpRequest = request as javax.servlet.http.HttpServletRequest
        
        val requestId = httpRequest.getHeader("X-Request-ID") 
            ?: java.util.UUID.randomUUID().toString()
        val userId = httpRequest.getHeader("X-User-ID") ?: "anonymous"
        
        MDC.put("requestId", requestId)
        MDC.put("userId", userId)
        MDC.put("path", httpRequest.requestURI)
        MDC.put("method", httpRequest.method)
        
        (response as javax.servlet.http.HttpServletResponse)
            .setHeader("X-Request-ID", requestId)
        
        try {
            chain.doFilter(request, response)
        } finally {
            MDC.clear()
        }
    }
}

// Coroutine-aware MDC propagation
suspend fun <T> withMdc(context: Map<String, String>, block: suspend () -> T): T {
    val previous = MDC.getCopyOfContextMap() ?: emptyMap()
    MDC.setContextMap(previous + context)
    return try {
        block()
    } finally {
        MDC.setContextMap(previous)
    }
}

// Kotlin logging extension
inline fun <reified T> logger() = LoggerFactory.getLogger(T::class.java)

// Usage:
// private val log = logger<MyService>()
```

---

## Prometheus + Grafana Dashboard

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'kotlin-app'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['kotlin-app:8080']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
```

```kotlin
// Prometheus alert rules
/*
# alerts.yml
groups:
  - name: kotlin-app
    rules:
      - alert: HighErrorRate
        expr: rate(orders_errors_total[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High order error rate"
          description: "Error rate {{ $value }} over last 5 minutes"
      
      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m])) > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High 95th percentile response time"
          description: "95th percentile is {{ $value }}s"
      
      - alert: DatabaseConnectionPoolExhausted
        expr: db_pool_active / (db_pool_active + db_pool_idle) > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Database connection pool nearly exhausted"
*/
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement complete observability for a payment service

@Service
class PaymentService(
    private val registry: MeterRegistry,
    private val tracer: Tracer
) {
    private val log = logger<PaymentService>()
    
    // TODO: Add metrics:
    // - payments.attempted: counter with tags: provider, currency
    // - payments.succeeded: counter
    // - payments.failed: counter with tag: reason
    // - payments.processing.time: timer
    // - payments.amount: distribution summary
    
    // TODO: Add tracing:
    // - span "process-payment" with tags: provider, amount, currency
    // - child span "validate-card" 
    // - child span "charge-card"
    
    // TODO: Add structured logging:
    // - log payment attempt with amount, currency, provider
    // - log success with transactionId
    // - log failure with reason (but NOT card details!)
    
    fun processPayment(request: PaymentRequest): PaymentResult {
        // Implement with observability
        TODO("Add metrics + tracing + logging")
    }
}

data class PaymentRequest(
    val amount: Double,
    val currency: String,
    val provider: String,
    val cardToken: String  // tokenized card, not actual card number
)

data class PaymentResult(
    val transactionId: String,
    val status: PaymentStatus,
    val message: String
)

enum class PaymentStatus { SUCCESS, FAILED, PENDING }

interface OrderRepository {
    fun findById(id: String): Order?
    fun save(order: Order): Order
}

interface PaymentService2 {
    fun charge(orderId: String): Any
}

interface InventoryService {
    fun reserve(orderId: String): Any
}

data class Order(val id: String = "", val customerId: String = "", val items: List<OrderItem> = emptyList(), val total: Double = 0.0)
data class OrderItem(val productId: String, val quantity: Int, val price: Double)
data class ProcessResult(val payment: Any, val inventory: Any)
```

---

## สรุป Part 39

```
✅ Micrometer: unified metrics API across backends
✅ Counter, Timer, Gauge, DistributionSummary
✅ Prometheus: pull-based metrics scraping
✅ Grafana: visualization and dashboards
✅ Distributed Tracing: OpenTelemetry + Jaeger/Zipkin
✅ @NewSpan + @SpanTag: annotation-driven tracing
✅ Manual spans: fine-grained control
✅ Trace context propagation: in coroutines and threads
✅ MDC: per-request log context (requestId, userId)
✅ Structured logging: JSON format for log aggregation
✅ Alerting: Prometheus rules for proactive monitoring
✅ Three pillars of observability: Metrics + Traces + Logs
```

---

*Part 39/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
