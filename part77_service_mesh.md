# Part 77: Service Mesh ด้วย Istio

## สารบัญ
1. [Service Mesh คืออะไร](#service-mesh-คืออะไร)
2. [Istio Architecture](#istio-architecture)
3. [Traffic Management](#traffic-management)
4. [Security: Mutual TLS](#security-mutual-tls)
5. [Observability ใน Istio](#observability-ใน-istio)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Service Mesh คืออะไร

```
Service Mesh: infrastructure layer สำหรับ service-to-service communication

ปัญหาที่แก้:
- Service Discovery: ค้นหา service instance ที่ available
- Load Balancing: กระจาย traffic ระหว่าง pods
- Circuit Breaking: หยุดส่ง request ไป service ที่ down
- Retry Logic: retry อัตโนมัติ
- mTLS: encrypt traffic ระหว่าง services
- Observability: distributed tracing โดยไม่ต้องแก้ code

ทำงานยังไง (Sidecar Proxy Pattern):
- Istio inject Envoy proxy เป็น sidecar container
- ทุก request ผ่าน Envoy proxy
- Envoy handle: load balance, retry, circuit break, TLS
- Application code ไม่ต้องเปลี่ยน

Control Plane:
- Istiod: จัดการ config, cert, service discovery

Data Plane:
- Envoy proxies: handle actual traffic
```

---

## Istio Architecture

```yaml
# Install Istio ด้วย istioctl

# 1. Enable Istio injection ใน namespace
---
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    istio-injection: enabled  # Auto-inject Envoy sidecar

# 2. Deployment (Envoy injected automatically)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: product-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: product-service
      version: v1
  template:
    metadata:
      labels:
        app: product-service
        version: v1
    spec:
      containers:
        - name: product-service
          image: ghcr.io/myorg/product-service:v1.2.3
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          # Application-level health checks
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5

# 3. Kubernetes Service (Istio builds on top of this)
---
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: production
spec:
  selector:
    app: product-service
  ports:
    - name: http  # Istio needs named ports!
      port: 80
      targetPort: 8080
```

---

## Traffic Management

```yaml
# VirtualService: routing rules
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service-vs
  namespace: production
spec:
  hosts:
    - product-service
  http:
    # Canary deployment: 90% v1, 10% v2
    - name: "canary-split"
      route:
        - destination:
            host: product-service
            subset: v1
          weight: 90
        - destination:
            host: product-service
            subset: v2
          weight: 10
    
    # A/B testing: route by header
    - name: "beta-users"
      match:
        - headers:
            x-user-group:
              exact: beta
      route:
        - destination:
            host: product-service
            subset: v2
    
    # Timeout and retry
    - name: "default"
      route:
        - destination:
            host: product-service
            subset: v1
      timeout: 10s
      retries:
        attempts: 3
        perTryTimeout: 3s
        retryOn: "5xx,reset,connect-failure"

---
# DestinationRule: load balancing + circuit breaking
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: product-service-dr
  namespace: production
spec:
  host: product-service
  
  trafficPolicy:
    # Load balancing: LEAST_CONN, ROUND_ROBIN, RANDOM
    loadBalancer:
      simple: LEAST_CONN
    
    # Connection pool: limit concurrent requests
    connectionPool:
      http:
        http2MaxRequests: 1000
        http1MaxPendingRequests: 100
      tcp:
        maxConnections: 100
    
    # Circuit breaker (Outlier Detection)
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
      minHealthPercent: 30
  
  # Subsets for canary
  subsets:
    - name: v1
      labels:
        version: v1
      trafficPolicy:
        loadBalancer:
          simple: ROUND_ROBIN
    - name: v2
      labels:
        version: v2

---
# Ingress Gateway
apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: api-gateway
  namespace: production
spec:
  selector:
    istio: ingressgateway
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      tls:
        mode: SIMPLE
        credentialName: api-tls-cert  # Kubernetes secret with TLS cert
      hosts:
        - "api.myapp.com"
    - port:
        number: 80
        name: http
        protocol: HTTP
      tls:
        httpsRedirect: true  # Redirect HTTP to HTTPS
      hosts:
        - "api.myapp.com"

---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: api-gateway-vs
  namespace: production
spec:
  hosts:
    - "api.myapp.com"
  gateways:
    - api-gateway
  http:
    - match:
        - uri:
            prefix: "/api/products"
      route:
        - destination:
            host: product-service
            port:
              number: 80
    - match:
        - uri:
            prefix: "/api/orders"
      route:
        - destination:
            host: order-service
            port:
              number: 80
```

---

## Security: Mutual TLS

```yaml
# PeerAuthentication: enforce mTLS between services
---
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT  # Require mTLS for all traffic in namespace

# Allow specific services to use permissive mode (during migration)
---
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: product-service-permissive
  namespace: production
spec:
  selector:
    matchLabels:
      app: product-service
  mtls:
    mode: PERMISSIVE  # Accept both plaintext and mTLS

# AuthorizationPolicy: who can talk to who
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: product-service-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: product-service
  action: ALLOW
  rules:
    # Allow order-service to call GET /api/products/**
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/order-service"
      to:
        - operation:
            methods: ["GET"]
            paths: ["/api/products/*"]
    
    # Allow frontend to call GET and POST
    - from:
        - source:
            principals:
              - "cluster.local/ns/production/sa/frontend"
      to:
        - operation:
            methods: ["GET", "POST"]
    
    # Allow ingress gateway
    - from:
        - source:
            namespaces: ["istio-system"]

# JWT validation at ingress
---
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: production
spec:
  selector:
    matchLabels:
      app: product-service
  jwtRules:
    - issuer: "https://auth.myapp.com"
      jwksUri: "https://auth.myapp.com/.well-known/jwks.json"
      audiences:
        - "api.myapp.com"
      forwardOriginalToken: true

---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: production
spec:
  selector:
    matchLabels:
      app: product-service
  action: DENY
  rules:
    - from:
        - source:
            notRequestPrincipals: ["*"]  # Deny if no JWT
      to:
        - operation:
            notPaths: ["/actuator/health"]  # Except health check
```

---

## Observability ใน Istio

```yaml
# Istio integrates with Prometheus, Grafana, Jaeger, Kiali

# Prometheus scraping Istio metrics
# metrics collected automatically from Envoy:
# - istio_requests_total
# - istio_request_duration_milliseconds
# - istio_request_bytes
# - istio_response_bytes

# Useful PromQL:
# Request rate:
# sum(rate(istio_requests_total{destination_service="product-service"}[5m]))

# Error rate (5xx):
# sum(rate(istio_requests_total{destination_service="product-service",response_code=~"5.."}[5m]))
# / sum(rate(istio_requests_total{destination_service="product-service"}[5m]))

# P99 latency:
# histogram_quantile(0.99, sum(rate(istio_request_duration_milliseconds_bucket{destination_service="product-service"}[5m])) by (le))

# Telemetry API: customize metrics/tracing
---
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: custom-metrics
  namespace: production
spec:
  metrics:
    - providers:
        - name: prometheus
      overrides:
        - match:
            metric: REQUEST_COUNT
          tagOverrides:
            destination_cluster:
              value: "my-cluster"
    
    # Add custom dimensions
    - providers:
        - name: prometheus
      overrides:
        - match:
            mode: CLIENT
          tagOverrides:
            user_tier:
              value: |
                request.headers["x-user-tier"] | "unknown"
  
  # Distributed tracing
  tracing:
    - providers:
        - name: jaeger
      randomSamplingPercentage: 1.0  # 1% sampling in production

# Fault Injection for testing circuit breaker
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-service-fault-injection
  namespace: staging
spec:
  hosts:
    - product-service
  http:
    - fault:
        delay:
          percentage:
            value: 10.0  # 10% of requests delayed
          fixedDelay: 5s
        abort:
          percentage:
            value: 5.0   # 5% of requests get HTTP 500
          httpStatus: 500
      route:
        - destination:
            host: product-service
```

---

## Kotlin Application Integration

```kotlin
// Spring Boot + Istio: propagate trace headers

@Component
class IstioTraceHeaderFilter : jakarta.servlet.Filter {
    
    // Istio trace headers that must be propagated
    private val traceHeaders = listOf(
        "x-request-id",
        "x-b3-traceid",
        "x-b3-spanid",
        "x-b3-parentspanid",
        "x-b3-sampled",
        "x-b3-flags",
        "x-ot-span-context",
        "traceparent",   // W3C trace context
        "tracestate"
    )
    
    override fun doFilter(
        request: jakarta.servlet.ServletRequest,
        response: jakarta.servlet.ServletResponse,
        chain: jakarta.servlet.FilterChain
    ) {
        val httpRequest = request as jakarta.servlet.http.HttpServletRequest
        
        // Store headers in context for outbound calls
        val headers = traceHeaders.associateWith {
            httpRequest.getHeader(it)
        }.filterValues { it != null }
        
        TraceContext.setHeaders(headers)
        
        try {
            chain.doFilter(request, response)
        } finally {
            TraceContext.clear()
        }
    }
}

object TraceContext {
    private val headers = ThreadLocal<Map<String, String?>>()
    
    fun setHeaders(h: Map<String, String?>) = headers.set(h)
    fun getHeaders() = headers.get() ?: emptyMap()
    fun clear() = headers.remove()
}

// HTTP client that propagates Istio trace headers
@Configuration
class IstioAwareHttpClientConfig {
    
    @Bean
    fun webClient(): org.springframework.web.reactive.function.client.WebClient {
        return org.springframework.web.reactive.function.client.WebClient.builder()
            .filter { request, next ->
                // Add trace headers from current context
                val headers = TraceContext.getHeaders()
                val mutated = org.springframework.web.reactive.function.client.ClientRequest
                    .from(request)
                    .apply {
                        headers.forEach { (key, value) ->
                            if (value != null) header(key, value)
                        }
                    }
                    .build()
                next.exchange(mutated)
            }
            .build()
    }
}

typealias Component = org.springframework.stereotype.Component
typealias Configuration = org.springframework.context.annotation.Configuration
typealias Bean = org.springframework.context.annotation.Bean

// Health check endpoints for Istio
@org.springframework.web.bind.annotation.RestController
class IstioHealthController {
    
    @org.springframework.web.bind.annotation.GetMapping("/health/ready")
    fun ready(): Map<String, String> = mapOf("status" to "ready")
    
    @org.springframework.web.bind.annotation.GetMapping("/health/live")
    fun live(): Map<String, String> = mapOf("status" to "alive")
    
    // Istio will call this to check if traffic should be sent
    @org.springframework.web.bind.annotation.GetMapping("/health/started")
    fun started(): Map<String, String> = mapOf("status" to "started")
}
```

---

## แบบฝึกหัด

```yaml
# Exercise: Setup Istio for e-commerce microservices

# Services:
# - product-service (v1, v2)
# - order-service
# - payment-service
# - user-service

# Requirements:
# 1. Enable mTLS for all internal traffic
# 2. Canary: send 5% to product-service v2
# 3. Circuit breaker: eject instance after 3 consecutive 5xx
# 4. Rate limiting: max 1000 req/s from order-service to payment-service
# 5. JWT validation at ingress gateway

# Solution outline:

---
# 1. Namespace-wide mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: ecommerce
spec:
  mtls:
    mode: STRICT

---
# 2. Canary: 95% v1, 5% v2
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: product-canary
  namespace: ecommerce
spec:
  hosts:
    - product-service
  http:
    - route:
        - destination:
            host: product-service
            subset: v1
          weight: 95
        - destination:
            host: product-service
            subset: v2
          weight: 5

---
# 3. Circuit breaker
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service-dr
  namespace: ecommerce
spec:
  host: payment-service
  trafficPolicy:
    outlierDetection:
      consecutiveGatewayErrors: 3
      interval: 30s
      baseEjectionTime: 60s

---
# 4. Rate limiting (EnvoyFilter)
apiVersion: networking.istio.io/v1alpha3
kind: EnvoyFilter
metadata:
  name: payment-rate-limit
  namespace: ecommerce
spec:
  workloadSelector:
    labels:
      app: payment-service
  configPatches:
    - applyTo: HTTP_FILTER
      match:
        context: SIDECAR_INBOUND
      patch:
        operation: INSERT_BEFORE
        value:
          name: envoy.filters.http.local_ratelimit
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.http.local_ratelimit.v3.LocalRateLimit
            stat_prefix: rate_limiter
            token_bucket:
              max_tokens: 1000
              tokens_per_fill: 1000
              fill_interval: 1s
            filter_enabled:
              runtime_key: local_rate_limit_enabled
              default_value:
                numerator: 100
                denominator: HUNDRED
            filter_enforced:
              runtime_key: local_rate_limit_enforced
              default_value:
                numerator: 100
                denominator: HUNDRED
```

---

## สรุป Part 77

```
✅ Service Mesh: sidecar proxy pattern (Envoy + Istiod)
✅ Istio injection: namespace label istio-injection=enabled
✅ Named ports: required for Istio protocol detection
✅ VirtualService: traffic routing rules
✅ DestinationRule: subsets, load balancing, circuit breaker
✅ Canary deployment: weight-based traffic split (90/10)
✅ Header-based routing: x-user-group: beta → v2
✅ Timeout: 10s per request
✅ Retry: 3 attempts, 3s per try, on 5xx/reset
✅ Circuit breaker: outlier detection, eject after 5 errors
✅ connectionPool: max 1000 HTTP2 requests
✅ Gateway: HTTPS with TLS certificate
✅ PeerAuthentication: STRICT mTLS between services
✅ AuthorizationPolicy: service-to-service access control
✅ RequestAuthentication: JWT validation at sidecar
✅ Istio metrics: istio_requests_total, duration histogram
✅ Fault injection: delay + abort for chaos testing
✅ Trace header propagation: x-b3-traceid, traceparent
✅ Kiali: visual service mesh topology
✅ EnvoyFilter: rate limiting at sidecar level
```

---

*Part 77/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
