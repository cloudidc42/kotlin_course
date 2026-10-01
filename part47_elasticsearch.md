# Part 47: Search ด้วย Elasticsearch

## สารบัญ
1. [Elasticsearch Concepts](#elasticsearch-concepts)
2. [Spring Data Elasticsearch](#spring-data-elasticsearch)
3. [Full-Text Search](#full-text-search)
4. [Aggregations](#aggregations)
5. [Autocomplete](#autocomplete)
6. [แบบฝึกหัด](#แบบฝึกหัด)

---

## Elasticsearch Concepts

```
Concepts ที่ต้องรู้:
- Index: คล้าย Table ใน RDBMS
- Document: คล้าย Row, เก็บเป็น JSON
- Field: คล้าย Column
- Shard: แบ่ง index เป็น chunks สำหรับ horizontal scaling
- Replica: สำเนา shard สำหรับ HA
- Mapping: schema ของ document

Search Types:
- Full-text search: วิเคราะห์ข้อความ, ตัดคำ, stemming
- Term search: exact match
- Range search: ตัวเลข/วันที่ ในช่วง
- Aggregations: group by + stats
- Geo search: ค้นหาตาม location
```

---

## Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-elasticsearch")
}
```

```yaml
# application.yml
spring:
  elasticsearch:
    uris: http://localhost:9200
    username: elastic
    password: ${ELASTIC_PASSWORD}
```

---

## Document Mapping

```kotlin
import org.springframework.data.annotation.Id
import org.springframework.data.elasticsearch.annotations.*
import java.time.LocalDateTime

@Document(indexName = "products")
@Setting(settingPath = "/elasticsearch/product-settings.json")
data class ProductDocument(
    @Id val id: String,
    
    @Field(type = FieldType.Text, analyzer = "thai_analyzer")
    val name: String,
    
    @Field(type = FieldType.Text, analyzer = "thai_analyzer")
    val description: String?,
    
    @Field(type = FieldType.Keyword)
    val category: String,
    
    @Field(type = FieldType.Double)
    val price: Double,
    
    @Field(type = FieldType.Integer)
    val stock: Int,
    
    @Field(type = FieldType.Keyword)
    val tags: List<String> = emptyList(),
    
    @Field(type = FieldType.Boolean)
    val active: Boolean = true,
    
    @Field(type = FieldType.Float)
    val rating: Float = 0f,
    
    @Field(type = FieldType.Integer)
    val reviewCount: Int = 0,
    
    @Field(type = FieldType.Object)
    val attributes: Map<String, String> = emptyMap(),
    
    @Field(type = FieldType.Date, format = [DateFormat.date_time])
    val createdAt: LocalDateTime = LocalDateTime.now(),
    
    // Completion field for autocomplete
    @CompletionField
    val suggest: Completion? = null
)

@Document(indexName = "users")
data class UserDocument(
    @Id val id: String,
    
    @MultiField(
        mainField = Field(type = FieldType.Text, analyzer = "standard"),
        otherFields = [
            InnerField(suffix = "keyword", type = FieldType.Keyword)
        ]
    )
    val username: String,
    
    @Field(type = FieldType.Keyword)
    val email: String,
    
    @Field(type = FieldType.Keyword)
    val role: String,
    
    @Field(type = FieldType.Date)
    val createdAt: LocalDateTime = LocalDateTime.now()
)
```

---

## Repository

```kotlin
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository

interface ProductSearchRepository : ElasticsearchRepository<ProductDocument, String> {
    
    // Derived queries
    fun findByCategory(category: String): List<ProductDocument>
    
    fun findByPriceBetween(min: Double, max: Double): List<ProductDocument>
    
    fun findByActiveTrue(): List<ProductDocument>
    
    fun findByTagsContaining(tag: String): List<ProductDocument>
    
    fun findByRatingGreaterThanEqual(minRating: Float): List<ProductDocument>
    
    // Custom query with @Query annotation
    @Query("""
        {
            "bool": {
                "must": [
                    {"match": {"name": "?0"}},
                    {"term": {"active": true}}
                ],
                "filter": [
                    {"range": {"price": {"gte": ?1, "lte": ?2}}}
                ]
            }
        }
    """)
    fun searchByNameAndPriceRange(name: String, minPrice: Double, maxPrice: Double): List<ProductDocument>
    
    // Full text search
    @Query("""
        {
            "multi_match": {
                "query": "?0",
                "fields": ["name^3", "description^1", "tags^2"],
                "type": "best_fields",
                "fuzziness": "AUTO"
            }
        }
    """)
    fun fullTextSearch(query: String): List<ProductDocument>
}
```

---

## Full-Text Search ขั้นสูง

```kotlin
import co.elastic.clients.elasticsearch._types.query_dsl.*
import co.elastic.clients.elasticsearch.core.SearchRequest
import org.springframework.data.elasticsearch.client.elc.ElasticsearchTemplate
import org.springframework.data.elasticsearch.core.SearchHits
import org.springframework.data.elasticsearch.core.query.Criteria
import org.springframework.data.elasticsearch.core.query.CriteriaQuery
import org.springframework.data.elasticsearch.core.query.NativeQuery
import org.springframework.stereotype.Service

@Service
class ProductSearchService(
    private val elasticsearchTemplate: ElasticsearchTemplate,
    private val productRepository: ProductSearchRepository
) {
    
    // Complex search with filters
    fun search(request: ProductSearchRequest): SearchResult<ProductDocument> {
        val query = NativeQuery.builder()
            .withQuery { qBuilder ->
                qBuilder.bool { boolQuery ->
                    // Full-text on name and description
                    if (request.keyword?.isNotBlank() == true) {
                        boolQuery.must { mustBuilder ->
                            mustBuilder.multiMatch { mm ->
                                mm.query(request.keyword)
                                mm.fields("name^3", "description^1", "tags^2")
                                mm.type(TextQueryType.BestFields)
                                mm.fuzziness("AUTO")
                            }
                        }
                    }
                    
                    // Filter by category
                    request.category?.let { cat ->
                        boolQuery.filter { filterBuilder ->
                            filterBuilder.term { it.field("category").value(cat) }
                        }
                    }
                    
                    // Filter by price range
                    if (request.minPrice != null || request.maxPrice != null) {
                        boolQuery.filter { filterBuilder ->
                            filterBuilder.range { range ->
                                range.field("price").apply {
                                    request.minPrice?.let { gte(it.toString()) }
                                    request.maxPrice?.let { lte(it.toString()) }
                                }
                            }
                        }
                    }
                    
                    // Filter by tags
                    request.tags?.forEach { tag ->
                        boolQuery.filter { filterBuilder ->
                            filterBuilder.term { it.field("tags").value(tag) }
                        }
                    }
                    
                    // Only active products
                    boolQuery.filter { filterBuilder ->
                        filterBuilder.term { it.field("active").value(true) }
                    }
                    
                    boolQuery
                }
            }
            .withSort { sortBuilder ->
                when (request.sortBy) {
                    "price_asc" -> sortBuilder.field { it.field("price").order(SortOrder.Asc) }
                    "price_desc" -> sortBuilder.field { it.field("price").order(SortOrder.Desc) }
                    "rating" -> sortBuilder.field { it.field("rating").order(SortOrder.Desc) }
                    else -> sortBuilder.score { it.order(SortOrder.Desc) }  // relevance
                }
            }
            .withPageable(
                org.springframework.data.domain.PageRequest.of(
                    request.page,
                    request.perPage
                )
            )
            // Highlight matching terms
            .withHighlightQuery(
                org.springframework.data.elasticsearch.core.query.HighlightQuery(
                    org.springframework.data.elasticsearch.core.query.highlight.Highlight(
                        listOf(
                            org.springframework.data.elasticsearch.core.query.highlight.HighlightField("name"),
                            org.springframework.data.elasticsearch.core.query.highlight.HighlightField("description")
                        )
                    ),
                    ProductDocument::class.java
                )
            )
            .build()
        
        val hits: SearchHits<ProductDocument> = elasticsearchTemplate.search(
            query, ProductDocument::class.java
        )
        
        return SearchResult(
            items = hits.searchHits.map { hit ->
                hit.content.copy()
            },
            total = hits.totalHits,
            page = request.page,
            perPage = request.perPage,
            highlights = hits.searchHits.associate { hit ->
                hit.content.id to hit.highlightFields
            }
        )
    }
    
    fun syncFromDatabase(product: ProductDomain) {
        val document = ProductDocument(
            id = product.id,
            name = product.name,
            description = product.description,
            category = product.categoryName,
            price = product.price,
            stock = product.stock,
            tags = product.tags,
            active = product.active,
            rating = product.rating,
            suggest = Completion(arrayOf(product.name))
        )
        productRepository.save(document)
    }
    
    fun delete(id: String) {
        productRepository.deleteById(id)
    }
}

data class ProductSearchRequest(
    val keyword: String? = null,
    val category: String? = null,
    val minPrice: Double? = null,
    val maxPrice: Double? = null,
    val tags: List<String>? = null,
    val sortBy: String = "relevance",
    val page: Int = 0,
    val perPage: Int = 20
)

data class SearchResult<T>(
    val items: List<T>,
    val total: Long,
    val page: Int,
    val perPage: Int,
    val highlights: Map<String, Map<String, List<String>>> = emptyMap()
)

data class ProductDomain(
    val id: String,
    val name: String,
    val description: String?,
    val categoryName: String,
    val price: Double,
    val stock: Int,
    val tags: List<String>,
    val active: Boolean,
    val rating: Float
)
```

---

## Aggregations

```kotlin
@Service
class SearchAggregationService(
    private val elasticsearchTemplate: ElasticsearchTemplate
) {
    
    fun getFacets(keyword: String?): SearchFacets {
        val query = NativeQuery.builder()
            .withQuery { qBuilder ->
                if (keyword?.isNotBlank() == true) {
                    qBuilder.multiMatch { mm ->
                        mm.query(keyword)
                        mm.fields("name^3", "description^1")
                    }
                } else {
                    qBuilder.matchAll { it }
                }
            }
            .withAggregation("categories", AggregationBuilders
                .terms("categories")
                .field("category")
                .size(20))
            .withAggregation("price_ranges", AggregationBuilders
                .range("price_ranges")
                .field("price")
                .addRange(0.0, 500.0)
                .addRange(500.0, 1000.0)
                .addRange(1000.0, 5000.0)
                .addRange(5000.0, null))
            .withAggregation("avg_rating", AggregationBuilders
                .avg("avg_rating")
                .field("rating"))
            .withAggregation("price_stats", AggregationBuilders
                .stats("price_stats")
                .field("price"))
            .withMaxResults(0)  // just aggregations, no documents
            .build()
        
        val hits = elasticsearchTemplate.search(query, ProductDocument::class.java)
        
        val categoryAgg = hits.aggregations?.get<Terms>("categories")
        val priceRangeAgg = hits.aggregations?.get<Range>("price_ranges")
        val avgRating = hits.aggregations?.get<Avg>("avg_rating")
        val priceStats = hits.aggregations?.get<Stats>("price_stats")
        
        return SearchFacets(
            categories = categoryAgg?.buckets()?.array()?.map {
                FacetItem(key = it.key(), count = it.docCount())
            } ?: emptyList(),
            priceRanges = priceRangeAgg?.buckets()?.array()?.map {
                FacetRange(
                    key = it.key(),
                    from = it.from(),
                    to = it.to(),
                    count = it.docCount()
                )
            } ?: emptyList(),
            avgRating = avgRating?.value() ?: 0.0,
            priceMin = priceStats?.min() ?: 0.0,
            priceMax = priceStats?.max() ?: 0.0
        )
    }
}

data class SearchFacets(
    val categories: List<FacetItem>,
    val priceRanges: List<FacetRange>,
    val avgRating: Double,
    val priceMin: Double,
    val priceMax: Double
)

data class FacetItem(val key: String, val count: Long)
data class FacetRange(val key: String, val from: Double?, val to: Double?, val count: Long)
```

---

## Autocomplete

```kotlin
@Service
class AutocompleteService(
    private val elasticsearchTemplate: ElasticsearchTemplate
) {
    
    fun suggest(prefix: String, size: Int = 5): List<String> {
        val query = NativeQuery.builder()
            .withSuggester { suggester ->
                suggester.suggesters("product-suggest") { suggest ->
                    suggest.prefix(prefix)
                    suggest.completion { completion ->
                        completion.field("suggest")
                        completion.size(size)
                        completion.skipDuplicates(true)
                        completion.fuzzy { fuzzy ->
                            fuzzy.fuzziness("AUTO")
                        }
                    }
                }
            }
            .withMaxResults(0)
            .build()
        
        val hits = elasticsearchTemplate.search(query, ProductDocument::class.java)
        
        return hits.suggest
            ?.get("product-suggest")
            ?.flatMap { it.options }
            ?.map { it.text }
            ?: emptyList()
    }
    
    // Simple prefix query as alternative
    fun prefixSearch(prefix: String, size: Int = 5): List<String> {
        val query = NativeQuery.builder()
            .withQuery { qBuilder ->
                qBuilder.prefix { pq ->
                    pq.field("name.keyword")
                    pq.value(prefix.lowercase())
                }
            }
            .withMaxResults(size)
            .withFields("name")
            .build()
        
        return elasticsearchTemplate.search(query, ProductDocument::class.java)
            .searchHits
            .map { it.content.name }
    }
}

// Placeholder type imports
typealias Completion = org.springframework.data.elasticsearch.core.suggest.Completion
typealias Terms = co.elastic.clients.elasticsearch._types.aggregations.StringTermsAggregate
typealias Range = co.elastic.clients.elasticsearch._types.aggregations.RangeAggregate
typealias Avg = co.elastic.clients.elasticsearch._types.aggregations.AvgAggregate
typealias Stats = co.elastic.clients.elasticsearch._types.aggregations.StatsAggregate
typealias AggregationBuilders = co.elastic.clients.elasticsearch._types.aggregations.AggregationBuilders
typealias SortOrder = co.elastic.clients.elasticsearch._types.SortOrder
typealias TextQueryType = co.elastic.clients.elasticsearch._types.query_dsl.TextQueryType
```

---

## แบบฝึกหัด

```kotlin
// Exercise: Build a job search engine

@Document(indexName = "jobs")
data class JobDocument(
    @Id val id: String,
    
    @Field(type = FieldType.Text)
    val title: String,
    
    @Field(type = FieldType.Text)
    val description: String,
    
    @Field(type = FieldType.Keyword)
    val company: String,
    
    @Field(type = FieldType.Keyword)
    val location: String,
    
    @Field(type = FieldType.Keyword)
    val employmentType: String,  // FULL_TIME, PART_TIME, CONTRACT, REMOTE
    
    @Field(type = FieldType.Keyword)
    val skills: List<String>,
    
    @Field(type = FieldType.Integer)
    val salaryMin: Int?,
    
    @Field(type = FieldType.Integer)
    val salaryMax: Int?,
    
    @Field(type = FieldType.Date)
    val postedAt: LocalDateTime = LocalDateTime.now(),
    
    @CompletionField
    val suggest: Completion? = null
)

data class JobSearchRequest(
    val keyword: String? = null,
    val location: String? = null,
    val employmentType: String? = null,
    val skills: List<String>? = null,
    val minSalary: Int? = null,
    val maxSalary: Int? = null,
    val postedWithinDays: Int? = null,
    val page: Int = 0,
    val perPage: Int = 20
)

// TODO: Implement JobSearchService with:
// 1. search(request): Full-text on title + description, filter by location/type/skills/salary
// 2. suggest(prefix): Autocomplete for job titles
// 3. getFacets(keyword): Aggregations by location, employment type, skills
// 4. getRelatedJobs(jobId): Jobs similar to given job (more-like-this query)

class JobSearchService(private val template: ElasticsearchTemplate) {
    fun search(request: JobSearchRequest): SearchResult<JobDocument> = TODO()
    fun suggest(prefix: String): List<String> = TODO()
    fun getFacets(keyword: String?): SearchFacets = TODO()
    fun getRelatedJobs(jobId: String, size: Int = 5): List<JobDocument> = TODO()
}
```

---

## สรุป Part 47

```
✅ Elasticsearch: distributed search and analytics
✅ @Document: index mapping annotation
✅ @Field: field type mapping (Text, Keyword, Date, etc.)
✅ @MultiField: same field with multiple analyzers
✅ @CompletionField: autocomplete
✅ ElasticsearchRepository: Spring Data CRUD
✅ NativeQuery: complex queries with full ES API
✅ bool query: must/should/filter/must_not
✅ multi_match: search across multiple fields with boost
✅ Highlighting: show matching text snippets
✅ Aggregations: faceted search (terms, range, avg, stats)
✅ Completion suggester: fast autocomplete
✅ Fuzzy matching: handle typos
```

---

*Part 47/100 | หลักสูตร Kotlin ฉบับสมบูรณ์*
