# 🏛️ TOPIC 17 — Spring Data JPA Projections

---

## 1️⃣ Conceptual Explanation

### Why Projections Exist — The Over-Fetching Problem

Loading full entities when only two fields are needed wastes memory, network bandwidth, and deserialization time. A `User` entity with 30 fields loaded for a dropdown that only needs `id` and `name` is a structural inefficiency that compounds at scale.

```
Without projections:
  SELECT u.id, u.first_name, u.last_name, u.email, u.password_hash,
         u.phone, u.address, u.bio, u.avatar_url, u.preferences,
         u.created_at, u.last_login, ... (30 columns)
  → Full entity in Persistence Context
  → Snapshot created for dirty checking
  → All fields deserialized

With projection:
  SELECT u.id, u.first_name
  → 2 columns fetched
  → No entity tracking overhead
  → Minimal memory footprint
```

Spring Data JPA supports three distinct projection mechanisms, each with different internal implementations, SQL generation patterns, and trade-offs.

---

### The Three Projection Types — Overview

```
Type 1: Interface Projection (Closed)
  → Interface with getter methods matching entity field names
  → Spring Data creates a JDK dynamic proxy at runtime
  → Only declared getter fields are SELECTed
  → Most efficient — smallest SELECT

Type 2: Interface Projection (Open)
  → Interface with @Value SpEL expressions
  → Spring Data loads FULL entity, then evaluates SpEL
  → NOT more efficient than loading full entity
  → Flexible — can compute derived values

Type 3: DTO Projection (Class-based)
  → POJO / record class with matching constructor
  → Uses JPQL constructor expression: new Dto(field1, field2)
  → No entity loaded into Persistence Context
  → No proxy — direct object construction
  → Fastest for read-only data transfer
```

---

### Interface Projections — Closed Projections

A **closed projection** declares exactly the fields it needs. Spring Data can optimize the SQL SELECT to include only those fields.

```java
// Closed projection interface:
public interface UserSummary {
    Long getId();
    String getFirstName();
    String getLastName();
    // No getEmail(), no getPasswordHash(), etc.
}

// Repository usage:
public interface UserRepository extends JpaRepository<User, Long> {

    // Derived query returning projection:
    List<UserSummary> findByActive(boolean active);

    // @Query returning projection:
    @Query("SELECT u.id as id, u.firstName as firstName, " +
           "u.lastName as lastName FROM User u WHERE u.active = :active")
    List<UserSummary> findSummaryByActive(@Param("active") boolean active);

    // findAll equivalent:
    List<UserSummary> findAllProjectedBy();
    // Derived query with no predicate — returns all as projection
}
```

**SQL generated for closed projection with derived query:**

```sql
-- Spring Data generates:
SELECT u.id, u.first_name, u.last_name
FROM users u
WHERE u.active = ?

-- Only the three columns declared in UserSummary
-- NOT SELECT * or SELECT u.*
```

**How Spring Data determines which columns to SELECT:**

```
1. Introspects the projection interface
2. Finds all abstract getter methods
3. Extracts property names: getId() → "id", getFirstName() → "firstName"
4. Maps to entity fields via naming strategy
5. Generates SELECT with only those columns

This optimization applies ONLY to derived queries.
For @Query: you must manually write the SELECT with only needed columns.
The @Query SELECT clause is used verbatim — Spring Data does not rewrite it.
```

---

### Interface Projection Internals — The JDK Proxy

Spring Data implements interface projections using JDK dynamic proxies. When a projection query returns, Spring Data does NOT create a `User` entity. Instead, for each result row, it creates a `Proxy` that:

1. Implements the projection interface
2. Holds the raw column values from the ResultSet
3. Routes getter calls to the stored values

```
ResultSet row: [1, "Alice", "Smith"]
                ↓
Spring Data ProjectionFactory
                ↓
JDK Proxy implementing UserSummary
  → getId()        returns 1L
  → getFirstName() returns "Alice"
  → getLastName()  returns "Smith"
  → hashCode()     based on all values
  → equals()       based on all values
  → toString()     includes all values

The proxy is NOT a User entity:
  userSummary instanceof User          → false
  userSummary instanceof UserSummary   → true
  userSummary.getClass().getSimpleName()→ "$ProxyXX" (JDK proxy class name)
```

**Consequence:** Projection results are **not** placed in the Persistence Context. No dirty checking, no identity map, no snapshots. This is what makes them faster for read operations.

---

### Closed Projection with Nested Properties

```java
// Nested projection — accessing associated entity properties:
public interface OrderSummary {
    Long getId();
    LocalDateTime getOrderDate();
    BigDecimal getTotal();
    CustomerInfo getCustomer();  // nested projection

    interface CustomerInfo {     // nested interface
        String getName();
        String getEmail();
    }
}

// Repository:
List<OrderSummary> findByStatus(OrderStatus status);

// SQL generated:
// SELECT o.id, o.order_date, o.total,
//        c.name, c.email          ← nested CustomerInfo fields
// FROM orders o
// LEFT JOIN customers c ON o.customer_id = c.id
// WHERE o.status = ?

// The nested CustomerInfo is also a JDK proxy
// orderSummary.getCustomer().getName() → "Alice"
// orderSummary.getCustomer() instanceof CustomerInfo → true
```

**Nested projection with `@Value`:**

