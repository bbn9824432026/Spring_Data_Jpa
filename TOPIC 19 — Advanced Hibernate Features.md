# 🏛️ TOPIC 19 — Advanced Hibernate Features

---

## 1️⃣ Conceptual Explanation

### Why These Features Exist

Standard JPA covers 80% of persistence needs. The remaining 20% — computed columns, dynamic filtering, soft deletes, multitenancy, custom types — requires Hibernate-specific extensions. These are the features that separate junior JPA users from architects who understand the full toolset.

---

### `@Formula` — Computed Columns from SQL Expressions

`@Formula` maps a read-only virtual field to an arbitrary SQL expression evaluated at SELECT time. The expression is embedded directly into the SELECT clause — not stored in a column.

```java
@Entity
public class Employee {
    @Id @GeneratedValue Long id;
    String firstName;
    String lastName;
    BigDecimal salary;
    BigDecimal bonus;
    LocalDate hireDate;

    // Computed from two columns — no DB column:
    @Formula("first_name || ' ' || last_name")
    String fullName;
    // PostgreSQL/H2 use ||; MySQL uses CONCAT(first_name, ' ', last_name)

    // Computed numeric expression:
    @Formula("salary + bonus")
    BigDecimal totalCompensation;

    // Date calculation:
    @Formula("DATEDIFF(CURDATE(), hire_date)")
    Integer daysEmployed;  // MySQL-specific

    // Subquery — related entity aggregate:
    @Formula("(SELECT COUNT(*) FROM orders o WHERE o.employee_id = id)")
    Long orderCount;
    // 'id' here refers to THIS entity's id column in the current row context

    // Conditional:
    @Formula("CASE WHEN salary > 100000 THEN 'SENIOR' ELSE 'STANDARD' END")
    String gradeLevel;
}
```

**How `@Formula` is included in SQL:**

```sql
-- Generated SELECT for Employee:
SELECT
    e.id,
    e.first_name,
    e.last_name,
    e.salary,
    e.bonus,
    e.hire_date,
    e.first_name || ' ' || e.last_name AS full_name,   ← @Formula
    e.salary + e.bonus AS total_compensation,           ← @Formula
    DATEDIFF(CURDATE(), e.hire_date) AS days_employed,  ← @Formula
    (SELECT COUNT(*) FROM orders o
     WHERE o.employee_id = e.id) AS order_count         ← @Formula subquery
FROM employees e
```

**Critical rules:**

```
1. @Formula expressions use TABLE COLUMN NAMES, not Java field names
   → 'first_name' not 'firstName'; 'hire_date' not 'hireDate'

2. @Formula is READ-ONLY — Hibernate never writes this field
   → No INSERT or UPDATE includes @Formula fields
   → Setter exists (for Hibernate to set after load) but should not be called manually

3. @Formula is dialect-dependent — SQL syntax varies by DB
   → || (PostgreSQL/H2) vs CONCAT() (MySQL) vs + (SQL Server)

4. @Formula in subqueries: unqualified column names refer to the OUTER QUERY's table alias
   → 'id' in "(SELECT COUNT(*) ... WHERE o.employee_id = id)"
   → refers to the employees table's id column in the outer query context

5. @Formula is evaluated on EVERY SELECT of the entity
   → Including lazy-loaded contexts where entity is fetched
   → Subquery @Formula on a 1000-row result = 1000 subqueries in SELECT clause
   → Use judiciously — consider JPQL with explicit projection instead for heavy formulas
```

---

### `@Filter` and `@FilterDef` — Dynamic Query Filters

Filters are reusable, activatable WHERE conditions applied transparently to entity queries. They are defined once on the entity/collection and activated per-session at runtime.

```java
// Step 1: Define filter (on entity or package):
@FilterDef(
    name = "activeFilter",
    defaultCondition = "active = true"
)
@FilterDef(
    name = "departmentFilter",
    parameters = @ParamDef(name = "deptName", type = String.class),
    defaultCondition = "department = :deptName"
)

// Step 2: Apply filter to entity:
@Entity
@Filter(name = "activeFilter")
@Filter(name = "departmentFilter")
public class Employee {
    @Id @GeneratedValue Long id;
    String name;
    String department;
    boolean active;

    // Filter can also apply to collection:
    @OneToMany(mappedBy = "employee")
    @Filter(name = "activeFilter")  // filters items in the collection
    List<Assignment> assignments;
}

// Step 3: Enable filter at runtime (per session):
@Service
public class EmployeeService {

    @PersistenceContext EntityManager em;

    @Transactional(readOnly = true)
    public List<Employee> findActiveEmployees() {
        Session session = em.unwrap(Session.class);
        session.enableFilter("activeFilter");
        // All queries in this session now add "AND active = true" automatically

        return em.createQuery("SELECT e FROM Employee e", Employee.class)
                 .getResultList();
        // SQL: SELECT e.* FROM employees e WHERE e.active = true
        //      ↑ filter condition injected automatically
    }

    @Transactional(readOnly = true)
    public List<Employee> findByDepartment(String dept) {
        Session session = em.unwrap(Session.class);
        session.enableFilter("departmentFilter")
               .setParameter("deptName", dept);

        return em.createQuery("SELECT e FROM Employee e", Employee.class)
                 .getResultList();
        // SQL: SELECT e.* FROM employees WHERE e.department = ?
    }

    @Transactional(readOnly = true)
    public List<Employee> findAllUnfiltered() {
        // Filter NOT enabled in this session → no WHERE condition added
        return em.createQuery("SELECT e FROM Employee e", Employee.class)
                 .getResultList();
        // SQL: SELECT e.* FROM employees  (no active filter)
    }
}
```

**How filters interact with `em.find()`:**

```java
// CRITICAL: @Filter does NOT apply to em.find() — only to queries
session.enableFilter("activeFilter");

// em.find() bypasses filter — loads entity regardless of filter condition:
Employee inactive = em.find(Employee.class, 99L);
// SQL: SELECT ... FROM employees WHERE id=99
// Returns entity even if active=false — filter ignored for em.find()

// @Filter applies to:
// ✓ JPQL queries (createQuery)
// ✓ Criteria API queries
// ✓ Spring Data derived queries
// ✓ Collection initialization (lazy loading)
// ✗ em.find() — direct PK lookup bypasses filter
// ✗ @Query with nativeQuery=true — filter not injected into native SQL
```

**Filter with Spring Data:**

