# 🏛️ TOPIC 13 — Specifications and the Criteria API

---

## 1️⃣ Conceptual Explanation

### Why Specifications Exist — The Dynamic Query Problem

Derived queries and `@Query` annotations are **static** — the predicate structure is fixed at compile time. But real applications need dynamic queries: a search screen where the user may fill in zero, one, or many of ten filter fields in any combination. Building this with derived queries produces a combinatorial explosion of method names. Building it with `@Query` requires string concatenation — fragile, unreadable, SQL-injection prone.

The Specification pattern solves this. A `Specification<T>` is a single predicate function. Specifications compose. You can combine ten individual filter Specifications into a compound query at runtime using `and()`, `or()`, `not()`, and null checks — with zero string manipulation.

---

### The `Specification<T>` Interface — Core Contract

```java
@FunctionalInterface
public interface Specification<T> {

    Predicate toPredicate(
        Root<T> root,               // FROM clause — the entity root
        CriteriaQuery<?> query,     // the full query object
        CriteriaBuilder cb          // factory for predicates and expressions
    );

    // Default composition methods:
    default Specification<T> and(Specification<T> other) { ... }
    default Specification<T> or(Specification<T> other) { ... }
    static <T> Specification<T> not(Specification<T> spec) { ... }
    static <T> Specification<T> where(Specification<T> spec) { ... }
    static <T> Specification<T> allOf(Iterable<Specification<T>> specs) { ... }
    static <T> Specification<T> anyOf(Iterable<Specification<T>> specs) { ... }
}
```

**What each parameter represents:**

```
Root<T> root:
  - Represents the entity being queried (FROM User u)
  - Access entity attributes: root.get("firstName")
  - Navigate joins: root.join("orders")
  - Navigate embedded: root.get("address").get("city")

CriteriaQuery<?> query:
  - The full query object
  - Used for: subqueries, DISTINCT, GROUP BY, HAVING, ORDER BY
  - Rarely used in simple Specifications
  - query.distinct(true) — enables DISTINCT

CriteriaBuilder cb:
  - Factory for all predicates and expressions
  - cb.equal(), cb.like(), cb.between(), cb.and(), cb.or()
  - cb.upper(), cb.lower(), cb.length(), cb.concat()
  - cb.count(), cb.sum(), cb.avg(), cb.max(), cb.min()
  - cb.subquery()
```

---

### `JpaSpecificationExecutor<T>` — The Repository Interface

```java
public interface JpaSpecificationExecutor<T> {
    Optional<T> findOne(Specification<T> spec);
    List<T> findAll(Specification<T> spec);
    Page<T> findAll(Specification<T> spec, Pageable pageable);
    List<T> findAll(Specification<T> spec, Sort sort);
    long count(Specification<T> spec);
    boolean exists(Specification<T> spec);
    
    // Spring Data 3.x additional:
    <S extends T, R> R findBy(Specification<T> spec,
        Function<FluentQuery.FetchableFluentQuery<S>, R> queryFunction);
}

// Usage: repository must extend BOTH JpaRepository AND JpaSpecificationExecutor:
public interface UserRepository
        extends JpaRepository<User, Long>,
                JpaSpecificationExecutor<User> {
}
```

---

### Internal Execution Pipeline — Specification to SQL

```
Step 1: Repository method call
  userRepo.findAll(spec, pageable)

Step 2: Spring Data routes to JpaSpecificationExecutor implementation
  → SimpleJpaRepository.findAll(Specification<T>, Pageable)

Step 3: CriteriaQuery construction
  CriteriaBuilder cb = em.getCriteriaBuilder();
  CriteriaQuery<User> cq = cb.createQuery(User.class);
  Root<User> root = cq.from(User.class);

Step 4: Specification.toPredicate() called
  Predicate predicate = spec.toPredicate(root, cq, cb);
  cq.where(predicate);

Step 5: Sort / Pagination applied
  (sort applied as ORDER BY, pageable as LIMIT/OFFSET)

Step 6: TypedQuery execution
  TypedQuery<User> typedQuery = em.createQuery(cq);
  typedQuery.setFirstResult(offset);
  typedQuery.setMaxResults(pageSize);

Step 7: Count query for Page<T>
  CriteriaQuery<Long> countCq = cb.createQuery(Long.class);
  Root<User> countRoot = countCq.from(User.class);
  countCq.select(cb.count(countRoot));
  Predicate countPredicate = spec.toPredicate(countRoot, countCq, cb);
  countCq.where(countPredicate);
  Long total = em.createQuery(countCq).getSingleResult();

Step 8: Hibernate compiles CriteriaQuery → dialect SQL → JDBC
```

The key insight: **`toPredicate()` is called TWICE** when using `Page<T>` — once for the data query and once for the count query. The `Root` and `CriteriaQuery` instances are different each time. Never store state on the `Root` instance between calls.

---

### Building Individual Specifications — The Full Predicate API

