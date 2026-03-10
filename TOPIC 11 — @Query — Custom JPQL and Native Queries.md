# 🏛️ TOPIC 11 — `@Query` — Custom JPQL and Native Queries

---

## 1️⃣ Conceptual Explanation

### Why `@Query` Exists — The Limits of Derived Queries

Derived queries cover simple property-based filtering. The moment you need aggregations, subqueries, complex joins, GROUP BY, window functions, or any SQL not expressible as a flat predicate chain, you need `@Query`. It is the escape hatch from method-name grammar into full JPQL or native SQL.

`@Query` annotations are processed at **application startup** — the JPQL is parsed and validated against the entity metamodel before the first request arrives. Errors are caught immediately.

---

### JPQL vs Native Query — The Fundamental Difference

```
JPQL (@Query):
  - References entity class names and field names (Java names)
  - Dialect-independent: works on any JPA provider + DB
  - Goes through Hibernate's JPQL compiler → dialect-specific SQL
  - Spring Data maps ResultSet to entities automatically
  - Supports: @EntityGraph, Pageable, Sort, projections
  - Does NOT support: DB-specific functions, window functions,
                      JSONB, CTEs, LATERAL JOINs (unless dialect supports them)

Native (@Query(nativeQuery=true)):
  - References table names and column names (DB schema names)
  - DB-specific: tied to one database dialect
  - Bypasses JPQL compiler → sent directly to JDBC
  - Spring Data maps ResultSet via SqlResultSetMapping or projection interfaces
  - Supports: every SQL feature the DB provides
  - Does NOT support: automatic entity graph loading, TypedQuery<Entity> mapping
                      without explicit @SqlResultSetMapping
```

---

### JPQL `@Query` — Internal Execution Pipeline

```
Step 1: Startup — query compilation
  @Query("SELECT u FROM User u WHERE u.email = :email")
  → Spring Data creates SimpleJpaQuery
  → Validates JPQL string against JPA metamodel
  → If invalid entity/field name: QueryException at startup
  → Stores pre-parsed query AST

Step 2: Method invocation
  userRepo.findByEmail("alice@example.com")
  → QueryExecutorMethodInterceptor intercepts
  → Retrieves pre-compiled SimpleJpaQuery
  → Creates TypedQuery<User> from EntityManager:
       em.createQuery("SELECT u FROM User u WHERE u.email = :email", User.class)
  → Binds parameters:
       query.setParameter("email", "alice@example.com")
  → Executes: query.getResultList() or query.getSingleResult()

Step 3: Hibernate compiles JPQL to SQL
  JPQL: SELECT u FROM User u WHERE u.email = :email
  SQL:  SELECT u.id, u.email, u.first_name, u.last_name, u.created_at
        FROM users u WHERE u.email = ?

Step 4: ResultSet hydration
  Each row → entity instance in Persistence Context
  (or DTO constructor / interface proxy if projection)
```

---

### Named Parameters vs Positional Parameters

**Named parameters (`:name` syntax) — recommended:**

```java
@Query("SELECT u FROM User u WHERE u.firstName = :first AND u.lastName = :last")
List<User> findByFullName(@Param("first") String firstName,
                          @Param("last") String lastName);

// @Param("first") binds the method parameter to :first in JPQL
// @Param is REQUIRED for named parameters in @Query
// If @Param is omitted → ParameterBinding exception at execution
```

**Positional parameters (`?1`, `?2` syntax) — legacy, avoid:**

```java
@Query("SELECT u FROM User u WHERE u.firstName = ?1 AND u.lastName = ?2")
List<User> findByFullName(String firstName, String lastName);

// ?1 = first method parameter, ?2 = second
// Order-based — fragile if parameters are reordered
// @Param NOT used (not needed)
// Works but harder to read and maintain
```

**Critical rule: mix positional and named in native queries:**

```java
// WRONG — cannot mix in same query:
@Query(value = "SELECT * FROM users WHERE first_name = ?1 AND last_name = :last",
       nativeQuery = true)
// Throws: HibernateException — mixing named and positional

// RIGHT — use one style consistently:
@Query(value = "SELECT * FROM users WHERE first_name = :first AND last_name = :last",
       nativeQuery = true)
List<User> findByFullName(@Param("first") String f, @Param("last") String l);
```

---

### `@Modifying` — The Write Query Annotation

Any `@Query` that modifies data (UPDATE, DELETE, INSERT) requires `@Modifying`. Without it, Hibernate treats the query as a SELECT and the modification is silently ignored or throws.

```java
@Modifying
@Transactional
@Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :cutoff")
int deactivateInactiveUsers(@Param("cutoff") LocalDateTime cutoff);
// Returns: number of affected rows (int or void)
```

**`@Modifying` internals:**

```java
// Without @Modifying on an UPDATE/DELETE:
// Spring Data tries to call query.getResultList() → TypeMismatchException
// or returns incorrect result type

// With @Modifying:
// Spring Data calls query.executeUpdate() instead of getResultList()
// Returns int (affected row count)
```

**`@Modifying(clearAutomatically = true)` — the stale data trap:**

```java
@Modifying(clearAutomatically = true)
@Transactional
@Query("UPDATE Product p SET p.price = p.price * 1.10 WHERE p.category = :cat")
int applyPriceIncrease(@Param("cat") String category);

// The problem without clearAutomatically:
// 1. Load Product id=1, price=100 → entity in Persistence Context
// 2. Execute bulk UPDATE: price = 110 in DB
// 3. Access product.getPrice() → returns 100 (stale, from PC cache!)
// Hibernate PC is NOT updated by bulk operations — it bypasses entity lifecycle

// clearAutomatically = true:
// After executeUpdate(), calls em.clear()
// ALL entities in PC become detached
// Next access reloads from DB → fresh price 110
// Downside: all in-memory state is discarded — unsaved changes lost
```

