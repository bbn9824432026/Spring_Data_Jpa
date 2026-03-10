# 🏛️ TOPIC 18 — Caching with Spring Data JPA

---

## 1️⃣ Conceptual Explanation

### The Two Cache Layers — Architecture Overview

JPA and Spring operate with fundamentally different cache abstractions that coexist but serve different purposes:

```
Request Thread
      │
      ▼
┌─────────────────────────────────────────────────────────┐
│  FIRST-LEVEL CACHE (L1)                                 │
│  = Persistence Context = EntityManager's identity map   │
│  Scope: single transaction / single EntityManager       │
│  Storage: JVM heap, ThreadLocal                         │
│  Managed by: Hibernate (automatic, cannot disable)      │
│  Eviction: TX commit/rollback, em.clear(), em.detach()  │
└──────────────────────┬──────────────────────────────────┘
                       │ MISS (entity not in PC)
                       ▼
┌─────────────────────────────────────────────────────────┐
│  SECOND-LEVEL CACHE (L2C)                               │
│  Scope: SessionFactory / application-wide               │
│  Storage: external (EhCache, Caffeine, Redis, Infinispan)│
│  Managed by: Hibernate + cache provider                 │
│  Shared across: all transactions, all threads           │
│  Opt-in: must configure + annotate each entity          │
└──────────────────────┬──────────────────────────────────┘
                       │ MISS (entity not in L2C)
                       ▼
┌─────────────────────────────────────────────────────────┐
│  DATABASE                                               │
│  SQL SELECT executed                                    │
│  Result hydrated into entity                            │
│  Entity stored in L1 (always) and L2C (if configured)  │
└─────────────────────────────────────────────────────────┘
```

---

### First-Level Cache (L1) — Persistence Context as Identity Map

The L1 cache is not optional — it is the Persistence Context itself. Every `EntityManager` instance maintains an **identity map**: a `Map<EntityKey, Object>` where `EntityKey = (EntityClass, PrimaryKey)`.

```
Identity map behavior:

em.find(User.class, 1L)   → cache miss  → SELECT FROM users WHERE id=1
                          → User{id=1, name="Alice"} stored in map

em.find(User.class, 1L)   → cache HIT   → returns same Java object
em.find(User.class, 1L)   → cache HIT   → same Java object reference

em.find(User.class, 2L)   → cache miss  → SELECT FROM users WHERE id=2

// All three calls within same transaction return identical references:
User a = em.find(User.class, 1L);
User b = em.find(User.class, 1L);
User c = em.find(User.class, 1L);
assert a == b; // true — same object
assert b == c; // true — same object
// Only ONE SQL SELECT executed
```

**L1 scope and lifespan:**

```
@Transactional method:
  TX start → EM created → L1 cache empty
  em.find() calls → populate L1
  em.flush() → SQL INSERTs/UPDATEs for pending changes
  TX commit → EM closed → L1 cache DESTROYED

Between transactions:
  Entities are DETACHED — not in any L1 cache
  Must be re-fetched in next transaction

spring.jpa.open-in-view=true:
  EM lives for entire HTTP request (not just TX duration)
  L1 cache accumulates across multiple TX within same request
  → Memory risk for large result sets in view layer
```

**L1 cache and JPQL queries:**

```java
// JPQL queries DO NOT check L1 cache first:
User user = em.find(User.class, 1L);  // cached in L1

// This still fires SQL even though id=1 is in L1:
User sameUser = em.createQuery(
    "SELECT u FROM User u WHERE u.id = 1", User.class)
    .getSingleResult();
// SQL: SELECT u.* FROM users WHERE u.id = 1
// BUT: result is the SAME Java object as 'user' (identity map merges)
// The SQL still executes, but the returned object is from identity map

// em.find() IS L1-aware:
// Second em.find(User.class, 1L) → NO SQL → returns cached instance
```

---

### Second-Level Cache (L2C) — Cross-Transaction Entity Cache

The L2C caches entity state across transactions, sessions, and threads. It stores a **disassembled** (dehydrated) representation of entities — not the Java objects themselves.

```
L2C stores (per entity type, per primary key):
  Key:   EntityClass + PrimaryKey (e.g., User + 42L)
  Value: CacheEntry {
    id:           42L
    firstName:    "Alice"          ← raw field values (not entity object)
    lastName:     "Smith"
    email:        "alice@example.com"
    version:      3
    dirtyCount:   0
    entityClass:  User.class
  }

NOT stored in L2C:
  - The Java entity object itself (no object sharing across threads)
  - Associations (unless separately cached)
  - JPQL query results (separate Query Cache)

When em.find(User.class, 42L) is called in a NEW transaction:
  1. Check L1 (empty — new TX) → MISS
  2. Check L2C → HIT: CacheEntry found
  3. Assemble new User entity from CacheEntry field values
  4. Put assembled entity into L1 (current PC)
  5. Return entity — NO SQL executed
```

---

### Enabling L2C — Configuration

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true      # enable L2C
          use_query_cache: false             # query cache (separate — see below)
          region:
            factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
            # or: org.hibernate.cache.caffeine.CaffeineRegionFactory
            # or: org.hibernate.cache.ehcache.EhCacheRegionFactory
```

```xml
<!-- Maven dependencies for Caffeine (JCache provider): -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>jcache</artifactId>
</dependency>
```

---

### `@Cache` — Marking Entities for L2C

```java
import org.hibernate.annotations.Cache;
import org.hibernate.annotations.CacheConcurrencyStrategy;

