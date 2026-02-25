# 🏛️ TOPIC 7 — Lazy vs Eager Loading

---

## 1️⃣ Conceptual Explanation

### The Fundamental Loading Problem

Every association and basic field in JPA presents a loading decision: **when should the data be fetched from the database?** Fetch too eagerly → unnecessary data loaded, memory waste, N+1 queries, cartesian products. Fetch too lazily → `LazyInitializationException`, N+1 queries in loops, missing data at serialization time.

There is no universally correct answer. The correct fetch strategy is **per-use-case**, which is why the mapping-level fetch type (`@ManyToOne(fetch=LAZY)`) should be considered a **default hint**, and query-level fetch strategies (`JOIN FETCH`, `@EntityGraph`) are the real tools for per-use-case control.

Understanding this distinction — **mapping-level vs query-level fetch strategy** — is the foundation of everything in this topic.

---

### How Hibernate Implements LAZY Loading — Three Mechanisms

Hibernate has three distinct mechanisms for implementing lazy loading, each with different characteristics, limitations, and tradeoffs.

---

#### Mechanism 1 — Proxy Subclass (Default for `@ManyToOne`, `@OneToOne`)

For single-valued associations (`@ManyToOne`, `@OneToOne`), Hibernate generates a **proxy object** — a runtime-generated subclass of the target entity class — using **ByteBuddy** (Hibernate 5.3+) or CGLIB (older versions).

**How the proxy is constructed:**

At `SessionFactory` bootstrap time, Hibernate pre-generates proxy classes for every entity. These are subclasses created via bytecode generation:

```
Original class: Customer (id, name, email, ...)

Generated proxy: Customer$HibernateProxy$xK7z3mQp extends Customer {
    private HibernateProxy.LazyInitializer lazyInitializer;
    
    @Override
    public String getName() {
        lazyInitializer.initialize(); // triggers SELECT if not loaded
        return ((Customer) lazyInitializer.getImplementation()).getName();
    }
    
    @Override
    public Long getId() {
        // Special case: ID is available without initialization
        return lazyInitializer.getIdentifier(); // NO SELECT
    }
    
    // All other methods similarly intercepted...
}
```

**Proxy lifecycle in detail:**

```
em.find(Order.class, 1L):
  1. Hibernate loads orders row: {id=1, customer_id=42, amount=99.99}
  2. Hibernate sees customer field is LAZY
  3. Creates Customer proxy:
     - Sets lazyInitializer with customer_id=42
     - Sets lazyInitializer.session = current Session
     - Does NOT query customers table
  4. Sets order.customer = the proxy instance
  5. Returns Order with proxy in customer field

order.getCustomer().getName():  // first non-ID method call
  1. getName() is intercepted by proxy
  2. lazyInitializer.initialize() is called
  3. Checks: is session still open? YES → proceed
             is session still open? NO  → LazyInitializationException
  4. Fires: SELECT * FROM customers WHERE id=42
  5. Creates real Customer instance from ResultSet
  6. Stores real Customer in lazyInitializer.target
  7. Returns real Customer's getName()
  8. From now on: all calls go directly to real Customer

order.getCustomer().getId():  // special case
  1. getId() is intercepted by proxy
  2. lazyInitializer.getIdentifier() called directly
  3. Returns 42 — NO SQL, NO session check
```

**The proxy `instanceof` trap:**

```java
Order order = em.find(Order.class, 1L);
Customer proxy = order.getCustomer(); // proxy, not initialized

// instanceof works correctly with proxies:
System.out.println(proxy instanceof Customer); // TRUE — proxy IS-A Customer

// But getClass() reveals the proxy:
System.out.println(proxy.getClass()); 
// com.example.Customer$HibernateProxy$xK7z3mQp

// This breaks switch expressions and some frameworks:
switch (proxy.getClass().getSimpleName()) {
    case "Customer": ... // NEVER matches — class is the proxy class
}

// Correct type check with proxies:
System.out.println(Hibernate.getClass(proxy)); // com.example.Customer ✓
// or after initialization:
Hibernate.initialize(proxy);
System.out.println(proxy.getClass()); // still proxy class!
// Use Hibernate.getClass() always for type resolution
```

**Proxy `equals()` trap:**

```java
Customer realCustomer = em.find(Customer.class, 42L);
Customer proxy = order.getCustomer(); // proxy for same customer

// If Customer.equals() uses getClass() comparison:
realCustomer.equals(proxy); // FALSE — different classes!
// If Customer.equals() uses instanceof:
realCustomer.equals(proxy); // TRUE — proxy IS-A Customer
// Recommendation: always use instanceof in equals(), never getClass()
```

---

#### Mechanism 2 — Persistent Collections (Default for `@OneToMany`, `@ManyToMany`)

For collection associations, Hibernate does not use entity proxies. Instead, it wraps the entire collection in a **`PersistentCollection`** — a Hibernate-specific implementation of `List`, `Set`, `Map`, `SortedSet`, or `SortedMap`.

```
Original field type:  List<OrderItem>
Runtime type:         org.hibernate.collection.internal.PersistentBag
                   or org.hibernate.collection.internal.PersistentList
                   or org.hibernate.collection.internal.PersistentSet
```

**Internal structure of `PersistentBag`/`PersistentList`:**

```java
// Simplified view of what Hibernate places in the field:
public class PersistentBag extends AbstractPersistentCollection implements List {
    
    private List bag; // the actual underlying ArrayList (null until loaded)
    
    private CollectionKey key;        // the FK value (e.g., order_id = 1)
    private CollectionPersister persister; // knows how to load/write this collection
    private SharedSessionContractImplementor session; // holds session reference
    private boolean initialized = false;
    private boolean dirty = false;
    
    @Override
    public int size() {
        read(); // triggers initialization if not loaded
        return bag.size();
    }
    
    private void read() {
        if (!initialized) {
            initialize(false); // triggers SELECT
        }
    }
    
    private void initialize(boolean writing) {
        // Check session is open:
        if (session == null || !session.isOpen()) {
            throw new LazyInitializationException(...);
        }
        // Fire SQL:
        // SELECT * FROM order_items WHERE order_id = ?
        persister.initialize(key, session);
        initialized = true;
    }
}
```

