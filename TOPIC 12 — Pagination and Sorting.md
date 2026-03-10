# 🏛️ TOPIC 12 — Pagination and Sorting

---

## 1️⃣ Conceptual Explanation

### Why Pagination Exists — The Memory Problem

Loading all rows from a large table into memory is not viable in production. A table with 10 million orders, each with 20 fields, can consume gigabytes of heap. Pagination solves this by fetching a fixed-size window of results at a time, keeping memory consumption bounded regardless of table size.

Spring Data's pagination abstraction sits above the JPA `setFirstResult()` / `setMaxResults()` API and provides a rich model: page metadata, navigation helpers, count queries, and sorting — all expressed through a clean interface.

---

### The `Pageable` Interface — Internal Architecture

`Pageable` is the **input** abstraction — it tells the query engine what window of data to return and in what order.

```java
public interface Pageable {
    int getPageNumber();      // zero-based page index
    int getPageSize();        // number of elements per page
    long getOffset();         // getPageNumber() * getPageSize()
    Sort getSort();           // sort specification
    Pageable next();          // next page descriptor
    Pageable previousOrFirst();
    Pageable first();
    boolean hasPrevious();
    boolean isPaged();        // true for PageRequest, false for Unpaged
    
    static Pageable unpaged() // Unpaged singleton — no pagination applied
    static Pageable ofSize(int pageSize) // page 0 with given size
}
```

**How Hibernate translates `Pageable` to SQL:**

```
Pageable: page=2, size=10, sort=lastName ASC, firstName DESC

JPQL generated:
  SELECT u FROM User u ORDER BY u.lastName ASC, u.firstName DESC

Hibernate translates to (PostgreSQL/MySQL):
  SELECT u.id, u.last_name, u.first_name, ...
  FROM users u
  ORDER BY u.last_name ASC, u.first_name DESC
  LIMIT 10 OFFSET 20

Hibernate translates to (Oracle pre-12c):
  SELECT * FROM (
    SELECT a.*, ROWNUM rnum FROM (
      SELECT u.id, u.last_name, ... FROM users u
      ORDER BY u.last_name ASC, u.first_name DESC
    ) a WHERE ROWNUM <= 30
  ) WHERE rnum > 20

SQL Server:
  SELECT u.id, u.last_name, ...
  FROM users u
  ORDER BY u.last_name ASC, u.first_name DESC
  OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY
```

The dialect-specific SQL generation is entirely transparent — you write the same `Pageable` regardless of the underlying database.

---

### `PageRequest` — The Primary `Pageable` Implementation

```java
// Factory methods (all zero-based page index):
PageRequest.of(int page, int size)
PageRequest.of(int page, int size, Sort sort)
PageRequest.of(int page, int size, Sort.Direction direction, String... properties)
PageRequest.ofSize(int pageSize)  // page=0 with given size

// Examples:
PageRequest.of(0, 20)
// page 0, 20 items, no sort → LIMIT 20 OFFSET 0

PageRequest.of(2, 10, Sort.by("lastName").ascending())
// page 2, 10 items, sort lastName ASC → LIMIT 10 OFFSET 20

PageRequest.of(0, 50, Sort.Direction.DESC, "createdAt", "id")
// sort by createdAt DESC, then id DESC → LIMIT 50 OFFSET 0

PageRequest.ofSize(25).withPage(3)
// page 3, 25 items → LIMIT 25 OFFSET 75
```

**`PageRequest` immutability:**

`PageRequest` is immutable. `withPage()`, `withSort()`, `withPageSize()` return NEW instances:

```java
PageRequest base = PageRequest.of(0, 20);
PageRequest page2 = base.withPage(1);    // new object, base unchanged
PageRequest sorted = base.withSort(Sort.by("name")); // new object
```

---

### `Page<T>` — The Output Abstraction

`Page<T>` is the **output** of a paginated query. It contains both the current page's data AND metadata about the full dataset.

```java
public interface Page<T> extends Slice<T> {
    int getTotalPages();           // total number of pages
    long getTotalElements();       // total number of matching records
    // inherited from Slice:
    int getNumber();               // current page number (0-based)
    int getSize();                 // page size
    int getNumberOfElements();     // actual elements on THIS page
    List<T> getContent();          // the actual data
    boolean hasContent();
    boolean isFirst();
    boolean isLast();
    boolean hasNext();
    boolean hasPrevious();
    Pageable nextPageable();
    Pageable previousPageable();
}
```

**How `Page<T>` fires two SQL queries:**

```java
Page<User> page = userRepo.findByActive(true, PageRequest.of(0, 20));

// SQL Query 1 — data query:
// SELECT u.* FROM users u WHERE u.active=true ORDER BY ... LIMIT 20 OFFSET 0

// SQL Query 2 — count query (auto-generated):
// SELECT COUNT(u.id) FROM users u WHERE u.active=true

// The count query is ALWAYS fired, even if data query returns fewer than pageSize rows
// (Spring Data cannot know totalElements without the count query)
```

**`Page.map()` — transforming page content:**

```java
Page<User> userPage = userRepo.findByActive(true, pageable);
Page<UserDto> dtoPage = userPage.map(user -> new UserDto(user));
// Preserves: page metadata (totalElements, totalPages, page number, etc.)
// Replaces: content with mapped DTOs
// Returns: new Page<UserDto> with same pagination metadata
```