@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    @Id @GeneratedValue Long id;
    String name;
    BigDecimal price;
    String category;
}
```

**Without `@Cache`, entities are NEVER stored in L2C** — even if L2C is enabled globally. `@Cache` is an opt-in per entity.

---

### Cache Concurrency Strategies — The Core Decision

The concurrency strategy defines how Hibernate coordinates cache reads and writes across concurrent transactions:

---

#### `READ_ONLY` — Immutable Entities

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
public class Country {
    @Id Long id;
    String name;
    String isoCode;
}

// Rules:
// → Entity is NEVER updated after initial insert
// → No version checking needed
// → Cache entry never invalidated (unless explicitly evicted)
// → Fastest strategy — zero locking overhead
// → Hibernate throws exception if you try to UPDATE a READ_ONLY entity

// Use for: reference data (countries, currencies, categories, status codes)
// Never use for: entities that change (orders, users, inventory)
```

---

#### `NONSTRICT_READ_WRITE` — Soft Invalidation

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE)
public class ProductCategory {
    @Id Long id;
    String name;
    String description;
}

// Rules:
// → On UPDATE: cache entry is INVALIDATED (removed), not updated
// → Next read: cache MISS → SQL → cache re-populated
// → No locking — concurrent readers may briefly see stale data
// → "Best effort" freshness

// The staleness window:
// T1: TX A reads category, gets from cache (version 3)
// T2: TX B updates category, invalidates cache
// T3: TX A reads category again WITHIN SAME TX: L1 hit (version 3) — still OK
// T4: NEW TX C reads: cache miss → fresh SQL → version 4 — fresh

// Use for: entities updated infrequently, brief staleness acceptable
// E.g.: product categories, configuration settings
```

---

#### `READ_WRITE` — Strict Freshness with Soft Locks

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Inventory {
    @Id Long id;
    Long productId;
    int quantity;
    @Version Integer version;
}

// Mechanism:
// → Before UPDATE: cache entry replaced with SOFT LOCK marker
// → Concurrent reads during update: SOFT LOCK detected → go to DB
// → After UPDATE commit: soft lock released, new value written to cache

// Soft lock sequence:
// T1: TX A begins updating inventory id=5
// T2: Hibernate places soft lock in L2C for key (Inventory, 5)
// T3: TX B tries to read inventory id=5
//     → L2C has soft lock marker → MISS (treat as miss) → SQL executed
// T4: TX A commits, soft lock released, new value in L2C
// T5: TX C reads inventory id=5 → L2C HIT → fresh value

// Use for: frequently read, occasionally updated entities
// Requires: @Version (for optimistic locking coordination)
// Thread-safe: YES
// Performance: moderate overhead (soft lock creation/release)
```

---

#### `TRANSACTIONAL` — Full JTA Transaction Coordination

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.TRANSACTIONAL)
public class Account {
    @Id Long id;
    BigDecimal balance;
    @Version Integer version;
}

// Rules:
// → Cache updates are part of the JTA transaction
// → Cache and DB are updated atomically (2-phase commit)
// → Requires: JTA transaction manager (JBoss, WildFly, Atomikos)
// → NOT supported in standard Spring Boot (uses JpaTransactionManager, not JTA)
// → Highest consistency guarantee

// Use for: financial data in JTA environments
// NOT suitable for: Spring Boot with simple JPA transactions
```

---

### Strategy Selection Matrix

```
┌──────────────────────┬───────────┬───────────┬───────────┬──────────────────────┐
│ Strategy             │ Writes?   │ Stale?    │ JTA req?  │ Use Case             │
├──────────────────────┼───────────┼───────────┼───────────┼──────────────────────┤
│ READ_ONLY            │ Never     │ Never     │ No        │ Static reference data │
│ NONSTRICT_READ_WRITE │ Rarely    │ Briefly   │ No        │ Infrequent updates   │
│ READ_WRITE           │ Sometimes │ Never*    │ No        │ Regular updates      │
│ TRANSACTIONAL        │ Frequent  │ Never     │ YES       │ JTA, financial       │
└──────────────────────┴───────────┴───────────┴───────────┴──────────────────────┘
*READ_WRITE uses soft locks to prevent stale reads
```

---

### Collection Caching — `@Cache` on Collections

Collections (`@OneToMany`, `@ManyToMany`) require a **separate** `@Cache` annotation on the collection field:

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    @Id @GeneratedValue Long id;
    String name;

    @OneToMany(mappedBy = "product")
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)  // ← separate annotation
    List<Review> reviews;
    // If @Cache omitted on reviews: collection always fetched from DB
    // Entity cache and collection cache are SEPARATE
}

// L2C stores collections separately from entity:
// Entity cache region:     "com.example.Product"
//   Key: (Product, 1L) → { id:1, name:"Widget", ... }
//
// Collection cache region: "com.example.Product.reviews"
//   Key: (Product.reviews, 1L) → [reviewId1, reviewId2, reviewId3]
//   (stores IDs only — individual Review entities from Review's own cache)
```

---

### Query Cache — Caching JPQL Results

The Query Cache caches JPQL/SQL query **result sets** (lists of primary keys), not the entities themselves. Entities are still looked up in entity cache or DB.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true  # must enable separately
```

```java
// Enabling query cache per query:
@QueryHints({
    @QueryHint(name = "org.hibernate.cacheable", value = "true"),
    @QueryHint(name = "org.hibernate.cacheRegion", value = "country-queries")
})
@Query("SELECT c FROM Country c WHERE c.region = :region ORDER BY c.name")
List<Country> findByRegion(@Param("region") String region);

// Spring Data Specification:
// Query results cached with key: (query string + parameters)

// How Query Cache works:
// First call findByRegion("EMEA"):
//   1. Query cache miss → SQL executed → [1, 3, 7, 12] (IDs)
//   2. IDs stored in query cache
//   3. Each ID looked up in entity cache (or DB if miss)
//   4. Entity list returned