```java
// A Specification is just a function returning a Predicate (or null):
// null return = no constraint for this specification
// This is critical for the optional filter pattern

public class UserSpecs {

    // Equality:
    public static Specification<User> hasFirstName(String name) {
        return (root, query, cb) ->
            name == null ? null : cb.equal(root.get("firstName"), name);
    }

    // Case-insensitive equality:
    public static Specification<User> hasEmailIgnoreCase(String email) {
        return (root, query, cb) ->
            email == null ? null :
            cb.equal(cb.lower(root.get("email")), email.toLowerCase());
    }

    // LIKE (contains):
    public static Specification<User> nameContains(String fragment) {
        return (root, query, cb) ->
            fragment == null ? null :
            cb.like(cb.lower(root.get("name")),
                    "%" + fragment.toLowerCase() + "%");
    }

    // Comparison:
    public static Specification<User> ageGreaterThan(Integer minAge) {
        return (root, query, cb) ->
            minAge == null ? null :
            cb.greaterThan(root.get("age"), minAge);
    }

    // Between:
    public static Specification<User> salaryBetween(
            BigDecimal min, BigDecimal max) {
        return (root, query, cb) ->
            (min == null || max == null) ? null :
            cb.between(root.get("salary"), min, max);
    }

    // Is null check:
    public static Specification<User> hasNoMiddleName() {
        return (root, query, cb) ->
            cb.isNull(root.get("middleName"));
    }

    // Is not null:
    public static Specification<User> hasProfilePhoto() {
        return (root, query, cb) ->
            cb.isNotNull(root.get("profilePhotoUrl"));
    }

    // Boolean field:
    public static Specification<User> isActive() {
        return (root, query, cb) ->
            cb.isTrue(root.get("active"));
    }

    // IN clause:
    public static Specification<User> statusIn(List<Status> statuses) {
        return (root, query, cb) ->
            (statuses == null || statuses.isEmpty()) ? null :
            root.get("status").in(statuses);
    }

    // Date comparison:
    public static Specification<User> createdAfter(LocalDateTime date) {
        return (root, query, cb) ->
            date == null ? null :
            cb.greaterThan(root.get("createdAt"), date);
    }

    // Joined entity property:
    public static Specification<User> hasDepartmentName(String deptName) {
        return (root, query, cb) -> {
            if (deptName == null) return null;
            Join<User, Department> dept = root.join("department",
                                                     JoinType.INNER);
            return cb.equal(dept.get("name"), deptName);
        };
    }

    // Nested join:
    public static Specification<User> hasOrderInStatus(OrderStatus status) {
        return (root, query, cb) -> {
            if (status == null) return null;
            Join<User, Order> orders = root.join("orders", JoinType.LEFT);
            return cb.equal(orders.get("status"), status);
        };
    }

    // Subquery:
    public static Specification<User> hasMoreThanNOrders(long n) {
        return (root, query, cb) -> {
            Subquery<Long> subquery = query.subquery(Long.class);
            Root<Order> orderRoot = subquery.from(Order.class);
            subquery.select(cb.count(orderRoot))
                    .where(cb.equal(orderRoot.get("customer"), root));
            return cb.greaterThan(subquery, n);
        };
    }
}
```

---

### Composing Specifications — The Power of the Pattern

```java
// Individual specs:
Specification<User> activeSpec     = UserSpecs.isActive();
Specification<User> nameSpec       = UserSpecs.nameContains("Smith");
Specification<User> ageSpec        = UserSpecs.ageGreaterThan(30);
Specification<User> deptSpec       = UserSpecs.hasDepartmentName("Engineering");

// Composition with and():
Specification<User> combined =
    Specification.where(activeSpec)
                 .and(nameSpec)
                 .and(ageSpec)
                 .and(deptSpec);
// WHERE active=true AND name LIKE '%Smith%' AND age>30 AND dept.name='Engineering'

// Composition with or():
Specification<User> eitherName =
    Specification.where(UserSpecs.hasFirstName("Alice"))
                 .or(UserSpecs.hasFirstName("Bob"));
// WHERE first_name='Alice' OR first_name='Bob'

// Mixed:
Specification<User> complex =
    Specification.where(activeSpec)
                 .and(nameSpec.or(deptSpec));
// WHERE active=true AND (name LIKE '%Smith%' OR dept.name='Engineering')

// Negation:
Specification<User> notActive = Specification.not(activeSpec);
// WHERE active != true (or active = false depending on CB implementation)

// Conditional composition — key pattern for dynamic filters:
public Specification<User> buildSearchSpec(UserSearchRequest req) {
    return Specification
        .where(UserSpecs.nameContains(req.getName()))           // null = skip
        .and(UserSpecs.hasEmailIgnoreCase(req.getEmail()))      // null = skip
        .and(UserSpecs.ageGreaterThan(req.getMinAge()))         // null = skip
        .and(UserSpecs.statusIn(req.getStatuses()))             // null = skip
        .and(UserSpecs.createdAfter(req.getCreatedAfter()));    // null = skip
    // Each spec returns null predicate when its parameter is null
    // Spring Data treats null predicates as "no restriction" for that field
}
```

---

### How `null` Predicates Are Handled

A `Specification.toPredicate()` returning `null` means **"no restriction"** — the clause is omitted from the WHERE clause. This is how optional filters work.

```java
// Spring Data SimpleJpaRepository composition logic (simplified):
Predicate[] predicates = specs.stream()
    .map(s -> s.toPredicate(root, query, cb))
    .filter(Objects::nonNull)           // null predicates filtered out
    .toArray(Predicate[]::new);

if (predicates.length == 0) {
    // No WHERE clause — returns all entities
} else if (predicates.length == 1) {
    cq.where(predicates[0]);
} else {
    cq.where(cb.and(predicates));       // AND all non-null predicates
}
```

**The `Specification.where(null)` identity:**

```java
Specification.where(null)
// Returns a Specification that produces no WHERE clause — fetches all
// Equivalent to findAll()
// Used as the starting point when all filters may be null

// Common pattern:
Specification<User> spec = Specification.where(null);
if (request.getName() != null) spec = spec.and(UserSpecs.nameContains(request.getName()));
if (request.getAge() != null)  spec = spec.and(UserSpecs.ageGreaterThan(request.getAge()));
// If neither set: spec = WHERE (nothing) → findAll
```

---

### The JPA Static Metamodel — Type-Safe Criteria

String-based attribute access (`root.get("firstName")`) is fragile — typos are only caught at runtime. The JPA static metamodel provides compile-time safety.