---

### `Slice<T>` — Efficient "Load More" Pagination

`Slice<T>` trades total count information for performance. It does NOT fire a count query.

```java
public interface Slice<T> {
    int getNumber();
    int getSize();
    int getNumberOfElements();
    List<T> getContent();
    boolean hasContent();
    boolean hasNext();        // key method — is there a next page?
    boolean hasPrevious();
    boolean isFirst();
    boolean isLast();
    Pageable nextPageable();
    Pageable previousPageable();
    
    // NOT available (unlike Page):
    // getTotalElements() — throws UnsupportedOperationException
    // getTotalPages()    — throws UnsupportedOperationException
}
```

**How `Slice<T>` determines `hasNext()` without a count query:**

```java
// For pageSize=20:
// Spring Data requests pageSize+1 = 21 rows from DB:
// SELECT u.* FROM users u WHERE u.active=true LIMIT 21 OFFSET 0

// If 21 rows returned: hasNext() = true, content = first 20 rows (strip extra)
// If <= 20 rows returned: hasNext() = false, content = all returned rows

// This is the ONLY mechanism — no separate COUNT query
```

**`Slice` use cases:**
- Mobile apps with "infinite scroll"
- "Load more" buttons on web pages
- Streaming through large datasets where total count is irrelevant
- Performance-critical pagination where COUNT query is expensive

---

### `Sort` — The Sorting Abstraction

`Sort` represents an ordered list of sort specifications. Each `Sort.Order` specifies a property and direction.

```java
// Simple sorts:
Sort.by("lastName")                          // lastName ASC (default)
Sort.by(Sort.Direction.DESC, "createdAt")    // createdAt DESC
Sort.by("lastName").ascending()
Sort.by("lastName").descending()

// Multiple properties:
Sort.by("lastName", "firstName")             // lastName ASC, firstName ASC
Sort.by(Sort.Direction.ASC, "lastName", "firstName")

// Chained sorts (different directions per property):
Sort.by("lastName").ascending()
    .and(Sort.by("firstName").descending())
    .and(Sort.by("id").ascending())
// ORDER BY last_name ASC, first_name DESC, id ASC

// Sort.Order with full options:
Sort.Order order = Sort.Order.by("lastName")
    .with(Sort.Direction.ASC)
    .withNullHandling(NullHandling.NULLS_LAST)  // NULLS LAST
    .ignoreCase();                               // LOWER(last_name)

Sort sort = Sort.by(order);

// Sort.unsorted():
Sort.unsorted()  // no ORDER BY clause
```

**`Sort.NullHandling` — database null position:**

```java
Sort.Order.by("score")
    .with(NullHandling.NULLS_FIRST)   // NULLs at top of results
    .with(NullHandling.NULLS_LAST)    // NULLs at bottom
    .with(NullHandling.NATIVE)        // use DB default (default)
```

**`Sort` immutability:**

Like `PageRequest`, `Sort` is immutable. `.ascending()`, `.descending()`, `.and()` all return new `Sort` instances.

---

### `JpaSort` — Type-Safe Sorting with Metamodel and Functions

Regular `Sort` validates property names against the entity metamodel at query execution time. `JpaSort` provides additional features:

```java
// JpaSort with metamodel (compile-time safety):
JpaSort.of(User_.lastName)           // uses JPA static metamodel
JpaSort.of(Sort.Direction.DESC, User_.createdAt)

// JpaSort.unsafe() for function-based sorting:
JpaSort.unsafe("LENGTH(u.firstName)")  // sort by string length
JpaSort.unsafe(Sort.Direction.DESC, "LOWER(u.email)")

// unsafe() bypasses property validation — can cause SQL injection if user-controlled
// Only use with hardcoded expressions
```

**Property validation in `Sort`:**

When a `Sort` is applied to a JPQL query, Hibernate validates the property path against the entity metamodel. Invalid property names in `Sort` produce `PropertyReferenceException` at query execution time (not startup):

```java
Sort badSort = Sort.by("nonExistentField");
userRepo.findAll(badSort);
// PropertyReferenceException: No property 'nonExistentField' found for type 'User'
```

---