// Second call findByRegion("EMEA"):
//   1. Query cache HIT → [1, 3, 7, 12] retrieved
//   2. Each ID looked up in entity cache → all hits
//   3. No SQL executed

// CRITICAL: Query cache is invalidated when ANY entity of the queried type changes
// Even if the specific query result is unaffected
// High-write tables + query cache = constant invalidation = performance regression
```

**Query Cache invalidation — the table-level trap:**

```
Query cache region stores results keyed by query + params.
Hibernate tracks which entity tables are referenced in each cached query.
When ANY row in that table is INSERT/UPDATE/DELETE:
  → ALL query cache entries referencing that table are invalidated.

Example:
  Cached queries for "Country": ["EMEA" → [1,3,7], "APAC" → [2,5,9]]
  Any country is updated:
  → BOTH cache entries invalidated (even if EMEA query result unchanged)
  
Implication: Query cache only beneficial for:
  ✓ Read-only or very infrequently modified tables
  ✓ Tables with high read:write ratio (100:1 or more)
  ✗ Never for high-write tables — constant invalidation negates benefit
```

---

### Cache Regions — Naming and Configuration

Each entity type and collection gets its own **cache region** — an independent namespace in the cache:

```
Default region names:
  Entity:     fully qualified class name  →  "com.example.Product"
  Collection: class + field name          →  "com.example.Product.reviews"
  Query:      configured per query        →  "default-query-results-region"

Custom region name:
  @Cache(usage = READ_WRITE, region = "products")
```

**Per-region Caffeine configuration:**

```java
@Bean
public CacheManager cacheManager() {
    CaffeineCacheManager manager = new CaffeineCacheManager();

    // Default spec for all regions:
    manager.setCaffeineSpec(CaffeineSpec.parse(
        "maximumSize=500,expireAfterWrite=10m"));

    // Region-specific configs:
    manager.registerCustomCache("products",
        Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(30, TimeUnit.MINUTES)
            .recordStats()
            .build());

    manager.registerCustomCache("country-queries",
        Caffeine.newBuilder()
            .maximumSize(100)
            .expireAfterWrite(24, TimeUnit.HOURS)  // rarely changes
            .build());

    return manager;
}
```

---

### Spring `@Cacheable` vs Hibernate L2C — Key Distinction

These are two separate caching systems that can coexist:

```
Hibernate L2C (@Cache):
  Level: ORM entity / query result
  Granularity: per entity type, per primary key
  Invalidation: Hibernate manages automatically (on flush/commit)
  What's cached: Disassembled entity state (field values)
  Scope: Within JPA/Hibernate layer (before entities reach service layer)
  Warm-up: Populated on first em.find() call per entity key

Spring @Cacheable:
  Level: Spring bean method return value
  Granularity: per method, per argument combination
  Invalidation: manual (@CacheEvict) or TTL-based
  What's cached: Full return value (entity, DTO, List, any object)
  Scope: Spring application layer (after entities reach service layer)
  Warm-up: Populated on first method invocation with those arguments
```

**Typical layered caching:**

```java
@Service
public class ProductService {

    // Spring @Cacheable caches the METHOD RETURN VALUE:
    @Cacheable(value = "products", key = "#id")
    @Transactional(readOnly = true)
    public ProductDto findById(Long id) {
        // If Spring cache HIT: method body never executes
        // If Spring cache MISS: method executes:
        //   → em.find(Product, id) called
        //   → Hibernate L2C checked (if configured)
        //   → If L2C HIT: no SQL, entity from L2C
        //   → If L2C MISS: SQL, entity cached in L2C
        //   → ProductDto created from entity
        //   → ProductDto stored in Spring cache
        Product product = productRepo.findById(id)
            .orElseThrow(ProductNotFoundException::new);
        return ProductDto.from(product);
    }

    @CacheEvict(value = "products", key = "#product.id")
    @Transactional
    public Product updateProduct(Product product) {
        return productRepo.save(product);
        // Spring cache evicted for this product ID
        // Hibernate L2C also updated (by Hibernate automatically)
    }
}
```

**When to use each:**

```
Hibernate L2C (@Cache):
  ✓ Entity-level caching transparent to application code
  ✓ Automatically invalidated by Hibernate on write
  ✓ Cross-service, cross-repository caching of same entity
  ✓ Benefits all em.find() calls without code changes
  ✗ Only works with em.find() — JPQL still goes to DB (unless Query Cache)

Spring @Cacheable:
  ✓ Method-level caching of any return type (DTOs, Lists, computed values)
  ✓ Cached after DTO transformation (avoids re-transformation)
  ✓ Works with any data source (not just JPA)
  ✓ Fine-grained cache key control
  ✗ Manual eviction required (@CacheEvict)
  ✗ Does not prevent DB queries within the method (unless method body skipped)
```

---

### Cache Statistics and Monitoring

```java
@Configuration
public class HibernateStatisticsConfig {

    @Bean
    public HibernateStatisticsFactoryBean hibernateStatistics(
            EntityManagerFactory emf) {
        HibernateStatisticsFactoryBean stats =
            new HibernateStatisticsFactoryBean();
        stats.setEntityManagerFactory(emf);
        return stats;
    }
}

// Access statistics:
@Autowired Statistics statistics;

