---
name: clickhouse-io
description: ClickHouse database patterns, query optimization, analytics, and data engineering best practices for high-performance analytical workloads with Kotlin/Spring Boot.
---

# ClickHouse Analytics Patterns

ClickHouse-specific patterns for high-performance analytics and data engineering with Kotlin/Spring Boot.

## Overview

ClickHouse is a column-oriented database management system (DBMS) for online analytical processing (OLAP). It's optimized for fast analytical queries on large datasets.

**Key Features:**
- Column-oriented storage
- Data compression
- Parallel query execution
- Distributed queries
- Real-time analytics

## Table Design Patterns

### MergeTree Engine (Most Common)

```sql
CREATE TABLE markets_analytics (
    date Date,
    market_id String,
    market_name String,
    volume UInt64,
    trades UInt32,
    unique_traders UInt32,
    avg_trade_size Float64,
    created_at DateTime
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, market_id)
SETTINGS index_granularity = 8192;
```

### ReplacingMergeTree (Deduplication)

```sql
-- For data that may have duplicates (e.g., from multiple sources)
CREATE TABLE user_events (
    event_id String,
    user_id String,
    event_type String,
    timestamp DateTime,
    properties String
) ENGINE = ReplacingMergeTree()
PARTITION BY toYYYYMM(timestamp)
ORDER BY (user_id, event_id, timestamp)
PRIMARY KEY (user_id, event_id);
```

### AggregatingMergeTree (Pre-aggregation)

```sql
-- For maintaining aggregated metrics
CREATE TABLE market_stats_hourly (
    hour DateTime,
    market_id String,
    total_volume AggregateFunction(sum, UInt64),
    total_trades AggregateFunction(count, UInt32),
    unique_users AggregateFunction(uniq, String)
) ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (hour, market_id);

-- Query aggregated data
SELECT
    hour,
    market_id,
    sumMerge(total_volume) AS volume,
    countMerge(total_trades) AS trades,
    uniqMerge(unique_users) AS users
FROM market_stats_hourly
WHERE hour >= toStartOfHour(now() - INTERVAL 24 HOUR)
GROUP BY hour, market_id
ORDER BY hour DESC;
```

## Query Optimization Patterns

### Efficient Filtering

```sql
-- GOOD: Use indexed columns first
SELECT *
FROM markets_analytics
WHERE date >= '2025-01-01'
  AND market_id = 'market-123'
  AND volume > 1000
ORDER BY date DESC
LIMIT 100;

-- BAD: Filter on non-indexed columns first
SELECT *
FROM markets_analytics
WHERE volume > 1000
  AND market_name LIKE '%election%'
  AND date >= '2025-01-01';
```

### Aggregations

```sql
-- GOOD: Use ClickHouse-specific aggregation functions
SELECT
    toStartOfDay(created_at) AS day,
    market_id,
    sum(volume) AS total_volume,
    count() AS total_trades,
    uniq(trader_id) AS unique_traders,
    avg(trade_size) AS avg_size
FROM trades
WHERE created_at >= today() - INTERVAL 7 DAY
GROUP BY day, market_id
ORDER BY day DESC, total_volume DESC;

-- Use quantile for percentiles (more efficient than percentile)
SELECT
    quantile(0.50)(trade_size) AS median,
    quantile(0.95)(trade_size) AS p95,
    quantile(0.99)(trade_size) AS p99
FROM trades
WHERE created_at >= now() - INTERVAL 1 HOUR;
```

### Window Functions

```sql
-- Calculate running totals
SELECT
    date,
    market_id,
    volume,
    sum(volume) OVER (
        PARTITION BY market_id
        ORDER BY date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_volume
FROM markets_analytics
WHERE date >= today() - INTERVAL 30 DAY
ORDER BY market_id, date;
```

## Kotlin/Spring Boot Integration

### Dependencies (build.gradle.kts)

```kotlin
dependencies {
    // ClickHouse JDBC Driver
    implementation("com.clickhouse:clickhouse-jdbc:0.6.0")
    implementation("com.clickhouse:clickhouse-http-client:0.6.0")

    // Connection pooling
    implementation("com.zaxxer:HikariCP:5.1.0")

    // Spring JDBC
    implementation("org.springframework.boot:spring-boot-starter-jdbc")
}
```

### Configuration

