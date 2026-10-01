# Part 83: Advanced Security — OAuth2 & OpenID Connect

## สารบัญ
1. [OAuth2 Flows](#oauth2-flows)
2. [Authorization Server ด้วย Spring Authorization Server](#authorization-server)
3. [PKCE Flow สำหรับ Mobile/SPA](#pkce-flow)
4. [Resource Server Configuration](#resource-server)
5. [OpenID Connect (OIDC)](#openid-connect)
6. [Security Best Practices](#security-best-practices)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## OAuth2 Flows

```
OAuth2 Grant Types ที่ใช้งานบ่อย:

1. Authorization Code + PKCE (สำหรับ SPA และ Mobile)
   User → App → Auth Server → User Login → App ← Auth Code → App ↔ Token

2. Client Credentials (สำหรับ Service-to-Service)
   Service A → Auth Server → Access Token → Service B

3. Refresh Token (ต่ออายุ token)
   App → Auth Server (refresh_token) → New Access Token

4. Device Code (สำหรับ Smart TV, CLI)
   Device → Auth Server → User Code → User → Browser → App

PKCE (Proof Key for Code Exchange):
- สร้าง code_verifier แบบ random (43-128 chars)
- คำนวณ code_challenge = BASE64URL(SHA256(code_verifier))
- ส่ง code_challenge ไปกับ authorization request
- ส่ง code_verifier ไปกับ token request
- ป้องกัน authorization code interception attack

Token Types:
- Access Token: short-lived (15min-1h), ใช้เข้า API
- Refresh Token: long-lived (7-30d), ใช้ขอ Access Token ใหม่
- ID Token: JWT ที่มีข้อมูล user (OIDC เท่านั้น)
```

---

## Authorization Server

```kotlin
// build.gradle.kts (Authorization Server)
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.security:spring-security-oauth2-authorization-server:1.2.1")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")
    implementation("com.nimbusds:nimbus-jose-jwt:9.37")
}

// AuthorizationServerConfig.kt
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.security.oauth2.server.authorization.config.annotation.web.configuration.OAuth2AuthorizationServerConfiguration
import org.springframework.security.oauth2.server.authorization.config.annotation.web.configurers.OAuth2AuthorizationServerConfigurer
import org.springframework.security.oauth2.server.authorization.settings.AuthorizationServerSettings
import org.springframework.security.oauth2.server.authorization.settings.ClientSettings
import org.springframework.security.oauth2.server.authorization.settings.TokenSettings
import org.springframework.security.oauth2.core.AuthorizationGrantType
import org.springframework.security.oauth2.core.ClientAuthenticationMethod
import org.springframework.security.oauth2.core.oidc.OidcScopes
import org.springframework.security.oauth2.server.authorization.client.RegisteredClient
import org.springframework.security.oauth2.server.authorization.client.RegisteredClientRepository
import org.springframework.security.oauth2.server.authorization.client.InMemoryRegisteredClientRepository
import java.time.Duration

@Configuration
class AuthorizationServerConfig {
    
    @Bean
    fun registeredClientRepository(): RegisteredClientRepository {
        // Web Application Client (Authorization Code + PKCE)
        val webClient = RegisteredClient.withId(java.util.UUID.randomUUID().toString())
            .clientId("ecommerce-web")
            .clientSecret("{noop}web-secret")  // Use BCrypt in production
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .redirectUri("https://app.ecommerce.com/callback")
            .redirectUri("https://app.ecommerce.com/silent-renew")
            .postLogoutRedirectUri("https://app.ecommerce.com/logout")
            .scope(OidcScopes.OPENID)
            .scope(OidcScopes.PROFILE)
            .scope(OidcScopes.EMAIL)
            .scope("products:read")
            .scope("orders:write")
            .clientSettings(
                ClientSettings.builder()
                    .requireAuthorizationConsent(false)
                    .requireProofKey(true)  // Require PKCE
                    .build()
            )
            .tokenSettings(
                TokenSettings.builder()
                    .accessTokenTimeToLive(Duration.ofMinutes(15))
                    .refreshTokenTimeToLive(Duration.ofDays(7))
                    .reuseRefreshTokens(false)
                    .build()
            )
            .build()
        
        // Mobile App Client (PKCE without client secret)
        val mobileClient = RegisteredClient.withId(java.util.UUID.randomUUID().toString())
            .clientId("ecommerce-mobile")
            .clientAuthenticationMethod(ClientAuthenticationMethod.NONE)  // Public client
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            .redirectUri("com.ecommerce.app://callback")  // Custom scheme
            .scope(OidcScopes.OPENID)
            .scope(OidcScopes.PROFILE)
            .scope("orders:write")
            .clientSettings(
                ClientSettings.builder()
                    .requireProofKey(true)
                    .build()
            )
            .tokenSettings(
                TokenSettings.builder()
                    .accessTokenTimeToLive(Duration.ofHours(1))
                    .refreshTokenTimeToLive(Duration.ofDays(30))
                    .build()
            )
            .build()
        
        // Service Client (Client Credentials)
        val serviceClient = RegisteredClient.withId(java.util.UUID.randomUUID().toString())
            .clientId("notification-service")
            .clientSecret("{bcrypt}\$2a\$12\$abc...")
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
            .scope("notifications:send")
            .scope("users:read")
            .tokenSettings(
                TokenSettings.builder()
                    .accessTokenTimeToLive(Duration.ofMinutes(30))
                    .build()
            )
            .build()
        
        return InMemoryRegisteredClientRepository(webClient, mobileClient, serviceClient)
    }
    
    @Bean
    fun jwkSource(): com.nimbusds.jose.jwk.source.JWKSource<com.nimbusds.jose.proc.SecurityContext> {
        val rsaKey = com.nimbusds.jose.jwk.gen.RSAKeyGenerator(2048)
            .keyID(java.util.UUID.randomUUID().toString())
            .generate()
        
        val jwkSet = com.nimbusds.jose.jwk.JWKSet(rsaKey)
        return com.nimbusds.jose.jwk.source.ImmutableJWKSet(jwkSet)
    }
    
    @Bean
    fun jwtDecoder(jwkSource: com.nimbusds.jose.jwk.source.JWKSource<com.nimbusds.jose.proc.SecurityContext>): 
        org.springframework.security.oauth2.jwt.JwtDecoder {
        return OAuth2AuthorizationServerConfiguration.jwtDecoder(jwkSource)
    }
    
    @Bean
    fun authorizationServerSettings(): AuthorizationServerSettings {
        return AuthorizationServerSettings.builder()
            .issuer("https://auth.ecommerce.com")
            .authorizationEndpoint("/oauth2/authorize")
            .tokenEndpoint("/oauth2/token")
            .tokenIntrospectionEndpoint("/oauth2/introspect")
            .tokenRevocationEndpoint("/oauth2/revoke")
            .jwkSetEndpoint("/oauth2/jwks")
            .oidcUserInfoEndpoint("/userinfo")
            .build()
    }
    
    // Add custom claims to JWT
    @Bean
    fun tokenCustomizer(userService: UserService): 
        org.springframework.security.oauth2.server.authorization.token.OAuth2TokenCustomizer<
            org.springframework.security.oauth2.server.authorization.token.JwtEncodingContext> {
        return org.springframework.security.oauth2.server.authorization.token.OAuth2TokenCustomizer { context ->
            if (context.tokenType == org.springframework.security.oauth2.server.authorization.OAuth2TokenType.ACCESS_TOKEN) {
                val principal = context.getPrincipal<org.springframework.security.authentication.UsernamePasswordAuthenticationToken>()
                val user = userService.findByUsername(principal.name)
                
                context.claims.apply {
                    claim("roles", user.roles)
                    claim("tenantId", user.tenantId)
                    claim("name", user.fullName)
                }
            }
        }
    }
}

// Security filter chain for Authorization Server
@org.springframework.context.annotation.Configuration
class SecurityConfig {
    
    @org.springframework.context.annotation.Bean
    @org.springframework.core.annotation.Order(1)
    fun authorizationServerSecurityFilterChain(
        http: org.springframework.security.config.annotation.web.builders.HttpSecurity
    ): org.springframework.security.web.SecurityFilterChain {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http)
        
        http.getConfigurer(OAuth2AuthorizationServerConfigurer::class.java)
            .oidc(org.springframework.security.config.Customizer.withDefaults())
        
        http.exceptionHandling {
            it.defaultAuthenticationEntryPointFor(
                org.springframework.security.web.authentication.LoginUrlAuthenticationEntryPoint("/login"),
                org.springframework.security.web.util.matcher.MediaTypeRequestMatcher(
                    org.springframework.http.MediaType.TEXT_HTML
                )
            )
        }
        
        return http.build()
    }
    
    @org.springframework.context.annotation.Bean
    @org.springframework.core.annotation.Order(2)
    fun defaultSecurityFilterChain(
        http: org.springframework.security.config.annotation.web.builders.HttpSecurity
    ): org.springframework.security.web.SecurityFilterChain {
        http
            .authorizeHttpRequests {
                it.requestMatchers("/login", "/error").permitAll()
                it.anyRequest().authenticated()
            }
            .formLogin(org.springframework.security.config.Customizer.withDefaults())
        
        return http.build()
    }
}
```

---

## PKCE Flow

```kotlin
// PKCE implementation for mobile/SPA clients

object PkceHelper {
    
    fun generateCodeVerifier(): String {
        val bytes = ByteArray(64)
        java.security.SecureRandom().nextBytes(bytes)
        return java.util.Base64.getUrlEncoder().withoutPadding().encodeToString(bytes)
    }
    
    fun generateCodeChallenge(verifier: String): String {
        val digest = java.security.MessageDigest.getInstance("SHA-256")
        val hash = digest.digest(verifier.toByteArray(Charsets.US_ASCII))
        return java.util.Base64.getUrlEncoder().withoutPadding().encodeToString(hash)
    }
}

// OAuth2 Client (Ktor-based)
class OAuth2Client(
    private val httpClient: io.ktor.client.HttpClient,
    private val config: OAuth2ClientConfig
) {
    
    data class OAuth2ClientConfig(
        val clientId: String,
        val authorizationEndpoint: String,
        val tokenEndpoint: String,
        val redirectUri: String,
        val scopes: List<String>
    )
    
    private val pendingVerifiers = mutableMapOf<String, String>()  // state -> verifier
    
    fun buildAuthorizationUrl(): Pair<String, String> {
        val state = java.util.UUID.randomUUID().toString()
        val verifier = PkceHelper.generateCodeVerifier()
        val challenge = PkceHelper.generateCodeChallenge(verifier)
        
        pendingVerifiers[state] = verifier
        
        val params = mapOf(
            "response_type" to "code",
            "client_id" to config.clientId,
            "redirect_uri" to config.redirectUri,
            "scope" to config.scopes.joinToString(" "),
            "state" to state,
            "code_challenge" to challenge,
            "code_challenge_method" to "S256"
        )
        
        val query = params.entries.joinToString("&") { (k, v) ->
            "${io.ktor.http.encodeURLPath(k)}=${io.ktor.http.encodeURLPath(v)}"
        }
        
        return "${config.authorizationEndpoint}?$query" to state
    }
    
    suspend fun exchangeCodeForTokens(code: String, state: String): TokenSet {
        val verifier = pendingVerifiers.remove(state)
            ?: throw IllegalStateException("Unknown state: $state")
        
        val response = httpClient.submitForm(
            url = config.tokenEndpoint,
            formParameters = io.ktor.http.Parameters.build {
                append("grant_type", "authorization_code")
                append("code", code)
                append("redirect_uri", config.redirectUri)
                append("client_id", config.clientId)
                append("code_verifier", verifier)
            }
        )
        
        return response.body()
    }
    
    suspend fun refreshAccessToken(refreshToken: String): TokenSet {
        val response = httpClient.submitForm(
            url = config.tokenEndpoint,
            formParameters = io.ktor.http.Parameters.build {
                append("grant_type", "refresh_token")
                append("refresh_token", refreshToken)
                append("client_id", config.clientId)
            }
        )
        
        return response.body()
    }
}

@kotlinx.serialization.Serializable
data class TokenSet(
    @kotlinx.serialization.SerialName("access_token") val accessToken: String,
    @kotlinx.serialization.SerialName("refresh_token") val refreshToken: String? = null,
    @kotlinx.serialization.SerialName("id_token") val idToken: String? = null,
    @kotlinx.serialization.SerialName("token_type") val tokenType: String,
    @kotlinx.serialization.SerialName("expires_in") val expiresIn: Int,
    val scope: String? = null
)
```

---

## Resource Server Configuration

```kotlin
// ResourceServerConfig.kt (API service)
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter

@org.springframework.context.annotation.Configuration
class ResourceServerConfig {
    
    @org.springframework.context.annotation.Bean
    fun resourceServerSecurityFilterChain(http: HttpSecurity): org.springframework.security.web.SecurityFilterChain {
        http
            .authorizeHttpRequests {
                it.requestMatchers("/actuator/health", "/api/v1/products").permitAll()
                it.requestMatchers("/api/v1/admin/**").hasAuthority("SCOPE_admin")
                it.requestMatchers(org.springframework.http.HttpMethod.POST, "/api/v1/orders").hasAuthority("SCOPE_orders:write")
                it.anyRequest().authenticated()
            }
            .oauth2ResourceServer { oauth2 ->
                oauth2.jwt { jwt ->
                    jwt.jwtAuthenticationConverter(jwtAuthenticationConverter())
                }
            }
            .sessionManagement {
                it.sessionCreationPolicy(org.springframework.security.config.http.SessionCreationPolicy.STATELESS)
            }
            .csrf { it.disable() }
        
        return http.build()
    }
    
    private fun jwtAuthenticationConverter(): JwtAuthenticationConverter {
        val grantedAuthoritiesConverter = JwtGrantedAuthoritiesConverter()
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles")
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_")
        
        val converter = JwtAuthenticationConverter()
        converter.setJwtGrantedAuthoritiesConverter { jwt ->
            val scopeAuthorities = JwtGrantedAuthoritiesConverter().convert(jwt) ?: emptyList()
            val roleAuthorities = grantedAuthoritiesConverter.convert(jwt) ?: emptyList()
            (scopeAuthorities + roleAuthorities).toMutableList()
        }
        
        return converter
    }
}

// application.yml (Resource Server)
/*
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.ecommerce.com
          jwk-set-uri: https://auth.ecommerce.com/oauth2/jwks
*/

// Custom SecurityContext for accessing token claims
import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.security.oauth2.jwt.Jwt
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken

object SecurityContext {
    fun getCurrentUserId(): String {
        val authentication = SecurityContextHolder.getContext().authentication
        return when (authentication) {
            is JwtAuthenticationToken -> authentication.token.subject
            else -> throw UnauthorizedException("Not authenticated")
        }
    }
    
    fun getCurrentUser(): JwtUser {
        val auth = SecurityContextHolder.getContext().authentication as? JwtAuthenticationToken
            ?: throw UnauthorizedException("Not authenticated")
        val jwt = auth.token
        
        return JwtUser(
            id = jwt.subject,
            email = jwt.getClaim("email") ?: "",
            name = jwt.getClaim("name") ?: "",
            roles = jwt.getClaim<List<String>>("roles") ?: emptyList(),
            tenantId = jwt.getClaim("tenantId"),
            scopes = auth.authorities.map { it.authority }.filter { it.startsWith("SCOPE_") }
                .map { it.removePrefix("SCOPE_") }
        )
    }
    
    fun hasScope(scope: String): Boolean {
        return getCurrentUser().scopes.contains(scope)
    }
    
    fun hasRole(role: String): Boolean {
        return getCurrentUser().roles.contains(role)
    }
}

data class JwtUser(
    val id: String,
    val email: String,
    val name: String,
    val roles: List<String>,
    val tenantId: String?,
    val scopes: List<String>
)

class UnauthorizedException(message: String) : RuntimeException(message)
```

---

## OpenID Connect (OIDC)

```kotlin
// OIDC UserInfo endpoint
@org.springframework.web.bind.annotation.RestController
@org.springframework.web.bind.annotation.RequestMapping("/userinfo")
class UserInfoController(private val userService: UserService) {
    
    @org.springframework.web.bind.annotation.GetMapping
    @org.springframework.security.access.prepost.PreAuthorize("hasAuthority('SCOPE_openid')")
    fun getUserInfo(
        authentication: org.springframework.security.core.Authentication
    ): Map<String, Any> {
        val jwt = (authentication as JwtAuthenticationToken).token
        val userId = jwt.subject
        val user = userService.findById(userId)
        
        return buildMap {
            put("sub", userId)
            
            if (jwt.getClaim<List<String>>("scp")?.contains("profile") == true) {
                put("name", user.fullName)
                put("given_name", user.firstName)
                put("family_name", user.lastName)
                put("picture", user.avatarUrl ?: "")
                put("updated_at", user.updatedAt.epochSecond)
            }
            
            if (jwt.getClaim<List<String>>("scp")?.contains("email") == true) {
                put("email", user.email)
                put("email_verified", user.emailVerified)
            }
        }
    }
}

// OAuth2 Login for the Authorization Server's own UI
@org.springframework.context.annotation.Configuration
class SocialLoginConfig {
    
    @org.springframework.context.annotation.Bean
    @org.springframework.core.annotation.Order(3)
    fun socialLoginSecurityFilterChain(
        http: org.springframework.security.config.annotation.web.builders.HttpSecurity
    ): org.springframework.security.web.SecurityFilterChain {
        http
            .authorizeHttpRequests {
                it.requestMatchers("/login", "/oauth2/**").permitAll()
                it.anyRequest().authenticated()
            }
            .formLogin { form ->
                form.loginPage("/login")
            }
            .oauth2Login { oauth2 ->
                oauth2.loginPage("/login")
                oauth2.userInfoEndpoint { userInfo ->
                    userInfo.userService(customOAuth2UserService())
                }
                oauth2.successHandler(oauth2SuccessHandler())
            }
        
        return http.build()
    }
    
    private fun customOAuth2UserService(): org.springframework.security.oauth2.client.userinfo.OAuth2UserService<
        org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest,
        org.springframework.security.oauth2.core.user.OAuth2User> {
        return org.springframework.security.oauth2.client.userinfo.DefaultOAuth2UserService()
    }
    
    private fun oauth2SuccessHandler(): org.springframework.security.web.authentication.AuthenticationSuccessHandler {
        return org.springframework.security.web.authentication.SimpleUrlAuthenticationSuccessHandler("/")
    }
}

// application.yml additions for social login
/*
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: read:user, user:email
*/
```

---

## Security Best Practices

```kotlin
// 1. Token Introspection (for opaque tokens)
@org.springframework.context.annotation.Configuration
class OpaqueTokenConfig {
    @org.springframework.context.annotation.Bean
    fun opaqueTokenIntrospector(
        @org.springframework.beans.factory.annotation.Value("\${spring.security.oauth2.resourceserver.opaquetoken.introspection-uri}") introspectionUri: String,
        @org.springframework.beans.factory.annotation.Value("\${spring.security.oauth2.resourceserver.opaquetoken.client-id}") clientId: String,
        @org.springframework.beans.factory.annotation.Value("\${spring.security.oauth2.resourceserver.opaquetoken.client-secret}") clientSecret: String
    ): org.springframework.security.oauth2.server.resource.introspection.OpaqueTokenIntrospector {
        return org.springframework.security.oauth2.server.resource.introspection.SpringOpaqueTokenIntrospector(
            introspectionUri, clientId, clientSecret
        )
    }
}

// 2. CSRF Protection for OAuth2 state parameter
class OAuth2StateCsrfFilter : javax.servlet.Filter {
    private val stateStore = com.google.common.cache.CacheBuilder.newBuilder()
        .expireAfterWrite(10, java.util.concurrent.TimeUnit.MINUTES)
        .build<String, String>()
    
    override fun doFilter(request: javax.servlet.ServletRequest, response: javax.servlet.ServletResponse, chain: javax.servlet.FilterChain) {
        val req = request as javax.servlet.http.HttpServletRequest
        val state = req.getParameter("state")
        
        if (req.requestURI == "/callback" && state != null) {
            if (stateStore.getIfPresent(state) == null) {
                (response as javax.servlet.http.HttpServletResponse).sendError(
                    javax.servlet.http.HttpServletResponse.SC_BAD_REQUEST,
                    "Invalid state parameter"
                )
                return
            }
            stateStore.invalidate(state)
        }
        
        chain.doFilter(request, response)
    }
}

// 3. Rate limiting for token endpoint
@org.springframework.stereotype.Component
class TokenEndpointRateLimiter(
    private val redisTemplate: org.springframework.data.redis.core.StringRedisTemplate
) : org.springframework.security.web.access.intercept.FilterSecurityInterceptor() {
    
    fun checkRateLimit(clientId: String, remoteAddr: String) {
        val key = "oauth2:ratelimit:$clientId:$remoteAddr"
        val count = redisTemplate.opsForValue().increment(key) ?: 1L
        
        if (count == 1L) {
            redisTemplate.expire(key, java.time.Duration.ofMinutes(1))
        }
        
        if (count > 20) {
            throw org.springframework.security.oauth2.core.OAuth2AuthenticationException(
                org.springframework.security.oauth2.core.OAuth2Error("too_many_requests", "Rate limit exceeded", null)
            )
        }
    }
}

// 4. Secure token storage
class SecureTokenStorage(private val encryptionKey: javax.crypto.SecretKey) {
    
    fun encrypt(token: String): String {
        val cipher = javax.crypto.Cipher.getInstance("AES/GCM/NoPadding")
        val iv = ByteArray(12).also { java.security.SecureRandom().nextBytes(it) }
        val spec = javax.crypto.spec.GCMParameterSpec(128, iv)
        cipher.init(javax.crypto.Cipher.ENCRYPT_MODE, encryptionKey, spec)
        val encrypted = cipher.doFinal(token.toByteArray())
        return java.util.Base64.getEncoder().encodeToString(iv + encrypted)
    }
    
    fun decrypt(encryptedToken: String): String {
        val bytes = java.util.Base64.getDecoder().decode(encryptedToken)
        val iv = bytes.take(12).toByteArray()
        val encrypted = bytes.drop(12).toByteArray()
        
        val cipher = javax.crypto.Cipher.getInstance("AES/GCM/NoPadding")
        val spec = javax.crypto.spec.GCMParameterSpec(128, iv)
        cipher.init(javax.crypto.Cipher.DECRYPT_MODE, encryptionKey, spec)
        return String(cipher.doFinal(encrypted))
    }
}

// 5. Audit logging
@org.springframework.stereotype.Component
class SecurityAuditLogger(
    private val auditRepository: AuditLogRepository
) {
    
    @org.springframework.context.event.EventListener
    fun onAuthentication(event: org.springframework.security.authentication.event.AuthenticationSuccessEvent) {
        auditRepository.log(AuditEvent(
            type = "AUTHENTICATION_SUCCESS",
            username = event.authentication.name,
            timestamp = java.time.Instant.now(),
            details = mapOf("source" to "oauth2")
        ))
    }
    
    @org.springframework.context.event.EventListener
    fun onFailedAuthentication(event: org.springframework.security.authentication.event.AbstractAuthenticationFailureEvent) {
        auditRepository.log(AuditEvent(
            type = "AUTHENTICATION_FAILURE",
            username = event.authentication.name,
            timestamp = java.time.Instant.now(),
            details = mapOf("reason" to (event.exception.message ?: "unknown"))
        ))
    }
}

data class AuditEvent(
    val type: String,
    val username: String,
    val timestamp: java.time.Instant,
    val details: Map<String, String>
)

interface AuditLogRepository {
    fun log(event: AuditEvent)
}

// 6. Content Security Policy header
@org.springframework.stereotype.Component
class SecurityHeadersFilter : javax.servlet.Filter {
    
    override fun doFilter(request: javax.servlet.ServletRequest, response: javax.servlet.ServletResponse, chain: javax.servlet.FilterChain) {
        val res = response as javax.servlet.http.HttpServletResponse
        
        res.setHeader("Content-Security-Policy",
            "default-src 'self'; " +
            "script-src 'self' 'nonce-{NONCE}'; " +
            "style-src 'self' 'unsafe-inline'; " +
            "img-src 'self' data: https:; " +
            "connect-src 'self' https://auth.ecommerce.com; " +
            "frame-ancestors 'none'")
        
        res.setHeader("Permissions-Policy", "camera=(), microphone=(), geolocation=()")
        res.setHeader("Referrer-Policy", "strict-origin-when-cross-origin")
        
        chain.doFilter(request, response)
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement Token Revocation with Redis Blocklist

// Requirements:
// 1. เมื่อ logout ให้ add access token ไป Redis blocklist (TTL = remaining time)
// 2. เมื่อ logout ให้ delete refresh token ด้วย
// 3. สร้าง JwtValidationFilter ที่ check blocklist ทุก request
// 4. เมื่อ user เปลี่ยน password ให้ revoke tokens ทั้งหมด

@org.springframework.stereotype.Service
class TokenRevocationService(
    private val redisTemplate: org.springframework.data.redis.core.StringRedisTemplate
) {
    private val BLOCKLIST_PREFIX = "oauth2:blocklist:"
    private val USER_TOKENS_PREFIX = "oauth2:user_tokens:"
    
    fun revokeAccessToken(tokenId: String, expiresAt: java.time.Instant) {
        val ttl = java.time.Duration.between(java.time.Instant.now(), expiresAt)
        if (ttl.isPositive) {
            redisTemplate.opsForValue().set(
                "$BLOCKLIST_PREFIX$tokenId",
                "revoked",
                ttl
            )
        }
    }
    
    fun isRevoked(tokenId: String): Boolean {
        return redisTemplate.hasKey("$BLOCKLIST_PREFIX$tokenId") == true
    }
    
    fun revokeAllUserTokens(userId: String) {
        // Track all tokens per user for bulk revocation
        val tokenIds = redisTemplate.opsForSet().members("$USER_TOKENS_PREFIX$userId") ?: return
        tokenIds.forEach { tokenId ->
            redisTemplate.opsForValue().set(
                "$BLOCKLIST_PREFIX$tokenId",
                "revoked",
                java.time.Duration.ofHours(24)
            )
        }
        redisTemplate.delete("$USER_TOKENS_PREFIX$userId")
    }
    
    fun trackToken(userId: String, tokenId: String) {
        redisTemplate.opsForSet().add("$USER_TOKENS_PREFIX$userId", tokenId)
        redisTemplate.expire("$USER_TOKENS_PREFIX$userId", java.time.Duration.ofDays(30))
    }
}

class TokenBlocklistFilter(
    private val revocationService: TokenRevocationService,
    private val jwtDecoder: org.springframework.security.oauth2.jwt.JwtDecoder
) : javax.servlet.Filter {
    
    override fun doFilter(request: javax.servlet.ServletRequest, response: javax.servlet.ServletResponse, chain: javax.servlet.FilterChain) {
        val req = request as javax.servlet.http.HttpServletRequest
        val token = extractBearerToken(req)
        
        if (token != null) {
            try {
                val jwt = jwtDecoder.decode(token)
                val tokenId = jwt.id ?: jwt.subject
                
                if (revocationService.isRevoked(tokenId)) {
                    (response as javax.servlet.http.HttpServletResponse).apply {
                        status = javax.servlet.http.HttpServletResponse.SC_UNAUTHORIZED
                        writer.write("""{"error":"token_revoked","description":"Token has been revoked"}""")
                    }
                    return
                }
            } catch (e: Exception) {
                // Invalid token, let it fail naturally
            }
        }
        
        chain.doFilter(request, response)
    }
    
    private fun extractBearerToken(request: javax.servlet.http.HttpServletRequest): String? {
        val header = request.getHeader("Authorization") ?: return null
        return if (header.startsWith("Bearer ")) header.substring(7) else null
    }
}
```

---

## สรุป Part 83

```
✅ OAuth2 grant types: Authorization Code, Client Credentials, Refresh Token, Device Code
✅ PKCE: code_verifier, code_challenge = BASE64URL(SHA256(verifier))
✅ Spring Authorization Server: RegisteredClient, ClientSettings, TokenSettings
✅ Three client types: web (confidential), mobile (public), service (credentials)
✅ requireProofKey(true): enforce PKCE for authorization code flow
✅ Token customizer: add roles, tenantId, name to JWT claims
✅ RSA JWK: 2048-bit RSA key for JWT signing
✅ JWKS endpoint: /oauth2/jwks for public key distribution
✅ Resource Server: jwt().jwtAuthenticationConverter()
✅ JwtGrantedAuthoritiesConverter: extract roles from claims
✅ SecurityContext.getCurrentUser(): typed user from JWT
✅ OIDC UserInfo endpoint: sub, profile, email scopes
✅ Social login: Google, GitHub via OAuth2 login
✅ Token Introspection: for opaque tokens
✅ CSRF state parameter validation
✅ Rate limiting for token endpoint
✅ AES-GCM token encryption at rest
✅ Security audit logging via Spring events
✅ Content Security Policy header
✅ Token revocation with Redis blocklist
✅ TokenBlocklistFilter: check blocklist on every request
✅ Bulk revocation on password change
```

---

*Part 83/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