public void printCacheStats() {
    statistics.setStatisticsEnabled(true);

    // Overall:
    System.out.println("L2C hit count: "   + statistics.getSecondLevelCacheHitCount());
    System.out.println("L2C miss count: "  + statistics.getSecondLevelCacheMissCount());
    System.out.println("L2C put count: "   + statistics.getSecondLevelCachePutCount());

    // Per region:
    SecondLevelCacheStatistics regionStats =
        statistics.getSecondLevelCacheStatistics("com.example.Product");
    System.out.println("Products in cache: " + regionStats.getElementCountInMemory());
    System.out.println("Products hit ratio: " +
        (double) regionStats.getHitCount() /
        (regionStats.getHitCount() + regionStats.getMissCount()));

    // Query cache:
    System.out.println("QC hits: "   + statistics.getQueryCacheHitCount());
    System.out.println("QC misses: " + statistics.getQueryCacheMissCount());
}
```

---

### Programmatic Cache Eviction

```java
@Service
public class CacheManagementService {

    @PersistenceUnit
    EntityManagerFactory emf;

    // Evict specific entity:
    public void evictProduct(Long productId) {
        Cache cache = emf.getCache();
        cache.evict(Product.class, productId);
        // Removes (Product, productId) from L2C
    }

    // Evict all of an entity type:
    public void evictAllProducts() {
        Cache cache = emf.getCache();
        cache.evict(Product.class);
        // Removes ALL Product entries from L2C
    }

    // Evict entire L2C:
    public void evictAll() {
        Cache cache = emf.getCache();
        cache.evictAll();
        // Clears ALL L2C regions — use with caution
    }

    // Check if in cache:
    public boolean isInCache(Long productId) {
        Cache cache = emf.getCache();
        return cache.contains(Product.class, productId);
    }

    // Session-level cache control (for current session):
    public void evictFromSession(EntityManager em, Long productId) {
        Session session = em.unwrap(Session.class);
        session.evict(productRepo.findById(productId).get());
        // Removes from L1 (current PC)
    }
}
```

---

### Cache Anti-Patterns

```
Anti-pattern 1: Caching high-churn entities
  Symptom: L2C hit ratio < 10% (mostly misses)
  Cause: Entities updated frequently → constant invalidation
  Fix: Remove @Cache, let DB serve writes directly

Anti-pattern 2: Query cache on high-write tables
  Symptom: High query cache miss ratio despite caching
  Cause: Any write to table invalidates ALL query cache for that table
  Fix: Disable query cache for high-write entities

Anti-pattern 3: Caching entities with large lazy collections
  Symptom: OutOfMemoryError under load
  Cause: Collection cache loaded into memory repeatedly
  Fix: @NotFound(action=IGNORE) or separate cache region with size limit

Anti-pattern 4: L2C without @Version
  Symptom: Stale reads despite READ_WRITE strategy
  Cause: READ_WRITE uses soft locks + version check for cache coherence
  Fix: Add @Version to all L2C-enabled entities

Anti-pattern 5: Duplicate caching (Hibernate L2C + Spring @Cacheable)
  Symptom: Complex invalidation bugs, stale data in Spring cache
  Cause: Hibernate updates L2C on write, but Spring cache is separate
  Fix: Choose ONE layer. Use Spring @Cacheable for DTO results,
       OR Hibernate L2C for entity caching — not both for the same data
```

---

## 2️⃣ Code Examples

### Example 1 — L2C with Caffeine — Complete Setup

```java
// ── 1. Dependencies (pom.xml) ─────────────────────────────────────────────
// hibernate-jcache, caffeine, jcache already listed above

// ── 2. application.yml ────────────────────────────────────────────────────
/*
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          use_query_cache: true
          region:
            factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        javax:
          cache:
            provider: com.github.benmanes.caffeine.jcache.spi.CaffeineCachingProvider
            missing_cache_strategy: create-warn
*/

// ── 3. Cache configuration ────────────────────────────────────────────────
@Configuration
@EnableCaching  // Enables Spring @Cacheable
public class CacheConfig {

    @Bean
    public javax.cache.CacheManager caffeineCacheManager() {
        CachingProvider provider = Caching.getCachingProvider(
            "com.github.benmanes.caffeine.jcache.spi.CaffeineCachingProvider");
        javax.cache.CacheManager cacheManager = provider.getCacheManager();

        // Configure each Hibernate region:
        configureRegion(cacheManager, "com.example.Product",
            Caffeine.newBuilder()
                .maximumSize(500)
                .expireAfterWrite(30, TimeUnit.MINUTES));

        configureRegion(cacheManager, "com.example.Category",
            Caffeine.newBuilder()
                .maximumSize(100)
                .expireAfterWrite(24, TimeUnit.HOURS));

        configureRegion(cacheManager, "country-queries",
            Caffeine.newBuilder()
                .maximumSize(50)
                .expireAfterWrite(12, TimeUnit.HOURS));

        return cacheManager;
    }

    private void configureRegion(javax.cache.CacheManager mgr,
                                  String name,
                                  Caffeine<?, ?> caffeine) {
        MutableConfiguration<Object, Object> config =
            new MutableConfiguration<>()
                .setStoreByValue(false)
                .setStatisticsEnabled(true);

        CaffeineConfiguration<Object, Object> caffeineConfig =
            new CaffeineConfiguration<>();
        caffeineConfig.setCaffeineSpec(Optional.of(caffeine.toString()));
        mgr.createCache(name, caffeineConfig);
    }
}

// ── 4. Entities with caching ──────────────────────────────────────────────
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE,
       region = "com.example.Product")
public class Product {
    @Id @GeneratedValue Long id;
    String name;
    BigDecimal price;
    boolean active;

    @Version Integer version;  // Required for READ_WRITE

    @ManyToOne
    @Cache(usage = CacheConcurrencyStrategy.READ_ONLY)  // Category rarely changes
    Category category;