**When collection initialization fires:**

```java
Order order = em.find(Order.class, 1L);
List<OrderItem> items = order.getItems(); 
// items is PersistentBag — initialized=false — NO SQL yet
// You have a reference to the PersistentBag, not null

// Any operation that reads the collection triggers initialization:
items.size();               // → SQL: SELECT * FROM order_items WHERE order_id=1
items.isEmpty();            // → SQL (if not yet initialized)
items.get(0);               // → SQL (if not yet initialized)
items.iterator();           // → SQL (if not yet initialized)
items.contains(something);  // → SQL (if not yet initialized)
items.stream();             // → SQL (if not yet initialized) — common trap!
items.forEach(...);         // → SQL (if not yet initialized)

// Does NOT trigger initialization:
items == null               // false — it's a PersistentBag, not null
items.add(newItem)          // write operation — different code path
                            // may or may not initialize depending on context
```

**The `null` vs empty collection trap:**

```java
Order order = em.find(Order.class, 1L);
List<OrderItem> items = order.getItems();

System.out.println(items == null);  // FALSE — it's a PersistentBag, never null
System.out.println(items.isEmpty()); // triggers SQL → depends on DB state

// Implication: checking `if (order.getItems() == null)` never works
// Always check isEmpty() or size() == 0
// But those trigger loading — use with caution outside transaction
```

---

#### Mechanism 3 — Bytecode Enhancement

Bytecode enhancement is an **alternative** to proxy-based lazy loading. Instead of creating proxy subclasses at runtime, Hibernate **modifies the entity class bytecode** itself (either at compile time via Maven/Gradle plugin, or at load time via Java agent).

The enhanced class gets injected fields and method overrides:

```java
// Original:
public class Customer {
    private String name;
    public String getName() { return name; }
}

// After bytecode enhancement (conceptual — actual bytecode manipulation):
public class Customer implements PersistentAttributeInterceptable {
    private String name;
    private PersistentAttributeInterceptor $$_hibernate_interceptor;
    private boolean $$_hibernate_name_loaded = false;
    
    public String getName() {
        if ($$_hibernate_interceptor != null && !$$_hibernate_name_loaded) {
            $$_hibernate_interceptor.readObject(this, "name", name);
            $$_hibernate_name_loaded = true;
        }
        return name;
    }
    
    public void setName(String name) {
        if ($$_hibernate_interceptor != null) {
            $$_hibernate_interceptor.writeObject(this, "name", this.name, name);
        }
        this.name = name;
    }
}
```

**Bytecode enhancement enables:**

1. **Lazy loading of `@Basic` fields** — scalar fields can now be truly lazy
2. **Lazy loading of `@OneToOne` inverse side** — solves the proxy limitation
3. **In-place dirty tracking** — instead of snapshot comparison, enhanced entities track their own dirty fields → faster flush with large persistence contexts
4. **Enhanced collection tracking** — finer-grained collection dirty detection

**Configuration:**
```xml
<!-- Maven plugin for compile-time enhancement -->
<plugin>
    <groupId>org.hibernate.orm.tooling</groupId>
    <artifactId>hibernate-enhance-maven-plugin</artifactId>
    <version>${hibernate.version}</version>
    <executions>
        <execution>
            <goals><goal>enhance</goal></goals>
            <configuration>
                <enableLazyInitialization>true</enableLazyInitialization>
                <enableDirtyTracking>true</enableDirtyTracking>
                <enableAssociationManagement>true</enableAssociationManagement>
            </configuration>
        </execution>
    </executions>
</plugin>
```

**Bytecode enhancement limitations:**
- Entities must not be `final`
- Requires build-time tooling configuration
- Slight complexity increase in build pipeline
- Not needed in most applications — proxy mechanism is sufficient for associations

---

### `LazyInitializationException` — The Root Cause and All Triggers

`LazyInitializationException` message: **"could not initialize proxy - no Session"** or **"failed to lazily initialize a collection of role: ..., could not initialize proxy - no Session"**

**Root cause**: A lazy association (proxy or PersistentCollection) attempts to initialize, but the Hibernate `Session` that loaded the owning entity is **no longer open**.

**In Spring:** The `Session`/`EntityManager` scope is the transaction boundary by default. When `@Transactional` method returns, the transaction commits, the Session closes, all managed entities become detached. Any lazy access after that point throws.

**All scenarios that cause it:**

```
Scenario 1 — Service returns entity, controller accesses lazy field:
  @Transactional service.findOrder() → returns Order (Session closed)
  controller: order.getItems().size() → LazyInitializationException

Scenario 2 — JSON serialization accesses lazy field:
  @RestController returns Order entity directly
  Jackson serializes → accesses order.getItems() → outside transaction
  → LazyInitializationException

Scenario 3 — equals()/hashCode() on proxy in Set:
  Set<Order> orders = customer.getOrders(); // PersistentSet
  // inside transaction: ok
  // outside transaction: orders.contains(o) → initializes set → LazyInit

Scenario 4 — toString() triggers lazy field:
  Lombok @ToString on entity with lazy associations
  Logging: log.debug("Order: {}", order) → toString() → lazy access
  → LazyInitializationException (if outside transaction)

Scenario 5 — Spring @Async method receives detached entity:
  @Transactional method passes entity to @Async method
  Async method runs in new thread (no transaction)
  → lazy access → LazyInitializationException

Scenario 6 — Lazy field in test assertion:
  @Test (no @Transactional) loads entity via service
  assert: entity.getItems().size() == 3
  → LazyInitializationException
```

