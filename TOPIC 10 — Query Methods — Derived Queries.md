# 🏛️ TOPIC 10 — Query Methods — Derived Queries

---

## 1️⃣ Conceptual Explanation

### What Derived Queries Are — The Parser Contract

Derived queries are Spring Data's most visually distinctive feature: repository methods whose implementation is **generated entirely from the method name**. No JPQL, no SQL, no implementation code — just a method name that follows a strict grammar.

```java
List<User> findByLastNameAndAgeGreaterThan(String lastName, int age);
// Spring Data parses this name → builds JPQL → executes at runtime
// JPQL: SELECT u FROM User u WHERE u.lastName = ?1 AND u.age > ?2
// SQL:  SELECT * FROM users WHERE last_name=? AND age>?
```

This is not magic — it is **deterministic grammar parsing**. Spring Data uses a component called `PartTree` to parse the method name into an Abstract Syntax Tree (AST), which is then compiled into JPQL by `JpaQueryCreator`.

Understanding the grammar means you can predict with certainty what any method name will produce — and what it will fail on.

---

### The `PartTree` Parser — Internal Architecture

`PartTree` splits a method name into two major parts separated by the first occurrence of `By`:

```
findDistinctTop3ByLastNameAndAgeGreaterThanOrderByCreatedAtDesc
│                │                                              │
│                └─ PREDICATE (criteria) ─────────────────────►│
└── SUBJECT ────►│                                              │
                 By                                             
                 
SUBJECT: findDistinctTop3
PREDICATE: LastNameAndAgeGreaterThanOrderByCreatedAtDesc
```

**Subject parsing:**

```
Subject structure: [keyword][Distinct][TopN/FirstN]
                   [find/read/get/query/search/stream/count/exists/delete/remove]

find    → SELECT query
read    → SELECT query (alias for find)
get     → SELECT query (alias for find)
query   → SELECT query (alias for find)
search  → SELECT query (alias for find)
stream  → SELECT query, returns Stream<T>
count   → COUNT query → returns long
exists  → EXISTS query → returns boolean
delete  → DELETE query → returns void or long (count deleted)
remove  → DELETE query (alias for delete)
```

**Predicate parsing — the `Part` structure:**

Each `Part` in the predicate represents one condition. Parts are combined with `And` and `Or`:

```
LastNameAndAgeGreaterThan
│         │  │
│         │  └─ Keyword: GreaterThan
│         └─ Connector: And
└─ Property: lastName

Part 1: Property=lastName,  Keyword=(none → SIMPLE_PROPERTY → equals)
Part 2: Property=age,       Keyword=GreaterThan
```

The parser uses **property resolution** to validate that each property path exists in the entity class. This happens at startup — invalid property names cause `PropertyReferenceException`.

---

### Subject Keywords — Complete Reference

#### Find/Read/Get/Query/Search Keywords

```java
// All equivalent — return T or List<T> or Optional<T>:
findBy...
readBy...
getBy...
queryBy...
searchBy...
streamBy...  // returns Stream<T> — must be used inside @Transactional

// Examples:
Optional<User> findByEmail(String email);
User getByUsername(String username);
Stream<Product> streamByCategory(String category); // @Transactional required
```

#### `Distinct` — Deduplication

```java
List<User> findDistinctByLastName(String lastName);
// JPQL: SELECT DISTINCT u FROM User u WHERE u.lastName = ?1
// SQL:  SELECT DISTINCT * FROM users WHERE last_name = ?

// Note: DISTINCT in JPQL applies at the Java object level
// With collections JOIN FETCH: use passDistinctThrough=false hint
```

#### `Top` / `First` — Result Limiting

```java
User findFirstByOrderByCreatedAtDesc();     // first result
User findTopByOrderByScoreDesc();           // same as First

List<User> findTop3ByLastName(String name); // top 3 results
List<User> findFirst10ByStatus(Status s);   // first 10 results

Optional<User> findFirstByStatus(Status s); // Optional wrapping

// Internally: setMaxResults(N) on TypedQuery
// SQL: SELECT * FROM users WHERE status=? ORDER BY ... LIMIT 3
```

#### `count`, `exists`, `delete`

```java
long countByStatus(Status status);
// JPQL: SELECT COUNT(u) FROM User u WHERE u.status = ?1
// SQL:  SELECT COUNT(*) FROM users WHERE status=?

boolean existsByEmail(String email);
// JPQL: SELECT COUNT(u) > 0 FROM User u WHERE u.email = ?1
// SQL:  SELECT COUNT(*) FROM users WHERE email=? (or EXISTS subquery)

long deleteByStatus(Status status);
// JPQL: DELETE FROM User u WHERE u.status = ?1
// SQL:  DELETE FROM users WHERE status=?
// NOTE: requires @Transactional — see detailed discussion below

void removeByCreatedAtBefore(LocalDateTime cutoff);
// Same as delete
```

**`delete`/`remove` derived queries — the SELECT+DELETE trap:**

```java
// Derived delete DOES load entities first by default!
// Spring Data derived delete: loads entities → calls em.remove() per entity
// This triggers cascade and callbacks — but is slower

// Contrast with @Modifying @Query DELETE — direct DELETE without loading

// When to use which:
// Derived delete: when cascade / @PreRemove callbacks are needed
// @Modifying DELETE: when you need raw performance and don't need cascade
```

---

### Predicate Keywords — Complete Grammar Reference

#### Equality and Comparison

