# Part 74: CI/CD ด้วย GitHub Actions

## สารบัญ
1. [GitHub Actions Fundamentals](#github-actions-fundamentals)
2. [Kotlin Build Pipeline](#kotlin-build-pipeline)
3. [Docker Build and Push](#docker-build-and-push)
4. [Deployment Pipeline](#deployment-pipeline)
5. [Security Scanning](#security-scanning)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## GitHub Actions Fundamentals

```yaml
# .github/workflows/ci.yml
# Basic concepts:
# - Workflow: ไฟล์ YAML ที่กำหนด automation
# - Trigger (on:): เมื่อไหร่ให้ run
# - Job: group of steps ที่ run บน runner เดียวกัน
# - Step: single task ใน job
# - Action: reusable unit (uses: actions/checkout@v4)
# - Runner: machine ที่ run jobs (ubuntu-latest, etc.)

name: CI Pipeline

on:
  push:
    branches: [main, develop]
    paths-ignore:
      - '**.md'
      - 'docs/**'
  pull_request:
    branches: [main, develop]
    types: [opened, synchronize, reopened]
  workflow_dispatch:  # Manual trigger

# Cancel in-progress runs on new push to same branch
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  JAVA_VERSION: '21'
  GRADLE_OPTS: '-Dorg.gradle.daemon=false -Dorg.gradle.parallel=true'

jobs:
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
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for Sonar
      
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'gradle'
      
      - name: Grant execute permission for gradlew
        run: chmod +x gradlew
      
      - name: Run tests
        run: ./gradlew test --info
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: test
          SPRING_DATASOURCE_PASSWORD: test
          SPRING_REDIS_HOST: localhost
          SPRING_REDIS_PORT: 6379
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: build/reports/tests/
          retention-days: 7
      
      - name: Generate test coverage report
        run: ./gradlew jacocoTestReport
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: build/reports/jacoco/test/jacocoTestReport.xml

  lint:
    name: Code Quality
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'gradle'
      
      - name: Run ktlint
        run: ./gradlew ktlintCheck
      
      - name: Run detekt
        run: ./gradlew detekt
      
      - name: Run OWASP Dependency Check
        run: ./gradlew dependencyCheckAnalyze
        env:
          NVD_API_KEY: ${{ secrets.NVD_API_KEY }}
      
      - name: Upload dependency check report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: dependency-check-report
          path: build/reports/dependency-check-report.html
```

---

## Kotlin Build Pipeline

```yaml
# .github/workflows/build.yml

name: Build and Publish

on:
  push:
    branches: [main]
    tags:
      - 'v*'

jobs:
  build:
    name: Build
    runs-on: ubuntu-latest
    
    outputs:
      version: ${{ steps.version.outputs.version }}
      image-tag: ${{ steps.meta.outputs.tags }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Determine version
        id: version
        run: |
          if [[ "${{ github.ref }}" == refs/tags/* ]]; then
            VERSION="${{ github.ref_name }}"
          else
            SHORT_SHA="${{ github.sha }}"
            VERSION="0.0.0-${SHORT_SHA:0:7}"
          fi
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "Building version: $VERSION"
      
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'
      
      # Gradle Build Cache
      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v3
        with:
          cache-read-only: ${{ github.ref != 'refs/heads/main' }}
      
      - name: Build JAR
        run: |
          ./gradlew build -x test \
            -Pversion=${{ steps.version.outputs.version }}
      
      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: build/libs/*.jar
          if-no-files-found: error
  
  docker:
    name: Docker Build & Push
    runs-on: ubuntu-latest
    needs: build
    
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Download JAR
        uses: actions/download-artifact@v4
        with:
          name: app-jar
          path: build/libs/
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix=sha-,format=short
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            APP_VERSION=${{ needs.build.outputs.version }}
            BUILD_DATE=${{ github.event.repository.updated_at }}
            VCS_REF=${{ github.sha }}
          platforms: linux/amd64,linux/arm64  # Multi-arch build
```

---

## Dockerfile สำหรับ Kotlin/Spring

```dockerfile
# Multi-stage Dockerfile for Kotlin Spring application

# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /build

# Copy only dependency files first (better cache utilization)
COPY gradle/ gradle/
COPY gradlew settings.gradle.kts build.gradle.kts ./
RUN chmod +x gradlew && ./gradlew dependencies --no-daemon || true

# Copy source and build
COPY src/ src/
RUN ./gradlew build -x test --no-daemon

# Extract layers for better Docker caching
RUN java -Djarmode=layertools \
    -jar build/libs/*.jar extract --destination /extracted

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine AS runtime

# Security: non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# JVM optimizations
ENV JAVA_OPTS="-XX:+UseContainerSupport \
               -XX:MaxRAMPercentage=75.0 \
               -XX:+UseG1GC \
               -XX:MaxGCPauseMillis=200 \
               -XX:+ExitOnOutOfMemoryError \
               -Djava.security.egd=file:/dev/./urandom"

# Copy layers (ordered by change frequency)
COPY --from=builder --chown=appuser:appgroup /extracted/dependencies/ ./
COPY --from=builder --chown=appuser:appgroup /extracted/spring-boot-loader/ ./
COPY --from=builder --chown=appuser:appgroup /extracted/snapshot-dependencies/ ./
COPY --from=builder --chown=appuser:appgroup /extracted/application/ ./

USER appuser

EXPOSE 8080

# Graceful shutdown support
STOPSIGNAL SIGTERM

HEALTHCHECK --interval=30s --timeout=3s --start-period=60s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

---

## Deployment Pipeline

```yaml
# .github/workflows/deploy.yml
# GitOps-style deployment

name: Deploy

on:
  workflow_run:
    workflows: ["Build and Publish"]
    types: [completed]
    branches: [main]

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    environment:
      name: staging
      url: https://staging.myapp.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Get image tag
        id: image
        run: |
          SHA="${{ github.event.workflow_run.head_sha }}"
          echo "tag=sha-${SHA:0:7}" >> $GITHUB_OUTPUT
      
      - name: Set up kubectl
        uses: azure/setup-kubectl@v4
      
      - name: Configure kubeconfig
        run: |
          echo "${{ secrets.KUBECONFIG_STAGING }}" | base64 -d > kubeconfig.yml
        env:
          KUBECONFIG: kubeconfig.yml
      
      - name: Deploy to staging
        run: |
          helm upgrade --install myapp ./helm/myapp \
            --namespace staging \
            --create-namespace \
            --set image.tag=${{ steps.image.outputs.tag }} \
            --set image.repository=ghcr.io/${{ github.repository }} \
            --values helm/values-staging.yaml \
            --wait \
            --timeout 10m \
            --atomic
      
      - name: Run smoke tests
        run: |
          # Wait for deployment to be ready
          kubectl rollout status deployment/myapp -n staging --timeout=5m
          
          # Basic health check
          curl -f https://staging.myapp.com/actuator/health || exit 1
          
          # Run integration tests against staging
          ./gradlew integrationTest \
            -Dapp.url=https://staging.myapp.com
      
      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: 'deployments'
          slack-message: |
            ❌ Staging deployment failed!
            Branch: ${{ github.ref_name }}
            Commit: ${{ github.sha }}
            Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
  
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    environment:
      name: production
      url: https://myapp.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Get image tag
        id: image
        run: |
          SHA="${{ github.event.workflow_run.head_sha }}"
          echo "tag=sha-${SHA:0:7}" >> $GITHUB_OUTPUT
      
      # Blue-Green Deployment
      - name: Deploy with blue-green strategy
        run: |
          # Deploy to inactive slot
          ACTIVE_SLOT=$(kubectl get service myapp -n production \
            -o jsonpath='{.spec.selector.slot}' 2>/dev/null || echo "blue")
          INACTIVE_SLOT=$([ "$ACTIVE_SLOT" == "blue" ] && echo "green" || echo "blue")
          
          echo "Active: $ACTIVE_SLOT, Deploying to: $INACTIVE_SLOT"
          
          # Deploy to inactive
          helm upgrade --install myapp-$INACTIVE_SLOT ./helm/myapp \
            --namespace production \
            --set image.tag=${{ steps.image.outputs.tag }} \
            --set slot=$INACTIVE_SLOT \
            --values helm/values-production.yaml \
            --wait --atomic
          
          # Smoke test inactive slot
          curl -f https://production-$INACTIVE_SLOT.myapp.internal/actuator/health
          
          # Switch traffic to new slot
          kubectl patch service myapp -n production \
            --type='json' \
            -p='[{"op":"replace","path":"/spec/selector/slot","value":"'$INACTIVE_SLOT'"}]'
          
          echo "Traffic switched to $INACTIVE_SLOT"
      
      - name: Notify Slack on success
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          channel-id: 'deployments'
          slack-message: |
            ✅ Production deployment successful!
            Version: ${{ steps.image.outputs.tag }}
            URL: https://myapp.com
        env:
          SLACK_BOT_TOKEN: ${{ secrets.SLACK_BOT_TOKEN }}
```

---

## Security Scanning

```yaml
# .github/workflows/security.yml

name: Security Scan

on:
  schedule:
    - cron: '0 6 * * 1'  # Every Monday at 6am UTC
  push:
    branches: [main]

jobs:
  sast:
    name: Static Analysis (SAST)
    runs-on: ubuntu-latest
    
    permissions:
      security-events: write
    
    steps:
      - uses: actions/checkout@v4
      
      # Semgrep SAST scan
      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/java
            p/kotlin
            p/owasp-top-ten
            p/secrets
          generateSarif: '1'
      
      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: semgrep.sarif
      
      # CodeQL Analysis
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: java-kotlin
          queries: security-and-quality
      
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'gradle'
      
      - name: Build for CodeQL
        run: ./gradlew build -x test
      
      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
  
  container-scan:
    name: Container Vulnerability Scan
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Build Docker image
        run: docker build -t myapp:scan .
      
      # Trivy vulnerability scanner
      - name: Run Trivy scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:scan'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
          ignore-unfixed: true
      
      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
  
  secrets-scan:
    name: Secrets Detection
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      # Gitleaks: find hardcoded secrets
      - name: Run Gitleaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_NOTIFY_USER_LIST: "@security-team"
```

---

## แบบฝึกหัด

```yaml
# Exercise: เพิ่ม performance testing ใน pipeline

# Requirements:
# 1. Run Gatling load tests after deployment to staging
# 2. Fail pipeline if p95 latency > 500ms
# 3. Fail pipeline if error rate > 1%
# 4. Store Gatling report as artifact
# 5. Post results to Slack

# .github/workflows/performance-test.yml

name: Performance Test

on:
  workflow_call:
    inputs:
      target-url:
        required: true
        type: string
      duration:
        required: false
        type: string
        default: '60'
      users:
        required: false
        type: string
        default: '50'
    secrets:
      SLACK_BOT_TOKEN:
        required: false

jobs:
  gatling:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      
      - name: Run Gatling simulation
        run: |
          ./gradlew gatlingRun \
            -Dgatling.simulationClass=com.example.simulations.ApiSimulation \
            -Dtarget.url=${{ inputs.target-url }} \
            -Dduration=${{ inputs.duration }} \
            -Dusers=${{ inputs.users }}
      
      - name: Parse Gatling results
        id: results
        run: |
          # Parse simulation.log for key metrics
          REPORT_DIR=$(ls -d build/reports/gatling/*/ | head -1)
          
          P95=$(grep "p95" $REPORT_DIR/simulation.log | tail -1 | awk '{print $5}')
          ERROR_RATE=$(grep "KO" $REPORT_DIR/simulation.log | tail -1 | awk '{print $7}')
          
          echo "p95=$P95" >> $GITHUB_OUTPUT
          echo "error-rate=$ERROR_RATE" >> $GITHUB_OUTPUT
          
          echo "P95 Latency: ${P95}ms"
          echo "Error Rate: ${ERROR_RATE}%"
          
          # Fail if thresholds exceeded
          if [ "$P95" -gt 500 ]; then
            echo "❌ P95 latency ${P95}ms exceeds 500ms threshold"
            exit 1
          fi
          
          if [ "$(echo "$ERROR_RATE > 1" | bc -l)" -eq 1 ]; then
            echo "❌ Error rate ${ERROR_RATE}% exceeds 1% threshold"
            exit 1
          fi
      
      - name: Upload Gatling report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: gatling-report
          path: build/reports/gatling/
          retention-days: 30
      
      - name: Post results to Slack
        if: always()
        run: |
          STATUS="${{ job.status }}"
          EMOJI=$([ "$STATUS" == "success" ] && echo "✅" || echo "❌")
          
          curl -X POST \
            -H "Authorization: Bearer ${{ secrets.SLACK_BOT_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{
              "channel": "performance",
              "text": "'$EMOJI' Performance Test: '$STATUS'\nURL: ${{ inputs.target-url }}\nP95: ${{ steps.results.outputs.p95 }}ms\nError Rate: ${{ steps.results.outputs.error-rate }}%"
            }' \
            https://slack.com/api/chat.postMessage
```

---

## สรุป Part 74

```
✅ GitHub Actions: workflows, jobs, steps, actions, runners
✅ Concurrency: cancel-in-progress on new push
✅ Services: postgres/redis containers in CI
✅ health-check options: --health-cmd pg_isready
✅ actions/cache: Gradle dependency caching
✅ Upload/Download artifacts: test reports, JAR files
✅ Codecov: test coverage reporting
✅ ktlint/detekt: Kotlin code quality
✅ OWASP Dependency Check: vulnerability scanning
✅ Multi-stage Dockerfile: builder → runtime
✅ Layer extraction: spring layertools for better caching
✅ Non-root user: security best practice
✅ Multi-arch: linux/amd64,linux/arm64
✅ Semantic versioning: git tag → image tag
✅ Docker metadata action: automatic tagging
✅ GitHub Container Registry: ghcr.io
✅ GitOps deployment: workflow_run trigger
✅ Blue-Green deployment: zero-downtime switch
✅ Helm atomic: rollback on failure
✅ SAST: Semgrep + CodeQL
✅ Container scanning: Trivy vulnerability scanner
✅ Secrets detection: Gitleaks
✅ SARIF: upload security findings to GitHub Security tab
✅ Performance testing: Gatling in CI pipeline
```

---

*Part 74/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