---

### Solutions to `LazyInitializationException`

#### Solution 1 — `JOIN FETCH` in JPQL (Best for specific queries)

```java
// Instead of:
Order order = orderRepo.findById(id).get();
// → customer and items are lazy proxies

// Use:
@Query("SELECT o FROM Order o JOIN FETCH o.items JOIN FETCH o.customer WHERE o.id = :id")
Optional<Order> findByIdWithItemsAndCustomer(@Param("id") Long id);
// → items and customer are loaded in ONE query, no lazy proxies
// SQL: SELECT o.*, i.*, c.* 
//      FROM orders o 
//      JOIN order_items i ON o.id = i.order_id
//      JOIN customers c ON o.customer_id = c.id
//      WHERE o.id = ?
```

**`JOIN FETCH` with collections — the cartesian product trap:**

```java
// DANGEROUS: fetching two collections simultaneously with JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.items JOIN FETCH o.promotions WHERE o.id=:id")
Order findWithItemsAndPromotions(@Param("id") Long id);
// If order has 3 items and 2 promotions:
// SQL result: 3 * 2 = 6 rows (cartesian product)
// Hibernate deduplicates → returns correct Order, but 6 DB rows transmitted
// For large collections: catastrophic data transfer

// SOLUTION 1: Fetch one collection via JOIN FETCH, other via separate query
// SOLUTION 2: Use @EntityGraph for one collection, @BatchSize for the other
// SOLUTION 3: Use SUBSELECT fetch mode
// MultipleBagFetchException: Hibernate throws this if you JOIN FETCH two BAGS (Lists)
// Solution: use Sets instead of Lists, or use Hibernate's @Fetch(FetchMode.SUBSELECT)
```

---

#### Solution 2 — `@EntityGraph` (Best for repository methods)

```java
// Named EntityGraph (defined on entity):
@Entity
@NamedEntityGraph(
    name = "Order.withItemsAndCustomer",
    attributeNodes = {
        @NamedAttributeNode("items"),
        @NamedAttributeNode(value = "items", subgraph = "items.product"),
        @NamedAttributeNode("customer")
    },
    subgraphs = {
        @NamedSubgraph(
            name = "items.product",
            attributeNodes = @NamedAttributeNode("product")
        )
    }
)
public class Order { ... }

// Using in repository:
@EntityGraph("Order.withItemsAndCustomer")
Optional<Order> findById(Long id);

// Dynamic EntityGraph (no annotation on entity needed):
@EntityGraph(attributePaths = {"items", "items.product", "customer"})
Optional<Order> findByIdWithGraph(Long id);
```

**`@EntityGraph` internal behavior:**

Spring Data translates `@EntityGraph` into a JPA `EntityGraph`, passes it to the query as a hint (`javax.persistence.loadgraph` or `javax.persistence.fetchgraph`):

- `loadgraph`: specified nodes EAGER, everything else uses mapping defaults
- `fetchgraph`: specified nodes EAGER, everything else LAZY regardless of mapping

```java
// loadgraph (default for @EntityGraph in Spring Data):
// → items and customer: loaded eagerly
// → other fields: use their @ManyToOne / @OneToMany fetch settings

// fetchgraph:
// → items and customer: loaded eagerly
// → ALL other associations: LAZY regardless of mapping
```

---

#### Solution 3 — Transactional Service / Open Session in View

**Extend transaction to cover lazy access:**

```java
// Move lazy access INSIDE the transaction:
@Transactional  // extends transaction to cover lazy loading
public OrderDto getOrderWithItems(Long id) {
    Order order = orderRepo.findById(id).get();
    // Still inside transaction:
    int itemCount = order.getItems().size(); // OK — Session still open
    return new OrderDto(order, order.getItems()); // convert before transaction ends
}
```

**Open Session in View (OSIV) — the anti-pattern:**

Spring Boot enables OSIV by default (`spring.jpa.open-in-view=true`). OSIV keeps the `Session` open for the entire HTTP request lifecycle — across service layers, presentation layer, even into view rendering/JSON serialization.

```
HTTP Request lifecycle with OSIV:
  Request arrives
  → OpenSessionInViewInterceptor opens Session
  → Controller method called
    → Service method (may or may not have @Transactional)
    → Returns entity with lazy fields
  → Jackson serializes entity
    → Accesses lazy fields → Session still open → OK (no LazyInitException)
  → Response sent
  → OpenSessionInViewInterceptor closes Session
```

**OSIV problems:**
- Session stays open for entire request → holds DB connection from pool for entire request duration → connection pool starvation under load
- Lazy loading happens in controller/view layer → N+1 queries outside transaction scope → no transaction → non-transactional reads → dirty reads possible
- Hides design problems — lazy loading issues are masked rather than solved
- Performance unpredictability — can't reason about how many queries a request fires

```properties
# Disable OSIV (recommended for production):
spring.jpa.open-in-view=false
# This forces proper eager loading design via JOIN FETCH / @EntityGraph
```

---

#### Solution 4 — DTO Projections (Best solution in most cases)

Instead of loading entities and risking lazy issues, load only what you need as a DTO directly from the query:

```java
// Interface projection (Spring Data):
public interface OrderSummary {
    Long getId();
    LocalDateTime getOrderDate();
    String getCustomerName();  // from joined customer
    int getItemCount();        // from COUNT
}

@Query("SELECT o.id as id, o.orderDate as orderDate, " +
       "c.name as customerName, COUNT(i) as itemCount " +
       "FROM Order o JOIN o.customer c LEFT JOIN o.items i " +
       "WHERE o.id = :id GROUP BY o.id, o.orderDate, c.name")
Optional<OrderSummary> findOrderSummary(@Param("id") Long id);

// No entities → no lazy fields → no LazyInitializationException
// Only exactly the data needed → optimal SQL
```