### `Pageable` in Repository Methods — All Supported Combinations

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Derived query + Pageable → Page:
    Page<User> findByActive(boolean active, Pageable pageable);

    // Derived query + Pageable → Slice (no count query):
    Slice<User> findByActive(boolean active, Pageable pageable);

    // Derived query + Pageable → List (no count, no hasNext info):
    List<User> findByActive(boolean active, Pageable pageable);

    // Derived query + Sort only (no pagination):
    List<User> findByActive(boolean active, Sort sort);

    // @Query + Pageable with auto-generated count query:
    @Query("SELECT u FROM User u WHERE u.active = true AND u.age > :minAge")
    Page<User> findActiveAboveAge(@Param("minAge") int minAge, Pageable pageable);

    // @Query + Pageable with explicit count query:
    @Query(value = "SELECT u FROM User u LEFT JOIN FETCH u.roles WHERE u.active = true",
           countQuery = "SELECT COUNT(u) FROM User u WHERE u.active = true")
    Page<User> findActiveWithRoles(Pageable pageable);

    // JpaRepository.findAll() with Pageable (built-in):
    // Page<User> findAll(Pageable pageable);  ← inherited from JpaRepository
}
```

---

### The Count Query Problem — When Auto-Generation Fails

When you write `@Query("SELECT u FROM User u JOIN FETCH u.roles WHERE u.active = true")` with `Pageable`, Spring Data auto-generates the count query by stripping the JOIN FETCH and replacing with COUNT:

```
Attempt: SELECT COUNT(u) FROM User u JOIN FETCH u.roles WHERE u.active = true
Problem: JOIN FETCH is illegal in COUNT queries (no SELECT on joined entity)
Result:  QueryException at query execution
```

**Always provide `countQuery` when the main query has:**
- `JOIN FETCH` on collections
- `GROUP BY` or `HAVING`
- Complex subqueries that complicate COUNT wrapping
- `DISTINCT` with multi-valued associations

```java
@Query(value = "SELECT DISTINCT u FROM User u JOIN FETCH u.roles WHERE u.active = true",
       countQuery = "SELECT COUNT(DISTINCT u) FROM User u JOIN u.roles WHERE u.active = true")
Page<User> findActiveWithRoles(Pageable pageable);
// countQuery: no FETCH, DISTINCT preserved, correct semantics
```

---

### `Unpaged` — Bypassing Pagination

```java
// Unpaged.INSTANCE — disables pagination:
Page<User> allUsers = userRepo.findByActive(true, Pageable.unpaged());
// SQL: SELECT * FROM users WHERE active=true (no LIMIT, no OFFSET)
// Page<User> returned with content = all results, totalElements = result size

// Use case: when caller wants all data but the method signature requires Pageable
// Avoids having two overloads
```

---

### Cursor-Based Pagination — `ScrollPosition` (Spring Data 3.1+)

Traditional offset pagination has a fundamental problem: if rows are inserted/deleted between pages, items are skipped or duplicated. Cursor-based pagination solves this.

```java
// WindowScrollPosition — cursor using last seen value:
Window<User> window = userRepo.findTop10By(
    ScrollPosition.keyset()  // cursor-based, stable across inserts
);

// Or using the scroll API:
@Query("SELECT u FROM User u WHERE u.active = true ORDER BY u.id ASC")
Window<User> findActive(Pageable pageable, ScrollPosition position);

Window<User> first = userRepo.findActive(PageRequest.ofSize(20), ScrollPosition.offset());
Window<User> next = userRepo.findActive(PageRequest.ofSize(20), first.positionAt(19));
// positionAt(index) captures the cursor at that position for next scroll
```

**`ScrollPosition.offset()` vs `ScrollPosition.keyset()`:**

```
offset() — standard OFFSET-based:
  - Equivalent to traditional LIMIT/OFFSET
  - Fast but unstable (inserts/deletes shift results)

keyset() — cursor-based:
  - Uses WHERE id > :last_seen_id (or composite key)
  - Stable under concurrent inserts/deletes
  - Requires sort by unique column(s)
  - More efficient for deep pagination (no OFFSET scan)
```

---

### Pagination Performance — Deep Offset Problem

A critical performance fact: `OFFSET N` in SQL requires the database to scan and discard N rows before returning results. For `OFFSET 1000000`, the DB must process 1 million rows just to skip them.

```sql
-- Page 1: LIMIT 20 OFFSET 0   → fast, reads 20 rows
-- Page 100: LIMIT 20 OFFSET 1980 → reads 2000 rows, discards 1980
-- Page 100000: LIMIT 20 OFFSET 1999980 → reads 2M rows, discards 1.99M
-- Complexity: O(offset) → becomes O(N) for deep pages
```

**Solutions for deep pagination:**

```java
// Solution 1: Keyset / cursor-based (ScrollPosition.keyset())
// WHERE id > :lastSeenId ORDER BY id ASC LIMIT 20
// DB uses index on id → O(1) regardless of position

// Solution 2: Deferred join (retrieve IDs first, then data)
@Query(value = "SELECT u.id FROM users u WHERE u.active=true ORDER BY u.id LIMIT :size OFFSET :offset",
       nativeQuery = true)
List<Long> findPageIds(@Param("size") int size, @Param("offset") long offset);
// Then: userRepo.findAllById(ids)
// ID-only scan is fast even at deep offsets

// Solution 3: Restrict max page depth in API
// Return 400 Bad Request for page > 1000
// Encourages users toward search (WHERE) rather than scroll
```

---

### `Page` vs `Slice` vs `List` — When to Use Each

```
Page<T>:
  Use when: UI shows total pages/count, traditional pagination controls
  Cost: always fires COUNT query
  When NOT: infinite scroll, mobile, when total count is irrelevant

Slice<T>:
  Use when: "load more" button, infinite scroll, mobile apps
  Cost: fetches pageSize+1 rows (no count query)
  When NOT: when total pages/count is needed for UI

List<T> with Pageable:
  Use when: you need pagination SQL but will compute metadata yourself
  Cost: no metadata at all (no count, no hasNext)
  When NOT: when metadata is needed

Unpaged:
  Use when: method signature requires Pageable but you want all results
  Cost: no LIMIT/OFFSET → full table scan
  When NOT: large tables (memory risk)