```java
// Equality (default — no keyword needed):
findByFirstName(String name)
// JPQL: WHERE u.firstName = ?1

findByFirstNameIs(String name)       // explicit Is — same as equality
findByFirstNameEquals(String name)   // explicit Equals — same

// Negation:
findByFirstNameNot(String name)
findByFirstNameIsNot(String name)
// JPQL: WHERE u.firstName != ?1  (or <> depending on dialect)

// Comparison operators:
findByAgeLessThan(int age)            // WHERE age < ?1
findByAgeLessThanEqual(int age)       // WHERE age <= ?1
findByAgeGreaterThan(int age)         // WHERE age > ?1
findByAgeGreaterThanEqual(int age)    // WHERE age >= ?1

// Between:
findByAgeBetween(int min, int max)    // WHERE age BETWEEN ?1 AND ?2
// Note: two parameters for Between

// Null checks:
findByMiddleNameIsNull()              // WHERE middle_name IS NULL — no parameter!
findByMiddleNameIsNotNull()           // WHERE middle_name IS NOT NULL — no parameter!
findByMiddleNameNull()                // alias for IsNull
findByMiddleNameNotNull()             // alias for IsNotNull
```

#### String-Specific Keywords

```java
// Like (manual wildcard placement):
findByNameLike(String pattern)        // WHERE name LIKE ?1
// Caller must include %: findByNameLike("%Smith%")

// Containing (auto-wraps with %):
findByNameContaining(String fragment) // WHERE name LIKE %?1%
findByNameContains(String fragment)   // alias

// StartingWith / EndingWith:
findByNameStartingWith(String prefix) // WHERE name LIKE ?1%
findByNameStartsWith(String prefix)   // alias
findByNameEndingWith(String suffix)   // WHERE name LIKE %?1
findByNameEndsWith(String suffix)     // alias

// Case insensitivity:
findByNameIgnoreCase(String name)         // WHERE LOWER(name) = LOWER(?1)
findByNameContainingIgnoreCase(String s)  // WHERE LOWER(name) LIKE LOWER(%?1%)
// Applicable to any String predicate keyword

// AllIgnoreCase — applies to ALL String parts:
findByFirstNameAndLastNameAllIgnoreCase(String first, String last)
// WHERE LOWER(first_name)=LOWER(?1) AND LOWER(last_name)=LOWER(?2)
```

#### Collection Keywords

```java
// In / NotIn:
findByStatusIn(Collection<Status> statuses)
// JPQL: WHERE u.status IN ?1
// SQL:  WHERE status IN (?,?,?)
// Parameter: Collection, List, Set, or array

findByStatusNotIn(Collection<Status> statuses)
// JPQL: WHERE u.status NOT IN ?1

// True / False (for boolean fields):
findByActiveTrue()                    // WHERE active = true  — no parameter
findByActiveFalse()                   // WHERE active = false — no parameter
findByActiveIsTrue()                  // alias
```

---

### Connector Keywords — `And` / `Or`

```java
// And — all conditions must be true:
findByFirstNameAndLastName(String first, String last)
// WHERE first_name=?1 AND last_name=?2

// Or — any condition must be true:
findByFirstNameOrLastName(String first, String last)
// WHERE first_name=?1 OR last_name=?2

// Mixed:
findByFirstNameAndLastNameOrEmail(String first, String last, String email)
// WHERE (first_name=?1 AND last_name=?2) OR email=?3
// AND binds tighter than OR — same as Java operator precedence
// ALWAYS test mixed And/Or carefully
```

**The And/Or precedence trap:**

```java
findByAAndBOrC(...)
// Parsed as: (A AND B) OR C
// NOT as: A AND (B OR C)

// If you need: A AND (B OR C) — use @Query instead:
@Query("SELECT u FROM User u WHERE u.a = :a AND (u.b = :b OR u.c = :c)")
List<User> findByAAndBOrC(@Param("a") String a, @Param("b") String b, @Param("c") String c);
```

---

### `OrderBy` — Sorting in Method Names

```java
// Ascending (default):
findByStatusOrderByCreatedAt(Status s)
// ORDER BY created_at ASC

// Explicit ascending:
findByStatusOrderByCreatedAtAsc(Status s)

// Descending:
findByStatusOrderByCreatedAtDesc(Status s)
// ORDER BY created_at DESC

// Multiple sort fields:
findByStatusOrderByLastNameAscFirstNameDesc(Status s)
// ORDER BY last_name ASC, first_name DESC

// Sort + Pageable — dynamic sort overrides OrderBy:
findByStatus(Status s, Pageable pageable)
// Pageable.sort overrides the static OrderBy

// Sort parameter only (no pagination):
findByStatus(Status s, Sort sort)
// Applies dynamic Sort — no static OrderBy needed

// Best practice: use Pageable/Sort parameters for dynamic ordering
// Use static OrderBy only when order is truly fixed
```

---

### Property Path Resolution — Nested Properties

Spring Data resolves property paths using dots in method names (implicit) or explicit path notation:

```java
// Nested property access:
findByAddressCity(String city)
// Spring Data looks for: user.address.city
// JPQL: WHERE u.address.city = ?1
// Generates implicit JOIN if needed (for embedded: same table; for association: JOIN)

findByOrdersCustomerName(String customerName)
// Resolves: user.orders → List<Order>, then order.customer → Customer, then .name
// JPQL: WHERE o.customer.name = ?1 (with implicit JOIN)

// Ambiguity resolution:
// If entity has: addressCity (one field) AND address.city (nested)
// Spring Data tries longest match first, then shorter paths
// To force: use _ as separator for disambiguation:
findByAddress_City(String city)    // explicit: address.city (nested property)
findByAddressCity(String city)     // might resolve to addressCity field if it exists
```

**Property path validation at startup:**