**`@Modifying(flushAutomatically = true)` — write ordering:**

```java
@Modifying(flushAutomatically = true)
@Transactional
@Query("UPDATE User u SET u.score = u.score + 10 WHERE u.department = :dept")
int boostDepartmentScore(@Param("dept") String dept);

// Without flushAutomatically:
// 1. user.setScore(50) → pending UPDATE in ActionQueue
// 2. @Modifying query executes: UPDATE WHERE department='Engineering'
//    → This bulk UPDATE operates on DB state, not in-memory state
//    → If user's pending UPDATE hasn't flushed yet → wrong base value
// 3. flush() → user UPDATE executes with score=50 (overwrites bulk update)

// flushAutomatically = true:
// Before executeUpdate(), calls em.flush()
// All pending ActionQueue changes written to DB first
// Then bulk UPDATE operates on consistent DB state
// Order: em.flush() → bulk UPDATE → (clearAutomatically clears PC if set)
```

**Combined `flushAutomatically + clearAutomatically`:**

```java
@Modifying(flushAutomatically = true, clearAutomatically = true)
@Transactional
@Query("UPDATE Account a SET a.balance = a.balance - :amount WHERE a.id = :id")
int debit(@Param("id") Long id, @Param("amount") BigDecimal amount);

// Execution sequence:
// 1. em.flush() → writes all pending changes to DB
// 2. executeUpdate() → runs the bulk UPDATE
// 3. em.clear() → clears PC to prevent stale data
// 4. Next access to any entity → fresh SELECT from DB
```

---

### SpEL Expressions in `@Query`

Spring Data supports Spring Expression Language (SpEL) in `@Query` values via the `#{...}` syntax:

**`#{#entityName}` — the most important SpEL expression:**

```java
// Without SpEL — hardcoded entity name:
@Query("SELECT p FROM Product p WHERE p.active = true")
List<Product> findAllActive();

// With SpEL — entity name resolved from repository's T type:
@Query("SELECT e FROM #{#entityName} e WHERE e.active = true")
List<T> findAllActive(); // works in @NoRepositoryBean base interfaces!

// Use case: base repository interface shared across entity types:
@NoRepositoryBean
public interface ActiveRepository<T, ID> extends JpaRepository<T, ID> {
    
    @Query("SELECT e FROM #{#entityName} e WHERE e.active = true")
    List<T> findAllActive();
    
    @Query("SELECT COUNT(e) FROM #{#entityName} e WHERE e.active = true")
    long countActive();
}

// Both Product and User repositories get findAllActive() with correct entity name:
public interface ProductRepository extends ActiveRepository<Product, Long> {}
public interface UserRepository extends ActiveRepository<User, Long> {}
// ProductRepository: FROM Product e WHERE e.active = true
// UserRepository:    FROM User e WHERE e.active = true
```

**`#{#entityName}` with `@Entity(name=...)`:**

```java
@Entity(name = "SystemUser")  // custom JPQL name
public class User { ... }

@Query("SELECT e FROM #{#entityName} e WHERE e.active = true")
// #{#entityName} resolves to "SystemUser" (from @Entity(name)), not "User"
// Generated JPQL: SELECT e FROM SystemUser e WHERE e.active = true
```

**SpEL with `@Param` values:**

```java
@Query("SELECT u FROM User u WHERE u.role IN :#{#roles}")
List<User> findByRoles(@Param("roles") List<String> roles);
// :#{#roles} accesses the @Param named 'roles'
// Allows SpEL manipulation: :#{#name.toLowerCase()} etc.

// SpEL for conditional parameter manipulation:
@Query("SELECT u FROM User u WHERE " +
       "(:#{#name} IS NULL OR u.name LIKE %:#{#name}%) AND " +
       "(:#{#active} IS NULL OR u.active = :#{#active})")
List<User> search(@Param("name") String name, @Param("active") Boolean active);
// Dynamic query simulation: null parameters effectively skip that condition
```

---

### Native Queries — `nativeQuery = true`

```java
@Query(value = "SELECT * FROM users WHERE email = :email", nativeQuery = true)
Optional<User> findByEmailNative(@Param("email") String email);
```

**Native query result mapping — three approaches:**

**Approach 1: Entity mapping (simple case)**

```java
@Query(value = "SELECT * FROM users WHERE status = :status", nativeQuery = true)
List<User> findByStatusNative(@Param("status") String status);
// Works ONLY if:
// - All non-nullable entity columns are present in SELECT *
// - Column names match what Hibernate expects (via naming strategy)
// - Entity is not abstract
```

**Approach 2: Interface projection (most practical)**

```java
public interface UserSummary {
    Long getId();
    String getEmail();
    String getFirstName();
}

@Query(value = "SELECT u.id, u.email, u.first_name FROM users u WHERE u.status = :s",
       nativeQuery = true)
List<UserSummary> findSummaryByStatus(@Param("s") String status);
// Spring Data creates proxy implementing UserSummary
// Column names in SELECT must match getter names (after snake_case conversion):
// u.first_name → getter getFirstName() (Spring Data converts automatically)
```

**Approach 3: `@SqlResultSetMapping` (full control)**

