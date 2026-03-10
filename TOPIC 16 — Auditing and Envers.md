# 🏛️ TOPIC 16 — Auditing and Envers

---

## 1️⃣ Conceptual Explanation

### Why Auditing Exists — The Compliance and Debugging Need

Enterprise applications universally need to answer: **who created this record, when, who last changed it, and what did it look like at any point in history?** These are compliance requirements (GDPR, SOX, HIPAA), debugging aids, and business audit trails.

Two distinct problems:

```
Problem 1: Basic field auditing
  → Who created this entity? When?
  → Who last modified it? When?
  → Solution: Spring Data JPA Auditing
               (@CreatedDate, @LastModifiedDate, @CreatedBy, @LastModifiedBy)

Problem 2: Full history / revision tracking
  → What did this entity look like at version N?
  → What changed between version 3 and version 7?
  → Who changed field X and when?
  → Solution: Hibernate Envers (@Audited)
```

---

### Spring Data JPA Auditing — Architecture

Spring Data JPA auditing hooks into the JPA entity lifecycle via `AuditingEntityListener`. This listener responds to `@PrePersist` and `@PreUpdate` JPA lifecycle callbacks to populate audit fields automatically.

```
Entity persist/update flow with auditing:

em.persist(entity)  OR  dirty-checking triggers UPDATE
         │
         ▼
┌─────────────────────────────────────┐
│  AuditingEntityListener             │
│  (@PrePersist / @PreUpdate)         │
│                                     │
│  On @PrePersist:                    │
│    Set @CreatedDate  (if null)      │
│    Set @CreatedBy    (if null)      │
│    Set @LastModifiedDate            │
│    Set @LastModifiedBy              │
│                                     │
│  On @PreUpdate:                     │
│    Set @LastModifiedDate            │
│    Set @LastModifiedBy              │
│    (Leave @CreatedDate alone)       │
└─────────────────────────────────────┘
         │
         ▼
    SQL INSERT or UPDATE executed
```

---

### Enabling Auditing — `@EnableJpaAuditing`

```java
// Configuration class:
@Configuration
@EnableJpaAuditing
public class JpaConfig {
    // Minimal — enables auditing with default settings
}

// With AuditorAware bean:
@Configuration
@EnableJpaAuditing(auditorAwareRef = "auditorProvider")
public class JpaConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        // Return current user from Spring Security:
        return () -> Optional.ofNullable(SecurityContextHolder.getContext())
            .map(SecurityContext::getAuthentication)
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName);
    }
}

// With custom date-time provider:
@Configuration
@EnableJpaAuditing(
    auditorAwareRef = "auditorProvider",
    dateTimeProviderRef = "dateTimeProvider"
)
public class JpaConfig {

    @Bean
    public DateTimeProvider dateTimeProvider() {
        return () -> Optional.of(LocalDateTime.now(ZoneOffset.UTC));
        // Always use UTC for audit timestamps
    }
}
```

---

### Auditing Annotations — Complete Reference

```java
@Entity
@EntityListeners(AuditingEntityListener.class)  // REQUIRED on entity
public class Order {

    @Id @GeneratedValue Long id;

    // Populated on INSERT only (never updated):
    @CreatedDate
    @Column(updatable = false)   // DB-level protection
    LocalDateTime createdAt;

    // Populated on INSERT and every UPDATE:
    @LastModifiedDate
    LocalDateTime lastModifiedAt;

    // Populated on INSERT only — who created:
    @CreatedBy
    @Column(updatable = false)
    String createdBy;

    // Populated on every UPDATE — who last modified:
    @LastModifiedBy
    String lastModifiedBy;

    // All other fields...
    String status;
    BigDecimal total;
}
```

**Supported date/time types for `@CreatedDate` / `@LastModifiedDate`:**

```java
// All supported:
LocalDate           createdAt;
LocalDateTime       createdAt;
Instant             createdAt;
ZonedDateTime       createdAt;
OffsetDateTime      createdAt;
java.util.Date      createdAt;  // legacy
java.util.Calendar  createdAt;  // legacy
long / Long         createdAt;  // epoch millis
Joda DateTime       createdAt;  // if Joda present

// RECOMMENDED: LocalDateTime or Instant
// Instant: timezone-neutral, best for distributed systems
// LocalDateTime: human-readable, requires consistent JVM timezone
```

**Supported types for `@CreatedBy` / `@LastModifiedBy`:**

```java
// Any type returned by AuditorAware<T>:
String  createdBy;   // username string (most common)
Long    createdBy;   // user ID
UUID    createdBy;   // user UUID
User    createdBy;   // @ManyToOne to User entity
```

---

### `@EntityListeners(AuditingEntityListener.class)` — The Wiring

`@EntityListeners` registers JPA lifecycle callbacks on the entity. Without this annotation, the `AuditingEntityListener` is never invoked and audit fields remain null.

```java
// Without @EntityListeners — BROKEN:
@Entity
public class Product {
    @CreatedDate LocalDateTime createdAt;  // NEVER populated
    @LastModifiedDate LocalDateTime updatedAt; // NEVER populated
}

// With @EntityListeners — CORRECT:
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Product {
    @CreatedDate LocalDateTime createdAt;  // auto-populated on INSERT
    @LastModifiedDate LocalDateTime updatedAt; // auto-populated on INSERT + UPDATE
}
```

**Multiple listeners:**

```java
@EntityListeners({
    AuditingEntityListener.class,
    CustomValidationListener.class
})
```

---

### `@MappedSuperclass` for Shared Audit Fields — Best Practice