```java
@Service
@Transactional(readOnly = true)
public class EmployeeQueryService {

    @PersistenceContext EntityManager em;
    @Autowired EmployeeRepository employeeRepo;

    public List<Employee> findFiltered() {
        // Enable filter before calling repository:
        em.unwrap(Session.class).enableFilter("activeFilter");

        // Repository methods now use the session with the filter enabled:
        return employeeRepo.findAll();
        // SQL: SELECT e.* FROM employees WHERE active=true
        // Filter injected automatically into all queries in this session
    }
}
```

---

### `@Where` — Static Soft Delete Filter

`@Where` adds a STATIC, always-applied SQL condition to all queries for an entity or collection. Unlike `@Filter`, it cannot be toggled — it is always active.

```java
// Soft delete pattern with @Where:
@Entity
@Where(clause = "deleted_at IS NULL")
public class Product {
    @Id @GeneratedValue Long id;
    String name;
    BigDecimal price;

    @Column(name = "deleted_at")
    LocalDateTime deletedAt;  // null = active, non-null = soft deleted
}

// @Where behavior:
// ALL queries for Product automatically add "AND deleted_at IS NULL"
productRepo.findAll();
// SQL: SELECT p.* FROM products p WHERE p.deleted_at IS NULL

productRepo.findById(5L);
// SQL: SELECT p.* FROM products WHERE id=5 AND deleted_at IS NULL
// If product 5 is soft-deleted: returns empty Optional — NOT the deleted product

productRepo.findByName("Widget");
// SQL: SELECT p.* FROM products WHERE name=? AND deleted_at IS NULL

// Soft delete service:
@Transactional
public void softDelete(Long productId) {
    Product product = productRepo.findById(productId)
        .orElseThrow(ProductNotFoundException::new);
    product.setDeletedAt(LocalDateTime.now());
    // No hard DELETE — just sets deletedAt timestamp
    // Future queries for this product return empty (filtered by @Where)
}
```

**`@Where` on collections:**

```java
@Entity
public class Category {
    @Id Long id;
    String name;

    @OneToMany(mappedBy = "category")
    @Where(clause = "deleted_at IS NULL")  // only non-deleted products in collection
    List<Product> activeProducts;
}
// category.getActiveProducts() → only returns non-soft-deleted products
```

**`@Where` vs `@Filter`:**

```
@Where:
  - STATIC — always applied, cannot be disabled
  - Simpler — no activation step needed
  - Applied to em.find() AND JPQL queries AND collections
  - Cannot access deleted records through normal JPA means
  - Best for: soft delete (deleted records should NEVER be visible in normal queries)

@Filter:
  - DYNAMIC — enabled/disabled per session at runtime
  - More flexible — same entity can be queried with and without filter
  - Does NOT apply to em.find()
  - Best for: multi-tenant filtering, role-based data visibility, optional conditions
```

---

### `@NaturalId` — Business Key Mapping

`@NaturalId` marks fields that form the entity's **natural business key** — a unique identifier meaningful in the business domain (email, ISBN, product code, tax ID).

```java
@Entity
public class User {
    @Id @GeneratedValue Long id;           // surrogate PK (internal)

    @NaturalId                             // business key (external)
    @Column(unique = true, nullable = false)
    String email;

    String firstName, lastName;
    LocalDateTime createdAt;
}

// With multiple fields forming composite natural ID:
@Entity
public class Book {
    @Id @GeneratedValue Long id;

    @NaturalId @Column(nullable = false)
    String isbn;                           // could be standalone natural ID

    // OR composite:
    @NaturalId String issn;
    @NaturalId String edition;
    // Together isbn+edition form the natural ID
}
```

**`@NaturalId` lookup — Hibernate-specific session API:**

```java
@Service
public class UserService {

    @PersistenceContext EntityManager em;

    @Transactional(readOnly = true)
    public Optional<User> findByEmail(String email) {
        Session session = em.unwrap(Session.class);

        User user = session.byNaturalId(User.class)
                           .using("email", email)
                           .load();
        // SQL: SELECT u.* FROM users u WHERE u.email=?
        // Result cached in L2C under NaturalId cache region (separate from entity cache)
        // Subsequent calls: natural ID lookup hits cache → no SQL

        return Optional.ofNullable(user);
    }

    // getReference() equivalent — proxy, no SQL unless accessed:
    public User referenceByEmail(String email) {
        Session session = em.unwrap(Session.class);
        return session.byNaturalId(User.class)
                      .using("email", email)
                      .getReference();
    }
}
```

**`@NaturalId` caching:**

```java
// L2C stores two separate cache regions per @NaturalId entity:
// 1. Entity cache: (User, id=42L) → { id:42, email:"alice@example.com", ... }
// 2. NaturalId cache: (User, email="alice@example.com") → id=42L

// Lookup by natural ID:
// Check NaturalId cache: ("alice@example.com") → 42L (cache HIT)
// Check entity cache: (User, 42L) → entity (cache HIT)
// → Zero SQL for warm cache

// Two-step resolution: natural ID → PK → entity
// Both steps can be served from cache
```

**`@NaturalId(mutable = true)` — when business key can change:**

```java
@NaturalId(mutable = true)  // default is mutable = false (immutable)
String email;
// mutable=false: Hibernate asserts email never changes (exception if you try)
// mutable=true: email CAN change; Hibernate updates NaturalId cache on change
// Use mutable=false when business key is truly immutable (ISBN, tax ID)
// Use mutable=true when business key can change (email addresses)
```

---

### `@Immutable` — Read-Only Entities and Collections

`@Immutable` tells Hibernate that an entity or collection is **never modified after initial creation**. Hibernate skips dirty checking entirely for immutable entities.

```java
@Entity
@Immutable
public class AuditLog {
    @Id @GeneratedValue Long id;
    String action;
    String entityType;
    Long entityId;
    String username;
    LocalDateTime timestamp;
    String details;
}

// What @Immutable does:
// ✓ Hibernate SKIPS dirty checking (no snapshot, no comparison)
// ✓ @PreUpdate callbacks are ignored
// ✓ UPDATE SQL is never generated — even if you call save() or set fields
// ✓ DELETE SQL IS generated (immutable ≠ indestructible)
// ✓ Significant memory saving: no snapshot copies created

// Attempting to update:
AuditLog log = auditLogRepo.findById(1L).get();
log.setAction("MODIFIED");  // setter executes (field changes in memory)
// TX commit → Hibernate detects @Immutable → skips dirty check → no UPDATE
// Change silently discarded

// @Immutable on collection:
@Entity
public class Product {
    @OneToMany(mappedBy = "product")
    @Immutable
    List<PriceHistory> priceHistory;
    // Price history never changes after recording
    // Hibernate skips dirty checking for this collection
}
```