```kotlin
@Configuration
class ClickHouseConfig {

    @Value("\${clickhouse.url}")
    private lateinit var url: String

    @Value("\${clickhouse.user}")
    private lateinit var user: String

    @Value("\${clickhouse.password}")
    private lateinit var password: String

    @Bean
    fun clickHouseDataSource(): DataSource {
        val config = HikariConfig().apply {
            jdbcUrl = url
            username = user
            password = password
            driverClassName = "com.clickhouse.jdbc.ClickHouseDriver"
            maximumPoolSize = 10
            minimumIdle = 2
            connectionTimeout = 30000
            idleTimeout = 600000
            maxLifetime = 1800000
            addDataSourceProperty("socket_timeout", "300000")
            addDataSourceProperty("compress", "true")
        }
        return HikariDataSource(config)
    }

    @Bean
    fun clickHouseJdbcTemplate(
        @Qualifier("clickHouseDataSource") dataSource: DataSource
    ): JdbcTemplate {
        return JdbcTemplate(dataSource)
    }
}
```

### application.yml

```yaml
clickhouse:
  url: jdbc:clickhouse://localhost:8123/default
  user: ${CLICKHOUSE_USER:default}
  password: ${CLICKHOUSE_PASSWORD:}
```

### Data Classes

```kotlin
data class Trade(
    val id: String,
    val marketId: String,
    val userId: String,
    val amount: BigDecimal,
    val timestamp: LocalDateTime
)

data class MarketAnalytics(
    val date: LocalDate,
    val marketId: String,
    val marketName: String,
    val volume: Long,
    val trades: Int,
    val uniqueTraders: Int,
    val avgTradeSize: Double
)

data class MarketStats(
    val hour: LocalDateTime,
    val marketId: String,
    val volume: Long,
    val trades: Int,
    val users: Int
)
```

### Repository Layer

```kotlin
@Repository
class ClickHouseTradeRepository(
    @Qualifier("clickHouseJdbcTemplate")
    private val jdbcTemplate: JdbcTemplate
) {
    private val logger = LoggerFactory.getLogger(javaClass)

    // GOOD: Batch insert (efficient)
    fun bulkInsertTrades(trades: List<Trade>) {
        if (trades.isEmpty()) return

        val sql = """
            INSERT INTO trades (id, market_id, user_id, amount, timestamp)
            VALUES (?, ?, ?, ?, ?)
        """.trimIndent()

        jdbcTemplate.batchUpdate(sql, trades, 1000) { ps, trade ->
            ps.setString(1, trade.id)
            ps.setString(2, trade.marketId)
            ps.setString(3, trade.userId)
            ps.setBigDecimal(4, trade.amount)
            ps.setTimestamp(5, Timestamp.valueOf(trade.timestamp))
        }

        logger.info("Inserted ${trades.size} trades")
    }

    // Query with row mapper
    fun findByMarketId(
        marketId: String,
        startDate: LocalDate,
        endDate: LocalDate,
        limit: Int = 1000
    ): List<Trade> {
        val sql = """
            SELECT id, market_id, user_id, amount, timestamp
            FROM trades
            WHERE market_id = ?
              AND toDate(timestamp) BETWEEN ? AND ?
            ORDER BY timestamp DESC
            LIMIT ?
        """.trimIndent()

        return jdbcTemplate.query(sql, tradeRowMapper, marketId, startDate, endDate, limit)
    }

    private val tradeRowMapper = RowMapper { rs, _ ->
        Trade(
            id = rs.getString("id"),
            marketId = rs.getString("market_id"),
            userId = rs.getString("user_id"),
            amount = rs.getBigDecimal("amount"),
            timestamp = rs.getTimestamp("timestamp").toLocalDateTime()
        )
    }
}
```

### Analytics Service