```java
public interface OrderSummary {
    Long getId();

    @Value("#{target.customer.name + ' (' + target.customer.email + ')'}")
    String getCustomerDisplay();
    // → "Alice Smith (alice@example.com)"
    // This loads the full Customer entity to access name and email via SpEL
}
```

---

### Interface Projections — Open Projections with `@Value`

An **open projection** uses SpEL expressions via `@Value`. The expression receives `target` — the full underlying entity.

```java
public interface UserDetails {

    // Simple @Value:
    @Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName();
    // → "Alice Smith"

    // Conditional:
    @Value("#{target.active ? 'ACTIVE' : 'INACTIVE'}")
    String getStatusLabel();

    // Method call:
    @Value("#{target.email.toLowerCase()}")
    String getLowerEmail();

    // Nested navigation:
    @Value("#{target.address?.city ?: 'Unknown'}")
    String getCity();

    // Multiple targets:
    @Value("#{target.firstName.substring(0,1) + '. ' + target.lastName}")
    String getFormalName();
    // → "A. Smith"
}
```

**Critical performance fact about open projections:**

```
Open projections (@Value with SpEL) require the FULL entity to be loaded.
Spring Data cannot determine which fields are needed from the SpEL expression.
→ SQL: SELECT * FROM users (all columns)
→ Full entity loaded into memory
→ SpEL evaluated against full entity

Open projections provide ZERO SELECT optimization.
They are syntactic sugar for computed properties, not a performance tool.
Use only when you need derived/computed values not present in the entity directly.
```

---

### DTO Projections — Constructor-Based

DTO projections use JPQL `new` keyword (constructor expression) or interface projections without a proxy.

**Class-based DTO projection:**

```java
// DTO class (or record):
public class UserDto {
    private final Long id;
    private final String firstName;
    private final String lastName;

    // Constructor MUST match JPQL new expression exactly:
    public UserDto(Long id, String firstName, String lastName) {
        this.id = id;
        this.firstName = firstName;
        this.lastName = lastName;
    }
    // getters...
}

// Or Java record (cleaner):
public record UserDto(Long id, String firstName, String lastName) {}
```

**Using constructor DTO in `@Query`:**

```java
// FULLY QUALIFIED class name required in JPQL:
@Query("SELECT new com.example.dto.UserDto(u.id, u.firstName, u.lastName) " +
       "FROM User u WHERE u.active = :active")
List<UserDto> findUserDtos(@Param("active") boolean active);

// SQL generated:
// SELECT u.id, u.first_name, u.last_name
// FROM users u WHERE u.active = ?
// ResultSet rows directly passed to UserDto constructor
// NO entity created, NO Persistence Context overhead
```

**DTO with aggregation:**

```java
public record DepartmentStats(
    String department,
    long employeeCount,
    BigDecimal avgSalary,
    BigDecimal maxSalary
) {}

@Query("SELECT new com.example.dto.DepartmentStats(" +
       "e.department, COUNT(e), AVG(e.salary), MAX(e.salary)) " +
       "FROM Employee e GROUP BY e.department")
List<DepartmentStats> findDepartmentStats();

// This is IMPOSSIBLE with entity loading or interface projections
// DTO projection is the only clean solution for aggregate results
```

---

### Spring Data Interface Projection vs `@Query` Constructor Expression — SQL Difference

```java
// Interface projection with DERIVED query (Spring Data rewrites SELECT):
List<UserSummary> findByActive(boolean active);
// SQL: SELECT u.id, u.first_name, u.last_name FROM users WHERE active=?
//      ↑ Only projection columns

// Interface projection with @Query (you write the SELECT):
@Query("SELECT u FROM User u WHERE u.active = :active")
List<UserSummary> findByActiveWithQuery(@Param("active") boolean active);
// SQL: SELECT u.id, u.first_name, u.last_name, u.email, u.password, ...
//      ↑ ALL columns — Spring Data uses full entity SQL from @Query
// WARNING: This loads full entity, then maps to projection proxy — no column optimization!

// Correct @Query for interface projection (alias-based):
@Query("SELECT u.id as id, u.firstName as firstName, " +
       "u.lastName as lastName FROM User u WHERE u.active = :active")
List<UserSummary> findByActiveCorrect(@Param("active") boolean active);
// SQL: SELECT u.id, u.first_name, u.last_name FROM users WHERE active=?
// ↑ Only projection columns — because @Query explicitly selects them
```

**The alias requirement for `@Query` + interface projection:**

```java
// ALIASES in SELECT must match getter names in interface:
public interface OrderSummary {
    Long getId();           // alias: 'id'
    String getCustomerName(); // alias: 'customerName'
    BigDecimal getTotal();  // alias: 'total'
}

@Query("SELECT o.id AS id, " +          // ← 'id' matches getId()
       "c.name AS customerName, " +      // ← 'customerName' matches getCustomerName()
       "o.total AS total " +             // ← 'total' matches getTotal()
       "FROM Order o JOIN o.customer c")
List<OrderSummary> findOrderSummaries();

// Without aliases: Spring Data cannot map ResultSet columns to interface getters
// Result: all getters return null
```

---

### Dynamic Projections — Projection Type as Generic Parameter

Spring Data allows the caller to specify which projection type to use at call time:

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Dynamic projection — projection type passed as parameter:
    <T> List<T> findByActive(boolean active, Class<T> type);

    // Single result:
    <T> Optional<T> findById(Long id, Class<T> type);
}

// Usage:
// Full entity:
List<User> users = userRepo.findByActive(true, User.class);