---

### `@DynamicUpdate` and `@DynamicInsert` — Partial SQL Generation

By default, Hibernate generates a static UPDATE that includes ALL columns. `@DynamicUpdate` generates an UPDATE that includes ONLY the changed columns.

```java
@Entity
@DynamicUpdate   // UPDATE only changed columns
@DynamicInsert   // INSERT only non-null columns
public class UserProfile {
    @Id Long id;
    String firstName;
    String lastName;
    String email;
    String bio;           // rarely updated
    byte[] avatar;        // rarely updated, large
    LocalDateTime lastLoginAt;  // updated on every login
    int loginCount;             // updated on every login
    String preferences;   // updated occasionally
    // ... 20 more fields
}

// WITHOUT @DynamicUpdate:
// UPDATE user_profiles SET
//   first_name=?, last_name=?, email=?, bio=?, avatar=?,
//   last_login_at=?, login_count=?, preferences=?, ... (all 20+ columns)
// WHERE id=?
// Sends ALL column values even if only lastLoginAt changed

// WITH @DynamicUpdate (only lastLoginAt and loginCount changed):
// UPDATE user_profiles SET
//   last_login_at=?, login_count=?
// WHERE id=?
// Much smaller UPDATE — less network traffic, less DB work

// WITHOUT @DynamicInsert:
// INSERT INTO user_profiles (id, first_name, last_name, email, bio, avatar, ...)
// VALUES (?, ?, ?, ?, null, null, ...)   ← includes null columns

// WITH @DynamicInsert (bio and avatar are null):
// INSERT INTO user_profiles (id, first_name, last_name, email, ...)
// VALUES (?, ?, ?, ?, ...)   ← null columns omitted
// Allows DB DEFAULT values to apply (instead of explicit null overriding defaults)
```

**When `@DynamicUpdate` helps and hurts:**

```
HELPS when:
  ✓ Wide entities (many columns) with few fields changing per operation
  ✓ Columns with expensive DB triggers per column
  ✓ Binary/BLOB columns (avoid sending unchanged LOBs on every update)
  ✓ Optimistic locking with many readers (smaller UPDATE = less lock time)

HURTS when:
  ✗ Narrow entities (few columns) — overhead of building dynamic SQL > benefit
  ✗ Entities where most fields change on every UPDATE
  ✗ Statement caching: static SQL is cached by JDBC; dynamic SQL creates many variants
  ✗ High-insert throughput: @DynamicInsert disables batch INSERT optimization
```

---

### `@SelectBeforeUpdate` — Force Pre-Update SELECT

```java
@Entity
@SelectBeforeUpdate
public class Configuration {
    @Id Long id;
    String key;
    String value;
    LocalDateTime updatedAt;
}

// Normal merge() behavior:
// em.merge(detachedConfig) → UPDATE config SET ... WHERE id=?
// (no SELECT before UPDATE — assumes all fields need updating)

// With @SelectBeforeUpdate:
// em.merge(detachedConfig) → SELECT config WHERE id=? first
//   → Hibernate compares loaded entity with detached state
//   → Only changed fields are included in UPDATE
//   → If nothing changed: NO UPDATE AT ALL

// Use case: detached entities merged from UI where most fields unchanged
// Prevents unnecessary UPDATEs when nothing actually changed
// Trade-off: extra SELECT before every merge()
```

---

### `AttributeConverter<X, Y>` — Custom Type Mapping

`AttributeConverter` maps a Java type (`X`) to a database column type (`Y`). It bridges types that JPA cannot handle natively.

```java
// Example 1: Enum to specific string values (not ORDINAL or STRING):
public enum Status { ACTIVE, INACTIVE, PENDING }

@Converter(autoApply = true)  // applies to ALL Status fields in ALL entities
public class StatusConverter implements AttributeConverter<Status, String> {

    @Override
    public String convertToDatabaseColumn(Status status) {
        if (status == null) return null;
        return switch (status) {
            case ACTIVE   -> "A";
            case INACTIVE -> "I";
            case PENDING  -> "P";
        };
    }

    @Override
    public Status convertToEntityAttribute(String dbValue) {
        if (dbValue == null) return null;
        return switch (dbValue) {
            case "A" -> Status.ACTIVE;
            case "I" -> Status.INACTIVE;
            case "P" -> Status.PENDING;
            default  -> throw new IllegalArgumentException(
                "Unknown status code: " + dbValue);
        };
    }
}

// Example 2: JSON field:
@Converter
public class JsonConverter implements AttributeConverter<Map<String, Object>, String> {

    private static final ObjectMapper mapper = new ObjectMapper();

    @Override
    public String convertToDatabaseColumn(Map<String, Object> map) {
        if (map == null) return null;
        try {
            return mapper.writeValueAsString(map);
        } catch (JsonProcessingException e) {
            throw new IllegalArgumentException("Cannot serialize to JSON", e);
        }
    }

    @Override
    public Map<String, Object> convertToEntityAttribute(String json) {
        if (json == null) return null;
        try {
            return mapper.readValue(json,
                new TypeReference<Map<String, Object>>() {});
        } catch (JsonProcessingException e) {
            throw new IllegalArgumentException("Cannot deserialize JSON", e);
        }
    }
}

// Example 3: Encrypt sensitive fields:
@Converter
public class EncryptedStringConverter
        implements AttributeConverter<String, String> {

    @Autowired EncryptionService encryptionService;
    // WARNING: @Autowired in converter requires special configuration
    // Better: use static encryption utility to avoid Spring dependency

    @Override
    public String convertToDatabaseColumn(String plaintext) {
        return plaintext == null ? null :
               encryptionService.encrypt(plaintext);
    }

    @Override
    public String convertToEntityAttribute(String encrypted) {
        return encrypted == null ? null :
               encryptionService.decrypt(encrypted);
    }
}

// Usage on entity:
@Entity
public class User {
    @Id Long id;

    @Convert(converter = StatusConverter.class)  // explicit if not autoApply
    Status status;

    @Convert(converter = JsonConverter.class)
    @Column(columnDefinition = "TEXT")
    Map<String, Object> metadata;

    @Convert(converter = EncryptedStringConverter.class)
    @Column(name = "ssn_encrypted")
    String ssn;  // stored encrypted in DB
}
```

**`autoApply = true` — when to use:**

```
@Converter(autoApply = true):
  → Applies to ALL entity fields of type X in the application
  → No need for @Convert annotation on each field
  → Convenient for: enums, commonly-used value types
  → Risk: unexpected conversion on fields where you don't want it

@Converter (no autoApply or autoApply = false):
  → Only applies when explicitly declared with @Convert on the field
  → More explicit and controllable
  → Required for: field-specific converters (encrypt only some Strings)
```

