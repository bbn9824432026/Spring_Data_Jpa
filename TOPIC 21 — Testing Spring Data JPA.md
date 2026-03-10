# 🏛️ TOPIC 21 — Testing Spring Data JPA

---

## 1️⃣ Conceptual Explanation

### Why JPA Testing Is Different

JPA testing has unique challenges that general unit testing does not:

```
Challenge 1: Database dependency
  → JPA requires a real database (or in-memory substitute)
  → Cannot mock away the database without losing test value

Challenge 2: Transaction boundaries
  → Test data must be set up, test must run, data must be cleaned up
  → Transaction isolation between tests is critical

Challenge 3: Context weight
  → Full Spring context (@SpringBootTest) loads everything
  → JPA tests need only a slice (repositories, entities, JPA config)

Challenge 4: Assertion correctness
  → L1 cache can return in-memory objects instead of hitting DB
  → Must flush + clear PC to truly test persistence behavior

Challenge 5: Test data management
  → Programmatic setup vs SQL scripts vs @Sql
  → Each approach has trade-offs for readability and isolation
```

---

### `@DataJpaTest` — The JPA Test Slice

`@DataJpaTest` loads only the persistence layer — entities, repositories, JPA configuration, and an embedded in-memory database. It explicitly excludes services, controllers, security, and all other beans.

```java
@DataJpaTest
class ProductRepositoryTest {
    // What @DataJpaTest configures:
    // ✓ H2 (or other embedded DB) auto-configured
    // ✓ Hibernate/JPA configured with embedded DB
    // ✓ All @Entity classes scanned
    // ✓ All @Repository classes instantiated
    // ✓ TestEntityManager available
    // ✓ @Transactional on each test method (rollback after)
    // ✓ Spring Data JPA repositories available
    //
    // What @DataJpaTest EXCLUDES:
    // ✗ @Service beans
    // ✗ @Controller beans
    // ✗ @Component beans (unless @Repository)
    // ✗ Security configuration
    // ✗ Web MVC configuration
    // ✗ Full application.properties (uses test-specific auto-config)
}
```

**Internal mechanics of `@DataJpaTest`:**

```
@DataJpaTest is a composed annotation:
  @Target(ElementType.TYPE)
  @Retention(RetentionPolicy.RUNTIME)
  @BootstrapWith(DataJpaTestContextBootstrapper.class)
  @ExtendWith(SpringExtension.class)
  @OverrideAutoConfiguration(enabled = false)
  @TypeExcludeFilters(DataJpaTypeExcludeFilter.class)
  @Transactional
  @AutoConfigureCache
  @AutoConfigureDataJpa
  @AutoConfigureTestDatabase
  @AutoConfigureTestEntityManager
  @ImportAutoConfiguration
  public @interface DataJpaTest

Key behaviors:
  @OverrideAutoConfiguration(enabled=false): disables all auto-configuration
  @AutoConfigureDataJpa: re-enables only JPA-related auto-configuration
  @AutoConfigureTestDatabase: replaces real DataSource with embedded DB
  @Transactional: wraps each test in a transaction, rolled back after
```

---

### `TestEntityManager` — The Testing EntityManager Wrapper

`TestEntityManager` wraps the actual `EntityManager` and adds testing-specific methods that control persistence context state explicitly:

```java
@DataJpaTest
class ProductRepositoryTest {

    @Autowired TestEntityManager tem;
    @Autowired ProductRepository productRepo;

    @Test
    void shouldFindProductByName() {
        // persist(): saves entity and flushes to DB immediately
        Product product = tem.persist(new Product("Widget", new BigDecimal("9.99")));
        // SQL: INSERT INTO products ... (executed immediately)
        // product.getId() is now populated

        // flush(): writes all pending changes to DB (same as em.flush())
        tem.flush();

        // clear(): evicts ALL entities from L1 cache (same as em.clear())
        // CRITICAL: without this, findByName() returns from L1 cache
        // not from DB — test would pass even if DB query was wrong
        tem.clear();

        // Now repository query hits DB (L1 is empty):
        Optional<Product> found = productRepo.findByName("Widget");

        assertThat(found).isPresent();
        assertThat(found.get().getPrice())
            .isEqualByComparingTo(new BigDecimal("9.99"));
    }

    @Test
    void shouldUseFlushAndClearPattern() {
        // Standard test pattern:
        // 1. Arrange: persist test data
        Product p = tem.persistAndFlush(new Product("TestProduct", BigDecimal.ONE));
        // persistAndFlush = persist() + flush() in one call

        // 2. Clear L1 — force next read from DB:
        tem.clear();

        // 3. Act: call the method under test
        Optional<Product> result = productRepo.findByName("TestProduct");

        // 4. Assert: verify DB result
        assertThat(result).isPresent();
    }
}
```

**`TestEntityManager` key methods:**

```java
tem.persist(entity)          // persist to PC (INSERT on flush)
tem.persistAndFlush(entity)  // persist + immediate flush (INSERT now)
tem.persistFlushFind(entity) // persist + flush + find (returns managed entity)
tem.flush()                  // flush pending changes to DB
tem.clear()                  // evict all from L1 cache
tem.find(Class, id)          // em.find() equivalent
tem.getEntityManager()       // access raw EntityManager if needed
tem.getId(entity)            // get the PK of a persisted entity
```

---

### The `flush()` + `clear()` Pattern — Why It's Critical

```java
// WRONG — test passes for wrong reasons:
@Test
void wrong_test() {
    Product product = productRepo.save(new Product("Widget", BigDecimal.ONE));
    // Entity is in L1 cache (managed)

    Optional<Product> found = productRepo.findByName("Widget");
    // Hibernate checks L1 first for JPQL: no... but identity map merges result
    // The SELECT fires but returns the same L1 object
    // Even if findByName() had a SQL bug, this test might pass
    // because the entity is in L1 and result is reconciled with it

    assertThat(found.get().getName()).isEqualTo("Widget"); // passes — but why?
}

// CORRECT — forces DB round-trip:
@Test
void correct_test() {
    Product product = tem.persistAndFlush(new Product("Widget", BigDecimal.ONE));
    // Flush: INSERT SQL executed
    // Entity still in L1

    tem.clear();
    // L1 is now EMPTY — next query MUST go to DB

    Optional<Product> found = productRepo.findByName("Widget");
    // SQL: SELECT * FROM products WHERE name=?
    // Truly tests the DB query and data retrieval

    assertThat(found.get().getName()).isEqualTo("Widget");
    // Now this assertion genuinely proves DB persistence + retrieval
}
```