```java
// This fails at startup with PropertyReferenceException:
findByNonExistentField(String value)
// "No property 'nonExistentField' found for type 'User'"

// This fails for wrong type:
findByAgeContaining(String fragment)
// age is int, Containing requires String → PropertyTypeMismatchException at startup
```

---

### Return Type Variations

Spring Data supports a rich set of return types for any given query method:

```java
// Single result:
User findByEmail(String email);
// Throws: IncorrectResultSizeDataAccessException if > 1 result
// Throws: nothing (returns null) if 0 results — dangerous!

Optional<User> findByEmail(String email);
// Returns Optional.empty() if 0 results
// Throws IncorrectResultSizeDataAccessException if > 1 result
// ALWAYS prefer Optional for single-result queries

// Multiple results:
List<User> findByLastName(String lastName);
Iterable<User> findByLastName(String lastName);
Collection<User> findByLastName(String lastName);

// With pagination:
Page<User> findByStatus(Status status, Pageable pageable);
Slice<User> findByStatus(Status status, Pageable pageable);
// Page: fires count query, knows total elements/pages
// Slice: NO count query, only knows if there's a next slice → faster

// Streaming:
Stream<User> findByStatus(Status status);
// Must use @Transactional — stream must be closed/consumed within session
// Usage: try(Stream<User> s = repo.findByStatus(ACTIVE)) { ... }

// Reactive (non-JPA, for reference):
Flux<User> findByStatus(Status status); // R2DBC, not JPA

// Projections:
List<UserNameOnly> findByStatus(Status status);
// Returns proxy implementing UserNameOnly interface
// SQL selects only projected columns

// Futures / CompletableFuture (async):
@Async
CompletableFuture<List<User>> findByStatus(Status status);
// Executed in a separate thread
```

**`Page<T>` vs `Slice<T>` — deep comparison:**

```java
Page<User> page = userRepo.findByStatus(ACTIVE, PageRequest.of(0, 20));
// SQL 1: SELECT * FROM users WHERE status=? LIMIT 20 OFFSET 0
// SQL 2: SELECT COUNT(*) FROM users WHERE status=? ← always fired
page.getTotalElements(); // total count across all pages
page.getTotalPages();    // total page count
page.hasNext();          // true/false

Slice<User> slice = userRepo.findByStatus(ACTIVE, PageRequest.of(0, 20));
// SQL: SELECT * FROM users WHERE status=? LIMIT 21 OFFSET 0
//                                              ↑ fetches 1 extra to detect hasNext
// NO count query
slice.hasNext();          // true if 21 rows returned (more exist)
slice.getTotalElements(); // NOT available — UnsupportedOperationException!
slice.getTotalPages();    // NOT available

// Slice is faster when total count is not needed (e.g., "load more" buttons)
// Page is needed when showing total count / page numbers
```

---

### Parameter Binding — `@Param` and Positional

```java
// Derived queries: parameters bound positionally and automatically:
List<User> findByFirstNameAndLastName(String firstName, String lastName);
// firstName → ?1, lastName → ?2
// No @Param needed — binding is automatic and positional

// @Param is NOT needed for derived queries:
List<User> findByFirstNameAndLastName(
    @Param("firstName") String firstName,  // redundant but not harmful
    @Param("lastName") String lastName
);

// @Param IS needed for @Query:
@Query("SELECT u FROM User u WHERE u.firstName = :first AND u.lastName = :last")
List<User> findByNames(@Param("first") String firstName, 
                       @Param("last") String lastName);
// :first must match @Param("first")

// Positional in @Query (less readable):
@Query("SELECT u FROM User u WHERE u.firstName = ?1 AND u.lastName = ?2")
List<User> findByNamesPositional(String firstName, String lastName);
```

---

### Limiting and Pagination in Derived Queries

```java
// Hard limit via Top/First:
List<Product> findTop10ByCategory(String category);
// Internally: query.setMaxResults(10)
// SQL: SELECT * FROM products WHERE category=? LIMIT 10

// Dynamic limiting via Pageable:
List<Product> findByCategory(String category, Pageable pageable);
// Caller controls page/size/sort:
PageRequest.of(0, 10, Sort.by("price").descending())

// Combination — limit + sort in name:
List<Product> findTop5ByCategoryOrderByPriceAsc(String category);
// SQL: SELECT * FROM products WHERE category=? ORDER BY price ASC LIMIT 5

// WARNING: Top/First with Pageable — Top wins:
List<Product> findTop5ByCategory(String category, Pageable pageable);
// setMaxResults(5) applied — Pageable.pageSize is IGNORED
// Only sort from Pageable is used
// This is confusing — avoid combining Top/First with Pageable
```

---

### `PartTree` AST — Internal Generation Sequence

This is the complete internal flow from method name to JPQL to SQL:

```
Step 1: Method name parsing
  "findDistinctTop3ByLastNameAndAgeGreaterThanOrderByCreatedAtDesc"
  
  PartTree:
    subject.distinct = true
    subject.maxResults = 3
    predicate.parts = [
      Part(property=lastName, type=SIMPLE_PROPERTY),
      Part(property=age, type=GREATER_THAN)
    ]
    predicate.orParts = [above parts in one OrPart]
    sort = Sort(createdAt, DESC)

Step 2: JpaQueryCreator builds JPQL
  CriteriaBuilder cb = em.getCriteriaBuilder();
  CriteriaQuery<User> cq = cb.createQuery(User.class);
  Root<User> root = cq.from(User.class);
  
  // Translate each Part to Predicate:
  Predicate p1 = cb.equal(root.get("lastName"), param1);
  Predicate p2 = cb.greaterThan(root.get("age"), param2);
  
  cq.where(cb.and(p1, p2));
  cq.orderBy(cb.desc(root.get("createdAt")));
  cq.distinct(true);
  
  TypedQuery<User> query = em.createQuery(cq);
  query.setMaxResults(3);

Step 3: Hibernate compiles CriteriaQuery to SQL
  SELECT DISTINCT u.id, u.last_name, u.age, u.created_at
  FROM users u
  WHERE u.last_name = ?
    AND u.age > ?
  ORDER BY u.created_at DESC
  LIMIT 3

Step 4: JDBC execution and result mapping
```