```kotlin
@Service
class ClickHouseAnalyticsService(
    @Qualifier("clickHouseJdbcTemplate")
    private val jdbcTemplate: JdbcTemplate
) {
    private val logger = LoggerFactory.getLogger(javaClass)

    fun getDailyActiveUsers(days: Int = 30): List<DailyActiveUsers> {
        val sql = """
            SELECT
                toDate(timestamp) AS date,
                uniq(user_id) AS daily_active_users
            FROM events
            WHERE timestamp >= today() - INTERVAL ? DAY
            GROUP BY date
            ORDER BY date
        """.trimIndent()

        return jdbcTemplate.query(sql, { rs, _ ->
            DailyActiveUsers(
                date = rs.getDate("date").toLocalDate(),
                count = rs.getLong("daily_active_users")
            )
        }, days)
    }

    fun getMarketStats(
        marketId: String,
        hours: Int = 24
    ): List<MarketStats> {
        val sql = """
            SELECT
                toStartOfHour(timestamp) AS hour,
                market_id,
                sum(amount) AS volume,
                count() AS trades,
                uniq(user_id) AS users
            FROM trades
            WHERE market_id = ?
              AND timestamp >= now() - INTERVAL ? HOUR
            GROUP BY hour, market_id
            ORDER BY hour DESC
        """.trimIndent()

        return jdbcTemplate.query(sql, { rs, _ ->
            MarketStats(
                hour = rs.getTimestamp("hour").toLocalDateTime(),
                marketId = rs.getString("market_id"),
                volume = rs.getLong("volume"),
                trades = rs.getInt("trades"),
                users = rs.getInt("users")
            )
        }, marketId, hours)
    }

    fun getRetentionAnalysis(cohortDays: Int = 30): List<RetentionCohort> {
        val sql = """
            SELECT
                signup_date,
                countIf(days_since_signup = 0) AS day_0,
                countIf(days_since_signup = 1) AS day_1,
                countIf(days_since_signup = 7) AS day_7,
                countIf(days_since_signup = 30) AS day_30
            FROM (
                SELECT
                    user_id,
                    min(toDate(timestamp)) AS signup_date,
                    toDate(timestamp) AS activity_date,
                    dateDiff('day', signup_date, activity_date) AS days_since_signup
                FROM events
                WHERE timestamp >= today() - INTERVAL ? DAY
                GROUP BY user_id, activity_date
            )
            GROUP BY signup_date
            ORDER BY signup_date DESC
        """.trimIndent()

        return jdbcTemplate.query(sql, { rs, _ ->
            RetentionCohort(
                signupDate = rs.getDate("signup_date").toLocalDate(),
                day0 = rs.getInt("day_0"),
                day1 = rs.getInt("day_1"),
                day7 = rs.getInt("day_7"),
                day30 = rs.getInt("day_30")
            )
        }, cohortDays)
    }

    fun getFunnelAnalysis(date: LocalDate): FunnelMetrics {
        val sql = """
            SELECT
                countIf(step = 'viewed_market') AS viewed,
                countIf(step = 'clicked_trade') AS clicked,
                countIf(step = 'completed_trade') AS completed
            FROM (
                SELECT
                    user_id,
                    session_id,
                    event_type AS step
                FROM events
                WHERE toDate(timestamp) = ?
            )
        """.trimIndent()

        return jdbcTemplate.queryForObject(sql, { rs, _ ->
            val viewed = rs.getLong("viewed")
            val clicked = rs.getLong("clicked")
            val completed = rs.getLong("completed")

            FunnelMetrics(
                viewed = viewed,
                clicked = clicked,
                completed = completed,
                viewToClickRate = if (viewed > 0) clicked.toDouble() / viewed * 100 else 0.0,
                clickToCompletionRate = if (clicked > 0) completed.toDouble() / clicked * 100 else 0.0
            )
        }, date)!!
    }
}

data class DailyActiveUsers(
    val date: LocalDate,
    val count: Long
)

data class RetentionCohort(
    val signupDate: LocalDate,
    val day0: Int,
    val day1: Int,
    val day7: Int,
    val day30: Int
)

data class FunnelMetrics(
    val viewed: Long,
    val clicked: Long,
    val completed: Long,
    val viewToClickRate: Double,
    val clickToCompletionRate: Double
)
```

## Materialized Views

### Real-time Aggregations

```sql
-- Create materialized view for hourly stats
CREATE MATERIALIZED VIEW market_stats_hourly_mv
TO market_stats_hourly
AS SELECT
    toStartOfHour(timestamp) AS hour,
    market_id,
    sumState(amount) AS total_volume,
    countState() AS total_trades,
    uniqState(user_id) AS unique_users
FROM trades
GROUP BY hour, market_id;

-- Query the materialized view
SELECT
    hour,
    market_id,
    sumMerge(total_volume) AS volume,
    countMerge(total_trades) AS trades,
    uniqMerge(unique_users) AS users
FROM market_stats_hourly
WHERE hour >= now() - INTERVAL 24 HOUR
GROUP BY hour, market_id;
```