---

### Hibernate-Specific Fetch Modes — `@Fetch`

```java
@Entity
public class Department {
    @Id Long id;
    String name;

    // FetchMode.SELECT (default for lazy):
    @OneToMany(mappedBy = "department", fetch = FetchType.LAZY)
    @Fetch(FetchMode.SELECT)
    List<Employee> employees;
    // Access employees: SELECT employees WHERE department_id=?  (1 extra query)

    // FetchMode.JOIN:
    @OneToMany(mappedBy = "department", fetch = FetchType.EAGER)
    @Fetch(FetchMode.JOIN)
    List<Employee> employees;
    // Access employees: JOIN in initial SELECT  (no extra query, but multiplies rows)

    // FetchMode.SUBSELECT:
    @OneToMany(mappedBy = "department")
    @Fetch(FetchMode.SUBSELECT)
    List<Employee> employees;
    // Access employees for ANY loaded department triggers one SUBSELECT:
    // SELECT e.* FROM employees WHERE department_id IN
    //   (SELECT d.id FROM departments d WHERE ...)
    // All departments' employees loaded in one query — solves N+1
}
```

**`@BatchSize` — batch loading to solve N+1:**

```java
@Entity
public class Order {
    @Id Long id;
    String status;

    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    @BatchSize(size = 10)  // load 10 orders' items at once
    List<OrderItem> items;
}

// Without @BatchSize — N+1:
// Query 1: SELECT * FROM orders WHERE status='PENDING' → 100 orders
// Query 2: SELECT items WHERE order_id=1
// Query 3: SELECT items WHERE order_id=2
// ... 100 extra queries

// With @BatchSize(size=10):
// Query 1: SELECT * FROM orders WHERE status='PENDING' → 100 orders
// Query 2: SELECT items WHERE order_id IN (1,2,3,4,5,6,7,8,9,10)
// Query 3: SELECT items WHERE order_id IN (11,12,13,14,15,16,17,18,19,20)
// ... 10 extra queries (not 100)
// Total: 1 + ceil(100/10) = 11 queries vs 101 queries

// @BatchSize on entity class (batches entity loading from L2C misses):
@Entity
@BatchSize(size = 20)
public class Product { ... }
// When Hibernate needs to load 50 Products by PK:
// Without: 50 SELECT queries
// With: ceil(50/20) = 3 IN-clause queries
```

---

### `@LazyCollection` and `@LazyToOne`

```java
// @LazyToOne — control lazy loading strategy for @ManyToOne / @OneToOne:
@ManyToOne(fetch = FetchType.LAZY)
@LazyToOne(LazyToOneOption.PROXY)          // default: JDK proxy placeholder
@LazyToOne(LazyToOneOption.NO_PROXY)       // bytecode-enhanced, no proxy
@LazyToOne(LazyToOneOption.FALSE)          // EAGER loading (same as FetchType.EAGER)
Department department;

// @LazyCollection — fine-grained collection lazy strategy:
@OneToMany(mappedBy = "order")
@LazyCollection(LazyCollectionOption.TRUE)   // lazy (default)
@LazyCollection(LazyCollectionOption.FALSE)  // eager
@LazyCollection(LazyCollectionOption.EXTRA) // EXTRA lazy:
List<OrderItem> items;
// EXTRA: size() and contains() do NOT initialize the full collection
// size() → SELECT COUNT(*) FROM order_items WHERE order_id=?
// contains(item) → SELECT 1 FROM order_items WHERE order_id=? AND id=?
// items.get(0) → full collection initialized
// Use: when you need collection size/membership without loading all elements
```

---

### Stored Procedures — `@NamedStoredProcedureQuery`

```java
// Define stored procedure mapping:
@Entity
@NamedStoredProcedureQuery(
    name = "Product.calculateDiscount",
    procedureName = "calculate_product_discount",
    parameters = {
        @StoredProcedureParameter(
            name = "p_product_id",
            mode = ParameterMode.IN,
            type = Long.class),
        @StoredProcedureParameter(
            name = "p_customer_tier",
            mode = ParameterMode.IN,
            type = String.class),
        @StoredProcedureParameter(
            name = "p_discount_rate",
            mode = ParameterMode.OUT,
            type = BigDecimal.class)
    }
)
public class Product {
    @Id Long id;
    String name;
    BigDecimal price;
}

// Invocation:
@Service
public class DiscountService {

    @PersistenceContext EntityManager em;

    @Transactional
    public BigDecimal calculateDiscount(Long productId, String tier) {
        StoredProcedureQuery query = em.createNamedStoredProcedureQuery(
            "Product.calculateDiscount");

        query.setParameter("p_product_id", productId);
        query.setParameter("p_customer_tier", tier);

        query.execute();

        return (BigDecimal) query.getOutputParameterValue("p_discount_rate");
    }

    // Ad-hoc stored procedure (without @NamedStoredProcedureQuery):
    @Transactional
    public void callAdHocProcedure(String param) {
        StoredProcedureQuery query = em.createStoredProcedureQuery(
            "process_data");
        query.registerStoredProcedureParameter(1, String.class, ParameterMode.IN);
        query.setParameter(1, param);
        query.execute();
    }
}
```

---

### Multitenancy — Data Isolation Patterns

```java
// Discriminator column approach (all tenants in one table):
@Entity
@Filter(
    name = "tenantFilter",
    condition = "tenant_id = :tenantId"
)
@FilterDef(
    name = "tenantFilter",
    parameters = @ParamDef(name = "tenantId", type = String.class)
)
public class Order {
    @Id Long id;

    @Column(nullable = false)
    String tenantId;

    String status;
    BigDecimal total;
}

// Tenant context setup (interceptor/filter applies to all requests):
@Component
public class TenantAwareSessionInterceptor {

    @PersistenceContext EntityManager em;

    public void setTenantFilter(String tenantId) {
        em.unwrap(Session.class)
          .enableFilter("tenantFilter")
          .setParameter("tenantId", tenantId);
    }
}

// Spring Boot 3.x Hibernate multitenancy (schema-per-tenant):
@Configuration
public class MultiTenancyConfig {

    @Bean
    public DataSource tenantRoutingDataSource() {
        // Route to different schemas/DBs based on current tenant
        return new TenantRoutingDataSource();
    }
}
```

---

### `@Subselect` — Entity Backed by a SQL View

