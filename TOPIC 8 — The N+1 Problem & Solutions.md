# 🏛️ TOPIC 8 — The N+1 Problem & Solutions

---

## 1️⃣ Conceptual Explanation

### What N+1 Actually Is — First Principles

The N+1 problem is not a Hibernate bug. It is a **fundamental consequence of lazy loading combined with iteration**. It occurs when:

1. You execute **1 query** to load a list of N parent entities
2. For each of the N entities, Hibernate executes **1 additional query** to load a lazy association
3. Total: **1 + N queries** instead of the optimal 1 or 2

The name is slightly misleading — it should be called "1 + N" (one first, then N more), but "N+1" is the established term.

```
Scenario: Load 100 Orders and display each order's customer name

Without N+1 awareness:
  Query 1: SELECT * FROM orders WHERE status='PENDING'   ← 1 query, 100 rows
  Query 2: SELECT * FROM customers WHERE id=1            ← for order 1
  Query 3: SELECT * FROM customers WHERE id=2            ← for order 2
  ...
  Query 101: SELECT * FROM customers WHERE id=100        ← for order 100
  Total: 101 queries

With JOIN FETCH:
  Query 1: SELECT o.*, c.* FROM orders o 
           JOIN customers c ON o.customer_id=c.id
           WHERE o.status='PENDING'
  Total: 1 query
```

**The N+1 problem is silent** — it doesn't throw exceptions, produce wrong results, or trigger warnings by default. It silently degrades performance. A method that works fine with 10 records in development becomes catastrophically slow with 10,000 records in production.

---

### Why N+1 Happens — The Internal Trigger Chain

Understanding the trigger chain prevents N+1 at the design level:

```
Step 1: Initial query loads parent entities
  List<Order> orders = orderRepo.findByStatus(PENDING);
  SQL: SELECT * FROM orders WHERE status='PENDING'
  Result: 100 Order objects, each with customer = PersistentProxy(id=X)

Step 2: Business logic iterates and accesses lazy association
  for (Order order : orders) {
      String customerName = order.getCustomer().getName(); // ← TRIGGER
      // .getName() is first non-ID call on proxy
      // Hibernate: "proxy not initialized, session is open → fire SELECT"
      // SQL: SELECT * FROM customers WHERE id=X
  }

Step 3: Each iteration triggers one proxy initialization
  100 orders × 1 SELECT per customer = 100 extra queries

Step 4: Total = 1 (orders) + 100 (customers) = 101 queries
```

**N+1 with collections (even worse):**

```
List<Customer> customers = customerRepo.findAll(); // 50 customers
for (Customer c : customers) {
    List<Order> orders = c.getOrders(); // PersistentBag
    int count = orders.size();          // TRIGGER — initializes PersistentBag
    // SQL: SELECT * FROM orders WHERE customer_id=X
}
// Total: 1 + 50 = 51 queries
// If each customer has 100 orders: 50 × 100 = 5,000 rows loaded total
```

---

### Detecting N+1 — The Four Methods

#### Method 1 — SQL Logging (Development)

```properties
# application.properties:
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
# TRACE shows bind parameters — reveals repeated queries with different IDs
```

**Recognizing N+1 in logs:**
```
-- Query 1: parent
Hibernate: select order0_.id, order0_.status, order0_.customer_id 
           from orders order0_ where order0_.status=?

-- N identical queries with different IDs = N+1:
Hibernate: select customer0_.id, customer0_.name 
           from customers customer0_ where customer0_.id=?
Hibernate: select customer0_.id, customer0_.name 
           from customers customer0_ where customer0_.id=?
Hibernate: select customer0_.id, customer0_.name 
           from customers customer0_ where customer0_.id=?
-- ... repeated 100 times
```

The pattern: **same query structure, different parameter values, repeated N times** = N+1.

---

#### Method 2 — Hibernate Statistics (Development + Production Monitoring)

```java
// Enable statistics:
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.stat=DEBUG

// Programmatic access:
@Autowired
SessionFactory sessionFactory;

public void checkStatistics() {
    Statistics stats = sessionFactory.getStatistics();
    
    long queryCount = stats.getQueryExecutionCount();
    long entityFetchCount = stats.getEntityFetchCount(); // proxy initializations
    long collectionFetchCount = stats.getCollectionFetchCount(); // collection initializations
    
    System.out.println("Queries executed: " + queryCount);
    System.out.println("Entity fetches (proxy init): " + entityFetchCount);
    System.out.println("Collection fetches: " + collectionFetchCount);
    
    // N+1 indicator: entityFetchCount >> queryCount
    // If queryCount=1 and entityFetchCount=100: N+1 present
}
```

**Statistics in test assertions:**

```java
@Test
public void shouldNotProduceNPlusOne() {
    sessionFactory.getStatistics().clear();
    
    orderService.findAllPendingOrders(); // method under test
    
    long queryCount = sessionFactory.getStatistics().getQueryExecutionCount();
    assertThat(queryCount).isLessThanOrEqualTo(2); // assert no N+1
}
```

---

#### Method 3 — P6Spy (Query Interception)

P6Spy wraps the JDBC driver and logs every SQL with timing and parameters — more powerful than Hibernate logging:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>p6spy</groupId>
    <artifactId>p6spy</artifactId>
    <version>3.9.1</version>
</dependency>
```

```properties
# application.properties
spring.datasource.url=jdbc:p6spy:postgresql://localhost:5432/mydb
spring.datasource.driver-class-name=com.p6spy.engine.spy.P6SpyDriver