## Performance Monitoring

### Query Performance

```sql
-- Check slow queries
SELECT
    query_id,
    user,
    query,
    query_duration_ms,
    read_rows,
    read_bytes,
    memory_usage
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query_duration_ms > 1000
  AND event_time >= now() - INTERVAL 1 HOUR
ORDER BY query_duration_ms DESC
LIMIT 10;
```

### Table Statistics

```sql
-- Check table sizes
SELECT
    database,
    table,
    formatReadableSize(sum(bytes)) AS size,
    sum(rows) AS rows,
    max(modification_time) AS latest_modification
FROM system.parts
WHERE active
GROUP BY database, table
ORDER BY sum(bytes) DESC;
```

## Data Pipeline Patterns

### ETL Pattern with Spring Batch

```kotlin
@Configuration
@EnableBatchProcessing
class ClickHouseEtlConfig(
    private val jobBuilderFactory: JobBuilderFactory,
    private val stepBuilderFactory: StepBuilderFactory,
    private val postgresDataSource: DataSource,
    @Qualifier("clickHouseDataSource")
    private val clickHouseDataSource: DataSource
) {
    @Bean
    fun etlJob(): Job {
        return jobBuilderFactory.get("clickHouseEtlJob")
            .start(extractTransformLoadStep())
            .build()
    }

    @Bean
    fun extractTransformLoadStep(): Step {
        return stepBuilderFactory.get("etlStep")
            .chunk<RawTradeData, Trade>(1000)
            .reader(postgresReader())
            .processor(tradeProcessor())
            .writer(clickHouseWriter())
            .build()
    }

    @Bean
    fun postgresReader(): JdbcCursorItemReader<RawTradeData> {
        return JdbcCursorItemReaderBuilder<RawTradeData>()
            .name("postgresReader")
            .dataSource(postgresDataSource)
            .sql("""
                SELECT id, market_slug, total_volume, trade_count, created_at
                FROM trade_summaries
                WHERE created_at > :lastProcessed
            """.trimIndent())
            .rowMapper { rs, _ ->
                RawTradeData(
                    id = rs.getString("id"),
                    marketSlug = rs.getString("market_slug"),
                    totalVolume = rs.getBigDecimal("total_volume"),
                    tradeCount = rs.getInt("trade_count"),
                    createdAt = rs.getTimestamp("created_at").toLocalDateTime()
                )
            }
            .build()
    }

    @Bean
    fun tradeProcessor(): ItemProcessor<RawTradeData, Trade> {
        return ItemProcessor { raw ->
            Trade(
                id = raw.id,
                marketId = raw.marketSlug,
                userId = "aggregated",
                amount = raw.totalVolume,
                timestamp = raw.createdAt
            )
        }
    }

    @Bean
    fun clickHouseWriter(): JdbcBatchItemWriter<Trade> {
        return JdbcBatchItemWriterBuilder<Trade>()
            .dataSource(clickHouseDataSource)
            .sql("""
                INSERT INTO trades (id, market_id, user_id, amount, timestamp)
                VALUES (:id, :marketId, :userId, :amount, :timestamp)
            """.trimIndent())
            .beanMapped()
            .build()
    }
}

data class RawTradeData(
    val id: String,
    val marketSlug: String,
    val totalVolume: BigDecimal,
    val tradeCount: Int,
    val createdAt: LocalDateTime
)
```

### Scheduled ETL Job

```kotlin
@Component
class ClickHouseEtlScheduler(
    private val jobLauncher: JobLauncher,
    private val etlJob: Job
) {
    private val logger = LoggerFactory.getLogger(javaClass)

    @Scheduled(cron = "0 0 * * * *")  // Every hour
    fun runEtl() {
        logger.info("Starting ClickHouse ETL job")

        val params = JobParametersBuilder()
            .addLong("timestamp", System.currentTimeMillis())
            .toJobParameters()

        val execution = jobLauncher.run(etlJob, params)

        logger.info("ETL job completed with status: ${execution.status}")
    }
}
```

### Change Data Capture (CDC) with Debezium