```

---

## 2️⃣ Code Examples

### Example 1 — Complete Pagination Implementation

```java
// Entity
@Entity
public class Product {
    @Id @GeneratedValue Long id;
    String name;
    String category;
    BigDecimal price;
    boolean active;
    LocalDateTime createdAt;
}

// Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Page (with count):
    Page<Product> findByActiveTrue(Pageable pageable);

    // Slice (no count):
    Slice<Product> findByCategory(String category, Pageable pageable);

    // List with pagination (no metadata):
    List<Product> findByPriceGreaterThan(BigDecimal price, Pageable pageable);

    // Custom JPQL with count:
    @Query(value = "SELECT p FROM Product p WHERE p.active=true AND p.price < :max",
           countQuery = "SELECT COUNT(p) FROM Product p WHERE p.active=true AND p.price < :max")
    Page<Product> findActiveCheaperThan(@Param("max") BigDecimal max, Pageable pageable);
}

// Service demonstrating all return type variants:
@Service
@Transactional(readOnly = true)
public class ProductService {

    public Map<String, Object> getProductPage(int page, int size, String sortField) {

        // Build Pageable:
        Sort sort = Sort.by(sortField).ascending()
                        .and(Sort.by("id").ascending()); // tiebreaker
        Pageable pageable = PageRequest.of(page, size, sort);

        // Fetch Page<T>:
        Page<Product> productPage = productRepo.findByActiveTrue(pageable);

        // SQL 1: SELECT * FROM products WHERE active=true
        //        ORDER BY ? ASC, id ASC LIMIT ? OFFSET ?
        // SQL 2: SELECT COUNT(p.id) FROM products WHERE active=true

        Map<String, Object> result = new HashMap<>();
        result.put("content", productPage.getContent());
        result.put("currentPage", productPage.getNumber());
        result.put("pageSize", productPage.getSize());
        result.put("totalElements", productPage.getTotalElements());
        result.put("totalPages", productPage.getTotalPages());
        result.put("isFirst", productPage.isFirst());
        result.put("isLast", productPage.isLast());
        result.put("hasNext", productPage.hasNext());
        result.put("hasPrevious", productPage.hasPrevious());
        return result;
    }

    public List<Product> getNextSlice(String category, Pageable lastPageable) {

        // Slice — no count query:
        Slice<Product> slice = productRepo.findByCategory(category, lastPageable);

        // SQL: SELECT * FROM products WHERE category=?
        //      ORDER BY ... LIMIT 21 OFFSET ?  (pageSize+1)

        if (slice.hasNext()) {
            // Get next Pageable for caller:
            Pageable nextPageable = slice.nextPageable();
            // ... return to client for next request
        }
        return slice.getContent();
    }
}
```

---

### Example 2 — Sort Combinations and JpaSort

```java
@Service
public class SortingExamples {

    public void demonstrateSortVariations() {

        // 1. Simple sort by one field:
        Sort s1 = Sort.by("lastName");
        // ORDER BY last_name ASC

        // 2. Descending:
        Sort s2 = Sort.by("createdAt").descending();
        // ORDER BY created_at DESC

        // 3. Multi-field sort (one direction):
        Sort s3 = Sort.by(Sort.Direction.ASC, "lastName", "firstName");
        // ORDER BY last_name ASC, first_name ASC

        // 4. Multi-field with different directions (chained):
        Sort s4 = Sort.by("lastName").ascending()
                      .and(Sort.by("firstName").descending())
                      .and(Sort.by("id").ascending());
        // ORDER BY last_name ASC, first_name DESC, id ASC

        // 5. Null handling:
        Sort.Order orderWithNull = Sort.Order
            .by("score")
            .with(Sort.Direction.DESC)
            .with(Sort.NullHandling.NULLS_LAST);
        Sort s5 = Sort.by(orderWithNull);
        // ORDER BY score DESC NULLS LAST

        // 6. Case-insensitive sort:
        Sort.Order caseInsensitive = Sort.Order
            .by("email")
            .ignoreCase();  // LOWER(email)
        Sort s6 = Sort.by(caseInsensitive);
        // ORDER BY LOWER(email) ASC

        // 7. JpaSort for function-based sort:
        Sort s7 = JpaSort.unsafe(Sort.Direction.ASC, "LENGTH(p.name)");
        // ORDER BY LENGTH(name) ASC

        // 8. Sort.unsorted() — no ORDER BY:
        Sort s8 = Sort.unsorted();
        // No ORDER BY clause

        // 9. Pageable with complex sort:
        Pageable p = PageRequest.of(0, 20, s4);
    }
}