```java
// Metamodel class (auto-generated by JPA processor, or written manually):
@StaticMetamodel(User.class)
public class User_ {
    public static volatile SingularAttribute<User, Long>        id;
    public static volatile SingularAttribute<User, String>      firstName;
    public static volatile SingularAttribute<User, String>      lastName;
    public static volatile SingularAttribute<User, String>      email;
    public static volatile SingularAttribute<User, Integer>     age;
    public static volatile SingularAttribute<User, Boolean>     active;
    public static volatile SingularAttribute<User, LocalDateTime> createdAt;
    public static volatile ListAttribute<User, Order>           orders;
    public static volatile SingularAttribute<User, Department>  department;
}

// Without metamodel (string-based, fragile):
root.get("firstName")           // typo → runtime exception
root.join("orders")             // typo → runtime exception

// With metamodel (type-safe):
root.get(User_.firstName)       // compile-time safe, type-inferred
root.join(User_.orders)         // compile-time safe

// Type safety benefits:
Expression<String> firstNameExpr = root.get(User_.firstName);
// → type is inferred as Expression<String>
// cb.like() requires Expression<String> — enforced at compile time
cb.like(firstNameExpr, "%Smith%");      // ✓ compiles
cb.greaterThan(firstNameExpr, 30);      // ✗ compile error: String vs Integer
```

**Generating the metamodel:**

```xml
<!-- Maven — hibernate-jpamodelgen processor: -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jpamodelgen</artifactId>
    <scope>provided</scope>
</dependency>
```

Generated classes end in `_` by convention (`User_`, `Order_`, `Product_`) and are placed in the same package as the entity.

---

### Join Types in Specifications

```java
// INNER JOIN (default):
Join<User, Department> dept = root.join("department");
// SQL: INNER JOIN departments d ON u.department_id = d.id
// Excludes users with NO department

// LEFT JOIN (outer):
Join<User, Order> orders = root.join("orders", JoinType.LEFT);
// SQL: LEFT JOIN orders o ON u.id = o.user_id
// Includes users with NO orders (orders attributes will be null)

// RIGHT JOIN (rarely used):
Join<User, Department> dept = root.join("department", JoinType.RIGHT);

// Fetch join (for loading, not filtering):
root.fetch("department");          // fetch join — not a Join<>, returns Fetch<>
root.fetch("orders", JoinType.LEFT); // left fetch join

// CRITICAL: Join vs Fetch in Specifications
// Use Join (not Fetch) when joining for filtering purposes in Specifications
// The Fetch interface does not extend Join — you cannot apply predicates to Fetch
// Use Fetch only when you want to eagerly load associations as a side effect
```

**The duplicate result problem with collection joins:**

```java
// Joining a collection (@OneToMany) produces multiple rows per parent:
root.join("orders", JoinType.LEFT);
// SQL: SELECT u.* FROM users u LEFT JOIN orders o ON u.id = o.user_id
// User with 5 orders → 5 result rows in ResultSet → 5 User instances in result list!

// Fix: DISTINCT in criteria query:
public static Specification<User> hasOrderInStatus(OrderStatus status) {
    return (root, query, cb) -> {
        query.distinct(true);   // ← prevents duplicate User instances
        Join<User, Order> orders = root.join("orders", JoinType.LEFT);
        return status == null ? null : cb.equal(orders.get("status"), status);
    };
}
```

**The DISTINCT + count query interaction:**

```java
// query.distinct(true) in the data Specification
// affects BOTH the data query AND the auto-generated count query:

// Data query:  SELECT DISTINCT u FROM User u LEFT JOIN u.orders o WHERE ...
// Count query: SELECT COUNT(DISTINCT u) FROM User u LEFT JOIN u.orders o WHERE ...
// ✓ Both are correct — DISTINCT propagates appropriately
```

---

### `allOf` and `anyOf` — Bulk Composition

```java
// allOf — AND of a list of specs:
List<Specification<User>> specs = new ArrayList<>();
if (req.getName() != null)    specs.add(UserSpecs.nameContains(req.getName()));
if (req.getMinAge() != null)  specs.add(UserSpecs.ageGreaterThan(req.getMinAge()));
if (req.isActiveOnly())       specs.add(UserSpecs.isActive());

Specification<User> combined = Specification.allOf(specs);
// WHERE name LIKE ? AND age > ? AND active=true

// anyOf — OR of a list of specs:
List<Specification<User>> orSpecs = List.of(
    UserSpecs.hasFirstName("Alice"),
    UserSpecs.hasFirstName("Bob"),
    UserSpecs.hasFirstName("Charlie")
);
Specification<User> anyName = Specification.anyOf(orSpecs);
// WHERE first_name='Alice' OR first_name='Bob' OR first_name='Charlie'

// allOf(empty list) = no WHERE clause (matches all)
// anyOf(empty list) = no WHERE clause (matches all)
```

---

### Raw Criteria API Without Specifications

Sometimes you need full Criteria API power without the Specification wrapper — for queries with multiple result types, GROUP BY, or non-entity return types:

```java
@Repository
public class UserCriteriaRepository {

    @PersistenceContext
    private EntityManager em;

    public List<UserSummaryDto> findUserSummaries(UserSearchCriteria criteria) {

        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<UserSummaryDto> cq = cb.createQuery(UserSummaryDto.class);
        Root<User> user = cq.from(User.class);
        Join<User, Department> dept = user.join("department", JoinType.LEFT);

        // SELECT clause — constructor expression:
        cq.select(cb.construct(UserSummaryDto.class,
            user.get(User_.id),
            user.get(User_.firstName),
            user.get(User_.lastName),
            dept.get(Department_.name)
        ));

        // WHERE clause:
        List<Predicate> predicates = new ArrayList<>();
        if (criteria.getName() != null) {
            predicates.add(cb.like(
                cb.lower(user.get(User_.firstName)),
                "%" + criteria.getName().toLowerCase() + "%"
            ));
        }
        if (criteria.isActiveOnly()) {
            predicates.add(cb.isTrue(user.get(User_.active)));
        }
        cq.where(cb.and(predicates.toArray(new Predicate[0])));

        // ORDER BY:
        cq.orderBy(
            cb.asc(user.get(User_.lastName)),
            cb.asc(user.get(User_.firstName))
        );

        return em.createQuery(cq)
                 .setFirstResult(criteria.getOffset())
                 .setMaxResults(criteria.getPageSize())
                 .getResultList();
    }

    // COUNT query for pagination:
    public long countUserSummaries(UserSearchCriteria criteria) {

        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Long> cq = cb.createQuery(Long.class);
        Root<User> user = cq.from(User.class);

        cq.select(cb.count(user));

        List<Predicate> predicates = new ArrayList<>();
        // ... same predicates as above ...
        cq.where(cb.and(predicates.toArray(new Predicate[0])));

        return em.createQuery(cq).getSingleResult();
    }
}
```

