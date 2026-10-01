# Part 60: WebSocket และ Real-Time Communication

## สารบัญ
1. [WebSocket ด้วย Spring Boot](#websocket-ด้วย-spring-boot)
2. [STOMP Protocol](#stomp-protocol)
3. [Presence System](#presence-system)
4. [Chat Room Implementation](#chat-room-implementation)
5. [WebSocket Security](#websocket-security)
6. [Testing WebSocket](#testing-websocket)
7. [แบบฝึกหัด](#แบบฝึกหัด)

---

## WebSocket ด้วย Spring Boot

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-websocket")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
}

// WebSocket Config
@Configuration
@EnableWebSocketMessageBroker
class WebSocketConfig : WebSocketMessageBrokerConfigurer {
    
    override fun configureMessageBroker(config: MessageBrokerRegistry) {
        // Use Redis as message broker for horizontal scaling
        // config.enableStompBrokerRelay("/topic", "/queue")
        //     .setRelayHost("localhost")
        //     .setRelayPort(61613)  // ActiveMQ STOMP port
        
        // In-memory broker for development
        config.enableSimpleBroker("/topic", "/queue")
        config.setApplicationDestinationPrefixes("/app")
        config.setUserDestinationPrefix("/user")
    }
    
    override fun registerStompEndpoints(registry: StompEndpointRegistry) {
        registry.addEndpoint("/ws")
            .setAllowedOriginPatterns("http://localhost:3000", "https://myapp.com")
            .withSockJS()  // Fallback for browsers without WebSocket
    }
    
    override fun configureWebSocketTransport(registration: WebSocketTransportRegistration) {
        registration.setMessageSizeLimit(64 * 1024)  // 64KB
        registration.setSendBufferSizeLimit(512 * 1024)
        registration.setSendTimeLimit(20_000)
    }
}
```

---

## STOMP Protocol

```kotlin
// Message types
data class ChatMessage(
    val messageId: String = java.util.UUID.randomUUID().toString(),
    val roomId: String,
    val senderId: String,
    val senderName: String,
    val content: String,
    val type: MessageType = MessageType.TEXT,
    val timestamp: Long = System.currentTimeMillis()
)

data class Notification(
    val type: NotificationType,
    val payload: Map<String, Any>
)

enum class MessageType { TEXT, IMAGE, FILE, SYSTEM }
enum class NotificationType { USER_JOINED, USER_LEFT, TYPING, ORDER_UPDATE, PAYMENT_CONFIRMED }

// Controller for WebSocket messages
@Controller
class ChatController(
    private val chatService: ChatService,
    private val simpMessagingTemplate: SimpMessagingTemplate
) {
    
    // Handle message to /app/chat.sendMessage
    @MessageMapping("/chat.sendMessage")
    @SendTo("/topic/rooms/{roomId}")
    fun sendMessage(@Payload message: ChatMessage, principal: Principal): ChatMessage {
        val savedMessage = chatService.saveMessage(message.copy(senderId = principal.name))
        return savedMessage
    }
    
    // Handle message to /app/chat.addUser
    @MessageMapping("/chat.addUser")
    fun addUser(
        @Payload message: ChatMessage,
        @DestinationVariable roomId: String,
        principal: Principal
    ) {
        val joinMessage = ChatMessage(
            roomId = roomId,
            senderId = "system",
            senderName = "System",
            content = "${principal.name} joined the room",
            type = MessageType.SYSTEM
        )
        
        // Broadcast to room
        simpMessagingTemplate.convertAndSend("/topic/rooms/$roomId", joinMessage)
        
        // Send room history to the new user
        val history = chatService.getRoomHistory(roomId, limit = 50)
        simpMessagingTemplate.convertAndSendToUser(
            principal.name,
            "/queue/history",
            history
        )
    }
    
    // Send notification to specific user
    @MessageMapping("/order.update")
    fun orderUpdate(
        @Payload notification: Notification,
        @Header("simpSessionId") sessionId: String
    ) {
        val userId = getUserIdFromSession(sessionId)
        simpMessagingTemplate.convertAndSendToUser(
            userId,
            "/queue/notifications",
            notification
        )
    }
    
    // Broadcast to all users
    fun broadcastSystemMessage(message: String) {
        simpMessagingTemplate.convertAndSend(
            "/topic/system",
            ChatMessage(
                roomId = "system",
                senderId = "system",
                senderName = "System",
                content = message,
                type = MessageType.SYSTEM
            )
        )
    }
    
    private fun getUserIdFromSession(sessionId: String): String = sessionId  // simplified
}
```

---

## Presence System

```kotlin
// Tracking who is online
@Component
class PresenceService(
    private val redisTemplate: org.springframework.data.redis.core.StringRedisTemplate,
    private val simpMessagingTemplate: SimpMessagingTemplate
) {
    
    private val ONLINE_USERS_KEY = "online:users"
    private val USER_ROOMS_PREFIX = "user:rooms:"
    
    fun userConnected(userId: String, sessionId: String) {
        // ใช้ Redis HSET: field=userId, value=sessionId
        redisTemplate.opsForHash<String, String>().put(ONLINE_USERS_KEY, userId, sessionId)
        
        // Set expiry (cleanup if server dies)
        redisTemplate.expire(ONLINE_USERS_KEY, java.time.Duration.ofHours(24))
        
        broadcastPresenceUpdate(userId, "ONLINE")
    }
    
    fun userDisconnected(userId: String) {
        redisTemplate.opsForHash<String, String>().delete(ONLINE_USERS_KEY, userId)
        broadcastPresenceUpdate(userId, "OFFLINE")
    }
    
    fun getOnlineUsers(): Set<String> {
        return redisTemplate.opsForHash<String, String>().keys(ONLINE_USERS_KEY)
    }
    
    fun isOnline(userId: String): Boolean {
        return redisTemplate.opsForHash<String, String>().hasKey(ONLINE_USERS_KEY, userId)
    }
    
    fun userJoinedRoom(userId: String, roomId: String) {
        val key = "$USER_ROOMS_PREFIX$userId"
        redisTemplate.opsForSet().add(key, roomId)
        redisTemplate.expire(key, java.time.Duration.ofHours(24))
    }
    
    fun userLeftRoom(userId: String, roomId: String) {
        redisTemplate.opsForSet().remove("$USER_ROOMS_PREFIX$userId", roomId)
    }
    
    fun getUserRooms(userId: String): Set<String> {
        return redisTemplate.opsForSet().members("$USER_ROOMS_PREFIX$userId") ?: emptySet()
    }
    
    private fun broadcastPresenceUpdate(userId: String, status: String) {
        simpMessagingTemplate.convertAndSend(
            "/topic/presence",
            mapOf("userId" to userId, "status" to status, "timestamp" to System.currentTimeMillis())
        )
    }
}

// WebSocket Event Listener
@Component
class WebSocketEventListener(
    private val presenceService: PresenceService,
    private val chatService: ChatService
) {
    
    @EventListener
    fun handleWebSocketConnectListener(event: SessionConnectedEvent) {
        val accessor = SimpMessageHeaderAccessor.wrap(event.message)
        val userId = accessor.user?.name ?: return
        val sessionId = accessor.sessionId ?: return
        
        presenceService.userConnected(userId, sessionId)
        println("User connected: $userId (session: $sessionId)")
    }
    
    @EventListener
    fun handleWebSocketDisconnectListener(event: SessionDisconnectEvent) {
        val accessor = SimpMessageHeaderAccessor.wrap(event.message)
        val userId = accessor.user?.name ?: return
        
        // Remove from all rooms
        presenceService.getUserRooms(userId).forEach { roomId ->
            chatService.handleUserLeft(roomId, userId)
            presenceService.userLeftRoom(userId, roomId)
        }
        
        presenceService.userDisconnected(userId)
        println("User disconnected: $userId")
    }
}

interface ChatService {
    fun saveMessage(message: ChatMessage): ChatMessage
    fun getRoomHistory(roomId: String, limit: Int): List<ChatMessage>
    fun handleUserLeft(roomId: String, userId: String)
}
```

---

## Typing Indicator

```kotlin
@Controller
class TypingController(
    private val simpMessagingTemplate: SimpMessagingTemplate
) {
    
    // Client sends to /app/chat.typing
    @MessageMapping("/chat.typing")
    fun typing(
        @Payload typingEvent: TypingEvent,
        principal: Principal
    ) {
        // Broadcast to others in the room (not the sender)
        simpMessagingTemplate.convertAndSend(
            "/topic/rooms/${typingEvent.roomId}/typing",
            TypingEvent(
                roomId = typingEvent.roomId,
                userId = principal.name,
                isTyping = typingEvent.isTyping
            )
        )
    }
}

data class TypingEvent(val roomId: String, val userId: String, val isTyping: Boolean)

// Debounced typing indicator (client-side concept)
// Server can also throttle with Redis:

@Component
class TypingThrottleService(
    private val redisTemplate: org.springframework.data.redis.core.StringRedisTemplate
) {
    
    // Only process one typing event per user per 2 seconds
    fun shouldBroadcast(userId: String, roomId: String): Boolean {
        val key = "typing:$userId:$roomId"
        val result = redisTemplate.opsForValue().setIfAbsent(key, "1", java.time.Duration.ofSeconds(2))
        return result == true
    }
}
```

---

## WebSocket Security

```kotlin
@Configuration
class WebSocketSecurityConfig : AbstractSecurityWebSocketMessageBrokerConfigurer() {
    
    override fun configureInbound(messages: MessageSecurityMetadataSourceRegistry) {
        messages
            .nullDestMatcher().authenticated()
            .simpSubscribeDestMatchers("/user/**").authenticated()
            .simpDestMatchers("/app/**").authenticated()
            .simpSubscribeDestMatchers("/topic/rooms/**").authenticated()
            .simpSubscribeDestMatchers("/topic/system").permitAll()
            .anyMessage().denyAll()
    }
    
    override fun sameOriginDisabled() = true  // เพราะใช้ CORS แทน
}

// Channel Interceptor: validate JWT on STOMP CONNECT
@Component
class JwtChannelInterceptor(
    private val jwtService: JwtService
) : ChannelInterceptor {
    
    override fun preSend(message: Message<*>, channel: MessageChannel): Message<*>? {
        val accessor = StompHeaderAccessor.wrap(message)
        
        if (StompCommand.CONNECT == accessor.command) {
            val authHeader = accessor.getFirstNativeHeader("Authorization")
            
            if (authHeader == null || !authHeader.startsWith("Bearer ")) {
                throw MessageDeliveryException("Authorization header required for WebSocket connection")
            }
            
            val token = authHeader.removePrefix("Bearer ")
            val claims = jwtService.validateToken(token)
                ?: throw MessageDeliveryException("Invalid or expired token")
            
            // Set principal for this session
            accessor.user = UsernamePasswordAuthenticationToken(
                claims.userId,
                null,
                claims.roles.map { SimpleGrantedAuthority("ROLE_$it") }
            )
        }
        
        return message
    }
}

@Configuration
class WebSocketBrokerConfig : WebSocketMessageBrokerConfigurer {
    
    @Autowired
    lateinit var jwtChannelInterceptor: JwtChannelInterceptor
    
    override fun configureClientInboundChannel(registration: ChannelRegistration) {
        registration.interceptors(jwtChannelInterceptor)
    }
}

// Placeholder types
data class JwtClaims2(val userId: String, val roles: List<String>)
interface JwtService { fun validateToken(token: String): JwtClaims2? }
class MessageDeliveryException(msg: String) : Exception(msg)
typealias SimpleGrantedAuthority = org.springframework.security.core.authority.SimpleGrantedAuthority
typealias UsernamePasswordAuthenticationToken = org.springframework.security.authentication.UsernamePasswordAuthenticationToken
```

---

## Testing WebSocket

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class WebSocketIntegrationTest {
    
    @LocalServerPort
    var port: Int = 0
    
    lateinit var stompClient: WebSocketStompClient
    lateinit var stompSession: StompSession
    
    @BeforeEach
    fun setup() {
        stompClient = WebSocketStompClient(StandardWebSocketClient()).apply {
            messageConverter = MappingJackson2MessageConverter()
        }
        
        stompSession = stompClient.connectAsync(
            "ws://localhost:$port/ws",
            object : StompSessionHandlerAdapter() {}
        ).get(5, java.util.concurrent.TimeUnit.SECONDS)
    }
    
    @AfterEach
    fun teardown() {
        if (stompSession.isConnected) stompSession.disconnect()
    }
    
    @Test
    fun `should receive message when subscribed to room`() {
        val latch = java.util.concurrent.CountDownLatch(1)
        val receivedMessage = java.util.concurrent.atomic.AtomicReference<ChatMessage>()
        
        // Subscribe to room
        stompSession.subscribe("/topic/rooms/room-1", object : StompFrameHandler {
            override fun getPayloadType(headers: StompHeaders) = ChatMessage::class.java
            
            override fun handleFrame(headers: StompHeaders, payload: Any?) {
                receivedMessage.set(payload as ChatMessage)
                latch.countDown()
            }
        })
        
        // Send message
        stompSession.send("/app/chat.sendMessage", ChatMessage(
            roomId = "room-1",
            senderId = "user-1",
            senderName = "Alice",
            content = "Hello World!"
        ))
        
        // Wait for message
        assertTrue(latch.await(5, java.util.concurrent.TimeUnit.SECONDS), "Message not received in time")
        assertEquals("Hello World!", receivedMessage.get().content)
    }
    
    @Test
    fun `should receive typing indicator`() {
        val latch = java.util.concurrent.CountDownLatch(1)
        val received = java.util.concurrent.atomic.AtomicReference<TypingEvent>()
        
        stompSession.subscribe("/topic/rooms/room-1/typing", object : StompFrameHandler {
            override fun getPayloadType(headers: StompHeaders) = TypingEvent::class.java
            override fun handleFrame(headers: StompHeaders, payload: Any?) {
                received.set(payload as TypingEvent)
                latch.countDown()
            }
        })
        
        stompSession.send("/app/chat.typing", TypingEvent("room-1", "user-2", true))
        
        assertTrue(latch.await(5, java.util.concurrent.TimeUnit.SECONDS))
        assertTrue(received.get().isTyping)
    }
}
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Implement read receipts

// When user reads a message, broadcast to other participants

data class ReadReceipt(
    val roomId: String,
    val messageId: String,
    val userId: String,
    val readAt: Long = System.currentTimeMillis()
)

@Controller
class ReadReceiptController(
    private val simpMessagingTemplate: SimpMessagingTemplate,
    private val messageRepository: MessageRepository
) {
    
    @MessageMapping("/chat.markRead")
    fun markAsRead(
        @Payload receipt: ReadReceipt,
        principal: Principal
    ) {
        val updatedReceipt = receipt.copy(userId = principal.name)
        
        // TODO: Save to database
        // TODO: Update message read status
        // TODO: Broadcast to room members
        // TODO: Handle "all read" notification when everyone has read
        
        simpMessagingTemplate.convertAndSend(
            "/topic/rooms/${receipt.roomId}/receipts",
            updatedReceipt
        )
    }
}

interface MessageRepository {
    fun markAsRead(messageId: String, userId: String)
    fun getReadReceipts(messageId: String): List<ReadReceipt>
    fun areAllRead(messageId: String, participantCount: Int): Boolean
}
```

---

## สรุป Part 60

```
✅ Spring WebSocket: full-duplex communication
✅ STOMP: higher-level messaging protocol over WebSocket
✅ @EnableWebSocketMessageBroker: Spring config
✅ SockJS: WebSocket fallback for older browsers
✅ @MessageMapping: handle inbound messages
✅ @SendTo: broadcast to topic/destination
✅ SimpMessagingTemplate: programmatic message sending
✅ convertAndSendToUser: send to specific user
✅ Presence System: track online users with Redis
✅ SessionConnectedEvent: handle connection events
✅ SessionDisconnectEvent: handle disconnection cleanup
✅ TypingIndicator: real-time typing status
✅ JwtChannelInterceptor: authenticate STOMP CONNECT
✅ AbstractSecurityWebSocketMessageBrokerConfigurer: WS security
✅ WebSocketStompClient: integration testing
✅ Read receipts: message acknowledgment pattern
```

---

*Part 60/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
