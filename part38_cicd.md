# Part 38: CI/CD สำหรับ Kotlin Projects

## สารบัญ
1. [GitHub Actions Basics](#github-actions-basics)
2. [Build และ Test Pipeline](#build-และ-test-pipeline)
3. [Docker Build and Push](#docker-build-and-push)
4. [Deploy to Kubernetes](#deploy-to-kubernetes)
5. [Code Quality Gates](#code-quality-gates)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## GitHub Actions Basics

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  JAVA_VERSION: '21'

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
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
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
      
      - name: Cache Gradle
        uses: actions/cache@v4
        with:
          path: |
            ~/.gradle/caches
            ~/.gradle/wrapper
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle.kts', '**/gradle-wrapper.properties') }}
          restore-keys: ${{ runner.os }}-gradle-
      
      - name: Run tests
        run: ./gradlew test --no-daemon
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: testuser
          SPRING_DATASOURCE_PASSWORD: testpass
          SPRING_REDIS_HOST: localhost
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: build/reports/tests/
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          file: build/reports/jacoco/test/jacocoTestReport.xml
          token: ${{ secrets.CODECOV_TOKEN }}
```

---

## Build และ Test Pipeline ที่สมบูรณ์

```yaml
# .github/workflows/pipeline.yml
name: Full Pipeline

on:
  push:
    branches: [ main ]
    tags: [ 'v*' ]

jobs:
  # Job 1: Code quality
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Needed for SonarCloud analysis
      
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      
      - name: Cache Gradle
        uses: actions/cache@v4
        with:
          path: ~/.gradle/caches
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle.kts') }}
      
      - name: Run Detekt
        run: ./gradlew detekt --no-daemon
      
      - name: Run ktlint
        run: ./gradlew ktlintCheck --no-daemon
      
      - name: SonarCloud analysis
        run: ./gradlew sonarqube --no-daemon
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      
      - name: Upload Detekt results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: build/reports/detekt/detekt.sarif

  # Job 2: Unit + Integration tests
  test:
    name: Test
    runs-on: ubuntu-latest
    needs: quality
    
    strategy:
      matrix:
        test-type: [unit, integration]
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      
      - name: Cache Gradle
        uses: actions/cache@v4
        with:
          path: ~/.gradle/caches
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle.kts') }}
      
      - name: Run ${{ matrix.test-type }} tests
        run: |
          if [ "${{ matrix.test-type }}" == "unit" ]; then
            ./gradlew test --no-daemon -x integrationTest
          else
            ./gradlew integrationTest --no-daemon
          fi
      
      - name: Publish Test Results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          files: build/test-results/**/*.xml

  # Job 3: Build Docker image
  build:
    name: Build
    runs-on: ubuntu-latest
    needs: test
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    
    permissions:
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      
      - name: Cache Gradle
        uses: actions/cache@v4
        with:
          path: ~/.gradle/caches
          key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle.kts') }}
      
      - name: Build JAR
        run: ./gradlew bootJar --no-daemon -x test
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata for Docker
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
            type=sha,prefix={{branch}}-
      
      - name: Build and push Docker image
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          file: Dockerfile.multistage
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64

  # Job 4: Deploy to staging
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    environment: staging
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up kubectl
        uses: azure/setup-kubectl@v4
        with:
          version: 'v1.29.0'
      
      - name: Configure kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG_STAGING }}" > ~/.kube/config
          chmod 600 ~/.kube/config
      
      - name: Deploy
        run: |
          kubectl set image deployment/kotlin-app \
            kotlin-app=${{ needs.build.outputs.image-tag }} \
            -n kotlin-app-staging
          
          kubectl rollout status deployment/kotlin-app \
            -n kotlin-app-staging \
            --timeout=5m
      
      - name: Smoke test
        run: |
          STAGING_URL="https://staging.myapp.com"
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "$STAGING_URL/actuator/health")
          if [ "$STATUS" != "200" ]; then
            echo "Smoke test failed: $STATUS"
            kubectl rollout undo deployment/kotlin-app -n kotlin-app-staging
            exit 1
          fi
          echo "Smoke test passed: $STATUS"
```

---

## Gradle Build Configuration

```kotlin
// build.gradle.kts สำหรับ CI/CD

plugins {
    kotlin("jvm") version "1.9.22"
    kotlin("plugin.spring") version "1.9.22"
    id("org.springframework.boot") version "3.2.3"
    id("io.spring.dependency-management") version "1.1.4"
    id("jacoco")
    id("io.gitlab.arturbosch.detekt") version "1.23.5"
    id("org.jlleitschuh.gradle.ktlint") version "12.1.0"
    id("org.sonarqube") version "4.4.1.3373"
}

jacoco {
    toolVersion = "0.8.11"
}

tasks.test {
    useJUnitPlatform()
    finalizedBy(tasks.jacocoTestReport)
}

tasks.jacocoTestReport {
    dependsOn(tasks.test)
    reports {
        xml.required = true
        html.required = true
    }
}

tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = "0.80".toBigDecimal()  // 80% coverage required
            }
        }
    }
}

