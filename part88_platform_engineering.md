# Part 88: Platform Engineering & Internal Developer Platform

## สารบัญ
1. [Platform Engineering คืออะไร](#platform-engineering-คืออะไร)
2. [Service Template](#service-template)
3. [Observability Stack](#observability-stack)
4. [Developer Portal](#developer-portal)
5. [Infrastructure as Code](#infrastructure-as-code)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Platform Engineering คืออะไร

```
Platform Engineering: สร้าง "Golden Path" สำหรับ developers

เป้าหมาย:
- ลด cognitive load ของ developers
- Standard tooling ที่ทุก team ใช้เหมือนกัน
- Self-service: developers provision infrastructure ได้เอง
- Paved road: ทำในแบบที่ถูกต้องได้ง่ายกว่าแบบผิด

Internal Developer Platform (IDP) ประกอบด้วย:
1. Service Templates: boilerplate ที่ถูก setup ถูกต้องแล้ว
2. CI/CD Pipelines: standard pipeline สำหรับทุก service
3. Observability: Metrics, Logs, Traces มาพร้อมกัน
4. Developer Portal: Backstage.io / ประตูกลาง
5. Infrastructure: Terraform modules ที่ผ่าน review แล้ว

DORA Metrics (วัด developer productivity):
- Deployment Frequency: deploy บ่อยแค่ไหน
- Lead Time for Changes: code → production ใช้เวลานานแค่ไหน
- Change Failure Rate: % of deploys ที่ทำให้ production fail
- Mean Time to Recover: recover จาก failure ได้เร็วแค่ไหน
```

---

## Service Template & Standardization

```kotlin
// Standard Application configuration (ทุก service ต้องมี)
// application.yml template
/*
spring:
  application:
    name: ${SERVICE_NAME:my-service}
  
  # Graceful shutdown
  lifecycle:
    timeout-per-shutdown-phase: 30s
  
  # Database
  datasource:
    url: ${DB_URL}
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: ${DB_POOL_SIZE:10}
      connection-timeout: 30000
  
  # Kafka
  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
    consumer:
      group-id: ${spring.application.name}
      auto-offset-reset: earliest

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
      probes:
        enabled: true
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
    tags:
      application: ${spring.application.name}
      environment: ${ENVIRONMENT:local}
      version: ${APP_VERSION:unknown}

server:
  port: ${PORT:8080}
  shutdown: graceful
  tomcat:
    threads:
      max: 200
    connection-timeout: 5000ms
    keep-alive-timeout: 75s
*/

// Standard health checks
@org.springframework.stereotype.Component
class ServiceHealthIndicator(
    private val dataSource: javax.sql.DataSource,
    private val redisTemplate: org.springframework.data.redis.core.StringRedisTemplate
) : org.springframework.boot.actuate.health.HealthIndicator {
    
    override fun health(): org.springframework.boot.actuate.health.Health {
        val builder = org.springframework.boot.actuate.health.Health.Builder()
        
        // Check DB
        try {
            dataSource.connection.use { conn ->
                conn.prepareStatement("SELECT 1").execute()
            }
            builder.withDetail("database", "UP")
        } catch (e: Exception) {
            builder.down().withDetail("database", "DOWN: ${e.message}")
            return builder.build()
        }
        
        // Check Redis
        try {
            redisTemplate.execute { conn -> conn.ping() }
            builder.withDetail("redis", "UP")
        } catch (e: Exception) {
            builder.withDetail("redis", "DOWN: ${e.message}")
        }
        
        builder.up()
        return builder.build()
    }
}

// Standard startup probe
@org.springframework.stereotype.Component
class StartupHealthIndicator(
    private val applicationContext: org.springframework.context.ApplicationContext
) : org.springframework.boot.actuate.health.HealthIndicator {
    
    @Volatile private var ready = false
    
    @org.springframework.context.event.EventListener(org.springframework.context.event.ContextRefreshedEvent::class)
    fun onApplicationReady() {
        // Perform warmup tasks
        ready = true
    }
    
    override fun health() = if (ready)
        org.springframework.boot.actuate.health.Health.up().build()
    else
        org.springframework.boot.actuate.health.Health.down().withDetail("reason", "Warming up").build()
}

// Standard shutdown hook
@org.springframework.stereotype.Component
class GracefulShutdownHook(
    private val kafkaListenerEndpointRegistry: org.springframework.kafka.config.KafkaListenerEndpointRegistry
) : org.springframework.context.ApplicationListener<org.springframework.context.event.ContextClosedEvent> {
    
    private val logger = org.slf4j.LoggerFactory.getLogger(javaClass)
    
    override fun onApplicationEvent(event: org.springframework.context.event.ContextClosedEvent) {
        logger.info("Graceful shutdown initiated")
        
        // Stop accepting new Kafka messages
        kafkaListenerEndpointRegistry.stop()
        
        // Wait for in-flight requests to complete (handled by Spring shutdown)
        logger.info("Kafka listeners stopped, waiting for in-flight requests...")
    }
}
```

---

## Observability Stack

```kotlin
// Structured Logging (Logback + Loki)
// logback-spring.xml
/*
<configuration>
  <springProfile name="!local">
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdcKeyName>traceId</includeMdcKeyName>
        <includeMdcKeyName>spanId</includeMdcKeyName>
        <includeMdcKeyName>userId</includeMdcKeyName>
        <includeMdcKeyName>requestId</includeMdcKeyName>
        <customFields>{"service":"${SERVICE_NAME}","env":"${ENVIRONMENT}"}</customFields>
      </encoder>
    </appender>
    <root level="INFO">
      <appender-ref ref="CONSOLE"/>
    </root>
  </springProfile>
</configuration>
*/

// Structured logging helper
@org.springframework.stereotype.Component
class StructuredLogger {
    
    fun info(event: String, vararg fields: Pair<String, Any?>) {
        val logger = org.slf4j.LoggerFactory.getLogger("app")
        fields.forEach { (key, value) ->
            org.slf4j.MDC.put(key, value?.toString())
        }
        logger.info(event)
        fields.forEach { (key, _) -> org.slf4j.MDC.remove(key) }
    }
    
    fun error(event: String, throwable: Throwable, vararg fields: Pair<String, Any?>) {
        val logger = org.slf4j.LoggerFactory.getLogger("app")
        fields.forEach { (key, value) -> org.slf4j.MDC.put(key, value?.toString()) }
        logger.error(event, throwable)
        fields.forEach { (key, _) -> org.slf4j.MDC.remove(key) }
    }
}

// Custom Metrics
@org.springframework.stereotype.Component
class BusinessMetrics(private val meterRegistry: io.micrometer.core.instrument.MeterRegistry) {
    
    private val ordersPlaced = meterRegistry.counter("business.orders.placed")
    private val orderRevenue = meterRegistry.gauge(
        "business.revenue.total",
        java.util.concurrent.atomic.AtomicDouble(0.0)
    ) { it.get() } ?: java.util.concurrent.atomic.AtomicDouble(0.0)
    
    private val orderProcessingTime = io.micrometer.core.instrument.Timer.builder("business.order.processing.time")
        .description("Time to process an order from placement to fulfillment")
        .publishPercentiles(0.5, 0.95, 0.99)
        .register(meterRegistry)
    
    private val activeUsers = meterRegistry.gauge(
        "business.users.active",
        java.util.concurrent.atomic.AtomicLong(0)
    ) { it.get() } ?: java.util.concurrent.atomic.AtomicLong(0)
    
    fun recordOrderPlaced(amount: Double) {
        ordersPlaced.increment()
        orderRevenue.addAndGet(amount)
    }
    
    fun recordOrderProcessingTime(durationMs: Long) {
        orderProcessingTime.record(durationMs, java.util.concurrent.TimeUnit.MILLISECONDS)
    }
    
    fun setActiveUsers(count: Long) = activeUsers.set(count)
}

// Distributed Tracing
@org.springframework.stereotype.Component
class TracingInterceptor : org.springframework.web.servlet.HandlerInterceptor {
    
    private val tracer = io.micrometer.tracing.Tracer.NOOP
    
    override fun preHandle(
        request: jakarta.servlet.http.HttpServletRequest,
        response: jakarta.servlet.http.HttpServletResponse,
        handler: Any
    ): Boolean {
        val span = tracer.currentSpan()
        if (span != null) {
            // Add business context to trace
            val userId = request.getAttribute("userId") as? String
            userId?.let { span.tag("user.id", it) }
            span.tag("service.version", System.getenv("APP_VERSION") ?: "unknown")
        }
        return true
    }
}

// Alerting rules (Prometheus AlertManager format as reference)
/*
groups:
  - name: service-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          rate(http_server_requests_seconds_count{status=~"5.."}[5m])
          /
          rate(http_server_requests_seconds_count[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.application }}"
          description: "Error rate is {{ $value | humanizePercentage }}"
      
      - alert: HighP95Latency
        expr: |
          histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High P95 latency on {{ $labels.application }}"
      
      - alert: HighMemoryUsage
        expr: jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.85
        for: 5m
        labels:
          severity: warning
*/

// Loki log query examples
/*
# Error logs in last hour
{service="product-service"} |= "ERROR" | json | line_format "{{.message}}"

# Slow requests
{service="product-service"} | json | duration > 1s

# Orders placed by user
{service="order-service"} | json | event="ORDER_PLACED" | json userId="xxx"

# Trace correlation
{service="product-service"} | json | traceId="abc123"
*/
```

---

## Kubernetes Production Setup

```kotlin
// Kubernetes manifests as Kotlin data structures (Fabric8 client)
// Or: maintain as YAML files, shown here as reference

/*
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: ecommerce
  labels:
    app: product-service
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0   # Zero-downtime
  template:
    spec:
      serviceAccountName: product-service
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      
      containers:
      - name: product-service
        image: registry.ecommerce.com/product-service:1.0.0
        ports:
        - containerPort: 8080
          name: http
        
        # Resource limits
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # Health probes
        startupProbe:
          httpGet:
            path: /actuator/health/liveness
            port: http
          failureThreshold: 30
          periodSeconds: 10
        
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: http
          initialDelaySeconds: 0
          periodSeconds: 5
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: http
          periodSeconds: 10
          failureThreshold: 3
        
        # Environment
        env:
        - name: JAVA_OPTS
          value: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75"
        - name: SPRING_PROFILES_ACTIVE
          value: "kubernetes"
        - name: DB_URL
          valueFrom:
            secretKeyRef:
              name: product-service-secrets
              key: db-url
        
        # Security
        securityContext:
          readOnlyRootFilesystem: true
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
        
        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
      
      # Topology spread for HA
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: product-service
      
      terminationGracePeriodSeconds: 60

---
# HorizontalPodAutoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: product-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: product-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 25
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 30

---
# PodDisruptionBudget: ensure at least 1 pod always up
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: product-service-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: product-service
*/
```

---

## Infrastructure as Code

```kotlin
// Terraform module wrapper in Kotlin (Pulumi)
// Or reference Terraform module structure

/*
# modules/kotlin-service/main.tf

variable "service_name" {}
variable "image" {}
variable "replicas" { default = 3 }
variable "memory_mb" { default = 512 }
variable "cpu_millicores" { default = 500 }

# Standard Kubernetes deployment
resource "kubernetes_deployment" "service" {
  metadata {
    name = var.service_name
    namespace = "ecommerce"
    labels = {
      app     = var.service_name
      managed_by = "terraform"
    }
  }
  
  spec {
    replicas = var.replicas
    
    selector {
      match_labels = { app = var.service_name }
    }
    
    template {
      metadata {
        labels = { app = var.service_name }
        annotations = {
          "prometheus.io/scrape" = "true"
          "prometheus.io/port"   = "8080"
          "prometheus.io/path"   = "/actuator/prometheus"
        }
      }
      
      spec {
        container {
          name  = var.service_name
          image = var.image
          
          resources {
            requests = { memory = "${var.memory_mb}Mi", cpu = "${var.cpu_millicores/2}m" }
            limits   = { memory = "${var.memory_mb}Mi", cpu = "${var.cpu_millicores}m" }
          }
          
          liveness_probe {
            http_get { path = "/actuator/health/liveness"; port = 8080 }
            period_seconds = 10
            failure_threshold = 3
          }
          
          readiness_probe {
            http_get { path = "/actuator/health/readiness"; port = 8080 }
            period_seconds = 10
          }
        }
      }
    }
  }
}

# Standard HPA
resource "kubernetes_horizontal_pod_autoscaler_v2" "service" {
  metadata { name = var.service_name; namespace = "ecommerce" }
  
  spec {
    scale_target_ref {
      api_version = "apps/v1"
      kind        = "Deployment"
      name        = var.service_name
    }
    
    min_replicas = max(2, var.replicas)
    max_replicas = var.replicas * 4
    
    metric {
      type = "Resource"
      resource {
        name = "cpu"
        target { type = "Utilization"; average_utilization = 70 }
      }
    }
  }
}

# Standard PDB
resource "kubernetes_pod_disruption_budget_v1" "service" {
  metadata { name = var.service_name; namespace = "ecommerce" }
  spec {
    min_available = 1
    selector { match_labels = { app = var.service_name } }
  }
}
*/

// Backstage integration
@org.springframework.web.bind.annotation.RestController
@org.springframework.web.bind.annotation.RequestMapping("/.well-known")
class BackstageIntegration {
    
    @org.springframework.web.bind.annotation.GetMapping("/backstage.yaml")
    fun getCatalogInfo(): Map<String, Any> {
        return mapOf(
            "apiVersion" to "backstage.io/v1alpha1",
            "kind" to "Component",
            "metadata" to mapOf(
                "name" to "product-service",
                "description" to "Product catalog and inventory management service",
                "tags" to listOf("kotlin", "spring-boot", "postgresql"),
                "links" to listOf(
                    mapOf("url" to "https://grafana.ecommerce.com/d/product-service", "title" to "Grafana Dashboard"),
                    mapOf("url" to "https://jaeger.ecommerce.com/search?service=product-service", "title" to "Traces"),
                    mapOf("url" to "https://docs.ecommerce.com/product-service", "title" to "Documentation")
                ),
                "annotations" to mapOf(
                    "backstage.io/techdocs-ref" to "dir:.",
                    "github.com/project-slug" to "ecommerce/product-service",
                    "prometheus.io/rule" to "product-service"
                )
            ),
            "spec" to mapOf(
                "type" to "service",
                "lifecycle" to "production",
                "owner" to "platform-team",
                "system" to "ecommerce",
                "dependsOn" to listOf("resource:product-database", "resource:product-cache"),
                "providesApis" to listOf("product-api-v1")
            )
        )
    }
    
    @org.springframework.web.bind.annotation.GetMapping("/service-info.json")
    fun getServiceInfo(): ServiceInfo {
        return ServiceInfo(
            name = "product-service",
            version = System.getenv("APP_VERSION") ?: "unknown",
            environment = System.getenv("ENVIRONMENT") ?: "local",
            deployedAt = System.getenv("DEPLOY_TIME") ?: "",
            gitCommit = System.getenv("GIT_COMMIT") ?: "unknown",
            team = "platform-team",
            oncall = "product-team-pagerduty",
            runbook = "https://docs.ecommerce.com/runbooks/product-service"
        )
    }
}

data class ServiceInfo(
    val name: String,
    val version: String,
    val environment: String,
    val deployedAt: String,
    val gitCommit: String,
    val team: String,
    val oncall: String,
    val runbook: String
)
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง Service SDK Template

// สร้าง spring boot starter ที่ทุก service ต้องใช้
// ต้องมี:
// 1. Auto-configure observability (metrics, tracing, logging)
// 2. Standard health checks
// 3. Graceful shutdown
// 4. Security headers
// 5. Correlation ID propagation

@org.springframework.boot.autoconfigure.AutoConfiguration
@org.springframework.boot.autoconfigure.condition.ConditionalOnClass(org.springframework.web.servlet.DispatcherServlet::class)
class PlatformAutoConfiguration {
    
    @org.springframework.context.annotation.Bean
    @org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
    fun correlationIdFilter(): javax.servlet.Filter {
        return javax.servlet.Filter { request, response, chain ->
            val req = request as jakarta.servlet.http.HttpServletRequest
            val res = response as jakarta.servlet.http.HttpServletResponse
            
            val correlationId = req.getHeader("X-Correlation-ID")
                ?: java.util.UUID.randomUUID().toString()
            
            org.slf4j.MDC.put("correlationId", correlationId)
            res.addHeader("X-Correlation-ID", correlationId)
            
            try {
                chain.doFilter(request, response)
            } finally {
                org.slf4j.MDC.remove("correlationId")
            }
        }
    }
    
    @org.springframework.context.annotation.Bean
    @org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean
    fun securityHeadersFilter(): javax.servlet.Filter {
        return javax.servlet.Filter { request, response, chain ->
            val res = response as jakarta.servlet.http.HttpServletResponse
            res.setHeader("X-Content-Type-Options", "nosniff")
            res.setHeader("X-Frame-Options", "DENY")
            res.setHeader("Referrer-Policy", "strict-origin-when-cross-origin")
            chain.doFilter(request, response)
        }
    }
    
    @org.springframework.context.annotation.Bean
    fun platformHealthIndicator(): org.springframework.boot.actuate.health.HealthIndicator {
        return org.springframework.boot.actuate.health.HealthIndicator {
            org.springframework.boot.actuate.health.Health.up()
                .withDetail("platform.sdk.version", "1.0.0")
                .build()
        }
    }
}

// resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports:
// com.ecommerce.platform.PlatformAutoConfiguration
```

---

## สรุป Part 88

```
✅ Platform Engineering: IDP สร้าง Golden Path สำหรับ developers
✅ DORA Metrics: Deployment Frequency, Lead Time, CFR, MTTR
✅ Standard application.yml: lifecycle, health probes, metrics
✅ ServiceHealthIndicator: DB + Redis health check
✅ GracefulShutdownHook: stop Kafka consumers before shutdown
✅ Structured logging: logstash-logback-encoder + MDC fields
✅ BusinessMetrics: ordersPlaced, revenue, processing time gauge
✅ TracingInterceptor: add business context to traces
✅ AlertManager rules: error rate, P95 latency, memory
✅ Loki queries: filter by service, level, event, traceId
✅ K8s deployment: RollingUpdate, maxUnavailable=0
✅ securityContext: runAsNonRoot, readOnlyRootFilesystem
✅ Resources: requests < limits for burstable QoS
✅ startupProbe: separate from liveness for slow starts
✅ preStop sleep: drain in-flight requests
✅ topologySpreadConstraints: spread across nodes
✅ HPA: CPU 70% + memory 80%, scale up/down behavior
✅ PodDisruptionBudget: minAvailable=1 for zero-downtime ops
✅ Terraform module: standardize deployment with variables
✅ Backstage catalog-info.yaml: service metadata, links
✅ ServiceInfo endpoint: version, gitCommit, team, runbook
✅ Platform AutoConfiguration: auto-configure correlation ID, headers
✅ spring.boot.autoconfigure.AutoConfiguration.imports: zero config
```

---

*Part 88/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
