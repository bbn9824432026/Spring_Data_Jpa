# 🏛️ TOPIC 9 — Repository Abstraction Architecture

---

## 1️⃣ Conceptual Explanation

### The Repository Pattern — Why It Exists

The Repository pattern (from Eric Evans' Domain-Driven Design) defines a collection-like interface for accessing domain objects. It abstracts the data access layer, presenting a clean API to the domain layer while hiding all persistence mechanics.

Spring Data JPA takes this pattern and **generates the implementation at runtime** — you declare an interface, Spring Data provides a fully functional implementation without you writing a single line of persistence code.

Understanding HOW this generation works — the proxy creation, the query compilation, the method routing — is what this topic is about.

---

### The Repository Hierarchy — Complete Inheritance Chain

```
java.lang.Object
  └── org.springframework.data.repository.Repository<T, ID>       ← marker interface (empty)
        └── CrudRepository<T, ID>                                  ← basic CRUD
              ├── PagingAndSortingRepository<T, ID>                ← pagination + sorting
              │     └── JpaRepository<T, ID>                       ← JPA-specific extensions
              └── ListCrudRepository<T, ID>                        ← returns List (SD 3.0+)
                    └── ListPagingAndSortingRepository<T, ID>      ← List + paging (SD 3.0+)

Reactive variants (not JPA):
  ReactiveCrudRepository, RxJava3CrudRepository...

JPA-specific executor interfaces:
  JpaSpecificationExecutor<T>     ← Criteria API / Specification support
  QuerydslPredicateExecutor<T>    ← Querydsl support
  QueryByExampleExecutor<T>       ← Query By Example support
```

---

### Interface-by-Interface Deep Analysis

#### `Repository<T, ID>` — The Marker Interface

```java
// Complete source code:
@Indexed
public interface Repository<T, ID> {
    // Empty — purely a marker interface
}
```

`T` = the domain type (entity class)
`ID` = the type of the entity's identifier

This marker interface serves two purposes:
1. **Type parameter capture** — Spring Data reads `T` and `ID` at startup to understand what entity and ID type this repository manages
2. **Spring component scanning** — `@EnableJpaRepositories` scans for interfaces extending `Repository` (directly or transitively)

You can extend `Repository` directly to create a **minimal repository** that exposes only the methods you define:

```java
// Explicitly minimal — only exposes findById and save:
public interface UserRepository extends Repository<User, Long> {
    Optional<User> findById(Long id);
    User save(User user);
    // deleteById is NOT exposed — consumer cannot delete
}
// Security/API design pattern: restrict what operations are available
```

---

#### `CrudRepository<T, ID>` — CRUD Operations

```java
public interface CrudRepository<T, ID> extends Repository<T, ID> {
    
    <S extends T> S save(S entity);                    // persist or merge
    <S extends T> Iterable<S> saveAll(Iterable<S> entities);
    
    Optional<T> findById(ID id);
    boolean existsById(ID id);
    Iterable<T> findAll();
    Iterable<T> findAllById(Iterable<ID> ids);
    
    long count();
    
    void deleteById(ID id);
    void delete(T entity);
    void deleteAllById(Iterable<? extends ID> ids);
    void deleteAll(Iterable<? extends T> entities);
    void deleteAll();
}
```

**`save()` internal behavior — critical:**

```java
// SimpleJpaRepository.save() source (simplified):
@Transactional
public <S extends T> S save(S entity) {
    
    Assert.notNull(entity, "Entity must not be null");
    
    if (entityInformation.isNew(entity)) {
        em.persist(entity);
        return entity;
    } else {
        return em.merge(entity);
    }
}
```

`isNew()` determination chain:
```
1. Does entity implement Persistable<ID>?
   → YES: use entity.isNew() method — developer controls the logic
   → NO: continue

2. Does entity have @Version field?
   → YES: is version null? → isNew=true; version non-null → isNew=false
   → NO: continue

3. Does entity have @Id field?
   → Is @Id field null?   → isNew=true  (persist)
   → Is @Id field non-null → isNew=false (merge)

4. Is @Id a primitive (int, long)?
   → Is value == 0?       → isNew=true
   → Is value != 0?       → isNew=false
```

**`saveAll()` internal behavior:**

```java
@Transactional
public <S extends T> List<S> saveAll(Iterable<S> entities) {
    List<S> result = new ArrayList<>();
    for (S entity : entities) {
        result.add(save(entity)); // calls save() per entity
    }
    return result;
}
```

`saveAll()` iterates and calls `save()` per entity — it is NOT a single bulk INSERT. Whether JDBC batching occurs depends on:
- Generator strategy (SEQUENCE enables batching, IDENTITY disables it)
- `hibernate.jdbc.batch_size` configuration
- No cascade differences per entity

```properties
# Enable JDBC batching:
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true
# order_inserts/order_updates: sorts actions by entity type
# Required for batch INSERT to group same-type inserts together
```

**`deleteById()` internals:**

```java
@Transactional
public void deleteById(ID id) {
    Assert.notNull(id, "The given id must not be null");
    
    delete(findById(id).orElseThrow(() -> 
        new EmptyResultDataAccessException(
            String.format("No %s entity with id %s exists", 
                          entityInformation.getJavaType(), id), 1)));
}
```

`deleteById()` first calls `findById()` to load the entity, THEN calls `delete()`. This means:
1. One SELECT before the DELETE
2. If entity doesn't exist → `EmptyResultDataAccessException`

If you want to avoid the SELECT: use `@Modifying @Query("DELETE FROM ...")` instead.

---

#### `PagingAndSortingRepository<T, ID>` — Pagination

```java
public interface PagingAndSortingRepository<T, ID> extends Repository<T, ID> {
    Iterable<T> findAll(Sort sort);
    Page<T> findAll(Pageable pageable);
}
```

Note: In Spring Data 3.x (Spring Boot 3.x), `PagingAndSortingRepository` no longer extends `CrudRepository`. It extends `Repository` directly. This means if you extend only `PagingAndSortingRepository`, you do NOT get CRUD methods. Extend `JpaRepository` to get everything.

---

#### `JpaRepository<T, ID>` — JPA-Specific Extensions

```java
public interface JpaRepository<T, ID> 
    extends ListCrudRepository<T, ID>, 
            ListPagingAndSortingRepository<T, ID>,
            QueryByExampleExecutor<T> {
    
    // Flush operations:
    void flush();
    <S extends T> S saveAndFlush(S entity);
    <S extends T> List<S> saveAllAndFlush(Iterable<S> entities);
    
    // Batch delete (more efficient than deleteAll):
    void deleteAllInBatch(Iterable<T> entities);
    void deleteAllByIdInBatch(Iterable<ID> ids);
    void deleteAllInBatch();
    
    // Reference loading:
    T getReferenceById(ID id);  // formerly getOne() — returns proxy, no SELECT
    
    // Returns List<T> (vs Iterable from CrudRepository):
    List<T> findAll();
    List<T> findAll(Sort sort);
    List<T> findAllById(Iterable<ID> ids);
}
```

**`getReferenceById()` vs `findById()`:**

```java
// findById() — eager load:
Optional<User> user = userRepo.findById(1L);
// SQL: SELECT * FROM users WHERE id=1
// Returns actual User or empty Optional

// getReferenceById() — proxy only:
User proxy = userRepo.getReferenceById(1L);
// NO SQL — returns Hibernate proxy
// proxy.getId() → 1L (no SQL)
// proxy.getName() → triggers SELECT (if in open Session)

// Use case: setting FK relationships without loading:
Order order = new Order();
order.setCustomer(customerRepo.getReferenceById(customerId));
// No customer SELECT — just sets the proxy as FK reference
// At flush: INSERT INTO orders (customer_id=?) — only FK needed
```

**`deleteAllInBatch()` vs `deleteAll()`:**

```java
// deleteAll() — loads all entities first, then deletes one by one:
userRepo.deleteAll();
// SQL: SELECT * FROM users (loads all!)
// SQL: DELETE FROM users WHERE id=1
// SQL: DELETE FROM users WHERE id=2
// ... N individual DELETEs

// deleteAllInBatch() — single SQL DELETE:
userRepo.deleteAllInBatch();
// SQL: DELETE FROM users
// No entity loading, no cascade processing, no lifecycle callbacks
// Much faster, but: cascade and orphanRemoval are NOT triggered
```

---

### How Spring Data Creates Repository Proxies — The Full Internal Sequence

This is the most important internal mechanism to understand. At application startup, Spring Data creates a **JDK dynamic proxy** for every repository interface. Here is the complete sequence:

```
Application startup:
  1. @EnableJpaRepositories triggers JpaRepositoriesRegistrar
  2. JpaRepositoriesRegistrar registers JpaRepositoryFactoryBean definition
     for each discovered repository interface
  
  3. Spring creates JpaRepositoryFactoryBean beans
  4. JpaRepositoryFactoryBean.afterPropertiesSet() is called
  5. Creates JpaRepositoryFactory (extends RepositoryFactorySupport)
  6. JpaRepositoryFactory.getRepository(repositoryInterface) is called:
  
     a. Determine repository metadata:
        - Extract T and ID from generic type parameters
        - Determine entity type, ID type
        - Validate entity has @Id
     
     b. Determine base class (SimpleJpaRepository or custom):
        - Is there a custom Impl class? (e.g., UserRepositoryImpl)
        - What is the repositoryBaseClass configuration?
     
     c. Compile all query methods:
        - Scan all methods in repository interface
        - For each method: create RepositoryQuery object
          * @Query present? → SimpleJpaQuery or NativeJpaQuery
          * Name matches derived query pattern? → PartTreeJpaQuery
          * Matches stored procedure? → StoredProcedureJpaQuery
        - Validate JPQL at startup (if possible)
     
     d. Create JDK dynamic proxy:
        ProxyFactory proxyFactory = new ProxyFactory();
        proxyFactory.setTarget(new SimpleJpaRepository<>(entityInfo, em));
        proxyFactory.addInterface(repositoryInterface);
        proxyFactory.addAdvisors(transactionInterceptor); // @Transactional
        proxyFactory.addAdvisors(queryInterceptor);       // query routing
        return proxyFactory.getProxy();
  
  7. The JDK proxy is registered as a Spring bean
  8. @Autowired fields receive this proxy
```

---

### `JpaRepositoryFactory` Internals

```java
// Simplified view of JpaRepositoryFactory:
public class JpaRepositoryFactory extends RepositoryFactorySupport {
    
    private final EntityManager em;
    
    @Override
    protected Object getTargetRepository(RepositoryInformation info) {
        // Creates the actual implementation:
        return new SimpleJpaRepository<>(
            getEntityInformation(info.getDomainType()),
            em
        );
    }
    
    @Override
    protected Class<?> getRepositoryBaseClass(RepositoryMetadata metadata) {
        // Determines which base class to use
        // Default: SimpleJpaRepository
        // Custom: specified via @EnableJpaRepositories(repositoryBaseClass=...)
        return SimpleJpaRepository.class;
    }
    
    @Override
    public <T, ID> JpaEntityInformation<T, ID> getEntityInformation(Class<T> domainClass) {
        // Builds JpaEntityInformation from entity metadata
        // Reads @Id, @Version, entity name, table name
        return JpaEntityInformationSupport.getEntityInformation(domainClass, em);
    }
}
```

---

### `QueryExecutorMethodInterceptor` — The Query Routing Engine

This is the interceptor inside the proxy that routes method calls to the correct query execution path:

```java
// Simplified view of what happens when you call any repository method:
class QueryExecutorMethodInterceptor implements MethodInterceptor {
    
    private final Map<Method, RepositoryQuery> queries; // built at startup
    
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Method method = invocation.getMethod();
        
        // 1. Is this a method from SimpleJpaRepository? (findById, save, etc.)
        //    → delegate directly to the target (SimpleJpaRepository instance)
        if (hasQueryFor(method)) {
            
            // 2. Get the pre-compiled RepositoryQuery for this method:
            RepositoryQuery query = queries.get(method);
            
            // 3. Execute the query with the invocation arguments:
            return query.execute(invocation.getArguments());
        }
        
        // 4. Fall through to target (SimpleJpaRepository) for base methods
        return invocation.proceed();
    }
}
```

**`RepositoryQuery` hierarchy:**

```
RepositoryQuery (interface)
  ├── PartTreeJpaQuery           ← derived from method name
  │     └── Uses: JpaQueryCreator to build JPQL from PartTree AST
  ├── SimpleJpaQuery             ← @Query with JPQL
  ├── NativeJpaQuery             ← @Query(nativeQuery=true)
  ├── NamedQuery                 ← @NamedQuery on entity
  ├── StoredProcedureJpaQuery    ← @Procedure
  └── ScrollableRepositoryQuery  ← for scrolling result sets (SD 3.1+)
```

---

### `SimpleJpaRepository` — The Actual Implementation

`SimpleJpaRepository<T, ID>` is the class that does all the actual work. Every repository method call ultimately reaches this class (or your custom implementation fragment).

**Key annotations on `SimpleJpaRepository`:**

```java
@Repository                                      // ← makes it a Spring bean
@Transactional(readOnly = true)                  // ← ALL methods are read-only by default
public class SimpleJpaRepository<T, ID>
    implements JpaRepositoryImplementation<T, ID> {

    @Transactional                               // ← write methods override to read-write
    public <S extends T> S save(S entity) { ... }
    
    @Transactional
    public void delete(T entity) { ... }
    
    @Transactional
    public void deleteAll() { ... }
    
    // Read methods — inherit class-level @Transactional(readOnly=true):
    public Optional<T> findById(ID id) { ... }  // readOnly=true
    public List<T> findAll() { ... }             // readOnly=true
    public long count() { ... }                  // readOnly=true
}
```

**Critical implication of `@Transactional(readOnly = true)` at class level:**

ALL repository methods — including your custom `findBy...()` derived queries — inherit `readOnly=true` unless you explicitly override. This means:
- Hibernate sets `FlushMode.MANUAL` for all find/query operations
- Dirty checking is disabled for read operations
- Database hint sent to driver (may optimize for read on some DBs)

If you want a custom query method to be writable:
```java
@Modifying
@Transactional  // override readOnly=true
@Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :cutoff")
int deactivateInactiveUsers(@Param("cutoff") LocalDateTime cutoff);
```

---

### `@EnableJpaRepositories` — The Bootstrap Trigger

```java
@Configuration
@EnableJpaRepositories(
    basePackages = "com.example.repository",     // where to scan for interfaces
    basePackageClasses = UserRepository.class,   // type-safe alternative
    
    entityManagerFactoryRef = "primaryEmf",      // which EMF to use (multi-datasource)
    transactionManagerRef = "primaryTxManager",  // which TX manager (multi-datasource)
    
    repositoryBaseClass = CustomBaseRepository.class, // custom base implementation
    
    repositoryFactoryBeanClass = JpaRepositoryFactoryBean.class, // factory (rarely changed)
    
    considerNestedRepositories = false,          // scan nested interfaces?
    
    namedQueriesLocation = "classpath:jpa-queries.properties", // external named queries
    
    bootstrapMode = BootstrapMode.DEFAULT        // DEFAULT, LAZY, DEFERRED
)
public class JpaConfig { }
```

**`BootstrapMode` — critical for startup performance:**

```java
BootstrapMode.DEFAULT:
  // All repositories initialized eagerly at startup
  // Query methods compiled and validated at startup
  // Startup is slower but all errors detected immediately

BootstrapMode.LAZY:
  // Repository beans created lazily (on first use)
  // Faster startup — but errors only detected at first use
  // Useful for: integration tests, fast dev restart

BootstrapMode.DEFERRED:
  // Repositories initialized when ApplicationContext is fully started
  // Can use Spring application events to delay initialization
  // Useful for: complex startup ordering requirements
```

**Spring Boot auto-configuration:**

In Spring Boot, `@EnableJpaRepositories` is applied automatically by `JpaRepositoriesAutoConfiguration`. You only need explicit `@EnableJpaRepositories` for:
- Custom `repositoryBaseClass`
- Multi-datasource configurations
- Non-standard package locations (outside main class package)

---

### Repository Scanning — How Spring Data Discovers Interfaces

Spring Data uses `RepositoryComponentProvider` to scan for repository interfaces:

```
Scan targets:
  1. Interfaces (not concrete classes)
  2. Extending Repository<T, ID> directly or transitively
  3. NOT annotated with @NoRepositoryBean
  
Exclusion via @NoRepositoryBean:
  @NoRepositoryBean
  public interface BaseCustomRepository<T, ID> extends JpaRepository<T, ID> {
      // This interface is NOT instantiated — it's a base interface
      void customMethod();
  }
  
  public interface UserRepository extends BaseCustomRepository<User, Long> {
      // THIS IS instantiated — concrete repository
  }
```

**`@NoRepositoryBean`** — marks an intermediate interface that should NOT be instantiated:
```java
@NoRepositoryBean
public interface SoftDeleteRepository<T, ID> extends JpaRepository<T, ID> {
    // Adds soft-delete behavior to all repositories that extend this
    @Query("UPDATE #{#entityName} e SET e.deleted=true WHERE e.id=:id")
    @Modifying
    void softDeleteById(@Param("id") ID id);
}

// Concrete repository — WILL be instantiated:
public interface ProductRepository extends SoftDeleteRepository<Product, Long> { }
```

---

### Custom Base Repository — Replacing `SimpleJpaRepository`

You can replace `SimpleJpaRepository` with your own base implementation to add custom behavior to ALL repositories:

```java
// Step 1: Define the custom base interface:
@NoRepositoryBean
public interface BaseRepository<T, ID> extends JpaRepository<T, ID> {
    void detach(T entity);
    Optional<T> findByIdReadOnly(ID id);
}

// Step 2: Implement it (extending SimpleJpaRepository):
public class BaseRepositoryImpl<T, ID> 
    extends SimpleJpaRepository<T, ID>
    implements BaseRepository<T, ID> {
    
    private final EntityManager em;
    
    public BaseRepositoryImpl(JpaEntityInformation<T, ID> entityInfo, 
                               EntityManager em) {
        super(entityInfo, em);
        this.em = em;
    }
    
    @Override
    public void detach(T entity) {
        em.detach(entity);
    }
    
    @Override
    @Transactional(readOnly = true)
    public Optional<T> findByIdReadOnly(ID id) {
        T entity = em.find(getDomainClass(), id);
        if (entity != null) em.detach(entity); // return detached read-only copy
        return Optional.ofNullable(entity);
    }
}

// Step 3: Configure Spring Data to use your base class:
@EnableJpaRepositories(repositoryBaseClass = BaseRepositoryImpl.class)
@Configuration
public class JpaConfig { }

// Step 4: Use in concrete repositories:
public interface ProductRepository extends BaseRepository<Product, Long> {
    // Gets detach() and findByIdReadOnly() for free
    List<Product> findByCategory(String category); // still works
}
```

---

### Repository Fragment Composition

Spring Data supports composing multiple behavior fragments into one repository — the most powerful customization mechanism:

```java
// Fragment 1: custom behavior interface
public interface ProductSearchFragment {
    List<Product> searchByKeyword(String keyword);
    List<Product> findByPriceRange(BigDecimal min, BigDecimal max);
}

// Fragment 1: implementation
public class ProductSearchFragmentImpl implements ProductSearchFragment {
    
    @PersistenceContext
    private EntityManager em;
    
    @Override
    public List<Product> searchByKeyword(String keyword) {
        return em.createQuery(
            "SELECT p FROM Product p WHERE p.name LIKE :kw OR p.description LIKE :kw",
            Product.class)
            .setParameter("kw", "%" + keyword + "%")
            .getResultList();
    }
    
    @Override
    public List<Product> findByPriceRange(BigDecimal min, BigDecimal max) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Product> cq = cb.createQuery(Product.class);
        Root<Product> root = cq.from(Product.class);
        cq.where(cb.between(root.get("price"), min, max));
        return em.createQuery(cq).getResultList();
    }
}

// Fragment 2: another behavior
public interface AuditFragment<T> {
    List<T> findModifiedSince(LocalDateTime since);
}

public class AuditFragmentImpl<T> implements AuditFragment<T> {
    // implementation...
}

// Final repository: composes fragments + JpaRepository:
public interface ProductRepository 
    extends JpaRepository<Product, Long>,
            ProductSearchFragment,       // fragment 1
            AuditFragment<Product> {     // fragment 2
    
    // Derived query:
    List<Product> findByCategory(String category);
}
```

**Fragment resolution rules:**

1. Spring Data finds `ProductSearchFragmentImpl` by convention: fragment interface name + `Impl` suffix
2. If multiple fragments implement the same method: **last fragment wins** (ordering matters)
3. Fragment implementations can inject `EntityManager`, other beans, etc.
4. `@Transactional` on fragment methods works via the proxy

**Custom Impl suffix configuration:**

```java
@EnableJpaRepositories(repositoryImplementationPostfix = "Impl") // default
// Can change to "Repository", "Dao", etc. if needed
```

---

### The Full Proxy Call Stack — Tracing a Single Method Call

```
userRepository.findByEmail("alice@example.com")

Call Stack:
  1. JDK Proxy.invoke()
     └── Spring AOP MethodInvocation
         └── TransactionInterceptor.invoke()        ← @Transactional handling
             ├── Checks: is there an active transaction?
             ├── If NO and method is @Transactional: begin transaction
             └── Proceeds to:
             └── QueryExecutorMethodInterceptor.invoke()
                 ├── Looks up: PartTreeJpaQuery for "findByEmail"
                 └── PartTreeJpaQuery.execute(["alice@example.com"])
                     ├── Gets current EntityManager (thread-local)
                     ├── Builds: TypedQuery<User>
                     │   └── JPQL: "SELECT u FROM User u WHERE u.email = ?1"
                     ├── Sets parameter: ?1 = "alice@example.com"
                     ├── Executes: TypedQuery.getResultList()
                     │   └── Hibernate compiles JPQL → SQL
                     │       └── SQL: SELECT * FROM users WHERE email=?
                     │           └── JDBC PreparedStatement.executeQuery()
                     │               └── ResultSet → Entity hydration
                     └── Returns: Optional<User>
             └── TransactionInterceptor: commit if transaction was started here
```

---

## 2️⃣ Code Examples

### Example 1 — Repository Hierarchy Choices and Consequences

```java
// Choice 1: Full JpaRepository — everything available:
public interface OrderRepository extends JpaRepository<Order, Long> {
    // Gets: save, findById, findAll, delete, flush, getReferenceById,
    //       deleteAllInBatch, saveAllAndFlush, count, existsById, etc.
}

// Choice 2: CrudRepository — basic operations only:
public interface ReadOnlyProductRepository extends CrudRepository<Product, Long> {
    // Gets: save, findById, findAll (returns Iterable), delete, count
    // Missing: flush, getReferenceById, deleteAllInBatch, List<T> returns
}

// Choice 3: Minimal Repository — explicit contract:
public interface InvoiceRepository extends Repository<Invoice, Long> {
    // ONLY these methods are exposed to callers:
    Invoice save(Invoice invoice);
    Optional<Invoice> findById(Long id);
    List<Invoice> findByCustomerId(Long customerId);
    // No delete, no deleteAll — consumers cannot delete invoices
    // Security through interface design
}

// Choice 4: PagingAndSortingRepository (SD 3.x — does NOT extend CrudRepository):
public interface ReportRepository 
    extends PagingAndSortingRepository<Report, Long>,
            CrudRepository<Report, Long> { // must explicitly add CRUD
    // Gets pagination + CRUD
}
```

---

### Example 2 — `save()` `isNew()` Behavior Demonstration

```java
// Entity with @Version:
@Entity
public class Document {
    @Id @GeneratedValue Long id;
    String title;
    @Version Long version; // version=null → isNew=true; version non-null → isNew=false
}

@Transactional
public void saveVersionDemo() {
    // New entity — id=null, version=null:
    Document doc = new Document("Design Doc");
    documentRepo.save(doc);
    // isNew: version==null → true → em.persist()
    // SQL: INSERT INTO documents (title, version) VALUES ('Design Doc', 0)
    // After persist: id=1, version=0
    
    // Existing entity — id=1, version=0:
    doc.setTitle("Updated Design Doc");
    documentRepo.save(doc);
    // isNew: version!=null (version=0) → false → em.merge()
    // SQL: SELECT * FROM documents WHERE id=1 (merge does SELECT first)
    //      UPDATE documents SET title=?, version=1 WHERE id=1 AND version=0
}

// Entity with natural key — Persistable:
@Entity
public class Country implements Persistable<String> {
    @Id
    private String isoCode; // e.g., "US", "GB"
    private String name;
    
    @Transient
    private boolean isNew = true; // new by default
    
    @Override public String getId() { return isoCode; }
    @Override public boolean isNew() { return isNew; }
    
    @PostPersist @PostLoad
    public void markNotNew() { this.isNew = false; }
}

@Transactional
public void naturalKeyDemo() {
    Country us = new Country("US", "United States");
    countryRepo.save(us);
    // isNew: Persistable.isNew()=true → em.persist()
    // SQL: INSERT INTO countries (iso_code, name) VALUES ('US', 'United States')
    // NO SELECT before insert (unlike merge)
}
```

---

### Example 3 — `deleteById()` SELECT Trap and Solution

```java
// STANDARD deleteById — fires SELECT first:
@Transactional
public void deleteOrderStandard(Long id) {
    orderRepo.deleteById(id);
    // SQL 1: SELECT * FROM orders WHERE id=?  ← loads entity first!
    // SQL 2: DELETE FROM orders WHERE id=?
    // If id not found: throws EmptyResultDataAccessException
}

// CUSTOM query — no SELECT:
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    @Modifying
    @Transactional
    @Query("DELETE FROM Order o WHERE o.id = :id")
    int deleteByIdDirect(@Param("id") Long id);
    // SQL: DELETE FROM orders WHERE id=?
    // Returns: number of deleted rows (0 if not found, no exception)
    // Caveat: cascade and orphanRemoval NOT triggered
    //         lifecycle callbacks (@PreRemove) NOT called
}

// deleteAllInBatch() — most efficient for bulk:
@Transactional
public void deleteAllOrders() {
    orderRepo.deleteAllInBatch();
    // SQL: DELETE FROM orders
    // No entity loading, no cascade, no callbacks
    // Use only when you don't need cascade behavior
}

// deleteAllByIdInBatch — batch delete by IDs:
@Transactional
public void deleteOrdersByIds(List<Long> ids) {
    orderRepo.deleteAllByIdInBatch(ids);
    // SQL: DELETE FROM orders WHERE id IN (?,?,?...)
    // No entity loading
}
```

---

### Example 4 — Full Repository Fragment Composition

```java
// Fragment for full-text search behavior:
public interface FullTextSearchable<T> {
    List<T> fullTextSearch(String query);
}

public class FullTextSearchableImpl<T> implements FullTextSearchable<T> {
    
    @PersistenceContext
    private EntityManager em;
    
    // Uses Hibernate Search or native query approach
    @Override
    @SuppressWarnings("unchecked")
    public List<T> fullTextSearch(String query) {
        // Native full-text search (PostgreSQL example):
        return em.createNativeQuery(
                "SELECT * FROM products WHERE to_tsvector(name || ' ' || description) " +
                "@@ plainto_tsquery(?)", Product.class)
            .setParameter(1, query)
            .getResultList();
    }
}

// Fragment for soft-delete:
@NoRepositoryBean
public interface SoftDeletable<T, ID> extends JpaRepository<T, ID> {
    
    @Modifying
    @Query("UPDATE #{#entityName} e SET e.deletedAt = CURRENT_TIMESTAMP WHERE e.id = :id")
    void softDeleteById(@Param("id") ID id);
    
    @Query("SELECT e FROM #{#entityName} e WHERE e.deletedAt IS NULL")
    List<T> findAllActive();
}

// No separate Impl needed for interface default queries using @Query

// Composed repository:
public interface ProductRepository 
    extends SoftDeletable<Product, Long>,
            FullTextSearchable<Product> {
    
    // Standard derived queries:
    List<Product> findByCategory(String category);
    Optional<Product> findBySku(String sku);
    
    // @Query:
    @Query("SELECT p FROM Product p WHERE p.price BETWEEN :min AND :max")
    List<Product> findByPriceRange(@Param("min") BigDecimal min, 
                                   @Param("max") BigDecimal max);
}
```

---

### Example 5 — `@EnableJpaRepositories` Multi-Datasource Setup

```java
// Primary datasource repositories:
@Configuration
@EnableJpaRepositories(
    basePackages = "com.example.primary.repository",
    entityManagerFactoryRef = "primaryEntityManagerFactory",
    transactionManagerRef = "primaryTransactionManager"
)
public class PrimaryJpaConfig {
    
    @Primary
    @Bean
    public LocalContainerEntityManagerFactoryBean primaryEntityManagerFactory(
            @Qualifier("primaryDataSource") DataSource dataSource) {
        LocalContainerEntityManagerFactoryBean factory = 
            new LocalContainerEntityManagerFactoryBean();
        factory.setDataSource(dataSource);
        factory.setPackagesToScan("com.example.primary.domain");
        factory.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
        return factory;
    }
    
    @Primary
    @Bean
    public PlatformTransactionManager primaryTransactionManager(
            @Qualifier("primaryEntityManagerFactory") EntityManagerFactory emf) {
        return new JpaTransactionManager(emf);
    }
}

// Reporting datasource repositories (read replica):
@Configuration
@EnableJpaRepositories(
    basePackages = "com.example.reporting.repository",
    entityManagerFactoryRef = "reportingEntityManagerFactory",
    transactionManagerRef = "reportingTransactionManager"
)
public class ReportingJpaConfig {
    // Similar configuration for reporting DB
}
```

---

### Example 6 — Verifying Proxy Behavior at Runtime

```java
@SpringBootTest
class RepositoryProxyTest {
    
    @Autowired
    UserRepository userRepo;
    
    @Test
    void verifyRepositoryIsProxy() {
        // The injected bean is a JDK proxy:
        System.out.println(userRepo.getClass().getName());
        // com.sun.proxy.$Proxy89 (JDK proxy)
        
        // The proxy implements the repository interface:
        System.out.println(userRepo instanceof UserRepository); // true
        System.out.println(userRepo instanceof JpaRepository);  // true
        
        // Underlying target is SimpleJpaRepository:
        // (accessed via AopUtils or reflection for debugging)
        Object target = AopUtils.getTargetClass(userRepo);
        System.out.println(target); // class SimpleJpaRepository (or custom)
        
        // The proxy is a Spring bean:
        System.out.println(AopUtils.isAopProxy(userRepo)); // true
        System.out.println(AopUtils.isJdkDynamicProxy(userRepo)); // true
    }
    
    @Test
    void verifyTransactionalBehavior() {
        // SimpleJpaRepository.findAll() has readOnly=true at class level
        // Verify no dirty writes happen in read-only context:
        
        List<User> users = userRepo.findAll();
        users.forEach(u -> u.setName("HACKED")); // modify entities
        
        // Because findAll() is @Transactional(readOnly=true):
        // FlushMode = MANUAL → no dirty check → no UPDATE
        // Changes are discarded when transaction closes
        
        // Re-load to verify no change persisted:
        List<User> reloaded = userRepo.findAll();
        reloaded.forEach(u -> 
            assertThat(u.getName()).doesNotContain("HACKED"));
    }
}
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ**
What is the runtime type of a Spring Data JPA repository bean injected via `@Autowired`?

A) `SimpleJpaRepository`  
B) The concrete class generated by Spring Data at startup  
C) A JDK dynamic proxy implementing the repository interface  
D) A CGLIB subclass of `SimpleJpaRepository`  

**Answer: C** — Repository beans are JDK dynamic proxies. The proxy implements the repository interface(s) and delegates to `SimpleJpaRepository` as the underlying target.

---

**Q2 — Select All That Apply**
Which statements about `SimpleJpaRepository` are TRUE? (Select all)

A) It is annotated with `@Transactional(readOnly = true)` at the class level  
B) All mutating methods (`save`, `delete`) override the class-level readOnly with `@Transactional`  
C) It is annotated with `@Repository`  
D) It directly implements the user-defined repository interface  
E) Custom `findBy...` derived query methods on user interfaces also get `readOnly=true` by default  

**Answer: A, B, C, E**
- D is false: `SimpleJpaRepository` implements `JpaRepositoryImplementation`. It does not implement user-defined interfaces. The proxy routes user-defined method calls through `QueryExecutorMethodInterceptor`.

---

**Q3 — Code Behavior**
```java
@Entity
public class Product {
    @Id
    private String code; // natural key, no @GeneratedValue
    private String name;
}

Product p = new Product();
p.setCode("SKU-001");
p.setName("Widget");
productRepo.save(p);
```

What SQL is executed?

A) `INSERT INTO product (code, name) VALUES ('SKU-001', 'Widget')`  
B) `SELECT * FROM product WHERE code='SKU-001'` then `INSERT` if not found  
C) `SELECT * FROM product WHERE code='SKU-001'` then `INSERT` if not found or `UPDATE` if found  
D) `Exception` — `@Id` without `@GeneratedValue` requires `Persistable`  

**Answer: C** — `code` is non-null → `isNew()` returns false → `em.merge()` is called. `merge()` first issues a SELECT to check if the entity exists. If found → UPDATE. If not found → INSERT. This is one SELECT plus either INSERT or UPDATE.

---

**Q4 — MCQ**
What does `userRepo.getReferenceById(1L)` do?

A) Executes `SELECT * FROM users WHERE id=1` and returns the User  
B) Returns a Hibernate proxy for User with id=1 without executing SQL  
C) Returns `Optional.empty()` if User with id=1 doesn't exist  
D) Throws `EntityNotFoundException` if User with id=1 doesn't exist  

**Answer: B** — `getReferenceById()` calls `em.getReference()` which returns a proxy immediately without a SELECT. The proxy is initialized lazily on first non-ID method access. `EntityNotFoundException` is thrown only when the proxy is initialized and no DB row is found — not at proxy creation time.

---

**Q5 — Select All That Apply**
Which behaviors are TRUE for `deleteById(Long id)` in `SimpleJpaRepository`? (Select all)

A) Executes a `SELECT` before the `DELETE`  
B) Executes a direct `DELETE FROM table WHERE id=?` without SELECT  
C) Throws `EmptyResultDataAccessException` if no entity with that ID exists  
D) Triggers `@PreRemove` lifecycle callbacks  
E) Triggers `CascadeType.REMOVE` cascade operations  

**Answer: A, C, D, E**
- A: `deleteById()` calls `findById()` first to load the entity
- B: False — SELECT does happen
- C: If `findById()` returns empty, throws `EmptyResultDataAccessException`
- D: Entity is managed before `em.remove()` → `@PreRemove` fires
- E: `em.remove()` is called on the managed entity → cascade triggers

---

**Q6 — `@NoRepositoryBean` Purpose**
```java
@NoRepositoryBean
public interface TimestampedRepository<T, ID> extends JpaRepository<T, ID> {
    List<T> findByCreatedAtAfter(LocalDateTime dateTime);
}

public interface ArticleRepository extends TimestampedRepository<Article, Long> { }
public interface VideoRepository extends TimestampedRepository<Video, Long> { }
```

What does `@NoRepositoryBean` do here?

A) Prevents `TimestampedRepository` from being instantiated as a Spring bean  
B) Makes `TimestampedRepository` abstract so it can't be extended  
C) Disables query method generation for `TimestampedRepository`  
D) Marks `TimestampedRepository` for lazy initialization  

**Answer: A** — `@NoRepositoryBean` tells Spring Data: "do not create a bean for this interface." Without it, Spring Data would try to instantiate `TimestampedRepository<T, ID>` as a bean — which would fail because `T` and `ID` are unresolved generics. `ArticleRepository` and `VideoRepository` ARE instantiated because they have concrete type parameters.

---

**Q7 — Startup Behavior**
```java
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByNonExistentField(String value);
}
```

When does this error surface?

A) At compile time — Spring Data validates fields at compile time  
B) At application startup — Spring Data validates derived query fields against the entity metamodel  
C) At first method invocation — Spring Data validates lazily  
D) Never — Spring Data silently ignores invalid derived query methods  

**Answer: B** — Spring Data compiles and validates all derived query methods at startup (during `JpaRepositoryFactory` initialization). `findByNonExistentField` is parsed into a `PartTree`, which tries to resolve `nonExistentField` against the `Order` entity metamodel. Since it doesn't exist → `PropertyReferenceException` at startup. This is a safety feature — errors are caught at startup, not in production.

---

**Q8 — Fragment Naming Convention**
```java
public interface ProductRepository 
    extends JpaRepository<Product, Long>, CustomProductOperations {
}

