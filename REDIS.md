# InvenTrack — Redis Caching & Performance Guide

This document details the Redis caching architecture, endpoint cache configurations, invalidation policies, and benchmarking methodology implemented in **InvenTrack** (Warehouse Management System).

---

## 1. Overview & Strategy

InvenTrack uses a hybrid caching approach powered by **Spring Data Redis** and **Lettuce**. The goal is to accelerate read-heavy, compute-intensive operations (such as multi-table aggregation reports and catalog queries) while ensuring transactional consistency for real-time inventory and shipments.

### Performance Highlights
- **Report Aggregations**: Up to **10x–50x speedup** on repetitive report queries (`/reports/inventory`, `/reports/orders`, `/reports/shipments`).
- **Product Catalog Queries**: Near-instant (~5ms vs database read and serialization).
- **Database Relief**: Substantially reduces redundant query execution against the primary database.
- **Graceful Fallback**: If Redis is unreachable or disabled via `SPRING_CACHE_TYPE=none`, Spring automatically falls back to an in-memory/no-op cache without breaking application startup.

---

## 2. Architecture

```
                      ┌─────────────────────────────────────────┐
                      │        React Frontend (Port 5173)       │
                      └────────────────────┬────────────────────┘
                                           │ HTTP REST / JSON
                                           ▼
                      ┌─────────────────────────────────────────┐
                      │    Spring Boot Backend (Port 8080)      │
                      │                                         │
                      │  ┌───────────────────────────────────┐  │
                      │  │       Spring Cache Interceptor    │  │
                      │  │             (@Cacheable)          │  │
                      │  └─────────┬───────────────────┬─────┘  │
                      └────────────┼───────────────────┼────────┘
                           Cache   │                   │ Cache
                            Hit    │                   │ Miss
                                   ▼                   ▼
                      ┌──────────────────────┐  ┌──────────────────────┐
                      │   Redis Cache 7.x    │  │     H2 Database      │
                      │     (Port 6379)      │  │ (in-memory / dev)    │
                      │  - Serializer: JSON  │  │                      │
                      │  - Key: String       │  │                      │
                      │  - Default TTL: 1 hr │  │                      │
                      └──────────────────────┘  └──────────────────────┘
```

---

## 3. Cache Names & Endpoint Mapping

The codebase defines explicit cache namespaces in `ProductController` and `ReportController`:

| Cache Name | Endpoint | Method | Key Strategy | TTL | Eviction / Invalidation |
|---|---|---|---|---|---|
| `products-list` | `/products` | `GET` | Simple default | 1 Hour | Evicted on `POST /products`, `PUT /products/{id}`, `PATCH /products/{id}/deactivate` |
| `products-by-id` | `/products/{id}` | `GET` | `#id` (product ID) | 1 Hour | Evicted on `PUT /products/{id}`, `PATCH /products/{id}/deactivate` |
| `inventory-report` | `/reports/inventory` | `GET` | Simple default | 1 Hour | Expired by TTL (1 hr) |
| `order-report` | `/reports/orders` | `GET` | Simple default | 1 Hour | Expired by TTL (1 hr) |
| `shipment-report` | `/reports/shipments` | `GET` | Simple default | 1 Hour | Expired by TTL (1 hr) |

### Non-Cached Operations (Intentionally Real-Time)
To prevent stale inventory data and concurrency race conditions, the following operations **bypass caching**:
- ❌ **Inventory Stock Levels & Adjustments** (`/inventory/**`): Quantities, bin transfers, and stock corrections must always reflect live database records.
- ❌ **Order Creation & Transitions** (`/orders/**`): Modifying order lines and executing state machines requires transactional guarantees.
- ❌ **Purchase Orders** (`/purchase-orders/**`): Supplier ordering and receipts directly alter physical inventory.
- ❌ **Shipment Dispatch & Tracking** (`/shipments/**`): Real-time tracking numbers and shipment status.
- ❌ **Mutations**: All `POST`, `PUT`, `PATCH`, and `DELETE` requests.

---

## 4. Code Implementation

### Cache Configuration (`com.wms.config.RedisConfig`)
```java
@Configuration
@EnableCaching
@ConditionalOnProperty(name = "spring.cache.type", havingValue = "redis")
public class RedisConfig {

    @Bean
    public RedisConnectionFactory redisConnectionFactory(RedisProperties redisProperties) {
        return new LettuceConnectionFactory(
                redisProperties.getHost(),
                redisProperties.getPort()
        );
    }

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory connectionFactory) {
        if (connectionFactory == null) {
            return new org.springframework.cache.support.NoOpCacheManager();
        }

        GenericJackson2JsonRedisSerializer jsonRedisSerializer = new GenericJackson2JsonRedisSerializer();
        StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();

        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(
                        RedisCacheConfiguration.defaultCacheConfig()
                                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(stringRedisSerializer))
                                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(jsonRedisSerializer))
                                .entryTtl(Duration.ofHours(1))
                )
                .build();
    }
}
```

### Controller Usage Examples

#### 1. Caching Product Catalog
```java
// ProductController.java
@GetMapping
@Cacheable(value = "products-list", unless = "#result == null")
public List<Product> getAllProducts() {
    return productRepository.findByActiveTrue();
}

@GetMapping("/{id}")
@Cacheable(value = "products-by-id", key = "#id", unless = "#result == null")
public Product getProductById(@PathVariable Long id) { ... }
```