---

### `CriteriaBuilder` — Complete Expression Reference

```java
// ── Comparison predicates ──────────────────────────────────────────────────
cb.equal(expr, value)              // =
cb.notEqual(expr, value)           // !=
cb.greaterThan(expr, value)        // >
cb.greaterThanOrEqualTo(expr, v)   // >=
cb.lessThan(expr, value)           // 
cb.lessThanOrEqualTo(expr, value)  // <=
cb.between(expr, lower, upper)     // BETWEEN ? AND ?

// ── Null / Boolean checks ──────────────────────────────────────────────────
cb.isNull(expr)                    // IS NULL
cb.isNotNull(expr)                 // IS NOT NULL
cb.isTrue(expr)                    // = true
cb.isFalse(expr)                   // = false

// ── String operations ──────────────────────────────────────────────────────
cb.like(expr, pattern)             // LIKE '%pattern%'
cb.notLike(expr, pattern)          // NOT LIKE
cb.upper(stringExpr)               // UPPER(col)
cb.lower(stringExpr)               // LOWER(col)
cb.length(stringExpr)              // LENGTH(col)
cb.concat(expr1, expr2)            // CONCAT(a, b)
cb.substring(expr, from, len)      // SUBSTRING(col, from, len)
cb.trim(expr)                      // TRIM(col)

// ── Logical connectors ─────────────────────────────────────────────────────
cb.and(pred1, pred2)               // pred1 AND pred2
cb.and(Predicate[] predicates)     // p1 AND p2 AND ... (varargs/array)
cb.or(pred1, pred2)                // pred1 OR pred2
cb.or(Predicate[] predicates)      // p1 OR p2 OR ...
cb.not(predicate)                  // NOT predicate
cb.conjunction()                   // 1=1 (always true — identity for AND)
cb.disjunction()                   // 1=0 (always false — identity for OR)

// ── Collection operations ──────────────────────────────────────────────────
expr.in(collection)                // col IN (v1, v2, v3)
expr.in(value1, value2, value3)    // col IN (v1, v2, v3)
cb.isEmpty(collectionExpr)         // collection is empty
cb.isNotEmpty(collectionExpr)      // collection is not empty
cb.isMember(element, collectionExpr) // element IN collection

// ── Aggregate functions ────────────────────────────────────────────────────
cb.count(expr)                     // COUNT(expr)
cb.countDistinct(expr)             // COUNT(DISTINCT expr)
cb.sum(expr)                       // SUM(expr)
cb.avg(expr)                       // AVG(expr)
cb.max(expr)                       // MAX(expr)
cb.min(expr)                       // MIN(expr)

// ── Subqueries ─────────────────────────────────────────────────────────────
Subquery<Long> sub = query.subquery(Long.class);
Root<Order> subRoot = sub.from(Order.class);
sub.select(cb.count(subRoot))
   .where(cb.equal(subRoot.get("customer"), root));
cb.exists(sub)                     // EXISTS (subquery)
cb.greaterThan(sub, 5L)           // (subquery) > 5
```

---

### `cb.conjunction()` and `cb.disjunction()` — Identity Elements

```java
// cb.conjunction() = 1=1 = identity element for AND
// Used as initial value when building AND predicates iteratively:

Predicate result = cb.conjunction(); // start with "always true"
if (nameFilter != null)
    result = cb.and(result, cb.like(root.get("name"), "%"+nameFilter+"%"));
if (ageFilter != null)
    result = cb.and(result, cb.greaterThan(root.get("age"), ageFilter));
// If neither filter applied: WHERE 1=1 (all rows)

// cb.disjunction() = 1=0 = identity element for OR
// Used when building OR chains where no conditions → return nothing:
Predicate result = cb.disjunction(); // start with "always false"
for (String status : statuses) {
    result = cb.or(result, cb.equal(root.get("status"), status));
}
// If statuses is empty: WHERE 1=0 (no rows)
// This is safer than generating WHERE status IN () which fails on some DBs
```

---

## 2️⃣ Code Examples

### Example 1 — Complete Dynamic Search Specification