// Repository using sort parameter:
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByDepartment(String dept, Sort sort);
    Page<User> findByDepartment(String dept, Pageable pageable);
}
```

---

### Example 3 — count Query Problems and Solutions

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // PROBLEM 1: JOIN FETCH with Pageable — in-memory pagination:
    // @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.active=true")
    // Page<Order> brokenPagination(Pageable pageable);
    // → HHH90003004 warning, all rows loaded in memory

    // SOLUTION to PROBLEM 1: Separate count query removes JOIN FETCH:
    @Query(value = "SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.active=true",
           countQuery = "SELECT COUNT(o) FROM Order o WHERE o.active=true")
    Page<Order> findActiveWithItems(Pageable pageable);
    // data query:  LIMIT/OFFSET applied correctly (no collection JOIN)
    // count query: efficient, no JOIN FETCH

    // PROBLEM 2: Native query without countQuery:
    // @Query(value = "SELECT o.* FROM orders o WHERE o.total > 1000",
    //        nativeQuery = true)
    // Page<Order> brokenNative(Pageable pageable);
    // → Spring Data wraps: SELECT COUNT(*) FROM (SELECT o.* ...) which fails on some DBs

    // SOLUTION to PROBLEM 2: Explicit native countQuery:
    @Query(value = "SELECT o.* FROM orders o WHERE o.total > 1000 ORDER BY o.created_at DESC",
           countQuery = "SELECT COUNT(*) FROM orders WHERE total > 1000",
           nativeQuery = true)
    Page<Order> findLargeOrdersNative(Pageable pageable);

    // PROBLEM 3: GROUP BY in query, auto-count wraps incorrectly:
    // @Query("SELECT o.customer.id, COUNT(o) FROM Order o GROUP BY o.customer.id")
    // Page<Object[]> brokenGroupBy(Pageable pageable);

    // SOLUTION to PROBLEM 3: Count distinct customers:
    @Query(value = "SELECT o.customer.id, COUNT(o) FROM Order o GROUP BY o.customer.id",
           countQuery = "SELECT COUNT(DISTINCT o.customer.id) FROM Order o")
    Page<Object[]> findOrderCountByCustomer(Pageable pageable);
}
```

---

### Example 4 — Pagination in REST Controller with DTO Mapping

```java
// Request/Response DTOs:
public record PageRequest(
    @Min(0) Integer page,
    @Min(1) @Max(100) Integer size,
    String sortBy,
    String direction
) {}

public record PageResponse<T>(
    List<T> content,
    int pageNumber,
    int pageSize,
    long totalElements,
    int totalPages,
    boolean first,
    boolean last
) {
    public static <T> PageResponse<T> from(Page<T> page) {
        return new PageResponse<>(
            page.getContent(),
            page.getNumber(),
            page.getSize(),
            page.getTotalElements(),
            page.getTotalPages(),
            page.isFirst(),
            page.isLast()
        );
    }
}

@RestController
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping
    public PageResponse<ProductDto> getProducts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "createdAt") String sortBy,
            @RequestParam(defaultValue = "DESC") String direction) {

        // Validate sort field against allowed set (prevent injection):
        Set<String> allowedSort = Set.of("name", "price", "createdAt", "category");
        if (!allowedSort.contains(sortBy)) {
            throw new IllegalArgumentException("Invalid sort field: " + sortBy);
        }

        Sort.Direction dir = Sort.Direction.fromString(direction);
        Pageable pageable = org.springframework.data.domain.PageRequest.of(
            page, size,
            Sort.by(dir, sortBy).and(Sort.by("id").ascending()) // stable sort
        );

        // Fetch and map:
        Page<Product> productPage = productService.findAll(pageable);
        Page<ProductDto> dtoPage = productPage.map(ProductDto::from);
        return PageResponse.from(dtoPage);
    }
}
```

---

### Example 5 — `Slice` for Infinite Scroll

```java
// Repository:
public interface ArticleRepository extends JpaRepository<Article, Long> {
    Slice<Article> findByPublishedTrue(Pageable pageable);
    Slice<Article> findByCategoryAndPublishedTrue(String category, Pageable pageable);
}

// Service:
@Service
@Transactional(readOnly = true)
public class ArticleService {

    // Initial load and "load more" using Slice:
    public Map<String, Object> loadMore(String category, int page, int size) {

        Pageable pageable = PageRequest.of(page, size,
            Sort.by("publishedAt").descending());

        Slice<Article> slice = articleRepo.findByCategoryAndPublishedTrue(
            category, pageable);

        // SQL: SELECT a.* FROM articles a
        //      WHERE a.category=? AND a.published=true
        //      ORDER BY a.published_at DESC
        //      LIMIT 11 OFFSET ?    ← pageSize+1 to detect hasNext

        return Map.of(
            "articles", slice.getContent().stream()
                .map(ArticleDto::from)
                .collect(Collectors.toList()),
            "hasMore", slice.hasNext(),      // true if more pages exist
            "nextPage", slice.hasNext() ? page + 1 : null
            // totalCount NOT available - that's the design
        );
    }
}

// Contrast: Page fires count query every time:
// articleRepo.findByCategoryAndPublishedTrue(category, pageable)
// → SQL 1: SELECT * ... LIMIT 11
// → SQL 2: SELECT COUNT(*) ...  ← extra query Slice avoids
```

---

### Example 6 — Deep Pagination Problem and Keyset Solution