```java
@SqlResultSetMapping(
    name = "UserSummaryMapping",
    classes = @ConstructorResult(
        targetClass = UserSummaryDto.class,
        columns = {
            @ColumnResult(name = "id", type = Long.class),
            @ColumnResult(name = "email"),
            @ColumnResult(name = "user_name")
        }
    )
)
@NamedNativeQuery(
    name = "User.findSummaries",
    query = "SELECT u.id, u.email, u.first_name as user_name FROM users u",
    resultSetMapping = "UserSummaryMapping"
)
@Entity
public class User { ... }
```

**Native query pagination:**

```java
@Query(value = "SELECT * FROM users WHERE status = :s ORDER BY created_at DESC",
       countQuery = "SELECT COUNT(*) FROM users WHERE status = :s",
       nativeQuery = true)
Page<User> findByStatusPaged(@Param("s") String status, Pageable pageable);

// countQuery is REQUIRED for Page<T> with native queries
// Without countQuery: Spring Data tries to auto-generate count query from the native SQL
// Auto-generation wraps in: SELECT COUNT(*) FROM (...) which fails on many DBs
// Always provide explicit countQuery for native Page<T> queries
```

**Native query with `Sort` — the `NativeQuery` limitation:**

```java
// DOES NOT WORK — Sort not supported for native queries:
@Query(value = "SELECT * FROM users WHERE status = :s", nativeQuery = true)
List<User> findByStatus(@Param("s") String status, Sort sort);
// Throws: InvalidJpaQueryMethodException at startup

// Workaround: embed ORDER BY in the query string directly
@Query(value = "SELECT * FROM users WHERE status = :s ORDER BY last_name ASC",
       nativeQuery = true)
List<User> findByStatusSorted(@Param("s") String status);
```

---

### `@Query` with Projections

`@Query` can return interface or DTO projections just like derived queries:

**Interface projection with JPQL:**

```java
public interface OrderSummary {
    Long getId();
    LocalDateTime getOrderDate();
    String getCustomerName(); // from JOIN
    BigDecimal getTotal();
}

@Query("SELECT o.id as id, o.orderDate as orderDate, " +
       "c.name as customerName, o.total as total " +
       "FROM Order o JOIN o.customer c WHERE o.status = :status")
List<OrderSummary> findOrderSummaries(@Param("status") OrderStatus status);

// Spring Data generates an interface proxy implementing OrderSummary
// Each alias in JPQL SELECT must match projection getter name (camelCase)
// 'as customerName' → getCustomerName() in interface
```

**Constructor DTO projection:**

```java
public record OrderDto(Long id, LocalDateTime orderDate, 
                       String customerName, BigDecimal total) {}

@Query("SELECT new com.example.OrderDto(o.id, o.orderDate, c.name, o.total) " +
       "FROM Order o JOIN o.customer c WHERE o.status = :status")
List<OrderDto> findOrderDtos(@Param("status") OrderStatus status);

// Full qualified class name required in JPQL
// Constructor must match exactly: parameter count, types, order
// No entity lifecycle overhead — pure data transfer
```

---

### `@Query` with `Pageable` and `Sort`

```java
// JPQL + Pageable — works cleanly:
@Query("SELECT u FROM User u WHERE u.active = true")
Page<User> findAllActive(Pageable pageable);
// Spring Data appends ORDER BY from Pageable, wraps in count query automatically
// Count query auto-generated: SELECT COUNT(u) FROM User u WHERE u.active = true

// JPQL + Pageable with custom count query:
@Query(value = "SELECT u FROM User u LEFT JOIN FETCH u.roles WHERE u.active = true",
       countQuery = "SELECT COUNT(u) FROM User u WHERE u.active = true")
Page<User> findActiveWithRoles(Pageable pageable);
// Custom countQuery avoids LEFT JOIN FETCH in count (which would be inefficient)

// JPQL + Sort (without pagination):
@Query("SELECT u FROM User u WHERE u.department = :dept")
List<User> findByDepartment(@Param("dept") String dept, Sort sort);
// Sort is applied dynamically
// Caller: Sort.by("lastName").ascending().and(Sort.by("firstName"))

// Safe sort with JpaSort (prevents injection):
@Query("SELECT u FROM User u WHERE u.dept = :dept")
List<User> findByDept(@Param("dept") String dept, Sort sort);
// Use: JpaSort.unsafe("LENGTH(u.lastName)") for function-based sort
// Unsafe because it's not validated against metamodel — use carefully
```

---

### `@Query` Validation at Startup

JPQL queries in `@Query` are validated at startup:

```java
// This fails at startup — 'User' entity has no 'username' field:
@Query("SELECT u FROM User u WHERE u.username = :name")
// QueryException: could not resolve property: username of: User

// This fails at startup — unknown entity 'Account':
@Query("SELECT a FROM Account a WHERE a.balance > 0")
// IllegalArgumentException: Unknown entity: Account

// This does NOT fail at startup — nativeQuery=true bypasses JPQL validation:
@Query(value = "SELECT * FROM non_existent_table", nativeQuery = true)
// Fails only at execution time — native SQL is not validated at startup
```

---

### Complete `@Query` Scenarios — Advanced Patterns

**Pattern 1: Soft delete + auditing query**

```java
@Query("SELECT e FROM #{#entityName} e WHERE e.deletedAt IS NULL")
List<T> findAllNotDeleted();

@Modifying
@Transactional
@Query("UPDATE #{#entityName} e SET e.deletedAt = CURRENT_TIMESTAMP, " +
       "e.deletedBy = :user WHERE e.id = :id AND e.deletedAt IS NULL")
int softDelete(@Param("id") Long id, @Param("user") String user);
```

