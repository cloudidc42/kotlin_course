# Part 64: Observability — Metrics, Tracing, Logging

## สารบัญ
1. [Metrics ด้วย Micrometer](#metrics-ด้วย-micrometer)
2. [Distributed Tracing ด้วย OpenTelemetry](#distributed-tracing-ด้วย-opentelemetry)
3. [Structured Logging](#structured-logging)
4. [Prometheus และ Grafana](#prometheus-และ-grafana)
5. [Alerting](#alerting)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Metrics ด้วย Micrometer

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-core")
    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("io.micrometer:micrometer-tracing-bridge-otel")
    implementation("io.opentelemetry:opentelemetry-exporter-otlp:1.41.0")
}

// Custom Metrics
@Component
class OrderMetrics(private val meterRegistry: MeterRegistry) {
    
    // Counter: จำนวนเหตุการณ์ (เพิ่มขึ้นเท่านั้น)
    private val ordersCreated = Counter.builder("orders.created")
        .description("Total number of orders created")
        .tag("environment", "production")
        .register(meterRegistry)
    
    private val ordersFailed = meterRegistry.counter(
        "orders.failed",
        "environment", "production"
    )
    
    // Timer: วัดเวลาและจำนวนการเรียก
    private val orderProcessingTimer = Timer.builder("orders.processing.duration")
        .description("Time taken to process an order")
        .publishPercentiles(0.5, 0.95, 0.99)
        .publishPercentileHistogram()
        .register(meterRegistry)
    
    // Gauge: current value (สามารถเพิ่มหรือลดได้)
    private var activeOrders = AtomicInteger(0)
    
    init {
        Gauge.builder("orders.active", activeOrders) { it.get().toDouble() }
            .description("Currently active orders")
            .register(meterRegistry)
    }
    
    // Summary/DistributionSummary: distribution of values
    private val orderValueSummary = DistributionSummary.builder("orders.value")
        .description("Distribution of order values in THB")
        .baseUnit("THB")
        .publishPercentiles(0.5, 0.9, 0.95, 0.99)
        .scale(1.0)
        .register(meterRegistry)
    
    // Business methods
    fun recordOrderCreated(orderValue: Double) {
        ordersCreated.increment()
        activeOrders.incrementAndGet()
        orderValueSummary.record(orderValue)
    }
    
    fun recordOrderCompleted(duration: Long) {
        activeOrders.decrementAndGet()
        orderProcessingTimer.record(duration, java.util.concurrent.TimeUnit.MILLISECONDS)
    }
    
    fun recordOrderFailed() {
        ordersFailed.increment()
        activeOrders.decrementAndGet()
    }
    
    // Time a block of code
    fun <T> timeOperation(operationName: String, block: () -> T): T {
        return Timer.builder("operation.duration")
            .tag("operation", operationName)
            .register(meterRegistry)
            .record(block)!!
    }
}

typealias MeterRegistry = io.micrometer.core.instrument.MeterRegistry
typealias Counter = io.micrometer.core.instrument.Counter
typealias Timer = io.micrometer.core.instrument.Timer
typealias Gauge = io.micrometer.core.instrument.Gauge
typealias DistributionSummary = io.micrometer.core.instrument.DistributionSummary
typealias AtomicInteger = java.util.concurrent.atomic.AtomicInteger

// Tagged metrics (multi-dimensional)
@Service
class PaymentMetricsService(private val meterRegistry: MeterRegistry) {
    
    fun recordPayment(method: String, currency: String, amount: Double, success: Boolean) {
        val tags = Tags.of(
            "payment_method", method,
            "currency", currency,
            "success", success.toString()
        )
        
        Counter.builder("payments.total")
            .tags(tags)
            .register(meterRegistry)
            .increment()
        
        if (success) {
            DistributionSummary.builder("payments.amount")
                .tags(tags)
                .register(meterRegistry)
                .record(amount)
        }
    }
    
    // Long task timer: for operations that span multiple requests
    fun startLongOrderProcessing(orderId: String): LongTaskTimer.Sample {
        val timer = LongTaskTimer.builder("orders.processing.long")
            .tag("orderId", orderId)
            .register(meterRegistry)
        return timer.start()
    }
}

typealias Tags = io.micrometer.core.instrument.Tags
typealias LongTaskTimer = io.micrometer.core.instrument.LongTaskTimer
```

---

## Distributed Tracing ด้วย OpenTelemetry

```kotlin
// application.yml configuration:
// management:
//   tracing:
//     sampling:
//       probability: 1.0  # 100% in dev, use 0.1 in production
// spring:
//   application:
//     name: order-service

// Using Micrometer Tracing
@Service
class OrderTracingService(
    private val tracer: Tracer,
    private val orderRepository: TracingOrderRepository
) {
    
    suspend fun processOrder(orderId: String): ProcessedOrderResult {
        // Create a new span
        val span = tracer.nextSpan()
            .name("order.process")
            .tag("order.id", orderId)
            .start()
        
        return tracer.withSpan(span).use {
            try {
                // Add events to span
                span.event("order.processing.started")
                
                val order = findOrder(orderId, span)
                val result = executeOrder(order, span)
                
                span.event("order.processing.completed")
                span.tag("order.status", result.status)
                
                result
            } catch (ex: Exception) {
                span.tag("error", ex.message ?: "Unknown error")
                span.event("order.processing.failed")
                throw ex
            } finally {
                span.end()
            }
        }
    }
    
    private suspend fun findOrder(orderId: String, parentSpan: Span): TracingOrder {
        val childSpan = tracer.nextSpan(parentSpan)
            .name("order.find")
            .tag("db.operation", "SELECT")
            .start()
        
        return try {
            orderRepository.findById(orderId) ?: throw NotFoundException("Order $orderId")
        } finally {
            childSpan.end()
        }
    }
    
    private suspend fun executeOrder(order: TracingOrder, parentSpan: Span): ProcessedOrderResult {
        val childSpan = tracer.nextSpan(parentSpan)
            .name("order.execute")
            .start()
        
        return try {
            ProcessedOrderResult(order.id, "COMPLETED")
        } finally {
            childSpan.end()
        }
    }
}

data class TracingOrder(val id: String, val status: String)
data class ProcessedOrderResult(val orderId: String, val status: String)

interface TracingOrderRepository {
    suspend fun findById(id: String): TracingOrder?
}

typealias Tracer = io.micrometer.tracing.Tracer
typealias Span = io.micrometer.tracing.Span

// Propagation via HTTP Headers (automatic with Spring WebMVC/WebFlux)
@RestController
@RequestMapping("/api/orders")
class TracedOrderController(
    private val orderService: OrderTracingService
) {
    
    @GetMapping("/{id}")
    suspend fun getOrder(@PathVariable id: String): ProcessedOrderResult {
        // traceId and spanId automatically propagated via:
        // - W3C TraceContext headers (traceparent, tracestate)
        // - B3 headers (X-B3-TraceId, X-B3-SpanId)
        return orderService.processOrder(id)
    }
}

// Context propagation for async/Kafka
@Component
class TracingKafkaProducer(
    private val kafkaTemplate: KafkaTemplate<String, String>,
    private val propagator: Propagator
) {
    
    fun sendWithTrace(topic: String, key: String, value: String) {
        val span = Tracer::class.java  // get from context
        
        val headers = ProducerRecord<String, String>(topic, key, value).headers()
        
        // Inject trace context into Kafka headers
        propagator.inject(
            io.micrometer.tracing.TraceContext.empty(),
            headers,
            { carrier, k, v -> carrier.add(org.apache.kafka.common.header.internals.RecordHeader(k, v.toByteArray())) }
        )
        
        kafkaTemplate.send(topic, key, value)
    }
}

typealias Propagator = io.micrometer.tracing.propagation.Propagator
typealias KafkaTemplate<K, V> = org.springframework.kafka.core.KafkaTemplate<K, V>
typealias ProducerRecord<K, V> = org.apache.kafka.clients.producer.ProducerRecord<K, V>
```

---

## Structured Logging

```kotlin
// Structured logging with Logback + Logstash Encoder
// logback-spring.xml:
// <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>

@Component
class StructuredLogger {
    
    private val log = LoggerFactory.getLogger(this::class.java)
    
    // Log with structured fields using MDC
    fun logOrderCreated(orderId: String, userId: String, amount: Double) {
        MDC.put("orderId", orderId)
        MDC.put("userId", userId)
        MDC.put("amount", amount.toString())
        
        try {
            log.info("Order created")
            // JSON output: {"timestamp":"...","level":"INFO","message":"Order created",
            //               "orderId":"abc","userId":"user-1","amount":"99.99","traceId":"..."}
        } finally {
            MDC.remove("orderId")
            MDC.remove("userId")
            MDC.remove("amount")
        }
    }
    
    // Kotlin extension: cleaner MDC usage
    fun withContext(vararg pairs: Pair<String, String>, block: () -> Unit) {
        pairs.forEach { (k, v) -> MDC.put(k, v) }
        try { block() }
        finally { pairs.forEach { (k, _) -> MDC.remove(k) } }
    }
}

// Coroutine-safe MDC propagation
// MDC is thread-local, doesn't work well with coroutines by default
// Use MDCContext from coroutines:
import kotlinx.coroutines.slf4j.MDCContext

suspend fun logInCoroutine() {
    MDC.put("requestId", "req-123")
    
    // Without MDCContext, MDC is lost after suspension
    withContext(MDCContext()) {
        // MDC is now propagated across coroutine suspensions
        delay(100)
        log.info("This has requestId in MDC")  // Works!
    }
}

private val log = LoggerFactory.getLogger("example")

typealias LoggerFactory = org.slf4j.LoggerFactory
typealias MDC = org.slf4j.MDC

// Request logging filter
@Component
class RequestLoggingFilter : OncePerRequestFilter() {
    
    private val log = LoggerFactory.getLogger(this::class.java)
    
    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val requestId = request.getHeader("X-Request-ID") ?: java.util.UUID.randomUUID().toString()
        val startTime = System.currentTimeMillis()
        
        MDC.put("requestId", requestId)
        MDC.put("method", request.method)
        MDC.put("path", request.requestURI)
        MDC.put("clientIp", request.remoteAddr)
        
        try {
            filterChain.doFilter(request, response)
            
            val duration = System.currentTimeMillis() - startTime
            MDC.put("status", response.status.toString())
            MDC.put("duration", "${duration}ms")
            
            if (duration > 5000) {
                log.warn("Slow request detected")
            } else {
                log.info("Request completed")
            }
        } finally {
            MDC.clear()
        }
    }
}

typealias OncePerRequestFilter = org.springframework.web.filter.OncePerRequestFilter
typealias FilterChain = jakarta.servlet.FilterChain
typealias HttpServletRequest = jakarta.servlet.http.HttpServletRequest
typealias HttpServletResponse = jakarta.servlet.http.HttpServletResponse
```

---

## Prometheus และ Grafana

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'kotlin-services'
    metrics_path: '/actuator/prometheus'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - production
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
```

```promql
# PromQL queries สำหรับ Grafana dashboards

# Request rate (requests per second)
rate(http_server_requests_seconds_count{job="kotlin-service"}[5m])

# Error rate (%)
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
  /
sum(rate(http_server_requests_seconds_count[5m])) * 100

# p95 response time
histogram_quantile(0.95,
  sum(rate(http_server_requests_seconds_bucket{job="kotlin-service"}[5m]))
  by (le, uri)
)

# JVM heap usage
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100

# Database connection pool
hikaricp_connections_active / hikaricp_connections_max * 100

# Kafka consumer lag
kafka_consumer_fetch_manager_records_lag{group="kotlin-service"}

# Order throughput
rate(orders_created_total[5m])

# Active orders
orders_active

# Custom business metrics
sum(rate(payments_total{success="true"}[5m])) by (payment_method)
```

---

## Alerting

```yaml
# Prometheus alert rules
groups:
  - name: kotlin-service-alerts
    rules:
      
      # High error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5..", job="kotlin-service"}[5m]))
          /
          sum(rate(http_server_requests_seconds_count{job="kotlin-service"}[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High HTTP error rate on {{ $labels.job }}"
          description: "Error rate is {{ $value | humanizePercentage }} for the past 5 minutes"
          runbook: "https://runbooks.company.com/high-error-rate"
      
      # High latency
      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            sum(rate(http_server_requests_seconds_bucket{job="kotlin-service"}[5m]))
            by (le)
          ) > 2.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High latency on {{ $labels.job }}"
          description: "p95 latency is {{ $value }}s"
      
      # Pod down
      - alert: PodNotReady
        expr: kube_pod_status_ready{namespace="production", condition="true"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} is not ready"
      
      # Kafka consumer lag
      - alert: HighKafkaLag
        expr: kafka_consumer_fetch_manager_records_lag > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Kafka consumer lag for {{ $labels.group }}"
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง custom business metrics dashboard

@Component
class EcommerceMetrics(private val meterRegistry: MeterRegistry) {
    
    // TODO: Implement these metrics:
    
    // 1. Revenue per minute (DistributionSummary)
    // 2. Conversion rate: orders created / product views (%)
    // 3. Cart abandonment rate: carts abandoned / carts created (%)
    // 4. Average order value by category
    // 5. Inventory alerts: products with stock < 10
    
    private val productViews = Counter.builder("ecommerce.product.views")
        .register(meterRegistry)
    
    private val cartsCreated = Counter.builder("ecommerce.carts.created")
        .register(meterRegistry)
    
    private val cartsAbandoned = Counter.builder("ecommerce.carts.abandoned")
        .register(meterRegistry)
    
    // TODO: Add more counters, gauges, and timers
    
    fun recordProductView(productId: String, category: String) {
        Counter.builder("ecommerce.product.views")
            .tag("productId", productId)
            .tag("category", category)
            .register(meterRegistry)
            .increment()
    }
    
    fun recordCartCreated() {
        cartsCreated.increment()
    }
    
    fun recordCartAbandoned() {
        cartsAbandoned.increment()
    }
    
    fun recordRevenue(amount: Double, currency: String, category: String) {
        // TODO: Record revenue with tags
    }
    
    fun setInventoryLevel(productId: String, stock: Int) {
        // TODO: Update gauge for this productId
    }
}

// PromQL for this dashboard:
// Conversion rate: rate(orders_created_total[5m]) / rate(ecommerce_product_views_total[5m]) * 100
// Cart abandonment: rate(ecommerce_carts_abandoned_total[5m]) / rate(ecommerce_carts_created_total[5m]) * 100
// Revenue per category: sum(rate(ecommerce_revenue_total[1h])) by (category)
```

---

## สรุป Part 64

```
✅ Micrometer: vendor-neutral metrics facade
✅ Counter: จำนวนเหตุการณ์ (เพิ่มขึ้นเท่านั้น)
✅ Timer: วัดเวลาและ percentiles
✅ Gauge: current value (มีขึ้นลง)
✅ DistributionSummary: distribution of recorded values
✅ LongTaskTimer: สำหรับ operations ที่ใช้เวลานาน
✅ Tags: multi-dimensional metrics (labels)
✅ Micrometer Tracing: distributed tracing facade
✅ Span: unit of work with name, tags, events
✅ Tracer.nextSpan(): create child span
✅ W3C TraceContext: traceparent header propagation
✅ MDC: Mapped Diagnostic Context for structured logging
✅ MDCContext: coroutine-safe MDC propagation
✅ LogstashEncoder: JSON structured logging
✅ RequestLoggingFilter: automatic request logging
✅ Prometheus scraping: /actuator/prometheus endpoint
✅ PromQL: query language for Prometheus metrics
✅ AlertManager: alert rules with severity and annotations
✅ Grafana dashboards: visualize with PromQL queries
```

---

*Part 64/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