```java
// Entity:
@Entity
public class Employee {
    @Id @GeneratedValue Long id;
    String firstName, lastName, email;
    String department;
    BigDecimal salary;
    int age;
    boolean active;
    LocalDateTime hiredAt;
    EmploymentStatus status;

    @ManyToOne Department dept;
    @OneToMany(mappedBy="employee") List<Project> projects;
}

// Search request DTO:
public record EmployeeSearchRequest(
    String firstName,
    String lastName,
    String department,
    Integer minAge,
    Integer maxAge,
    BigDecimal minSalary,
    BigDecimal maxSalary,
    Boolean active,
    List<EmploymentStatus> statuses,
    LocalDateTime hiredAfter
) {}

// Specifications — one per filter dimension:
public class EmployeeSpecs {

    public static Specification<Employee> firstNameContains(String name) {
        return (root, query, cb) -> name == null ? null :
            cb.like(cb.lower(root.get("firstName")),
                    "%" + name.toLowerCase() + "%");
    }

    public static Specification<Employee> lastNameContains(String name) {
        return (root, query, cb) -> name == null ? null :
            cb.like(cb.lower(root.get("lastName")),
                    "%" + name.toLowerCase() + "%");
    }

    public static Specification<Employee> inDepartment(String dept) {
        return (root, query, cb) -> dept == null ? null :
            cb.equal(root.get("department"), dept);
    }

    public static Specification<Employee> ageBetween(Integer min, Integer max) {
        return (root, query, cb) -> {
            if (min == null && max == null) return null;
            if (min == null) return cb.lessThanOrEqualTo(root.get("age"), max);
            if (max == null) return cb.greaterThanOrEqualTo(root.get("age"), min);
            return cb.between(root.get("age"), min, max);
        };
    }

    public static Specification<Employee> salaryBetween(
            BigDecimal min, BigDecimal max) {
        return (root, query, cb) -> {
            if (min == null && max == null) return null;
            if (min == null) return cb.lessThanOrEqualTo(root.get("salary"), max);
            if (max == null) return cb.greaterThanOrEqualTo(root.get("salary"), min);
            return cb.between(root.get("salary"), min, max);
        };
    }

    public static Specification<Employee> isActive(Boolean active) {
        return (root, query, cb) -> active == null ? null :
            active ? cb.isTrue(root.get("active"))
                   : cb.isFalse(root.get("active"));
    }

    public static Specification<Employee> statusIn(
            List<EmploymentStatus> statuses) {
        return (root, query, cb) ->
            (statuses == null || statuses.isEmpty()) ? null :
            root.get("status").in(statuses);
    }

    public static Specification<Employee> hiredAfter(LocalDateTime date) {
        return (root, query, cb) -> date == null ? null :
            cb.greaterThan(root.get("hiredAt"), date);
    }
}

// Service — composing dynamically:
@Service
public class EmployeeSearchService {

    @Autowired EmployeeRepository employeeRepo;

    public Page<Employee> search(EmployeeSearchRequest req, Pageable pageable) {

        Specification<Employee> spec = Specification
            .where(EmployeeSpecs.firstNameContains(req.firstName()))
            .and(EmployeeSpecs.lastNameContains(req.lastName()))
            .and(EmployeeSpecs.inDepartment(req.department()))
            .and(EmployeeSpecs.ageBetween(req.minAge(), req.maxAge()))
            .and(EmployeeSpecs.salaryBetween(req.minSalary(), req.maxSalary()))
            .and(EmployeeSpecs.isActive(req.active()))
            .and(EmployeeSpecs.statusIn(req.statuses()))
            .and(EmployeeSpecs.hiredAfter(req.hiredAfter()));

        return employeeRepo.findAll(spec, pageable);
        // SQL generated at runtime based on which specs returned non-null
    }

    public long countSearch(EmployeeSearchRequest req) {
        Specification<Employee> spec = Specification
            .where(EmployeeSpecs.firstNameContains(req.firstName()))
            .and(EmployeeSpecs.inDepartment(req.department()))
            // ... etc
            ;
        return employeeRepo.count(spec);
        // SQL: SELECT COUNT(e.id) FROM employees e WHERE ...
    }
}
```

---

### Example 2 — Specifications with Joins and Subqueries

```java
public class OrderSpecs {

    // Join to associated entity:
    public static Specification<Order> customerNameContains(String name) {
        return (root, query, cb) -> {
            if (name == null) return null;
            Join<Order, Customer> customer = root.join("customer",
                                                        JoinType.INNER);
            return cb.like(
                cb.lower(customer.get("name")),
                "%" + name.toLowerCase() + "%"
            );
        };
    }

    // Collection join with DISTINCT to prevent duplicates:
    public static Specification<Order> hasItemWithProduct(String productName) {
        return (root, query, cb) -> {
            if (productName == null) return null;
            query.distinct(true);  // prevent duplicate Orders from JOIN
            Join<Order, OrderItem> items = root.join("items", JoinType.LEFT);
            Join<OrderItem, Product> product = items.join("product",
                                                          JoinType.LEFT);
            return cb.like(
                cb.lower(product.get("name")),
                "%" + productName.toLowerCase() + "%"
            );
        };
    }

    // Subquery — orders where customer has placed > N total orders:
    public static Specification<Order> customerHasMoreThanNOrders(long n) {
        return (root, query, cb) -> {
            Subquery<Long> sub = query.subquery(Long.class);
            Root<Order> subRoot = sub.from(Order.class);
            sub.select(cb.count(subRoot))
               .where(cb.equal(
                   subRoot.get("customer"),
                   root.get("customer")
               ));
            return cb.greaterThan(sub, n);
        };
    }

    // Existence subquery — has at least one item over price:
    public static Specification<Order> hasExpensiveItem(BigDecimal threshold) {
        return (root, query, cb) -> {
            if (threshold == null) return null;
            Subquery<Long> sub = query.subquery(Long.class);
            Root<OrderItem> itemRoot = sub.from(OrderItem.class);
            sub.select(itemRoot.get("id"))
               .where(
                   cb.and(
                       cb.equal(itemRoot.get("order"), root),
                       cb.greaterThan(itemRoot.get("unitPrice"), threshold)
                   )
               );
            return cb.exists(sub);
        };
    }

    // Combining in service:
    public Page<Order> searchOrders(OrderSearchRequest req, Pageable p) {
        return orderRepo.findAll(
            Specification
                .where(customerNameContains(req.customerName()))
                .and(hasItemWithProduct(req.productName()))
                .and(customerHasMoreThanNOrders(req.minCustomerOrders())),
            p
        );
    }
}
```

---

### Example 3 — Type-Safe Metamodel Specifications