```java
// PROBLEM: Deep offset pagination is slow
@Test
public void demonstrateOffsetProblem() {

    // Page 1: FAST
    Pageable page1 = PageRequest.of(0, 20);
    Page<User> first = userRepo.findAll(page1);
    // SQL: SELECT * FROM users ORDER BY id LIMIT 20 OFFSET 0
    // DB reads 20 rows → fast

    // Page 50000: VERY SLOW
    Pageable deepPage = PageRequest.of(49999, 20);
    Page<User> deep = userRepo.findAll(deepPage);
    // SQL: SELECT * FROM users ORDER BY id LIMIT 20 OFFSET 999980
    // DB reads and discards ~1 million rows to return 20 → O(N) complexity
}

// SOLUTION 1: Two-query approach (ID-based)
@Query("SELECT u.id FROM User u WHERE u.active = true ORDER BY u.id ASC")
Slice<Long> findActiveUserIds(Pageable pageable);

@Transactional(readOnly = true)
public List<UserDto> getPageFast(int page, int size) {
    // Step 1: Get IDs with OFFSET (ID-only scan uses clustered index efficiently)
    Pageable pageable = PageRequest.of(page, size);
    Slice<Long> ids = userRepo.findActiveUserIds(pageable);
    // SQL: SELECT u.id FROM users WHERE active=true ORDER BY id LIMIT 20 OFFSET X
    // Much faster: ID-only index scan

    // Step 2: Load full entities by ID (no OFFSET, just IN clause)
    List<User> users = userRepo.findAllById(ids.getContent());
    return users.stream().map(UserDto::from).collect(Collectors.toList());
}

// SOLUTION 2: Keyset pagination (cursor-based) — Spring Data 3.1+
public interface UserRepository
        extends JpaRepository<User, Long>, CrudRepository<User, Long> {

    // Keyset scroll:
    Window<User> findTop20ByActiveTrue(ScrollPosition position,
                                        Sort sort);
}

@Transactional(readOnly = true)
public Window<User> scrollUsers(ScrollPosition position) {
    return userRepo.findTop20ByActiveTrue(
        position,
        Sort.by("id").ascending()
    );
    // First call: ScrollPosition.keyset() or ScrollPosition.offset()
    // Subsequent: window.positionAt(window.size() - 1)
    // SQL: WHERE id > :lastSeenId ORDER BY id ASC LIMIT 20
    // O(log N) with index — fast at any depth
}
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ**
`PageRequest.of(2, 10)` is passed to a repository method returning `Page<User>`. What `OFFSET` value appears in the generated SQL?

A) 2  
B) 10  
C) 20  
D) 12  

**Answer: C** — `offset = pageNumber * pageSize = 2 * 10 = 20`. Page 0 = rows 0-9, page 1 = rows 10-19, page 2 = rows 20-29. The SQL is `LIMIT 10 OFFSET 20`.

---

**Q2 — Select All That Apply**
Which statements about `Slice<T>` are TRUE? (Select all)

A) `Slice.getTotalElements()` throws `UnsupportedOperationException`  
B) `Slice` fires `pageSize + 1` SQL rows to determine `hasNext()`  
C) `Slice` fires a COUNT query to determine `hasNext()`  
D) `Slice.getTotalPages()` throws `UnsupportedOperationException`  
E) `Slice` is more efficient than `Page` because it avoids the COUNT query  
F) `Slice.hasNext()` is always accurate regardless of concurrent inserts  

**Answer: A, B, D, E** — C is false (no COUNT query). F is partially true but not guaranteed with concurrent modifications — it only reflects the moment the query ran.

---

**Q3 — Code Output Prediction**
```java
Page<User> page = userRepo.findAll(PageRequest.of(0, 20));
// Database has 45 users

System.out.println(page.getTotalPages());     // Line A
System.out.println(page.getTotalElements());  // Line B
System.out.println(page.getNumberOfElements()); // Line C
System.out.println(page.isLast());            // Line D
```

A) 3, 45, 20, false  
B) 2, 45, 20, false  
C) 3, 45, 20, true  
D) 2, 20, 20, false  

**Answer: A** — `totalPages = ceil(45/20) = 3`. `totalElements = 45`. `numberOfElements = 20` (first page is full). `isLast = false` (there are pages 1 and 2 remaining). Page 0: rows 0-19, Page 1: rows 20-39, Page 2: rows 40-44.

---

**Q4 — Sort Direction Default**
```java
Sort sort = Sort.by("lastName", "firstName");
```
What is the sort direction for both fields?

A) DESC for both  
B) ASC for both (default direction)  
C) Natural ordering determined by the DB  
D) Throws `IllegalArgumentException` — direction is required  

**Answer: B** — `Sort.by(String... properties)` defaults to `Sort.Direction.ASC` for all specified properties. `Sort.by("lastName", "firstName")` = `ORDER BY last_name ASC, first_name ASC`.

---

**Q5 — COUNT Query Problem**
```java
@Query("SELECT u FROM User u JOIN FETCH u.roles WHERE u.active = true")
Page<User> findActiveWithRoles(Pageable pageable);
```
What happens when this method is called?

A) Works correctly — Spring Data handles JOIN FETCH with pagination automatically  
B) Returns wrong page sizes — JOIN FETCH causes row multiplication  
C) Loads all matching rows into memory (in-memory pagination), HHH90003004 warning  
D) Throws `QueryException` — JOIN FETCH is not allowed with `Pageable`  

**Answer: C** — JOIN FETCH on a collection with Pageable causes Hibernate to load ALL matching rows into memory and paginate in Java. SQL has no LIMIT/OFFSET. Warning HHH90003004 is logged. Fix: provide explicit `countQuery` and optionally use `@BatchSize` or separate query for roles.

---

**Q6 — `Unpaged` Behavior**
```java
Page<User> result = userRepo.findByActive(true, Pageable.unpaged());
```
Which statements are TRUE? (Select all)

A) No `LIMIT` or `OFFSET` is added to the SQL  
B) Returns a `Page` with `totalElements` equal to all matching records  
C) Throws `UnsupportedOperationException` — `unpaged()` cannot return `Page`  
D) `result.getTotalPages()` returns 1  
E) The count query is NOT fired for `Pageable.unpaged()`  

**Answer: A, B, D, E** — `Pageable.unpaged()` removes pagination from the SQL. All matching results are returned as one page. `totalElements = result size = content size`. `totalPages = 1` (everything fits in one page). No count query is needed because count = content size.

---

**Q7 — Offset Calculation**
```java
Pageable p = PageRequest.of(5, 15);
```
What SQL `LIMIT` and `OFFSET` are generated?

A) `LIMIT 5 OFFSET 15`  
B) `LIMIT 15 OFFSET 75`  
C) `LIMIT 15 OFFSET 5`  
D) `LIMIT 5 OFFSET 75`  

**Answer: B** — `LIMIT = pageSize = 15`. `OFFSET = pageNumber * pageSize = 5 * 15 = 75`. Page 0: 0-14, Page 1: 15-29, Page 2: 30-44, Page 3: 45-59, Page 4: 60-74, Page 5: 75-89.

---

**Q8 — Stable Sorting**
```java
Pageable p1 = PageRequest.of(0, 10, Sort.by("status"));
Pageable p2 = PageRequest.of(1, 10, Sort.by("status"));

