# Part 93: DevOps, CI/CD และ Container Orchestration

## สารบัญ
1. [Docker Multi-stage Build](#docker-multi-stage-build)
2. [GitHub Actions CI/CD Pipeline](#github-actions-cicd-pipeline)
3. [Kubernetes Production Setup](#kubernetes-production-setup)
4. [Helm Charts](#helm-charts)
5. [GitOps ด้วย ArgoCD](#gitops-ด้วย-argocd)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Docker Multi-stage Build

```dockerfile
# Dockerfile (multi-stage สำหรับ Spring Boot)
# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /build

# Copy gradle files first (layer caching)
COPY gradle/ gradle/
COPY gradlew build.gradle.kts settings.gradle.kts ./

# Download dependencies (cached layer)
RUN ./gradlew dependencies --no-daemon 2>/dev/null || true

# Copy source
COPY src/ src/

# Build
RUN ./gradlew bootJar --no-daemon -x test

# Stage 2: Extract layers (Spring Boot layered jar)
FROM builder AS extractor
WORKDIR /build
RUN java -Djarmode=layertools -jar build/libs/*.jar extract --destination ./extracted

# Stage 3: Runtime
FROM eclipse-temurin:21-jre-alpine AS runtime

# Security: create non-root user
RUN addgroup -S spring && adduser -S spring -G spring

WORKDIR /app

# Copy layers in order (most stable → least stable)
COPY --from=extractor --chown=spring:spring /build/extracted/dependencies ./
COPY --from=extractor --chown=spring:spring /build/extracted/spring-boot-loader ./
COPY --from=extractor --chown=spring:spring /build/extracted/snapshot-dependencies ./
COPY --from=extractor --chown=spring:spring /build/extracted/application ./

USER spring

# JVM settings for containers
ENV JAVA_OPTS="\
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=200 \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp/heap-dump.hprof \
  -Djava.security.egd=file:/dev/./urandom"

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

```kotlin
// build.gradle.kts: Spring Boot layered jar configuration
import org.springframework.boot.gradle.tasks.bundling.BootJar

tasks.named<BootJar>("bootJar") {
    layered {
        application {
            intoLayer("spring-boot-loader") {
                include("org/springframework/boot/loader/**")
            }
            intoLayer("application")
        }
        dependencies {
            intoLayer("snapshot-dependencies") {
                include("*:*:*SNAPSHOT")
            }
            intoLayer("dependencies")
        }
        layerOrder.set(
            listOf("dependencies", "spring-boot-loader", "snapshot-dependencies", "application")
        )
    }
}

// Docker Compose for local development
/*
version: '3.9'
services:
  app:
    build:
      context: .
      target: runtime
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: local
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/ecommerce
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      SPRING_DATA_REDIS_HOST: redis
    depends_on:
      postgres:
        condition: service_healthy
      kafka:
        condition: service_healthy
    volumes:
      - /tmp:/tmp  # for heap dumps

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ecommerce
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d ecommerce"]
      interval: 10s
      timeout: 5s
      retries: 5
    volumes:
      - postgres_data:/var/lib/postgresql/data

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    healthcheck:
      test: kafka-topics --bootstrap-server localhost:9092 --list
      interval: 30s
      timeout: 10s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

volumes:
  postgres_data:
*/
```

---

## GitHub Actions CI/CD Pipeline

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  JAVA_VERSION: '21'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # Job 1: Test
  test:
    name: Test
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'gradle'
      
      - name: Grant execute permission
        run: chmod +x gradlew
      
      - name: Run tests
        run: ./gradlew test --no-daemon
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: test
          SPRING_DATASOURCE_PASSWORD: test
          SPRING_DATA_REDIS_HOST: localhost
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: build/reports/tests/
      
      - name: Upload coverage report
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          file: build/reports/jacoco/test/jacocoTestReport.xml

  # Job 2: Code Quality
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'gradle'
      
      - name: Lint with ktlint
        run: ./gradlew ktlintCheck --no-daemon
      
      - name: Static analysis with detekt
        run: ./gradlew detekt --no-daemon
      
      - name: SonarQube scan
        run: ./gradlew sonar --no-daemon
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      
      - name: Dependency vulnerability check
        run: ./gradlew dependencyCheckAnalyze --no-daemon
        continue-on-error: true

  # Job 3: Build & Push Docker image
  build:
    name: Build & Push
    runs-on: ubuntu-latest
    needs: [test, quality]
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    
    permissions:
      contents: read
      packages: write
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build-push.outputs.digest }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix=sha-
            type=semver,pattern={{version}}
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}
      
      - name: Build and Push
        id: build-push
        uses: docker/build-push-action@v5
        with:
          context: .
          target: runtime
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64
      
      - name: Sign image
        uses: sigstore/cosign-installer@v3
      
      - name: Sign with cosign
        run: |
          cosign sign --yes \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build-push.outputs.digest }}

  # Job 4: Deploy to staging
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/develop'
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.28.0'
      
      - name: Configure kubectl
        run: |
          echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > ~/.kube/config
      
      - name: Update image tag
        run: |
          kubectl set image deployment/ecommerce-api \
            api=${{ needs.build.outputs.image-tag }} \
            -n staging
      
      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/ecommerce-api -n staging --timeout=300s
      
      - name: Run smoke tests
        run: |
          STAGING_URL="https://staging.ecommerce.example.com"
          curl -f "$STAGING_URL/actuator/health" || exit 1
          curl -f "$STAGING_URL/api/v1/products" || exit 1

  # Job 5: Deploy to production (manual approval)
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [build, deploy-staging]
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://ecommerce.example.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
      
      - name: Configure kubectl
        run: echo "${{ secrets.KUBE_CONFIG_PROD }}" | base64 -d > ~/.kube/config
      
      - name: Blue-Green Deploy
        run: |
          # Switch traffic to new version
          kubectl set image deployment/ecommerce-api-green \
            api=${{ needs.build.outputs.image-tag }} \
            -n production
          
          kubectl rollout status deployment/ecommerce-api-green -n production --timeout=600s
          
          # Verify green is healthy
          GREEN_PODS=$(kubectl get pods -n production -l app=ecommerce-api,slot=green \
            -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].status}')
          
          if echo "$GREEN_PODS" | grep -q "False"; then
            echo "Green deployment unhealthy, rolling back"
            kubectl rollout undo deployment/ecommerce-api-green -n production
            exit 1
          fi
          
          # Switch service to green
          kubectl patch service ecommerce-api-svc -n production \
            -p '{"spec":{"selector":{"slot":"green"}}}'
      
      - name: Notify deployment
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: |
            Deployed to production: ${{ needs.build.outputs.image-tag }}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Kubernetes Production Setup

```yaml
# k8s/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ecommerce-api
  labels:
    app: ecommerce-api
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ecommerce-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # zero downtime
  template:
    metadata:
      labels:
        app: ecommerce-api
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      serviceAccountName: ecommerce-api
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values: [ecommerce-api]
                topologyKey: kubernetes.io/hostname
      
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: ecommerce-api
      
      containers:
        - name: api
          image: ghcr.io/org/ecommerce-api:latest
          imagePullPolicy: IfNotPresent
          
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 8081
              name: management
          
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: production
            - name: JAVA_OPTS
              value: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: ecommerce-secrets
                  key: db-password
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
                name: ecommerce-config
          
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8081
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8081
            initialDelaySeconds: 60
            periodSeconds: 30
            failureThreshold: 3
          
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8081
            failureThreshold: 30
            periodSeconds: 10
          
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 10"]
          
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
          
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      
      volumes:
        - name: tmp
          emptyDir: {}
      
      terminationGracePeriodSeconds: 60

---
# k8s/base/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ecommerce-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ecommerce-api
  minReplicas: 3
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
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: 1k
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
```

---

## Helm Charts

```yaml
# helm/ecommerce/Chart.yaml
apiVersion: v2
name: ecommerce
description: E-commerce microservice
version: 1.0.0
appVersion: "1.0.0"

# helm/ecommerce/values.yaml
replicaCount: 3
image:
  repository: ghcr.io/org/ecommerce-api
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
  hosts:
    - host: api.ecommerce.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: ecommerce-tls
      hosts:
        - api.ecommerce.example.com

resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70

postgresql:
  enabled: true
  auth:
    database: ecommerce
    username: app
    existingSecret: ecommerce-db-secret

redis:
  enabled: true
  architecture: standalone
  auth:
    enabled: true
    existingSecret: ecommerce-redis-secret
```

---

## GitOps ด้วย ArgoCD

```yaml
# argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: ecommerce-production
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: ecommerce
  source:
    repoURL: https://github.com/org/ecommerce-k8s-configs
    targetRevision: main
    path: environments/production
    helm:
      valueFiles:
        - values.yaml
        - values-production.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true        # remove resources not in git
      selfHeal: true     # revert manual changes
      allowEmpty: false
    syncOptions:
      - Validate=true
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - RespectIgnoreDifferences=true
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas  # HPA controls replicas
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง Kotlin app ที่ Kubernetes-ready

// 1. Health indicators สำหรับ readiness/liveness
@org.springframework.stereotype.Component
class DatabaseHealthIndicator(
    private val dataSource: javax.sql.DataSource
) : org.springframework.boot.actuate.health.HealthIndicator {
    
    override fun health(): org.springframework.boot.actuate.health.Health {
        return try {
            dataSource.connection.use { conn ->
                conn.prepareStatement("SELECT 1").executeQuery()
            }
            org.springframework.boot.actuate.health.Health.up()
                .withDetail("database", "PostgreSQL")
                .build()
        } catch (e: Exception) {
            org.springframework.boot.actuate.health.Health.down()
                .withException(e)
                .build()
        }
    }
}

// 2. Graceful shutdown: ปฏิเสธ new requests ก่อน shutdown
@org.springframework.stereotype.Component
class GracefulShutdownHandler(
    private val registry: org.springframework.kafka.config.KafkaListenerEndpointRegistry
) : org.springframework.context.ApplicationListener<org.springframework.context.event.ContextClosedEvent> {
    
    private val log = org.slf4j.LoggerFactory.getLogger(GracefulShutdownHandler::class.java)
    
    override fun onApplicationEvent(event: org.springframework.context.event.ContextClosedEvent) {
        log.info("Shutting down gracefully...")
        
        // Stop consuming new Kafka messages
        registry.stop()
        
        // Wait for in-flight requests (preStop: sleep 10 handles this)
        Thread.sleep(5000)
        
        log.info("Graceful shutdown complete")
    }
}

// 3. Structured logging for Kubernetes log aggregation
@org.springframework.stereotype.Component
class RequestLoggingFilter : org.springframework.web.filter.OncePerRequestFilter() {
    
    override fun doFilterInternal(
        request: javax.servlet.http.HttpServletRequest,
        response: javax.servlet.http.HttpServletResponse,
        filterChain: javax.servlet.FilterChain
    ) {
        val start = System.currentTimeMillis()
        val traceId = request.getHeader("X-Trace-ID") ?: java.util.UUID.randomUUID().toString()
        
        org.slf4j.MDC.put("traceId", traceId)
        org.slf4j.MDC.put("method", request.method)
        org.slf4j.MDC.put("path", request.requestURI)
        
        try {
            filterChain.doFilter(request, response)
        } finally {
            val duration = System.currentTimeMillis() - start
            org.slf4j.MDC.put("statusCode", response.status.toString())
            org.slf4j.MDC.put("duration", duration.toString())
            
            org.slf4j.LoggerFactory.getLogger(RequestLoggingFilter::class.java)
                .info("HTTP ${request.method} ${request.requestURI} ${response.status} ${duration}ms")
            
            org.slf4j.MDC.clear()
        }
    }
}
```

---

## สรุป Part 93

```
✅ Multi-stage Docker: builder → extractor → runtime
✅ Layered jar: dependencies/loader/snapshot/application layers
✅ Non-root user: adduser spring, USER spring
✅ HEALTHCHECK: wget actuator/health
✅ JAVA_OPTS: UseContainerSupport, MaxRAMPercentage=75
✅ Docker Compose: depends_on with healthcheck conditions
✅ GitHub Actions: on.push, services, jobs, needs
✅ services: postgres/redis containers in CI
✅ DynamicPropertySource equivalence: env vars in GHA
✅ docker/metadata-action: auto-tags (branch, sha, semver)
✅ docker/build-push-action: multi-platform, layer cache
✅ sigstore/cosign: image signing for supply chain security
✅ environment: staging/production (manual approval gate)
✅ kubectl set image: rolling update with new tag
✅ Blue-Green deploy: green slot → switch service selector
✅ K8s Deployment: RollingUpdate maxUnavailable=0
✅ topologySpreadConstraints: spread across zones
✅ podAntiAffinity: avoid same node
✅ resources.requests/limits: CPU/memory bounds
✅ readinessProbe: /actuator/health/readiness
✅ livenessProbe: /actuator/health/liveness
✅ startupProbe: longer initialDelay for boot
✅ lifecycle.preStop: sleep 10 for graceful drain
✅ readOnlyRootFilesystem: security hardening
✅ HPA v2: CPU + memory + custom metric
✅ scaleDown stabilizationWindowSeconds: avoid thrashing
✅ Helm: Chart.yaml, values.yaml parameterization
✅ ArgoCD: GitOps, automated sync with prune/selfHeal
✅ ignoreDifferences: skip HPA-managed replicas
✅ HealthIndicator: custom database health
✅ GracefulShutdownHandler: stop Kafka before exit
✅ MDC logging: traceId, method, path, statusCode, duration
```

---

*Part 93/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