```java
// Auto-generated metamodel (or manually written):
@StaticMetamodel(Employee.class)
public class Employee_ {
    public static volatile SingularAttribute<Employee, Long>           id;
    public static volatile SingularAttribute<Employee, String>         firstName;
    public static volatile SingularAttribute<Employee, String>         lastName;
    public static volatile SingularAttribute<Employee, String>         email;
    public static volatile SingularAttribute<Employee, Integer>        age;
    public static volatile SingularAttribute<Employee, BigDecimal>     salary;
    public static volatile SingularAttribute<Employee, Boolean>        active;
    public static volatile SingularAttribute<Employee, LocalDateTime>  hiredAt;
    public static volatile SingularAttribute<Employee, Department>     dept;
    public static volatile ListAttribute<Employee, Project>            projects;
}

// Type-safe specifications using metamodel:
public class TypeSafeEmployeeSpecs {

    // Compile-time safe — type mismatch caught by compiler:
    public static Specification<Employee> ageGreaterThan(int minAge) {
        return (root, query, cb) ->
            cb.greaterThan(
                root.get(Employee_.age),   // Expression<Integer> inferred
                minAge
            );
        // cb.greaterThan(Expression<Y>, Y) — both are Integer: compiles
        // If you tried: cb.like(root.get(Employee_.age), "%30%")
        // → compile error: like() requires Expression<String>
    }

    public static Specification<Employee> salaryBetween(
            BigDecimal min, BigDecimal max) {
        return (root, query, cb) ->
            (min == null || max == null) ? null :
            cb.between(
                root.get(Employee_.salary),  // Expression<BigDecimal>
                min,
                max
            );
    }

    // Join with metamodel:
    public static Specification<Employee> inDepartmentWithBudgetOver(
            BigDecimal budget) {
        return (root, query, cb) -> {
            if (budget == null) return null;
            Join<Employee, Department> dept =
                root.join(Employee_.dept, JoinType.INNER);
            // dept.get(Department_.budget) → Expression<BigDecimal>
            return cb.greaterThan(dept.get(Department_.budget), budget);
        };
    }
}
```

---

### Example 4 — `cb.conjunction()` / `cb.disjunction()` Patterns

```java
// Pattern: build AND chain iteratively (safe with empty list):
public Specification<User> buildFromMap(Map<String, Object> filters) {
    return (root, query, cb) -> {
        // Start with 1=1 (conjunction = always true):
        Predicate result = cb.conjunction();

        if (filters.containsKey("name")) {
            result = cb.and(result,
                cb.like(root.get("name"),
                        "%" + filters.get("name") + "%"));
        }
        if (filters.containsKey("active")) {
            result = cb.and(result,
                cb.equal(root.get("active"), filters.get("active")));
        }
        if (filters.containsKey("minAge")) {
            result = cb.and(result,
                cb.greaterThan(root.get("age"),
                               (Integer) filters.get("minAge")));
        }
        return result;
        // If no filters: returns cb.conjunction() = 1=1 = no WHERE restriction
    };
}

// Pattern: build OR chain for multi-keyword search:
public Specification<Product> matchesAnyKeyword(List<String> keywords) {
    return (root, query, cb) -> {
        if (keywords == null || keywords.isEmpty())
            return cb.conjunction(); // return all if no keywords

        // Build OR across name, description, sku:
        Predicate[] predicates = keywords.stream()
            .map(kw -> {
                String pattern = "%" + kw.toLowerCase() + "%";
                return cb.or(
                    cb.like(cb.lower(root.get("name")), pattern),
                    cb.like(cb.lower(root.get("description")), pattern),
                    cb.like(cb.lower(root.get("sku")), pattern)
                );
            })
            .toArray(Predicate[]::new);

        // Each keyword must match at least one field (AND across keywords):
        return cb.and(predicates);
    };
}
```

---

### Example 5 — Reusable Base Specification Class

```java
// Reusable base for common patterns across entity types:
public abstract class BaseSpecs<T> {

    // Template method — override for entity-specific field name:
    protected abstract String getNameField();
    protected abstract String getActiveField();
    protected abstract String getCreatedAtField();

    public Specification<T> nameContains(String name) {
        return (root, query, cb) -> name == null ? null :
            cb.like(cb.lower(root.get(getNameField())),
                    "%" + name.toLowerCase() + "%");
    }

    public Specification<T> isActive() {
        return (root, query, cb) ->
            cb.isTrue(root.get(getActiveField()));
    }

    public Specification<T> createdAfter(LocalDateTime dt) {
        return (root, query, cb) -> dt == null ? null :
            cb.greaterThan(root.get(getCreatedAtField()), dt);
    }
}

// Entity-specific extension:
public class ProductSpecs extends BaseSpecs<Product> {
    protected String getNameField()      { return "name"; }
    protected String getActiveField()    { return "active"; }
    protected String getCreatedAtField() { return "createdAt"; }

    // Additional product-specific:
    public Specification<Product> priceBetween(BigDecimal lo, BigDecimal hi) {
        return (root, query, cb) ->
            (lo == null || hi == null) ? null :
            cb.between(root.get("price"), lo, hi);
    }
}
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ**
What does `Specification.toPredicate()` returning `null` mean?

A) The query returns no results (WHERE 1=0)  
B) A `NullPointerException` is thrown when Spring Data applies the predicate  
C) The specification adds no restriction — the condition is omitted from WHERE  
D) Equivalent to `cb.conjunction()` (1=1) — explicitly returns all  

**Answer: C** — A null predicate returned by `toPredicate()` signals Spring Data to skip that condition. Spring Data filters out null predicates before building the WHERE clause. This is the foundation of the optional filter pattern. Note: C and D have the same practical effect (no restriction), but the mechanism is different — null means "skip this spec," while `cb.conjunction()` explicitly adds a 1=1 predicate.

---

**Q2 — Select All That Apply**
Which statements about `JpaSpecificationExecutor` are TRUE?

A) A repository must extend both `JpaRepository` and `JpaSpecificationExecutor` to use Specifications  
B) `JpaSpecificationExecutor` is automatically added to any repository that has `JpaRepository`  
C) `findAll(Specification, Pageable)` fires both a data query and a count query  
D) `count(Specification)` uses `cb.count()` on the entity root  
E) `exists(Specification)` always fires a COUNT query and checks if > 0  
F) `toPredicate()` is called once for data and once for count when using `Page<T>`  

**Answer: A, C, D, F**
- B is false: `JpaSpecificationExecutor` must be explicitly added to the repository interface
- E is false: `exists()` may use an EXISTS subquery or optimized COUNT — the exact approach is implementation-dependent

---

**Q3 — Code Trace**
```java
Specification<User> spec1 = (root, query, cb) -> cb.isTrue(root.get("active"));
Specification<User> spec2 = (root, query, cb) -> cb.greaterThan(root.get("age"), 25);
Specification<User> spec3 = (root, query, cb) -> null;