```java
// Map an entity to a SQL view or subselect (read-only):
@Entity
@Subselect("""
    SELECT o.id,
           o.customer_id,
           c.name AS customer_name,
           COUNT(oi.id) AS item_count,
           SUM(oi.quantity * oi.unit_price) AS total
    FROM orders o
    JOIN customers c ON c.id = o.customer_id
    JOIN order_items oi ON oi.order_id = o.id
    GROUP BY o.id, o.customer_id, c.name
    """)
@Synchronize({"orders", "customers", "order_items"})
// @Synchronize: tells Hibernate to flush these tables before executing
//               queries on this entity (ensures consistent results)
@Immutable
public class OrderSummaryView {
    @Id Long id;
    Long customerId;
    String customerName;
    Long itemCount;
    BigDecimal total;
}

// Usage — exactly like a normal entity (read-only):
public interface OrderSummaryViewRepository
        extends JpaRepository<OrderSummaryView, Long> {
    List<OrderSummaryView> findByCustomerId(Long customerId);
}
// SQL: SELECT ... FROM (the subselect above) WHERE customer_id=?
// No DB view object needed — Hibernate inlines the SQL
```

---

## 2️⃣ Code Examples

### Example 1 — `@Formula` with Subqueries

```java
@Entity
public class Customer {
    @Id @GeneratedValue Long id;
    String name;
    String email;
    String tier;  // BRONZE, SILVER, GOLD

    // Total orders count — subquery:
    @Formula("(SELECT COUNT(*) FROM orders o WHERE o.customer_id = id)")
    Long totalOrders;

    // Total spend — subquery with aggregate:
    @Formula("(SELECT COALESCE(SUM(o.total), 0) FROM orders o " +
             "WHERE o.customer_id = id AND o.status = 'COMPLETED')")
    BigDecimal totalSpend;

    // Days since last order:
    @Formula("(SELECT DATEDIFF(CURRENT_DATE, MAX(o.created_at)) " +
             "FROM orders o WHERE o.customer_id = id)")
    Integer daysSinceLastOrder;

    // Computed display name:
    @Formula("UPPER(CONCAT(LEFT(name, 1), '. ', " +
             "SUBSTRING_INDEX(name, ' ', -1)))")
    String formalName;
    // "Alice Smith" → "A. Smith"
}

// Repository and service:
@Service
@Transactional(readOnly = true)
public class CustomerAnalyticsService {

    public List<Customer> findHighValueCustomers(BigDecimal minSpend) {
        // @Formula fields are populated on every load:
        return customerRepo.findAll().stream()
            .filter(c -> c.getTotalSpend()
                          .compareTo(minSpend) >= 0)
            .collect(toList());
        // For 1000 customers: each row's SELECT includes the subqueries
        // SQL: SELECT c.id, c.name, ...,
        //        (SELECT COUNT(*) FROM orders WHERE customer_id=c.id) AS total_orders,
        //        (SELECT COALESCE(SUM(...)) FROM orders WHERE ...) AS total_spend,
        //        ...
        //      FROM customers c
    }

    // Better approach for @Formula aggregates at scale:
    // Use a dedicated @Query with JOIN instead of @Formula subquery per row
    @Query("SELECT new CustomerStats(c.id, c.name, COUNT(o), SUM(o.total)) " +
           "FROM Customer c LEFT JOIN c.orders o GROUP BY c.id, c.name")
    List<CustomerStats> findCustomerStats();
    // Single query with GROUP BY vs N+1 subqueries
}
```

---

### Example 2 — Soft Delete with `@Where` and `@Filter`

```java
// ── APPROACH 1: @Where (always-on soft delete) ───────────────────────────
@Entity
@Where(clause = "deleted_at IS NULL")
@SQLDelete(sql = "UPDATE products SET deleted_at=NOW() WHERE id=?")
// @SQLDelete: overrides Hibernate's DELETE with custom soft-delete SQL
public class Product {
    @Id @GeneratedValue Long id;
    String name;
    BigDecimal price;
    LocalDateTime deletedAt;
}

public interface ProductRepository extends JpaRepository<Product, Long> {
    // All derived queries automatically add "AND deleted_at IS NULL"
    List<Product> findByCategory(String category);
    // SQL: SELECT * FROM products WHERE category=? AND deleted_at IS NULL

    Optional<Product> findById(Long id);
    // SQL: SELECT * FROM products WHERE id=? AND deleted_at IS NULL
    // Returns empty if soft-deleted — perfect for normal business operations
}

// Service:
@Service
public class ProductService {

    @Transactional
    public void delete(Long id) {
        productRepo.deleteById(id);
        // @SQLDelete intercepts: UPDATE products SET deleted_at=NOW() WHERE id=?
        // No actual DELETE issued
    }
}

// ── APPROACH 2: @FilterDef + @Filter (togglable) ────────────────────────
@FilterDef(
    name = "softDeleteFilter",
    defaultCondition = "deleted_at IS NULL"
)
@Entity
@Filter(name = "softDeleteFilter")
public class Document {
    @Id @GeneratedValue Long id;
    String title;
    String content;
    LocalDateTime deletedAt;
}

@Service
public class DocumentService {

    @PersistenceContext EntityManager em;

    // Normal access — filter enabled (see only non-deleted):
    @Transactional(readOnly = true)
    public List<Document> findActive() {
        em.unwrap(Session.class).enableFilter("softDeleteFilter");
        return em.createQuery("SELECT d FROM Document d", Document.class)
                 .getResultList();
    }

    // Admin access — filter disabled (see everything including deleted):
    @Transactional(readOnly = true)
    public List<Document> findAll_IncludingDeleted() {
        // Filter NOT enabled — sees all documents
        return em.createQuery("SELECT d FROM Document d", Document.class)
                 .getResultList();
    }
}
```

---

### Example 3 — `AttributeConverter` for Complex Types

