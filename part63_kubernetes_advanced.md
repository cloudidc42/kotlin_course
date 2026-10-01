# Part 63: Kubernetes ขั้นสูงสำหรับ Kotlin Applications

## สารบัญ
1. [Helm Charts](#helm-charts)
2. [StatefulSets](#statefulsets)
3. [Horizontal Pod Autoscaler](#horizontal-pod-autoscaler)
4. [RBAC ใน Kubernetes](#rbac-ใน-kubernetes)
5. [ConfigMaps และ Secrets](#configmaps-และ-secrets)
6. [Health Probes ด้วย Spring Boot Actuator](#health-probes-ด้วย-spring-boot-actuator)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Helm Charts

Helm คือ package manager สำหรับ Kubernetes — จัดการ YAML templates ด้วย values

```yaml
# Chart.yaml
apiVersion: v2
name: kotlin-microservice
description: A Kotlin Spring Boot microservice
type: application
version: 0.1.0
appVersion: "1.0.0"

# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "kotlin-microservice.fullname" . }}
  labels:
    {{- include "kotlin-microservice.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "kotlin-microservice.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "kotlin-microservice.selectorLabels" . | nindent 8 }}
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      serviceAccountName: {{ include "kotlin-microservice.serviceAccountName" . }}
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: {{ .Values.environment }}
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "kotlin-microservice.fullname" . }}-db
                  key: password
            - name: JAVA_OPTS
              value: {{ .Values.jvmOptions | quote }}
          resources:
            requests:
              cpu: {{ .Values.resources.requests.cpu }}
              memory: {{ .Values.resources.requests.memory }}
            limits:
              cpu: {{ .Values.resources.limits.cpu }}
              memory: {{ .Values.resources.limits.memory }}
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 30
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            failureThreshold: 30
            periodSeconds: 10
          volumeMounts:
            - name: config
              mountPath: /config
              readOnly: true
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: config
          configMap:
            name: {{ include "kotlin-microservice.fullname" . }}-config
        - name: tmp
          emptyDir: {}

---
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "kotlin-microservice.fullname" . }}
  labels:
    {{- include "kotlin-microservice.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "kotlin-microservice.selectorLabels" . | nindent 4 }}

---
# templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "kotlin-microservice.fullname" . }}
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - {{ .Values.ingress.host }}
      secretName: {{ .Values.ingress.host }}-tls
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "kotlin-microservice.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

```yaml
# values.yaml
replicaCount: 2
environment: production
jvmOptions: "-Xmx512m -Xms256m -XX:+UseContainerSupport"

image:
  repository: registry.example.com/myapp/kotlin-service
  pullPolicy: IfNotPresent
  tag: ""

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  host: api.example.com

resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

serviceAccount:
  create: true
  name: ""
```

```bash
# Helm commands
helm create kotlin-microservice
helm lint kotlin-microservice/
helm template kotlin-microservice/ --values values-prod.yaml

# Install or upgrade
helm upgrade --install kotlin-microservice ./kotlin-microservice \
  --namespace production \
  --create-namespace \
  --values values-prod.yaml \
  --set image.tag=1.2.3 \
  --wait \
  --timeout 10m

# Rollback
helm rollback kotlin-microservice 1 --namespace production

# History
helm history kotlin-microservice --namespace production
```

---

## StatefulSets

```yaml
# StatefulSet สำหรับ stateful services (database, kafka, etc.)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      initContainers:
        - name: init-replica
          image: postgres:16-alpine
          command:
            - sh
            - -c
            - |
              if [ $(hostname | grep -o '[0-9]*$') -eq 0 ]; then
                echo "Primary node"
              else
                echo "Replica node"
              fi
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: myapp
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
            limits:
              cpu: 2000m
              memory: 4Gi
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 20Gi

---
# Headless Service สำหรับ StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None  # headless
  selector:
    app: postgres
  ports:
    - port: 5432

---
# Regular Service สำหรับ read
apiVersion: v1
kind: Service
metadata:
  name: postgres-read
spec:
  selector:
    app: postgres
  ports:
    - port: 5432
```

---

## Horizontal Pod Autoscaler

```yaml
# HPA based on CPU and Memory
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kotlin-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kotlin-service
  minReplicas: 2
  maxReplicas: 20
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
    # Custom metric จาก Prometheus (KEDA หรือ custom metrics adapter)
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 100
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
        - type: Pods
          value: 4
          periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 20
          periodSeconds: 60

---
# PodDisruptionBudget: เพื่อให้มี pod อย่างน้อย N ตัวตลอดเวลา
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: kotlin-service-pdb
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: kotlin-service
```

---

## RBAC ใน Kubernetes

```yaml
# ServiceAccount สำหรับ application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kotlin-service
  namespace: production
  annotations:
    iam.gke.io/gcp-service-account: kotlin-service@my-project.iam.gserviceaccount.com

---
# Role: permission ใน namespace เดียว
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: kotlin-service-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps", "secrets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: kotlin-service-rolebinding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: kotlin-service
    namespace: production
roleRef:
  kind: Role
  apiVersion: rbac.authorization.k8s.io/v1
  name: kotlin-service-role

---
# ClusterRole: permission across all namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: readonly-cluster-role
rules:
  - apiGroups: [""]
    resources: ["nodes", "namespaces"]
    verbs: ["get", "list", "watch"]
```

---

## ConfigMaps และ Secrets

```yaml
# ConfigMap: configuration data
apiVersion: v1
kind: ConfigMap
metadata:
  name: kotlin-service-config
  namespace: production
data:
  application.yml: |
    spring:
      application:
        name: kotlin-service
      datasource:
        url: jdbc:postgresql://postgres-headless:5432/myapp
        hikari:
          maximum-pool-size: 10
          minimum-idle: 2
          connection-timeout: 30000
      redis:
        host: redis-service
        port: 6379
        timeout: 2000ms
      kafka:
        bootstrap-servers: kafka-headless:9092
        consumer:
          group-id: kotlin-service
          auto-offset-reset: earliest
    management:
      endpoints:
        web:
          exposure:
            include: health,info,prometheus,metrics
      health:
        probes:
          enabled: true
      metrics:
        export:
          prometheus:
            enabled: true
    logging:
      level:
        com.example: INFO
        org.springframework.security: WARN
  
  logback-spring.xml: |
    <configuration>
      <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
          <includeMdcKeyName>traceId</includeMdcKeyName>
          <includeMdcKeyName>spanId</includeMdcKeyName>
        </encoder>
      </appender>
      <root level="INFO">
        <appender-ref ref="STDOUT"/>
      </root>
    </configuration>

---
# Secret: sensitive data (base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: kotlin-service-db
  namespace: production
type: Opaque
stringData:  # stringData: plaintext (kubernetes encodes automatically)
  password: "my-super-secret-password"
  url: "postgresql://postgres-headless:5432/myapp"

---
# ExternalSecret (External Secrets Operator จาก AWS Secrets Manager / Vault)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: kotlin-service-external-secret
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: kotlin-service-credentials
    creationPolicy: Owner
  data:
    - secretKey: db-password
      remoteRef:
        key: production/kotlin-service/db
        property: password
    - secretKey: jwt-secret
      remoteRef:
        key: production/kotlin-service/jwt
        property: secret
```

---

## Health Probes ด้วย Spring Boot Actuator

```kotlin
// Spring Boot Actuator Health Probes Configuration
// application.yml:
// management:
//   health:
//     probes:
//       enabled: true
//   endpoint:
//     health:
//       show-details: when_authorized
//   endpoints:
//     web:
//       exposure:
//         include: health,info,prometheus

// Custom Health Indicator
@Component
class DatabaseHealthIndicator(
    private val dataSource: javax.sql.DataSource
) : HealthIndicator {
    
    override fun health(): Health {
        return try {
            dataSource.connection.use { conn ->
                val stmt = conn.createStatement()
                stmt.execute("SELECT 1")
                Health.up()
                    .withDetail("database", "PostgreSQL")
                    .withDetail("status", "connected")
                    .build()
            }
        } catch (ex: Exception) {
            Health.down()
                .withDetail("database", "PostgreSQL")
                .withDetail("error", ex.message)
                .build()
        }
    }
}

@Component
class KafkaHealthIndicator(
    private val kafkaAdmin: KafkaAdmin
) : HealthIndicator {
    
    override fun health(): Health {
        return try {
            kafkaAdmin.describeTopics("orders", "products")
            Health.up().withDetail("kafka", "connected").build()
        } catch (ex: Exception) {
            Health.down().withDetail("kafka", ex.message).build()
        }
    }
}

// Liveness vs Readiness
@Component
class ApplicationLivenessState : LivenessStateHealthIndicator(ApplicationAvailability::class.java) {
    // Returns UP when application is alive (not in deadlock/corrupted state)
}

@Component 
class ApplicationReadinessState : ReadinessStateHealthIndicator(ApplicationAvailability::class.java) {
    // Returns ACCEPTING_TRAFFIC or REFUSING_TRAFFIC
}

// Custom readiness check
@Component
class CustomReadinessCheck : HealthIndicator {
    
    @Autowired
    private lateinit var availability: ApplicationAvailability
    
    var isWarmupComplete = false
    
    @PostConstruct
    fun warmup() {
        Thread.sleep(5000)  // Simulate warmup
        isWarmupComplete = true
        // Signal ready
        ApplicationContextEvent(context = TODO())
    }
    
    override fun health(): Health {
        return if (isWarmupComplete) {
            Health.up().withDetail("warmup", "complete").build()
        } else {
            Health.down().withDetail("warmup", "in-progress").build()
        }
    }
}

typealias HealthIndicator = org.springframework.boot.actuate.health.HealthIndicator
typealias Health = org.springframework.boot.actuate.health.Health
typealias KafkaAdmin = org.springframework.kafka.core.KafkaAdmin
typealias ApplicationAvailability = org.springframework.boot.availability.ApplicationAvailability
typealias LivenessStateHealthIndicator = org.springframework.boot.actuate.availability.LivenessStateHealthIndicator
typealias ReadinessStateHealthIndicator = org.springframework.boot.actuate.availability.ReadinessStateHealthIndicator
```

---

## Graceful Shutdown

```kotlin
// application.yml:
// server:
//   shutdown: graceful
// spring:
//   lifecycle:
//     timeout-per-shutdown-phase: 30s

@Component
class GracefulShutdownHandler : ApplicationListener<ContextClosingEvent> {
    
    override fun onApplicationEvent(event: ContextClosingEvent) {
        println("Application is shutting down gracefully...")
        // Stop accepting new requests (happens automatically with graceful shutdown)
        // Drain in-flight requests
        // Close Kafka consumers
        // Flush metrics
        Thread.sleep(1000)  // Wait for in-flight requests
        println("Shutdown complete")
    }
}

// Kafka consumer graceful shutdown
@Component
class KafkaGracefulShutdown(
    private val kafkaListenerEndpointRegistry: KafkaListenerEndpointRegistry
) : ApplicationListener<ContextClosingEvent> {
    
    override fun onApplicationEvent(event: ContextClosingEvent) {
        kafkaListenerEndpointRegistry.allListenerContainers.forEach { container ->
            container.stop()
        }
    }
}

typealias ContextClosingEvent = org.springframework.context.event.ContextClosingEvent
typealias KafkaListenerEndpointRegistry = org.springframework.kafka.config.KafkaListenerEndpointRegistry
```

---

## แบบฝึกหัด

```yaml
# Exercise: สร้าง Helm chart สำหรับ microservice ด้วย:

# 1. Deployment ที่มี:
#    - Rolling update strategy (maxSurge=1, maxUnavailable=0)
#    - Resource requests/limits
#    - All 3 health probes (liveness, readiness, startup)
#    - Environment variables from ConfigMap + Secrets

# 2. HPA ที่ scale บน CPU 70% และ custom metric (http_requests_per_second > 100)

# 3. PodDisruptionBudget: minAvailable=1

# 4. NetworkPolicy: อนุญาตให้ inbound traffic เฉพาะจาก ingress-controller namespace

---
# NetworkPolicy template (แนวทาง)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: kotlin-service-netpol
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: kotlin-service
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
        - podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: production
      ports:
        - protocol: TCP
          port: 5432  # postgres
        - protocol: TCP
          port: 6379  # redis
        - protocol: TCP
          port: 9092  # kafka
```

---

## สรุป Part 63

```
✅ Helm Charts: package manager สำหรับ Kubernetes
✅ Chart.yaml: metadata, version
✅ templates/: Go templates สำหรับ K8s manifests
✅ values.yaml: default configuration values
✅ helm upgrade --install: deploy/update application
✅ StatefulSets: stateful applications (DB, Kafka)
✅ volumeClaimTemplates: persistent storage per pod
✅ Headless Service: pod-to-pod communication
✅ HPA: autoscaling based on CPU/Memory/custom metrics
✅ PodDisruptionBudget: maintain availability during updates
✅ RBAC: ServiceAccount, Role, RoleBinding, ClusterRole
✅ ConfigMap: configuration files and key-value data
✅ Secret: sensitive data (base64/external secrets)
✅ ExternalSecret: sync from AWS Secrets Manager/Vault
✅ Actuator Health Probes: liveness, readiness, startup
✅ Custom HealthIndicator: DB, Kafka health checks
✅ NetworkPolicy: restrict inbound/outbound traffic
✅ Graceful Shutdown: drain requests before pod stops
```

---

*Part 63/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