    @OneToMany(mappedBy = "product")
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
    List<Review> reviews;
}

@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY,
       region = "com.example.Category")
public class Category {
    @Id Long id;
    String name;
    String description;
}

// ── 5. Repository with query cache ────────────────────────────────────────
public interface CategoryRepository extends JpaRepository<Category, Long> {

    @QueryHints({
        @QueryHint(name = "org.hibernate.cacheable",   value = "true"),
        @QueryHint(name = "org.hibernate.cacheRegion", value = "country-queries")
    })
    @Query("SELECT c FROM Category c WHERE c.active = true ORDER BY c.name")
    List<Category> findAllActive();
    // First call: SQL → cache [1, 2, 3, 7] + entity cache for each
    // Subsequent calls: query cache HIT → entity cache HITs → no SQL
}

// ── 6. Service layer ──────────────────────────────────────────────────────
@Service
@Transactional(readOnly = true)
public class ProductService {

    public Product findById(Long id) {
        return productRepo.findById(id).get();
        // L1 miss (new TX) → L2C check → hit or miss → SQL if miss
        // All transparent to this code
    }

    @Transactional
    public Product update(Product product) {
        Product saved = productRepo.save(product);
        // Hibernate automatically updates L2C after successful commit:
        // 1. Soft lock placed in L2C for this entity
        // 2. UPDATE SQL executed
        // 3. TX commits
        // 4. Soft lock released, new value written to L2C
        return saved;
    }
}
```

---

### Example 2 — Spring `@Cacheable` Layered with JPA

```java
// Spring cache configuration:
@Configuration
@EnableCaching
public class SpringCacheConfig {

    @Bean
    public CacheManager springCacheManager() {
        CaffeineCacheManager manager = new CaffeineCacheManager();
        manager.setCaffeineSpec(CaffeineSpec.parse(
            "maximumSize=200,expireAfterWrite=5m,recordStats"));
        return manager;
    }
}

// Service with Spring @Cacheable:
@Service
public class UserService {

    // Cache the DTO result — not the entity:
    @Cacheable(value = "users", key = "#id",
               condition = "#id > 0",
               unless = "#result == null")
    @Transactional(readOnly = true)
    public UserDto findUserById(Long id) {
        // Spring cache MISS: this executes
        User user = userRepo.findById(id)
            .orElseThrow(UserNotFoundException::new);
        return UserDto.from(user);  // DTO creation happens here
        // DTO stored in Spring cache
        // Next call with same id: method body NEVER executes
    }

    // Evict on update:
    @CacheEvict(value = "users", key = "#id")
    @Transactional
    public UserDto updateUser(Long id, UserUpdateRequest request) {
        User user = userRepo.findById(id)
            .orElseThrow(UserNotFoundException::new);
        user.setFirstName(request.firstName());
        user.setLastName(request.lastName());
        // Spring cache evicted for this id
        // Hibernate L2C also updated automatically
        return UserDto.from(user);
    }

    // Multiple cache eviction:
    @Caching(evict = {
        @CacheEvict(value = "users", key = "#id"),
        @CacheEvict(value = "user-summaries", allEntries = true)
    })
    @Transactional
    public void deleteUser(Long id) {
        userRepo.deleteById(id);
    }

    // Cache a list result:
    @Cacheable(value = "user-summaries",
               key = "#dept + '-' + #active")
    @Transactional(readOnly = true)
    public List<UserSummaryDto> findSummaries(String dept,
                                               boolean active) {
        return userRepo.findByDepartmentAndActive(dept, active)
            .stream()
            .map(UserSummaryDto::from)
            .collect(Collectors.toList());
    }

    // Conditional caching — only cache small result sets:
    @Cacheable(value = "search-results",
               key = "#query",
               unless = "#result.size() > 100")
    @Transactional(readOnly = true)
    public List<UserDto> search(String query) {
        return userRepo.findByNameContaining(query)
            .stream().map(UserDto::from).collect(toList());
    }
}
```

---

### Example 3 — Cache Statistics and Monitoring

```java
@RestController
@RequestMapping("/admin/cache")
public class CacheMonitoringController {

    @PersistenceUnit EntityManagerFactory emf;
    @Autowired CacheManager springCacheManager;

    // Hibernate L2C statistics:
    @GetMapping("/hibernate/stats")
    public Map<String, Object> hibernateStats() {
        SessionFactory sf = emf.unwrap(SessionFactory.class);
        Statistics stats = sf.getStatistics();

        Map<String, Object> result = new LinkedHashMap<>();
        result.put("l2cHits",   stats.getSecondLevelCacheHitCount());
        result.put("l2cMisses", stats.getSecondLevelCacheMissCount());
        result.put("l2cPuts",   stats.getSecondLevelCachePutCount());
        result.put("queryCacheHits",   stats.getQueryCacheHitCount());
        result.put("queryCacheMisses", stats.getQueryCacheMissCount());
        result.put("sqlQueryCount",    stats.getQueryExecutionCount());
        result.put("entityFetchCount", stats.getEntityFetchCount());

        // Per-region breakdown:
        Map<String, Object> regions = new LinkedHashMap<>();
        for (String region : stats.getSecondLevelCacheRegionNames()) {
            SecondLevelCacheStatistics regStats =
                stats.getSecondLevelCacheStatistics(region);
            regions.put(region, Map.of(
                "inMemory", regStats.getElementCountInMemory(),
                "hits",     regStats.getHitCount(),
                "misses",   regStats.getMissCount()
            ));
        }
        result.put("regions", regions);
        return result;
    }