# spy.properties (on classpath)
driverlist=org.postgresql.Driver
appender=com.p6spy.engine.spy.appender.Slf4JLogger
logMessageFormat=com.p6spy.engine.spy.appender.CustomLineFormat
customLogMessageFormat=%(executionTime) ms | %(sql)
```

P6Spy reveals **exact parameters** and **timing per query** — easier to spot N+1 than raw SQL logging.

---

#### Method 4 — Hypersistence Optimizer / datasource-proxy (Advanced)

```java
// datasource-proxy: wrap DataSource to count queries
@Bean
public DataSource dataSource(DataSource originalDataSource) {
    return ProxyDataSourceBuilder
        .create(originalDataSource)
        .name("DS-Proxy")
        .countQuery()
        .logQueryToSysOut()
        .build();
}

// In tests: assert exact query count
@Autowired
ProxyDataSource proxyDataSource;

@Test
public void testQueryCount() {
    proxyDataSource.getQueryExecutor().getSuccess(); // reset
    
    orderService.findPendingOrders();
    
    assertThat(proxyDataSource.getQueryExecutor().getSuccess())
        .isEqualTo(2); // exactly 2 queries expected
}
```

---

### Solution 1 — `JOIN FETCH` (JPQL)

`JOIN FETCH` instructs Hibernate to load the association in the same SQL query using a JOIN. This is the most direct solution.

**Syntax and behavior:**

```java
// JPQL with JOIN FETCH:
@Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.status = :status")
List<Order> findByStatusWithCustomer(@Param("status") OrderStatus status);

// Generated SQL:
// SELECT o.id, o.status, o.customer_id, c.id, c.name, c.email
// FROM orders o
// INNER JOIN customers c ON o.customer_id = c.id
// WHERE o.status = ?

// OUTER JOIN FETCH (for optional associations):
@Query("SELECT o FROM Order o LEFT JOIN FETCH o.discount WHERE o.status = :status")
List<Order> findByStatusWithOptionalDiscount(@Param("status") OrderStatus status);
// LEFT JOIN preserves orders that have no discount
```

**`JOIN FETCH` with `DISTINCT`:**

When fetching a collection (`@OneToMany`), JOIN FETCH creates multiple result rows per parent (one per child). Without `DISTINCT`, the parent entity appears multiple times in the result list:

```java
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
List<Order> findByStatusWithItems(@Param("status") OrderStatus status);

// Without DISTINCT: if order has 3 items → order appears 3 times in result List
// SQL DISTINCT operates on columns; JPA DISTINCT deduplicates at the entity level
// Hibernate 6: DISTINCT in JPQL does NOT add DISTINCT to SQL by default
//              Use QueryHints.HINT_PASS_DISTINCT_THROUGH = false to suppress SQL DISTINCT
```

```java
// Hibernate 6 + Spring Data — correct approach:
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items")
@QueryHints(@QueryHint(name = "hibernate.query.passDistinctThrough", value = "false"))
List<Order> findAllWithItems();
// JPQL DISTINCT: deduplicates Order objects in Java
// passDistinctThrough=false: does NOT add DISTINCT to SQL (avoids sort overhead)
```

---

### `JOIN FETCH` Limitations

#### Limitation 1 — Pagination with Collection Fetch

This is the most dangerous `JOIN FETCH` trap in production:

```java
// DANGEROUS — Hibernate warning: HHH90003004
@Query("SELECT o FROM Order o JOIN FETCH o.items")
Page<Order> findAllWithItems(Pageable pageable);

// What Hibernate does:
// SQL: SELECT o.*, i.* FROM orders o JOIN order_items i ON o.id=i.order_id
// NO SQL-level LIMIT/OFFSET
// Loads ALL rows into memory
// Then paginates in Java (in-memory pagination)
// Warning: "HHH90003004: firstResult/maxResults specified with collection fetch; 
//          applying in memory"

// With 1 million orders × 10 items = 10 million rows in memory → OutOfMemoryError
```

**Solutions for pagination + collection fetch:**

```java
// SOLUTION A: Two-query approach (recommended)
// Query 1: paginate IDs only (no fetch)
@Query(value = "SELECT o FROM Order o",
       countQuery = "SELECT COUNT(o) FROM Order o")
Page<Order> findAllPaged(Pageable pageable);
// SQL: SELECT * FROM orders LIMIT ? OFFSET ?

// Query 2: fetch associations for those IDs
@Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.id IN :ids")
List<Order> findByIdsWithItems(@Param("ids") List<Long> ids);

// Service layer combines them:
@Transactional(readOnly = true)
public Page<Order> findOrdersWithItems(Pageable pageable) {
    Page<Order> orderPage = orderRepo.findAllPaged(pageable);
    List<Long> ids = orderPage.map(Order::getId).toList();
    List<Order> withItems = orderRepo.findByIdsWithItems(ids);
    // Map withItems back to the Page...
}

// SOLUTION B: @BatchSize on collection mapping
@BatchSize(size = 25)
@OneToMany(mappedBy = "order")
private List<OrderItem> items;
// Pagination works correctly (SQL LIMIT/OFFSET on orders)
// Items loaded in batches of 25 when accessed
```

#### Limitation 2 — MultipleBagFetchException

```java
// THROWS MultipleBagFetchException at query execution:
@Query("SELECT o FROM Order o JOIN FETCH o.items JOIN FETCH o.promotions")
List<Order> findWithItemsAndPromotions();

// Root cause: 'items' and 'promotions' are both List (Bag in Hibernate terms)
// Hibernate refuses to fetch multiple Bags simultaneously — cartesian product is ambiguous

// Fix 1: Change one or both to Set:
private Set<OrderItem> items = new HashSet<>();
private Set<Promotion> promotions = new HashSet<>();
// Now: JOIN FETCH both Sets works (cartesian product is acceptable for Sets)

// Fix 2: Use separate queries:
// First fetch with items, then @BatchSize or separate query for promotions