**Pattern 2: Bulk status update with return count**

```java
@Modifying(flushAutomatically = true, clearAutomatically = true)
@Transactional
@Query("UPDATE Order o SET o.status = :newStatus, o.updatedAt = CURRENT_TIMESTAMP " +
       "WHERE o.status = :oldStatus AND o.createdAt < :before")
int bulkStatusTransition(
    @Param("oldStatus") OrderStatus oldStatus,
    @Param("newStatus") OrderStatus newStatus,
    @Param("before") LocalDateTime before);
```

**Pattern 3: Exists check via JPQL (more efficient than loading entity)**

```java
@Query("SELECT CASE WHEN COUNT(u) > 0 THEN true ELSE false END " +
       "FROM User u WHERE u.email = :email AND u.active = true")
boolean existsActiveByEmail(@Param("email") String email);
// More explicit than the derived existsByEmailAndActive — shows intent clearly
```

**Pattern 4: Aggregate query returning scalar value**

```java
@Query("SELECT AVG(o.total) FROM Order o WHERE o.customer.id = :customerId")
BigDecimal findAverageOrderTotal(@Param("customerId") Long customerId);

@Query("SELECT MAX(p.price) FROM Product p WHERE p.category = :cat")
Optional<BigDecimal> findMaxPriceByCategory(@Param("cat") String category);
// Optional wraps scalar result to handle case where no products exist
```

**Pattern 5: Native query with DB-specific feature (PostgreSQL JSONB)**

```java
@Query(value = "SELECT * FROM events WHERE metadata @> :filter::jsonb",
       nativeQuery = true)
List<Event> findByJsonbFilter(@Param("filter") String jsonFilter);
// :filter::jsonb = PostgreSQL JSONB cast syntax
// Impossible in JPQL — requires native query
```

---

## 2️⃣ Code Examples

### Example 1 — Named vs Positional Parameters + Startup Validation

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Named parameters — RECOMMENDED:
    @Query("SELECT u FROM User u WHERE u.email = :email AND u.active = :active")
    Optional<User> findByEmailAndActive(
        @Param("email") String email,
        @Param("active") boolean active);

    // Positional parameters — legacy:
    @Query("SELECT u FROM User u WHERE u.email = ?1 AND u.active = ?2")
    Optional<User> findByEmailAndActivePositional(String email, boolean active);

    // Common mistake — missing @Param:
    // @Query("SELECT u FROM User u WHERE u.email = :email")
    // Optional<User> findByEmail(String email); // ← @Param missing → exception

    // Multiple uses of same parameter:
    @Query("SELECT u FROM User u WHERE u.firstName = :name OR u.lastName = :name")
    List<User> findByFirstOrLastName(@Param("name") String name);
    // :name bound once, used twice in JPQL — valid

    // SpEL entity name:
    @Query("SELECT u FROM #{#entityName} u WHERE u.createdAt > :since")
    List<User> findCreatedAfter(@Param("since") LocalDateTime since);
}
```

---

### Example 2 — `@Modifying` with Both Flags

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Simple UPDATE — returns affected row count:
    @Modifying
    @Transactional
    @Query("UPDATE Product p SET p.active = false WHERE p.expiresAt < :now")
    int deactivateExpiredProducts(@Param("now") LocalDateTime now);

    // With flushAutomatically — ensures pending changes are written first:
    @Modifying(flushAutomatically = true)
    @Transactional
    @Query("UPDATE Product p SET p.stock = p.stock - :qty WHERE p.id = :id AND p.stock >= :qty")
    int decrementStock(@Param("id") Long id, @Param("qty") int qty);

    // With clearAutomatically — prevents stale PC after bulk update:
    @Modifying(clearAutomatically = true)
    @Transactional
    @Query("UPDATE Product p SET p.price = p.price * :multiplier WHERE p.category = :cat")
    int applyPriceMultiplier(@Param("cat") String cat, @Param("multiplier") BigDecimal mult);

    // With both — full consistency:
    @Modifying(flushAutomatically = true, clearAutomatically = true)
    @Transactional
    @Query("DELETE FROM Product p WHERE p.active = false AND p.updatedAt < :cutoff")
    int purgeInactiveProducts(@Param("cutoff") LocalDateTime cutoff);

    // INSERT via native query:
    @Modifying
    @Transactional
    @Query(value = "INSERT INTO product_archive SELECT * FROM products WHERE id = :id",
           nativeQuery = true)
    int archiveProduct(@Param("id") Long id);
}

// Service demonstrating stale data problem:
@Service
public class ProductService {

    @Transactional
    public void demonstrateStalenessWithoutClear() {
        Product p = productRepo.findById(1L).get();
        System.out.println("Before: " + p.getPrice()); // 100.00

        // Bulk UPDATE bypasses entity lifecycle:
        productRepo.applyPriceMultiplier("Electronics", new BigDecimal("1.1"));
        // DB now has price = 110.00

        // WITHOUT clearAutomatically:
        System.out.println("After: " + p.getPrice()); // Still 100.00! (stale PC)

        // WITH clearAutomatically=true:
        // p is now detached (PC cleared)
        Product fresh = productRepo.findById(1L).get(); // re-fetches from DB
        System.out.println("After reload: " + fresh.getPrice()); // 110.00
    }
}
```