// Table has 50 users. Between page 1 and page 2 fetches,
// a new user with status='ACTIVE' is inserted.
// User on position 10 (last of page 1) was status='ACTIVE'.
```

What may happen on the page 2 fetch?

A) Page 2 returns rows 10-19 correctly — OFFSET guarantees position  
B) The new insert may push a row that was at position 10 to position 11, causing that user to appear on BOTH page 1 and page 2  
C) The new insert may cause a row to be skipped entirely  
D) Both B and C — offset pagination is unstable under concurrent modifications  

**Answer: D** — OFFSET pagination is inherently unstable under concurrent modifications. New inserts can cause the same row to appear on multiple pages (phantom reads across pages) or rows to be completely skipped. This is the core problem that keyset/cursor-based pagination solves.

---

**Q9 — `Sort` Parameter with `@Query`**
```java
@Query("SELECT u FROM User u WHERE u.active = true")
List<User> findActive(Sort sort);

// Called with:
Sort badSort = Sort.by("nonExistentField");
userRepo.findActive(badSort);
```

When does the error occur?

A) At compile time — Spring validates sort fields at compile time  
B) At application startup — `PropertyReferenceException`  
C) At query execution time — `PropertyReferenceException`  
D) Never — invalid sort fields are silently ignored  

**Answer: C** — Sort property validation against the entity metamodel happens at **execution time** when the sort is applied to the query. Not at startup. This is different from derived query property validation (which IS at startup). Always validate user-provided sort fields against an allowed set before constructing `Sort`.

---

## 4️⃣ Trick Analysis

**Zero-based page index (Q1, Q7)**:
`PageRequest.of(pageNumber, pageSize)` is **zero-based**. Page 0 = first page. This catches developers who pass `page=1` expecting the first page — they get the second. REST APIs often receive 1-based page numbers from clients; always subtract 1 before creating `PageRequest`. The offset formula is always `pageNumber * pageSize`.

**`Slice.getTotalElements()` throws, not returns 0 (Q2, A)**:
A common misconception is that `Slice.getTotalElements()` returns 0 or -1 when not available. It throws `UnsupportedOperationException`. Code that calls `getTotalElements()` on a `Slice` fails at runtime. The method is intentionally absent from `Slice`'s contract to make this distinction explicit.

**JOIN FETCH + Pageable = silent in-memory pagination (Q5)**:
This is a recurring theme. The warning `HHH90003004` is logged but no exception is thrown. In unit tests with 10 rows, it "works." In production with 100,000 rows, it causes `OutOfMemoryError`. The only reliable fix for fetching collections and paginating: provide an explicit `countQuery` AND avoid JOIN FETCH on collections in the data query (use `@BatchSize` or two separate queries instead).

**Unstable offset pagination under concurrent modification (Q8)**:
`OFFSET N` means "skip N rows as they exist at query time." If a new row is inserted between page fetches at a position that falls before the next offset, all subsequent rows shift by 1, causing the row at the page boundary to appear twice across consecutive pages. If a row is deleted, a row is skipped. This is fundamental SQL behavior, not a Hibernate bug. Keyset pagination solves this with `WHERE id > :lastSeen`.

**Sort field validation at execution time, not startup (Q9)**:
Derived query property names are validated at startup (PartTree parsing). Sort property names passed at runtime (in `Sort` or `Pageable`) are validated when the query executes. This means a sort field typo in user input only surfaces at runtime. Always whitelist allowed sort fields at the API layer before constructing `Sort` objects. User-controlled sort fields are also a potential SQL injection vector with `JpaSort.unsafe()`.

**`Pageable.unpaged()` still returns `Page`, not all results in a list (Q6)**:
`Pageable.unpaged()` does not mean "return everything as a list." It means "don't apply LIMIT/OFFSET to the SQL." The return type is still `Page<T>` (or `Slice<T>`), and `getTotalPages()` returns 1 (everything on one page). No count query is fired because the count equals the content size. This is useful for making a method signature flexible — the same method works with and without pagination.

---

## 5️⃣ Summary Sheet

### `Pageable` Implementations

```
PageRequest.of(page, size)                     — standard page + size
PageRequest.of(page, size, sort)               — with Sort object
PageRequest.of(page, size, direction, fields)  — with direction + field names
PageRequest.ofSize(size)                       — page=0, given size
PageRequest.ofSize(size).withPage(n)           — page n with given size