---

### Limits of Derived Queries

Derived queries cannot express:

```
❌ GROUP BY / HAVING
❌ Subqueries
❌ Aggregate functions (SUM, AVG, MAX, MIN) — except count
❌ UNION / UNION ALL
❌ Complex OR logic with grouping: A AND (B OR C)
❌ JOIN with ON conditions (beyond property path traversal)
❌ Window functions
❌ CASE expressions
❌ Updates or complex deletes with JOINs
❌ Native SQL features (JSONB, full-text, etc.)

→ Use @Query for any of these
```

---

## 2️⃣ Code Examples

### Example 1 — Comprehensive Derived Query Reference

```java
@Entity
public class Employee {
    @Id @GeneratedValue Long id;
    String firstName;
    String lastName;
    String email;
    int age;
    boolean active;
    BigDecimal salary;
    LocalDateTime hiredAt;
    EmploymentStatus status;
    String department;
    
    @ManyToOne Address address;  // has city, state, country fields
}

public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    
    // ── EQUALITY ──────────────────────────────────────────────────
    Optional<Employee> findByEmail(String email);
    List<Employee> findByDepartment(String department);
    List<Employee> findByDepartmentIs(String department);           // same
    List<Employee> findByDepartmentEquals(String department);       // same
    List<Employee> findByDepartmentNot(String department);
    
    // ── COMPARISON ───────────────────────────────────────────────
    List<Employee> findByAgeGreaterThan(int age);
    List<Employee> findByAgeGreaterThanEqual(int age);
    List<Employee> findByAgeLessThan(int age);
    List<Employee> findByAgeLessThanEqual(int age);
    List<Employee> findByAgeBetween(int min, int max);
    List<Employee> findBySalaryGreaterThan(BigDecimal salary);
    
    // ── NULL CHECKS ───────────────────────────────────────────────
    List<Employee> findByLastNameIsNull();           // no parameter
    List<Employee> findByLastNameIsNotNull();        // no parameter
    
    // ── BOOLEAN ───────────────────────────────────────────────────
    List<Employee> findByActiveTrue();              // no parameter
    List<Employee> findByActiveFalse();             // no parameter
    
    // ── STRING ────────────────────────────────────────────────────
    List<Employee> findByFirstNameContaining(String fragment);
    List<Employee> findByFirstNameStartingWith(String prefix);
    List<Employee> findByFirstNameEndingWith(String suffix);
    List<Employee> findByFirstNameLike(String pattern);  // manual %
    List<Employee> findByFirstNameIgnoreCase(String name);
    List<Employee> findByFirstNameContainingIgnoreCase(String fragment);
    
    // ── COLLECTION ────────────────────────────────────────────────
    List<Employee> findByStatusIn(Collection<EmploymentStatus> statuses);
    List<Employee> findByDepartmentIn(List<String> departments);
    List<Employee> findByStatusNotIn(Set<EmploymentStatus> statuses);
    
    // ── COMPOUND ─────────────────────────────────────────────────
    List<Employee> findByDepartmentAndActive(String dept, boolean active);
    List<Employee> findByDepartmentAndAgeGreaterThan(String dept, int age);
    List<Employee> findByFirstNameOrLastName(String first, String last);
    List<Employee> findByActiveAndSalaryGreaterThan(boolean active, BigDecimal salary);
    
    // ── NESTED PROPERTY ─────────────────────────────────────────
    List<Employee> findByAddressCity(String city);
    List<Employee> findByAddress_State(String state);  // explicit path
    
    // ── ORDERING ─────────────────────────────────────────────────
    List<Employee> findByDepartmentOrderBySalaryDesc(String dept);
    List<Employee> findByActiveOrderByLastNameAscFirstNameAsc(boolean active);
    
    // ── LIMITING ──────────────────────────────────────────────────
    Employee findFirstByDepartmentOrderBySalaryDesc(String dept);
    List<Employee> findTop5ByDepartmentOrderBySalaryDesc(String dept);
    Optional<Employee> findFirstByEmailIgnoreCase(String email);
    
    // ── COUNT / EXISTS ────────────────────────────────────────────
    long countByDepartment(String dept);
    long countByActiveTrue();
    boolean existsByEmail(String email);
    boolean existsByDepartmentAndActive(String dept, boolean active);
    
    // ── DELETE ────────────────────────────────────────────────────
    long deleteByActiveFalse();
    void removeByHiredAtBefore(LocalDateTime date);
    
    // ── WITH PAGEABLE ─────────────────────────────────────────────
    Page<Employee> findByDepartment(String dept, Pageable pageable);
    Slice<Employee> findByActive(boolean active, Pageable pageable);
    List<Employee> findByDepartment(String dept, Sort sort);
    
    // ── DISTINCT ─────────────────────────────────────────────────
    List<Employee> findDistinctByLastName(String lastName);
    long countDistinctByDepartment(String dept);  // SELECT COUNT(DISTINCT dept)
}
```

---

### Example 2 — JPQL Generated for Complex Method Names