---

### `@DataJpaTest` Transaction Behavior — Rollback by Default

Each `@DataJpaTest` test method is wrapped in a transaction that **rolls back** after the test completes. This provides automatic test isolation:

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired UserRepository userRepo;
    @Autowired TestEntityManager tem;

    @Test
    void test1() {
        // TX starts at test method entry
        userRepo.save(new User("alice@example.com", "Alice"));
        // User saved in DB (within this TX)
        assertThat(userRepo.count()).isEqualTo(1);
        // TX rolls back after test — DB returns to empty state
    }

    @Test
    void test2() {
        // Fresh TX — DB is empty (previous test's TX rolled back)
        assertThat(userRepo.count()).isEqualTo(0); // ✓
    }
}

// Disable rollback for a specific test (commit to DB):
@Test
@Commit  // overrides default @Rollback(true)
void testWithCommit() {
    userRepo.save(new User("bob@example.com", "Bob"));
    // TX commits — data persists in embedded DB
    // Use sparingly — can cause test pollution if not cleaned up
}

// Explicit rollback annotation:
@Test
@Rollback(false)  // same as @Commit
void testWithRollbackFalse() { }

// Class-level commit (all tests commit):
@DataJpaTest
@Commit
class CommittingTest { }
```

---

### H2 vs Real Database — When Each Is Appropriate

```
H2 Embedded Database (default for @DataJpaTest):
  Pros:
    ✓ No external infrastructure needed
    ✓ Fast startup (in-process)
    ✓ Automatic cleanup (in-memory)
    ✓ Parallel test execution safe
  Cons:
    ✗ Dialect differences from production DB
    ✗ Missing PostgreSQL/MySQL-specific features
      (JSON operators, window functions, specific functions)
    ✗ @Formula with DB-specific SQL fails
    ✗ Native queries may fail (table names, functions differ)
    ✗ Index and constraint behavior differs

Real Database (Testcontainers):
  Pros:
    ✓ Identical to production environment
    ✓ Tests pass = production works
    ✓ All DB features available
    ✓ Native queries work correctly
  Cons:
    ✗ Requires Docker
    ✗ Slower startup (container spin-up)
    ✗ More complex CI setup
```

**When to use each:**

```
Use H2 when:
  ✓ JPQL-only queries (dialect-independent)
  ✓ Standard JPA operations (no native SQL)
  ✓ Fast feedback during development
  ✓ CI with no Docker available

Use Testcontainers when:
  ✓ Native queries with DB-specific syntax
  ✓ @Formula with DB-specific functions
  ✓ JSON column operations
  ✓ Stored procedures
  ✓ DB-specific constraints or index behavior
  ✓ Integration/regression test suites
```

---

### Testcontainers — Real Database in Tests

```java
// Dependency:
// testImplementation "org.testcontainers:postgresql:1.19.x"
// testImplementation "org.testcontainers:junit-jupiter:1.19.x"

// Approach 1: Per-test-class container (slower, full isolation):
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
// replace=NONE: don't replace DataSource with embedded DB — use our DataSource
@Testcontainers
class ProductRepositoryPostgresTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:15")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired ProductRepository productRepo;
    @Autowired TestEntityManager tem;

    @Test
    void shouldUsePostgresSpecificQuery() {
        tem.persistAndFlush(new Product("Widget", new BigDecimal("9.99")));
        tem.clear();

        // This JPQL with PostgreSQL-specific @Formula works now:
        List<Product> products = productRepo.findAllWithComputedFields();
        assertThat(products).isNotEmpty();
    }
}

// Approach 2: Shared container across all tests (faster):
// Base class with static container:
@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
public abstract class BasePostgresTest {

    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:15")
            .withReuse(true);  // reuse container across test classes

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}

// Test class extends base:
class ProductRepositoryTest extends BasePostgresTest {
    @Autowired ProductRepository productRepo;

    @Test
    void test() { ... }
}
```

---

### `@Sql` — Test Data via SQL Scripts

```java
// Inline SQL (before test):
@Test
@Sql("classpath:test-data/products.sql")
void testWithSqlFile() {
    // SQL file executed before test method
    List<Product> products = productRepo.findAll();
    assertThat(products).hasSize(5); // 5 rows in products.sql
}

// Multiple scripts:
@Test
@Sql(scripts = {
    "classpath:test-data/categories.sql",
    "classpath:test-data/products.sql"
})
void testWithMultipleScripts() { ... }