```kotlin
@Component
class ClickHouseCdcListener(
    private val clickHouseRepository: ClickHouseTradeRepository
) {
    private val logger = LoggerFactory.getLogger(javaClass)

    @KafkaListener(topics = ["postgres.public.trades"])
    fun handleTradeChange(record: ConsumerRecord<String, String>) {
        val payload = objectMapper.readTree(record.value())
        val operation = payload.get("op").asText()
        val after = payload.get("after")

        when (operation) {
            "c", "u" -> {  // Create or Update
                val trade = Trade(
                    id = after.get("id").asText(),
                    marketId = after.get("market_id").asText(),
                    userId = after.get("user_id").asText(),
                    amount = BigDecimal(after.get("amount").asText()),
                    timestamp = LocalDateTime.parse(after.get("timestamp").asText())
                )
                clickHouseRepository.bulkInsertTrades(listOf(trade))
                logger.info("Synced trade ${trade.id} to ClickHouse")
            }
            "d" -> {  // Delete
                logger.info("Delete operation ignored for analytics")
            }
        }
    }

    companion object {
        private val objectMapper = ObjectMapper()
    }
}
```

## Best Practices

### 1. Partitioning Strategy
- Partition by time (usually month or day)
- Avoid too many partitions (performance impact)
- Use DATE type for partition key

### 2. Ordering Key
- Put most frequently filtered columns first
- Consider cardinality (high cardinality first)
- Order impacts compression

### 3. Data Types
- Use smallest appropriate type (UInt32 vs UInt64)
- Use LowCardinality for repeated strings
- Use Enum for categorical data

### 4. Avoid
- SELECT * (specify columns)
- FINAL (merge data before query instead)
- Too many JOINs (denormalize for analytics)
- Small frequent inserts (batch instead)

### 5. Monitoring
- Track query performance
- Monitor disk usage
- Check merge operations
- Review slow query log

## Testing ClickHouse Integration

```kotlin
@SpringBootTest
@Testcontainers
class ClickHouseRepositoryTest {

    companion object {
        @Container
        val clickhouse = ClickHouseContainer("clickhouse/clickhouse-server:23.8")
            .withDatabaseName("test")

        @JvmStatic
        @DynamicPropertySource
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("clickhouse.url") { clickhouse.jdbcUrl }
            registry.add("clickhouse.user") { clickhouse.username }
            registry.add("clickhouse.password") { clickhouse.password }
        }
    }

    @Autowired
    lateinit var repository: ClickHouseTradeRepository

    @Autowired
    @Qualifier("clickHouseJdbcTemplate")
    lateinit var jdbcTemplate: JdbcTemplate

    @BeforeEach
    fun setup() {
        jdbcTemplate.execute("""
            CREATE TABLE IF NOT EXISTS trades (
                id String,
                market_id String,
                user_id String,
                amount Decimal(18, 8),
                timestamp DateTime
            ) ENGINE = MergeTree()
            ORDER BY (market_id, timestamp)
        """.trimIndent())
    }

    @AfterEach
    fun cleanup() {
        jdbcTemplate.execute("TRUNCATE TABLE trades")
    }

    @Test
    fun `should bulk insert trades`() {
        // Given
        val trades = (1..100).map { i ->
            Trade(
                id = "trade-$i",
                marketId = "market-1",
                userId = "user-${i % 10}",
                amount = BigDecimal("${i * 10}.00"),
                timestamp = LocalDateTime.now().minusHours(i.toLong())
            )
        }

        // When
        repository.bulkInsertTrades(trades)

        // Then
        val count = jdbcTemplate.queryForObject(
            "SELECT count() FROM trades",
            Int::class.java
        )
        assertThat(count).isEqualTo(100)
    }

    @Test
    fun `should find trades by market id`() {
        // Given
        val trades = listOf(
            Trade("t1", "market-1", "user-1", BigDecimal("100"), LocalDateTime.now()),
            Trade("t2", "market-2", "user-2", BigDecimal("200"), LocalDateTime.now())
        )
        repository.bulkInsertTrades(trades)

        // When
        val results = repository.findByMarketId(
            "market-1",
            LocalDate.now().minusDays(1),
            LocalDate.now().plusDays(1)
        )

        // Then
        assertThat(results).hasSize(1)
        assertThat(results[0].marketId).isEqualTo("market-1")
    }
}
```

**Remember**: ClickHouse excels at analytical workloads. Design tables for your query patterns, batch inserts, and leverage materialized views for real-time aggregations.