```java
// Method: findTop3ByDepartmentAndAgeGreaterThanOrderBySalaryDescLastNameAsc
// Generated JPQL:
// SELECT e FROM Employee e 
// WHERE e.department = ?1 AND e.age > ?2 
// ORDER BY e.salary DESC, e.lastName ASC
// (with setMaxResults(3))

// SQL:
// SELECT * FROM employees
// WHERE department=? AND age>?
// ORDER BY salary DESC, last_name ASC
// LIMIT 3

// Method: findByFirstNameContainingIgnoreCaseAndActiveTrue
// Generated JPQL:
// SELECT e FROM Employee e
// WHERE LOWER(e.firstName) LIKE LOWER(?1)  -- %fragment% in LOWER
// AND e.active = true

// SQL:
// SELECT * FROM employees
// WHERE LOWER(first_name) LIKE LOWER('%?%')
// AND active=true

// Method: findByStatusInAndAgeBetweenOrderByLastNameAsc
// Generated JPQL:
// SELECT e FROM Employee e
// WHERE e.status IN ?1 AND e.age BETWEEN ?2 AND ?3
// ORDER BY e.lastName ASC

// SQL:
// SELECT * FROM employees
// WHERE status IN (?,?,?) AND age BETWEEN ? AND ?
// ORDER BY last_name ASC
```

---

### Example 3 — Return Type Variations with Same Query

```java
public interface ProductRepository extends JpaRepository<Product, Long> {
    
    // Returns null if not found (dangerous — avoid):
    Product findBySkuCode(String sku);
    
    // Returns Optional (preferred for single result):
    Optional<Product> findBySkuCode(String sku);
    
    // Returns List (safe for potentially many results):
    List<Product> findByCategory(String category);
    
    // Page with count query:
    Page<Product> findByCategory(String category, Pageable pageable);
    // Fires: SELECT * FROM products WHERE category=? LIMIT ? OFFSET ?
    //        SELECT COUNT(*) FROM products WHERE category=?
    
    // Slice — no count query (faster for infinite scroll):
    Slice<Product> findByCategory(String category, Pageable pageable);
    // Fires: SELECT * FROM products WHERE category=? LIMIT pageSize+1 OFFSET ?
    // No count query
    
    // Stream — process one by one without loading all into memory:
    @Transactional(readOnly = true)
    Stream<Product> findByCategory(String category);
    // MUST be consumed within @Transactional scope
    // MUST be closed after use: try-with-resources
    
    // CompletableFuture — async execution:
    @Async
    CompletableFuture<List<Product>> findByCategory(String category);
    
    // Interface projection — only selected columns:
    List<ProductNameAndPrice> findByCategory(String category);
    // SQL: SELECT name, price FROM products WHERE category=?
    // (Spring Data infers columns from projection interface)
}

// Projection interface:
public interface ProductNameAndPrice {
    String getName();
    BigDecimal getPrice();
}

// Stream usage:
@Transactional(readOnly = true)
public void processAllActiveProducts() {
    try (Stream<Product> products = productRepo.findByActiveTrue()) {
        products
            .filter(p -> p.getStock() > 0)
            .forEach(this::processProduct);
    } // stream closed here — important to prevent resource leak
}
```

---

### Example 4 — Property Path Ambiguity and `_` Separator

```java
@Entity
public class Person {
    @Id Long id;
    String addressZip;       // single field named "addressZip"
    
    @Embedded
    Address address;         // has a field named "zip"
}

@Embeddable
public class Address {
    String zip;
    String city;
}

// AMBIGUOUS method name:
List<Person> findByAddressZip(String zip);
// Spring Data tries longest match first:
// Option A: person.address → Address, then .zip → String (nested property)
// Option B: person.addressZip → String (direct field)
// If both exist: Spring Data prioritizes longest property path
// In this case: "address" + "zip" → address.zip (nested property)
// RESULT: WHERE address = ? is interpreted as address.zip — ambiguous!

// EXPLICIT resolution with underscore:
List<Person> findByAddress_Zip(String zip);    // explicitly: address.zip
List<Person> findByAddressZip(String zip);     // addressZip field directly

// Rule: use _ whenever property path is ambiguous
// Best practice: name entity fields unambiguously
```

---

### Example 5 — Edge Cases That Break Derived Queries

```java
// EDGE CASE 1: findBy with no criteria — findAll()
List<User> findBy();
// Valid! Equivalent to findAll() — no WHERE clause
// Rarely used but valid

// EDGE CASE 2: Boolean parameter vs BooleanLiteral keyword:
List<User> findByActive(boolean active);     // WHERE active = ?  (parameter)
List<User> findByActiveTrue();               // WHERE active = true (literal, no param)
// They look similar but one takes a parameter, one doesn't

// EDGE CASE 3: IsNull — no parameter:
List<User> findByMiddleNameIsNull();
// No parameter needed — WHERE middle_name IS NULL
// Incorrect: findByMiddleNameIsNull(Object ignored) — extra param breaks it

// EDGE CASE 4: Between — two parameters required:
List<User> findByAgeBetween(int min, int max); // ✓ correct
List<User> findByAgeBetween(int min);          // ✗ wrong — Between needs 2 params
// Spring Data: PropertyTypeMismatchException at startup

// EDGE CASE 5: In with single value:
List<User> findByStatusIn(Collection<Status> statuses);
// If caller passes Set.of(ACTIVE): WHERE status IN (?)
// If caller passes empty collection: WHERE status IN ()
// DANGER: empty IN clause → DB-specific behavior or exception
// Guard: if (statuses.isEmpty()) return List.of();

// EDGE CASE 6: Stream return type outside @Transactional:
Stream<User> findByActive(boolean active); // method itself doesn't need @Transactional
// But CALLER must be @Transactional or stream must be consumed immediately
// If Hibernate Session closes before stream is fully consumed: LazyInitializationException

// EDGE CASE 7: findBy with Pageable — count query:
Page<User> findByStatus(Status status, Pageable pageable);
// Spring Data auto-generates count query:
// SELECT COUNT(u) FROM User u WHERE u.status = ?1
// This count query runs EVERY PAGE request — can be expensive
// Custom count query: @Query(countQuery="SELECT COUNT(u.id) FROM User u WHERE ...")
```

