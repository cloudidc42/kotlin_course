# Part 98: Production Operations & SRE

## สารบัญ
1. [SRE หลักการและ SLO/SLI/SLA](#sre-หลักการและ-sloslisla)
2. [Incident Management](#incident-management)
3. [Chaos Engineering](#chaos-engineering)
4. [Capacity Planning](#capacity-planning)
5. [Production Readiness Checklist](#production-readiness-checklist)
6. [On-Call Runbooks](#on-call-runbooks)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## SRE หลักการและ SLO/SLI/SLA

```
SRE (Site Reliability Engineering):
- Software engineers who manage production systems
- Apply software engineering to operations problems

SLA (Service Level Agreement): 
- สัญญากับลูกค้า: "we guarantee 99.9% uptime"
- มีบทลงโทษถ้าทำไม่ได้ (refund, credits)

SLO (Service Level Objective):
- เป้าหมายภายใน: "we target 99.95% uptime"
- ควรดีกว่า SLA เพื่อมี buffer

SLI (Service Level Indicator):
- สิ่งที่วัดได้จริง: actual uptime %, latency, error rate

Error Budget:
- 99.9% availability = 0.1% downtime = 43.8 min/month
- ถ้าใช้ error budget หมด → freeze deployments
- ถ้าเหลือ error budget → เร่ง development

Golden Signals (Google SRE book):
1. Latency:    time to serve a request (P50, P95, P99)
2. Traffic:    requests per second
3. Errors:     error rate (5xx, timeouts)
4. Saturation: CPU, memory, disk utilization %
```

---

## SLO Implementation

```kotlin
@org.springframework.stereotype.Component
class SloMonitor(
    private val meterRegistry: io.micrometer.core.instrument.MeterRegistry,
    private val alertManager: AlertManager
) {
    
    // Track request success/failure for error rate SLI
    fun recordRequest(path: String, method: String, statusCode: Int, durationMs: Long) {
        val tags = io.micrometer.core.instrument.Tags.of(
            "path", normalizePath(path),
            "method", method,
            "status_class", "${statusCode / 100}xx"
        )
        
        meterRegistry.counter("http.requests.total", tags).increment()
        meterRegistry.timer("http.request.duration", tags).record(
            java.time.Duration.ofMillis(durationMs)
        )
        
        if (statusCode >= 500) {
            meterRegistry.counter("http.errors.total", tags).increment()
        }
    }
    
    // Calculate current error rate (SLI)
    fun getErrorRate(windowMinutes: Int): Double {
        val total = meterRegistry.counter("http.requests.total").count()
        val errors = meterRegistry.counter("http.errors.total").count()
        
        return if (total > 0) errors / total else 0.0
    }
    
    // Check if SLO is at risk
    @org.springframework.scheduling.annotation.Scheduled(fixedDelay = 60_000)
    fun checkSlo() {
        val errorRate = getErrorRate(60)
        val sloTarget = 0.001  // 0.1% error rate target
        
        if (errorRate > sloTarget * 2) {
            alertManager.fire(Alert(
                name = "SLOAtRisk",
                severity = "critical",
                message = "Error rate ${errorRate * 100}% is ${errorRate / sloTarget}x above SLO target",
                runbook = "https://runbook.example.com/high-error-rate"
            ))
        }
        
        // Check latency SLO: P95 < 500ms
        val p95 = meterRegistry.timer("http.request.duration")
            .percentile(0.95)
        
        if (p95 > 500.0) {
            alertManager.fire(Alert(
                name = "LatencySLOBreach",
                severity = "warning",
                message = "P95 latency ${p95}ms exceeds SLO target 500ms",
                runbook = "https://runbook.example.com/high-latency"
            ))
        }
    }
    
    private fun normalizePath(path: String): String {
        // /api/v1/products/123 → /api/v1/products/:id
        return path.replace(Regex("/[0-9a-f-]{8,}"), "/:id")
            .replace(Regex("/\\d+"), "/:id")
    }
}

data class Alert(val name: String, val severity: String, val message: String, val runbook: String)
interface AlertManager { fun fire(alert: Alert) }
fun io.micrometer.core.instrument.Timer.percentile(p: Double): Double = 0.0
```

---

## Incident Management

```kotlin
// Incident handling automation
@org.springframework.stereotype.Service
class IncidentService(
    private val slackClient: SlackClient,
    private val pagerDutyClient: PagerDutyClient,
    private val statusPageClient: StatusPageClient,
    private val incidentRepository: IncidentRepository
) {
    
    private val log = org.slf4j.LoggerFactory.getLogger(IncidentService::class.java)
    
    fun declareIncident(
        title: String,
        severity: IncidentSeverity,
        affectedServices: List<String>,
        triggeredBy: String
    ): Incident {
        val incident = Incident(
            id = java.util.UUID.randomUUID().toString(),
            title = title,
            severity = severity,
            status = IncidentStatus.INVESTIGATING,
            affectedServices = affectedServices,
            startedAt = java.time.Instant.now(),
            declaredBy = triggeredBy
        )
        
        incidentRepository.save(incident)
        
        // Notify on-call
        when (severity) {
            IncidentSeverity.SEV1 -> {
                pagerDutyClient.triggerHighUrgency(incident)
                statusPageClient.createIncident(incident, ComponentStatus.MAJOR_OUTAGE)
                slackClient.postToChannel("#incidents", formatIncidentMessage(incident))
            }
            IncidentSeverity.SEV2 -> {
                pagerDutyClient.triggerLowUrgency(incident)
                statusPageClient.createIncident(incident, ComponentStatus.PARTIAL_OUTAGE)
                slackClient.postToChannel("#incidents", formatIncidentMessage(incident))
            }
            IncidentSeverity.SEV3 -> {
                slackClient.postToChannel("#alerts", formatIncidentMessage(incident))
            }
        }
        
        log.warn("INCIDENT DECLARED: ${incident.id} - ${incident.title} [${incident.severity}]")
        
        return incident
    }
    
    fun updateIncident(id: String, update: IncidentUpdate) {
        val incident = incidentRepository.findById(id)
            ?: throw RuntimeException("Incident $id not found")
        
        val updated = incident.copy(
            status = update.newStatus ?: incident.status,
            updates = incident.updates + update
        )
        
        incidentRepository.save(updated)
        
        slackClient.postToChannel("#incidents", 
            "📊 Incident Update [${incident.id}]\n" +
            "Status: ${incident.status} → ${updated.status}\n" +
            "Update: ${update.message}")
        
        if (updated.status == IncidentStatus.RESOLVED) {
            statusPageClient.resolveIncident(id)
            schedulePostMortem(updated)
        }
    }
    
    private fun schedulePostMortem(incident: Incident) {
        // Schedule post-mortem 48h after resolution
        val postMortemAt = incident.resolvedAt!!.plus(java.time.Duration.ofHours(48))
        
        slackClient.scheduleMessage(
            "#incidents",
            "📝 Post-mortem due for incident ${incident.id}: ${incident.title}\n" +
            "Template: https://wiki.example.com/post-mortem-template",
            postMortemAt
        )
    }
    
    private fun formatIncidentMessage(incident: Incident): String {
        return """
🚨 *INCIDENT DECLARED* [${incident.severity}]
*ID:* ${incident.id}
*Title:* ${incident.title}
*Status:* ${incident.status}
*Services:* ${incident.affectedServices.joinToString(", ")}
*Declared by:* ${incident.declaredBy}
*Runbook:* https://runbook.example.com/${incident.title.lowercase().replace(" ", "-")}
        """.trimIndent()
    }
}

enum class IncidentSeverity { SEV1, SEV2, SEV3 }
enum class IncidentStatus { INVESTIGATING, IDENTIFIED, MONITORING, RESOLVED }
enum class ComponentStatus { OPERATIONAL, DEGRADED_PERFORMANCE, PARTIAL_OUTAGE, MAJOR_OUTAGE }

data class Incident(
    val id: String, val title: String, val severity: IncidentSeverity,
    val status: IncidentStatus, val affectedServices: List<String>,
    val startedAt: java.time.Instant, val resolvedAt: java.time.Instant? = null,
    val declaredBy: String, val updates: List<IncidentUpdate> = emptyList()
)

data class IncidentUpdate(val message: String, val newStatus: IncidentStatus?, val timestamp: java.time.Instant = java.time.Instant.now())

interface IncidentRepository { fun save(i: Incident); fun findById(id: String): Incident? }
interface SlackClient { fun postToChannel(ch: String, msg: String); fun scheduleMessage(ch: String, msg: String, at: java.time.Instant) }
interface PagerDutyClient { fun triggerHighUrgency(i: Incident); fun triggerLowUrgency(i: Incident) }
interface StatusPageClient { fun createIncident(i: Incident, status: ComponentStatus); fun resolveIncident(id: String) }
```

---

## Production Readiness Checklist

```kotlin
// Automate production readiness checks
@org.springframework.stereotype.Service
class ProductionReadinessChecker {
    
    data class CheckResult(val name: String, val passed: Boolean, val details: String)
    data class ReadinessReport(val checks: List<CheckResult>, val passed: Boolean = checks.all { it.passed })
    
    fun runChecks(service: ServiceConfig): ReadinessReport {
        val checks = listOf(
            checkHealthEndpoints(service),
            checkMetricsEndpoint(service),
            checkLoggingConfig(service),
            checkResourceLimits(service),
            checkSecurityConfig(service),
            checkDependencyVersions(service),
            checkDocumentation(service)
        )
        
        return ReadinessReport(checks)
    }
    
    private fun checkHealthEndpoints(service: ServiceConfig): CheckResult {
        val hasReadiness = service.endpoints.contains("/actuator/health/readiness")
        val hasLiveness = service.endpoints.contains("/actuator/health/liveness")
        
        return CheckResult(
            name = "Health Endpoints",
            passed = hasReadiness && hasLiveness,
            details = when {
                !hasReadiness -> "Missing /actuator/health/readiness"
                !hasLiveness -> "Missing /actuator/health/liveness"
                else -> "✓ readiness and liveness probes configured"
            }
        )
    }
    
    private fun checkMetricsEndpoint(service: ServiceConfig): CheckResult {
        return CheckResult(
            name = "Metrics",
            passed = service.endpoints.contains("/actuator/prometheus"),
            details = if (service.endpoints.contains("/actuator/prometheus"))
                "✓ Prometheus endpoint available"
            else "✗ Missing /actuator/prometheus - add micrometer-registry-prometheus"
        )
    }
    
    private fun checkResourceLimits(service: ServiceConfig): CheckResult {
        val hasLimits = service.k8sConfig?.resourceLimits != null
        val hasRequests = service.k8sConfig?.resourceRequests != null
        
        return CheckResult(
            name = "Resource Limits",
            passed = hasLimits && hasRequests,
            details = if (hasLimits && hasRequests)
                "✓ CPU and memory limits configured"
            else "✗ Missing resource limits/requests in K8s deployment"
        )
    }
    
    private fun checkSecurityConfig(service: ServiceConfig): CheckResult {
        val issues = mutableListOf<String>()
        
        if (!service.k8sConfig?.runAsNonRoot!!) issues.add("runAsNonRoot not set")
        if (!service.k8sConfig.readOnlyRootFilesystem) issues.add("readOnlyRootFilesystem not set")
        if (service.k8sConfig.allowPrivilegeEscalation) issues.add("allowPrivilegeEscalation=true")
        
        return CheckResult(
            name = "Security Context",
            passed = issues.isEmpty(),
            details = if (issues.isEmpty()) "✓ Security context properly hardened"
            else "✗ Security issues: ${issues.joinToString(", ")}"
        )
    }
    
    private fun checkLoggingConfig(service: ServiceConfig): CheckResult {
        return CheckResult(
            name = "Structured Logging",
            passed = service.loggingFormat == "json",
            details = if (service.loggingFormat == "json")
                "✓ JSON structured logging configured"
            else "✗ Use JSON logging for log aggregation (logstash-logback-encoder)"
        )
    }
    
    private fun checkDependencyVersions(service: ServiceConfig): CheckResult {
        val outdated = service.dependencies
            .filter { it.hasKnownVulnerability || it.majorVersionsBehind > 1 }
        
        return CheckResult(
            name = "Dependencies",
            passed = outdated.isEmpty(),
            details = if (outdated.isEmpty()) "✓ All dependencies up to date"
            else "✗ Outdated/vulnerable: ${outdated.map { it.name }.joinToString(", ")}"
        )
    }
    
    private fun checkDocumentation(service: ServiceConfig): CheckResult {
        val hasCatalogInfo = service.files.contains("catalog-info.yaml")
        val hasRunbook = service.files.contains("docs/runbook.md")
        
        return CheckResult(
            name = "Documentation",
            passed = hasCatalogInfo && hasRunbook,
            details = buildString {
                if (!hasCatalogInfo) append("✗ Missing catalog-info.yaml (Backstage)\n")
                if (!hasRunbook) append("✗ Missing docs/runbook.md\n")
                if (hasCatalogInfo && hasRunbook) append("✓ catalog-info.yaml and runbook present")
            }.trim()
        )
    }
}

data class ServiceConfig(
    val name: String,
    val endpoints: List<String>,
    val k8sConfig: K8sSecurityConfig?,
    val loggingFormat: String,
    val dependencies: List<DependencyInfo>,
    val files: List<String>
)

data class K8sSecurityConfig(
    val runAsNonRoot: Boolean,
    val readOnlyRootFilesystem: Boolean,
    val allowPrivilegeEscalation: Boolean,
    val resourceLimits: ResourceSpec?,
    val resourceRequests: ResourceSpec?
)

data class ResourceSpec(val cpu: String, val memory: String)
data class DependencyInfo(val name: String, val version: String, val hasKnownVulnerability: Boolean, val majorVersionsBehind: Int)
```

---

## On-Call Runbooks

```kotlin
// Runbook as Code: structured troubleshooting guides
data class RunbookStep(
    val step: Int,
    val description: String,
    val commands: List<String>,
    val expectedResult: String,
    val escalateTo: String? = null
)

data class Runbook(
    val alertName: String,
    val severity: String,
    val description: String,
    val triage: List<RunbookStep>,
    val mitigation: List<RunbookStep>,
    val rootCauseFix: List<RunbookStep>
)

val highErrorRateRunbook = Runbook(
    alertName = "HighErrorRate",
    severity = "critical",
    description = "5xx error rate > 5% for more than 5 minutes",
    triage = listOf(
        RunbookStep(1, "Check recent deployments", 
            listOf("kubectl rollout history deployment/ecommerce-api -n production"),
            "Look for deployments in last 30 minutes"),
        RunbookStep(2, "Check pod logs for errors",
            listOf(
                "kubectl logs -l app=ecommerce-api -n production --tail=100",
                "kubectl logs -l app=ecommerce-api -n production --since=10m | grep ERROR"
            ),
            "Identify common error pattern"),
        RunbookStep(3, "Check database connectivity",
            listOf(
                "kubectl exec -it <pod> -n production -- sh -c 'nc -zv postgres 5432'",
                "kubectl get pods -n databases"
            ),
            "Postgres pod should be Running")
    ),
    mitigation = listOf(
        RunbookStep(1, "If caused by deployment: rollback",
            listOf("kubectl rollout undo deployment/ecommerce-api -n production"),
            "Error rate should drop within 2 minutes"),
        RunbookStep(2, "If database issue: check connection pool",
            listOf("curl http://ecommerce-api:8080/actuator/metrics/hikaricp.connections"),
            "active connections should be < maxPoolSize"),
        RunbookStep(3, "If memory issue: restart pods",
            listOf("kubectl rollout restart deployment/ecommerce-api -n production"),
            "New pods should start healthy")
    ),
    rootCauseFix = listOf(
        RunbookStep(1, "Create incident ticket with findings", emptyList(), "Document root cause"),
        RunbookStep(2, "Create post-mortem action items", emptyList(), "Prevent recurrence")
    )
)
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง automated health dashboard

@org.springframework.web.bind.annotation.RestController
@org.springframework.web.bind.annotation.RequestMapping("/ops")
class OperationsDashboardController(
    private val readinessChecker: ProductionReadinessChecker,
    private val incidentService: IncidentService,
    private val sloMonitor: SloMonitor
) {
    
    @org.springframework.web.bind.annotation.GetMapping("/health-report")
    fun getHealthReport(): Map<String, Any> {
        return mapOf(
            "timestamp" to java.time.Instant.now(),
            "errorRate" to sloMonitor.getErrorRate(60),
            "status" to if (sloMonitor.getErrorRate(60) < 0.001) "HEALTHY" else "DEGRADED"
        )
    }
    
    @org.springframework.web.bind.annotation.PostMapping("/incidents")
    @org.springframework.security.access.prepost.PreAuthorize("hasRole('SRE')")
    fun declareIncident(@org.springframework.web.bind.annotation.RequestBody request: DeclareIncidentRequest): Incident {
        return incidentService.declareIncident(
            title = request.title,
            severity = request.severity,
            affectedServices = request.affectedServices,
            triggeredBy = "sre-team"
        )
    }
    
    @org.springframework.web.bind.annotation.GetMapping("/readiness-check")
    @org.springframework.security.access.prepost.PreAuthorize("hasRole('ADMIN')")
    fun runReadinessCheck(): ProductionReadinessChecker.ReadinessReport {
        val config = ServiceConfig(
            name = "ecommerce-api",
            endpoints = listOf("/actuator/health/readiness", "/actuator/health/liveness", "/actuator/prometheus"),
            k8sConfig = K8sSecurityConfig(true, true, false, 
                ResourceSpec("1000m", "1Gi"), ResourceSpec("250m", "512Mi")),
            loggingFormat = "json",
            dependencies = emptyList(),
            files = listOf("catalog-info.yaml", "docs/runbook.md")
        )
        return readinessChecker.runChecks(config)
    }
}

data class DeclareIncidentRequest(val title: String, val severity: IncidentSeverity, val affectedServices: List<String>)
```

---

## สรุป Part 98

```
✅ SLI: what we measure (error rate, latency)
✅ SLO: internal target (99.95% uptime)
✅ SLA: customer promise (99.9% uptime)
✅ Error Budget: 1 - SLO = allowable downtime
✅ Golden Signals: latency, traffic, errors, saturation
✅ SloMonitor: track requests, compute error rate
✅ normalizePath: /products/123 → /products/:id
✅ meterRegistry.counter/timer: Micrometer recording
✅ @Scheduled: periodic SLO check
✅ IncidentService: declare/update/resolve incidents
✅ IncidentSeverity: SEV1 (P0), SEV2 (P1), SEV3 (P2)
✅ SEV1: PagerDuty high urgency + status page + Slack
✅ schedulePostMortem: 48h after resolution
✅ Runbook as Code: structured troubleshooting
✅ RunbookStep: step, description, commands, expected
✅ ProductionReadinessChecker: automated checks
✅ CheckResult: pass/fail with details
✅ ReadinessReport: aggregate all checks
✅ Check: health, metrics, logging, resources, security, docs
✅ catalog-info.yaml: Backstage service catalog
✅ OperationsDashboard: health report, incident management
✅ @PreAuthorize("hasRole('SRE')"): ops endpoints secured
✅ Post-mortem: blameless, action items, prevent recurrence
```

---

*Part 98/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