Rather than repeating audit annotations on every entity, extract to a common base:

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditableEntity {

    @CreatedDate
    @Column(updatable = false, nullable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private LocalDateTime lastModifiedAt;

    @CreatedBy
    @Column(updatable = false, length = 100)
    private String createdBy;

    @LastModifiedBy
    @Column(length = 100)
    private String lastModifiedBy;

    // Getters only — no setters for audit fields:
    public LocalDateTime getCreatedAt()     { return createdAt; }
    public LocalDateTime getLastModifiedAt(){ return lastModifiedAt; }
    public String getCreatedBy()            { return createdBy; }
    public String getLastModifiedBy()       { return lastModifiedBy; }
}

// All entities simply extend:
@Entity
public class Order extends AuditableEntity {
    @Id @GeneratedValue Long id;
    String status;
    BigDecimal total;
    // createdAt, lastModifiedAt, createdBy, lastModifiedBy inherited
}

@Entity
public class Product extends AuditableEntity {
    @Id @GeneratedValue Long id;
    String name;
    BigDecimal price;
    // audit fields inherited
}
```

---

### `AuditorAware<T>` — Who Is the Current User?

`AuditorAware<T>` is the interface Spring Data calls to get the current auditor (usually the logged-in user). Its return type `T` must match the `@CreatedBy` / `@LastModifiedBy` field type.

```java
// String-based auditor (username):
@Component
public class SecurityAuditorAware implements AuditorAware<String> {

    @Override
    public Optional<String> getCurrentAuditor() {
        Authentication auth = SecurityContextHolder.getContext()
                                                   .getAuthentication();
        if (auth == null || !auth.isAuthenticated()
                || auth.getPrincipal().equals("anonymousUser")) {
            return Optional.of("SYSTEM"); // fallback for system operations
        }
        return Optional.of(auth.getName());
    }
}

// Long-based auditor (user ID):
@Component
public class UserIdAuditorAware implements AuditorAware<Long> {

    @Override
    public Optional<Long> getCurrentAuditor() {
        return Optional.ofNullable(SecurityContextHolder.getContext()
                .getAuthentication())
                .filter(Authentication::isAuthenticated)
                .map(auth -> ((UserPrincipal) auth.getPrincipal()).getId());
    }
}

// Entity-based auditor (@ManyToOne User):
@Component
public class UserEntityAuditorAware implements AuditorAware<User> {

    @Autowired UserRepository userRepo;

    @Override
    public Optional<User> getCurrentAuditor() {
        return Optional.ofNullable(SecurityContextHolder.getContext()
                .getAuthentication())
                .filter(Authentication::isAuthenticated)
                .map(auth -> auth.getName())
                .flatMap(userRepo::findByUsername);
    }
}
```

**`AuditorAware` returning `Optional.empty()`:**

```java
// If AuditorAware returns Optional.empty():
// → @CreatedBy and @LastModifiedBy fields are NOT populated (remain null/unchanged)
// → No exception is thrown
// → Useful for batch jobs or system processes with no authenticated user
```

---

### `@CreatedDate` vs `@LastModifiedDate` — Behaviour Differences

```
@CreatedDate:
  Populated: On @PrePersist ONLY (first INSERT)
  Never populated again: @PreUpdate callbacks leave @CreatedDate alone
  Verification: AuditingEntityListener checks if field is null before setting
  → If already non-null: skipped (not overwritten)
  Column protection: Add @Column(updatable = false) for DB-level enforcement

@LastModifiedDate:
  Populated: On BOTH @PrePersist (INSERT) and @PreUpdate (UPDATE)
  Always set to current time on every modification
  → Each UPDATE gets a fresh timestamp

Combined:
  createdAt = INSERT time (never changes after that)
  lastModifiedAt = time of last change (updates on every modification)
```

---

### Auditing with `@Version` — Interaction

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Document {
    @Id @GeneratedValue Long id;
    String content;

    @Version Integer version;       // optimistic lock counter
    @LastModifiedDate LocalDateTime lastModifiedAt; // audit timestamp

    // version and lastModifiedAt update together on every modification:
    // UPDATE documents SET content=?, version=N+1, last_modified_at=?
    //                  WHERE id=? AND version=N
}
```

---

### Hibernate Envers — Full History Tracking

Spring Data JPA auditing tracks WHO and WHEN on the current row. Envers tracks the **full history** — every version of every entity, stored in revision tables.

**What Envers does:**

```
Normal table: products
  id | name       | price | version
   1 | Widget     | 9.99  | 3

With @Audited Envers creates: products_aud (audit table)
  id | REV | REVTYPE | name       | price
   1 | 1   | 0 (ADD) | Widget     | 9.99   ← initial create
   1 | 2   | 1 (MOD) | Widget     | 14.99  ← price updated
   1 | 3   | 1 (MOD) | Super Widget| 14.99 ← name updated
   1 | 4   | 2 (DEL) | null       | null   ← deleted

revinfo table (revision metadata):
  REV | REVTSTMP (timestamp)
   1  | 2024-01-15 10:00:00
   2  | 2024-01-20 14:30:00
   3  | 2024-02-01 09:15:00
   4  | 2024-02-10 16:45:00
```

**REVTYPE values:**

```
0 = ADD    (INSERT)
1 = MOD    (UPDATE)
2 = DEL    (DELETE)
```

---

### Envers Configuration — Dependency and Setup

```xml
<!-- Maven dependency: -->
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-envers</artifactId>
</dependency>
<!-- Spring Boot auto-configures Envers when this is on classpath -->
```

```java
// Entity annotated for auditing:
@Entity
@Audited  // ← all fields audited by Envers
public class Product {
    @Id @GeneratedValue Long id;
    String name;
    BigDecimal price;
    boolean active;
}

// Selectively audit some fields:
@Entity
@Audited
public class Customer {
    @Id @GeneratedValue Long id;

    @Audited          // explicitly audited (redundant if class is @Audited)
    String email;

    @NotAudited       // excluded from audit (e.g., large BLOB fields)
    byte[] profilePhoto;

    String name;      // audited (class-level @Audited applies)
}
```

---

### `@Audited` on Associations

```java
@Entity
@Audited
public class Order {
    @Id @GeneratedValue Long id;
    String status;

    @ManyToOne
    @Audited(targetAuditMode = RelationTargetAuditMode.NOT_AUDITED)
    Customer customer;
    // Customer is NOT @Audited — only the FK value is stored in orders_aud
    // RelationTargetAuditMode.NOT_AUDITED: store FK, don't require Customer audit table

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    @Audited  // OrderItems are also audited
    List<OrderItem> items;
}
```

**`RelationTargetAuditMode`:**

```
AUDITED (default):
  → Related entity must also be @Audited
  → Envers links to the related entity's revision
  → Full history of both sides

NOT_AUDITED:
  → Related entity is NOT @Audited (or you don't want to track it)
  → Only the FK value is stored in the audit table
  → Cannot query historical state of the related entity
```

---

### Custom Revision Entity — Adding User to Revisions

By default, Envers stores only a revision number and timestamp. To track WHO made each change, create a custom revision entity:

```java
// Custom revision entity:
@Entity
@RevisionEntity(UserRevisionListener.class)
@Table(name = "revinfo")
public class UserRevision extends DefaultRevisionEntity {
    // Inherits: int id, long timestamp

    @Column(name = "username")
    private String username;

    @Column(name = "user_id")
    private Long userId;

    // getters and setters
}

// Listener that populates custom fields:
@Component
public class UserRevisionListener
        implements RevisionListener {

    // NOTE: cannot @Autowired here — Envers creates this via reflection
    // Use static accessor or ApplicationContext trick:

    @Override
    public void newRevision(Object revisionEntity) {
        UserRevision revision = (UserRevision) revisionEntity;

        Authentication auth = SecurityContextHolder.getContext()
                                                   .getAuthentication();
        if (auth != null && auth.isAuthenticated()) {
            revision.setUsername(auth.getName());
            if (auth.getPrincipal() instanceof UserPrincipal up) {
                revision.setUserId(up.getId());
            }
        } else {
            revision.setUsername("SYSTEM");
        }
    }
}
```

**Resulting `revinfo` table:**

```
REV | REVTSTMP            | username | user_id
  1 | 2024-01-15 10:00:00 | alice    | 42
  2 | 2024-01-20 14:30:00 | bob      | 17
  3 | 2024-02-01 09:15:00 | SYSTEM   | null
```

---

### `AuditReader` — Querying Historical Data

`AuditReader` is the Envers API for reading historical entity states:

```java
@Service
public class ProductHistoryService {

    @PersistenceContext
    EntityManager em;

    private AuditReader getAuditReader() {
        return AuditReaderFactory.get(em);
    }

    // Find entity at a specific revision:
    @Transactional(readOnly = true)
    public Product findAtRevision(Long productId, int revision) {
        AuditReader reader = getAuditReader();
        return reader.find(Product.class, productId, revision);
        // Returns Product as it was at that revision number
        // Returns null if entity didn't exist at that revision
    }

    // Find entity as it was at a specific date/time:
    @Transactional(readOnly = true)
    public Product findAtDate(Long productId, Date date) {
        AuditReader reader = getAuditReader();
        Number revision = reader.getRevisionNumberForDate(date);
        return reader.find(Product.class, productId, revision);
    }

    // Get all revision numbers for an entity:
    @Transactional(readOnly = true)
    public List<Number> getRevisions(Long productId) {
        AuditReader reader = getAuditReader();
        return reader.getRevisions(Product.class, productId);
        // Returns list of revision numbers [1, 3, 7, 12, ...]
    }

    // Get full history with revision info:
    @Transactional(readOnly = true)
    public List<Object[]> getFullHistory(Long productId) {
        AuditReader reader = getAuditReader();
        return reader.createQuery()
            .forRevisionsOfEntity(Product.class, false, true)
            // false = return Object[] (not just entity)
            // true  = include DELETE revisions
            .add(AuditEntity.id().eq(productId))
            .addOrder(AuditEntity.revisionNumber().asc())
            .getResultList();
        // Each element: Object[] { Product entity, RevisionEntity, RevisionType }
    }

    // Query revisions where a field had a specific value:
    @Transactional(readOnly = true)
    public List<Product> findWhereNameWas(String oldName) {
        AuditReader reader = getAuditReader();
        return reader.createQuery()
            .forRevisionsOfEntity(Product.class, true, false)
            // true = return only entity (not revision metadata)
            .add(AuditEntity.property("name").eq(oldName))
            .getResultList();
    }

    // Get latest revision of all deleted entities:
    @Transactional(readOnly = true)
    public List<Product> findAllDeleted() {
        AuditReader reader = getAuditReader();
        return reader.createQuery()
            .forRevisionsOfEntity(Product.class, true, true)
            .add(AuditEntity.revisionType().eq(RevisionType.DEL))
            .getResultList();
    }
}
```

---

### `AuditQuery` API — Querying History

```java
AuditReader reader = AuditReaderFactory.get(em);

// Query at a specific revision:
reader.createQuery()
    .forEntitiesAtRevision(Product.class, revisionNumber)
    .add(AuditEntity.property("price").gt(50.0))
    .addOrder(AuditEntity.property("name").asc())
    .setMaxResults(10)
    .setFirstResult(0)
    .getResultList();

// Aggregate functions:
reader.createQuery()
    .forRevisionsOfEntity(Product.class, false, false)
    .addProjection(AuditEntity.revisionNumber().max())
    .add(AuditEntity.id().eq(productId))
    .getSingleResult();  // gets latest revision number for entity

// Property changed in a specific revision:
reader.createQuery()
    .forRevisionsOfEntity(Product.class, false, false)
    .add(AuditEntity.property("price").hasChanged())
    .add(AuditEntity.revisionNumber().gt(5))
    .getResultList();

// Combined with Spring Data — RevisionRepository:
// If repository extends RevisionRepository<T, ID, RevisionType>:
public interface ProductRepository
        extends JpaRepository<Product, Long>,
                RevisionRepository<Product, Long, Integer> {
}

// Then:
Revisions<Integer, Product> revisions = productRepo.findRevisions(productId);
Optional<Revision<Integer, Product>> latest = productRepo.findLastChangeRevision(productId);
Page<Revision<Integer, Product>> page = productRepo.findRevisions(productId, pageable);
```

---

### Envers Table Naming and Configuration

```properties
# application.properties — Envers configuration:

# Audit table suffix (default: _AUD):
spring.jpa.properties.org.hibernate.envers.audit_table_suffix=_aud

# Audit table prefix (default: empty):
spring.jpa.properties.org.hibernate.envers.audit_table_prefix=

# Revision field name in audit tables (default: REV):
spring.jpa.properties.org.hibernate.envers.revision_field_name=rev

# RevisionType field name (default: REVTYPE):
spring.jpa.properties.org.hibernate.envers.revision_type_field_name=revtype

# Store data at deletion (default: false — stores nulls):
spring.jpa.properties.org.hibernate.envers.store_data_at_delete=true
# true: DELETE revision stores entity data before deletion (not nulls)
# Very useful for "what was deleted?" queries

# Default schema for audit tables:
spring.jpa.properties.org.hibernate.envers.default_schema=audit

# Audit table for revinfo (default: revinfo):
spring.jpa.properties.org.hibernate.envers.revision_table_name=rev_info
```

---

### `@AuditTable` — Custom Audit Table Name

```java
@Entity
@Audited
@AuditTable(value = "product_history",  // custom audit table name
            schema = "audit",            // custom schema
            catalog = "mydb")
public class Product { ... }
// Instead of default: products_aud
// Uses: audit.product_history
```

---

### `@AuditOverride` — Inherited Audit Configurations

```java
// Base class with audit configuration:
@MappedSuperclass
@Audited
public abstract class BaseEntity {
    @Id @GeneratedValue Long id;
    String name;
}

// Override audit configuration in subclass:
@Entity
@Audited
@AuditOverride(forClass = BaseEntity.class, isAudited = false)
// → Exclude BaseEntity's fields from auditing for THIS entity
public class TemporaryEntity extends BaseEntity {
    LocalDateTime expiresAt;
    // Only expiresAt is audited; id and name (from BaseEntity) are NOT
}
```

---

### `store_data_at_delete = true` — Why It Matters

```java
// Without store_data_at_delete=true (default):
// products_aud on DELETE:
// id=1, REV=5, REVTYPE=2(DEL), name=null, price=null
// → You cannot recover what the product was called before deletion

// With store_data_at_delete=true:
// products_aud on DELETE:
// id=1, REV=5, REVTYPE=2(DEL), name="Widget", price=9.99
// → Full state preserved at time of deletion
// → "What was deleted?" queries work correctly
```

---

## 2️⃣ Code Examples

### Example 1 — Complete Spring Data Auditing Setup

```java
// ── 1. Configuration ──────────────────────────────────────────────────────
@Configuration
@EnableJpaAuditing(
    auditorAwareRef = "auditorProvider",
    dateTimeProviderRef = "utcDateTimeProvider"
)
public class AuditingConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> {
            Authentication auth = SecurityContextHolder.getContext()
                                                       .getAuthentication();
            if (auth == null || !auth.isAuthenticated()
                    || "anonymousUser".equals(auth.getPrincipal())) {
                return Optional.of("SYSTEM");
            }
            return Optional.of(auth.getName());
        };
    }

    @Bean
    public DateTimeProvider utcDateTimeProvider() {
        return () -> Optional.of(
            LocalDateTime.now(ZoneOffset.UTC)
        );
    }
}

// ── 2. Base entity ────────────────────────────────────────────────────────
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditableEntity {

    @CreatedDate
    @Column(name = "created_at", updatable = false, nullable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "last_modified_at", nullable = false)
    private LocalDateTime lastModifiedAt;

    @CreatedBy
    @Column(name = "created_by", updatable = false, length = 100)
    private String createdBy;

    @LastModifiedBy
    @Column(name = "last_modified_by", length = 100)
    private String lastModifiedBy;

    // Getters only:
    public LocalDateTime getCreatedAt()     { return createdAt; }
    public LocalDateTime getLastModifiedAt(){ return lastModifiedAt; }
    public String getCreatedBy()            { return createdBy; }
    public String getLastModifiedBy()       { return lastModifiedBy; }
}

// ── 3. Entity extending base ──────────────────────────────────────────────
@Entity
@Table(name = "orders")
public class Order extends AuditableEntity {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String customerId;
    private BigDecimal total;
    private OrderStatus status;

    @Version
    private Integer version;

    // Domain fields only — audit fields inherited
}

// ── 4. Demonstration ──────────────────────────────────────────────────────
@Service
public class OrderService {

    @Transactional
    public Order createOrder(String customerId, BigDecimal total) {
        Order order = new Order();
        order.setCustomerId(customerId);
        order.setTotal(total);
        order.setStatus(OrderStatus.PENDING);

        Order saved = orderRepo.save(order);
        // @PrePersist fires → AuditingEntityListener sets:
        //   createdAt      = now() UTC
        //   lastModifiedAt = now() UTC
        //   createdBy      = "alice" (from SecurityContext)
        //   lastModifiedBy = "alice"

        System.out.println(saved.getCreatedAt());     // 2024-01-15T10:00:00
        System.out.println(saved.getCreatedBy());     // "alice"
        return saved;
    }

    @Transactional
    public Order updateStatus(Long orderId, OrderStatus status) {
        Order order = orderRepo.findById(orderId).get();
        order.setStatus(status);
        // @PreUpdate fires → AuditingEntityListener sets:
        //   lastModifiedAt = now() UTC (updated)
        //   lastModifiedBy = "bob" (currently logged in)
        //   createdAt      = unchanged (updatable=false)
        //   createdBy      = unchanged (updatable=false)
        return order;
    }
}
```

---

### Example 2 — Envers Full Setup with Custom Revision Entity

```java
// ── 1. Custom revision entity ──────────────────────────────────────────
@Entity
@RevisionEntity(AuditRevisionListener.class)
@Table(name = "audit_revision")
public class AuditRevision extends DefaultRevisionEntity {

    @Column(name = "username", length = 100)
    private String username;

    @Column(name = "user_id")
    private Long userId;

    @Column(name = "ip_address", length = 45)
    private String ipAddress;

    // getters and setters
    public String getUsername()     { return username; }
    public void setUsername(String u) { username = u; }
    public Long getUserId()         { return userId; }
    public void setUserId(Long id)  { userId = id; }
    public String getIpAddress()    { return ipAddress; }
    public void setIpAddress(String ip) { ipAddress = ip; }
}

// ── 2. Revision listener ───────────────────────────────────────────────
public class AuditRevisionListener implements RevisionListener {

    @Override
    public void newRevision(Object revisionEntity) {
        AuditRevision rev = (AuditRevision) revisionEntity;

        Authentication auth = SecurityContextHolder.getContext()
                                                   .getAuthentication();
        if (auth != null && auth.isAuthenticated()
                && !"anonymousUser".equals(auth.getPrincipal())) {
            rev.setUsername(auth.getName());
            if (auth.getPrincipal() instanceof UserDetails ud) {
                // Set user ID if available
            }
        } else {
            rev.setUsername("SYSTEM");
        }

        // Capture IP from request context (if in web request):
        RequestAttributes attrs =
            RequestContextHolder.getRequestAttributes();
        if (attrs instanceof ServletRequestAttributes sra) {
            rev.setIpAddress(sra.getRequest().getRemoteAddr());
        }
    }
}

// ── 3. Audited entity ─────────────────────────────────────────────────
@Entity
@Audited
@AuditTable("product_audit_history")
public class Product {

    @Id @GeneratedValue Long id;

    String name;
    BigDecimal price;
    String category;

    @NotAudited  // Large field — exclude from audit to save space
    String fullDescription;

    @ManyToOne
    @Audited(targetAuditMode = RelationTargetAuditMode.NOT_AUDITED)
    Category cat;  // category not audited — only FK stored
}

// ── 4. History service ─────────────────────────────────────────────────
@Service
@Transactional(readOnly = true)
public class ProductAuditService {

    @PersistenceContext EntityManager em;

    // Get all revisions of a product:
    public List<ProductRevisionDto> getProductHistory(Long productId) {
        AuditReader reader = AuditReaderFactory.get(em);
        List<Object[]> history = reader.createQuery()
            .forRevisionsOfEntity(Product.class, false, true)
            .add(AuditEntity.id().eq(productId))
            .addOrder(AuditEntity.revisionNumber().asc())
            .getResultList();

        return history.stream().map(row -> {
            Product product      = (Product) row[0];
            AuditRevision rev    = (AuditRevision) row[1];
            RevisionType revType = (RevisionType) row[2];

            return new ProductRevisionDto(
                rev.getId(),
                rev.getRevisionDate(),
                rev.getUsername(),
                revType.name(),
                product == null ? null : product.getName(),
                product == null ? null : product.getPrice()
            );
        }).collect(Collectors.toList());
    }

    // Restore product to a previous state:
    @Transactional
    public Product restoreToRevision(Long productId, int targetRevision) {
        AuditReader reader = AuditReaderFactory.get(em);
        Product historicalState = reader.find(
            Product.class, productId, targetRevision);

        if (historicalState == null) {
            throw new RevisionNotFoundException(productId, targetRevision);
        }

        Product current = productRepo.findById(productId)
            .orElseThrow(ProductNotFoundException::new);

        // Apply historical values to current entity:
        current.setName(historicalState.getName());
        current.setPrice(historicalState.getPrice());
        current.setCategory(historicalState.getCategory());
        // New revision created with restored values
        return current;
    }

    // Find what changed between two revisions:
    public List<String> findChangedFields(Long productId,
                                           int revFrom, int revTo) {
        AuditReader reader = AuditReaderFactory.get(em);
        Product stateFrom = reader.find(Product.class, productId, revFrom);
        Product stateTo   = reader.find(Product.class, productId, revTo);

        List<String> changes = new ArrayList<>();
        if (!Objects.equals(stateFrom.getName(), stateTo.getName()))
            changes.add("name: " + stateFrom.getName()
                                 + " → " + stateTo.getName());
        if (!Objects.equals(stateFrom.getPrice(), stateTo.getPrice()))
            changes.add("price: " + stateFrom.getPrice()
                                  + " → " + stateTo.getPrice());
        return changes;
    }
}
```

---

### Example 3 — Spring Data `RevisionRepository`

```java
// Repository extending RevisionRepository:
public interface ProductRepository
        extends JpaRepository<Product, Long>,
                RevisionRepository<Product, Long, Integer> {
    // Integer = revision number type (from DefaultRevisionEntity.id)
}

@Service
public class ProductRevisionService {

    @Autowired ProductRepository productRepo;

    // Get all revisions:
    public Revisions<Integer, Product> getAllRevisions(Long productId) {
        return productRepo.findRevisions(productId);
        // Returns Revisions object with List<Revision<Integer, Product>>
    }

    // Get specific revision:
    public Optional<Revision<Integer, Product>> getRevision(
            Long productId, int revNumber) {
        return productRepo.findRevision(productId, revNumber);
    }

    // Get latest revision:
    public Optional<Revision<Integer, Product>> getLatest(Long productId) {
        return productRepo.findLastChangeRevision(productId);
    }

    // Paginated revisions:
    public Page<Revision<Integer, Product>> getRevisionPage(
            Long productId, Pageable pageable) {
        return productRepo.findRevisions(productId, pageable);
    }

    // Using revision metadata:
    public void printHistory(Long productId) {
        productRepo.findRevisions(productId).forEach(revision -> {
            System.out.printf("Rev %d at %s: %s%n",
                revision.getRevisionNumber().orElse(-1),
                revision.getRevisionInstant().orElse(null),
                revision.getEntity().getName());
        });
    }
}
```

---

### Example 4 — `@NotAudited` and Selective Auditing

```java
@Entity
@Audited
public class UserProfile {

    @Id @GeneratedValue Long id;

    // Audited fields (tracked in history):
    String username;
    String email;
    String role;
    boolean active;

    // NOT audited — large/sensitive data excluded:
    @NotAudited
    @Lob
    String bio;                     // Large text — no audit value

    @NotAudited
    byte[] avatarImage;             // Binary — too large for audit table

    @NotAudited
    String passwordHash;            // Security — never audit password hashes!

    @NotAudited
    LocalDateTime lastLoginAt;      // High-frequency, low audit value
    // (login updates would create too many revisions)

    // Audited association:
    @ManyToMany
    @Audited(targetAuditMode = RelationTargetAuditMode.NOT_AUDITED)
    List<Permission> permissions;   // Audit which permissions assigned
                                    // but Permission entity itself not audited
}

// Result: user_profile_aud tracks changes to username, email, role,
//         active, and permissions — but not bio, avatar, password, or lastLogin
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ: Missing `@EntityListeners`**
```java
@Entity
@EnableJpaAuditing  // ← on entity (wrong place, but ignore for this question)
public class Invoice {
    @Id @GeneratedValue Long id;
    @CreatedDate LocalDateTime createdAt;
    @LastModifiedDate LocalDateTime updatedAt;
    BigDecimal amount;
}
```

`@EnableJpaAuditing` is correctly placed on a `@Configuration` class. The entity definition above is missing something. What is the result?

A) `createdAt` and `updatedAt` are populated correctly  
B) `createdAt` and `updatedAt` remain null — `@EntityListeners(AuditingEntityListener.class)` is missing  
C) `ApplicationContext` fails to start — `@CreatedDate` requires `@EntityListeners`  
D) Only `createdAt` is populated; `updatedAt` requires additional config  

**Answer: B** — Without `@EntityListeners(AuditingEntityListener.class)` on the entity (or on a `@MappedSuperclass` it extends), the `AuditingEntityListener` is never registered for that entity. JPA lifecycle callbacks (`@PrePersist`, `@PreUpdate`) are never fired. Audit fields stay null. No startup error — it silently fails.

---

**Q2 — Select All That Apply: `@CreatedDate` Behaviour**

Which statements about `@CreatedDate` are TRUE?

A) `@CreatedDate` is set on both `@PrePersist` and `@PreUpdate`  
B) `@CreatedDate` is set only on `@PrePersist` and left unchanged on `@PreUpdate`  
C) `@Column(updatable = false)` provides DB-level protection against overwriting  
D) If `@CreatedDate` field is already non-null, `AuditingEntityListener` skips it  
E) `@CreatedDate` requires `@Version` on the entity  
F) `@CreatedDate` is set by the `AuditorAware` bean  

**Answer: B, C, D**
- A is false: `@PreUpdate` leaves `@CreatedDate` unchanged
- E is false: `@Version` is independent of auditing
- F is false: `AuditorAware` provides the **user** for `@CreatedBy`/`@LastModifiedBy`; dates use `DateTimeProvider` or `Clock`

---

**Q3 — Envers `REVTYPE` Values**

A product entity with `@Audited` is created, then updated twice, then deleted. How many rows appear in the `products_aud` table, and what are their `REVTYPE` values?

A) 3 rows: REVTYPE = 0, 1, 2  
B) 4 rows: REVTYPE = 0, 1, 1, 2  
C) 2 rows: REVTYPE = 0, 2 (only create and delete tracked)  
D) 4 rows: REVTYPE = 1, 1, 1, 2  

**Answer: B** — Envers creates one audit row per modification: INSERT (REVTYPE=0), first UPDATE (REVTYPE=1), second UPDATE (REVTYPE=1), DELETE (REVTYPE=2). Total = 4 rows with REVTYPE values 0, 1, 1, 2.

---

**Q4 — `AuditorAware` Returning Empty**
```java
@Bean
public AuditorAware<String> auditorProvider() {
    return Optional::empty;  // always returns empty
}
```

What happens to `@CreatedBy` and `@LastModifiedBy` fields?

A) `IllegalStateException` — `AuditorAware` must return a non-empty `Optional`  
B) Fields are set to `null` (or remain `null`) — no exception  
C) Fields are set to `"anonymous"`  
D) Auditing is disabled entirely for the application  

**Answer: B** — When `AuditorAware` returns `Optional.empty()`, Spring Data skips populating `@CreatedBy` and `@LastModifiedBy` fields. They remain null (or whatever value they had). No exception is thrown. This is the expected behavior for batch jobs or system processes with no authenticated user.

---

**Q5 — Envers `store_data_at_delete`**
```properties
spring.jpa.properties.org.hibernate.envers.store_data_at_delete=false
```

A product with name="Widget" and price=9.99 is deleted. What is stored in `products_aud`?

A) `id=1, REVTYPE=2, name="Widget", price=9.99` (full state at deletion)  
B) `id=1, REVTYPE=2, name=null, price=null` (entity data is null)  
C) No row inserted — delete operations are not tracked by default  
D) Row is removed from `products_aud` when the entity is deleted  

**Answer: B** — With `store_data_at_delete=false` (the default), Envers stores null for all entity fields in the DELETE revision row. Only the `id` and `REVTYPE=2` are meaningful. To preserve entity data at deletion, set `store_data_at_delete=true`.

---

**Q6 — `@NotAudited` Effect**
```java
@Entity
@Audited
public class Employee {
    @Id Long id;
    String name;
    @NotAudited String salary;
    boolean active;
}
```

Employee id=1 has `name="Alice"`, `salary="50000"`, `active=true`. The `salary` is updated to `"60000"`. What appears in `employees_aud`?

A) A new revision row with `name="Alice"`, `salary="60000"`, `active=true`  
B) A new revision row with `name="Alice"`, `salary=null`, `active=true`  
C) No new revision — only audited fields trigger revision creation  
D) A new revision row with `name="Alice"` and `active=true`; no `salary` column in `employees_aud`  

**Answer: D** — `@NotAudited` fields are excluded from the audit table entirely — the `salary` **column does not exist** in `employees_aud`. However, any modification to the entity (even of a `@NotAudited` field) still creates a new revision row. The audited fields (`name`, `active`) are captured at the time of that revision.

---

**Q7 — `RevisionRepository` Method**

Which method returns the most recent revision of a specific entity?

A) `findRevisions(id)`  
B) `findRevision(id, revisionNumber)`  
C) `findLastChangeRevision(id)`  
D) `findLatestRevision(id)`  

**Answer: C** — `findLastChangeRevision(ID id)` returns `Optional<Revision<N, T>>` representing the most recent revision. `findRevisions(id)` returns ALL revisions. `findRevision(id, revisionNumber)` returns a specific numbered revision. `findLatestRevision()` does not exist in the API.

---

**Q8 — Audit with `@MappedSuperclass`**
```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {
    @CreatedDate LocalDateTime createdAt;
    @LastModifiedDate LocalDateTime updatedAt;
}

@Entity
public class Invoice extends BaseEntity {
    @Id @GeneratedValue Long id;
    BigDecimal amount;
}
```

Is `@EntityListeners(AuditingEntityListener.class)` on `BaseEntity` sufficient for `Invoice`?

A) No — `@EntityListeners` must be on the `@Entity` class directly  
B) Yes — `@EntityListeners` on `@MappedSuperclass` is inherited by all subclass entities  
C) Only if `Invoice` also has `@Audited`  
D) No — `@MappedSuperclass` cannot have `@EntityListeners`  

**Answer: B** — JPA propagates `@EntityListeners` from `@MappedSuperclass` to all concrete entity subclasses. This is the recommended pattern — place `@EntityListeners(AuditingEntityListener.class)` on the shared `@MappedSuperclass` base and all entities that extend it automatically get auditing without repeating the annotation.

---

## 4️⃣ Trick Analysis

**Missing `@EntityListeners` — silent failure (Q1)**:
This is the most common auditing bug. `@EnableJpaAuditing` activates the auditing infrastructure, but each entity must explicitly register the listener via `@EntityListeners(AuditingEntityListener.class)`. Without it, `@CreatedDate` and `@LastModifiedDate` annotations exist on fields but are never invoked — fields stay null. No exception, no warning. The fix is always to put `@EntityListeners` on the `@MappedSuperclass` that all entities extend, ensuring it applies universally.

**`@CreatedDate` checked for null before setting (Q2, D)**:
On `@PrePersist`, Spring Data sets `@CreatedDate` only if the field is currently null. If you pre-populate `createdAt` before saving (for migration or import purposes), the existing value is preserved. On `@PreUpdate`, `@CreatedDate` fields are completely skipped — the listener does not overwrite them. The combination of this behavior plus `@Column(updatable=false)` gives two layers of protection (Java layer + DB layer).

**`@NotAudited` removes the column from audit table entirely (Q6)**:
A subtle distinction: `@NotAudited` doesn't mean "null in audit table." It means the column doesn't exist in `_aud` table at all. Modifications to `@NotAudited` fields still trigger revision creation — Envers captures all entity state changes. But the excluded field's column simply isn't part of the audit table schema. This means: an employee's salary change creates a new revision, but the `salary` column won't be there to tell you what changed. Use `@NotAudited` only for fields that have no audit value (large binaries, frequently-updated low-value fields, sensitive fields like passwords).

**`store_data_at_delete=false` — the deletion black hole (Q5)**:
The default Envers behavior stores nulls for entity fields in DELETE revision rows. You know something was deleted and when, but not what it contained. For most audit requirements this is insufficient. `store_data_at_delete=true` is almost always the right production setting — it captures the last known state at deletion time. The exam tests whether you know the default behavior produces null-filled DELETE rows.

**`AuditorAware` returning empty vs throwing (Q4)**:
Many developers assume `Optional.empty()` from `AuditorAware` causes an exception. It does not. Spring Data uses the result defensively: empty = skip `@CreatedBy`/`@LastModifiedBy` population. This is intentional for background tasks, batch jobs, and system processes that operate without a user context. The contract is: if there's no current user, just don't set the field. Your `AuditorAware` implementation should return `Optional.empty()` (not throw) for system contexts.

**`@EntityListeners` inheritance from `@MappedSuperclass` (Q8)**:
JPA specification defines that entity listeners declared on a `@MappedSuperclass` are inherited by all entity subclasses unless explicitly excluded with `@ExcludeSuperclassListeners`. This makes `@MappedSuperclass` the ideal location for `@EntityListeners(AuditingEntityListener.class)` — declare once, applies everywhere. The exam tests this inheritance rule because many developers mistakenly repeat the annotation on every entity.

---

## 5️⃣ Summary Sheet

### Spring Data JPA Auditing — Setup Checklist

```
Step 1: Add @EnableJpaAuditing to a @Configuration class

Step 2: Implement AuditorAware<T> bean (for @CreatedBy/@LastModifiedBy)
        → Bean name referenced in @EnableJpaAuditing(auditorAwareRef="...")

Step 3: Add @EntityListeners(AuditingEntityListener.class)
        → Best on @MappedSuperclass (inherited by all entities)

Step 4: Add audit annotations to entity fields:
        @CreatedDate    — set once on INSERT
        @LastModifiedDate — set on INSERT and every UPDATE
        @CreatedBy      — set once on INSERT (from AuditorAware)
        @LastModifiedBy — set on INSERT and every UPDATE (from AuditorAware)

Step 5: Add @Column(updatable = false) on @CreatedDate and @CreatedBy fields
        (DB-level protection to complement Java-level protection)
```

### Audit Annotation Behaviour Reference

| Annotation | `@PrePersist` | `@PreUpdate` | Source |
|---|---|---|---|
| `@CreatedDate` | SET (if null) | Skipped | `DateTimeProvider` / `Clock` |
| `@LastModifiedDate` | SET | SET (always) | `DateTimeProvider` / `Clock` |
| `@CreatedBy` | SET (if null) | Skipped | `AuditorAware.getCurrentAuditor()` |
| `@LastModifiedBy` | SET | SET (always) | `AuditorAware.getCurrentAuditor()` |

### Envers Audit Table Structure

```
{entity_table}_aud:
  → All entity columns (except @NotAudited fields)
  → REV: FK to revinfo.rev (revision number)
  → REVTYPE: 0=ADD, 1=MOD, 2=DEL

revinfo (or custom @RevisionEntity table):
  → REV: revision number (auto-incremented)
  → REVTSTMP: timestamp (epoch milliseconds)
  → (custom fields from @RevisionEntity subclass)
```

### Envers Key Configuration

```properties
org.hibernate.envers.audit_table_suffix=_aud        # default
org.hibernate.envers.store_data_at_delete=true       # RECOMMENDED (not default!)
org.hibernate.envers.revision_field_name=REV         # default
org.hibernate.envers.revision_type_field_name=REVTYPE # default
```

### `AuditReader` Query Method Reference

```java
reader.find(Class, id, revision)         // entity at specific revision
reader.getRevisions(Class, id)           // all revision numbers for entity
reader.getRevisionNumberForDate(date)    // revision number at a point in time

reader.createQuery()
  .forEntitiesAtRevision(Class, rev)     // all entities of type at revision
  .forRevisionsOfEntity(Class, selectEntities, selectDeleted)
                                         // history of a type
  .add(AuditEntity.id().eq(id))          // filter by id
  .add(AuditEntity.property("name").eq(value))
  .add(AuditEntity.revisionType().eq(RevisionType.DEL))
  .add(AuditEntity.property("price").hasChanged())
  .addOrder(AuditEntity.revisionNumber().asc())
  .setMaxResults(n)
  .getResultList() / getSingleResult()
```

### Key Rules

```
1.  @EntityListeners(AuditingEntityListener.class) REQUIRED on entity or @MappedSuperclass
2.  @EnableJpaAuditing belongs on @Configuration class — NOT on entity
3.  @CreatedDate: set once on INSERT, never touched on UPDATE
4.  @LastModifiedDate: set on both INSERT and UPDATE
5.  AuditorAware.getCurrentAuditor() returning Optional.empty() → fields not populated (no exception)
6.  @Column(updatable=false) on @CreatedDate and @CreatedBy for DB-level protection
7.  @EntityListeners on @MappedSuperclass is inherited by all entity subclasses
8.  Envers REVTYPE: 0=ADD, 1=MOD, 2=DEL
9.  store_data_at_delete=false (default): DELETE revision stores null for entity fields
10. store_data_at_delete=true (recommended): DELETE revision stores last-known entity state
11. @NotAudited: column excluded from _aud table entirely (not stored as null)
12. @NotAudited fields still trigger revision creation if other fields change
13. @Audited(targetAuditMode=NOT_AUDITED): audit FK of related entity, not full related entity
14. RevisionRepository.findLastChangeRevision(id): latest revision for entity
15. AuditReader.find(Class, id, revision): entity state at specific revision number
16. Custom @RevisionEntity: extend DefaultRevisionEntity, annotate with @RevisionEntity(Listener.class)
17. RevisionListener.newRevision(): populate custom revision fields (username, IP, etc.)
18. @AuditTable: override audit table name/schema per entity
```

---