// Fix 3: Use @Fetch(FetchMode.SUBSELECT) for one collection
@Fetch(FetchMode.SUBSELECT)
private List<Promotion> promotions;
// JOIN FETCH items only, SUBSELECT loads promotions separately
```

---

### Solution 2 — `@EntityGraph`

`@EntityGraph` provides a declarative, annotation-based way to specify which associations to fetch eagerly for a specific repository method, without writing JPQL.

**Named EntityGraph:**

```java
@Entity
@NamedEntityGraph(
    name = "Order.full",
    attributeNodes = {
        @NamedAttributeNode("customer"),
        @NamedAttributeNode(value = "items", subgraph = "items.subgraph")
    },
    subgraphs = {
        @NamedSubgraph(
            name = "items.subgraph",
            attributeNodes = {
                @NamedAttributeNode("product"),
                @NamedAttributeNode("discount")
            }
        )
    }
)
public class Order {
    @ManyToOne(fetch = LAZY) Customer customer;
    @OneToMany(mappedBy="order", fetch=LAZY) List<OrderItem> items;
}

// In repository:
@EntityGraph("Order.full")
Optional<Order> findById(Long id);
// Loads: customer, items, items.product, items.discount in ONE query
```

**Dynamic EntityGraph (no annotation on entity):**

```java
@EntityGraph(attributePaths = {"customer", "items", "items.product"})
List<Order> findByStatus(OrderStatus status);
// Spring Data creates EntityGraph at runtime
// attributePaths uses dot notation for nested associations
```

**`loadgraph` vs `fetchgraph` semantics:**

```java
// loadgraph (Spring Data @EntityGraph default):
// → specified attributes: EAGER
// → unspecified attributes: use their MAPPING defaults
// → if mapping says LAZY: stays LAZY
// → if mapping says EAGER: stays EAGER

// fetchgraph (override Spring Data default):
@EntityGraph(attributePaths = {"customer"}, type = EntityGraph.EntityGraphType.FETCH)
// → specified attributes: EAGER
// → unspecified attributes: ALL become LAZY regardless of mapping
// → useful when you have EAGER mappings you want to suppress
```

**`@EntityGraph` internal implementation:**

Spring Data passes the `EntityGraph` to JPA as a query hint:
```java
query.setHint("javax.persistence.loadgraph", entityGraph); // or fetchgraph
```

Hibernate receives this hint and modifies its fetch plan for that specific query execution. The entity mapping's fetch type is overridden for this query only.

**`@EntityGraph` generates LEFT JOIN or separate SELECT?**

By default, Hibernate uses `LEFT JOIN FETCH` for the specified attributes. For collections, same cartesian product and pagination issues as JOIN FETCH apply. Use `@BatchSize` as a complement for paginated collection fetching.

---

### Solution 3 — `@BatchSize`

`@BatchSize` is a Hibernate-specific annotation that changes how lazy proxies and collections are loaded. Instead of loading one at a time (N individual SELECTs), it loads them in batches using SQL `IN` clauses.

**How `@BatchSize` works internally:**

```java
@Entity
public class Customer {
    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    @BatchSize(size = 25)
    private List<Order> orders;
}

// Load 100 customers:
List<Customer> customers = customerRepo.findAll();
// SQL: SELECT * FROM customers — 100 customers, orders are lazy PersistentBags

// Access first customer's orders:
customers.get(0).getOrders().size(); // TRIGGER
// Hibernate sees: need to initialize orders for customer 1
// BatchSize=25: "let me initialize for the next 25 customers at once"
// SQL: SELECT * FROM orders WHERE customer_id IN (1,2,3,...,25)
// Initializes orders for customers 1-25 in one query

// Access 26th customer's orders:
customers.get(25).getOrders().size(); // new batch
// SQL: SELECT * FROM orders WHERE customer_id IN (26,27,...,50)

// Total queries: 1 (customers) + 4 (batches of 25) = 5
// Without @BatchSize: 1 + 100 = 101 queries
```

**`@BatchSize` on entity level (for proxy batching):**

```java
@Entity
@BatchSize(size = 25) // batch load PROXIES for this entity type
public class Customer {
    ...
}

// When loading orders with lazy customer proxies:
List<Order> orders = orderRepo.findAll(); // 100 orders
// Each has Customer proxy

orders.get(0).getCustomer().getName(); // triggers batch
// SQL: SELECT * FROM customers WHERE id IN (1,2,...,25)
// Loads customers for orders 0-24

orders.get(25).getCustomer().getName(); // triggers next batch
// SQL: SELECT * FROM customers WHERE id IN (26,...,50)
```

**Global `@BatchSize` configuration:**

```properties
# Apply default batch size to ALL lazy associations:
spring.jpa.properties.hibernate.default_batch_fetch_size=25
# Equivalent to @BatchSize(25) on every lazy association
# Strongly recommended as a baseline N+1 mitigation for most applications
```

**`@BatchSize` advantages:**
- Works correctly with pagination (no in-memory pagination issue)
- Simple to configure globally
- No JPQL changes required
- Reduces N+1 without eliminating the lazy loading pattern

**`@BatchSize` disadvantages:**
- Still fires multiple queries (just fewer)
- Less predictable than JOIN FETCH (query count depends on batch size and collection sizes)
- Extra `IN` clause queries

---

### Solution 4 — `@Fetch(FetchMode.SUBSELECT)`

SUBSELECT fires ONE query for all collections of the same type across ALL loaded parent entities.

```java
@Entity
public class Customer {
    @OneToMany(mappedBy = "customer")
    @Fetch(FetchMode.SUBSELECT)
    private List<Order> orders;
}

// Load customers:
List<Customer> customers = em.createQuery(
    "FROM Customer WHERE region = 'EMEA'", Customer.class).getResultList();
// SQL: SELECT * FROM customers WHERE region='EMEA'
// 50 customers loaded, orders are lazy

// First access:
customers.get(0).getOrders().size();
// SQL: SELECT * FROM orders WHERE customer_id IN
//      (SELECT id FROM customers WHERE region='EMEA')
// ↑ Original query repeated as subselect
// Loads orders for ALL 50 customers in ONE query