detekt {
    config.setFrom(files("detekt.yml"))
    buildUponDefaultConfig = true
}

ktlint {
    version = "1.2.1"
    android = false
}

sonarqube {
    properties {
        property("sonar.projectKey", "myapp")
        property("sonar.organization", "myorg")
        property("sonar.host.url", "https://sonarcloud.io")
        property("sonar.coverage.jacoco.xmlReportPaths", 
            "build/reports/jacoco/test/jacocoTestReport.xml")
    }
}
```

---

## Semantic Versioning และ Release

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
      
      - name: Build
        run: ./gradlew bootJar --no-daemon -x test
      
      - name: Generate changelog
        id: changelog
        uses: orhun/git-cliff-action@v3
        with:
          config: cliff.toml
          args: --verbose --current
      
      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          body: ${{ steps.changelog.outputs.content }}
          files: |
            build/libs/*.jar
          draft: false
          prerelease: ${{ contains(github.ref_name, '-') }}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
```

---

## แบบฝึกหัด

```yaml
# Exercise: Create a complete CI/CD pipeline for a microservices project
# 
# Requirements:
# 1. Triggered on: push to main, PR to main, tags v*
# 2. Jobs in order:
#    a. lint (detekt + ktlint)
#    b. test (unit + integration, parallel)
#    c. security-scan (OWASP dependency check)
#    d. build (Docker multi-arch)
#    e. deploy-dev (auto on main push)
#    f. deploy-staging (manual approval)
#    g. deploy-prod (manual approval, only on tags)
# 3. Notifications: Slack on failure
# 4. Rollback: automatic on deployment failure

# Hint: Use GitHub environments for approvals

# .github/workflows/microservices-pipeline.yml
# TODO: Implement this pipeline

# Slack notification example:
# - name: Notify Slack on failure
#   if: failure()
#   uses: slackapi/slack-github-action@v1.26.0
#   with:
#     payload: |
#       {
#         "text": "❌ Pipeline failed: ${{ github.workflow }} on ${{ github.ref }}"
#       }
#   env:
#     SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## สรุป Part 38

```
✅ GitHub Actions: event triggers, jobs, steps
✅ Matrix strategy: run tests in parallel across types
✅ Service containers: PostgreSQL, Redis สำหรับ integration tests
✅ Docker layer cache: --cache-from/to type=gha
✅ Multi-arch builds: linux/amd64, linux/arm64
✅ Environments: staging, production with approval gates
✅ Secrets: ไม่ hardcode credentials ใน code
✅ JaCoCo: test coverage report
✅ Detekt + ktlint: code quality enforcement
✅ SonarCloud: deeper code analysis
✅ Semantic versioning: tags v1.2.3 trigger release
✅ Smoke tests: verify deployment successful
✅ Rollback: automatic on failure
```

---

*Part 38/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