```java
// ── Money value object ────────────────────────────────────────────────────
public record Money(BigDecimal amount, String currency) {
    @Override public String toString() {
        return amount.toPlainString() + " " + currency;
    }
    public static Money of(BigDecimal amount, String currency) {
        return new Money(amount.setScale(2, RoundingMode.HALF_UP), currency);
    }
    public static Money parse(String value) {
        if (value == null) return null;
        String[] parts = value.split(" ");
        return new Money(new BigDecimal(parts[0]), parts[1]);
    }
}

// Converter: Money → "100.00 USD" in DB:
@Converter
public class MoneyConverter implements AttributeConverter<Money, String> {
    @Override
    public String convertToDatabaseColumn(Money money) {
        return money == null ? null : money.toString();
    }
    @Override
    public Money convertToEntityAttribute(String column) {
        return Money.parse(column);
    }
}

// ── List<String> → comma-separated ───────────────────────────────────────
@Converter
public class StringListConverter
        implements AttributeConverter<List<String>, String> {
    @Override
    public String convertToDatabaseColumn(List<String> list) {
        return (list == null || list.isEmpty()) ? null :
               String.join(",", list);
    }
    @Override
    public List<String> convertToEntityAttribute(String joined) {
        if (joined == null || joined.isBlank()) return new ArrayList<>();
        return new ArrayList<>(Arrays.asList(joined.split(",")));
    }
}

// ── Entity using converters ────────────────────────────────────────────────
@Entity
public class Order {
    @Id @GeneratedValue Long id;
    String customerId;

    @Convert(converter = MoneyConverter.class)
    @Column(name = "total_amount", length = 30)
    Money total;
    // DB: "199.99 USD"

    @Convert(converter = StringListConverter.class)
    @Column(name = "tags")
    List<String> tags;
    // DB: "express,fragile,priority"

    @Convert(converter = StatusConverter.class)  // from earlier example
    Status status;
    // DB: "A" / "I" / "P"
}

// Usage:
@Transactional
public Order createOrder(String customerId, BigDecimal amount, String currency) {
    Order order = new Order();
    order.setCustomerId(customerId);
    order.setTotal(Money.of(amount, currency));
    order.setTags(List.of("express", "fragile"));
    order.setStatus(Status.PENDING);
    return orderRepo.save(order);
    // INSERT: (customer_id, total_amount, tags, status)
    //         ("C123",      "199.99 USD", "express,fragile", "P")
}

// @Query with converted field — use the Java type in JPQL:
@Query("SELECT o FROM Order o WHERE o.status = :status")
List<Order> findByStatus(@Param("status") Status status);
// Hibernate applies converter: Status.ACTIVE → "A" in SQL parameter
```

---

### Example 4 — `@NaturalId` with L2C and Lookup

```java
// Entity:
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    @Id @GeneratedValue Long id;

    @NaturalId
    @Column(unique = true, nullable = false, length = 50)
    String sku;  // Stock Keeping Unit — unique business code

    @NaturalId
    String barcode;  // second natural ID field (composite natural key)

    String name;
    BigDecimal price;

    @Version Integer version;
}

// Repository extending NaturalIdRepository (Spring Data 3.x):
public interface ProductRepository
        extends JpaRepository<Product, Long>,
                NaturalIdRepository<Product, Long> {
}

// Service:
@Service
@Transactional(readOnly = true)
public class ProductLookupService {

    @PersistenceContext EntityManager em;

    // Hibernate session API — most efficient:
    public Optional<Product> findBySku(String sku) {
        Session session = em.unwrap(Session.class);
        Product product = session.byNaturalId(Product.class)
            .using("sku", sku)
            .load();
        // Step 1: Check NaturalId cache: "sku:ABC123" → id=42
        // Step 2: Check entity cache: (Product, 42) → entity
        // Step 3 (if both miss): SELECT id FROM products WHERE sku=?
        //         Then: em.find(Product.class, 42) (which checks entity cache)
        return Optional.ofNullable(product);
    }

    // Composite natural ID:
    public Optional<Product> findBySkuAndBarcode(String sku, String barcode) {
        Session session = em.unwrap(Session.class);
        Product product = session.byNaturalId(Product.class)
            .using("sku", sku)
            .using("barcode", barcode)
            .load();
        return Optional.ofNullable(product);
    }

    // Spring Data NaturalIdRepository (Spring Data 3.x):
    @Autowired ProductRepository productRepo;

    public Optional<Product> findBySkuViaRepository(String sku) {
        return productRepo.findByNaturalId(Map.of("sku", sku));
    }
}
```

---

### Example 5 — `@DynamicUpdate` Performance Pattern

```java
@Entity
@DynamicUpdate
@DynamicInsert
public class UserSession {
    @Id @GeneratedValue Long id;
    String userId;
    String sessionToken;
    String ipAddress;

    // High-frequency updates — only these change on each request:
    LocalDateTime lastActivityAt;
    int requestCount;

    // Rarely change after creation:
    LocalDateTime createdAt;
    String userAgent;
    String deviceType;
    String country;
    String city;
    // ... 10 more metadata fields

    @Version Integer version;
}

@Service
public class SessionService {

    @Transactional
    public void recordActivity(Long sessionId) {
        UserSession session = sessionRepo.findById(sessionId).get();
        session.setLastActivityAt(LocalDateTime.now());
        session.setRequestCount(session.getRequestCount() + 1);
        // @DynamicUpdate: only changed fields in UPDATE:
        // UPDATE user_sessions SET
        //   last_activity_at = ?,
        //   request_count = ?,
        //   version = ?
        // WHERE id = ? AND version = ?
        // NOT: all 15+ metadata columns
    }

    @Transactional
    public UserSession createSession(String userId, String token,
                                     String ip, String userAgent) {
        UserSession session = new UserSession();
        session.setUserId(userId);
        session.setSessionToken(token);
        session.setIpAddress(ip);
        session.setUserAgent(userAgent);
        session.setCreatedAt(LocalDateTime.now());
        session.setLastActivityAt(LocalDateTime.now());
        // country, city, deviceType remain null
        // @DynamicInsert: null columns omitted from INSERT
        // INSERT INTO user_sessions (user_id, session_token, ip_address,
        //   user_agent, created_at, last_activity_at)
        // VALUES (?, ?, ?, ?, ?, ?)
        // DB DEFAULT applies to: country, city, deviceType
        return sessionRepo.save(session);
    }
}
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ: `@Formula` Column Reference**
```java
@Entity
@Table(name = "employees")
public class Employee {
    @Id Long id;
    String firstName;
    String lastName;
    BigDecimal baseSalary;
    BigDecimal bonusPercentage;