// Subsequent access:
customers.get(49).getOrders().size(); // NO SQL — already loaded
```

**SUBSELECT vs BatchSize tradeoffs:**

| Aspect | `@BatchSize(25)` | `@Fetch(SUBSELECT)` |
|---|---|---|
| Number of queries | `ceil(N/25)` batches | Always 1 |
| Memory | Batch-by-batch | ALL at once |
| Works with pagination | ✅ | ✅ |
| Predictable | ✅ (depends on N) | ✅ (always 1) |
| Risk | Small if N manageable | Large if all collections huge |
| Scope | Next N in session | ALL loaded parents |

**SUBSELECT limitation**: It captures the **original query's WHERE clause**. If the parent query changes or is complex, the subselect may be incorrect or overly broad.

---

### Solution 5 — DTO Projections

The most performance-optimal solution: **never load entities at all**. Load exactly the data needed via JPQL constructor expressions or interface projections.

**Constructor expression (DTO):**

```java
public class OrderSummaryDto {
    private final Long id;
    private final String customerName;
    private final BigDecimal total;
    private final int itemCount;
    
    public OrderSummaryDto(Long id, String customerName, 
                           BigDecimal total, Long itemCount) {
        this.id = id;
        this.customerName = customerName;
        this.total = total;
        this.itemCount = itemCount.intValue();
    }
    // getters...
}

@Query("SELECT new com.example.OrderSummaryDto(" +
       "o.id, c.name, o.total, COUNT(i)) " +
       "FROM Order o " +
       "JOIN o.customer c " +
       "LEFT JOIN o.items i " +
       "WHERE o.status = :status " +
       "GROUP BY o.id, c.name, o.total")
List<OrderSummaryDto> findOrderSummaries(@Param("status") OrderStatus status);

// SQL:
// SELECT o.id, c.name, o.total, COUNT(i.id)
// FROM orders o
// JOIN customers c ON o.customer_id=c.id
// LEFT JOIN order_items i ON o.id=i.order_id
// WHERE o.status=?
// GROUP BY o.id, c.name, o.total

// No entities loaded → no Persistence Context overhead
// No lazy associations → no LazyInitializationException possible
// Exactly the columns needed → minimal data transfer
```

**Interface projection (Spring Data):**

```java
public interface OrderSummary {
    Long getId();
    @Value("#{target.customer.name}")
    String getCustomerName();
    BigDecimal getTotal();
}

// Spring Data automatically generates query selecting only needed columns
List<OrderSummary> findByStatus(OrderStatus status);
// SQL: SELECT o.id, o.total, c.name FROM orders o JOIN customers c ... WHERE status=?
// Spring Data proxy implements OrderSummary interface
```

**When to use DTO projections:**
- Read-only operations (no need to track changes)
- Reporting / list views / summary data
- REST API responses (never expose entities directly)
- Performance-critical paths

**When NOT to use:**
- When you need to modify and save the loaded data (entities required)
- When the full entity is genuinely needed across the codebase

---

### Solution 6 — Native Queries for Extreme Cases

When JPQL cannot express the needed query efficiently, native SQL is the escape hatch:

```java
@Query(value = """
    SELECT o.id, o.status, o.total,
           c.name as customer_name,
           COUNT(i.id) as item_count,
           SUM(i.quantity * i.unit_price) as calculated_total
    FROM orders o
    JOIN customers c ON o.customer_id = c.id
    LEFT JOIN order_items i ON o.id = i.order_id
    WHERE o.created_at > :since
    GROUP BY o.id, o.status, o.total, c.name
    ORDER BY o.created_at DESC
    """, nativeQuery = true)
List<Object[]> findOrderSummariesNative(@Param("since") LocalDateTime since);

// Or with interface projection for cleaner access:
@Query(value = "SELECT o.id, c.name as customerName ...", nativeQuery = true)
List<OrderSummary> findOrderSummariesNativeProjected(@Param("since") LocalDateTime since);
```

---

### The Complete N+1 Decision Matrix

```
Is this a READ-ONLY operation (no entity modification)?
  YES → Use DTO projection (best performance, zero lazy issues)
  NO  → Need entities → continue

Is pagination involved?
  YES → Do NOT use JOIN FETCH on collections
        Use: two-query approach OR @BatchSize globally
  NO  → Continue

How many associations need fetching?
  One @ManyToOne or @OneToOne → JOIN FETCH or @EntityGraph
  One @OneToMany              → JOIN FETCH (with DISTINCT) or @EntityGraph
  Multiple @OneToMany (List)  → MultipleBagFetchException risk
                                 Use: one JOIN FETCH + @BatchSize on others
                                   OR: @Fetch(SUBSELECT) for secondary collections
                                   OR: DTO projection

Is this a global "always fetch together" pattern?
  YES → @BatchSize globally (hibernate.default_batch_fetch_size=25)
  NO  → Per-query solution (JOIN FETCH / @EntityGraph)

Is N very large (1000+ parents)?
  YES → @Fetch(SUBSELECT) or DTO projection
  NO  → @BatchSize or JOIN FETCH acceptable
```

---

## 2️⃣ Code Examples

### Example 1 — Detecting N+1 in Tests

```java
@DataJpaTest
@Import(QueryCountTestConfig.class)
class OrderRepositoryNPlusOneTest {
    
    @Autowired OrderRepository orderRepo;
    @Autowired SessionFactory sessionFactory;
    
    @BeforeEach
    void enableStats() {
        sessionFactory.getStatistics().setStatisticsEnabled(true);
        sessionFactory.getStatistics().clear();
    }
    