public interface CustomProductOperations {
    List<Product> findExpensive();
}
```

What class name must the implementation of `CustomProductOperations` have by default?

A) `CustomProductOperationsRepository`  
B) `ProductRepositoryImpl`  
C) `CustomProductOperationsImpl`  
D) `ProductRepositoryCustom`  

**Answer: C** — Spring Data looks for an implementation class named `{FragmentInterfaceName}Impl`. For fragment interface `CustomProductOperations`, the implementation must be `CustomProductOperationsImpl`. Alternatively (legacy): the repository interface name + `Impl` (`ProductRepositoryImpl`) also works but applies to the entire repository, not a specific fragment.

---

**Q9 — `deleteAll()` vs `deleteAllInBatch()`**
```java
// 10,000 products in database
productRepo.deleteAll();      // Option A
productRepo.deleteAllInBatch(); // Option B
```

Which statement correctly describes the difference?

A) Both execute a single `DELETE FROM products` SQL statement  
B) Option A loads all 10,000 entities then deletes one by one; Option B executes `DELETE FROM products`  
C) Option A executes `DELETE FROM products`; Option B loads then deletes one by one  
D) Both load all entities first, then batch-delete them  

**Answer: B** — `deleteAll()` calls `findAll()` first (loads all 10,000 entities into memory), then calls `em.remove()` per entity (10,000 DELETEs). `deleteAllInBatch()` executes a single `DELETE FROM products` SQL without loading anything. `deleteAllInBatch()` skips cascade, orphanRemoval, and lifecycle callbacks.

---

## 4️⃣ Trick Analysis

**The injected bean is a proxy, not `SimpleJpaRepository` (Q1)**:
Calling `userRepo.getClass()` never returns `SimpleJpaRepository`. It returns a `com.sun.proxy.$ProxyXX` class. `SimpleJpaRepository` is the **target** behind the proxy — accessible via `AopUtils.getTargetClass()` or `AopProxyUtils.ultimateTargetClass()`. This matters when using reflection, checking types, or debugging.

**`readOnly=true` inherited by derived queries (Q2, Q6 in Examples)**:
Developers often add `@Modifying @Query("UPDATE...")` but forget `@Transactional`. The class-level `readOnly=true` causes the UPDATE to fail with `TransactionRequiredException` or silently not execute (FlushMode.MANUAL). Always add `@Transactional` on `@Modifying` methods. Also note: custom `findBy...` methods get `readOnly=true` for free — this is actually desirable (disables dirty checking for read queries).

**Natural key `save()` → `merge()` → extra SELECT (Q3)**:
The most common surprise for developers working with natural keys (String IDs, UUIDs set by application, business codes). Every `save()` on a non-null-ID entity goes through `merge()`, which does a SELECT first. For bulk inserts of always-new entities with natural keys, this doubles the SQL count. Fix: `Persistable<ID>` implementation.

**`getReferenceById()` deferred exception (Q4)**:
`getReferenceById()` (formerly `getOne()`) never throws `EntityNotFoundException` immediately. The exception is thrown when the proxy is initialized — which may be much later, potentially after the transaction has ended, causing confusing exceptions. Always use `getReferenceById()` only when you're certain the ID exists (e.g., when setting an FK reference from a trusted source).

**`@NoRepositoryBean` on intermediate interfaces (Q6)**:
Without `@NoRepositoryBean`, Spring Data tries to instantiate every interface extending `Repository`. For intermediate interfaces with unresolved generics (`T`, `ID`), Spring Data cannot determine the entity type → fails. `@NoRepositoryBean` is the escape hatch that says "this is an abstract definition, not a concrete repository."

**`deleteById()` extra SELECT (Q5)**:
Most developers assume `deleteById()` executes a direct `DELETE WHERE id=?`. It doesn't. The SELECT is needed so Hibernate can: (1) load the entity into the PC, (2) trigger `@PreRemove` callbacks, (3) process `CascadeType.REMOVE`, (4) handle `orphanRemoval`. If you need pure efficiency without these features, use `@Modifying @Query("DELETE FROM ...")`.

---

## 5️⃣ Summary Sheet

### Repository Hierarchy Reference

```
Repository<T,ID>                    ← marker, empty
  └── CrudRepository                ← save, findById, findAll(Iterable),
  │                                    existsById, count, delete, deleteAll
  └── ListCrudRepository            ← same as CrudRepository but returns List (SD 3.0+)
  └── PagingAndSortingRepository    ← findAll(Sort), findAll(Pageable) [SD 3: no CRUD!]
  └── JpaRepository                 ← extends ListCrudRepository + ListPagingAndSorting
                                       flush, saveAndFlush, deleteAllInBatch,
                                       getReferenceById, QueryByExampleExecutor