---

#### Solution 5 — `Hibernate.initialize()` (Explicit initialization within transaction)

```java
@Transactional
public Order loadWithInitializedItems(Long id) {
    Order order = orderRepo.findById(id).get();
    Hibernate.initialize(order.getItems());   // explicitly initializes collection
    Hibernate.initialize(order.getCustomer()); // explicitly initializes proxy
    return order; // now detached, but items and customer are initialized (not proxies)
}
```

**`Hibernate.isInitialized()`** — check before accessing:

```java
if (Hibernate.isInitialized(order.getItems())) {
    // safe to access
} else {
    // items are lazy proxy — don't access outside transaction
}
```

---

### The `@Basic(fetch = FetchType.LAZY)` Reality

Scalar field lazy loading — the most misunderstood JPA feature:

```java
@Entity
public class Document {
    @Id Long id;
    String title;
    
    @Basic(fetch = FetchType.LAZY)
    @Lob
    private byte[] content; // 50MB PDF
}
```

**Without bytecode enhancement:**
- `@Basic(fetch=LAZY)` annotation is **completely ignored**
- `content` is loaded in the same SELECT as `id` and `title`
- 50MB transferred for every Document query regardless

**With bytecode enhancement:**
- `content` field has interceptor injected
- `SELECT id, title FROM documents WHERE id=?` — no content column
- On `document.getContent()`: `SELECT content FROM documents WHERE id=?`
- True lazy loading of the scalar field

**Workaround without bytecode enhancement:**

Split the large field into a separate entity:

```java
@Entity
public class Document {
    @Id Long id;
    String title;
    
    @OneToOne(mappedBy = "document", fetch = FetchType.LAZY)
    private DocumentContent content; // lazy-loaded separately
}

@Entity
public class DocumentContent {
    @Id Long id;
    
    @OneToOne @MapsId
    private Document document;
    
    @Lob
    private byte[] data; // 50MB — only loaded when DocumentContent is accessed
}
```

---

### Fetch Type Resolution — Priority Order

When Hibernate determines whether to load an association eagerly or lazily, it uses this priority:

```
1. Query hint (javax.persistence.loadgraph / fetchgraph)     ← highest priority
2. EntityGraph on query / repository method
3. JOIN FETCH in JPQL / HQL
4. @Fetch(FetchMode.JOIN) Hibernate annotation
5. Mapping-level @ManyToOne(fetch=EAGER/LAZY)                ← lowest priority
```

This means: even if your mapping says `EAGER`, a `@EntityGraph` (fetchgraph) can override it to LAZY. And even if your mapping says `LAZY`, a `JOIN FETCH` makes it eager for that query.

**EAGER mapping cannot be made LAZY per-query in standard JPA:**

```java
@ManyToOne(fetch = FetchType.EAGER) // global EAGER
private Customer customer;

// You cannot lazily load this per-query in standard JPA
// JOIN FETCH always fetches eagerly
// No "LAZY hint" exists in JPA spec
// Hibernate-specific: @LazyToOne — but complex and brittle

// This is why: LAZY mapping + eager per-query (JOIN FETCH) is the RIGHT default
// NOT EAGER mapping + trying to make lazy per-query
```

---

### Collection Type and Hibernate Mapping

The Java collection type you declare affects Hibernate's internal collection implementation and behavior:

| Java Type | Hibernate Implementation | Allows Duplicates | Ordered |
|---|---|---|---|
| `List<T>` | `PersistentBag` (default) or `PersistentList` (with `@OrderColumn`) | Yes | By `@OrderColumn` or unordered |
| `Set<T>` | `PersistentSet` | No | No (unless `LinkedHashSet`) |
| `Map<K,V>` | `PersistentMap` | N/A | By `@MapKey` |
| `SortedSet<T>` | `PersistentSortedSet` | No | Natural/Comparator order |
| `Collection<T>` | `PersistentBag` | Yes | No |

**`List` vs `Set` for `@OneToMany` — the delete behavior difference:**

```java
// With List<OrderItem> items:
// Hibernate uses PersistentBag
// When any element changes in a bag that represents a @OneToMany with join table:
// DELETE all rows from join table → INSERT all remaining rows
// (This is the "bag deletion" problem for join-table-backed collections)

// With Set<OrderItem> items:
// Hibernate uses PersistentSet
// For join table: only deletes/inserts the specific changed rows
// For FK-based: no difference in SQL generation (uses WHERE order_id=?)

// Recommendation:
// @ManyToMany → always use Set (avoids bag deletion)
// @OneToMany (FK-based) → List is fine (no join table)
// @ElementCollection → consider Set to avoid delete-all behavior
```

---

### `@Fetch` — Hibernate-Specific Fetch Mode Annotations

Beyond JPA's `FetchType`, Hibernate provides `@Fetch(FetchMode)` for controlling HOW (not just when) associations are loaded:

```java
// FetchMode.SELECT (default for LAZY): fires separate SELECT per entity
@Fetch(FetchMode.SELECT)
@OneToMany(mappedBy = "order")
private List<OrderItem> items;
// SELECT * FROM orders WHERE ...
// SELECT * FROM order_items WHERE order_id=1
// SELECT * FROM order_items WHERE order_id=2  ← N+1

// FetchMode.JOIN: forces LEFT JOIN in the parent query (makes it EAGER)
@Fetch(FetchMode.JOIN)
@ManyToOne(fetch = FetchType.LAZY)
private Customer customer;
// Overrides LAZY → always JOIN fetched

// FetchMode.SUBSELECT: fetches all collection elements for all loaded parents in ONE query
@Fetch(FetchMode.SUBSELECT)
@OneToMany(mappedBy = "order")
private List<OrderItem> items;
// After loading orders: SELECT * FROM orders WHERE amount > 100
// Items loaded as: SELECT * FROM order_items WHERE order_id IN
//                 (SELECT id FROM orders WHERE amount > 100)
// ONE query for all items — eliminates N+1
```