// Summary projection:
List<UserSummary> summaries = userRepo.findByActive(true, UserSummary.class);

// Different DTO:
List<UserDto> dtos = userRepo.findByActive(true, UserDto.class);

// Works for both interface and DTO projections
// Spring Data detects the type and applies appropriate optimization:
//   User.class     → SELECT * (full entity)
//   UserSummary.class → SELECT id, first_name, last_name (only projection fields)
//   UserDto.class  → SELECT id, first_name, last_name (constructor expression)
```

---

### Projection with Native Queries

Interface projections work with native queries, but require careful column name matching:

```java
public interface ProductNativeSummary {
    Long getId();
    String getName();
    BigDecimal getPrice();
}

@Query(value = "SELECT p.id, p.name, p.price FROM products p WHERE p.active = :active",
       nativeQuery = true)
List<ProductNativeSummary> findActiveSummaryNative(@Param("active") boolean active);

// Column names in SQL must match getter names after Spring Data's name conversion:
// p.id    → getId()    ✓ (exact match)
// p.name  → getName()  ✓ (exact match)
// p.price → getPrice() ✓ (exact match)

// Snake case columns auto-converted:
// p.created_at → getCreatedAt()  ✓ (Spring converts snake_case to camelCase)
// p.user_name  → getUserName()   ✓

// Aliasing for non-matching names:
@Query(value = "SELECT p.id, p.product_name AS name, " +
               "p.unit_price AS price FROM products p",
       nativeQuery = true)
List<ProductNativeSummary> findWithAliases();
// p.product_name → aliased as 'name' → matches getName()
```

---

### `@Value` in Interface Projections — SpEL Reference

```java
public interface ProductProjection {

    // Simple field access (closed):
    Long getId();
    String getName();

    // Computed from single field (open — loads full entity):
    @Value("#{target.price * 1.2}")
    BigDecimal getPriceWithTax();

    // Conditional:
    @Value("#{target.stock > 0 ? 'IN_STOCK' : 'OUT_OF_STOCK'}")
    String getAvailability();

    // String manipulation:
    @Value("#{target.name.toUpperCase()}")
    String getNameUpperCase();

    // Null-safe navigation:
    @Value("#{target.category?.name ?: 'Uncategorized'}")
    String getCategoryName();

    // Bean invocation (call Spring bean in SpEL):
    @Value("#{@currencyFormatter.format(target.price)}")
    String getFormattedPrice();
    // @currencyFormatter = Spring bean named 'currencyFormatter'
    // Very powerful but: entire entity loaded, bean invoked per row
}
```

---

### Projection Comparison — Internal Mechanics

```
Closed Interface Projection:
  SQL: SELECT only declared columns
  Binding: ResultSet values stored in proxy's internal map
  PC: Entity NOT placed in Persistence Context
  Proxy: JDK dynamic proxy per result row
  Dirty checking: None (not in PC)
  Memory: Minimal (only declared fields stored)
  Best for: Reading subsets of fields for display

Open Interface Projection (@Value SpEL):
  SQL: SELECT * (full entity loaded)
  Binding: Full entity created (may go into PC temporarily)
  PC: Depends on implementation — entity may be managed
  SpEL: Evaluated per row against full entity
  Memory: Full entity in memory for SpEL evaluation
  Best for: Computed/derived display values

DTO Projection (Constructor Expression):
  SQL: SELECT only declared constructor parameters
  Binding: Constructor called directly per result row
  PC: Entity NOT placed in Persistence Context
  No proxy: Direct POJO/record creation
  Dirty checking: None
  Memory: Only constructor parameters
  Best for: Read-only transfer objects, aggregation results

Dynamic Projection:
  SQL: Depends on type — closed projection = column subset
  Type determined at: call time by Class<T> parameter
  Same method, different SQL based on projection type
  Best for: Single repository method serving multiple consumers
```

---

### Projection and N+1 — The Association Trap

```java
// OrderSummary projection with nested CustomerInfo:
public interface OrderSummary {
    Long getId();
    BigDecimal getTotal();
    CustomerInfo getCustomer();  // navigates @ManyToOne association

    interface CustomerInfo {
        String getName();
        String getEmail();
    }
}

List<OrderSummary> orders = orderRepo.findByStatus(OrderStatus.PENDING);
// SQL 1: SELECT o.id, o.total, c.name, c.email
//        FROM orders o LEFT JOIN customers c ON o.customer_id = c.id
// Spring Data handles the JOIN for the nested projection — no N+1 here

// BUT: if CustomerInfo is a separate @Audited entity with lazy fields:
// → Spring Data may need to initialize proxies for nested data
// → N+1 risk for deeply nested projections

// Safe pattern: explicit @Query controls the JOIN:
@Query("SELECT o.id as id, o.total as total, " +
       "c.name as customerName, c.email as customerEmail " +
       "FROM Order o JOIN o.customer c WHERE o.status = :status")
List<OrderFlatProjection> findFlat(@Param("status") OrderStatus status);