Specification<User> combined = Specification
    .where(spec1)
    .and(spec2)
    .and(spec3);

userRepo.findAll(combined);
```

What SQL WHERE clause is generated?

A) `WHERE active=true AND age>25 AND null`  
B) `WHERE active=true AND age>25`  
C) `WHERE active=true AND age>25 AND 1=1`  
D) `WHERE 1=1` (null spec overrides all)  

**Answer: B** — `spec3` returns null, which Spring Data filters out. The final WHERE clause contains only the predicates from `spec1` and `spec2`.

---

**Q4 — Join Duplicate Results**
```java
public static Specification<Customer> hasOrderOver(BigDecimal amount) {
    return (root, query, cb) -> {
        Join<Customer, Order> orders = root.join("orders", JoinType.LEFT);
        return cb.greaterThan(orders.get("total"), amount);
    };
}

List<Customer> results = customerRepo.findAll(hasOrderOver(new BigDecimal("100")));
// A customer has 3 orders all over $100
```

How many times does this customer appear in `results`?

A) 1  
B) 3  
C) 0 — LEFT JOIN requires a match  
D) Depends on database  

**Answer: B** — LEFT JOIN on `orders` produces one result row per order. A customer with 3 matching orders appears 3 times. Fix: add `query.distinct(true)` inside the Specification, or use a subquery with EXISTS instead of a JOIN.

---

**Q5 — Specification Composition**
```java
Specification<User> s1 = UserSpecs.isActive();
Specification<User> s2 = UserSpecs.ageGreaterThan(18);
Specification<User> s3 = UserSpecs.nameContains("Smith");

Specification<User> result = s1.and(s2.or(s3));
```

Which WHERE clause is produced?

A) `WHERE active=true AND age>18 OR name LIKE '%Smith%'`  
B) `WHERE active=true AND (age>18 OR name LIKE '%Smith%')`  
C) `WHERE (active=true AND age>18) OR name LIKE '%Smith%'`  
D) `WHERE active=true OR age>18 AND name LIKE '%Smith%'`  

**Answer: B** — `s2.or(s3)` creates a compound specification: `(age>18 OR name LIKE '%Smith%')`. Then `s1.and(...)` wraps it: `active=true AND (age>18 OR name LIKE '%Smith%')`. The grouping is explicit in the CriteriaBuilder — parentheses are preserved exactly as coded.

---

**Q6 — Metamodel Type Safety**
```java
@StaticMetamodel(User.class)
public class User_ {
    public static volatile SingularAttribute<User, Integer> age;
    public static volatile SingularAttribute<User, String>  firstName;
}

// Specification code:
(root, query, cb) -> cb.like(root.get(User_.age), "%30%")
```

What happens?

A) Works — `like` accepts any expression  
B) Compile-time error — `like()` requires `Expression<String>`, `age` is `Expression<Integer>`  
C) Runtime error — `ClassCastException` at execution  
D) Works — Hibernate converts Integer to String  

**Answer: B** — The metamodel makes `root.get(User_.age)` return `Expression<Integer>`. `CriteriaBuilder.like()` requires `Expression<String>`. This type mismatch is a compile-time error — the primary benefit of metamodel-based type safety.

---

**Q7 — `toPredicate` Call Count**
```java
Page<User> result = userRepo.findAll(mySpec, PageRequest.of(0, 20));
```

How many times is `mySpec.toPredicate()` called?

A) Once — for the data query  
B) Twice — once for data query, once for count query  
C) Three times — data, count, and validation  
D) Depends on whether count query is cached  

**Answer: B** — `Page<T>` requires both a data query and a count query. Spring Data calls `toPredicate()` once for each — with different `Root` and `CriteriaQuery` instances. State stored on `root` between calls would be lost. Never cache `Root` references across calls.

---

**Q8 — `cb.conjunction()` vs `null`**
```java
// Spec A:
Specification<User> specA = (root, query, cb) -> null;