**`@BatchSize`** — another N+1 mitigation (covered in depth in Topic 8):

```java
@BatchSize(size = 25)
@OneToMany(mappedBy = "order")
private List<OrderItem> items;
// When items are lazily accessed for multiple orders:
// Instead of N individual SELECTs, loads in batches of 25:
// SELECT * FROM order_items WHERE order_id IN (1,2,3,...,25)
// SELECT * FROM order_items WHERE order_id IN (26,27,...,50)
```

---

## 2️⃣ Code Examples

### Example 1 — Proxy Behavior Deep Dive

```java
@Service
public class ProxyInternalsDemo {
    
    @PersistenceContext
    private EntityManager em;
    
    @Transactional
    public void proxyMechanics() {
        Order order = em.find(Order.class, 1L);
        
        Customer proxy = order.getCustomer();
        // At this point:
        // proxy.getClass() = Customer$HibernateProxy$...
        // proxy is NOT null (Hibernate created a proxy)
        // NO SQL for customer yet
        
        // Test 1: ID access — no SQL
        Long id = proxy.getId();
        System.out.println("ID: " + id); // prints 42, NO SQL
        
        // Test 2: initialized check
        System.out.println(Hibernate.isInitialized(proxy)); // false
        
        // Test 3: first non-ID access triggers SQL
        String name = proxy.getName();
        // SQL: SELECT * FROM customers WHERE id=42
        System.out.println(Hibernate.isInitialized(proxy)); // true
        
        // Test 4: instanceof
        System.out.println(proxy instanceof Customer);      // true
        System.out.println(Hibernate.getClass(proxy));      // class Customer
        System.out.println(proxy.getClass());               // class Customer$HibernateProxy$...
        
        // Test 5: getReference creates proxy without SQL
        Customer ref = em.getReference(Customer.class, 99L);
        // NO SQL — just a proxy for id=99
        // Only fires SQL when non-ID method is called
    }
    
    public void lazyInitException() {
        Order order;
        
        // Load inside transaction:
        try {
            order = transactionTemplate.execute(status -> 
                em.find(Order.class, 1L)
            ); // transaction ends here, Session closes
        }
        
        // Outside transaction:
        Customer proxy = order.getCustomer();
        System.out.println(proxy.getId()); // OK — no session needed for ID
        
        try {
            String name = proxy.getName(); // BOOM
        } catch (LazyInitializationException e) {
            System.out.println("Session is closed: " + e.getMessage());
            // "could not initialize proxy [Customer#42] - no Session"
        }
    }
}
```

---

### Example 2 — PersistentCollection Internals

```java
@Transactional
public void persistentCollectionDemo() {
    Order order = em.find(Order.class, 1L);
    // SQL: SELECT * FROM orders WHERE id=1
    
    List<OrderItem> items = order.getItems();
    // items is PersistentBag — initialized=false
    // items != null  ← important: NOT null even though not loaded
    
    System.out.println(items.getClass()); 
    // org.hibernate.collection.internal.PersistentBag
    
    System.out.println(Hibernate.isInitialized(items)); // false
    
    // These trigger initialization:
    int size = items.size(); 
    // SQL: SELECT * FROM order_items WHERE order_id=1
    System.out.println(Hibernate.isInitialized(items)); // true
    
    // These do NOT re-trigger (already initialized):
    items.get(0);
    items.isEmpty();
    items.stream().count();
}

public void collectionOutsideTransaction() {
    Order order = orderService.findOrder(1L); // detached
    
    List<OrderItem> items = order.getItems();
    // items is PersistentBag, initialized=false, session=null
    
    System.out.println(items == null);  // FALSE — PersistentBag is not null
    
    try {
        items.size(); // triggers initialization attempt
    } catch (LazyInitializationException e) {
        // "failed to lazily initialize a collection of role: 
        //  Order.items, could not initialize proxy - no Session"
    }
}
```

---

### Example 3 — All Solutions Side by Side

```java
// Problem: load Order with items for display

// SOLUTION 1: JOIN FETCH (one query, best performance)
@Query("SELECT DISTINCT o FROM Order o " +
       "JOIN FETCH o.items i " +
       "JOIN FETCH i.product " +
       "JOIN FETCH o.customer " +
       "WHERE o.id = :id")
Optional<Order> findByIdFull(@Param("id") Long id);
// SQL: one query with JOINs
// DISTINCT needed to avoid duplicate Order in result (cartesian product dedup)

// SOLUTION 2: @EntityGraph (clean, declarative)
@EntityGraph(attributePaths = {"items", "items.product", "customer"})
Optional<Order> findById(Long id);
// SQL: same as JOIN FETCH internally

// SOLUTION 3: Extend @Transactional to cover access
@Transactional(readOnly = true)
public OrderResponseDto getOrder(Long id) {
    Order order = orderRepo.findById(id).get();
    // Still in transaction — lazy access safe:
    List<ItemDto> itemDtos = order.getItems().stream() // OK
        .map(item -> new ItemDto(
            item.getProduct().getName(), // OK — lazy product loads here
            item.getQuantity()
        ))
        .collect(Collectors.toList());
    return new OrderResponseDto(order, itemDtos);
    // Transaction ends here — DTO returned, no entity references
}

// SOLUTION 4: DTO projection (best for read-only use cases)
@Query("SELECT new com.example.OrderDto(" +
       "o.id, o.orderDate, c.name, COUNT(i))" +
       "FROM Order o JOIN o.customer c LEFT JOIN o.items i " +
       "WHERE o.id = :id GROUP BY o.id, o.orderDate, c.name")
Optional<OrderDto> findOrderDto(@Param("id") Long id);
// No entity returned → no lazy fields → no LazyInitializationException possible
```