---

### Example 6 — Complex Scenario: Debugging Wrong Method Names

```java
// SCENARIO: Developer wants employees in a department, hired after date, sorted by name

// ATTEMPT 1: Wrong order of OrderBy and predicate:
List<Employee> findByOrderByHiredAtAndDepartment(String dept, LocalDateTime date);
// ✗ Parsed as: findBy (all) OrderBy (hiredAt AND department?) — wrong!
// Spring Data tries to order by "hiredAt" then by "department" and "dept" becomes extra param

// ATTEMPT 2: OrderBy must come after predicate parts:
List<Employee> findByDepartmentAndHiredAtAfterOrderByLastNameAsc(
    String dept, LocalDateTime date);
// ✓ Correct parse:
// findBy: subject
// Department: Part(property=department, keyword=SIMPLE_PROPERTY)
// And: connector
// HiredAt: property part
// After: keyword (AFTER → GreaterThan for dates)
// OrderBy: sort begins
// LastName: sort property
// Asc: sort direction
// JPQL: SELECT e FROM Employee e 
//       WHERE e.department=?1 AND e.hiredAt > ?2 
//       ORDER BY e.lastName ASC

// ATTEMPT 3: What about employees active=true, salary > X, in departments A or B?
// Derived query — AND binds before OR, so:
List<Employee> findByActiveTrueAndSalaryGreaterThanAndDepartmentIn(
    BigDecimal salary, Collection<String> departments);
// JPQL: WHERE active=true AND salary>?1 AND department IN ?2
// ✓ This works

// But: active=true AND (salary>X OR department IN depts) — NOT possible:
// → Use @Query
@Query("SELECT e FROM Employee e WHERE e.active=true AND (e.salary>:s OR e.department IN :depts)")
List<Employee> findActiveWithSalaryOrDepartment(
    @Param("s") BigDecimal salary, @Param("depts") List<String> depts);
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ**
What is the correct method name for "find all products where price is between min and max, ordered by name ascending"?

A) `findByPriceBetweenMinAndMaxOrderByNameAsc(BigDecimal min, BigDecimal max)`  
B) `findByPriceBetweenOrderByNameAsc(BigDecimal min, BigDecimal max)`  
C) `findByPriceFromToOrderByNameAscending(BigDecimal min, BigDecimal max)`  
D) `findAllByPriceRangeOrderByNameAsc(BigDecimal min, BigDecimal max)`  

**Answer: B** — `Between` is a keyword that takes exactly two consecutive parameters. The method name is `findByPriceBetweenOrderByNameAsc` with parameters `(BigDecimal min, BigDecimal max)`. Option A incorrectly includes `MinAndMax` in the name. C and D use non-existent keywords (`FromTo`, `AllBy...Range`).

---

**Q2 — Select All That Apply**
Which derived query method names are valid and correctly use Spring Data keywords? (Select all)

A) `findByAgeIsNull()`  
B) `findByActiveIsTrue()`  
C) `findByNameContains(String fragment)`  
D) `findByEmailNotEmpty()`  
E) `findByStatusIn(List<Status> statuses)`  
F) `findByCreatedAtAfter(LocalDateTime date)`  

**Answer: A, B, C, E, F**
- D is invalid: `NotEmpty` is not a standard Spring Data keyword (it exists in Bean Validation but not as a query keyword). Use `IsNotNull` instead, or `@Query`.

---

**Q3 — SQL Prediction**
```java
List<User> findByFirstNameIgnoreCaseAndActiveTrue();
```
What SQL is generated?

A) `SELECT * FROM users WHERE first_name LIKE ? AND active=true`  
B) `SELECT * FROM users WHERE LOWER(first_name)=LOWER(?) AND active=true`  
C) Compilation error — `IgnoreCase` cannot be combined with `True`  
D) `SELECT * FROM users WHERE first_name=? AND active=1`  

**Answer: B** — `IgnoreCase` applies to the `firstName` condition, wrapping both sides with `LOWER()`. `True` is a keyword that adds `AND active=true` with no parameter. Note: this method has only ONE parameter (the firstName). No parameter for active.

Wait — actually this method is `findByFirstNameIgnoreCaseAndActiveTrue()` — but what is the firstName parameter? It has `IgnoreCase` but no parameter listed! The correct interpretation:

```java
List<User> findByFirstNameIgnoreCaseAndActiveTrue(String firstName);
// firstName is the parameter, activeTrue has no parameter
// SQL: WHERE LOWER(first_name)=LOWER(?) AND active=true
```

**Answer: B** — with `firstName` as the single parameter.

---

**Q4 — Parameter Count Trap**
```java
List<Order> findByStatusAndCreatedAtBetween(OrderStatus status, 
                                             LocalDateTime start, 
                                             LocalDateTime end);