// Flat projection (no nesting):
public interface OrderFlatProjection {
    Long getId();
    BigDecimal getTotal();
    String getCustomerName();   // from JOIN, no nested proxy
    String getCustomerEmail();  // from JOIN, no nested proxy
}
// Single query, no N+1, no nested proxy
```

---

### Projection with `Page<T>` and `Slice<T>`

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Paginated projection — works cleanly:
    Page<UserSummary> findByActive(boolean active, Pageable pageable);

    Slice<UserSummary> findByDepartment(String dept, Pageable pageable);

    // Data query: SELECT u.id, u.first_name, u.last_name
    //             FROM users WHERE active=? LIMIT ? OFFSET ?
    // Count query: SELECT COUNT(u.id) FROM users WHERE active=?
    // (Count query doesn't need projection — uses COUNT on entity)
}

// Service:
Page<UserSummary> page = userRepo.findByActive(true,
    PageRequest.of(0, 20, Sort.by("lastName")));

// page.map() on projection Page:
Page<UserDto> dtoPage = page.map(summary ->
    new UserDto(summary.getId(),
                summary.getFirstName(),
                summary.getLastName()));
// Maps projection proxies to DTOs — preserves pagination metadata
```

---

### Record Projections — Modern Approach (Java 16+)

```java
// Java record as projection DTO:
public record UserSummaryRecord(
    Long id,
    String firstName,
    String lastName
) {}

// In repository:
@Query("SELECT new com.example.dto.UserSummaryRecord" +
       "(u.id, u.firstName, u.lastName) FROM User u WHERE u.active = true")
List<UserSummaryRecord> findActiveSummaries();

// Benefits of records for projections:
// - Immutable by default
// - Built-in equals(), hashCode(), toString()
// - Compact canonical constructor
// - No boilerplate setters

// Interface projection as alternative to record (no @Query needed for simple cases):
public interface UserSummaryInterface {
    Long getId();
    String getFirstName();
    String getLastName();
}
List<UserSummaryInterface> findByActive(boolean active);
// Generates optimized SELECT automatically — no @Query needed
```

---

### When to Use Which Projection

```
Use Closed Interface Projection when:
  ✓ Need a subset of entity fields
  ✓ Simple derived query (no @Query)
  ✓ Nesting into related entities needed
  ✓ Dynamic projection (same method, multiple types)
  ✓ Don't want to write @Query for simple cases

Use DTO (Constructor) Projection when:
  ✓ Need aggregate functions (COUNT, SUM, AVG)
  ✓ Need fields from multiple unrelated entities
  ✓ Need immutable, fully-typed value object
  ✓ Performance-critical — avoiding proxy overhead
  ✓ Using Java records (cleanest integration)
  ✓ Cross-aggregate queries

Use Open Projection (@Value) when:
  ✓ Need computed/derived display value
  ✓ String concatenation from multiple fields
  ✓ Conditional display formatting
  ✗ NOT for performance — loads full entity

Avoid Projections and Use Full Entity when:
  ✓ Entity will be modified (need dirty checking)
  ✓ Need full entity state for business logic
  ✓ Will call lazy-loaded associations extensively
  ✓ Entity lifecycle callbacks needed
```

---

## 2️⃣ Code Examples

### Example 1 — Closed Interface Projection with All Patterns

```java
// Entity:
@Entity
public class Employee {
    @Id @GeneratedValue Long id;
    String firstName, lastName, email, department;
    BigDecimal salary;
    boolean active;
    LocalDateTime hiredAt;

    @ManyToOne Department dept;
    @OneToMany(mappedBy = "employee") List<Project> projects;
}

// Projection interfaces:
public interface EmployeeSummary {
    Long getId();
    String getFirstName();
    String getLastName();
    String getDepartment();
}

public interface EmployeeContactInfo {
    Long getId();
    String getFirstName();
    String getLastName();
    String getEmail();
}

// Nested projection:
public interface EmployeeWithDept {
    Long getId();
    String getFirstName();
    DepartmentInfo getDept();  // navigates @ManyToOne

    interface DepartmentInfo {
        Long getId();
        String getName();
        String getLocation();
    }
}

// Repository with dynamic projection:
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    // Fixed projection:
    List<EmployeeSummary> findByActive(boolean active);
    // SQL: SELECT e.id, e.first_name, e.last_name, e.department
    //      FROM employees WHERE active=?

    // Dynamic projection:
    <T> List<T> findByDepartment(String dept, Class<T> type);
    // Caller chooses: Employee.class, EmployeeSummary.class, EmployeeContactInfo.class

    // Nested projection with derived query:
    List<EmployeeWithDept> findByActiveTrue();
    // SQL: SELECT e.id, e.first_name, d.id, d.name, d.location
    //      FROM employees e LEFT JOIN departments d ON e.dept_id = d.id

    // Paginated projection:
    Page<EmployeeSummary> findBySalaryGreaterThan(
        BigDecimal salary, Pageable pageable);
}

// Service using dynamic projection:
@Service
public class EmployeeService {

    public <T> List<T> findByDept(String dept, Class<T> projectionType) {
        return employeeRepo.findByDepartment(dept, projectionType);
    }
}

// Controller using different projections:
@RestController
public class EmployeeController {

    @GetMapping("/employees/summary")
    public List<EmployeeSummary> getSummaries() {
        return employeeService.findByDept("Engineering", EmployeeSummary.class);
        // SQL: SELECT e.id, e.first_name, e.last_name, e.department ...
    }

    @GetMapping("/employees/contacts")
    public List<EmployeeContactInfo> getContacts() {
        return employeeService.findByDept("Engineering", EmployeeContactInfo.class);
        // SQL: SELECT e.id, e.first_name, e.last_name, e.email ...
        // Different columns selected for different projection type
    }

    @GetMapping("/employees/full")
    public List<Employee> getFull() {
        return employeeService.findByDept("Engineering", Employee.class);
        // SQL: SELECT e.* ... (full entity)
    }
}
```