---

### Example 3 — Native Queries with All Mapping Approaches

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Approach 1: Map to entity directly (needs all columns):
    @Query(value = "SELECT o.* FROM orders o " +
                   "INNER JOIN customers c ON o.customer_id = c.id " +
                   "WHERE c.region = :region AND o.status = 'PENDING'",
           nativeQuery = true)
    List<Order> findPendingOrdersByRegion(@Param("region") String region);

    // Approach 2: Interface projection (most practical for partial data):
    @Query(value = "SELECT o.id, o.order_date, c.name as customer_name, " +
                   "COUNT(i.id) as item_count, SUM(i.unit_price * i.quantity) as total " +
                   "FROM orders o " +
                   "JOIN customers c ON o.customer_id = c.id " +
                   "LEFT JOIN order_items i ON o.id = i.order_id " +
                   "WHERE o.status = :status " +
                   "GROUP BY o.id, o.order_date, c.name",
           nativeQuery = true)
    List<OrderSummaryProjection> findOrderSummaries(@Param("status") String status);

    // Approach 3: Object array (raw, avoid in production):
    @Query(value = "SELECT o.id, o.total, c.name FROM orders o JOIN customers c ...",
           nativeQuery = true)
    List<Object[]> findRawData();

    // Paginated native query — MUST provide countQuery:
    @Query(value = "SELECT o.* FROM orders o WHERE o.customer_id = :cid " +
                   "ORDER BY o.created_at DESC",
           countQuery = "SELECT COUNT(*) FROM orders WHERE customer_id = :cid",
           nativeQuery = true)
    Page<Order> findByCustomerPaged(@Param("cid") Long customerId, Pageable pageable);
}

// Projection interface for native query:
public interface OrderSummaryProjection {
    Long getId();
    LocalDateTime getOrderDate();
    String getCustomerName();   // matches 'as customer_name' (Spring converts snake_case)
    Long getItemCount();        // matches 'as item_count'
    BigDecimal getTotal();      // matches 'as total'
}
```

---

### Example 4 — JPQL with Aggregations, Subqueries, GROUP BY

```java
public interface SalesRepository extends JpaRepository<Order, Long> {

    // GROUP BY with HAVING:
    @Query("SELECT o.customer.id, COUNT(o), SUM(o.total) " +
           "FROM Order o " +
           "WHERE o.createdAt >= :since " +
           "GROUP BY o.customer.id " +
           "HAVING COUNT(o) >= :minOrders")
    List<Object[]> findHighValueCustomers(
        @Param("since") LocalDateTime since,
        @Param("minOrders") long minOrders);

    // Subquery in WHERE:
    @Query("SELECT o FROM Order o WHERE o.total > " +
           "(SELECT AVG(o2.total) FROM Order o2 WHERE o2.customer = o.customer)")
    List<Order> findAboveAverageOrders();

    // EXISTS subquery:
    @Query("SELECT p FROM Product p WHERE EXISTS " +
           "(SELECT oi FROM OrderItem oi WHERE oi.product = p AND oi.order.status = 'PENDING')")
    List<Product> findProductsWithPendingOrders();

    // Constructor DTO + GROUP BY:
    @Query("SELECT new com.example.dto.MonthlySalesDto(" +
           "FUNCTION('YEAR', o.createdAt), " +
           "FUNCTION('MONTH', o.createdAt), " +
           "COUNT(o), SUM(o.total)) " +
           "FROM Order o " +
           "WHERE o.createdAt >= :since " +
           "GROUP BY FUNCTION('YEAR', o.createdAt), FUNCTION('MONTH', o.createdAt) " +
           "ORDER BY FUNCTION('YEAR', o.createdAt), FUNCTION('MONTH', o.createdAt)")
    List<MonthlySalesDto> findMonthlySales(@Param("since") LocalDateTime since);

    // IN subquery:
    @Query("SELECT u FROM User u WHERE u.id IN " +
           "(SELECT DISTINCT o.customer.id FROM Order o WHERE o.total > :threshold)")
    List<User> findCustomersWithLargeOrders(@Param("threshold") BigDecimal threshold);
}
```

---

### Example 5 — Dynamic Queries with SpEL and Optional Parameters

```java
public interface CustomerRepository extends JpaRepository<Customer, Long> {

    // SpEL for dynamic null-safe parameter:
    @Query("SELECT c FROM Customer c WHERE " +
           "(:name IS NULL OR LOWER(c.name) LIKE LOWER(CONCAT('%', :name, '%'))) AND " +
           "(:email IS NULL OR c.email = :email) AND " +
           "(:active IS NULL OR c.active = :active)")
    List<Customer> search(
        @Param("name") String name,
        @Param("email") String email,
        @Param("active") Boolean active);
    // Null parameter = condition effectively skipped
    // search(null, null, null) = findAll()
    // search("Smith", null, true) = active customers named Smith

    // SpEL for collection parameter handling:
    @Query("SELECT c FROM Customer c WHERE " +
           "(:#{#regions.size()} = 0 OR c.region IN :regions)")
    List<Customer> findByRegions(@Param("regions") List<String> regions);
    // Empty list = find all; non-empty = filter by regions