Pageable.unpaged()     — no LIMIT/OFFSET
Pageable.ofSize(size)  — shortcut for page 0

Offset formula:  offset = pageNumber * pageSize
Example: page=3, size=25 → LIMIT 25 OFFSET 75
```

### `Page<T>` vs `Slice<T>` vs `List<T>` Comparison

| Aspect | `Page<T>` | `Slice<T>` | `List<T>` with Pageable |
|---|---|---|---|
| COUNT query fired? | Always YES | Never | No |
| `getTotalElements()` | YES | `UnsupportedOperationException` | Not available |
| `getTotalPages()` | YES | `UnsupportedOperationException` | Not available |
| `hasNext()` | YES (from count) | YES (pageSize+1 trick) | Not available |
| SQL rows fetched | Exactly pageSize | pageSize + 1 | Exactly pageSize |
| Best for | Traditional pagination UI | Infinite scroll, load-more | Custom metadata handling |

### `Sort` Construction Reference

```java
Sort.by("field")                           // ASC (default)
Sort.by(Direction.DESC, "field")           // DESC
Sort.by("f1", "f2")                        // both ASC
Sort.by(Direction.ASC, "f1", "f2")         // both ASC explicit
Sort.by("f1").ascending()
    .and(Sort.by("f2").descending())       // mixed directions
Sort.by(Sort.Order.by("f")
    .with(Direction.DESC)
    .with(NullHandling.NULLS_LAST)
    .ignoreCase())                         // full options

JpaSort.unsafe("LENGTH(e.name)")           // function-based, no validation
Sort.unsorted()                            // no ORDER BY
```

### Key Pitfalls Table

| Pitfall | Cause | Fix |
|---|---|---|
| Wrong offset for "first page" | Using page=1 for first page | Always use page=0 for first page |
| In-memory pagination | JOIN FETCH on collection + Pageable | Provide explicit `countQuery`, use `@BatchSize` |
| Missing countQuery for native Page | No `countQuery` on native `@Query` | Always provide `countQuery` for native `Page<T>` |
| `Slice.getTotalElements()` throws | Calling unavailable method | Use `Page<T>` if total is needed |
| Sort field injection risk | User-controlled sort fields | Whitelist allowed sort fields before `Sort.by()` |
| Deep offset slowness | Large OFFSET values (O(N) scan) | Use keyset pagination or restrict max page |
| Unstable pagination under writes | OFFSET shifts with concurrent inserts | Use keyset (`ScrollPosition.keyset()`) |

### Count Query Auto-Generation Rules

```
When Spring Data auto-generates countQuery:

  For JPQL @Query:
    - Strips ORDER BY, SELECT clause replaced with COUNT
    - Works for simple queries
    - FAILS for: JOIN FETCH on collections, GROUP BY, DISTINCT with joins
    → Always provide countQuery when query has any of the above

  For native @Query:
    - Wraps in SELECT COUNT(*) FROM (...) x
    - Fails on: older MySQL, some Oracle, DB2, Sybase
    → ALWAYS provide explicit countQuery for native Page<T>

  Safe to omit countQuery:
    - Simple JPQL without JOIN FETCH, GROUP BY, or DISTINCT issues
    - When using Slice<T> (no count query at all)
    - When using Pageable.unpaged() (count = content size)
```

### Key Rules

```
1.  PageRequest is zero-based: page=0 is the first page
2.  offset = pageNumber * pageSize
3.  Page<T> always fires a COUNT query — even if content fits in one page
4.  Slice<T> never fires COUNT — uses pageSize+1 to detect hasNext()
5.  Slice.getTotalElements() throws UnsupportedOperationException
6.  JOIN FETCH collection + Pageable = in-memory pagination (HHH90003004) — dangerous
7.  Always provide countQuery when @Query has JOIN FETCH, GROUP BY, or is native
8.  Sort defaults to ASC direction when no direction specified
9.  Sort field validation happens at EXECUTION time (not startup)
10. JpaSort.unsafe() bypasses metamodel validation — never use with user input
11. Pageable.unpaged() = no LIMIT/OFFSET, returns Page with totalPages=1
12. Page.map(fn) transforms content while preserving all pagination metadata
13. Offset pagination is unstable under concurrent inserts/deletes
14. Deep OFFSET is O(N) — use keyset/cursor-based for large datasets
15. PageRequest is immutable — withPage(), withSort() return new instances
```

---