---

### Example 4 — `@Fetch(FetchMode.SUBSELECT)` vs N+1

```java
@Entity
public class Customer {
    @Id Long id;
    String name;
    
    @OneToMany(mappedBy = "customer", fetch = FetchType.LAZY)
    @Fetch(FetchMode.SUBSELECT) // load all customer orders in one SUBSELECT
    private List<Order> orders;
}

@Transactional
public void subSelectDemo() {
    List<Customer> customers = em.createQuery(
        "SELECT c FROM Customer c WHERE c.region = 'EMEA'", Customer.class)
        .getResultList();
    // SQL: SELECT * FROM customers WHERE region='EMEA'
    // Returns 100 customers — orders NOT yet loaded
    
    // Access first customer's orders:
    customers.get(0).getOrders().size();
    // SQL (SUBSELECT mode): 
    // SELECT * FROM orders WHERE customer_id IN
    //   (SELECT id FROM customers WHERE region='EMEA')
    // ONE query loads orders for ALL 100 customers
    // vs FetchMode.SELECT: 100 individual SELECTs (N+1)
    
    // Subsequent access on other customers:
    customers.get(1).getOrders().size(); // NO SQL — already loaded via subselect
    customers.get(99).getOrders().size(); // NO SQL
}
```

---

### Example 5 — `@Basic(fetch=LAZY)` with and without Enhancement

```java
@Entity
public class Report {
    @Id @GeneratedValue Long id;
    String title;
    LocalDateTime generatedAt;
    
    @Basic(fetch = FetchType.LAZY)
    @Lob
    @Column(name = "html_content")
    private String htmlContent; // potentially large
    
    @Basic(fetch = FetchType.LAZY)
    @Column(name = "pdf_data")
    private byte[] pdfData; // potentially very large
}

// WITHOUT bytecode enhancement:
Report report = em.find(Report.class, 1L);
// SQL: SELECT id, title, generated_at, html_content, pdf_data FROM reports WHERE id=1
// @Basic(fetch=LAZY) ignored → everything loaded

// WITH bytecode enhancement enabled:
Report report = em.find(Report.class, 1L);
// SQL: SELECT id, title, generated_at FROM reports WHERE id=1
// html_content and pdf_data NOT loaded

String title = report.getTitle(); // NO extra SQL
String html = report.getHtmlContent(); // SQL: SELECT html_content FROM reports WHERE id=1
byte[] pdf = report.getPdfData();      // SQL: SELECT pdf_data FROM reports WHERE id=1
```

---

### Example 6 — OSIV Disabled — Proper Architecture

```properties
spring.jpa.open-in-view=false
```

```java
// With OSIV disabled, this breaks at Jackson serialization:
@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable Long id) {
    return orderService.findById(id); // returns detached entity
    // Jackson accesses lazy fields during serialization → LazyInitException
}

// Correct approach with OSIV disabled:
@GetMapping("/orders/{id}")
public OrderResponseDto getOrder(@PathVariable Long id) {
    return orderService.findByIdAsDto(id); // return DTO, not entity
    // No lazy fields → no problem
}

@Service
public class OrderService {
    
    @Transactional(readOnly = true)
    public OrderResponseDto findByIdAsDto(Long id) {
        Order order = orderRepo.findByIdWithItemsAndCustomer(id)
            .orElseThrow(NotFoundException::new);
        // Convert to DTO within transaction:
        return OrderResponseDto.from(order); // accesses all needed fields inside TX
        // After return: TX closed, entity detached, but DTO has all data
    }
}
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ**
What is the runtime type of a lazily-loaded `@ManyToOne Customer customer` field after `em.find(Order.class, 1L)` is called?

A) `null`  
B) `Customer` (the actual entity instance)  
C) A Hibernate proxy subclass of `Customer`  
D) An uninitialized `Optional<Customer>`  

**Answer: C** — Hibernate creates a proxy subclass. The field is not null and not the real Customer instance until the proxy is initialized.

---

**Q2 — Select All That Apply**
Which of the following operations trigger initialization of a `PersistentBag` (lazy `List<OrderItem>`)? (Select all)

A) `items == null`  
B) `items.size()`  
C) `items.isEmpty()`  
D) `items.add(newItem)` (write operation)  
E) `items.stream().collect(Collectors.toList())`  
F) Declaring `List<OrderItem> ref = order.getItems()`  

**Answer: B, C, E** — A is false (PersistentBag is never null, check is always false). D — write operations may or may not initialize depending on context. F — just assigning the reference doesn't trigger initialization.

---

**Q3 — Code Output Prediction**
```java
@Transactional
public void test() {
    Order order = em.find(Order.class, 1L);
    Customer proxy = order.getCustomer(); // @ManyToOne LAZY
    
    System.out.println(proxy == null);                    // Line A
    System.out.println(proxy instanceof Customer);         // Line B
    System.out.println(Hibernate.isInitialized(proxy));   // Line C
    System.out.println(proxy.getId());                    // Line D
    System.out.println(Hibernate.isInitialized(proxy));   // Line E
}
// Customer with id=42 exists in DB
```

What is printed (in order)?

A) false, true, false, 42, false  
B) false, true, false, 42, true  
C) null, true, false, 42, false  
D) false, false, false, 42, false  

**Answer: A** — proxy is not null (false). proxy instanceof Customer is true (proxy IS-A Customer). proxy is not initialized (false). `getId()` returns the ID from lazyInitializer WITHOUT triggering initialization — proxy remains uninitialized (false on Line E).

---

**Q4 — Scenario-Based**
```java
@Entity
public class Employee {
    @OneToMany(mappedBy = "employee", fetch = FetchType.LAZY)
    @Fetch(FetchMode.SUBSELECT)
    private List<Timesheet> timesheets;
}