---

### Example 2 — DTO Projection with Aggregates and Records

```java
// Projection records:
public record ProductSummary(Long id, String name, BigDecimal price) {}
public record CategoryStats(
    String category,
    long productCount,
    BigDecimal avgPrice,
    BigDecimal minPrice,
    BigDecimal maxPrice
) {}
public record OrderCustomerSummary(
    Long orderId,
    LocalDateTime orderDate,
    String customerName,
    String customerEmail,
    BigDecimal total
) {}

// Repository:
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Simple DTO projection:
    @Query("SELECT new com.example.dto.ProductSummary(p.id, p.name, p.price) " +
           "FROM Product p WHERE p.active = true ORDER BY p.name")
    List<ProductSummary> findActiveSummaries();

    // Aggregate DTO:
    @Query("SELECT new com.example.dto.CategoryStats(" +
           "p.category, COUNT(p), AVG(p.price), MIN(p.price), MAX(p.price)) " +
           "FROM Product p WHERE p.active = true GROUP BY p.category")
    List<CategoryStats> findCategoryStats();

    // Multi-entity JOIN DTO:
    @Query("SELECT new com.example.dto.OrderCustomerSummary(" +
           "o.id, o.orderDate, c.name, c.email, o.total) " +
           "FROM Order o JOIN o.customer c " +
           "WHERE o.status = :status AND o.total > :minTotal")
    List<OrderCustomerSummary> findOrderCustomerSummaries(
        @Param("status") OrderStatus status,
        @Param("minTotal") BigDecimal minTotal);

    // Paginated DTO:
    @Query(value = "SELECT new com.example.dto.ProductSummary(p.id, p.name, p.price) " +
                   "FROM Product p WHERE p.active = true",
           countQuery = "SELECT COUNT(p) FROM Product p WHERE p.active = true")
    Page<ProductSummary> findActiveSummaryPage(Pageable pageable);
}

// Service demonstrating all patterns:
@Service
@Transactional(readOnly = true)
public class ProductAnalyticsService {

    public List<CategoryStats> getCategoryReport() {
        List<CategoryStats> stats = productRepo.findCategoryStats();
        // SQL: SELECT p.category, COUNT(p.id), AVG(p.price),
        //             MIN(p.price), MAX(p.price)
        //      FROM products p WHERE p.active=true GROUP BY p.category
        // Direct record construction — zero Persistence Context overhead
        stats.forEach(s -> System.out.printf(
            "Category: %s | Count: %d | Avg: %s%n",
            s.category(), s.productCount(), s.avgPrice()));
        return stats;
    }

    public Page<ProductSummary> getProductPage(int page, int size) {
        Pageable pageable = PageRequest.of(page, size,
            Sort.by("name").ascending());
        return productRepo.findActiveSummaryPage(pageable);
        // Data: SELECT id, name, price ... LIMIT ? OFFSET ?
        // Count: SELECT COUNT(p) ...
    }
}
```

---

### Example 3 — Alias-Based Interface Projection with `@Query`

```java
// Projection interface with aliases:
public interface RevenueReport {
    String getMonth();
    Long getOrderCount();
    BigDecimal getTotalRevenue();
    BigDecimal getAvgOrderValue();
    String getTopCategory();
}

// Repository with carefully aliased @Query:
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("SELECT " +
           "FUNCTION('DATE_FORMAT', o.createdAt, '%Y-%m') AS month, " +
           "COUNT(o) AS orderCount, " +
           "SUM(o.total) AS totalRevenue, " +
           "AVG(o.total) AS avgOrderValue, " +
           "o.primaryCategory AS topCategory " +
           "FROM Order o " +
           "WHERE o.createdAt >= :since " +
           "GROUP BY FUNCTION('DATE_FORMAT', o.createdAt, '%Y-%m'), o.primaryCategory " +
           "ORDER BY month ASC")
    List<RevenueReport> findMonthlyRevenue(@Param("since") LocalDateTime since);

    // CRITICAL: alias names must EXACTLY match getter names (camelCase):
    // 'month'         → getMonth()
    // 'orderCount'    → getOrderCount()
    // 'totalRevenue'  → getTotalRevenue()
    // 'avgOrderValue' → getAvgOrderValue()
    // 'topCategory'   → getTopCategory()

    // Each row → JDK proxy implementing RevenueReport
    // Proxy stores raw values from ResultSet
    // Getter calls retrieve stored values
}

// Native query with interface projection:
public interface ProductInventory {
    Long getId();
    String getName();
    Integer getCurrentStock();
    Integer getReservedStock();
    Integer getAvailableStock();  // computed in SQL
}

@Query(value = """
    SELECT
        p.id,
        p.name,
        p.stock AS current_stock,
        COALESCE(SUM(r.quantity), 0) AS reserved_stock,
        p.stock - COALESCE(SUM(r.quantity), 0) AS available_stock
    FROM products p
    LEFT JOIN reservations r ON r.product_id = p.id
        AND r.status = 'ACTIVE'
    GROUP BY p.id, p.name, p.stock
    HAVING available_stock < :threshold
    """,
    nativeQuery = true)
List<ProductInventory> findLowAvailability(@Param("threshold") int threshold);
// Snake case column names auto-converted:
// current_stock   → getCurrentStock()
// reserved_stock  → getReservedStock()
// available_stock → getAvailableStock()
```

---

### Example 4 — Open Projection and Performance Comparison