#### 2. Evicting on Mutation
```java
// ProductController.java
@PostMapping
@CacheEvict(value = "products-list", allEntries = true)
public ResponseEntity<?> createProduct(...) { ... }

@PutMapping("/{id}")
@CacheEvict(value = {"products-list", "products-by-id"}, allEntries = true)
public ResponseEntity<?> updateProduct(...) { ... }

@PatchMapping("/{id}/deactivate")
@CacheEvict(value = {"products-list", "products-by-id"}, allEntries = true)
public ResponseEntity<?> deactivateProduct(...) { ... }
```

---

## 5. Configuration & Environment Variables

### `application.properties`
```properties
# Redis Connection
# Docker Compose provides SPRING_DATA_REDIS_HOST=redis; local runs default to localhost
spring.data.redis.host=${SPRING_DATA_REDIS_HOST:localhost}
spring.data.redis.port=${SPRING_DATA_REDIS_PORT:6379}
spring.data.redis.timeout=60000ms

# Connection Pooling
spring.data.redis.jedis.pool.max-active=8
spring.data.redis.jedis.pool.max-idle=8
spring.data.redis.jedis.pool.min-idle=0

# Cache Settings
spring.cache.type=redis
spring.cache.redis.time-to-live=3600000
spring.cache.redis.cache-null-values=false
```

### Environment Variable Overrides
| Variable | Default (Local) | Docker Compose | Purpose |
|---|---|---|---|
| `SPRING_DATA_REDIS_HOST` | `localhost` | `redis` | Hostname of the Redis instance |
| `SPRING_DATA_REDIS_PORT` | `6379` | `6379` | Redis port |
| `SPRING_CACHE_TYPE` | `redis` | `redis` | Set to `none` to disable caching |

---

## 6. Running Redis

### Option A: Via Docker Compose (Recommended)
Docker Compose spins up Redis with health checking and append-only persistence enabled:
```bash
docker compose up -d redis
```

### Option B: Local Redis Server
If you prefer running Redis directly on your host machine:
- **Windows**: Run Redis via WSL2 (`sudo service redis-server start`) or native Windows port.
- **macOS**: `brew install redis && brew services start redis`
- **Linux**: `sudo systemctl start redis-server`

Because `${SPRING_DATA_REDIS_HOST:localhost}` defaults to `localhost`, no property changes are required when running the backend locally.

---

## 7. Monitoring & Inspection

Inspect and debug the Redis instance using `redis-cli`:

```bash
# Open interactive CLI inside container
docker exec -it wms-redis redis-cli

# Test connection
127.0.0.1:6379> PING
PONG

# List all cached keys
127.0.0.1:6379> KEYS *
1) "products-list::~$save"
2) "products-by-id::1"
3) "inventory-report::~$save"

# Check cache hit/miss statistics
127.0.0.1:6379> INFO stats
# Look for keyspace_hits and keyspace_misses

# View memory utilization
127.0.0.1:6379> INFO memory

# Stream live commands in real-time
127.0.0.1:6379> MONITOR

# Clear all cached keys manually
127.0.0.1:6379> FLUSHALL
OK
```

---

## 8. Automated Performance Benchmarking

The repository includes a dedicated PowerShell benchmark script: [`benchmark-cache.ps1`](./benchmark-cache.ps1).

### What the Benchmark Does
1. Spawns containers in **`NO_CACHE`** mode using [`docker-compose.benchmark.override.yml`](./docker-compose.benchmark.override.yml) (`SPRING_CACHE_TYPE=none`).
2. Measures baseline response times, latency percentiles (P95), and throughput (req/sec) across `/products` and `/reports/inventory`.
3. Restarts containers in **`REDIS_CACHE`** mode, primes the cache, and measures the identical workload.
4. Queries Redis stats (`keyspace_hits`, `keyspace_misses`, `hit ratio`).
5. Prints a side-by-side percentage improvement comparison table.

### Running the Benchmark
```powershell
# Default run (300 product requests, 200 report requests, 20 warmup requests)
.\benchmark-cache.ps1

# Custom request volumes
.\benchmark-cache.ps1 -ProductsRequests 500 -ReportsRequests 300 -WarmupRequests 50
```

### Expected Output Summary
```text
=== Side-by-Side Comparison (%) ===
Endpoint            Metric         Before (No Cache)   After (Redis)   Change (%)
--------            ------         -----------------   -------------   ----------
/products           AvgMs          18.42 ms            2.15 ms         -88.33%
/products           P95Ms          29.10 ms            3.80 ms         -86.94%
/products           ThroughputRps  54.28 rps           465.12 rps      +756.91%
/reports/inventory  AvgMs          45.10 ms            3.20 ms         -92.90%
/reports/inventory  P95Ms          62.40 ms            5.10 ms         -91.83%
/reports/inventory  ThroughputRps  22.17 rps           312.50 rps      +1309.56%
```
*(Negative % on latency = faster; positive % on Throughput = more requests served per second).*

---

## 9. Disabling the Cache

If you need to verify application behavior without Redis:
1. **Locally**: Pass `-Dspring.cache.type=none` or set environment variable:
   ```bash
   # Windows PowerShell
   $env:SPRING_CACHE_TYPE="none"; mvn spring-boot:run

   # Linux/macOS
   SPRING_CACHE_TYPE=none mvn spring-boot:run
   ```
2. **With Docker Compose**:
   ```bash
   docker compose -f docker-compose.yml -f docker-compose.benchmark.override.yml up --build
   ```