    // Programmatic eviction:
    @DeleteMapping("/hibernate/{entityClass}/{id}")
    public ResponseEntity<String> evictEntity(
            @PathVariable String entityClass,
            @PathVariable Long id) throws ClassNotFoundException {
        Cache cache = emf.getCache();
        Class<?> clazz = Class.forName(entityClass);
        cache.evict(clazz, id);
        return ResponseEntity.ok("Evicted " + entityClass + " id=" + id);
    }

    // Spring cache stats:
    @GetMapping("/spring/stats")
    public Map<String, Object> springStats() {
        Map<String, Object> result = new LinkedHashMap<>();
        springCacheManager.getCacheNames().forEach(name -> {
            Cache cache = springCacheManager.getCache(name);
            if (cache instanceof CaffeineCache cc) {
                com.github.benmanes.caffeine.cache.Cache<?, ?> native_ =
                    cc.getNativeCache();
                CacheStats stats = native_.stats();
                result.put(name, Map.of(
                    "hitRate", stats.hitRate(),
                    "missRate", stats.missRate(),
                    "size", native_.estimatedSize(),
                    "evictionCount", stats.evictionCount()
                ));
            }
        });
        return result;
    }
}
```

---

### Example 4 — Read-Only Reference Data with Query Cache

```java
// Reference data entities (rarely change):
@Entity
@Table(name = "countries")
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
public class Country {
    @Id Long id;
    String name;
    String isoCode;
    String region;
    boolean active;
}

@Entity
@Table(name = "currencies")
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
public class Currency {
    @Id Long id;
    String code;
    String name;
    String symbol;
}

// Repository with query cache:
public interface CountryRepository extends JpaRepository<Country, Long> {

    @QueryHints({
        @QueryHint(name = "org.hibernate.cacheable", value = "true"),
        @QueryHint(name = "org.hibernate.cacheRegion",
                   value = "reference-data-queries")
    })
    List<Country> findByActiveTrue();
    // SQL: SELECT ... FROM countries WHERE active=true
    // Cached indefinitely (READ_ONLY + no writes expected)

    @QueryHints({
        @QueryHint(name = "org.hibernate.cacheable", value = "true"),
        @QueryHint(name = "org.hibernate.cacheRegion",
                   value = "reference-data-queries")
    })
    @Query("SELECT c FROM Country c WHERE c.region = :region AND c.active=true")
    List<Country> findByRegion(@Param("region") String region);
}

// Service pre-loading cache at startup:
@Service
public class ReferenceDataService implements ApplicationRunner {

    @Transactional(readOnly = true)
    @Override
    public void run(ApplicationArguments args) {
        // Load all countries at startup → populates L2C entity + query cache
        List<Country> countries = countryRepo.findByActiveTrue();
        log.info("Pre-loaded {} countries into cache", countries.size());
        // All subsequent calls to findByActiveTrue() → query cache HIT → no SQL
    }

    @Transactional(readOnly = true)
    public List<Country> getCountriesByRegion(String region) {
        return countryRepo.findByRegion(region);
        // Query cache hit if already fetched for this region
        // Entity cache hit for each country ID
        // → Zero SQL for warm cache
    }
}
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ: L1 Cache and JPQL**
```java
@Transactional
public void test() {
    User user1 = em.find(User.class, 1L);   // Line A
    User user2 = userRepo.findById(1L).get(); // Line B (uses em.find internally)

    String jpql = "SELECT u FROM User u WHERE u.id = 1";
    User user3 = em.createQuery(jpql, User.class).getSingleResult(); // Line C

    System.out.println(user1 == user2); // Line D
    System.out.println(user1 == user3); // Line E
}
```

How many SQL SELECT statements are executed?

A) 1 — all three use L1 cache  
B) 2 — em.find() twice hits L1, JPQL always goes to DB  
C) 3 — each call hits the DB independently  
D) 2 — Line A hits DB, Line B hits L1 (same em.find internally), Line C hits DB  

**Answer: D** — Line A: DB hit (cache miss), User loaded and stored in L1. Line B: `findById()` uses `em.find()` internally — L1 HIT, no SQL. Line C: JPQL queries **do not check L1 before executing** — SQL fires. But the result is reconciled with the identity map — `user3` is the **same Java object** as `user1`. Lines D and E both print `true`.

---

**Q2 — Select All That Apply: `READ_WRITE` Strategy**

Which statements about `READ_WRITE` cache strategy are TRUE?

A) It uses soft locks to prevent stale reads during concurrent updates  
B) It requires `@Version` on the entity for correct operation  
C) It commits cache updates as part of the JTA transaction  
D) A concurrent reader encountering a soft lock gets the cached (possibly stale) value  
E) After a successful commit, the new entity value is written to L2C  
F) It is suitable for Spring Boot applications without JTA  

**Answer: A, B, E, F**
- C is false: JTA transaction coordination is `TRANSACTIONAL` strategy, not `READ_WRITE`
- D is false: A concurrent reader encountering a soft lock treats it as a cache MISS and goes to the DB — they do NOT read the stale cached value

---

**Q3 — Query Cache Invalidation**
```java
// productRepo.findAllActive() is query-cached
// Database has 1000 products

productRepo.findAllActive(); // Call 1
productRepo.save(new Product("New Item", ...)); // Line A
productRepo.findAllActive(); // Call 2
```

How many SQL queries execute total?

A) 1 — second `findAllActive()` hits query cache  
B) 2 — `save()` inserts but doesn't affect query cache; second `findAllActive()` hits cache  
C) 3 — save() executes INSERT; second `findAllActive()` has query cache miss (INSERT invalidated cache)  
D) 2 — `findAllActive()` twice = 1 SQL; `save()` = 1 SQL  