```java
// Open projection — demonstration of performance cost:
public interface UserDisplayInfo {

    // Closed fields (would benefit from column optimization IF no @Value):
    Long getId();
    String getEmail();

    // Open fields — these force full entity load:
    @Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName();

    @Value("#{target.active ? 'Active' : 'Deactivated'}")
    String getStatusDisplay();

    @Value("#{T(java.time.Period).between(target.birthDate, " +
           "T(java.time.LocalDate).now()).getYears()}")
    int getAge();  // computed from birthDate

    @Value("#{@greetingService.greet(target)}")
    String getPersonalizedGreeting(); // calls Spring bean
}

// Because @Value is present, full entity is loaded:
// SQL: SELECT u.id, u.first_name, u.last_name, u.email, u.active,
//            u.birth_date, u.phone, u.address, ... (ALL columns)
// Then SpEL evaluated on each entity instance

// Performance comparison test:
@Test
public void compareProjectionPerformance() {
    // Closed projection — minimal SQL:
    long t1 = System.currentTimeMillis();
    List<UserSummary> closed = userRepo.findAllByActiveTrue();
    // SQL: SELECT id, first_name, last_name FROM users WHERE active=true
    long closedTime = System.currentTimeMillis() - t1;

    // Open projection — full SQL:
    long t2 = System.currentTimeMillis();
    List<UserDisplayInfo> open =
        userRepo.findAllDisplayInfo(); // uses @Value
    // SQL: SELECT * FROM users WHERE active=true
    long openTime = System.currentTimeMillis() - t2;

    // DTO projection — most efficient:
    long t3 = System.currentTimeMillis();
    List<UserDto> dto = userRepo.findUserDtos();
    // SQL: SELECT id, first_name, last_name FROM users WHERE active=true
    // No proxy, direct constructor
    long dtoTime = System.currentTimeMillis() - t3;

    System.out.printf("Closed: %dms | Open: %dms | DTO: %dms%n",
                      closedTime, openTime, dtoTime);
    // Typical: Open ≈ 2-3x slower than Closed/DTO for large result sets
}
```

---

### Example 5 — Projection with Spring Data `findBy` Fluent API

```java
public interface EmployeeRepository
        extends JpaRepository<Employee, Long>,
                JpaSpecificationExecutor<Employee> {

    // findBy fluent API with projection (Spring Data 3.x):
    <T> T findById(Long id, Class<T> type);

    <T> List<T> findByDepartment(String dept, Class<T> type);
}

@Service
public class EmployeeQueryService {

    // Using findBy with fluent specification API + projection:
    public List<EmployeeSummary> findActiveSeniors(BigDecimal minSalary) {

        return employeeRepo.findBy(
            Specification
                .where(EmployeeSpecs.isActive())
                .and(EmployeeSpecs.salaryGreaterThan(minSalary)),
            query -> query
                .as(EmployeeSummary.class)  // projection type
                .all()
        );
        // SQL: SELECT e.id, e.first_name, e.last_name, e.department
        //      FROM employees e WHERE e.active=true AND e.salary > ?
        // Uses Specification for WHERE + projection for SELECT
    }

    public Page<EmployeeSummary> findActivePagedSummary(Pageable pageable) {
        return employeeRepo.findBy(
            EmployeeSpecs.isActive(),
            query -> query
                .as(EmployeeSummary.class)
                .page(pageable)
        );
    }
}
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ: Open vs Closed Projection SQL**
```java
public interface UserProjection {
    Long getId();
    String getFirstName();

    @Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName();
}

List<UserProjection> results = userRepo.findByActive(true);
```

What SQL is generated?

A) `SELECT u.id, u.first_name FROM users WHERE active=true`  
B) `SELECT u.id, u.first_name, u.last_name FROM users WHERE active=true`  
C) `SELECT u.* FROM users WHERE active=true` (full entity)  
D) `SELECT u.id, u.first_name, u.first_name, u.last_name FROM users WHERE active=true`  

**Answer: C** — The presence of any `@Value` annotation makes the projection **open**. Spring Data cannot determine which fields the SpEL expression `target.firstName + ' ' + target.lastName` will need at SQL generation time (SpEL is evaluated at runtime against the full object). Therefore, the full entity is loaded: `SELECT u.*`.

---

**Q2 — Select All That Apply: Closed Interface Projection**

Which statements are TRUE about closed interface projections?

A) They are implemented as JDK dynamic proxies at runtime  
B) Projected entities are placed in the Persistence Context for dirty checking  
C) Spring Data generates a SELECT with only the declared getter columns for derived queries  
D) `@Query("SELECT u FROM User u")` with a projection return type still fetches all columns  
E) Nested interface projections result in a JOIN being added to the SQL  
F) DTO (constructor) projections are always faster than closed interface projections  

**Answer: A, C, D, E**
- B is false: projection results are NOT placed in the Persistence Context
- F is false: closed interface projections and DTO projections have similar performance for simple cases; the difference is marginal in practice (both avoid entity lifecycle overhead)

---

**Q3 — Alias Matching**
```java
public interface RevenueStats {
    Long getOrderCount();
    BigDecimal getTotalAmount();
}

@Query("SELECT COUNT(o) AS order_count, SUM(o.total) AS total_amount " +
       "FROM Order o WHERE o.status = 'COMPLETED'")