```

### `save()` Decision Logic

```
save(entity):
  entity implements Persistable → use entity.isNew()
  @Version field exists         → version==null → persist(); version!=null → merge()
  @Id field                     → null → persist(); non-null → merge()
  primitive @Id                 → 0 → persist(); non-zero → merge()

persist() → INSERT (no SELECT before)
merge()   → SELECT first, then INSERT (if not found) or UPDATE (if found)
```

### Startup Sequence Summary

```
@EnableJpaRepositories
  → JpaRepositoriesRegistrar
    → JpaRepositoryFactoryBean per interface
      → JpaRepositoryFactory
        → Validates entity type and @Id
        → Compiles all query methods (startup validation!)
        → Creates SimpleJpaRepository as target
        → Wraps in JDK dynamic proxy with:
            TransactionInterceptor
            QueryExecutorMethodInterceptor
        → Registers proxy as Spring bean
```

### `deleteAll` Methods Comparison

| Method | SQL | Entities Loaded? | Cascade? | Callbacks? |
|---|---|---|---|---|
| `deleteAll()` | N individual DELETEs | ✅ Yes | ✅ Yes | ✅ Yes |
| `deleteAllInBatch()` | `DELETE FROM table` | ❌ No | ❌ No | ❌ No |
| `deleteAllByIdInBatch(ids)` | `DELETE WHERE id IN(...)` | ❌ No | ❌ No | ❌ No |
| `@Modifying @Query DELETE` | Custom SQL | ❌ No | ❌ No | ❌ No |

### Key Rules

```
1. Repository bean = JDK proxy (com.sun.proxy.$ProxyXX)
2. SimpleJpaRepository = the proxy target (actual implementation)
3. ALL repository methods inherit @Transactional(readOnly=true) from SimpleJpaRepository
4. @Modifying write methods MUST add @Transactional (override readOnly)
5. deleteById() → findById() first → EmptyResultDataAccessException if not found
6. deleteAllInBatch() → DELETE FROM table (no cascade, no callbacks)
7. getReferenceById() → proxy, no SQL until first non-ID access
8. save(entity with non-null ID) → merge() → SELECT + INSERT/UPDATE
9. @NoRepositoryBean → interface is NOT instantiated by Spring Data
10. Fragment Impl name = {FragmentInterface}Impl (not repository name by default in modern SD)
11. BootstrapMode.LAZY → faster startup, errors detected at first use
12. PropertyReferenceException at startup → invalid field in derived query method
```

---
