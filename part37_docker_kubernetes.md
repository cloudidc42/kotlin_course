# Part 37: Docker และ Kubernetes สำหรับ Kotlin Applications

## สารบัญ
1. [Dockerfile สำหรับ Kotlin](#dockerfile-สำหรับ-kotlin)
2. [Docker Compose](#docker-compose)
3. [Multi-stage Build](#multi-stage-build)
4. [Kubernetes Basics](#kubernetes-basics)
5. [Deployment และ Service](#deployment-และ-service)
6. [ConfigMap และ Secret](#configmap-และ-secret)
7. [Health Checks](#health-checks)
8. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Dockerfile สำหรับ Kotlin/Spring Boot

```dockerfile
# Dockerfile (Single stage - ง่ายแต่ image ใหญ่)
FROM eclipse-temurin:21-jre

WORKDIR /app

# Copy the fat JAR
COPY build/libs/*.jar app.jar

# Non-root user for security
RUN addgroup --system appgroup && adduser --system appuser --ingroup appgroup
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

EXPOSE 8080

ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-jar", "app.jar"]
```

```dockerfile
# Dockerfile.layered (Layered JAR - faster builds)
FROM eclipse-temurin:21-jre as builder
WORKDIR /app
COPY build/libs/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

FROM eclipse-temurin:21-jre
WORKDIR /app
RUN addgroup --system appgroup && adduser --system appuser --ingroup appgroup

COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

USER appuser

HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

EXPOSE 8080

ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "org.springframework.boot.loader.launch.JarLauncher"]
```

---

## Multi-stage Build

```dockerfile
# Dockerfile.multistage (Build and run in one file)
# Stage 1: Build
FROM gradle:8.6-jdk21 AS build

WORKDIR /app

# Cache dependencies first
COPY build.gradle.kts settings.gradle.kts ./
COPY gradle/ gradle/
RUN gradle dependencies --no-daemon 2>/dev/null || true

# Copy source and build
COPY src/ src/
RUN gradle bootJar --no-daemon -x test

# Stage 2: Extract layers
FROM eclipse-temurin:21-jre AS extract
WORKDIR /app
COPY --from=build /app/build/libs/*.jar app.jar
RUN java -Djarmode=layertools -jar app.jar extract

# Stage 3: Runtime
FROM eclipse-temurin:21-jre

WORKDIR /app

RUN addgroup --system appgroup \
    && adduser --system appuser --ingroup appgroup \
    && chown -R appuser:appgroup /app

COPY --from=extract /app/dependencies/ ./
COPY --from=extract /app/spring-boot-loader/ ./
COPY --from=extract /app/snapshot-dependencies/ ./
COPY --from=extract /app/application/ ./

USER appuser

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8080/actuator/health || exit 1

EXPOSE 8080

ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-Djava.security.egd=file:/dev/./urandom", \
    "org.springframework.boot.loader.launch.JarLauncher"]
```

---

## Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.multistage
    container_name: kotlin-app
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=docker
      - SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/mydb
      - SPRING_DATASOURCE_USERNAME=myuser
      - SPRING_DATASOURCE_PASSWORD=${DB_PASSWORD}
      - SPRING_REDIS_HOST=redis
      - SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      kafka:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512m
          cpus: '0.5'

  postgres:
    image: postgres:16-alpine
    container_name: postgres
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./docker/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myuser -d mydb"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    container_name: redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network
    command: redis-server --appendonly yes

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    container_name: kafka
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      CLUSTER_ID: "MkU3OEVBNTcwNTJENDM2Qk"
    ports:
      - "9092:9092"
    volumes:
      - kafka-data:/var/lib/kafka/data
    healthcheck:
      test: ["CMD", "kafka-topics", "--bootstrap-server", "kafka:9092", "--list"]
      interval: 30s
      timeout: 10s
      retries: 5
    networks:
      - app-network

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./docker/prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"
    networks:
      - app-network

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  postgres-data:
  redis-data:
  kafka-data:
  grafana-data:
```

```yaml
# docker-compose.test.yml (for integration tests)
version: '3.8'

services:
  postgres-test:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
    ports:
      - "5433:5432"
    tmpfs:
      - /var/lib/postgresql/data  # in-memory for speed

  redis-test:
    image: redis:7-alpine
    ports:
      - "6380:6379"
```

---

## Kubernetes Basics

```yaml
# k8s/namespace.yml
apiVersion: v1
kind: Namespace
metadata:
  name: kotlin-app
  labels:
    name: kotlin-app

---
# k8s/deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kotlin-app
  namespace: kotlin-app
  labels:
    app: kotlin-app
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kotlin-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: kotlin-app
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/actuator/prometheus"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: kotlin-app-sa
      
      # Graceful shutdown
      terminationGracePeriodSeconds: 60
      
      containers:
        - name: kotlin-app
          image: myregistry/kotlin-app:1.0.0
          imagePullPolicy: Always
          
          ports:
            - containerPort: 8080
              name: http
          
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "kubernetes"
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          
          envFrom:
            - configMapRef:
                name: kotlin-app-config
            - secretRef:
                name: kotlin-app-secrets
          
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 45
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5
          
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
            failureThreshold: 3
            timeoutSeconds: 5
          
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 30
            timeoutSeconds: 5
          
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
      
      # Distribute pods across nodes
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: kotlin-app
```

---

## Service และ Ingress

```yaml
# k8s/service.yml
apiVersion: v1
kind: Service
metadata:
  name: kotlin-app
  namespace: kotlin-app
spec:
  selector:
    app: kotlin-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      name: http
  type: ClusterIP

---
# k8s/ingress.yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kotlin-app
  namespace: kotlin-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.myapp.com
      secretName: myapp-tls
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kotlin-app
                port:
                  number: 80

---
# k8s/hpa.yml (Horizontal Pod Autoscaler)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: kotlin-app-hpa
  namespace: kotlin-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: kotlin-app
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
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

---

## ConfigMap และ Secret

```yaml
# k8s/configmap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kotlin-app-config
  namespace: kotlin-app
data:
  SPRING_DATASOURCE_URL: "jdbc:postgresql://postgres-service:5432/mydb"
  SPRING_REDIS_HOST: "redis-service"
  SPRING_REDIS_PORT: "6379"
  SPRING_KAFKA_BOOTSTRAP_SERVERS: "kafka-service:9092"
  SERVER_PORT: "8080"
  MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: "health,info,metrics,prometheus"
  MANAGEMENT_ENDPOINT_HEALTH_SHOW_DETAILS: "always"
  MANAGEMENT_ENDPOINT_HEALTH_PROBES_ENABLED: "true"
  MANAGEMENT_HEALTH_LIVENESSSTATE_ENABLED: "true"
  MANAGEMENT_HEALTH_READINESSSTATE_ENABLED: "true"

---
# k8s/secret.yml (ค่าต้องเป็น base64)
apiVersion: v1
kind: Secret
metadata:
  name: kotlin-app-secrets
  namespace: kotlin-app
type: Opaque
data:
  # echo -n "value" | base64
  SPRING_DATASOURCE_USERNAME: bXl1c2Vy
  SPRING_DATASOURCE_PASSWORD: c2VjcmV0cGFzcw==
  JWT_SECRET: bXlzdXBlcnNlY3JldGtleXRoYXRpczcyY2hhcnNsb25n
  
# ใน production ใช้ External Secrets Operator หรือ Vault
```

---

## Spring Boot Health Checks Configuration

```kotlin
// Health check configuration สำหรับ Kubernetes
// application.yml
/*
management:
  endpoint:
    health:
      probes:
        enabled: true
      show-details: always
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
*/

// Custom health indicator
@Component
class DatabaseHealthIndicator(
    private val dataSource: DataSource
) : HealthIndicator {
    
    override fun health(): Health {
        return try {
            dataSource.connection.use { connection ->
                val isValid = connection.isValid(5)
                if (isValid) {
                    Health.up()
                        .withDetail("database", "PostgreSQL")
                        .withDetail("status", "connected")
                        .build()
                } else {
                    Health.down()
                        .withDetail("database", "PostgreSQL")
                        .withDetail("status", "unreachable")
                        .build()
                }
            }
        } catch (e: Exception) {
            Health.down(e)
                .withDetail("database", "PostgreSQL")
                .withDetail("error", e.message)
                .build()
        }
    }
}

@Component
class KafkaHealthIndicator(
    private val kafkaTemplate: KafkaTemplate<String, String>
) : AbstractHealthIndicator() {
    
    override fun doHealthCheck(builder: Health.Builder) {
        try {
            kafkaTemplate.defaultTopic ?: throw IllegalStateException("No default topic")
            builder.up()
                .withDetail("kafka", "connected")
        } catch (e: Exception) {
            builder.down()
                .withDetail("kafka", "error: ${e.message}")
        }
    }
}

// Graceful shutdown configuration
@Configuration
class GracefulShutdownConfig {
    
    @Bean
    fun gracefulShutdown(): GracefulShutdown = GracefulShutdown()
    
    @Bean
    fun tomcatServletWebServerFactory(
        gracefulShutdown: GracefulShutdown
    ): TomcatServletWebServerFactory {
        return TomcatServletWebServerFactory().apply {
            addConnectorCustomizers(gracefulShutdown)
        }
    }
}
```

---

## แบบฝึกหัด

```yaml
# Exercise: Deploy a complete microservices stack

# TODO: Create Kubernetes manifests for these services:
# 1. user-service (Spring Boot + PostgreSQL)
# 2. product-service (Spring Boot + PostgreSQL)
# 3. order-service (Spring Boot + PostgreSQL + Kafka)
# 4. notification-service (Spring Boot + Kafka consumer)
# 5. api-gateway (Spring Cloud Gateway)

# Requirements:
# - All services in 'microservices' namespace
# - Each service: 2-5 replicas with HPA
# - ConfigMaps for non-sensitive config
# - Secrets for credentials
# - Liveness + Readiness probes
# - Resource limits
# - Ingress for api-gateway only

# Hint: api-gateway/k8s/deployment.yml
```

```kotlin
// Kubernetes-aware Spring Boot configuration

@Configuration
@Profile("kubernetes")
class KubernetesConfig {
    
    @Value("\${POD_NAME:unknown}")
    lateinit var podName: String
    
    @Value("\${POD_NAMESPACE:default}")
    lateinit var podNamespace: String
    
    @Bean
    fun applicationInfoContributor() = InfoContributor { builder ->
        builder.withDetail("pod", mapOf(
            "name" to podName,
            "namespace" to podNamespace
        ))
    }
}

// Graceful shutdown handler
@Component
class GracefulShutdownHandler {
    
    @EventListener
    fun onShutdown(event: ContextClosedEvent) {
        println("Shutting down gracefully...")
        // Stop accepting new requests (readiness probe will fail)
        // Wait for in-flight requests to complete
        // Clean up resources
    }
}

// Readiness state management
@Component
class ReadinessManager(
    private val applicationEventPublisher: ApplicationEventPublisher
) {
    
    private var ready = false
    
    fun setReady() {
        ready = true
        applicationEventPublisher.publishEvent(
            AvailabilityChangeEvent.publish(this, ReadinessState.ACCEPTING_TRAFFIC)
        )
    }
    
    fun setNotReady(reason: String) {
        ready = false
        applicationEventPublisher.publishEvent(
            AvailabilityChangeEvent.publish(this, ReadinessState.REFUSING_TRAFFIC)
        )
        println("Service not ready: $reason")
    }
}
```

---

## Docker Build Script

```bash
#!/bin/bash
# scripts/docker-build.sh

set -e

APP_NAME="kotlin-app"
REGISTRY="myregistry.io"
VERSION=$(git describe --tags --always --dirty)
IMAGE="${REGISTRY}/${APP_NAME}:${VERSION}"
LATEST="${REGISTRY}/${APP_NAME}:latest"

echo "Building ${IMAGE}..."

# Build the Gradle project
./gradlew bootJar -x test

# Build Docker image
docker build \
    --file Dockerfile.multistage \
    --tag "${IMAGE}" \
    --tag "${LATEST}" \
    --build-arg BUILD_DATE=$(date -u +"%Y-%m-%dT%H:%M:%SZ") \
    --build-arg VERSION="${VERSION}" \
    .

echo "Build complete: ${IMAGE}"

# Optional: push to registry
if [ "$1" == "--push" ]; then
    echo "Pushing to registry..."
    docker push "${IMAGE}"
    docker push "${LATEST}"
    echo "Push complete."
fi
```

---

## สรุป Part 37

```
✅ Dockerfile: layered JAR สำหรับ fast rebuilds
✅ Multi-stage build: build + run ใน Dockerfile เดียว
✅ Non-root user: security best practice
✅ docker-compose: local development environment
✅ Kubernetes Deployment: replicas, rolling update, resource limits
✅ Liveness probe: restart ถ้า app stuck
✅ Readiness probe: traffic routing เมื่อพร้อม
✅ Startup probe: รอ app startup ก่อน check liveness
✅ HPA: auto-scale ตาม CPU/memory
✅ ConfigMap: environment config
✅ Secret: sensitive data (base64 encoded)
✅ Ingress: expose service ออก internet
✅ Graceful shutdown: รอ request เสร็จก่อน shutdown
```

---

*Part 37/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