**Answer: C** — Query cache is invalidated when ANY write (INSERT, UPDATE, DELETE) occurs to a table referenced by the cached query. `productRepo.save(new Product())` inserts into the `products` table. Hibernate invalidates ALL query cache entries for the `products` table — including `findAllActive()`. Second `findAllActive()` = query cache MISS → SQL. Total: 1 (first findAllActive) + 1 (INSERT) + 1 (second findAllActive) = 3 SQL statements.

---

**Q4 — L2C and Collection Cache**
```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Category {
    @Id Long id;
    String name;

    @OneToMany(mappedBy = "category")
    // NO @Cache on this collection
    List<Product> products;
}
```

When `em.find(Category.class, 1L)` is called in a new transaction, which data is served from L2C?

A) Both Category entity AND its products collection  
B) Only the Category entity; products collection always fetched from DB  
C) Neither — L2C only caches entities with `@Cache` on all fields  
D) Category from L2C; products from separate collection cache region  

**Answer: B** — L2C caching for entity and collection are **independent**. `@Cache` on the entity caches the entity fields. The `products` collection has **no `@Cache` annotation**, so it is NEVER stored in L2C — it is always fetched from DB when accessed. To cache the collection, add `@Cache(usage = READ_WRITE)` on the `List<Product> products` field separately.

---

**Q5 — Concurrency Strategy Selection**

A banking system needs to cache `ExchangeRate` entities. Exchange rates are updated once per day at midnight and read millions of times per day. The system uses Spring Boot with JpaTransactionManager (no JTA). Which strategy is optimal?

A) `READ_ONLY` — rates never change  
B) `TRANSACTIONAL` — financial data needs transactional consistency  
C) `NONSTRICT_READ_WRITE` — rates change rarely, brief staleness acceptable overnight  
D) `READ_WRITE` — strict freshness during the daily update window  

**Answer: D** — `READ_ONLY` is wrong: rates DO change (once/day). `TRANSACTIONAL` is unavailable without JTA. `NONSTRICT_READ_WRITE` risks serving stale rates briefly during midnight update — unacceptable for financial calculations. `READ_WRITE` uses soft locks: during the midnight update window, readers get consistent fresh data. For the other 23.5 hours, reads are served from cache efficiently. This is the correct balance.

---

**Q6 — Spring `@Cacheable` vs Hibernate L2C**
```java
@Cacheable(value = "products", key = "#id")
@Transactional(readOnly = true)
public ProductDto findProductById(Long id) {
    Product product = productRepo.findById(id).get();
    return ProductDto.from(product);
}
```

If `@Cacheable` returns a cached result (Spring cache HIT), which statement is TRUE?

A) Hibernate L2C is also checked and updated  
B) No transaction is started — the method body doesn't execute  
C) A transaction is started but no SQL is executed  
D) Both Spring cache and Hibernate L2C return data to assemble the response  

**Answer: B** — When Spring `@Cacheable` has a cache HIT, the **method body does not execute at all**. The proxy intercepts the call, finds the cached `ProductDto`, and returns it immediately. `@Transactional` is never applied (transaction never starts). Hibernate L2C is never consulted. No SQL executes. The `ProductDto` is returned directly from Spring's cache.

---

**Q7 — L1 Cache Scope**
```java
@Transactional
public void txA() {
    User user = userRepo.findById(1L).get(); // TX A's L1 cache
    user.setName("Alice");
}

@Transactional
public void txB() {
    User user = userRepo.findById(1L).get(); // TX B's L1 cache
    System.out.println(user.getName());
}
```

If `txB()` is called after `txA()` commits, what does `txB()` print?

A) Old name — L1 cache from TX A is shared and still holds old value  
B) "Alice" — L1 cache is application-scoped  
C) "Alice" — TX B gets a fresh L1 cache, loads from DB (or L2C), sees committed value  
D) `null` — entity was detached after TX A  

**Answer: C** — Each transaction gets its own `EntityManager` with its own **empty** L1 cache. TX B's `em.find(User.class, 1L)` starts with an empty L1. It checks L2C (if configured), then DB. Since TX A committed the name change as "Alice", TX B reads "Alice". L1 caches are never shared between transactions.

---

**Q8 — `READ_ONLY` Violation**
```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_ONLY)
public class Country {
    @Id Long id;
    String name;
}

@Transactional
public void updateCountryName(Long id, String newName) {
    Country country = countryRepo.findById(id).get();
    country.setName(newName);
    // flush happens at commit
}
```

What happens when the transaction commits?

A) UPDATE SQL executed, L2C updated, no issue  
B) `UnsupportedOperationException` or `IllegalStateException` thrown — READ_ONLY entities cannot be modified  
C) UPDATE SQL executed but L2C not updated (READ_ONLY prevents cache write)  
D) Silently ignored — dirty checking is disabled for READ_ONLY cached entities  

**Answer: B** — Hibernate enforces `READ_ONLY` at the cache level. When the transaction attempts to flush the modification, Hibernate detects that the entity uses `READ_ONLY` cache strategy and throws an exception. The `READ_ONLY` strategy includes an assertion: modifications to READ_ONLY entities are programming errors, and Hibernate fails fast to catch them.

---

## 4️⃣ Trick Analysis

**JPQL does not check L1 cache before executing SQL (Q1)**:
This is the most common L1 cache misconception. `em.find()` IS L1-aware — it checks the identity map before any SQL. JPQL queries (`createQuery()`, derived queries, `@Query`) are NOT L1-aware at execution time — they always fire SQL. However, when JPQL results are reconciled with the identity map, if the fetched primary key is already in L1, the in-memory L1 instance is returned (not the DB value). This means JPQL fires SQL but still returns the L1-cached object. The output is: same SQL fired + same Java object returned.

