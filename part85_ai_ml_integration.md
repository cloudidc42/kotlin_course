# Part 85: AI/ML Integration — LLM APIs & Embeddings

## สารบัญ
1. [AI Integration Patterns](#ai-integration-patterns)
2. [LLM API Integration](#llm-api-integration)
3. [Embeddings & Vector Search](#embeddings--vector-search)
4. [RAG (Retrieval-Augmented Generation)](#rag)
5. [AI-Powered Features](#ai-powered-features)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## AI Integration Patterns

```
AI/ML ใน Backend Application:

1. LLM APIs: OpenAI, Anthropic, Google Gemini
   - Text generation, classification, summarization
   - Chat, Q&A, code generation

2. Embeddings: Vector representations ของ text
   - Semantic search: ค้นหาด้วยความหมาย ไม่ใช่ keyword
   - Recommendation: สินค้าที่คล้ายกัน
   - Duplicate detection: หา content ที่ซ้ำกัน

3. RAG: Retrieval-Augmented Generation
   - ดึงข้อมูลที่เกี่ยวข้องจาก vector DB
   - ส่งเป็น context ให้ LLM ตอบคำถาม
   - ป้องกัน hallucination

4. Fine-tuning: ปรับ model สำหรับ use case เฉพาะ
   - Classification: category prediction
   - NER: extract entities

Best Practices:
- Rate limiting: avoid API throttling
- Retry with exponential backoff
- Caching: cache identical prompts
- Cost tracking: monitor token usage
- Fallback: graceful degradation เมื่อ API down
```

---

## LLM API Integration

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.aallam.openai:openai-client:3.6.3")
    implementation("io.ktor:ktor-client-okhttp:2.3.7")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
    implementation("io.github.resilience4j:resilience4j-kotlin:2.1.0")
    implementation("io.github.resilience4j:resilience4j-retry:2.1.0")
    implementation("io.github.resilience4j:resilience4j-ratelimiter:2.1.0")
    implementation("com.pgvector:pgvector:0.1.4")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
}

// OpenAI Client wrapper
import com.aallam.openai.api.chat.*
import com.aallam.openai.api.model.ModelId
import com.aallam.openai.client.OpenAI
import com.aallam.openai.client.OpenAIConfig
import io.github.resilience4j.retry.Retry
import io.github.resilience4j.retry.RetryConfig
import io.github.resilience4j.ratelimiter.RateLimiter
import io.github.resilience4j.ratelimiter.RateLimiterConfig
import kotlinx.coroutines.flow.Flow
import java.time.Duration

@org.springframework.stereotype.Service
class LlmService(
    private val openAI: OpenAI,
    private val tokenUsageTracker: TokenUsageTracker,
    private val promptCache: PromptCache
) {
    
    private val retry: Retry = Retry.of("openai-retry", RetryConfig.custom<Any>()
        .maxAttempts(3)
        .waitDuration(Duration.ofSeconds(1))
        .retryOnException { e -> e is com.aallam.openai.api.exception.OpenAITimeoutException }
        .build()
    )
    
    private val rateLimiter: RateLimiter = RateLimiter.of("openai-rate", RateLimiterConfig.custom()
        .limitForPeriod(100)
        .limitRefreshPeriod(Duration.ofMinutes(1))
        .timeoutDuration(Duration.ofSeconds(30))
        .build()
    )
    
    suspend fun chat(
        systemPrompt: String,
        userMessage: String,
        model: String = "gpt-4o-mini",
        maxTokens: Int = 1000
    ): String {
        val cacheKey = "$model:${systemPrompt.hashCode()}:${userMessage.hashCode()}"
        
        // Check cache first
        promptCache.get(cacheKey)?.let { return it }
        
        rateLimiter.acquirePermission()
        
        val response = retry.executeSuspendFunction {
            openAI.chatCompletion(
                ChatCompletionRequest(
                    model = ModelId(model),
                    messages = listOf(
                        ChatMessage(role = ChatRole.System, content = systemPrompt),
                        ChatMessage(role = ChatRole.User, content = userMessage)
                    ),
                    maxTokens = maxTokens,
                    temperature = 0.7
                )
            )
        }
        
        val content = response.choices.first().message.content ?: ""
        val usage = response.usage
        
        // Track token usage
        tokenUsageTracker.track(
            model = model,
            promptTokens = usage?.promptTokens ?: 0,
            completionTokens = usage?.completionTokens ?: 0
        )
        
        // Cache the response
        promptCache.set(cacheKey, content, Duration.ofHours(1))
        
        return content
    }
    
    fun streamChat(
        systemPrompt: String,
        userMessage: String,
        model: String = "gpt-4o-mini"
    ): Flow<String> {
        return openAI.chatCompletions(
            ChatCompletionRequest(
                model = ModelId(model),
                messages = listOf(
                    ChatMessage(role = ChatRole.System, content = systemPrompt),
                    ChatMessage(role = ChatRole.User, content = userMessage)
                ),
                stream = true
            )
        ).let { chunksFlow ->
            kotlinx.coroutines.flow.transform(chunksFlow) { chunk ->
                chunk.choices.firstOrNull()?.delta?.content?.let { emit(it) }
            }
        }
    }
    
    suspend fun extractStructured(
        text: String,
        schema: String
    ): String {
        return chat(
            systemPrompt = """
                You are a data extraction assistant. Extract structured data from the provided text.
                Return ONLY valid JSON matching this schema, no explanation:
                $schema
            """.trimIndent(),
            userMessage = text
        )
    }
    
    suspend fun classify(
        text: String,
        categories: List<String>
    ): String {
        return chat(
            systemPrompt = """
                Classify the following text into exactly one of these categories: ${categories.joinToString(", ")}.
                Respond with ONLY the category name, nothing else.
            """.trimIndent(),
            userMessage = text
        )
    }
}

// Config
@org.springframework.context.annotation.Configuration
class AiConfig {
    
    @org.springframework.context.annotation.Bean
    fun openAI(): OpenAI {
        return OpenAI(OpenAIConfig(
            token = System.getenv("OPENAI_API_KEY") ?: throw IllegalStateException("OPENAI_API_KEY not set")
        ))
    }
    
    @org.springframework.context.annotation.Bean
    fun promptCache(redisTemplate: org.springframework.data.redis.core.StringRedisTemplate): PromptCache {
        return RedisPromptCache(redisTemplate)
    }
}

@org.springframework.stereotype.Service
class TokenUsageTracker(
    private val meterRegistry: io.micrometer.core.instrument.MeterRegistry
) {
    fun track(model: String, promptTokens: Int, completionTokens: Int) {
        meterRegistry.counter("ai.tokens.prompt", "model", model).increment(promptTokens.toDouble())
        meterRegistry.counter("ai.tokens.completion", "model", model).increment(completionTokens.toDouble())
    }
}

interface PromptCache {
    fun get(key: String): String?
    fun set(key: String, value: String, ttl: Duration)
}

class RedisPromptCache(private val redisTemplate: org.springframework.data.redis.core.StringRedisTemplate) : PromptCache {
    override fun get(key: String) = redisTemplate.opsForValue().get("prompt:$key")
    override fun set(key: String, value: String, ttl: Duration) {
        redisTemplate.opsForValue().set("prompt:$key", value, ttl)
    }
}

// Resilience4j extension for suspend functions
suspend fun <T> Retry.executeSuspendFunction(block: suspend () -> T): T {
    var lastException: Exception? = null
    repeat(config.maxAttempts) { attempt ->
        try {
            return block()
        } catch (e: Exception) {
            lastException = e
            if (attempt < config.maxAttempts - 1) {
                kotlinx.coroutines.delay(config.intervalFunction.apply(attempt + 1L))
            }
        }
    }
    throw lastException!!
}
```

---

## Embeddings & Vector Search

```kotlin
// Embedding generation and vector search

@org.springframework.stereotype.Service
class EmbeddingService(
    private val openAI: OpenAI,
    private val embeddingRepository: EmbeddingRepository
) {
    
    suspend fun generateEmbedding(text: String): FloatArray {
        val response = openAI.embeddings(
            com.aallam.openai.api.embedding.EmbeddingRequest(
                model = ModelId("text-embedding-3-small"),
                input = listOf(text)
            )
        )
        return response.embeddings.first().embedding.map { it.toFloat() }.toFloatArray()
    }
    
    suspend fun generateBatchEmbeddings(texts: List<String>): List<FloatArray> {
        val response = openAI.embeddings(
            com.aallam.openai.api.embedding.EmbeddingRequest(
                model = ModelId("text-embedding-3-small"),
                input = texts
            )
        )
        return response.embeddings
            .sortedBy { it.index }
            .map { it.embedding.map { d -> d.toFloat() }.toFloatArray() }
    }
    
    fun cosineSimilarity(a: FloatArray, b: FloatArray): Float {
        require(a.size == b.size)
        var dotProduct = 0f
        var normA = 0f
        var normB = 0f
        for (i in a.indices) {
            dotProduct += a[i] * b[i]
            normA += a[i] * a[i]
            normB += b[i] * b[i]
        }
        return dotProduct / (Math.sqrt(normA.toDouble()) * Math.sqrt(normB.toDouble())).toFloat()
    }
}

// pgvector integration with JPA
@jakarta.persistence.Entity
@jakarta.persistence.Table(name = "product_embeddings")
data class ProductEmbedding(
    @jakarta.persistence.Id
    val productId: String,
    
    val content: String,  // text that was embedded
    
    @org.hibernate.annotations.JdbcTypeCode(org.hibernate.type.SqlTypes.VECTOR)
    @org.hibernate.annotations.Array(length = 1536)  // text-embedding-3-small dimension
    val embedding: FloatArray,
    
    val updatedAt: java.time.Instant = java.time.Instant.now()
)

// Repository with vector similarity search
@org.springframework.data.jpa.repository.JpaRepository
interface EmbeddingRepository : org.springframework.data.jpa.repository.JpaRepository<ProductEmbedding, String> {
    
    @org.springframework.data.jpa.repository.Query(
        value = """
            SELECT pe.product_id, pe.content,
                   1 - (pe.embedding <=> CAST(:queryVector AS vector)) AS similarity
            FROM product_embeddings pe
            ORDER BY pe.embedding <=> CAST(:queryVector AS vector)
            LIMIT :limit
        """,
        nativeQuery = true
    )
    fun findSimilar(
        @org.springframework.data.repository.query.Param("queryVector") queryVector: String,
        @org.springframework.data.repository.query.Param("limit") limit: Int = 10
    ): List<SimilarityResult>
    
    @org.springframework.data.jpa.repository.Query(
        value = """
            SELECT pe.product_id, pe.content,
                   1 - (pe.embedding <=> CAST(:queryVector AS vector)) AS similarity
            FROM product_embeddings pe
            WHERE 1 - (pe.embedding <=> CAST(:queryVector AS vector)) > :threshold
            ORDER BY similarity DESC
            LIMIT :limit
        """,
        nativeQuery = true
    )
    fun findSimilarWithThreshold(
        @org.springframework.data.repository.query.Param("queryVector") queryVector: String,
        @org.springframework.data.repository.query.Param("threshold") threshold: Float = 0.7f,
        @org.springframework.data.repository.query.Param("limit") limit: Int = 10
    ): List<SimilarityResult>
}

interface SimilarityResult {
    fun getProductId(): String
    fun getContent(): String
    fun getSimilarity(): Float
}

// Semantic search service
@org.springframework.stereotype.Service
class SemanticSearchService(
    private val embeddingService: EmbeddingService,
    private val embeddingRepository: EmbeddingRepository,
    private val productRepository: ProductRepository
) {
    
    suspend fun semanticSearch(query: String, limit: Int = 10): List<ProductSearchResult> {
        val queryEmbedding = embeddingService.generateEmbedding(query)
        val vectorString = queryEmbedding.joinToString(",", prefix = "[", postfix = "]")
        
        val similar = embeddingRepository.findSimilar(vectorString, limit)
        
        val productIds = similar.map { it.getProductId() }
        val products = productRepository.findAllById(productIds).associateBy { it.id }
        
        return similar.mapNotNull { result ->
            products[result.getProductId()]?.let { product ->
                ProductSearchResult(
                    product = product,
                    similarity = result.getSimilarity(),
                    snippet = result.getContent().take(200)
                )
            }
        }
    }
    
    suspend fun indexProduct(product: Product) {
        val content = buildEmbeddingContent(product)
        val embedding = embeddingService.generateEmbedding(content)
        
        embeddingRepository.save(ProductEmbedding(
            productId = product.id,
            content = content,
            embedding = embedding
        ))
    }
    
    private fun buildEmbeddingContent(product: Product): String {
        return """
            Product: ${product.name}
            Category: ${product.category}
            Description: ${product.description}
            Tags: ${product.tags.joinToString(", ")}
            Price: ${product.price} ${product.currency}
        """.trimIndent()
    }
    
    // Find similar products
    suspend fun findSimilarProducts(productId: String, limit: Int = 5): List<ProductSearchResult> {
        val sourceEmbedding = embeddingRepository.findById(productId)
            .orElseThrow { NotFoundException("Product embedding not found: $productId") }
        
        val vectorString = sourceEmbedding.embedding.joinToString(",", prefix = "[", postfix = "]")
        
        return embeddingRepository.findSimilarWithThreshold(vectorString, threshold = 0.8f, limit = limit + 1)
            .filter { it.getProductId() != productId }
            .take(limit)
            .mapNotNull { result ->
                productRepository.findById(result.getProductId()).orElse(null)?.let { product ->
                    ProductSearchResult(product, result.getSimilarity(), "")
                }
            }
    }
}

data class ProductSearchResult(
    val product: Product,
    val similarity: Float,
    val snippet: String
)

// Migration for pgvector
// V10__Add_vector_extension.sql:
// CREATE EXTENSION IF NOT EXISTS vector;
// CREATE TABLE product_embeddings (
//   product_id TEXT PRIMARY KEY,
//   content TEXT NOT NULL,
//   embedding vector(1536) NOT NULL,
//   updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
// );
// CREATE INDEX product_embeddings_embedding_idx
//   ON product_embeddings USING ivfflat (embedding vector_cosine_ops)
//   WITH (lists = 100);
```

---

## RAG (Retrieval-Augmented Generation)

```kotlin
// RAG system for customer support Q&A

@org.springframework.stereotype.Service
class RagService(
    private val llmService: LlmService,
    private val embeddingService: EmbeddingService,
    private val knowledgeBaseRepository: KnowledgeBaseRepository
) {
    
    suspend fun answer(question: String, userId: String): RagResponse {
        // 1. Retrieve relevant documents
        val queryEmbedding = embeddingService.generateEmbedding(question)
        val vectorString = queryEmbedding.joinToString(",", prefix = "[", postfix = "]")
        val relevantDocs = knowledgeBaseRepository.findRelevant(vectorString, limit = 5, threshold = 0.7f)
        
        if (relevantDocs.isEmpty()) {
            return RagResponse(
                answer = "ขอโทษนะครับ ไม่พบข้อมูลที่เกี่ยวข้องกับคำถามของคุณ กรุณาติดต่อ support@ecommerce.com",
                sources = emptyList(),
                confidence = 0.0f
            )
        }
        
        // 2. Build context from retrieved documents
        val context = relevantDocs.joinToString("\n\n---\n\n") { doc ->
            "Source: ${doc.getTitle()}\n${doc.getContent()}"
        }
        
        // 3. Generate answer with LLM
        val systemPrompt = """
            You are a helpful customer support assistant for ECommerce platform.
            Answer questions based ONLY on the provided context. 
            If the answer is not in the context, say you don't know.
            Always respond in Thai language.
            Be concise and helpful.
            
            Context:
            $context
        """.trimIndent()
        
        val answer = llmService.chat(
            systemPrompt = systemPrompt,
            userMessage = question,
            model = "gpt-4o-mini",
            maxTokens = 500
        )
        
        val avgConfidence = relevantDocs.map { it.getSimilarity() }.average().toFloat()
        
        return RagResponse(
            answer = answer,
            sources = relevantDocs.map { RagSource(it.getTitle(), it.getDocumentId()) },
            confidence = avgConfidence
        )
    }
    
    // Streaming version for real-time response
    fun streamAnswer(question: String): Flow<String> = kotlinx.coroutines.flow.flow {
        val context = getRelevantContext(question)
        
        val systemPrompt = """
            You are a helpful customer support assistant. 
            Answer based on context. Respond in Thai.
            Context: $context
        """.trimIndent()
        
        llmService.streamChat(systemPrompt, question).collect { chunk ->
            emit(chunk)
        }
    }
    
    private suspend fun getRelevantContext(question: String): String {
        val embedding = embeddingService.generateEmbedding(question)
        val vectorString = embedding.joinToString(",", prefix = "[", postfix = "]")
        val docs = knowledgeBaseRepository.findRelevant(vectorString, limit = 3, threshold = 0.6f)
        return docs.joinToString("\n\n") { it.getContent() }
    }
}

data class RagResponse(
    val answer: String,
    val sources: List<RagSource>,
    val confidence: Float
)

data class RagSource(val title: String, val documentId: String)

interface KnowledgeBaseRepository {
    fun findRelevant(queryVector: String, limit: Int, threshold: Float): List<KnowledgeDoc>
}

interface KnowledgeDoc {
    fun getDocumentId(): String
    fun getTitle(): String
    fun getContent(): String
    fun getSimilarity(): Float
}
```

---

## AI-Powered Features

```kotlin
// Product Description Generator
@org.springframework.stereotype.Service
class ProductDescriptionGenerator(private val llmService: LlmService) {
    
    suspend fun generate(
        productName: String,
        category: String,
        features: List<String>,
        targetAudience: String = "ทั่วไป"
    ): GeneratedDescription {
        val prompt = """
            สร้างคำอธิบายสินค้าภาษาไทยสำหรับ:
            ชื่อสินค้า: $productName
            หมวดหมู่: $category
            คุณสมบัติ: ${features.joinToString(", ")}
            กลุ่มเป้าหมาย: $targetAudience
            
            ต้องการ:
            1. ชื่อสินค้าที่น่าสนใจ (30-50 ตัวอักษร)
            2. คำอธิบายสั้น (50-100 ตัวอักษร)
            3. คำอธิบายยาว (200-300 ตัวอักษร)
            4. Bullet points 3-5 จุด
            
            ตอบในรูปแบบ JSON:
            {"shortName":"...","shortDesc":"...","longDesc":"...","bullets":["...","..."]}
        """.trimIndent()
        
        val json = llmService.chat(
            systemPrompt = "You are a Thai product copywriter. Respond only with valid JSON.",
            userMessage = prompt
        )
        
        return com.fasterxml.jackson.module.kotlin.jacksonObjectMapper()
            .readValue(json, GeneratedDescription::class.java)
    }
}

// Review Sentiment Analyzer
@org.springframework.stereotype.Service
class ReviewAnalyzer(private val llmService: LlmService) {
    
    suspend fun analyze(reviewText: String): ReviewAnalysis {
        val json = llmService.extractStructured(
            text = reviewText,
            schema = """
                {
                  "sentiment": "POSITIVE|NEGATIVE|NEUTRAL",
                  "score": 1-5,
                  "aspects": {
                    "quality": "POSITIVE|NEGATIVE|NEUTRAL|N/A",
                    "delivery": "POSITIVE|NEGATIVE|NEUTRAL|N/A",
                    "value": "POSITIVE|NEGATIVE|NEUTRAL|N/A",
                    "service": "POSITIVE|NEGATIVE|NEUTRAL|N/A"
                  },
                  "keyPhrases": ["phrase1", "phrase2"],
                  "language": "th|en|other",
                  "isSpam": true|false
                }
            """.trimIndent()
        )
        
        return com.fasterxml.jackson.module.kotlin.jacksonObjectMapper()
            .readValue(json, ReviewAnalysis::class.java)
    }
    
    suspend fun moderateContent(text: String): ModerationResult {
        val result = llmService.classify(
            text = text,
            categories = listOf("SAFE", "SPAM", "OFFENSIVE", "MISLEADING", "ADULT")
        )
        return ModerationResult(
            category = result,
            isAllowed = result == "SAFE"
        )
    }
}

// Search Intent Classifier
@org.springframework.stereotype.Service
class SearchIntentClassifier(private val llmService: LlmService) {
    
    suspend fun classify(query: String): SearchIntent {
        val json = llmService.extractStructured(
            text = query,
            schema = """
                {
                  "intent": "PRODUCT_SEARCH|CATEGORY_BROWSE|PRICE_COMPARISON|REVIEW_LOOKUP|ORDER_STATUS",
                  "extractedEntities": {
                    "productName": "string or null",
                    "category": "string or null", 
                    "priceRange": {"min": number|null, "max": number|null},
                    "brand": "string or null"
                  },
                  "suggestedQuery": "improved search query"
                }
            """.trimIndent()
        )
        
        return com.fasterxml.jackson.module.kotlin.jacksonObjectMapper()
            .readValue(json, SearchIntent::class.java)
    }
}

data class GeneratedDescription(
    val shortName: String,
    val shortDesc: String,
    val longDesc: String,
    val bullets: List<String>
)

data class ReviewAnalysis(
    val sentiment: String,
    val score: Int,
    val aspects: Map<String, String>,
    val keyPhrases: List<String>,
    val language: String,
    val isSpam: Boolean
)

data class ModerationResult(val category: String, val isAllowed: Boolean)

data class SearchIntent(
    val intent: String,
    val extractedEntities: Map<String, Any?>,
    val suggestedQuery: String
)
```

---

## แบบฝึกหัด

```kotlin
// Exercise: สร้าง AI Customer Support Chatbot ด้วย Conversation History

@org.springframework.stereotype.Service
class ChatbotService(
    private val llmService: LlmService,
    private val ragService: RagService,
    private val conversationRepository: ConversationRepository
) {
    
    private val systemPrompt = """
        คุณเป็น customer support assistant ของ ECommerce Thailand
        หน้าที่ของคุณ:
        1. ตอบคำถามเกี่ยวกับสินค้า การสั่งซื้อ และ shipping
        2. ช่วยแก้ปัญหาให้ลูกค้า
        3. ให้ข้อมูลที่ถูกต้องและเป็นประโยชน์
        
        กฎ:
        - ตอบเป็นภาษาไทยเสมอ
        - ใจเย็น เป็นมิตร และมืออาชีพ
        - ถ้าไม่รู้คำตอบ ให้บอกว่าจะส่งต่อให้ทีมงาน
        - ห้ามบอกข้อมูลส่วนตัวของลูกค้าท่านอื่น
    """.trimIndent()
    
    suspend fun chat(
        userId: String,
        sessionId: String,
        message: String
    ): ChatResponse {
        // Load conversation history
        val history = conversationRepository.getHistory(sessionId, limit = 10)
        
        // RAG: find relevant docs
        val ragResponse = ragService.answer(message, userId)
        
        // Build messages with history
        val messages = buildList {
            if (ragResponse.confidence > 0.7f) {
                add(ChatMessage(
                    role = ChatRole.System,
                    content = "$systemPrompt\n\nข้อมูลที่เกี่ยวข้อง:\n${ragResponse.answer}"
                ))
            } else {
                add(ChatMessage(role = ChatRole.System, content = systemPrompt))
            }
            
            history.forEach { msg ->
                add(ChatMessage(
                    role = if (msg.isFromUser) ChatRole.User else ChatRole.Assistant,
                    content = msg.content
                ))
            }
            
            add(ChatMessage(role = ChatRole.User, content = message))
        }
        
        val response = openAI.chatCompletion(
            ChatCompletionRequest(
                model = ModelId("gpt-4o-mini"),
                messages = messages,
                maxTokens = 500
            )
        )
        
        val botReply = response.choices.first().message.content ?: ""
        
        // Save to history
        conversationRepository.save(ConversationMessage(
            sessionId = sessionId,
            userId = userId,
            content = message,
            isFromUser = true,
            timestamp = java.time.Instant.now()
        ))
        conversationRepository.save(ConversationMessage(
            sessionId = sessionId,
            userId = userId,
            content = botReply,
            isFromUser = false,
            timestamp = java.time.Instant.now()
        ))
        
        return ChatResponse(
            message = botReply,
            sessionId = sessionId,
            sources = ragResponse.sources
        )
    }
}

data class ChatResponse(val message: String, val sessionId: String, val sources: List<RagSource>)

data class ConversationMessage(
    val sessionId: String,
    val userId: String,
    val content: String,
    val isFromUser: Boolean,
    val timestamp: java.time.Instant
)

interface ConversationRepository {
    fun getHistory(sessionId: String, limit: Int): List<ConversationMessage>
    fun save(message: ConversationMessage)
}

val openAI: OpenAI = TODO("inject")
```

---

## สรุป Part 85

```
✅ LLM Integration: openai-client for Kotlin
✅ Retry with Resilience4j: 3 attempts, exponential backoff
✅ RateLimiter: 100 requests/minute token bucket
✅ PromptCache: Redis-based response caching
✅ TokenUsageTracker: Micrometer counters for cost monitoring
✅ streamChat: Flow<String> for streaming responses
✅ extractStructured: JSON schema extraction
✅ classify: zero-shot classification
✅ EmbeddingService: text-embedding-3-small (1536 dim)
✅ cosineSimilarity: manual calculation
✅ pgvector: vector column with ivfflat index
✅ <=> operator: cosine distance in PostgreSQL
✅ SemanticSearch: query embedding → vector search → products
✅ findSimilarProducts: embedding-based recommendations
✅ RAG pipeline: retrieve → build context → generate
✅ Confidence threshold: fallback when no relevant docs
✅ KnowledgeBase: documents indexed with embeddings
✅ streamAnswer: streaming RAG responses
✅ ProductDescriptionGenerator: Thai copywriting with LLM
✅ ReviewAnalyzer: sentiment, aspects, spam detection
✅ ContentModeration: safety classification
✅ SearchIntentClassifier: extract entities from query
✅ Chatbot with conversation history (last 10 messages)
✅ RAG-augmented chatbot: inject relevant docs as context
```

---

*Part 85/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