```
How many method parameters are required?

A) 1 — `Between` uses a range object  
B) 2 — status + one range parameter  
C) 3 — status + start + end  
D) Compilation error — cannot combine `And` with `Between`  

**Answer: C** — `Between` consumes exactly 2 consecutive parameters. `status` is one parameter for the equality check. `start` and `end` are the two parameters for `Between`. Total: 3 parameters.

---

**Q5 — Return Type Behavior**
```java
User findByEmail(String email);
// DB has NO user with this email
```
What happens?

A) Returns `null`  
B) Returns an empty `Optional`  
C) Throws `EmptyResultDataAccessException`  
D) Throws `IncorrectResultSizeDataAccessException`  

**Answer: A** — When the return type is the entity directly (not `Optional`), Spring Data returns `null` if no result is found. This is why `Optional<User>` is strongly preferred — it forces callers to handle the empty case explicitly.

---

**Q6 — Method Name Parsing Trap**
```java
List<Employee> findByDepartmentNameAndSalaryGreaterThan(String name, BigDecimal salary);
```
Assuming `Employee` has a `@ManyToOne Department department` and `Department` has `String name`:

How is `DepartmentName` resolved?

A) Compile error — nested properties require `_` separator  
B) `department.name` — Spring Data resolves nested property path  
C) A field called `departmentName` on `Employee`  
D) Spring Data uses the `_` separator by default for all nested paths  

**Answer: B** — Spring Data resolves `DepartmentName` as `department.name` by splitting at property boundaries. It first tries `department` as a property of `Employee`, then `name` as a property of `Department`. This generates an implicit JOIN: `WHERE d.name = ?1 AND e.salary > ?2`.

---

**Q7 — `Page` vs `Slice`**
```java
Slice<Product> findByCategory(String category, Pageable pageable);
```
Which SQL queries are executed when this method is called with `PageRequest.of(0, 20)`?

A) `SELECT * FROM products WHERE category=? LIMIT 20` and `SELECT COUNT(*) FROM products WHERE category=?`  
B) `SELECT * FROM products WHERE category=? LIMIT 21` (no count query)  
C) `SELECT * FROM products WHERE category=? LIMIT 20` (no count query)  
D) `SELECT * FROM products WHERE category=? LIMIT 20 OFFSET 0` and count query  

**Answer: B** — `Slice` requests `pageSize + 1` rows (`LIMIT 21` for pageSize=20) to determine if a next page exists (`hasNext()` = true if 21 rows returned). No count query is fired. If exactly 20 rows returned, `hasNext()` = false.

---

**Q8 — Ordering Trap**
```java
List<User> findByStatusOrderByLastNameAsc(Status status, Sort sort);
```
When called with `Sort.by("firstName").descending()`, what is the ORDER BY clause?

A) `ORDER BY last_name ASC` — static OrderBy in method name takes precedence  
B) `ORDER BY first_name DESC` — Sort parameter overrides static OrderBy  
C) `ORDER BY last_name ASC, first_name DESC` — both applied  
D) Exception — cannot combine static OrderBy with Sort parameter  

**Answer: B** — When a `Sort` parameter is provided, it **overrides** the static `OrderBy` defined in the method name. The static `OrderByLastNameAsc` is ignored. The dynamic `Sort.by("firstName").descending()` is applied.

---

**Q9 — Scope of `AllIgnoreCase`**
```java
List<Employee> findByFirstNameAndLastNameAllIgnoreCase(String first, String last);
```

What does `AllIgnoreCase` apply to?

A) Only the last property in the method name (`lastName`)  
B) All String properties in the method (`firstName` AND `lastName`)  
C) The entire query including non-String fields  
D) It's invalid — `IgnoreCase` must be applied per-property  

**Answer: B** — `AllIgnoreCase` applies case-insensitive comparison to ALL String-type properties in the method. Both `firstName` and `lastName` comparisons use `LOWER()`. For non-String fields, it is silently ignored. `IgnoreCase` (without `All`) applies to only the adjacent property it follows.

---

**Q10 — Startup vs Runtime Error**
```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByCustomerAddress(String address);
    // Order has: Customer customer (@ManyToOne)
    // Customer has: String email, String phone
    // Customer has NO 'address' field
}
```
When does the error occur?

A) Compile time — Spring Data validates at compile time  
B) Application startup — `PropertyReferenceException`  
C) First method invocation — Spring Data validates lazily  
D) Never — Spring Data silently returns empty list for invalid paths  

**Answer: B** — Spring Data validates all property paths in derived queries at application startup (during `PartTreeJpaQuery` construction). `customer.address` is not a valid property path (Customer has no `address` field) → `PropertyReferenceException` at startup.

---

## 4️⃣ Trick Analysis

**`Between` requires two parameters (Q4)**:
`Between` is unlike all other keywords in that it consumes exactly 2 consecutive method parameters. Forgetting this causes `IllegalArgumentException` at startup when the parameter count doesn't match. The parameters must immediately follow in the method signature in the order: `min`, `max`.

**Null return type for non-Optional single result (Q5)**:
Spring Data returns `null` (not `Optional.empty()`, not an exception) when a non-Optional single-result method finds no match. This silently propagates through code until a `NullPointerException` occurs somewhere downstream. The exam strongly tests this: **always use `Optional<T>` for single-result queries**.

**`Sort` parameter overrides static `OrderBy` (Q8)**:
Developers sometimes write `findByStatusOrderByCreatedAtDesc` then pass a `Sort` parameter expecting both to apply. The dynamic `Sort` wins completely — the static `OrderBy` is discarded. This makes the `OrderBy` in the method name misleading and effectively dead code when `Sort` is passed. Best practice: never mix static `OrderBy` with a `Sort` parameter.

**`AllIgnoreCase` vs `IgnoreCase` scope (Q9)**:
`IgnoreCase` applied after a specific property affects only that property. `AllIgnoreCase` at the end of the subject or predicate applies to ALL String properties. This is a subtle distinction: `findByFirstNameIgnoreCaseAndLastName` (only firstName is case-insensitive) vs `findByFirstNameAndLastNameAllIgnoreCase` (both are case-insensitive).

**`IsNull` methods have zero parameters (Q2 edge case)**:
`findByMiddleNameIsNull()` takes no parameters — the condition is `IS NULL` with no value to bind. Adding a parameter to such a method causes a parameter binding mismatch at startup. Similarly, `findByActiveTrue()` and `findByActiveFalse()` take no parameters.

**Property path resolution is depth-first, longest-match-wins**:
When Spring Data encounters `DepartmentName`, it tries: (1) is `departmentName` a field on the entity? (2) is `department.name` a valid path? It prefers longer matches first, then shorter. The `_` separator forces an explicit split: `Department_Name` → `department.name` always, regardless of whether `departmentName` exists as a field.

**`Slice` fetches pageSize + 1, `Page` fires count query**:
`Slice` achieves its "hasNext" without a count query by fetching one extra row. The result contains at most `pageSize` items (the extra is stripped), and `hasNext()` = true if the extra row was found. `Page` always fires a separate count query — adding 1 SQL statement per page request. For "infinite scroll" / "load more" patterns, `Slice` is strictly more efficient.

---

## 5️⃣ Summary Sheet

### `PartTree` Method Name Grammar

```
[Subject][By][Predicate][OrderBy][Sort]

