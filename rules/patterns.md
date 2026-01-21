# Common Patterns

## API Response Format

```kotlin
// Sealed class for type-safe API responses
sealed class ApiResponse<out T> {
    data class Success<T>(
        val data: T,
        val meta: Meta? = null
    ) : ApiResponse<T>()

    data class Error(
        val error: String,
        val code: String? = null,
        val details: Map<String, Any>? = null
    ) : ApiResponse<Nothing>()
}

data class Meta(
    val total: Long,
    val page: Int,
    val limit: Int
)

// Or simple data class approach
data class ApiResult<T>(
    val success: Boolean,
    val data: T? = null,
    val error: String? = null,
    val meta: Meta? = null
)
```

## Service Layer Pattern

```kotlin
@Service
class MarketService(
    private val marketRepository: MarketRepository,
    private val cacheService: CacheService
) {
    @Transactional(readOnly = true)
    fun findAll(filters: MarketFilters): List<Market> {
        return marketRepository.findByFilters(filters)
    }

    @Transactional(readOnly = true)
    fun findById(id: Long): Market? {
        return marketRepository.findById(id).orElse(null)
    }

    @Transactional
    fun create(request: CreateMarketRequest): Market {
        val market = Market(
            name = request.name,
            slug = request.slug,
            status = MarketStatus.DRAFT
        )
        return marketRepository.save(market)
    }

    @Transactional
    fun update(id: Long, request: UpdateMarketRequest): Market {
        val market = marketRepository.findById(id)
            .orElseThrow { EntityNotFoundException("Market not found") }
        return marketRepository.save(market.copy(
            name = request.name ?: market.name,
            status = request.status ?: market.status
        ))
    }

    @Transactional
    fun delete(id: Long) {
        marketRepository.deleteById(id)
    }
}
```

## Repository Pattern

```kotlin
interface MarketRepository : JpaRepository<Market, Long> {
    fun findBySlug(slug: String): Market?
    fun findByStatus(status: MarketStatus): List<Market>

    @Query("SELECT m FROM Market m WHERE m.name LIKE %:query% OR m.description LIKE %:query%")
    fun searchByQuery(@Param("query") query: String): List<Market>

    @Query("SELECT m FROM Market m WHERE m.status = :status ORDER BY m.createdAt DESC")
    fun findByFilters(@Param("status") status: MarketStatus?, pageable: Pageable): Page<Market>
}

// Custom repository implementation
interface MarketRepositoryCustom {
    fun findByFilters(filters: MarketFilters): List<Market>
}

class MarketRepositoryCustomImpl(
    private val entityManager: EntityManager
) : MarketRepositoryCustom {
    override fun findByFilters(filters: MarketFilters): List<Market> {
        val cb = entityManager.criteriaBuilder
        val query = cb.createQuery(Market::class.java)
        val root = query.from(Market::class.java)

        val predicates = mutableListOf<Predicate>()
        filters.status?.let { predicates.add(cb.equal(root.get<MarketStatus>("status"), it)) }
        filters.query?.let { predicates.add(cb.like(root.get("name"), "%$it%")) }

        query.where(*predicates.toTypedArray())
        return entityManager.createQuery(query).resultList
    }
}
```

## Skeleton Projects

When implementing new functionality:
1. Search for battle-tested skeleton projects
2. Use parallel agents to evaluate options:
   - Security assessment
   - Extensibility analysis
   - Relevance scoring
   - Implementation planning
3. Clone best match as foundation
4. Iterate within proven structure