    // Base interface with SpEL entity name for reuse:
    @Query("SELECT e FROM #{#entityName} e WHERE e.createdAt BETWEEN :from AND :to")
    List<T> findCreatedBetween(
        @Param("from") LocalDateTime from,
        @Param("to") LocalDateTime to);
}
```

---

### Example 6 — `@Query` + `@EntityGraph` Combination

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // @Query for complex WHERE + @EntityGraph for fetch control:
    @Query("SELECT o FROM Order o WHERE o.status = :status AND o.total > :threshold")
    @EntityGraph(attributePaths = {"customer", "items", "items.product"})
    List<Order> findLargeOrdersWithDetails(
        @Param("status") OrderStatus status,
        @Param("threshold") BigDecimal threshold);
    // The @Query controls the filter condition
    // The @EntityGraph controls what is eagerly loaded
    // Spring Data applies EntityGraph hint to the TypedQuery
    // Result: single SQL with LEFT JOINs for customer, items, items.product

    // Pageable + @Query + @EntityGraph + custom count query:
    @Query(value = "SELECT o FROM Order o WHERE o.customer.id = :cid",
           countQuery = "SELECT COUNT(o) FROM Order o WHERE o.customer.id = :cid")
    @EntityGraph(attributePaths = {"customer", "items"})
    Page<Order> findByCustomerWithDetails(
        @Param("cid") Long customerId, Pageable pageable);
    // countQuery doesn't use EntityGraph (no need to load associations for count)
}
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ**
```java
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(String email); // no @Param
```
What happens at runtime when this method is called?

A) Works correctly — Spring Data infers the binding automatically  
B) Throws `IllegalStateException` at startup because `@Param` is missing  
C) Throws `ParameterNotFoundOrAmbiguousException` or similar at execution  
D) Returns `Optional.empty()` because parameter is unbound  

**Answer: C** — Without `@Param("email")`, Spring Data cannot bind `:email` to the method parameter. This fails at query execution time (not startup). In some configurations with Java 8+ `-parameters` compiler flag, it may work by reflection. On exams without that assumption: missing `@Param` = binding exception at execution.

---

**Q2 — Select All That Apply**
Which statements about `@Modifying` are TRUE? (Select all)

A) `@Modifying` is required for any `@Query` that performs UPDATE or DELETE  
B) Without `@Modifying`, Spring Data calls `executeUpdate()` automatically  
C) `clearAutomatically = true` evicts all entities from the Persistence Context after the bulk operation  
D) `flushAutomatically = true` flushes pending changes to DB before executing the bulk query  
E) `@Modifying` alone (without `@Transactional`) is sufficient for UPDATE queries  
F) `@Modifying` queries can return `int`, `Integer`, `void`, or `long`  

**Answer: A, C, D, F**
- B is false: without `@Modifying`, Spring Data calls `getResultList()` (SELECT semantics) — not `executeUpdate()`
- E is false: `@Modifying` without `@Transactional` in a context with no active transaction throws `TransactionRequiredException`

---

**Q3 — SQL/JPQL Prediction**
```java
@Query("SELECT p FROM Product p WHERE p.category IN :cats AND p.price < :maxPrice")
List<Product> findByCategoryAndMaxPrice(
    @Param("cats") List<String> categories,
    @Param("maxPrice") BigDecimal maxPrice);

// Called with:
findByCategoryAndMaxPrice(List.of("Electronics", "Books"), new BigDecimal("50.00"));
```

What SQL is generated?

A) `SELECT * FROM products WHERE category = ? AND price < ?`  
B) `SELECT * FROM products WHERE category IN (?,?) AND price < ?`  
C) `SELECT * FROM products WHERE category = 'Electronics' AND category = 'Books' AND price < ?`  
D) `HibernateException` — IN clause with collection parameter is not supported  

**Answer: B** — JPQL `IN :cats` with a collection parameter generates SQL `IN (?,?)` with one placeholder per collection element. Hibernate expands the collection into individual bind parameters.

---

**Q4 — `clearAutomatically` Behavior**
```java
@Entity
public class Product {
    @Id Long id;
    BigDecimal price;
}

@Modifying // NO clearAutomatically
@Transactional
@Query("UPDATE Product p SET p.price = p.price * 2 WHERE p.category = 'Electronics'")
int doubleElectronicsPrice();

@Transactional
public void test() {
    Product laptop = productRepo.findById(1L).get(); // price=1000, category=Electronics
    productRepo.doubleElectronicsPrice(); // bulk UPDATE: DB now has price=2000
    System.out.println(laptop.getPrice()); // What is printed?
}
```

A) 2000 — entity is updated by bulk operation  
B) 1000 — entity in PC is stale; bulk UPDATE bypasses entity lifecycle  
C) `LazyInitializationException`  
D) Depends on whether `flush()` was called  

**Answer: B** — Bulk UPDATE via JPQL bypasses the Persistence Context entirely. The entity `laptop` in PC still holds the old snapshot value (1000). Without `clearAutomatically=true`, the PC is never cleared and the stale value persists. Only `em.refresh(laptop)` or re-fetching after `em.clear()` would show 2000.

---

**Q5 — Native Query countQuery**
```java
@Query(value = "SELECT o.* FROM orders o JOIN customers c ON o.customer_id = c.id " +
               "WHERE c.region = :region",
       nativeQuery = true)
