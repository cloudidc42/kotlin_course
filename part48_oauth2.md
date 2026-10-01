# Part 48: OAuth2 และ Social Login ด้วย Kotlin

## สารบัญ
1. [OAuth2 Flow](#oauth2-flow)
2. [Spring Security OAuth2](#spring-security-oauth2)
3. [JWT + OAuth2 Integration](#jwt--oauth2-integration)
4. [Social Login](#social-login)
5. [Resource Server](#resource-server)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## OAuth2 Flow

```
Authorization Code Flow (แนะนำสำหรับ web apps):

1. User คลิก "Login with Google"
2. App redirect -> Google Authorization Server
   GET https://accounts.google.com/o/oauth2/auth
       ?client_id=YOUR_CLIENT_ID
       &redirect_uri=https://myapp.com/callback
       &response_type=code
       &scope=openid email profile
       &state=random-state-value

3. User กรอก credentials ที่ Google
4. Google redirect กลับพร้อม authorization code
   GET https://myapp.com/callback
       ?code=AUTHORIZATION_CODE
       &state=random-state-value

5. App แลก code เป็น tokens (server-side, ปลอดภัย)
   POST https://oauth2.googleapis.com/token
       client_id, client_secret, code, redirect_uri, grant_type

6. Google ส่ง tokens กลับ
   { access_token, id_token, refresh_token, expires_in }

7. App ใช้ access_token เรียก API หรือ id_token decode user info
```

---

## Spring Security OAuth2 Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-client")
    implementation("org.springframework.boot:spring-boot-starter-oauth2-resource-server")
    implementation("com.nimbusds:nimbus-jose-jwt:9.37.3")
}
```

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, email, profile
          
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: user:email
          
          line:
            client-id: ${LINE_CLIENT_ID}
            client-secret: ${LINE_CLIENT_SECRET}
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/line"
            scope: profile, openid, email
        
        provider:
          line:
            authorization-uri: https://access.line.me/oauth2/v2.1/authorize
            token-uri: https://api.line.me/oauth2/v2.1/token
            user-info-uri: https://api.line.me/v2/profile
            user-name-attribute: userId
      
      resourceserver:
        jwt:
          issuer-uri: ${JWT_ISSUER_URI:http://localhost:8080}
          jwk-set-uri: ${JWK_SET_URI:http://localhost:8080/.well-known/jwks.json}
```

---

## Security Configuration

```kotlin
@Configuration
@EnableWebSecurity
class SecurityConfig(
    private val oAuth2UserService: CustomOAuth2UserService,
    private val successHandler: OAuth2LoginSuccessHandler,
    private val failureHandler: OAuth2LoginFailureHandler
) {
    
    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        return http
            .csrf { it.disable() }
            .sessionManagement { session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            }
            .authorizeHttpRequests { auth ->
                auth
                    .requestMatchers("/api/v1/auth/**", "/login/**", "/oauth2/**").permitAll()
                    .requestMatchers("/actuator/health").permitAll()
                    .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                    .anyRequest().authenticated()
            }
            .oauth2Login { oauth2 ->
                oauth2
                    .userInfoEndpoint { userInfo ->
                        userInfo.userService(oAuth2UserService)
                    }
                    .successHandler(successHandler)
                    .failureHandler(failureHandler)
            }
            .oauth2ResourceServer { resourceServer ->
                resourceServer.jwt { jwt ->
                    jwt.jwtAuthenticationConverter(jwtAuthenticationConverter())
                }
            }
            .build()
    }
    
    @Bean
    fun jwtAuthenticationConverter(): JwtAuthenticationConverter {
        val rolesConverter = JwtGrantedAuthoritiesConverter().apply {
            setAuthoritiesClaimName("roles")
            setAuthorityPrefix("ROLE_")
        }
        return JwtAuthenticationConverter().apply {
            setJwtGrantedAuthoritiesConverter(rolesConverter)
        }
    }
    
    @Bean
    fun passwordEncoder(): PasswordEncoder = BCryptPasswordEncoder()
}
```

---

## Custom OAuth2 User Service

```kotlin
@Component
class CustomOAuth2UserService(
    private val userRepository: UserRepository2,
    private val oauth2UserInfoFactory: OAuth2UserInfoFactory
) : DefaultOAuth2UserService() {
    
    override fun loadUser(userRequest: OAuth2UserRequest): OAuth2User {
        val oAuth2User = super.loadUser(userRequest)
        
        val provider = userRequest.clientRegistration.registrationId
        val userInfo = oauth2UserInfoFactory.getOAuth2UserInfo(provider, oAuth2User.attributes)
        
        val user = userRepository.findByProviderAndProviderId(provider, userInfo.id)
            ?: createUser(provider, userInfo)
        
        return CustomUserDetails(user, oAuth2User.attributes)
    }
    
    private fun createUser(provider: String, userInfo: OAuth2UserInfo): UserEntity2 {
        return userRepository.save(UserEntity2(
            id = java.util.UUID.randomUUID().toString(),
            email = userInfo.email ?: "${userInfo.id}@${provider}.local",
            name = userInfo.name,
            imageUrl = userInfo.imageUrl,
            provider = provider,
            providerId = userInfo.id,
            emailVerified = true,
            role = "USER"
        ))
    }
}

// User info abstraction
interface OAuth2UserInfo {
    val id: String
    val name: String
    val email: String?
    val imageUrl: String?
}

class GoogleOAuth2UserInfo(private val attributes: Map<String, Any>) : OAuth2UserInfo {
    override val id: String get() = attributes["sub"] as String
    override val name: String get() = attributes["name"] as String
    override val email: String? get() = attributes["email"] as? String
    override val imageUrl: String? get() = attributes["picture"] as? String
}

class GithubOAuth2UserInfo(private val attributes: Map<String, Any>) : OAuth2UserInfo {
    override val id: String get() = attributes["id"].toString()
    override val name: String get() = attributes["name"] as? String ?: attributes["login"] as String
    override val email: String? get() = attributes["email"] as? String
    override val imageUrl: String? get() = attributes["avatar_url"] as? String
}

class LineOAuth2UserInfo(private val attributes: Map<String, Any>) : OAuth2UserInfo {
    override val id: String get() = attributes["userId"] as String
    override val name: String get() = attributes["displayName"] as String
    override val email: String? get() = attributes["email"] as? String
    override val imageUrl: String? get() = attributes["pictureUrl"] as? String
}

@Component
class OAuth2UserInfoFactory {
    fun getOAuth2UserInfo(provider: String, attributes: Map<String, Any>): OAuth2UserInfo {
        return when (provider.lowercase()) {
            "google" -> GoogleOAuth2UserInfo(attributes)
            "github" -> GithubOAuth2UserInfo(attributes)
            "line" -> LineOAuth2UserInfo(attributes)
            else -> throw OAuth2AuthenticationProcessingException("Provider $provider not supported")
        }
    }
}

data class CustomUserDetails(
    val user: UserEntity2,
    private val attributes: Map<String, Any>
) : OAuth2User {
    override fun getName() = user.id
    override fun getAttributes() = attributes
    override fun getAuthorities() = listOf(
        SimpleGrantedAuthority("ROLE_${user.role}")
    )
}
```

---

## OAuth2 Login Success/Failure Handlers

```kotlin
@Component
class OAuth2LoginSuccessHandler(
    private val jwtTokenProvider: JwtTokenProvider,
    private val appProperties: AppProperties
) : SimpleUrlAuthenticationSuccessHandler() {
    
    override fun onAuthenticationSuccess(
        request: HttpServletRequest,
        response: HttpServletResponse,
        authentication: Authentication
    ) {
        val targetUrl = determineTargetUrl(request, response, authentication)
        
        if (response.isCommitted) {
            return
        }
        
        clearAuthenticationAttributes(request, response)
        redirectStrategy.sendRedirect(request, response, targetUrl)
    }
    
    override fun determineTargetUrl(
        request: HttpServletRequest,
        response: HttpServletResponse,
        authentication: Authentication
    ): String {
        val redirectUri = CookieUtils.getCookie(request, REDIRECT_URI_PARAM_COOKIE_NAME)
            ?.value
        
        val targetUrl = redirectUri ?: defaultTargetUrl
        
        val userDetails = authentication.principal as CustomUserDetails
        val tokens = jwtTokenProvider.createTokens(userDetails.user)
        
        return UriComponentsBuilder.fromUriString(targetUrl)
            .queryParam("token", tokens.accessToken)
            .queryParam("refresh_token", tokens.refreshToken)
            .build().toUriString()
    }
    
    private fun clearAuthenticationAttributes(
        request: HttpServletRequest,
        response: HttpServletResponse
    ) {
        super.clearAuthenticationAttributes(request)
        CookieUtils.deleteCookie(request, response, REDIRECT_URI_PARAM_COOKIE_NAME)
    }
    
    companion object {
        const val REDIRECT_URI_PARAM_COOKIE_NAME = "redirect_uri"
    }
}

@Component
class OAuth2LoginFailureHandler : SimpleUrlAuthenticationFailureHandler() {
    
    override fun onAuthenticationFailure(
        request: HttpServletRequest,
        response: HttpServletResponse,
        exception: AuthenticationException
    ) {
        val redirectUri = CookieUtils.getCookie(request, REDIRECT_URI_PARAM_COOKIE_NAME)
            ?.value ?: "/"
        
        val targetUrl = UriComponentsBuilder.fromUriString(redirectUri)
            .queryParam("error", exception.localizedMessage)
            .build().toUriString()
        
        redirectStrategy.sendRedirect(request, response, targetUrl)
    }
    
    companion object {
        const val REDIRECT_URI_PARAM_COOKIE_NAME = "redirect_uri"
    }
}
```

---

## JWT Token Provider

```kotlin
@Component
class JwtTokenProvider(private val appProperties: AppProperties) {
    
    private val key by lazy {
        Keys.hmacShaKeyFor(appProperties.jwtSecret.toByteArray())
    }
    
    fun createTokens(user: UserEntity2): TokenPair {
        val now = Date()
        
        val accessToken = Jwts.builder()
            .subject(user.id)
            .claim("email", user.email)
            .claim("name", user.name)
            .claim("roles", listOf(user.role))
            .claim("provider", user.provider)
            .issuedAt(now)
            .expiration(Date(now.time + appProperties.accessTokenExpiration))
            .signWith(key)
            .compact()
        
        val refreshToken = Jwts.builder()
            .subject(user.id)
            .claim("type", "refresh")
            .issuedAt(now)
            .expiration(Date(now.time + appProperties.refreshTokenExpiration))
            .signWith(key)
            .compact()
        
        return TokenPair(accessToken, refreshToken)
    }
    
    fun validateToken(token: String): JwtClaims? {
        return try {
            val claims = Jwts.parser()
                .verifyWith(key)
                .build()
                .parseSignedClaims(token)
                .payload
            
            JwtClaims(
                userId = claims.subject,
                email = claims["email"] as? String,
                roles = @Suppress("UNCHECKED_CAST") (claims["roles"] as? List<String>) ?: emptyList()
            )
        } catch (e: Exception) {
            null
        }
    }
}

data class TokenPair(val accessToken: String, val refreshToken: String)

data class JwtClaims(
    val userId: String,
    val email: String?,
    val roles: List<String>
)
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement token refresh and account linking

// When user already has an account (email login) and tries to social login with same email:
// - Should link the accounts instead of creating a new one

// Token refresh endpoint
@RestController
@RequestMapping("/api/v1/auth")
class AuthController(
    private val jwtTokenProvider: JwtTokenProvider,
    private val userRepository: UserRepository2
) {
    
    @PostMapping("/refresh")
    fun refreshToken(@RequestBody request: RefreshTokenRequest): ResponseEntity<TokenPair> {
        // TODO: Validate refresh token
        // TODO: Find user by userId from token
        // TODO: Generate new token pair
        // TODO: Optionally invalidate old refresh token (token rotation)
        TODO("Implement token refresh")
    }
    
    @PostMapping("/logout")
    fun logout(@RequestHeader("Authorization") token: String): ResponseEntity<Void> {
        // TODO: Blacklist the token in Redis
        // Token is stateless JWT so we need to store invalidated tokens
        TODO("Implement logout with token blacklist")
    }
    
    @GetMapping("/me")
    fun getCurrentUser(authentication: Authentication): UserResponse {
        val userId = authentication.name
        val user = userRepository.findById(userId)
            ?: throw ResourceNotFoundException("users", userId)
        return user.toResponse()
    }
}

data class RefreshTokenRequest(val refreshToken: String)
data class UserResponse(val id: String, val name: String, val email: String?, val provider: String)

// Placeholder types
data class UserEntity2(
    val id: String,
    val email: String,
    val name: String,
    val imageUrl: String?,
    val provider: String,
    val providerId: String,
    val emailVerified: Boolean,
    val role: String
) {
    fun toResponse() = UserResponse(id, name, email, provider)
}

interface UserRepository2 {
    fun findByProviderAndProviderId(provider: String, providerId: String): UserEntity2?
    fun findById(id: String): UserEntity2?
    fun save(user: UserEntity2): UserEntity2
}

data class AppProperties(
    val jwtSecret: String = "default-secret-32-chars-minimum",
    val accessTokenExpiration: Long = 15 * 60 * 1000,  // 15 min
    val refreshTokenExpiration: Long = 7 * 24 * 60 * 60 * 1000  // 7 days
)

class OAuth2AuthenticationProcessingException(message: String) : Exception(message)
class ResourceNotFoundException(resource: String, id: String) : Exception("$resource: $id not found")

typealias SimpleGrantedAuthority = org.springframework.security.core.authority.SimpleGrantedAuthority
typealias BCryptPasswordEncoder = org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder
typealias CookieUtils = Any  // implement your cookie utils
```

---

## สรุป Part 48

```
✅ OAuth2: industry-standard authorization protocol
✅ Authorization Code Flow: ปลอดภัยที่สุดสำหรับ web apps
✅ Spring Security OAuth2 Client: built-in integration
✅ Google/GitHub/LINE providers: pre-configured
✅ Custom OAuth2UserService: normalize user info across providers
✅ OAuth2UserInfo abstraction: handle different attribute keys
✅ Success/Failure handlers: redirect with JWT token
✅ JwtTokenProvider: create + validate JWT
✅ Token refresh: extend session without re-login
✅ Token blacklist (Redis): implement logout properly
✅ Account linking: connect multiple providers to one account
✅ PKCE: enhanced security for public clients
```

---

*Part 48/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