// Explicit phase (BEFORE_TEST_METHOD is default):
@Test
@Sql(scripts = "classpath:cleanup.sql",
     executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
void testWithCleanup() { ... }

// Both before and after:
@Test
@Sql(scripts = "classpath:insert-data.sql")
@Sql(scripts = "classpath:cleanup.sql",
     executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
void testWithSetupAndCleanup() { ... }

// Class-level @Sql (applies to all test methods):
@DataJpaTest
@Sql("classpath:test-data/base-data.sql")
class OrderRepositoryTest {
    // base-data.sql runs before EVERY test method
}

// @SqlConfig for configuring SQL execution:
@Test
@Sql(scripts = "classpath:data.sql",
     config = @SqlConfig(
         encoding = "UTF-8",
         separator = ";",           // statement separator
         commentPrefix = "--",
         transactionMode = SqlConfig.TransactionMode.ISOLATED
         // ISOLATED: run in separate TX (useful for DDL)
         // INLINED: run in current test TX (default)
     ))
void testWithConfig() { }
```

**`src/test/resources/test-data/products.sql`:**

```sql
INSERT INTO categories (id, name) VALUES (1, 'Electronics');
INSERT INTO products (id, name, price, category_id, active)
VALUES (1, 'Widget', 9.99, 1, true);
INSERT INTO products (id, name, price, category_id, active)
VALUES (2, 'Gadget', 19.99, 1, true);
INSERT INTO products (id, name, price, category_id, active)
VALUES (3, 'Doohickey', 4.99, 1, false);
```

---

### `@AutoConfigureTestDatabase` — Controlling Database Choice

```java
// Default (@DataJpaTest): replaces configured DataSource with H2
@DataJpaTest
// Equivalent to:
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.ANY)

// Replace options:
Replace.ANY:         replace any DataSource (including embedded) with test DB
Replace.AUTO_CONFIGURED: replace only auto-configured DataSource
Replace.NONE:        don't replace — use application's configured DataSource

// Use real database from application.properties:
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class RealDbTest {
    // Uses the DataSource configured in application.properties/yml
    // Points to your local/test database server
}

// Specify embedded DB type:
@DataJpaTest
@AutoConfigureTestDatabase(connection = EmbeddedDatabaseConnection.H2)
@AutoConfigureTestDatabase(connection = EmbeddedDatabaseConnection.DERBY)
@AutoConfigureTestDatabase(connection = EmbeddedDatabaseConnection.HSQLDB)
```

---

### Testing with `@MockBean` and `@SpyBean`

`@DataJpaTest` does not load service beans. But sometimes repositories depend on Spring components (e.g., AuditingEntityListener depends on `AuditorAware`). Use `@MockBean` to provide mocked dependencies:

```java
@DataJpaTest
class AuditingRepositoryTest {

    @Autowired ProductRepository productRepo;
    @Autowired TestEntityManager tem;

    // Mock the AuditorAware bean for testing:
    @MockBean AuditorAware<String> auditorAware;

    @BeforeEach
    void setUp() {
        // Configure mock to return a test user:
        given(auditorAware.getCurrentAuditor())
            .willReturn(Optional.of("test-user"));
    }

    @Test
    void shouldSetCreatedByOnSave() {
        Product product = new Product("Widget", BigDecimal.ONE);
        Product saved = tem.persistAndFlush(product);
        tem.clear();

        Product loaded = productRepo.findById(saved.getId()).get();
        assertThat(loaded.getCreatedBy()).isEqualTo("test-user");
        assertThat(loaded.getCreatedAt()).isNotNull();
    }
}

// @SpyBean — partial mock (real implementation + selectively override):
@DataJpaTest
class SpyBeanTest {

    @SpyBean ProductRepository productRepo;

    @Test
    void shouldTrackRepositoryCallCount() {
        tem.persistAndFlush(new Product("Widget", BigDecimal.ONE));
        tem.clear();

        productRepo.findAll();
        productRepo.findAll();

        // Verify real repository was called exactly twice:
        verify(productRepo, times(2)).findAll();
        // @SpyBean wraps real bean in Mockito spy
    }
}
```

---

### Testing Specifications

```java
@DataJpaTest
class ProductSpecificationTest {

    @Autowired ProductRepository productRepo;
    @Autowired TestEntityManager tem;

    @BeforeEach
    void setUp() {
        tem.persistAndFlush(
            new Product("Widget A", new BigDecimal("9.99"), "Electronics", true));
        tem.persistAndFlush(
            new Product("Widget B", new BigDecimal("49.99"), "Electronics", true));
        tem.persistAndFlush(
            new Product("Doohickey", new BigDecimal("5.00"), "Tools", false));
        tem.clear();
    }

    @Test
    void shouldFilterByActiveAndCategory() {
        Specification<Product> spec =
            Specification.where(ProductSpecs.isActive())
                         .and(ProductSpecs.hasCategory("Electronics"));

        List<Product> results = productRepo.findAll(spec);

        assertThat(results).hasSize(2);
        assertThat(results).extracting(Product::getName)
            .containsExactlyInAnyOrder("Widget A", "Widget B");
    }

    @Test
    void shouldReturnAllWhenNoFiltersApplied() {
        Specification<Product> spec = Specification.where(null);
        List<Product> results = productRepo.findAll(spec);
        assertThat(results).hasSize(3);
    }

    @Test
    void shouldFilterByPriceRange() {
        Specification<Product> spec =
            ProductSpecs.priceBetween(
                new BigDecimal("5.00"),
                new BigDecimal("20.00"));

        List<Product> results = productRepo.findAll(spec);
        assertThat(results)
            .extracting(Product::getPrice)
            .allMatch(p -> p.compareTo(new BigDecimal("5.00")) >= 0
                        && p.compareTo(new BigDecimal("20.00")) <= 0);
    }
}
```

---

### Testing Projections

```java
@DataJpaTest
class ProjectionTest {

    @Autowired UserRepository userRepo;
    @Autowired TestEntityManager tem;

    @BeforeEach
    void setUp() {
        tem.persistAndFlush(
            new User("Alice", "Smith", "alice@example.com", "Engineering"));
        tem.persistAndFlush(
            new User("Bob", "Jones", "bob@example.com", "Marketing"));
        tem.clear();
    }

    // Interface projection test:
    @Test
    void shouldReturnClosedProjection() {
        List<UserSummary> summaries = userRepo.findByDepartment("Engineering");

        assertThat(summaries).hasSize(1);
        UserSummary summary = summaries.get(0);
        assertThat(summary.getFirstName()).isEqualTo("Alice");
        assertThat(summary.getLastName()).isEqualTo("Smith");
        // Verify only declared fields accessible:
        // summary.getEmail() would be compile error if not in interface
    }

    // DTO projection test:
    @Test
    void shouldReturnDtoProjection() {
        List<UserDto> dtos = userRepo.findUserDtosByDepartment("Engineering");

        assertThat(dtos).hasSize(1);
        assertThat(dtos.get(0).firstName()).isEqualTo("Alice");
        assertThat(dtos.get(0).email()).isEqualTo("alice@example.com");
    }

    // Dynamic projection test:
    @Test
    void shouldSupportDynamicProjection() {
        List<UserSummary> summaries =
            userRepo.findByDepartment("Engineering", UserSummary.class);
        List<User> full =
            userRepo.findByDepartment("Engineering", User.class);

        assertThat(summaries).hasSize(1);
        assertThat(full).hasSize(1);
        // Both return same data, different representations
    }
}
```

---

### Testing Optimistic Locking

```java
@DataJpaTest
class OptimisticLockingTest {

    @Autowired ProductRepository productRepo;
    @Autowired TestEntityManager tem;

    @Test
    void shouldThrowOnConcurrentModification()
            throws InterruptedException, ExecutionException {
        // Arrange: persist product with version=0
        Product product = tem.persistAndFlush(
            new Product("Widget", new BigDecimal("9.99")));
        Long productId = product.getId();
        tem.clear();

        // Simulate concurrent modification:
        // Load same entity in two "sessions" (simulated with separate finds):
        Product instance1 = productRepo.findById(productId).get();
        Product instance2 = productRepo.findById(productId).get();
        // Both have version=0

        // First update succeeds:
        instance1.setPrice(new BigDecimal("14.99"));
        productRepo.saveAndFlush(instance1);
        // UPDATE ... SET price=14.99, version=1 WHERE id=? AND version=0
        // Affected rows: 1 — success, version now=1 in DB

        // Second update fails — still has version=0:
        instance2.setPrice(new BigDecimal("19.99"));
        assertThatThrownBy(() -> productRepo.saveAndFlush(instance2))
            .isInstanceOf(ObjectOptimisticLockingFailureException.class);
        // UPDATE ... SET price=19.99, version=1 WHERE id=? AND version=0
        // Affected rows: 0 — version mismatch — exception thrown
    }
}
```

---

### Testing Custom Queries and N+1 Detection

```java
@DataJpaTest
class QueryPerformanceTest {

    @Autowired OrderRepository orderRepo;
    @Autowired TestEntityManager tem;

    // Inject Hibernate Statistics for SQL count assertions:
    @Autowired EntityManagerFactory emf;

    private Statistics statistics;

    @BeforeEach
    void setUp() {
        statistics = emf.unwrap(SessionFactory.class).getStatistics();
        statistics.setStatisticsEnabled(true);
        statistics.clear();

        // Create test data:
        Customer customer = tem.persistAndFlush(new Customer("Alice"));
        for (int i = 0; i < 5; i++) {
            Order order = new Order(customer,
                new BigDecimal(10 * (i + 1)));
            for (int j = 0; j < 3; j++) {
                order.addItem(new OrderItem("Item" + j, BigDecimal.ONE));
            }
            tem.persistAndFlush(order);
        }
        tem.clear();
        statistics.clear(); // reset after setup
    }

    @Test
    void shouldNotHaveNPlusOneWithJoinFetch() {
        List<Order> orders = orderRepo.findAllWithItems();
        // @Query("SELECT DISTINCT o FROM Order o JOIN FETCH o.items")

        // Access all items — should not trigger additional queries:
        orders.forEach(o -> o.getItems().size());

        long queryCount = statistics.getQueryExecutionCount();
        assertThat(queryCount)
            .as("Should execute exactly 1 query (JOIN FETCH)")
            .isEqualTo(1);
    }

    @Test
    void demonstratesNPlusOneWithoutFetch() {
        List<Order> orders = orderRepo.findAll();
        // SELECT * FROM orders — 5 rows, items NOT fetched

        orders.forEach(o -> o.getItems().size());
        // Each .size() triggers: SELECT * FROM order_items WHERE order_id=?
        // 5 orders = 5 extra queries = N+1

        long queryCount = statistics.getQueryExecutionCount();
        assertThat(queryCount)
            .as("N+1: 1 order query + 5 items queries")
            .isEqualTo(6); // demonstrates the problem
    }
}
```

---

### `@SpringBootTest` vs `@DataJpaTest` — When to Use Which

```
@DataJpaTest:
  Use when: testing repository methods, query correctness, entity mappings
  Context: persistence layer only (entities, repos, JPA, embedded DB)
  Speed: fast (minimal context)
  Transactions: each test auto-rolled back
  Scope: unit/integration tests for persistence layer

@SpringBootTest:
  Use when: testing cross-layer behavior (service + repo interaction)
  Context: full application context
  Speed: slow (entire application starts)
  Transactions: no auto-rollback (must manage manually)
  Scope: integration tests, end-to-end tests
  
@SpringBootTest with @Transactional:
  Adds auto-rollback to @SpringBootTest tests
  But: service methods that start their own @Transactional run in same TX
  → Changes visible within test, rolled back after
  Caution: @Transactional in test != same TX as @Transactional in service
    if service uses REQUIRES_NEW
```

---

### Testing Auditing

```java
@DataJpaTest
@Import(AuditingConfig.class)  // Import the @EnableJpaAuditing config
class AuditingTest {

    @Autowired ArticleRepository articleRepo;
    @Autowired TestEntityManager tem;

    @MockBean AuditorAware<String> auditorAware;

    @BeforeEach
    void setUp() {
        given(auditorAware.getCurrentAuditor())
            .willReturn(Optional.of("alice"));
    }

    @Test
    void shouldPopulateCreatedAt() {
        Article article = new Article("Spring Testing", "Content...");
        Article saved = tem.persistAndFlush(article);
        tem.clear();

        Article loaded = articleRepo.findById(saved.getId()).get();
        assertThat(loaded.getCreatedAt()).isNotNull();
        assertThat(loaded.getCreatedAt()).isBeforeOrEqualTo(LocalDateTime.now());
        assertThat(loaded.getCreatedBy()).isEqualTo("alice");
    }

    @Test
    void shouldUpdateLastModifiedAt() throws InterruptedException {
        Article article = tem.persistAndFlush(new Article("Title", "Content"));
        LocalDateTime createdAt = article.getCreatedAt();
        tem.clear();

        Thread.sleep(10); // ensure time difference

        Article loaded = articleRepo.findById(article.getId()).get();
        loaded.setContent("Updated content");
        tem.persistAndFlush(loaded);
        tem.clear();

        Article updated = articleRepo.findById(article.getId()).get();
        assertThat(updated.getCreatedAt())
            .isEqualTo(createdAt); // createdAt unchanged
        assertThat(updated.getLastModifiedAt())
            .isAfter(createdAt);   // lastModifiedAt updated
        assertThat(updated.getLastModifiedBy())
            .isEqualTo("alice");
    }
}
```

---

## 2️⃣ Code Examples

### Example 1 — Complete Repository Test Suite

```java
@DataJpaTest
@DisplayName("ProductRepository tests")
class ProductRepositoryTest {

    @Autowired ProductRepository productRepo;
    @Autowired TestEntityManager tem;

    // ── Test data setup ───────────────────────────────────────────────────

    private Product electronics;
    private Product tools;
    private Product inactive;

    @BeforeEach
    void setUp() {
        Category cat = tem.persistAndFlush(new Category("Electronics"));

        electronics = tem.persistAndFlush(
            Product.builder()
                .name("Widget")
                .price(new BigDecimal("9.99"))
                .category(cat)
                .active(true)
                .stock(100)
                .build());

        tools = tem.persistAndFlush(
            Product.builder()
                .name("Hammer")
                .price(new BigDecimal("24.99"))
                .category(cat)
                .active(true)
                .stock(50)
                .build());

        inactive = tem.persistAndFlush(
            Product.builder()
                .name("OldPart")
                .price(new BigDecimal("1.99"))
                .category(cat)
                .active(false)
                .stock(0)
                .build());

        tem.clear(); // Force DB reads in tests
    }

    // ── Basic CRUD tests ──────────────────────────────────────────────────

    @Test
    @DisplayName("should find by id")
    void shouldFindById() {
        Optional<Product> found = productRepo.findById(electronics.getId());
        assertThat(found)
            .isPresent()
            .hasValueSatisfying(p -> {
                assertThat(p.getName()).isEqualTo("Widget");
                assertThat(p.getPrice()).isEqualByComparingTo("9.99");
                assertThat(p.isActive()).isTrue();
            });
    }

    @Test
    @DisplayName("should return empty for unknown id")
    void shouldReturnEmptyForUnknownId() {
        Optional<Product> found = productRepo.findById(999L);
        assertThat(found).isEmpty();
    }

    @Test
    @DisplayName("should find only active products")
    void shouldFindOnlyActive() {
        List<Product> active = productRepo.findByActiveTrue();
        assertThat(active)
            .hasSize(2)
            .extracting(Product::getName)
            .containsExactlyInAnyOrder("Widget", "Hammer")
            .doesNotContain("OldPart");
    }

    // ── Query method tests ─────────────────────────────────────────────────

    @Test
    @DisplayName("should find by price range")
    void shouldFindByPriceRange() {
        List<Product> results = productRepo.findByPriceBetween(
            new BigDecimal("5.00"),
            new BigDecimal("15.00"));

        assertThat(results)
            .hasSize(1)
            .first()
            .extracting(Product::getName)
            .isEqualTo("Widget");
    }

    @Test
    @DisplayName("should count by active status")
    void shouldCountActive() {
        long activeCount = productRepo.countByActive(true);
        long inactiveCount = productRepo.countByActive(false);

        assertThat(activeCount).isEqualTo(2);
        assertThat(inactiveCount).isEqualTo(1);
    }

    // ── Custom @Query tests ────────────────────────────────────────────────

    @Test
    @DisplayName("should find low stock products")
    void shouldFindLowStock() {
        List<Product> lowStock = productRepo.findLowStock(60);
        // @Query("SELECT p FROM Product p WHERE p.stock < :threshold")

        assertThat(lowStock)
            .extracting(Product::getStock)
            .allSatisfy(stock -> assertThat(stock).isLessThan(60));
    }

    // ── @Modifying query tests ─────────────────────────────────────────────

    @Test
    @DisplayName("should bulk deactivate by category")
    void shouldBulkDeactivate() {
        int count = productRepo.deactivateByCategory("Electronics");
        // @Modifying @Query("UPDATE Product p SET p.active=false WHERE ...")

        tem.clear(); // clear cache — bulk update bypasses PC

        assertThat(count).isEqualTo(2); // Widget and Hammer (OldPart already inactive)

        List<Product> stillActive = productRepo.findByActiveTrue();
        assertThat(stillActive).isEmpty();
    }

    // ── Pagination tests ───────────────────────────────────────────────────

    @Test
    @DisplayName("should paginate correctly")
    void shouldPaginate() {
        Page<Product> page = productRepo.findAll(
            PageRequest.of(0, 2, Sort.by("name").ascending()));

        assertThat(page.getTotalElements()).isEqualTo(3);
        assertThat(page.getTotalPages()).isEqualTo(2);
        assertThat(page.getContent()).hasSize(2);
        assertThat(page.getContent().get(0).getName()).isEqualTo("Hammer");
        // Alphabetical: Hammer, OldPart (page 0), Widget (page 1)
    }

    // ── Constraint tests ───────────────────────────────────────────────────

    @Test
    @DisplayName("should throw on duplicate name")
    void shouldThrowOnDuplicateName() {
        Product duplicate = Product.builder()
            .name("Widget")  // same name as electronics
            .price(BigDecimal.ONE)
            .build();

        assertThatThrownBy(() -> tem.persistAndFlush(duplicate))
            .isInstanceOf(PersistenceException.class); // unique constraint violation
    }
}
```

---

### Example 2 — Testcontainers with PostgreSQL

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@ActiveProfiles("test")
class ProductRepositoryPostgresTest {

    // Shared static container — started once for all tests in class:
    @Container
    static PostgreSQLContainer<?> postgres =
        new PostgreSQLContainer<>("postgres:15-alpine")
            .withDatabaseName("test_products")
            .withUsername("test")
            .withPassword("secret")
            .withInitScript("schema.sql"); // optional: custom schema init

    @DynamicPropertySource
    static void configureDataSource(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.datasource.driver-class-name",
                     () -> "org.postgresql.Driver");
        registry.add("spring.jpa.database-platform",
                     () -> "org.hibernate.dialect.PostgreSQLDialect");
    }

    @Autowired ProductRepository productRepo;
    @Autowired TestEntityManager tem;

    @Test
    @DisplayName("should handle PostgreSQL JSONB operations")
    void shouldHandleJsonbMetadata() {
        // Test PostgreSQL-specific JSON operations:
        Product p = new Product("Widget", new BigDecimal("9.99"));
        p.setMetadata(Map.of("color", "blue", "weight", "0.5kg"));
        tem.persistAndFlush(p);
        tem.clear();

        // Test native query with JSONB operator:
        List<Product> found =
            productRepo.findByMetadataContainingKey("color");
        // @Query(value = "SELECT p.* FROM products WHERE p.metadata ? :key",
        //        nativeQuery = true)

        assertThat(found).hasSize(1);
        assertThat(found.get(0).getName()).isEqualTo("Widget");
    }

    @Test
    @DisplayName("should use PostgreSQL-specific formula")
    void shouldComputeFormula() {
        tem.persistAndFlush(new Product("Widget A", new BigDecimal("10.00")));
        tem.persistAndFlush(new Product("Widget B", new BigDecimal("20.00")));
        tem.clear();

        List<Product> products = productRepo.findAll();
        // @Formula("CONCAT(UPPER(name), ' [', price::text, ']')")
        // PostgreSQL-specific :: casting — works only with real PostgreSQL
        products.forEach(p ->
            assertThat(p.getDisplayLabel()).matches("WIDGET .+ \\[.+\\]"));
    }
}
```

---

### Example 3 — `@Sql` with Test Data Management

```java
// src/test/resources/sql/orders-setup.sql:
/*
INSERT INTO customers (id, name, email) VALUES (1, 'Alice', 'alice@test.com');
INSERT INTO orders (id, customer_id, total, status)
VALUES (1, 1, 99.99, 'PENDING');
INSERT INTO orders (id, customer_id, total, status)
VALUES (2, 1, 199.99, 'COMPLETED');
INSERT INTO order_items (id, order_id, product_name, quantity, unit_price)
VALUES (1, 1, 'Widget', 2, 49.995);
INSERT INTO order_items (id, order_id, product_name, quantity, unit_price)
VALUES (2, 2, 'Gadget', 1, 199.99);
*/

// src/test/resources/sql/orders-cleanup.sql:
/*
DELETE FROM order_items;
DELETE FROM orders;
DELETE FROM customers;
*/

@DataJpaTest
class OrderRepositoryTest {

    @Autowired OrderRepository orderRepo;

    @Test
    @Sql("classpath:sql/orders-setup.sql")
    @Sql(scripts = "classpath:sql/orders-cleanup.sql",
         executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
    @Rollback(false) // Using @Sql for cleanup instead of rollback
    void shouldFindPendingOrders() {
        List<Order> pending = orderRepo.findByStatus(OrderStatus.PENDING);
        assertThat(pending)
            .hasSize(1)
            .first()
            .satisfies(o -> {
                assertThat(o.getTotal()).isEqualByComparingTo("99.99");
                assertThat(o.getItems()).hasSize(1);
            });
    }

    @Test
    @Sql({
        "classpath:sql/orders-setup.sql"
    })
    void shouldCalculateTotalRevenue() {
        // Uses @Sql data for this test (rolled back automatically after):
        BigDecimal revenue = orderRepo.calculateTotalRevenue();
        assertThat(revenue).isEqualByComparingTo("299.98"); // 99.99 + 199.99
    }
}
```

---

### Example 4 — Testing Locking Behavior

```java
@DataJpaTest
class LockingTest {

    @Autowired InventoryRepository inventoryRepo;
    @Autowired TestEntityManager tem;

    @Test
    @DisplayName("should apply PESSIMISTIC_WRITE lock")
    void shouldApplyPessimisticWriteLock() {
        Inventory inv = tem.persistAndFlush(
            new Inventory(1L, 100));
        tem.clear();

        // Verify @Lock annotation applies FOR UPDATE:
        // (In H2, FOR UPDATE is supported)
        Optional<Inventory> locked =
            inventoryRepo.findByIdForUpdate(inv.getId());

        assertThat(locked).isPresent();
        assertThat(locked.get().getQuantity()).isEqualTo(100);
        // Cannot directly assert FOR UPDATE in H2 test
        // Use Testcontainers + pg_locks view for lock verification in PostgreSQL
    }

    @Test
    @DisplayName("should increment version on update")
    void shouldIncrementVersion() {
        Product product = tem.persistAndFlush(
            new Product("Widget", BigDecimal.ONE));
        Integer initialVersion = product.getVersion();
        tem.clear();

        Product loaded = productRepo.findById(product.getId()).get();
        assertThat(loaded.getVersion()).isEqualTo(initialVersion);

        loaded.setName("Updated Widget");
        tem.persistAndFlush(loaded);
        tem.clear();

        Product updated = productRepo.findById(product.getId()).get();
        assertThat(updated.getVersion())
            .isEqualTo(initialVersion + 1);
    }
}
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ: `flush()` + `clear()` necessity**
```java
@DataJpaTest
class ProductTest {
    @Autowired ProductRepository productRepo;
    @Autowired TestEntityManager tem;

    @Test
    void test() {
        tem.persistAndFlush(new Product("Widget", BigDecimal.ONE));
        // No tem.clear()

        Optional<Product> found = productRepo.findByName("Widget");
        assertThat(found.get().getName()).isEqualTo("Widget");
    }
}
```

What is the problem with this test?

A) `persistAndFlush` is not available — use `persist` + `flush` separately  
B) Test passes but doesn't prove DB query works — L1 cache may serve the result  
C) Test will fail — `findByName` always returns empty without `clear()`  
D) No problem — `persistAndFlush` automatically clears L1 cache  

**Answer: B** — Without `tem.clear()`, the entity is still in the L1 (Persistence Context) cache. When `findByName` executes JPQL, Hibernate fires the SQL, but the returned entity is reconciled with the L1 identity map — the in-memory object is returned. Even if the SQL or mapping was incorrect, the test might pass because the entity data comes from L1. The test is testing cache behavior, not DB persistence.

---

**Q2 — Select All That Apply: `@DataJpaTest`**

Which are TRUE about `@DataJpaTest`?

A) Each test method runs in a transaction that rolls back after the test  
B) `@Service` beans are available for injection in `@DataJpaTest` tests  
C) `@DataJpaTest` replaces the configured DataSource with H2 by default  
D) `TestEntityManager` is automatically available  
E) `@DataJpaTest` loads the full Spring application context  
F) `@MockBean` can be used inside `@DataJpaTest` tests  

**Answer: A, C, D, F**
- B is false: `@Service` beans are excluded from the slice — use `@SpringBootTest` for full context
- E is false: `@DataJpaTest` loads only the persistence slice, not the full context

---

**Q3 — Transaction rollback in tests**
```java
@DataJpaTest
class CountTest {
    @Autowired UserRepository userRepo;

    @Test
    void test1() {
        userRepo.save(new User("alice@test.com", "Alice"));
        assertThat(userRepo.count()).isEqualTo(1);
    }

    @Test
    void test2() {
        assertThat(userRepo.count()).isEqualTo(???);
    }
}
```

What is `userRepo.count()` in `test2()`?

A) 1 — `test1()` committed the user  
B) 0 — `test1()`'s transaction rolled back after the test  
C) Depends on test execution order  
D) Throws `IllegalStateException` — cannot reuse the same repository across tests  

**Answer: B** — Each `@DataJpaTest` test method runs in a transaction that automatically rolls back after the test completes. The `User` saved in `test1()` is never committed to the database — its transaction rolls back when `test1()` finishes. `test2()` starts with a fresh transaction and an empty database (assuming no other data setup). Count is 0.

---

**Q4 — `@AutoConfigureTestDatabase` Replace**
```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class MyRepoTest { }
```

What database is used?

A) H2 embedded database (always)  
B) The DataSource configured in `application.properties` / `application.yml`  
C) A randomly selected embedded database (H2, HSQLDB, or Derby)  
D) No database — `Replace.NONE` disables the datasource  

**Answer: B** — `Replace.NONE` instructs Spring not to replace the configured DataSource with an embedded database. The test uses whatever DataSource is defined in the application's configuration files. This is how you use Testcontainers or a real test database with `@DataJpaTest` — configure the DataSource via `@DynamicPropertySource` and use `Replace.NONE`.

---

**Q5 — `@Sql` Execution Phase**
```java
@Test
@Sql(scripts = "classpath:test-data.sql")
@Sql(scripts = "classpath:cleanup.sql",
     executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
@Rollback(false)
void myTest() {
    // test logic
}
```

In what order do events occur?

A) cleanup.sql → test-data.sql → test runs → transaction commits  
B) test-data.sql → test runs → cleanup.sql → transaction commits  
C) test-data.sql → test runs → transaction commits → cleanup.sql  
D) test-data.sql → transaction commits → test runs → cleanup.sql  

**Answer: C** — With `@Rollback(false)` (commit enabled): `BEFORE_TEST_METHOD` SQL (`test-data.sql`) runs first, then the test method executes, then the transaction commits, then `AFTER_TEST_METHOD` SQL (`cleanup.sql`) runs in a new transaction. The cleanup SQL runs AFTER the commit of the test transaction.

---

**Q6 — Hibernate Statistics for N+1 Detection**
```java
@Test
void testQueryCount() {
    statistics.clear();

    List<Order> orders = orderRepo.findAll();          // 1 query
    orders.forEach(o -> o.getItems().size());          // N queries

    long count = statistics.getQueryExecutionCount();
}
```

If `findAll()` returns 10 orders and `items` is lazy, what is `count`?

A) 1 — lazy loading doesn't count as queries in statistics  
B) 10 — only the N lazy queries count (not the main query)  
C) 11 — 1 main query + 10 lazy collection queries  
D) Depends on L1 cache state  

**Answer: C** — `getQueryExecutionCount()` counts ALL SQL queries including lazy-loaded collections. 1 query for `findAll()` + 10 queries for each `o.getItems().size()` call = 11 total. This is the N+1 problem in numbers. The test demonstrates why `JOIN FETCH` or `@BatchSize` is needed.

---

**Q7 — `@Import` in `@DataJpaTest`**
```java
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class AuditingConfig {
    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.of("system");
    }
}

@DataJpaTest
// Missing something for auditing to work
class AuditTest {
    @Autowired ArticleRepository articleRepo;

    @Test
    void shouldSetCreatedAt() {
        Article a = articleRepo.save(new Article("Title"));
        assertThat(a.getCreatedAt()).isNotNull(); // FAILS
    }
}
```

Why does `getCreatedAt()` return null?

A) `AuditingEntityListener` is not on the entity  
B) `AuditingConfig` is not loaded — `@DataJpaTest` doesn't load `@Configuration` classes outside the slice  
C) `@MockBean AuditorAware` must be declared in the test  
D) `getCreatedAt()` is always null until the entity is re-fetched  

**Answer: B** — `@DataJpaTest` loads only the persistence slice. Custom `@Configuration` classes like `AuditingConfig` are NOT automatically included. To use auditing in a `@DataJpaTest`, you must explicitly import the configuration: `@DataJpaTest @Import(AuditingConfig.class)`. Without `@Import`, `@EnableJpaAuditing` is never processed, and the auditing infrastructure is not set up.

---

**Q8 — `TestEntityManager.persistFlushFind()`**
```java
Product product = tem.persistFlushFind(new Product("Widget", BigDecimal.ONE));
```

What does `persistFlushFind()` do in sequence?

A) `persist()` only — flush and find are deferred  
B) `persist()` → `flush()` — no automatic find  
C) `persist()` → `flush()` → `clear()` → `find()` — returns reloaded entity from DB  
D) `persist()` → `find()` — skips flush (entity returned from L1)  

**Answer: C** — `persistFlushFind()` does three things: `persist()` (save to PC), `flush()` (write SQL to DB), `clear()` (evict from L1), then `find()` (reload from DB). The returned entity is a fresh instance loaded from the database — not the same Java object as the input. This is the most comprehensive test setup method when you need to verify the entity was correctly persisted AND correctly loaded.

---

## 4️⃣ Trick Analysis

**`flush()` + `clear()` omission = testing L1 cache, not DB (Q1)**:
This is the single most common `@DataJpaTest` mistake. Developers save an entity and immediately query it, seeing the correct value — but the value came from L1 cache, not the database. A query with a typo in the WHERE clause, a wrong column mapping, or a missing `@Column` annotation would still "pass" because the entity is in L1. Always `tem.clear()` before assertions that should verify database state. `persistAndFlush()` writes to DB but does NOT clear L1 — you must clear explicitly.

**`@DataJpaTest` excludes `@Service` and `@Configuration` outside the slice (Q7)**:
`@DataJpaTest` is deliberately narrow. It includes only: `@Repository` beans, `@Entity` classes, JPA/Hibernate auto-configuration. Any `@Configuration` class you wrote (like `AuditingConfig`) is not in the slice. The exception: `@Configuration` classes that are specifically designed for the JPA layer and auto-detected. For anything else, use `@Import(YourConfig.class)` explicitly. This is the source of "my auditing works in the app but not in tests" bugs.

**Transaction rollback = 0 count in next test (Q3)**:
The `@Transactional` annotation on `@DataJpaTest` means ROLLBACK by default. Each test method starts with the DB state left by `@BeforeEach` (which runs in the SAME transaction as the test) and ends with rollback. The next test starts fresh. This is excellent for test isolation but surprises developers who expect committed data to persist between tests. `@Commit` / `@Rollback(false)` must be explicit.

**`persistFlushFind()` clears L1 — returns DB-fresh entity (Q8)**:
This three-step method is specifically designed for tests that need to verify both write and read correctness. Many developers use only `persistAndFlush()` and get back the same Java object they passed in (still in L1). `persistFlushFind()` clears L1 and returns a newly loaded entity from the database — any field mapping bug, missing column, or wrong conversion would be caught. Prefer `persistFlushFind()` when you need to assert on the state of an entity after it has been saved to and reloaded from the database.

**`@Import` for `@DataJpaTest` slice extensions (Q7)**:
The JPA test slice is strict. When your persistence layer depends on Spring beans outside the default slice (custom `@EnableJpaAuditing`, custom `@Converter` beans that use Spring injection, etc.), you must explicitly import those configurations. `@Import(SomeConfig.class)` adds that configuration to the slice without loading the full application context. `@MockBean` is the other option for dependencies that can be mocked — use `@MockBean AuditorAware<String>` instead of importing the full auditing config.

**Hibernate Statistics for SQL counting (Q6)**:
`statistics.getQueryExecutionCount()` counts every SQL statement Hibernate sends to the database — including lazy-loaded collections, JOIN FETCHes, and count queries. This makes it a powerful tool for detecting N+1 in tests. Pattern: `statistics.clear()` before the code under test → execute code → assert exact query count. This approach is more reliable than SQL log inspection in CI environments. Combine with `setStatisticsEnabled(true)` — statistics are disabled by default for performance.

---

## 5️⃣ Summary Sheet

### `@DataJpaTest` vs `@SpringBootTest`

| Aspect | `@DataJpaTest` | `@SpringBootTest` |
|---|---|---|
| Context size | Persistence slice only | Full application |
| Speed | Fast | Slow |
| Beans available | `@Repository`, entities, JPA config | All beans |
| Default DB | H2 embedded | Configured DataSource |
| Auto-rollback | YES (per test method) | NO |
| `TestEntityManager` | Auto-configured | Not available |
| Use for | Repository, query, entity tests | Cross-layer integration |

### `TestEntityManager` Method Reference

```java
tem.persist(entity)          // persist → PC (INSERT on flush)
tem.persistAndFlush(entity)  // persist + flush (INSERT now), entity in L1
tem.persistFlushFind(entity) // persist + flush + clear + find (DB-fresh entity)
tem.flush()                  // flush pending SQL to DB
tem.clear()                  // evict all from L1 (CRITICAL for valid tests)
tem.find(Class, id)          // load by PK (checks L1 first)
tem.getId(entity)            // get entity PK value
tem.getEntityManager()       // raw EntityManager access
```

### The `persistFlushFind` vs `persistAndFlush` Choice

```
persistAndFlush():
  → persist + flush
  → Entity REMAINS in L1 cache
  → Subsequent queries may return L1 object (not DB)
  → Use when: you only need to test write behavior
              or will call tem.clear() manually afterward

persistFlushFind():
  → persist + flush + clear + find
  → Returns FRESH entity from DB
  → L1 cache cleared — subsequent queries go to DB
  → Use when: you need to verify both write AND read correctness
              or testing field mappings, converters, @Formula
```

### `@Sql` Execution Phases

```
ExecutionPhase.BEFORE_TEST_METHOD  (default): SQL runs before test method
ExecutionPhase.AFTER_TEST_METHOD:             SQL runs after test method
ExecutionPhase.BEFORE_TEST_CLASS:             SQL runs once before all tests in class
ExecutionPhase.AFTER_TEST_CLASS:              SQL runs once after all tests in class
```

### Key Rules

```
1.  Always tem.clear() before asserting DB-queried results — prevents L1 cache false positives
2.  @DataJpaTest auto-rollback: each test starts with DB state from @BeforeEach
3.  @Commit / @Rollback(false): explicit commit within @DataJpaTest test
4.  @DataJpaTest excludes @Service, @Controller, custom @Configuration by default
5.  @Import(SomeConfig.class): include specific config in @DataJpaTest slice
6.  @MockBean: create mock beans available in @DataJpaTest slice context
7.  Replace.NONE: use real DataSource (for Testcontainers or real DB tests)
8.  @DynamicPropertySource: inject Testcontainers connection properties at runtime
9.  persistFlushFind(): persist + flush + clear + DB reload — most thorough setup
10. statistics.clear() + statistics.getQueryExecutionCount(): detect N+1 in tests
11. @Sql: load test data from SQL files; phase controls before/after timing
12. @Sql with @Rollback(false): combine SQL cleanup with explicit commit pattern
13. @BeforeEach runs in SAME transaction as test method (both rolled back together)
14. @DataJpaTest does NOT scan @Component beans (only @Repository and related)
15. Testcontainers + @DynamicPropertySource: real DB integration without @SpringBootTest
16. assertThatThrownBy(()->...).isInstanceOf(PersistenceException.class): constraint tests
17. statistics must be enabled: statistics.setStatisticsEnabled(true) in setUp
18. JPA auditing in tests: @Import(AuditingConfig.class) + @MockBean AuditorAware<String>
19. @SpyBean: wraps real repository bean in Mockito spy for call verification
20. @DataJpaTest default: Replace.ANY = H2 replaces any configured DataSource
```

---

This completes the full **Spring Data JPA Mastery curriculum** — all 21 topics from foundational JPA architecture through advanced testing strategies. You now have architect-level coverage of every major domain:

- **Topics 1–5**: Entity mapping, lifecycle, inheritance, embeddables
- **Topics 6–8**: Associations, lazy loading, N+1 problem
- **Topics 9–12**: Repository architecture, queries, pagination, sorting
- **Topics 13–15**: Specifications, transactions, locking
- **Topics 16–18**: Auditing, Envers, projections, caching
- **Topics 19–21**: Advanced Hibernate, Spring Data REST, testing