    @Test
    @Transactional
    void shouldNotProduceNPlusOneForOrderList() {
        // Given: 50 orders with customers in DB (set up in @BeforeEach)
        
        // When:
        List<Order> orders = orderRepo.findAllWithCustomer();
        orders.forEach(o -> o.getCustomer().getName()); // access lazy field
        
        // Then: assert no N+1
        Statistics stats = sessionFactory.getStatistics();
        
        assertThat(stats.getQueryExecutionCount())
            .as("Should execute at most 2 queries (orders + batch customers)")
            .isLessThanOrEqualTo(2);
        
        assertThat(stats.getEntityFetchCount())
            .as("Should not fetch entities one by one (N+1 indicator)")
            .isZero();
    }
    
    @Test
    @Transactional
    void demonstratesNPlusOne() {
        // Method WITHOUT JOIN FETCH:
        List<Order> orders = orderRepo.findAll(); // 50 orders
        orders.forEach(o -> o.getCustomer().getName()); // triggers N+1
        
        Statistics stats = sessionFactory.getStatistics();
        System.out.println("Queries: " + stats.getQueryExecutionCount()); // 51
        System.out.println("Entity fetches: " + stats.getEntityFetchCount()); // 50
    }
}
```

---

### Example 2 — JOIN FETCH All Variations

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Basic JOIN FETCH — @ManyToOne (no collection, no cartesian product):
    @Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.status = :s")
    List<Order> findByStatusWithCustomer(@Param("s") OrderStatus s);

    // JOIN FETCH collection — need DISTINCT:
    @Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.status = :s")
    @QueryHints(@QueryHint(name = "hibernate.query.passDistinctThrough", value = "false"))
    List<Order> findByStatusWithItems(@Param("s") OrderStatus s);
    // passDistinctThrough=false: DISTINCT is applied in Java, not added to SQL

    // Multiple associations (one @ManyToOne + one @OneToMany — OK):
    @Query("SELECT DISTINCT o FROM Order o " +
           "JOIN FETCH o.customer " +
           "JOIN FETCH o.items i " +
           "JOIN FETCH i.product " + // nested — item's product
           "WHERE o.status = :s")
    @QueryHints(@QueryHint(name = "hibernate.query.passDistinctThrough", value = "false"))
    List<Order> findByStatusFull(@Param("s") OrderStatus s);

    // WRONG — two collections = MultipleBagFetchException if both are List:
    // @Query("SELECT o FROM Order o JOIN FETCH o.items JOIN FETCH o.promotions")
    // List<Order> broken(); // throws MultipleBagFetchException

    // Correct for two collections — use Set for at least one:
    // private Set<Promotion> promotions = new HashSet<>();
    @Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items JOIN FETCH o.promotions")
    List<Order> findWithItemsAndPromotions(); // works if promotions is Set
    
    // Pagination with JOIN FETCH — DANGEROUS for collections:
    // @Query("SELECT o FROM Order o JOIN FETCH o.items")
    // Page<Order> findPaged(Pageable p); // HHH90003004 warning — in-memory pagination!
    
    // Correct pagination approach — separate count query:
    @Query(value = "SELECT o FROM Order o JOIN FETCH o.customer WHERE o.status = :s",
           countQuery = "SELECT COUNT(o) FROM Order o WHERE o.status = :s")
    Page<Order> findByStatusPagedWithCustomer(@Param("s") OrderStatus s, Pageable p);
    // JOIN FETCH on @ManyToOne is safe with pagination (no cartesian product)
}
```

---

### Example 3 — `@EntityGraph` Complete Variations

```java
// Entity with named graphs:
@Entity
@NamedEntityGraph(
    name = "Order.withCustomer",
    attributeNodes = @NamedAttributeNode("customer")
)
@NamedEntityGraph(
    name = "Order.withItemsAndCustomer",
    attributeNodes = {
        @NamedAttributeNode("customer"),
        @NamedAttributeNode(value = "items", subgraph = "items")
    },
    subgraphs = @NamedSubgraph(
        name = "items",
        attributeNodes = @NamedAttributeNode("product")
    )
)
public class Order {
    @ManyToOne(fetch = LAZY) Customer customer;
    @OneToMany(mappedBy = "order", fetch = LAZY) List<OrderItem> items;
}

// Repository using named graphs:
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    @EntityGraph("Order.withCustomer")
    Optional<Order> findById(Long id); // overrides default findById
    
    @EntityGraph("Order.withItemsAndCustomer")
    List<Order> findByStatus(OrderStatus status);
    
    // Dynamic graph — no named graph needed:
    @EntityGraph(attributePaths = {"customer", "items", "items.product"})
    List<Order> findByStatusAndCreatedAtAfter(OrderStatus status, LocalDateTime since);
    
    // fetchgraph — ALL unspecified become LAZY:
    @EntityGraph(
        attributePaths = {"customer"},
        type = EntityGraph.EntityGraphType.FETCH
    )
    List<Order> findByStatusForListing(OrderStatus status);
    // Only customer is loaded; items LAZY even if mapped EAGER
}
```

---

### Example 4 — `@BatchSize` Global Configuration

```java
// Global configuration — affects all lazy associations:
// application.properties:
// spring.jpa.properties.hibernate.default_batch_fetch_size=25

// Or programmatic:
@Configuration
public class HibernateConfig {
    @Bean
    public HibernatePropertiesCustomizer hibernateCustomizer() {
        return props -> props.put("hibernate.default_batch_fetch_size", 25);
    }
}

// With global BatchSize=25 and 100 customers:
@Test
@Transactional
void batchSizeGlobalDemo() {
    List<Customer> customers = customerRepo.findAll(); // 100 customers
    // SQL: SELECT * FROM customers — 100 rows
    
    // Access orders for customer 0 (triggers batch 1):
    customers.get(0).getOrders().size();
    // SQL: SELECT * FROM orders WHERE customer_id IN (1,2,...,25)
    // Loads orders for customers 0-24 in one query
    
    customers.get(24).getOrders().size(); // NO SQL — already in batch 1
    
    customers.get(25).getOrders().size(); // triggers batch 2
    // SQL: SELECT * FROM orders WHERE customer_id IN (26,...,50)
    
    // Total for 100 customers: 1 + 4 = 5 queries
    // Without BatchSize: 1 + 100 = 101 queries
}

// Per-association BatchSize (overrides global):
@Entity
public class Customer {
    @OneToMany(mappedBy = "customer")
    @BatchSize(size = 10) // override global 25 for this specific collection
    private List<Order> orders;
}
```