RevenueStats findCompletedStats();
```

What do `getOrderCount()` and `getTotalAmount()` return?

A) Correct values — `order_count` converts to `orderCount` matching `getOrderCount()`  
B) `null` for both — aliases use snake_case, getter names use camelCase  
C) Correct values — Spring Data converts `order_count` → `orderCount` automatically  
D) Correct values — Spring Data converts `total_amount` → `totalAmount` automatically  

**Answer: C and D (all correct — A, C, D express the same truth)** — Spring Data automatically converts snake_case alias names from SQL to camelCase when matching to projection getter names. `order_count` → `orderCount` → matches `getOrderCount()`. `total_amount` → `totalAmount` → matches `getTotalAmount()`.

---

**Q4 — Dynamic Projection SQL**
```java
<T> List<T> findByDepartment(String dept, Class<T> type);

// Called with:
List<Employee> full     = repo.findByDepartment("Engineering", Employee.class);
List<EmpSummary> summary = repo.findByDepartment("Engineering", EmpSummary.class);

// EmpSummary interface:
// Long getId(); String getFirstName(); String getLastName();
```

How many SQL queries are executed, and do they differ?

A) 2 queries — identical SQL `SELECT * FROM employees WHERE department=?`  
B) 2 queries — different SQL: full entity gets `SELECT *`, projection gets `SELECT id, first_name, last_name`  
C) 1 query — Spring Data caches and reuses the result  
D) 2 queries — identical SQL, but projection result is filtered in Java  

**Answer: B** — Dynamic projection generates different SQL based on `Class<T>`. `Employee.class` → full entity select. `EmpSummary.class` (closed interface projection) → Spring Data introspects the interface and generates a SELECT with only the three declared columns. Two separate, different SQL queries.

---

**Q5 — Constructor DTO FQCN Requirement**
```java
// Package: com.example.dto
public record ProductDto(Long id, String name) {}

// In repository:
@Query("SELECT new ProductDto(p.id, p.name) FROM Product p")
List<ProductDto> findDtos();
```

What happens?

A) Works correctly — Spring Data resolves the class from the classpath  
B) `QueryException` at startup — fully qualified class name required  
C) `QueryException` at execution — fully qualified class name required  
D) `NullPointerException` at runtime — class not found  

**Answer: B** — JPQL `new` expressions require the **fully qualified class name**. `new ProductDto(...)` is invalid JPQL. The correct form is `new com.example.dto.ProductDto(p.id, p.name)`. This is validated at startup when the JPQL is compiled against the entity metamodel — `QueryException: Unable to locate class [ProductDto]`.

---

**Q6 — Projection + Page Count Query**
```java
Page<UserSummary> page = userRepo.findByActive(true, PageRequest.of(0, 20));
```

What SQL is generated for the count query?

A) `SELECT COUNT(u.id), u.first_name, u.last_name FROM users WHERE active=true`  
B) `SELECT COUNT(u.id) FROM users WHERE active=true`  
C) `SELECT COUNT(*) FROM (SELECT u.id, u.first_name, u.last_name FROM users WHERE active=true)`  
D) Same as data query but wrapped in COUNT  

**Answer: B** — Spring Data auto-generates the count query for `Page<T>` by extracting the FROM and WHERE clauses and applying `COUNT`. The projection type does not affect the count query — count queries don't need the projected columns. Result: efficient `SELECT COUNT(u.id) FROM users WHERE active=true`.

---

**Q7 — `@Query` + Interface Projection Column Fetch**
```java
public interface NameOnly {
    String getFirstName();
    String getLastName();
}

@Query("SELECT u FROM User u WHERE u.department = :dept")
List<NameOnly> findNamesByDept(@Param("dept") String dept);
```

How many columns are fetched?

A) 2 — Spring Data optimizes the `@Query` to fetch only `first_name, last_name`  
B) All columns — `SELECT u FROM User u` fetches the full entity; projection applied after  
C) 0 — compilation error: `@Query` returning entity cannot be mapped to interface projection  
D) Depends on `FetchType` configuration  

**Answer: B** — When `@Query` uses `SELECT u FROM User u` (entity-level SELECT), Hibernate fetches all columns for the full entity. The interface projection is applied AFTER the full entity is loaded. Spring Data does NOT rewrite the `@Query` SELECT clause to optimize columns. To get column optimization, use explicit column selection in `@Query`: `SELECT u.firstName as firstName, u.lastName as lastName FROM User u`.

---

**Q8 — Proxy Identity**
```java
List<UserSummary> results = userRepo.findByActive(true);
UserSummary first = results.get(0);