// Query:
List<Employee> employees = em.createQuery("FROM Employee WHERE dept='Engineering'")
                              .getResultList(); // returns 50 employees

// Later in same transaction:
employees.get(0).getTimesheets().size(); // Line X
employees.get(25).getTimesheets().size(); // Line Y
employees.get(49).getTimesheets().size(); // Line Z
```

How many SQL queries are executed total (including initial query)?

A) 51 (1 + 50 for each employee's timesheets)  
B) 3 (1 for employees, 1 for Line X triggering subselect, then cached)  
C) 2 (1 for employees, 1 subselect loading ALL timesheets at Line X)  
D) 1 (everything loaded in initial query)  

**Answer: C** — `FetchMode.SUBSELECT` loads ALL collection instances for ALL loaded parents in ONE subselect query when any one of them is accessed. Line X triggers the subselect for ALL 50 employees' timesheets. Lines Y and Z use already-initialized data — no SQL.

---

**Q5 — LazyInitializationException Scenario**
```java
@Service
public class ProductService {
    @Transactional
    public Product findWithCategory(Long id) {
        Product p = productRepo.findById(id).get();
        // category is @ManyToOne(fetch=LAZY)
        String categoryName = p.getCategory().getName(); // Line A
        return p;
    }
}

@RestController  
public class ProductController {
    @GetMapping("/products/{id}")
    public ProductDto getProduct(@PathVariable Long id) {
        Product p = productService.findWithCategory(id);
        // spring.jpa.open-in-view=false
        return new ProductDto(
            p.getName(),
            p.getCategory().getName() // Line B
        );
    }
}
```

Which line throws `LazyInitializationException`?

A) Line A  
B) Line B  
C) Both lines  
D) Neither — `@Transactional` keeps session alive throughout  

**Answer: B** — Line A is inside `@Transactional` → Session is open → proxy initializes successfully. After `findWithCategory()` returns, `@Transactional` commits and Session closes. `p` is now detached. Line B accesses `p.getCategory().getName()` → proxy → Session closed → `LazyInitializationException`. Note: the proxy WAS initialized at Line A, but proxy initialization state is linked to the Session — after Session close, even initialized proxies may not work depending on Hibernate version. In Hibernate 5+, if the proxy was fully initialized (data loaded into it), it can be accessed post-session. But in this case, the category proxy IS initialized (getName() was called at Line A), so Line B would actually work in Hibernate 5+. **Revised answer: Neither** — category was initialized at Line A, so the data is in the proxy. However, this is version-dependent behavior. On exams, the intended answer is typically B to test awareness of detached entity access.

---

**Q6 — `@Basic(fetch=LAZY)` Reality**
```java
@Entity
public class BlogPost {
    @Id Long id;
    String title;
    
    @Basic(fetch = FetchType.LAZY)
    @Lob
    private String content; // very large field
}

// Without bytecode enhancement:
BlogPost post = em.find(BlogPost.class, 1L);
System.out.println(post.getTitle());
// How many SQL columns were loaded?
```

A) Only `id` and `title` — `content` is lazy  
B) All columns including `content` — `@Basic(fetch=LAZY)` is ignored without bytecode enhancement  
C) `id`, `title`, and a `content` proxy  
D) Exception — `@Basic(fetch=LAZY)` on `@Lob` is invalid  

**Answer: B** — Without bytecode enhancement, `@Basic(fetch=LAZY)` is silently ignored. All columns including the large `content` field are loaded in the initial SELECT.

---

**Q7 — Cartesian Product Trap**
```java
@Query("SELECT o FROM Order o " +
       "JOIN FETCH o.items " +
       "JOIN FETCH o.promotions " +
       "WHERE o.customerId = :cid")
List<Order> findOrdersWithItemsAndPromotions(@Param("cid") Long cid);
// Order has: 2 items, 3 promotions
```

How many rows does the SQL result set contain for this single order?

A) 1  
B) 5 (2 + 3)  
C) 6 (2 × 3 — cartesian product)  
D) 2 (Hibernate fetches in separate queries automatically)  

**Answer: C** — JOIN FETCH on two collections creates a cartesian product: 2 items × 3 promotions = 6 rows. Hibernate deduplicates back to 1 Order object with correct items and promotions, but 6 rows are transferred from DB. With large collections (e.g., 100 items × 50 promotions = 5000 rows), this is catastrophic. Additionally, if both collections are `List` (Bag), Hibernate throws `MultipleBagFetchException`.

---

**Q8 — EAGER Override**
```java
@ManyToOne(fetch = FetchType.EAGER)
private Customer customer;