    @Formula("base_salary * (1 + bonus_percentage / 100)")
    BigDecimal totalCompensation;
}
```

The `@Formula` expression is correct because:

A) It uses Java field names `baseSalary` and `bonusPercentage`  
B) It uses database column names `base_salary` and `bonus_percentage`  
C) Either field name or column name works — Hibernate maps automatically  
D) It should use `:baseSalary` parameter syntax  

**Answer: B** — `@Formula` expressions are raw SQL fragments embedded in the SELECT clause. They must use **database column names** (snake_case), not Java field names (camelCase). Hibernate inserts the expression verbatim into the generated SQL.

---

**Q2 — Select All That Apply: `@Filter` Behaviour**

Which statements about `@Filter` are TRUE?

A) `@Filter` conditions are automatically applied to `em.find()` calls  
B) `@Filter` must be enabled per-session using `session.enableFilter()`  
C) `@Filter` can accept runtime parameters  
D) `@Filter` applies to JPQL queries when enabled on the session  
E) `@Filter` applies to native SQL queries when enabled  
F) Multiple `@Filter` annotations can be active simultaneously  

**Answer: B, C, D, F**
- A is false: `em.find()` bypasses `@Filter` — only queries are filtered
- E is false: `@Filter` does NOT inject conditions into native SQL

---

**Q3 — `@Where` vs `@Filter`**
```java
@Entity
@Where(clause = "status != 'DELETED'")
public class Article { ... }
```

Which statement is FALSE about this `@Where` annotation?

A) `article.findAll()` adds `AND status != 'DELETED'` to the SQL  
B) `em.find(Article.class, 5L)` also applies the `@Where` condition  
C) `@Where` can be disabled per-session for admin queries  
D) A soft-deleted article with `id=5` returns empty from `findById(5L)`  

**Answer: C** — `@Where` is a STATIC, always-applied condition. It **cannot** be disabled per-session, per-query, or at runtime. Unlike `@Filter`, there is no mechanism to bypass `@Where`. For admin queries that need to see deleted records, you must use native SQL or `@Filter` instead of `@Where`. Note: B is TRUE (unlike `@Filter`, `@Where` DOES apply to `em.find()`).

---

**Q4 — `AttributeConverter` `autoApply`**
```java
@Converter(autoApply = true)
public class CurrencyConverter
        implements AttributeConverter<Currency, String> { ... }

@Entity
public class Invoice {
    @Id Long id;
    Currency invoiceCurrency;    // Field A
    Currency paymentCurrency;    // Field B

    @Convert(converter = CurrencyConverter.class)
    Currency preferredCurrency;  // Field C
}
```

Which fields use `CurrencyConverter`?

A) Only Field C — `@Convert` explicitly declared  
B) Fields A and B — `autoApply=true` applies to all `Currency` fields  
C) All three fields — `autoApply=true` + explicit `@Convert` both apply  
D) No fields — `autoApply=true` only works if no `@Convert` is declared  

**Answer: C** — `autoApply=true` applies the converter to ALL entity fields of type `Currency`, including Fields A and B. Field C has an explicit `@Convert` declaration, which is redundant but not harmful — the same converter is applied. All three fields use `CurrencyConverter`.

---

**Q5 — `@DynamicUpdate` Trade-off**

An entity has 50 columns. In a typical operation, only 3 columns change. Which statement about `@DynamicUpdate` is TRUE?

A) `@DynamicUpdate` improves performance because JDBC statement caching works better with dynamic SQL  
B) `@DynamicUpdate` reduces UPDATE payload but reduces JDBC prepared statement cache effectiveness  
C) `@DynamicUpdate` disables dirty checking, relying only on `@Version` for change detection  
D) `@DynamicUpdate` requires `@SelectBeforeUpdate` to know which columns changed  

**Answer: B** — `@DynamicUpdate` generates UPDATE statements with only the changed columns. This is beneficial (smaller payload, less DB work) but degrades JDBC prepared statement cache: static SQL `UPDATE t SET col1=?,col2=?,...col50=? WHERE id=?` is cached once; dynamic SQL varies based on which columns changed, creating many prepared statement variants. For high-throughput scenarios, this cache thrashing can negate the payload benefit.

---

**Q6 — `@NaturalId` Cache Behavior**

An entity with `@NaturalId String email` is loaded twice in different transactions:
```java
// TX 1:
User u1 = session1.byNaturalId(User.class).using("email","alice@ex.com").load();

// TX 2 (new transaction, entity cache enabled):
User u2 = session2.byNaturalId(User.class).using("email","alice@ex.com").load();
```

How many SQL queries execute across both transactions?

A) 2 — each transaction has its own L1 cache, both hit DB  
B) 1 — L2C NaturalId cache hit on TX 2, no SQL  
C) 0 — NaturalId cache pre-populated at startup  
D) 2 — NaturalId lookup always requires SQL, entity is then cached  

**Answer: B** — TX 1: L1 miss, NaturalId cache miss, entity cache miss → SQL executes → NaturalId cache populated ("alice@ex.com" → id=42) + entity cache populated (User, 42). TX 2: L1 miss (new session), NaturalId cache HIT → id=42, entity cache HIT → entity. Zero SQL in TX 2. Total: 1 SQL query across both transactions.

---

**Q7 — `@Immutable` Behavior**
```java
@Entity
@Immutable
public class AuditEntry {
    @Id @GeneratedValue Long id;
    String action;
    LocalDateTime timestamp;
}

@Transactional
public void test() {
    AuditEntry entry = auditRepo.findById(1L).get();
    entry.setAction("MODIFIED");
    auditRepo.save(entry);  // explicit save call
}
```

What happens at transaction commit?

A) UPDATE SQL executed — `save()` forces the update  
B) No UPDATE SQL — `@Immutable` skips dirty checking, change silently discarded  
C) `UnsupportedOperationException` thrown at `setAction()`  
D) `IllegalStateException` thrown at commit  

**Answer: B** — `@Immutable` tells Hibernate to skip dirty checking for this entity. At commit, Hibernate does not compare the current state to the snapshot because there is no snapshot — dirty checking is disabled. No UPDATE is generated. `save()` on an already-managed `@Immutable` entity is a no-op. The change is silently discarded. No exception is thrown.

---

**Q8 — `@Formula` Subquery Reference**
```java
@Entity
@Table(name = "departments")
public class Department {
    @Id Long id;
    String name;