**Collection cache is independent of entity cache (Q4)**:
Entity `@Cache` and collection `@Cache` are completely separate. An entity may be in L2C while its collection always hits the DB. The collection must have its own `@Cache` annotation to be cached. The collection cache stores only the primary keys of child entities, not the full child entities. Each child entity is then looked up in its own entity cache region. Missing either annotation means that piece always hits the DB.

**Query cache invalidation is table-level, not query-level (Q3)**:
Any INSERT, UPDATE, or DELETE on a table invalidates ALL query cache entries for that table — regardless of whether the specific query result was affected. Adding a new product to a different category invalidates the `findActiveProductsByCategory("Electronics")` cache even though Electronics products didn't change. This makes query cache unsuitable for active tables and valuable only for truly static reference data.

**`READ_WRITE` soft lock causes MISS not stale read (Q2, D)**:
When a reader encounters a soft lock on a cache entry (placed by an updating transaction), it does NOT serve the old cached value. It treats the entry as a cache MISS and fetches from the DB. This is what makes `READ_WRITE` strictly consistent — concurrent readers during an update always get either the new value (after commit) or the DB value (during update), never a stale cached value.

**Spring `@Cacheable` HIT skips `@Transactional` entirely (Q6)**:
`@Cacheable` is a higher-level Spring AOP interceptor applied before `@Transactional`. When the cache has a hit, the method body — including all Spring AOP advice including `@Transactional` — never executes. No transaction is opened, no EntityManager is created, no Hibernate interaction occurs. The return value comes purely from Spring's cache. This is why `@Cacheable` is so powerful: it completely bypasses the entire JPA stack.

**`READ_ONLY` entities throw on modification (Q8)**:
`READ_ONLY` is a promise to Hibernate that the entity will never be modified. Hibernate enforces this at flush time — not at setter time. The setter executes normally (it's just a field assignment). The exception is thrown during flush when Hibernate detects a dirty `READ_ONLY` entity. This behavior varies slightly by Hibernate version but the exam expectation is that modifying a `READ_ONLY` cached entity throws an exception.

---

## 5️⃣ Summary Sheet

### L1 vs L2C vs Spring Cache Comparison

| Aspect | L1 (PC Identity Map) | L2C (Hibernate) | Spring @Cacheable |
|---|---|---|---|
| Scope | Single TX / EM | Application-wide | Application-wide |
| Storage | JVM heap, ThreadLocal | External (Caffeine, EhCache, Redis) | External (Caffeine, Redis, etc.) |
| What's cached | Entity object | Disassembled field values | Method return value (any type) |
| Opt-in | No — always active | Yes — @Cache per entity | Yes — @Cacheable per method |
| Invalidation | TX end, em.clear() | Automatic on write | Manual @CacheEvict or TTL |
| em.find() aware? | YES — checks L1 first | YES — checks L2C after L1 miss | No |
| JPQL aware? | No — SQL fires always | Via Query Cache only | No |
| Thread-safe | No (per-thread) | Yes | Yes |

### Cache Concurrency Strategy Reference

| Strategy | Writes OK? | Brief Stale? | JTA Required? | Best For |
|---|---|---|---|---|
| `READ_ONLY` | No (exception) | Never | No | Static reference data |
| `NONSTRICT_READ_WRITE` | Yes | Yes (briefly) | No | Infrequent updates |
| `READ_WRITE` | Yes | Never (soft lock) | No | Regular updates, Spring Boot |
| `TRANSACTIONAL` | Yes | Never | YES | JTA, financial, JBoss |

### L2C Setup Checklist

```
1. Add hibernate-jcache + caffeine + jcache dependencies
2. Set hibernate.cache.use_second_level_cache=true in properties
3. Set hibernate.cache.region.factory_class to JCacheRegionFactory
4. Add @Cache(usage=...) to each entity class (opt-in per entity)
5. Add @Cache(usage=...) to each collection field to cache collections
6. Add @Version to entities using READ_WRITE strategy
7. Optionally: set hibernate.cache.use_query_cache=true for Query Cache
8. Optionally: add @QueryHints with cacheable=true per query method
```

### Key Rules

```
1.  L1 cache = identity map, per-transaction, automatically managed, cannot disable
2.  em.find() checks L1 then L2C then DB; JPQL always goes to DB first
3.  JPQL results reconciled with L1 identity map — same Java object returned if in L1
4.  L2C is OPT-IN — @Cache required on each entity; default: no L2C
5.  Collection caching is SEPARATE from entity caching — needs own @Cache annotation
6.  L2C stores disassembled field values, NOT Java entity objects
7.  READ_ONLY: fastest, throws exception on modification
8.  READ_WRITE: soft lock prevents stale reads during concurrent updates; needs @Version
9.  NONSTRICT_READ_WRITE: invalidates on write, brief staleness possible
10. TRANSACTIONAL: requires JTA; not available in standard Spring Boot
11. Query Cache invalidated by ANY write to referenced table (table-level, not query-level)
12. Query Cache useful ONLY for read-mostly or read-only tables
13. Spring @Cacheable: cache HIT = method body NEVER executes (no TX, no SQL)
14. Spring @Cacheable eviction is MANUAL (@CacheEvict); Hibernate L2C eviction is AUTOMATIC
15. Hibernate L2C statistics: getSecondLevelCacheHitCount(), getMissCount(), getPutCount()
16. emf.getCache().evict(Class, id) — programmatic L2C eviction
17. emf.getCache().evictAll() — clears entire L2C
18. Duplicate caching (L2C + @Cacheable for same data): complex invalidation bugs — avoid
```

---