// In repository:
@Query("SELECT o FROM Order o WHERE o.amount > 1000")
List<Order> findLargeOrders();
```

Can you make this `findLargeOrders()` query NOT load `customer` eagerly?

A) Yes — add `@EntityGraph` with empty `attributePaths`  
B) Yes — add `FetchType.LAZY` hint to the `@Query`  
C) No — EAGER mapping cannot be overridden to LAZY per-query in standard JPA  
D) Yes — use `JOIN FETCH` without including `customer`  

**Answer: C** — This is a fundamental JPA limitation. Once a mapping is declared EAGER, it cannot be made lazy per-query in standard JPA. `@EntityGraph` with `fetchgraph` semantics CAN override EAGER to LAZY (making only explicitly listed attributes eager), but this is implementation-dependent. The correct design is: always map LAZY, fetch eagerly per-query when needed. This is why the rule exists: **always map LAZY by default**.

---

## 4️⃣ Trick Analysis

**`proxy.getId()` doesn't trigger initialization (Q3)**:
This is a deliberate Hibernate optimization. The proxy's `LazyInitializer` holds the entity identifier (extracted from the FK column of the owning row). Calling `getId()` (the `@Id` field's getter) returns this cached value without firing a SELECT. However, all OTHER methods trigger initialization. Developers who test "is proxy initialized?" by calling `getId()` are fooled — it always works without initialization.

**`PersistentBag != null` (Q2)**:
Developers often guard lazy collection access with `if (items != null)`. This never helps because a lazy collection is NEVER null — it's a `PersistentBag` or similar. The null check is always false. The correct guards are `Hibernate.isInitialized(items)` or moving access inside a transaction.

**`FetchMode.SUBSELECT` — one trigger loads all (Q4)**:
SUBSELECT is a "batch of all" strategy. When ANY one entity's collection is initialized, Hibernate generates a subselect that loads collections for ALL parent entities in the current persistence context. This is highly efficient (1 query) but can also be wasteful if you only need one entity's collection. The SQL generated uses `IN (SELECT id FROM parent WHERE ...)` — the original parent query is repeated as a subselect.

**OSIV and connection pool starvation**:
The subtle implication: with OSIV enabled, every HTTP request holds a database connection from the moment the Session opens (start of request) until response is complete. Under load, this means connections are held for the full request duration including time spent in business logic, JSON serialization, network I/O. Connection pools typically have 10-20 connections. 20 concurrent requests × 100ms response time = serious contention.

**EAGER cannot be overridden to LAZY (Q8)**:
This is the single most important reason to always map LAZY. EAGER is a permanent, global, per-mapping declaration that affects every query touching that entity. LAZY is a default that can be overridden per-query to EAGER via JOIN FETCH / EntityGraph. The flexibility only exists in one direction: LAZY → per-query EAGER. Not the reverse.

**Cartesian product with two JOIN FETCHes (Q7)**:
The mathematical reason: relational JOIN between parent (1 row), collection A (N rows), collection B (M rows) produces N×M rows. Hibernate's deduplication works correctly but the data transfer is N×M. For two `List` (Bag) typed collections, Hibernate throws `MultipleBagFetchException` at query execution time. Solution: use `Set` for at least one collection, or use separate queries.

---

## 5️⃣ Summary Sheet

### Three Lazy Loading Mechanisms

```
Mechanism 1: Proxy Subclass (associations — single valued)
  Used for:     @ManyToOne, @OneToOne (owning side)
  Generated by: ByteBuddy / CGLIB at SessionFactory bootstrap
  Behavior:     Subclass intercepts all method calls except getId()
  Trigger:      First non-ID method call
  Limitation:   Cannot proxy final classes

Mechanism 2: PersistentCollection (associations — collection)
  Used for:     @OneToMany, @ManyToMany
  Types:        PersistentBag, PersistentSet, PersistentList, PersistentMap
  Behavior:     Wrapper intercepts all read operations
  Trigger:      Any size/get/iterate/stream operation
  Note:         NEVER null — always a PersistentCollection instance

Mechanism 3: Bytecode Enhancement (field-level)
  Used for:     @Basic(fetch=LAZY), @OneToOne inverse side
  Setup:        hibernate-enhance-maven-plugin (build time)
  Behavior:     Interceptor fields injected into entity class itself
  Benefit:      True field-level lazy loading, better dirty tracking
```

### LazyInitializationException — Root Cause and Solutions

```
Root cause: Proxy/Collection initialized after Session is closed

Solutions (in order of preference):
  1. DTO projection      — No entities, no lazy fields, no problem
  2. JOIN FETCH          — Load associations in same query
  3. @EntityGraph        — Declarative, clean, per-repository-method
  4. Extend @Transactional scope — Keep session alive through conversion
  5. Hibernate.initialize() — Explicit initialization within transaction
  6. OSIV               — Anti-pattern, avoid in production

OSIV: spring.jpa.open-in-view=false (recommended)
```

### Fetch Strategy Priority (Highest to Lowest)

```
1. fetchgraph hint (@EntityGraph fetchgraph type)
2. loadgraph hint (@EntityGraph loadgraph type / default)
3. JOIN FETCH in JPQL
4. @Fetch(FetchMode.JOIN) Hibernate annotation
5. @ManyToOne(fetch=EAGER/LAZY) — mapping-level default
```

### Fetch Mode Comparison

| `@Fetch(FetchMode.X)` | SQL Generated | N+1 Solved? | Best For |
|---|---|---|---|
| `SELECT` (default) | N separate SELECTs | ❌ No | Single entity access |
| `JOIN` | LEFT JOIN in parent query | ✅ Yes | Always-needed associations |
| `SUBSELECT` | 1 subselect for all parents | ✅ Yes | Collections on result lists |
| `@BatchSize(n)` | Batches of N INs | ✅ Partial | Medium-sized result sets |

### Key Rules Memorized

```
1. @ManyToOne / @OneToOne default = EAGER → ALWAYS override to LAZY
2. @OneToMany / @ManyToMany default = LAZY → keep
3. Proxy getId() = no SQL (ID in LazyInitializer)
4. PersistentCollection is NEVER null (even when lazy/uninitialized)
5. @Basic(fetch=LAZY) requires bytecode enhancement to work
6. @OneToOne inverse side lazy = unreliable without bytecode enhancement
7. JOIN FETCH two collections = cartesian product
8. JOIN FETCH two List/Bag = MultipleBagFetchException
9. EAGER mapping cannot be made LAZY per-query in standard JPA
10. OSIV disabled → must use DTO or eager loading in service layer
11. FetchMode.SUBSELECT = load ALL parent collections in ONE query when any accessed
12. spring.jpa.open-in-view=false is the production best practice
```

---