    @Formula("(SELECT COUNT(*) FROM employees e WHERE e.department_id = id)")
    Long employeeCount;
}
```

What does `id` refer to in the `@Formula` subquery?

A) The Java field name `id` of the `Department` entity  
B) The `departments` table's `id` column in the outer SELECT context  
C) An ambiguous reference that may fail — should be `d.id`  
D) Always zero — subquery cannot reference outer query  

**Answer: B** — In a `@Formula` subquery, unqualified column names refer to the **outer query's table** — the table being mapped by the entity (`departments` in this case). Hibernate includes this SQL fragment in the outer SELECT, where `id` implicitly refers to `departments.id` in that row context. This is standard SQL correlated subquery syntax. Alternatively, you can be explicit: `e.department_id = departments.id` to avoid any ambiguity.

---

## 4️⃣ Trick Analysis

**`@Formula` uses column names not field names (Q1)**:
This trips up developers who think Hibernate translates field names inside `@Formula`. It does not. `@Formula` is a raw SQL fragment inserted verbatim into the SELECT clause. The ORM naming strategy (camelCase → snake_case conversion) only applies to `@Column` mappings, not to `@Formula` content. The database will error with "unknown column 'baseSalary'" if you use Java names.

**`@Filter` does NOT apply to `em.find()` (Q2, A)**:
This is the most commonly tested `@Filter` trap. `em.find()` bypasses all query filters — it goes directly to the identity map then L2C then DB using only the primary key. The filter condition is injected only into queries (JPQL, Criteria, derived queries, collection lazy loading). A soft-deleted entity with `deletedAt` set CAN be loaded via `em.find()` even with an active filter. This is actually useful for admin operations — direct PK lookup always works.

**`@Where` is static and cannot be bypassed (Q3, C)**:
Unlike `@Filter`, `@Where` has no enable/disable mechanism. It is permanently compiled into Hibernate's entity metadata. The only way to see filtered-out records is to use native SQL (which Hibernate does not rewrite) or JDBC directly. This makes `@Where` simpler but inflexible. Important: `@Where` DOES apply to `em.find()` — unlike `@Filter`. This is a key behavioral difference.

**`@Immutable` silently discards changes, no exception (Q7)**:
Developers expect an exception when modifying an `@Immutable` entity. Hibernate does not throw. The setter executes (it's just Java code), the field changes in memory, but at flush time, dirty checking is skipped entirely — Hibernate doesn't even look at the entity's fields. The change disappears at the end of the transaction. This is very different from `READ_ONLY` cache strategy, which DOES throw an exception on flush.

**`@DynamicUpdate` hurts JDBC statement caching (Q5)**:
The common assumption is that `@DynamicUpdate` is always beneficial ("smaller SQL = faster"). The hidden cost is JDBC prepared statement cache fragmentation. A prepared statement is cached by its exact SQL text. With static SQL, one prepared statement covers all updates to an entity. With `@DynamicUpdate`, each unique combination of changed fields produces a different SQL string and a different cache entry. Under high load with many field combinations, the prepared statement cache fills with variants, evicting more valuable cached statements.

**`@NaturalId` two-step cache resolution (Q6)**:
`@NaturalId` maintains a separate cache region mapping natural ID values to primary keys. The lookup chain is: NaturalId cache (natural value → PK) → Entity cache (PK → entity). Both steps can be served from cache, enabling zero-SQL lookups in warm cache scenarios. This is Hibernate's optimization for "find by email" patterns — far more efficient than a full table scan or even an indexed query when the L2C is warm.

---

## 5️⃣ Summary Sheet

### Feature Quick Reference

| Feature | Purpose | Key Constraint |
|---|---|---|
| `@Formula` | Computed/virtual SQL column | SQL column names, read-only, DB-dialect-specific |
| `@Filter` | Toggleable runtime query filter | Enable per-session; bypassed by `em.find()` |
| `@Where` | Static always-on SQL filter | Cannot disable; applied to `em.find()` too |
| `@NaturalId` | Business key with cache support | Two-step cache: NaturalId→PK→entity |
| `@Immutable` | Skip dirty checking | Silently discards changes, no exception |
| `@DynamicUpdate` | Only changed columns in UPDATE | Hurts JDBC prepared statement cache |
| `@DynamicInsert` | Only non-null columns in INSERT | Allows DB DEFAULT values to apply |
| `@SelectBeforeUpdate` | SELECT before merge() | Extra SELECT per merge — use sparingly |
| `AttributeConverter` | Custom Java↔DB type mapping | `autoApply` affects all fields of that type |
| `@BatchSize` | Batch N+1 collection loading | `ceil(N/batchSize)` queries instead of N |
| `@Subselect` | Entity backed by SQL subquery | Combine with `@Synchronize` and `@Immutable` |

### `@Filter` vs `@Where` Decision Matrix

```
Use @Where when:
  ✓ Condition should ALWAYS apply (soft delete visible to no one by default)
  ✓ Simple static condition
  ✓ Should apply to em.find() too
  ✗ Cannot be disabled for admin access

Use @Filter when:
  ✓ Condition should be OPTIONAL (some callers see all, some see filtered)
  ✓ Runtime parameter needed (tenant ID, status value)
  ✓ Admin needs unfiltered access
  ✗ em.find() bypass is acceptable
```

### `AttributeConverter` Rules

```
autoApply = true:   applies to ALL fields of type X, no @Convert needed
autoApply = false:  requires explicit @Convert(converter=...) per field

Converter is called:
  Write path: convertToDatabaseColumn() before INSERT/UPDATE
  Read path:  convertToEntityAttribute() after SELECT

Works with:
  ✓ JPQL queries (parameters are converted automatically)
  ✓ Dirty checking (comparison uses Java type)
  ✗ Native SQL parameters (must pass DB type manually)
  ✗ @Formula expressions (raw SQL, no conversion)
```

### Key Rules

```
1.  @Formula uses RAW SQL (table column names, not Java field names)
2.  @Formula is evaluated on EVERY entity SELECT — subquery @Formula = N subqueries
3.  @Filter must be enabled via session.enableFilter() — disabled by default
4.  @Filter bypassed by em.find() — only affects JPQL/Criteria/collection queries
5.  @Filter does NOT apply to native SQL queries
6.  @Where is STATIC — cannot be disabled at runtime
7.  @Where DOES apply to em.find() (unlike @Filter)
8.  @NaturalId: two-step cache: NaturalId value → PK → entity (both cacheable)
9.  @NaturalId(mutable=false): exception if you change the natural ID field
10. @Immutable: silently discards modifications — NO exception thrown
11. @DynamicUpdate: UPDATE includes only changed columns
12. @DynamicUpdate: hurts JDBC prepared statement caching (many SQL variants)
13. @DynamicInsert: omits null columns from INSERT — allows DB DEFAULT to apply
14. @SelectBeforeUpdate: forces SELECT before merge() — extra query per merge
15. AttributeConverter autoApply=true: applies to ALL fields of that Java type
16. AttributeConverter with JPQL: @Param values automatically converted
17. @BatchSize(size=N): reduces N+1 to ceil(N/N_batch) queries
18. @Subselect + @Synchronize: flush named tables before querying subselect entity
19. @Formula with subquery: unqualified column names refer to OUTER query's table
20. @SQLDelete: overrides Hibernate's DELETE with custom SQL (soft delete pattern)
```

---