Page<Order> findByRegion(@Param("region") String region, Pageable pageable);
// No countQuery provided
```

What happens?

A) Works correctly — Spring Data auto-generates an efficient count query  
B) Works but may fail or perform poorly — Spring Data wraps the query in `SELECT COUNT(*) FROM (...)` which may fail on some DBs  
C) Fails at startup — `countQuery` is mandatory for native `Page<T>` queries  
D) Works correctly — Hibernate handles count internally  

**Answer: B** — When `countQuery` is not provided for a native query, Spring Data attempts to auto-derive a count query by wrapping: `SELECT COUNT(*) FROM (original_query) x`. This fails on some databases (e.g., older MySQL, some Oracle configurations). Always provide an explicit `countQuery` for native `Page<T>`.

---

**Q6 — SpEL `#{#entityName}`**
```java
@Entity(name = "AppUser")
public class User { ... }

@NoRepositoryBean
public interface BaseRepo<T, ID> extends JpaRepository<T, ID> {
    @Query("SELECT e FROM #{#entityName} e WHERE e.active = true")
    List<T> findAllActive();
}

public interface UserRepository extends BaseRepo<User, Long> { }
```

What JPQL is generated when `userRepo.findAllActive()` is called?

A) `SELECT e FROM User e WHERE e.active = true`  
B) `SELECT e FROM AppUser e WHERE e.active = true`  
C) `SELECT e FROM T e WHERE e.active = true`  
D) `PropertyReferenceException` — `#{#entityName}` is not valid SpEL  

**Answer: B** — `#{#entityName}` resolves to the `@Entity(name=...)` value if provided, otherwise the simple class name. Since `@Entity(name="AppUser")` is set, JPQL uses `AppUser`. This is why `#{#entityName}` is preferred in base repository interfaces — it always resolves correctly regardless of the entity's Java class name.

---

**Q7 — `@Modifying` Return Types**
```java
@Modifying
@Transactional
@Query("DELETE FROM Session s WHERE s.expiredAt < :now")
// What return types are valid here?
```

A) Only `void`  
B) Only `int`  
C) `void`, `int`, `Integer`, `long`  
D) `boolean` (true if any rows deleted)  

**Answer: C** — `@Modifying` query methods can return `void`, `int`, `Integer`, or `long`. The numeric return represents affected row count. `boolean` is NOT a valid return type for modifying queries.

---

**Q8 — JPQL Validation Timing**
```java
// In UserRepository:
@Query("SELECT u FROM UnknownEntity u WHERE u.email = :email")
List<User> findByEmailWrong(@Param("email") String email);
```

When does the error surface?

A) At compile time — Spring validates JPQL at compile time  
B) At application startup — `QueryException` or `IllegalArgumentException`  
C) At first method invocation — lazy validation  
D) Never — Spring Data silently returns empty list  

**Answer: B** — JPQL in `@Query` is validated against the JPA metamodel at application startup (during `SimpleJpaQuery` construction). `UnknownEntity` is not a registered entity → `IllegalArgumentException: Unknown entity: UnknownEntity` at startup.

Contrast: `nativeQuery=true` bypasses this — native SQL is NOT validated at startup, only at first execution.

---

**Q9 — Parameter Style Mixing**
```java
@Query(value = "SELECT * FROM users WHERE first_name = ?1 AND last_name = :last",
       nativeQuery = true)
List<User> findByName(String firstName, @Param("last") String lastName);
```

What happens?

A) Works — mixing is allowed for native queries  
B) HibernateException — mixing positional (`?1`) and named (`:last`) parameters in the same query is not allowed  
C) Only the named parameter is bound; positional is ignored  
D) Works for JPQL but not native queries  

**Answer: B** — Mixing positional (`?1`) and named (`:last`) parameters in the same query string is not permitted. Choose one style and use it consistently throughout the query.

---

## 4️⃣ Trick Analysis

**Missing `@Param` on named parameter (Q1)**:
The most common `@Query` mistake. `:email` in JPQL needs a `@Param("email")` annotation on the corresponding method parameter. Without it, Spring Data cannot resolve the binding. The `-parameters` compiler flag makes this work without `@Param` in some environments by preserving method parameter names — but this is not guaranteed and should never be relied upon for exam questions. Always use `@Param`.

**`@Modifying` without `@Transactional` (Q2, E)**:
`@Modifying` tells Spring Data the query is a write operation (use `executeUpdate()`). But it does NOT provide a transaction. If there is no active transaction when the method is called, the JDBC driver throws `TransactionRequiredException`. The class-level `@Transactional(readOnly=true)` on `SimpleJpaRepository` does NOT apply to custom interface methods — you must add `@Transactional` explicitly on `@Modifying` methods.

**Stale Persistence Context after bulk UPDATE (Q4)**:
This is one of the most dangerous silent bugs in JPA. Bulk JPQL operations (`UPDATE`, `DELETE`) bypass the entity lifecycle and go directly to the database. The Persistence Context has no knowledge of these changes. Entities already loaded in PC retain their old values. Without `clearAutomatically=true` or explicit `em.clear()`, all subsequent reads from PC return stale data. The fix sequence: `flushAutomatically=true` → bulk operation → `clearAutomatically=true`.

**`countQuery` omission with native `Page<T>` (Q5)**:
Spring Data's auto-generated count query for native queries wraps the original SQL: `SELECT COUNT(*) FROM (your_sql) x`. This fails on databases that don't support subquery aliasing in COUNT. Always provide an explicit `countQuery` for native `Page<T>` methods. This is a production stability issue, not just a performance concern.

**`#{#entityName}` resolves to `@Entity(name=...)` not class name (Q6)**:
When a class is annotated `@Entity(name="AppUser")`, the JPA entity name is `AppUser` — not `User`. `#{#entityName}` resolves to this JPA entity name. If your base interface query uses `FROM #{#entityName}`, and an entity has a custom `@Entity(name=...)`, the JPQL uses that custom name. This is correct behavior but surprises developers who expect the Java class name.