// Spec B:
Specification<User> specB = (root, query, cb) -> cb.conjunction();
```

What is the practical difference in the generated SQL?

A) Both produce no WHERE clause — functionally identical  
B) specA adds no restriction, specB adds `WHERE 1=1` which may affect query plans  
C) specA returns all users, specB returns zero users  
D) specA throws NullPointerException, specB works correctly  

**Answer: B** — Functionally both return all results. The practical difference: `null` causes Spring Data to skip the predicate entirely (no contribution to WHERE). `cb.conjunction()` explicitly adds a `1=1` predicate to the WHERE clause. Most optimizers treat `WHERE 1=1` as a no-op, but technically the WHERE clause is present vs absent. For the exam: both are "no restriction" but the mechanism differs.

---

## 4️⃣ Trick Analysis

**Null predicate vs `cb.conjunction()` — subtle but important (Q8)**:
Returning `null` from `toPredicate()` and returning `cb.conjunction()` both produce "no restriction" behavior, but they differ in whether a WHERE clause is present. `null` = predicate removed by Spring Data filter before query construction. `cb.conjunction()` = predicate included as 1=1. For exam questions asking "what does returning null do" — the answer is always "no restriction/condition omitted," not "NullPointerException."

**`toPredicate()` called twice for `Page<T>` (Q7)**:
This is the most commonly missed fact about Specifications. For data queries, Spring Data creates one `Root<T>` and one `CriteriaQuery<T>`. For the count query, it creates a separate `Root<T>` and `CriteriaQuery<Long>`. If your Specification stores anything on the `Root` instance or modifies the `CriteriaQuery` and expects that state to persist, it will fail. Each call to `toPredicate()` is completely independent.

**Join duplicate results — `query.distinct(true)` is the fix (Q4)**:
This is one of the most common production bugs with Specifications. Joining a `@OneToMany` or `@ManyToMany` association multiplies result rows by the collection size. A customer with 5 orders appears 5 times. The fix `query.distinct(true)` in the Specification instructs Hibernate to add DISTINCT to the JPQL — but be careful: this also affects the auto-generated count query, which is what you want (COUNT DISTINCT). For complex scenarios, consider EXISTS subquery instead of JOIN to avoid the multiplication entirely.

**Specification composition grouping — parentheses matter (Q5)**:
`s1.and(s2.or(s3))` produces `A AND (B OR C)`. `s1.and(s2).or(s3)` produces `(A AND B) OR C`. These are logically different. The method chaining order determines grouping. The CriteriaBuilder respects the grouping exactly as you compose it — no operator precedence reordering occurs between `and()`/`or()` composition methods.

**`JpaSpecificationExecutor` must be explicitly added (Q2, B)**:
Spring Data does not magically add `JpaSpecificationExecutor` to repositories. You must explicitly include it in the interface declaration. `extends JpaRepository<T, ID>` alone gives no Specification support. `extends JpaRepository<T, ID>, JpaSpecificationExecutor<T>` provides both. Exam questions often test this explicit extension requirement.

**Metamodel compile-time safety is real and tested (Q6)**:
The whole point of `@StaticMetamodel` and generated `Entity_` classes is that type mismatches — like passing an `Expression<Integer>` to `cb.like()` — become compile errors. Without the metamodel, `root.get("age")` returns `Path<Object>` which accepts any operation, and you only find type mismatches at runtime. The exam tests that you understand the metamodel provides compile-time (not runtime) type safety.

---

## 5️⃣ Summary Sheet

### `Specification<T>` Pattern Reference

```
Interface:  Specification<T>
Method:     Predicate toPredicate(Root<T>, CriteriaQuery<?>, CriteriaBuilder)

Return null   → no restriction (condition omitted from WHERE)
Return predicate → condition added to WHERE

Called:
  Once  for List/Slice results
  Twice for Page results (data query + count query)
  Each call gets fresh Root<T> and CriteriaQuery instances
```

### Composition Rules

```java
// AND composition:
spec1.and(spec2)          // WHERE spec1_pred AND spec2_pred
Specification.where(s1).and(s2).and(s3)
Specification.allOf(List.of(s1, s2, s3))

// OR composition:
spec1.or(spec2)           // WHERE spec1_pred OR spec2_pred
Specification.anyOf(List.of(s1, s2, s3))

// NOT:
Specification.not(spec1)  // WHERE NOT spec1_pred

// Grouping:
s1.and(s2.or(s3))        // WHERE pred1 AND (pred2 OR pred3)
s1.and(s2).or(s3)        // WHERE (pred1 AND pred2) OR pred3

// Identity elements:
Specification.where(null)  // no WHERE (find all)
cb.conjunction()           // 1=1 (no effective restriction)
cb.disjunction()           // 1=0 (returns nothing — use with care)
```

### Join Type Reference

| `JoinType` | SQL | Includes entities with no match? |
|---|---|---|
| `INNER` (default) | `INNER JOIN` | No — excludes nulls |
| `LEFT` | `LEFT OUTER JOIN` | Yes — nulls for no match |
| `RIGHT` | `RIGHT OUTER JOIN` | Yes — rarely used |

**Collection join → always `query.distinct(true)` to prevent duplicates**

### `CriteriaBuilder` Quick Reference

```
Equality:     cb.equal, cb.notEqual
Comparison:   cb.greaterThan, cb.greaterThanOrEqualTo, cb.lessThan, cb.lessThanOrEqualTo
Range:        cb.between
Null checks:  cb.isNull, cb.isNotNull
Boolean:      cb.isTrue, cb.isFalse
String:       cb.like, cb.notLike, cb.upper, cb.lower, cb.length, cb.concat
Collection:   expr.in(list), cb.isEmpty, cb.isNotEmpty, cb.isMember
Aggregates:   cb.count, cb.countDistinct, cb.sum, cb.avg, cb.max, cb.min
Subquery:     query.subquery(T.class), cb.exists(subquery)
Logical:      cb.and(p1,p2), cb.or(p1,p2), cb.not(p), cb.conjunction(), cb.disjunction()
```

### Key Rules

```
1.  Repository must explicitly extend JpaSpecificationExecutor<T> to use Specifications
2.  toPredicate() returning null = no restriction for that spec (not NullPointerException)
3.  toPredicate() called TWICE for Page<T>: data query + count query (different Root instances)
4.  Collection JOIN (OneToMany/ManyToMany) → always query.distinct(true) to prevent duplicates
5.  s1.and(s2.or(s3)) = A AND (B OR C)  vs  s1.and(s2).or(s3) = (A AND B) OR C
6.  cb.conjunction() = 1=1 (safe AND-chain start)  |  cb.disjunction() = 1=0 (safe OR-chain start)
7.  Metamodel (User_) provides compile-time type safety for root.get() and join()
8.  Without metamodel: root.get("field") returns Path<Object> — type errors at runtime only
9.  Use JoinType.LEFT when filtering to preserve parent entities that have no children
10. EXISTS subquery avoids duplicate results and is more efficient for existence checks
11. Specification.where(null) = fetch all (no WHERE clause)
12. Specification.allOf(emptyList) = no WHERE clause (fetch all)
13. Use null-returning specs for optional filters: null input → null predicate → skip
14. Never store Root<T> reference between toPredicate() calls (different instance each call)
15. String-based root.get("field") is validated at execution; metamodel validated at compile time
```

---