---

### Example 5 — Two-Query Pagination Solution

```java
// Service combining paginated IDs + batch fetch:
@Service
public class OrderService {
    
    @Transactional(readOnly = true)
    public Page<OrderDto> findOrdersWithItems(Pageable pageable) {
        
        // Step 1: Paginate efficiently — no JOIN FETCH, gets correct LIMIT/OFFSET
        Page<Long> orderIdPage = orderRepo.findOrderIds(pageable);
        // SQL: SELECT o.id FROM orders o ORDER BY ... LIMIT ? OFFSET ?
        //      SELECT COUNT(o.id) FROM orders  (count query for Page)
        
        if (orderIdPage.isEmpty()) {
            return Page.empty(pageable);
        }
        
        List<Long> ids = orderIdPage.getContent();
        
        // Step 2: Fetch full data for those IDs with associations
        List<Order> orders = orderRepo.findByIdInWithItems(ids);
        // SQL: SELECT DISTINCT o.*, i.* FROM orders o 
        //      JOIN order_items i ON o.id=i.order_id
        //      WHERE o.id IN (?,?,...,?)
        
        // Step 3: Preserve page order (IN clause doesn't guarantee order):
        Map<Long, Order> orderMap = orders.stream()
            .collect(Collectors.toMap(Order::getId, o -> o));
        List<Order> orderedList = ids.stream()
            .map(orderMap::get)
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
        
        // Step 4: Convert to DTOs and wrap in Page:
        List<OrderDto> dtos = orderedList.stream()
            .map(OrderDto::from)
            .collect(Collectors.toList());
        
        return new PageImpl<>(dtos, pageable, orderIdPage.getTotalElements());
    }
}

// Repository methods:
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    @Query("SELECT o.id FROM Order o")
    Page<Long> findOrderIds(Pageable pageable);
    
    @Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.id IN :ids")
    @QueryHints(@QueryHint(name = "hibernate.query.passDistinctThrough", value = "false"))
    List<Order> findByIdInWithItems(@Param("ids") List<Long> ids);
}
```

---

### Example 6 — DTO Projection vs Entity — Performance Comparison

```java
// Scenario: Dashboard showing order statistics

// APPROACH 1: Load entities (bad for this use case):
@Transactional(readOnly = true)
public List<DashboardDto> getDashboardSlow() {
    List<Order> orders = orderRepo.findAll(); // loads ALL order fields
    return orders.stream().map(o -> {
        o.getCustomer().getName();  // N+1 trigger
        o.getItems().size();        // N+1 trigger
        return new DashboardDto(
            o.getId(),
            o.getCustomer().getName(),
            o.getItems().size(),
            o.getTotal()
        );
    }).collect(Collectors.toList());
    // Queries: 1 + N (customers) + N (items) = 2N+1
    // All entity fields loaded even though only 4 values needed
}

// APPROACH 2: DTO projection (optimal):
public record DashboardDto(Long orderId, String customerName, 
                            Long itemCount, BigDecimal total) {}

@Query("SELECT new com.example.DashboardDto(" +
       "o.id, c.name, COUNT(i), o.total) " +
       "FROM Order o " +
       "JOIN o.customer c " +
       "LEFT JOIN o.items i " +
       "GROUP BY o.id, c.name, o.total")
List<DashboardDto> findDashboardData();
// SQL: SELECT o.id, c.name, COUNT(i.id), o.total
//      FROM orders o JOIN customers c LEFT JOIN order_items i
//      GROUP BY o.id, c.name, o.total
// Queries: 1
// Columns loaded: 4 (not all entity fields)
// No entities in Persistence Context → no dirty checking overhead
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ**
What is the primary cause of the N+1 query problem?

A) Using `EAGER` fetch type on all associations  
B) Accessing a lazily-loaded association in a loop after the initial query  
C) Not using `@Transactional` on repository methods  
D) Having more than one `@OneToMany` association on an entity  

**Answer: B** — N+1 is caused by loading N parent entities and then accessing their lazy associations (one query per parent) in iteration. It can also occur with EAGER loading (Hibernate fires N separate SELECTs instead of a JOIN), but the root mechanism is per-entity loading in a loop.

---

**Q2 — Select All That Apply**
Which solutions correctly eliminate N+1 when loading 100 orders with their customer names for a paginated list? (Select all)

A) `JOIN FETCH o.customer` in JPQL with `Pageable` parameter  
B) `@EntityGraph(attributePaths={"customer"})` on the repository method  
C) `@BatchSize(size=25)` on the `customer` field  
D) `@Fetch(FetchMode.SUBSELECT)` on `customer`  
E) DTO projection with JOIN in JPQL  
F) `@ManyToOne(fetch=FetchType.EAGER)` on customer  

**Answer: B, C, E**
- A: JOIN FETCH on `@ManyToOne` with Pageable is safe (no collection cartesian product) but be careful — some versions may still have in-memory pagination. In most cases this is acceptable.
- B: Correct — EntityGraph is safe with pagination for @ManyToOne
- C: Correct — BatchSize reduces to ceil(100/25)=4 queries
- D: SUBSELECT is for collections not @ManyToOne — it generates incorrect SQL for single-valued associations
- E: Correct — DTO projection is always safe
- F: EAGER without JOIN → still N+1 (Hibernate fires N individual SELECTs for EAGER @ManyToOne in a list context)

---

**Q3 — SQL Prediction**
```java
@Entity
public class Department {
    @Id Long id;
    