**`@Modifying` return type restrictions (Q7)**:
`boolean` is NOT a valid return type for `@Modifying` queries. The valid types are `void`, `int`, `Integer`, and `long`. A common mistake is returning `boolean` to indicate "was anything deleted" — use `int` and check if `> 0` instead.

**JPQL validated at startup, native SQL at execution (Q8)**:
JPQL in `@Query` is compiled and validated against the JPA metamodel when the Spring context loads. Invalid entity names or property references fail fast. Native SQL (`nativeQuery=true`) is NOT validated at startup — it is passed directly to JDBC at execution time. A typo in a native query's table name is only discovered when the method is first called in production.

---

## 5️⃣ Summary Sheet

### JPQL vs Native Query — Decision Matrix

```
Use JPQL (@Query) when:
  - Query is portable (may change DB vendor)
  - Query complexity: GROUP BY, subqueries, aggregates
  - Return type is entity or DTO (automatic mapping)
  - Need pagination with auto-generated count query
  - Need Pageable/Sort parameter support
  - JPQL is validated at startup (faster error detection)

Use native (@Query nativeQuery=true) when:
  - DB-specific feature needed (JSONB, full-text, CTEs, LATERAL)
  - Complex window functions
  - Stored procedure output parameters
  - Performance-critical query needing exact SQL control
  - Migrating from legacy SQL codebase
  - Always provide explicit countQuery for Page<T>
```

### `@Modifying` Attribute Reference

```
clearAutomatically = false (default):
  - After bulk UPDATE/DELETE, PC still has OLD values
  - Loaded entities show stale data
  - Risk: stale reads after bulk operations
  
clearAutomatically = true:
  - After executeUpdate(), calls em.clear()
  - All entities become DETACHED
  - Next access to any entity = fresh SELECT from DB
  - Risk: all in-memory unsaved state is lost
  
flushAutomatically = false (default):
  - Bulk query executes against DB state at that moment
  - Pending entity changes in ActionQueue NOT yet written
  - Risk: bulk update operates on stale DB state
  
flushAutomatically = true:
  - Before executeUpdate(), calls em.flush()
  - All pending ActionQueue changes written to DB first
  - Bulk query sees all in-memory changes
  - Safe ordering guaranteed

Recommended for most bulk operations:
  @Modifying(flushAutomatically = true, clearAutomatically = true)
```

### Parameter Binding Reference

| Style | Syntax | `@Param` Needed? | Example |
|---|---|---|---|
| Named (JPQL) | `:paramName` | YES — always | `:email` + `@Param("email")` |
| Positional (JPQL) | `?1`, `?2` | No | `?1` = first param |
| Named (Native) | `:paramName` | YES | `:region` + `@Param("region")` |
| Positional (Native) | `?1`, `?2` | No | `?1` = first param |
| SpEL | `:#{#param}` | YES — for `#param` | `:#{#name.toLowerCase()}` |
| Mixed | NOT ALLOWED | — | Mixing named+positional = exception |

### Return Type Rules for `@Modifying`

```
Valid @Modifying return types:
  void    - ignore affected count
  int     - affected row count (primitive)
  Integer - affected row count (nullable)
  long    - affected row count (large tables)

INVALID:
  boolean  - NOT valid, use int > 0 check
  List<T>  - NOT valid for @Modifying
  Optional - NOT valid for @Modifying
```

### SpEL Expression Reference

```
#{#entityName}
  - Resolves to: @Entity(name=...) value, or simple class name if not set
  - Use case: base repository interface queries for reuse across entities

:#{#paramName}
  - Accesses @Param value via SpEL
  - Allows SpEL manipulation: :#{#name.toUpperCase()}

:#{#paramName.size()}
  - Collection size check for conditional logic

:#{[0]}
  - Positional access to first method parameter (rare)
```

### Validation Timing Reference

```
Validated at STARTUP:
  - JPQL in @Query (entity names, field names, parameter count)
  - Derived query method names (property paths, keyword types)
  - @NamedQuery JPQL
  
Validated at EXECUTION TIME:
  - Native SQL (nativeQuery=true) — no startup validation
  - @Param binding (at first invocation)
  - Constructor DTO parameter count/type match
```

### Key Rules

```
1.  @Param("name") is REQUIRED for named parameters in @Query - never omit
2.  @Modifying requires @Transactional - readOnly=true at class level is NOT overridden by @Modifying alone
3.  Bulk UPDATE/DELETE bypasses PC - entities in PC show STALE data without clearAutomatically=true
4.  flushAutomatically=true - flush before bulk op (write ordering)
5.  clearAutomatically=true - clear PC after bulk op (stale prevention)
6.  countQuery is REQUIRED for native Page<T> queries (auto-generation is unreliable)
7.  nativeQuery=true - SQL not validated at startup; only at execution time
8.  JPQL @Query - validated at startup; invalid entity/field = QueryException at startup
9.  #{#entityName} resolves to @Entity(name=...) value, not Java class name
10. Cannot mix named (:param) and positional (?1) in same query
11. @Modifying valid return types: void, int, Integer, long - NOT boolean
12. Native queries do NOT support Sort parameter - embed ORDER BY in SQL string
13. Interface projection aliases in JPQL must match getter names (camelCase)
14. Constructor DTO requires fully-qualified class name in JPQL: new com.example.Dto(...)
15. clearAutomatically=true discards ALL in-memory state - unsaved changes lost
```

---