System.out.println(first instanceof UserSummary);  // Line A
System.out.println(first instanceof User);          // Line B
System.out.println(first.getClass().getSimpleName()); // Line C
```

What are the outputs?

A) `true`, `true`, `User`  
B) `true`, `false`, `UserSummary`  
C) `true`, `false`, `$ProxyXX` (JDK proxy class name)  
D) `false`, `false`, `HashMap`  

**Answer: C** — Interface projection results are JDK dynamic proxies. The proxy implements `UserSummary` → `instanceof UserSummary` is `true`. The proxy does NOT implement `User` → `instanceof User` is `false`. `getClass().getSimpleName()` returns the JDK proxy class name like `$Proxy42` (the actual name varies).

---

## 4️⃣ Trick Analysis

**Any `@Value` annotation makes the entire projection open — full entity loaded (Q1)**:
This is the most commonly misunderstood projection fact. A projection interface with a mix of plain getters AND one `@Value` getter is an **open** projection. The entire entity is loaded regardless of how many plain getters exist. If `UserProjection` has `getId()`, `getFirstName()`, and one `@Value` getter, the SQL fetches all 30 columns. The plain getters provide no SELECT optimization in this case. For performance: keep `@Value` getters in a separate interface, or use DTO projections for computed values.

**`@Query("SELECT u FROM User u")` with projection return type does NOT optimize columns (Q7)**:
Spring Data rewrites the SELECT for **derived queries** returning projections — it introspects the projection interface and generates the minimal SELECT. For `@Query`, Spring Data uses the query as-is. `SELECT u FROM User u` means "select the full User entity" — Hibernate fetches all columns. The projection is applied afterward as a view over the already-loaded entity. To get column optimization with `@Query`, you must explicitly write `SELECT u.id as id, u.firstName as firstName FROM User u`.

**Fully qualified class name in JPQL constructor expression (Q5)**:
JPQL has no import mechanism. Every class reference in JPQL must be fully qualified: `new com.example.dto.ProductDto(...)`. This is validated at startup when Spring Data compiles the JPQL. The error message "Unable to locate class" is the tell. This is why many developers prefer interface projections for simple cases — no FQCN required, no `@Query` needed, automatic SQL optimization.

**Projection results are NOT in Persistence Context (Q2, B)**:
Closed interface projections and DTO projections are never placed in the Persistence Context. They cannot be made managed. Calling `save()` on a projection result is meaningless — you'd need to load the actual entity. This is deliberate: projections are read-only data views. If you need to modify data, load the full entity. If you only need to read, use a projection.

**Dynamic projection generates different SQL per type (Q4)**:
`<T> List<T> findByDept(String dept, Class<T> type)` is not "one query fits all." Spring Data evaluates the projection type at call time and generates the appropriate SQL. `Employee.class` = full entity SELECT. Closed interface = column-subset SELECT. Each call generates a different prepared statement. This is the power of dynamic projections — one method signature, multiple efficient SQL variants based on caller intent.

**Snake_case alias to camelCase getter conversion (Q3)**:
Spring Data automatically converts SQL alias names from snake_case to camelCase when matching to interface projection getters. `order_count` → `orderCount` → matches `getOrderCount()`. This means native query aliases written in snake_case (the SQL convention) work with camelCase getter names (the Java convention). No manual mapping needed. The conversion is one-way: SQL snake_case → Java camelCase.

---

## 5️⃣ Summary Sheet

### Three Projection Types Compared

| Aspect | Closed Interface | Open Interface (`@Value`) | DTO Constructor |
|---|---|---|---|
| SQL columns fetched | Only declared getters | ALL columns | Only constructor params |
| Implementation | JDK dynamic proxy | JDK dynamic proxy | Direct POJO construction |
| In Persistence Context | No | Entity may be | No |
| Dirty checking | None | None | None |
| Computed values | No | Yes (SpEL) | Via constructor logic |
| `@Query` needed | No (derived OK) | No | Yes (with `new`) |
| FQCN required | No | No | YES in JPQL |
| Aggregate queries | No | No | Yes |
| Performance | High | Low (full entity) | Highest |

### SQL Optimization Rules

```
Derived query + Closed Interface Projection:
  → Spring Data rewrites SELECT to include only declared columns ✓

Derived query + Open Interface Projection (@Value present):
  → Spring Data loads full entity (SELECT *) ✗

@Query("SELECT u FROM User u") + any projection:
  → Full entity loaded, projection applied as view ✗

@Query("SELECT u.id as id, u.name as name FROM User u") + interface projection:
  → Only declared columns fetched ✓

@Query("SELECT new com.example.Dto(u.id, u.name) FROM User u") + DTO:
  → Only constructor parameters fetched ✓
```

### Alias Matching Rules

```
JPQL alias    → Getter name matching:
  'firstName' → getFirstName()  ✓ (exact match)
  'first_name'→ getFirstName()  ✓ (auto snake_case → camelCase)
  'fname'     → getFirstName()  ✗ (no match → null)

Native SQL:
  'first_name' → getFirstName() ✓ (auto-converted)
  'FIRST_NAME' → getFirstName() ✓ (case-insensitive + conversion)
```

### Key Rules

```
1.  Any @Value in interface projection = open projection = full entity loaded
2.  Closed projection on derived query: Spring Data generates minimal SELECT
3.  @Query("SELECT u FROM User u") never optimized — always fetches full entity
4.  @Query with explicit column SELECT + aliases = optimized columns for interface projection
5.  DTO constructor projection: JPQL requires FULLY QUALIFIED class name
6.  Interface projection results are JDK dynamic proxies (NOT entity instances)
7.  Projection results NOT in Persistence Context — cannot be dirtied or saved
8.  Dynamic projection <T> findBy(Class<T> type): different SQL per projection type
9.  Nested interface projection = JOIN added to query automatically
10. @Value SpEL: target = the full entity/source object
11. Snake_case SQL aliases auto-converted to camelCase for getter matching
12. @Query + interface projection aliases MUST match getter names (after case conversion)
13. Open projection loads full entity — SpEL evaluated against it per row
14. DTO projection bypasses Persistence Context entirely — fastest for read-only
15. Page<ProjectionType>: data query uses projection columns, count query uses COUNT only
16. Record classes work as DTO projections with JPQL new expression
17. Projection proxy: instanceof ProjectionInterface = true, instanceof Entity = false
18. findAllProjectedBy() = derived query with no predicate, returns all as projection
```

---