    @OneToMany(mappedBy = "department")
    @Fetch(FetchMode.SUBSELECT)
    private List<Employee> employees;
}

// Execute:
List<Department> depts = em.createQuery(
    "FROM Department WHERE location='NYC'", Department.class).getResultList();
// Returns 10 departments

depts.get(0).getEmployees().size(); // Line A
depts.get(5).getEmployees().size(); // Line B
```

How many total SQL queries are executed?

A) 1 (initial) + 10 (one per department) = 11  
B) 1 (initial) + 1 (subselect for all departments at Line A) = 2  
C) 1 (initial) + 2 (one per access at Lines A and B) = 3  
D) 3 (initial + one batch for first 5 + one batch for next 5)  

**Answer: B** — SUBSELECT fires ONE query for ALL loaded departments when any one collection is accessed. Line A triggers: `SELECT * FROM employees WHERE department_id IN (SELECT id FROM departments WHERE location='NYC')`. Line B uses already-initialized data. Total: 2 queries.

---

**Q4 — MultipleBagFetchException**
```java
@Entity
public class Post {
    @Id Long id;
    
    @OneToMany(mappedBy = "post")
    private List<Comment> comments;
    
    @OneToMany(mappedBy = "post")
    private List<Tag> tags;
}

@Query("SELECT p FROM Post p JOIN FETCH p.comments JOIN FETCH p.tags")
List<Post> findAllWithCommentsAndTags();
```

What happens when this query is executed?

A) Returns correct results with all comments and tags loaded  
B) Returns correct results but with cartesian product rows (deduped by Hibernate)  
C) Throws `MultipleBagFetchException` at query execution time  
D) Throws `MultipleBagFetchException` at application startup  

**Answer: C** — Two `List` (Bag) collections cannot both be JOIN FETCHed simultaneously. Hibernate throws `MultipleBagFetchException` at query execution time (not startup). Fix: change at least one to `Set<Comment>` or `Set<Tag>`.

---

**Q5 — Pagination Trap**
```java
@Query("SELECT o FROM Order o JOIN FETCH o.items")
Page<Order> findAllWithItems(Pageable pageable);

orderRepo.findAllWithItems(PageRequest.of(0, 10));
```

What is the actual behavior?

A) Returns page 0 with 10 orders, items loaded via JOIN  
B) Hibernate warns and loads ALL orders into memory, paginates in Java  
C) SQL uses LIMIT 10 with JOIN, returns 10 * avg(items) rows  
D) Throws exception — pagination with JOIN FETCH is invalid  

**Answer: B** — Hibernate logs `HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory`. The SQL does NOT include LIMIT/OFFSET. All matching rows are loaded into memory, deduplicated, then paginated. With large datasets, this causes `OutOfMemoryError`.

---

**Q6 — BatchSize Calculation**
```java
// Global: hibernate.default_batch_fetch_size=10
// 35 orders loaded, each with lazy customer proxy

// First customer proxy is accessed:
orders.get(0).getCustomer().getName();
// How many SQL queries are fired to load ALL 35 customers 
// as they are accessed one by one?
```

A) 35 — one per customer  
B) 4 — batches of 10: 10+10+10+5  
C) 3 — batches: 10+10+15  
D) 1 — all loaded at once  

**Answer: B** — BatchSize=10 means: when ANY proxy is initialized, Hibernate also initializes the next (up to) 9 uninitialized proxies of the same type in the current persistence context. Batch 1: ids 1-10, Batch 2: ids 11-20, Batch 3: ids 21-30, Batch 4: ids 31-35. Total: 4 queries.

---

**Q7 — DTO vs Entity**
```java
// Approach A:
List<Order> orders = orderRepo.findAll();
return orders.stream()
    .map(o -> new OrderDto(o.getId(), o.getTotal()))
    .collect(toList());

// Approach B:
@Query("SELECT new com.example.OrderDto(o.id, o.total) FROM Order o")
List<OrderDto> findAllDtos();
```

Which statements are TRUE about the difference? (Select all)

A) Approach A loads all Order columns; Approach B loads only id and total  
B) Both approaches generate the same SQL  
C) Approach A puts entities in Persistence Context; Approach B does not  
D) Approach A enables dirty checking overhead; Approach B does not  
E) Approach B is always preferable regardless of context  

**Answer: A, C, D**
- B is false: A loads all columns, B loads only id and total
- E is false: if you need to MODIFY the entities, A is necessary (entities are tracked)

---

**Q8 — EAGER Still Causes N+1**
```java
@ManyToOne(fetch = FetchType.EAGER)
private Customer customer;