SUBJECT:
  find/read/get/query/search/stream  → SELECT
  count                              → SELECT COUNT
  exists                             → SELECT COUNT > 0 → boolean
  delete/remove                      → DELETE
  + Distinct: findDistinct...
  + Limit:    findTop5..., findFirst...

PREDICATE KEYWORDS:
  [none]                → SIMPLE_PROPERTY → = (equals)
  Is / Equals           → =
  Not / IsNot           → !=
  LessThan              → 
  LessThanEqual         → <=
  GreaterThan           → >
  GreaterThanEqual      → >=
  Between               → BETWEEN ? AND ?    (2 params)
  IsNull / Null         → IS NULL            (0 params)
  IsNotNull / NotNull   → IS NOT NULL        (0 params)
  Like                  → LIKE ?             (manual %)
  NotLike               → NOT LIKE ?
  StartingWith/StartsWith → LIKE ?%
  EndingWith/EndsWith     → LIKE %?
  Containing/Contains     → LIKE %?%
  In                    → IN (?)
  NotIn                 → NOT IN (?)
  True / IsTrue         → = true             (0 params)
  False / IsFalse       → = false            (0 params)
  After                 → > (for dates)
  Before                → < (for dates)
  IgnoreCase            → LOWER(col)=LOWER(?) (String only)
  AllIgnoreCase         → ALL String parts use LOWER

CONNECTORS:
  And → AND (higher precedence)
  Or  → OR  (lower precedence)
  → Mixed: (A AND B) OR C
```

### Return Type Reference

```
T                           → null if not found (dangerous)
Optional<T>                 → empty if not found (recommended)
List<T>                     → empty list if not found
Iterable<T>                 → empty if not found
Collection<T>               → empty if not found
Page<T>                     → count query fired; getTotalElements() available
Slice<T>                    → NO count query; hasNext() via pageSize+1 trick
Stream<T>                   → @Transactional required; try-with-resources
CompletableFuture<List<T>>  → @Async required
long (for count/exists)     → scalar result
boolean (for exists)        → scalar result
void (for delete)           → no return
```

### Parameter Count Rules

```
Keyword          → Parameters consumed
───────────────────────────────────────
SIMPLE_PROPERTY  → 1
Not              → 1
LessThan/GreaterThan → 1
LessThanEqual/GreaterThanEqual → 1
Between          → 2 (min, max)
IsNull/IsNotNull → 0
Like/NotLike     → 1
Containing       → 1
StartingWith     → 1
EndingWith       → 1
In/NotIn         → 1 (Collection)
True/False       → 0
After/Before     → 1
IgnoreCase       → 0 (modifier, not extra param)
```

### Limits — When to Abandon Derived Queries

```
Use @Query when:
  ✗ GROUP BY or HAVING needed
  ✗ Aggregate functions (SUM, AVG, MAX, MIN)
  ✗ Subqueries
  ✗ Complex OR grouping: A AND (B OR C)
  ✗ JOIN with conditions beyond property navigation
  ✗ UNION / UNION ALL
  ✗ Native SQL features
  ✗ Performance-critical queries needing SQL tuning

Derived queries are perfect for:
  ✓ Simple CRUD by field combinations
  ✓ Basic pagination and sorting
  ✓ Existence checks
  ✓ Count queries
  ✓ Up to 3-4 conditions with And/Or
```

### Key Rules

```
1. Method name parsed by PartTree into subject + predicate + sort
2. Predicate validation against entity at startup → PropertyReferenceException if invalid
3. Between requires 2 consecutive parameters
4. IsNull, IsNotNull, True, False → 0 parameters
5. Optional<T> always preferred over T for single-result
6. Sort parameter overrides static OrderBy in method name
7. AllIgnoreCase → all String properties in query use LOWER()
8. Page<T> → count query always fired; Slice<T> → no count query
9. Stream<T> → must be @Transactional; use try-with-resources
10. And binds tighter than Or: A AND B OR C → (A AND B) OR C
11. _ separator forces explicit property path split
12. findByActiveTrue() → no parameter; findByActive(boolean) → one parameter
13. delete derived queries load entities first (cascade/callbacks)
14. Top/First + Pageable: Top wins for size, Pageable used for sort only
```

---