List<Order> orders = em.createQuery("FROM Order", Order.class).getResultList();
```

What SQL is generated for the customers?

A) One JOIN query loading all customers with orders  
B) N individual `SELECT * FROM customers WHERE id=?` queries (N+1)  
C) No customer queries — EAGER is overridden by JPQL  
D) One `SELECT * FROM customers WHERE id IN (...)` batch query  

**Answer: B** — EAGER without JOIN FETCH still causes N+1 for JPQL queries. Hibernate knows it must load customers eagerly, but it does so via individual SELECTs (one per unique customer FK value), not via JOIN. This is because the JPQL query was written without JOIN FETCH, so Hibernate loads associations via secondary selects. `JOIN FETCH` is needed to get the JOIN behavior.

---

## 4️⃣ Trick Analysis

**EAGER still causes N+1 (Q8)**:
This shocks many developers. They think EAGER = JOIN. It doesn't. `FetchType.EAGER` means "always load before returning." HOW it loads depends on context. For `em.find()`, Hibernate uses a JOIN. For JPQL queries without JOIN FETCH, Hibernate uses separate SELECT statements per entity. The result: `@ManyToOne(fetch=EAGER)` on a list query = N+1. The only safe approach: map LAZY, use JOIN FETCH per-query.

**`@Fetch(SUBSELECT)` on `@ManyToOne` is wrong (Q2)**:
SUBSELECT makes semantic sense only for COLLECTIONS (it generates `WHERE fk IN (SELECT id FROM parent WHERE ...)`). For `@ManyToOne` (a single-valued association), applying SUBSELECT doesn't make sense — Hibernate either ignores it or generates incorrect SQL. BatchSize is the correct tool for batching proxy initialization on `@ManyToOne`.

**MultipleBagFetchException at execution, not startup (Q4)**:
This exception fires when the query executes, not at startup. Many developers are surprised by this — the application starts cleanly, but the specific endpoint that calls this query throws at runtime. The fix is changing `List` to `Set` on at least one of the collections.

**Pagination with JOIN FETCH on collections is silent data destruction (Q5)**:
The warning `HHH90003004` is logged but doesn't throw. The query "works" (returns correct results), but with a catastrophically wrong execution plan. In tests with small datasets (10 orders, 3 items each = 30 rows), it seems fine. In production (100,000 orders, 10 items each = 1,000,000 rows in memory), it causes OutOfMemoryError. This is one of the most dangerous silent performance bugs.

**BatchSize calculation (Q6)**:
Hibernate's batch initialization collects ALL uninitialized proxies/collections of the same type in the current PC and groups them into batches. With 35 proxies and batch=10: first trigger initializes 10, second trigger initializes 10, etc. Hibernate uses Fibonacci-like batch sizes internally for optimization (not strictly `n, n, n, remainder`) — the exact batching may vary — but the exam expectation is the simple `ceil(N/batchSize)` calculation.

**DTO projection bypasses Persistence Context (Q7)**:
When a JPQL constructor expression `new com.example.Dto(...)` is used, the results are constructed via the DTO constructor directly from the `ResultSet`. No entities are placed in the identity map, no snapshots created, no dirty checking. This makes DTO queries faster not just because fewer columns are loaded — but because the entire entity-tracking overhead is bypassed.

---

## 5️⃣ Summary Sheet

### N+1 Detection Checklist

```
Signs of N+1:
  □ Repeated identical SQL with different parameter values in logs
  □ Statistics: entityFetchCount or collectionFetchCount >> queryCount
  □ Response time grows linearly with result set size
  □ P6Spy shows clusters of rapid identical queries

Detection tools:
  □ spring.jpa.show-sql=true (basic)
  □ logging.level.org.hibernate.SQL=DEBUG (full SQL)
  □ hibernate.generate_statistics=true (programmatic)
  □ P6Spy (JDBC-level interception with timing)
  □ datasource-proxy (query count assertions in tests)
```

### Solution Comparison Matrix

| Solution | Queries | Pagination Safe | Modifiable? | Complexity |
|---|---|---|---|---|
| JOIN FETCH (×1 assoc) | 1 | ✅ (@ManyToOne) | ✅ | Low |
| JOIN FETCH (collection) | 1 | ❌ (in-memory!) | ✅ | Low |
| @EntityGraph | 1 | ✅ (@ManyToOne) | ✅ | Low |
| @BatchSize(n) | ceil(N/n)+1 | ✅ | ✅ | Very Low |
| @Fetch(SUBSELECT) | 2 | ✅ | ✅ | Low |
| Two-query approach | 2 | ✅ | ✅ | Medium |
| DTO projection | 1 | ✅ | ❌ | Medium |
| Native query | 1 | ✅ | ❌ | High |

### MultipleBagFetchException — Causes and Fixes

```
Cause:    JOIN FETCH on two List<T> collections simultaneously
Error:    MultipleBagFetchException at query execution time
Fix 1:    Change at least one collection to Set<T>
Fix 2:    JOIN FETCH one collection, @Fetch(SUBSELECT) or @BatchSize on other
Fix 3:    Use two separate queries
Fix 4:    DTO projection (no collection loading at all)
```

### Pagination + Collections — Safe Strategies

```
DANGEROUS (avoid):
  @Query("SELECT o FROM Order o JOIN FETCH o.items")
  Page<Order> findPaged(Pageable p);
  → In-memory pagination, HHH90003004 warning, OOM risk

SAFE:
  Option A: Two queries
    1. Page<Long> findIds(Pageable p)        ← correct LIMIT/OFFSET
    2. List<Order> findByIdIn(List<Long> ids) ← JOIN FETCH on IDs
    
  Option B: @BatchSize
    Page<Order> findAll(Pageable p)           ← correct LIMIT/OFFSET
    @BatchSize(size=N) on collection          ← batch loads when accessed
    
  Option C: DTO projection
    Page<OrderDto> findDtos(Pageable p)       ← no entities, no collections
```

### Global Baseline Configuration (Recommended)

```properties
# Eliminates most N+1 problems with zero code changes:
spring.jpa.properties.hibernate.default_batch_fetch_size=25

# Enable during development to detect N+1:
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.generate_statistics=true
logging.level.org.hibernate.stat=DEBUG

# Disable in production (performance cost):
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.generate_statistics=false
```

### Key Rules

```
1. N+1 = 1 parent query + N association queries (one per parent)
2. EAGER fetch type does NOT prevent N+1 for JPQL without JOIN FETCH
3. JOIN FETCH collection + Pageable = in-memory pagination (DANGEROUS)
4. JOIN FETCH two Lists simultaneously = MultipleBagFetchException
5. @BatchSize(n) is safe with pagination — recommended global default
6. SUBSELECT = ONE query for ALL parent collections when any is accessed
7. DTO projection = zero N+1 risk, zero Persistence Context overhead
8. hibernate.default_batch_fetch_size=25 is a baseline production setting
9. Set<T> preferred over List<T> for @ManyToMany to avoid bag issues
10. Test query count in unit tests to catch N+1 regressions early
```

---
