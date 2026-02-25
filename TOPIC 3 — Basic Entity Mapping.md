# 🏛️ TOPIC 3 — Basic Entity Mapping

---

## 1️⃣ Conceptual Explanation

### The Entity Mapping Contract

When Hibernate bootstraps, it builds a **mapping metamodel** — a complete in-memory description of how every Java class maps to database tables and columns. This metamodel is constructed once at `SessionFactory` creation time and is immutable thereafter. Every query, dirty check, and hydration operation consults this metamodel.

The mapping metadata can come from three sources (in order of precedence):
1. **Annotations** on the class (most common in modern Spring)
2. **`orm.xml`** — XML override file (can partially or fully override annotations)
3. **`hbm.xml`** — legacy Hibernate mapping files (avoid in new projects)

Understanding how Hibernate reads and processes these annotations is fundamental to understanding why certain combinations work, fail silently, or produce unexpected SQL.

---

### `@Entity` — The Declaration Contract

```java
@Entity
public class Product { ... }
```

When Hibernate sees `@Entity`, it:

1. Registers the class in the metamodel as a **managed type**
2. Generates a `SingleTableEntityPersister` (or `JoinedSubclassEntityPersister` / `UnionSubclassEntityPersister` depending on inheritance strategy)
3. Introspects all fields and methods to build `AttributeMapping` descriptors
4. Resolves the table name via naming strategy
5. Validates the presence of exactly one `@Id` (or `@EmbeddedId` / `@IdClass`)

**Requirements enforced by JPA spec:**
- Must have a **no-arg constructor** (public or protected) — Hibernate uses it for object instantiation during hydration
- Must **not be final** (prevents proxy subclass generation for lazy loading — a critical constraint)
- Must **not have final methods** that need to be overridden in proxies
- Must be a **top-level class** (not inner/anonymous)

**The no-arg constructor trap with Kotlin/Lombok:**
```java
// Lombok @Data generates no no-arg constructor if fields are final
// This BREAKS Hibernate hydration silently
@Data  // generates constructor with all fields
@Entity
public class Product {
    @Id private Long id;
    private String name;
    // No no-arg constructor → HibernateException at runtime
}

// Fix: add @NoArgsConstructor
@Data
@NoArgsConstructor
@Entity
public class Product { ... }
```

**`@Entity(name = "...")`** — sets the entity name used in JPQL (NOT the table name):
```java
@Entity(name = "Prod")  // JPQL entity name
@Table(name = "products") // SQL table name
public class Product { ... }

// JPQL uses entity name:
"SELECT p FROM Prod p" // correct
"SELECT p FROM Product p" // wrong — entity name is "Prod"
"SELECT p FROM products p" // wrong — that's SQL, not JPQL
```

This distinction between entity name and table name is a **classic exam trap**.

---

### `@Table` — The Table Mapping

```java
@Table(
    name = "products",
    schema = "inventory",
    catalog = "mydb",
    uniqueConstraints = {
        @UniqueConstraint(name = "uk_product_sku", columnNames = {"sku"})
    },
    indexes = {
        @Index(name = "idx_product_name", columnList = "name"),
        @Index(name = "idx_product_price_name", columnList = "price DESC, name ASC")
    }
)
```

`@Table` is optional. Without it, the table name is derived by the naming strategy from the class name.

**`schema` and `catalog`**: Used in generated DDL and all SQL. If your DB requires schema-qualified queries: `SELECT * FROM inventory.products`. These are included automatically.

**`uniqueConstraints`**: Only used for **DDL generation** (`ddl-auto = create/update`). They do NOT affect query behavior or enforce uniqueness at the JPA level. The constraint is created in the database schema. At runtime, violations throw `ConstraintViolationException` from the database, wrapped in `PersistenceException`.

**`indexes`**: Also DDL-only. Generated as `CREATE INDEX` statements. No effect on query execution — the DB optimizer decides whether to use them.

---

### `@Id` — The Primary Key Contract

Every `@Entity` must have exactly one `@Id` field (or `@EmbeddedId` / `@IdClass` for composite keys — covered in Topic 4 adjacent).

```java
@Id
private Long id;
```

**Supported `@Id` types** (JPA spec mandates these work):
- `byte`, `short`, `int`, `long` and their wrappers
- `java.math.BigDecimal`, `java.math.BigInteger`
- `java.lang.String`
- `java.util.Date`, `java.sql.Date`
- `java.util.UUID`

**Why use `Long` over `long` (primitive)?**

Hibernate uses `null` ID to determine if an entity is new (transient) vs detached. With primitive `long`, the ID defaults to `0` — which Hibernate may interpret as a valid ID, causing `merge()` to try SELECT for ID=0 instead of INSERT.

```java
// Dangerous:
@Id
private long id; // defaults to 0 → Hibernate may think it's detached

// Safe:
@Id
private Long id; // defaults to null → Hibernate correctly identifies as new
```

`SimpleJpaRepository.save()` uses this logic:
```java
// from SimpleJpaRepository source:
public <S extends T> S save(S entity) {
    if (entityInformation.isNew(entity)) {
        em.persist(entity);
        return entity;
    } else {
        return em.merge(entity);
    }
}

// isNew() checks:
// 1. If entity implements Persistable<ID>: uses entity.isNew()
// 2. Otherwise: checks if @Id field is null (or 0 for primitives)
```

This `isNew()` determination is fundamental. It decides `persist()` vs `merge()` — the wrong choice causes bugs.

---

### `@GeneratedValue` — The Four Strategies

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE)
private Long id;
```

#### Strategy 1: `GenerationType.AUTO`

The **default**. JPA delegates to the provider to pick the best strategy for the dialect.

- In Hibernate 5+ with most dialects: uses `hibernate_sequence` (a single shared sequence)
- In Hibernate 6+: defaults to SEQUENCE for most DBs, TABLE for others
- **Problem**: Hibernate 5 AUTO uses a single global sequence for ALL entities → ID collision is impossible (each entity gets globally unique IDs) but allocation is inefficient and confusing

```java
// AUTO behavior in Hibernate 5 — one sequence for everything:
@Entity class Product { @Id @GeneratedValue Long id; }
@Entity class Order   { @Id @GeneratedValue Long id; }

// Both use: SELECT nextval('hibernate_sequence')
// Product might get id=1, Order gets id=2, Product gets id=3
// IDs are globally unique across tables
```

#### Strategy 2: `GenerationType.IDENTITY`

Relies on the database's auto-increment column (`AUTO_INCREMENT` in MySQL, `SERIAL`/`BIGSERIAL` in PostgreSQL, `IDENTITY` in SQL Server).

```sql
CREATE TABLE products (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255)
);
```

**Internal mechanics — the critical IDENTITY problem:**

IDENTITY requires the database to generate the ID at INSERT time. Hibernate needs to know the generated ID immediately after INSERT (to put it in the identity map). Therefore:

- Hibernate **cannot defer the INSERT** to flush time
- The INSERT must execute **immediately** when `em.persist()` is called
- This **breaks JDBC batch inserts**

```java
// With SEQUENCE: persist() → enqueue action → flush → batch INSERT 100 rows in one batch
// With IDENTITY: persist() → immediate INSERT → get generated key → repeat per entity
// IDENTITY = 100 individual INSERT statements, no batching
```

This is why **SEQUENCE is strongly preferred over IDENTITY for high-throughput applications**.

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

// When you call:
em.persist(product);
// SQL fires IMMEDIATELY:
// INSERT INTO products (name) VALUES (?) 
// then: SELECT SCOPE_IDENTITY() or use getGeneratedKeys()
// id is now populated in the Java object
```

#### Strategy 3: `GenerationType.SEQUENCE` ⭐ (Preferred)

Uses a database sequence object. Hibernate calls `SELECT nextval('seq_name')` to get the next ID value, then uses it in the INSERT.

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
@SequenceGenerator(
    name = "product_seq",         // generator name (matches above)
    sequenceName = "seq_products", // actual DB sequence name
    allocationSize = 50,           // how many IDs to allocate per DB call
    initialValue = 1
)
private Long id;
```

**The `allocationSize` optimization — Hi-Lo Algorithm:**

This is one of Hibernate's most important performance features and a deep exam topic.

Without optimization: every `persist()` → one `SELECT nextval()` → one DB roundtrip for every entity.

With `allocationSize = 50`: Hibernate calls `SELECT nextval()` once, gets back value N, then allocates IDs from `(N-1)*50 + 1` to `N*50` **in memory**. For 50 entities, only 1 DB sequence call. 99% reduction in sequence calls.

```
DB Sequence state: INCREMENT BY 50 (must match allocationSize!)
First call:  SELECT nextval() → returns 1
Hibernate allocates: IDs 1-50 in memory
Next 49 persists: use IDs 2-50 with NO DB calls
51st persist: SELECT nextval() → returns 51
Hibernate allocates: IDs 51-100
...
```

**Critical setup requirement**: The database sequence must have `INCREMENT BY 50` (matching `allocationSize`). If it's `INCREMENT BY 1` (default), IDs will be: 1, 51, 101... with massive gaps. This is a common misconfiguration.

```sql
-- Correct sequence definition for allocationSize=50:
CREATE SEQUENCE seq_products
    START WITH 1
    INCREMENT BY 50
    NO MINVALUE
    NO MAXVALUE
    CACHE 1;
```

Hibernate's default `allocationSize` is **50**. This surprises many developers who see gaps in IDs and wonder why.

**Pooled Optimizer** (Hibernate 5+):
The classic Hi-Lo can cause issues when multiple app nodes share a sequence (because each node caches its own block of 50 IDs). The **Pooled optimizer** (default in Hibernate 5+) solves this by having the sequence itself track the "upper bound" of the current block, making it safe for clustering.

```java
@SequenceGenerator(
    name = "product_seq",
    sequenceName = "seq_products",
    allocationSize = 50
)
// Hibernate 5+ uses Pooled optimizer by default
// Can also set: hibernate.id.optimizer.pooled.prefer_lo=true
```

#### Strategy 4: `GenerationType.TABLE`

Uses a dedicated database table to simulate sequences. **Avoid in production.**

```sql
CREATE TABLE hibernate_sequences (
    sequence_name VARCHAR(255) NOT NULL,
    next_val BIGINT,
    PRIMARY KEY (sequence_name)
);
```

Problems:
- Requires pessimistic row-level locking on the sequence table → serialization bottleneck
- Every ID generation = `SELECT ... FOR UPDATE` + `UPDATE` → catastrophic under load
- Exists for portability with databases that don't support sequences (very old DBs)

---

### `@Column` — The Column Mapping Contract

```java
@Column(
    name = "product_name",       // SQL column name
    nullable = false,             // NOT NULL constraint (DDL only)
    unique = false,               // UNIQUE constraint (DDL only)
    length = 255,                 // VARCHAR length (DDL only)
    precision = 10,               // for BigDecimal — total digits
    scale = 2,                    // for BigDecimal — decimal places
    insertable = true,            // include in INSERT SQL
    updatable = true,             // include in UPDATE SQL
    columnDefinition = "TEXT"     // override DDL type completely
)
private String name;
```

**`nullable = false`** — DDL-only. Does NOT add a runtime null check. Hibernate will let you persist a null value; the database will reject it. Exception: Bean Validation (`@NotNull`) adds runtime validation.

**`insertable = false, updatable = false`** — Very powerful:
```java
// Classic use case: read-only column managed by DB (e.g., creation timestamp)
@Column(name = "created_at", insertable = false, updatable = false)
private LocalDateTime createdAt;
// Hibernate never includes this in INSERT or UPDATE SQL
// Value comes from DB default or trigger

// Another use: join column in bidirectional mapping (prevent double-write)
@Column(name = "order_id", insertable = false, updatable = false)
private Long orderId; // read the FK value as a plain field without managing it
```

**`columnDefinition`** — Overrides Hibernate's DDL type inference:
```java
@Column(columnDefinition = "TEXT")
private String description; // Generates: description TEXT (not VARCHAR(255))

@Column(columnDefinition = "JSONB")
private String metadata; // PostgreSQL JSONB type
```

**`length`** — Only for `VARCHAR`. Ignored for `TEXT`, `CLOB`, or numeric types.

**`precision` and `scale`** for `BigDecimal`:
```java
@Column(precision = 19, scale = 4)
private BigDecimal price;
// DDL: DECIMAL(19,4)
// Stores up to 999999999999999.9999
```

---

### `@Basic` — The Implicit Default

Every persistent field that isn't annotated with anything else implicitly has `@Basic`. Rarely written explicitly but important to understand.

```java
@Basic(fetch = FetchType.LAZY, optional = true)
private String description;
```

**`@Basic(fetch = FetchType.LAZY)`** on a scalar field — this is a common misconception:

JPA spec says this is supported, but **Hibernate does NOT honor `@Basic(fetch=LAZY)` without bytecode enhancement** by default. The annotation is silently ignored. The field is loaded EAGERLY regardless.

To actually get lazy scalar loading (e.g., for large `BLOB`/`CLOB` fields):
```java
// Option 1: Bytecode enhancement (compile-time or agent-based)
// hibernate-enhance-maven-plugin

// Option 2: @Lob with separate entity
// Put the large field in a separate entity, load lazily via association
```

This is a significant exam trap.

---

### `@Transient` — Exclusion from Persistence

```java
@Transient
private String calculatedDisplayName; // never persisted

// Also: Java transient keyword
transient private String cache; // also excluded
```

Both `@Transient` (JPA) and `transient` (Java keyword) exclude a field from persistence. Difference:
- `@Transient` — JPA semantic: not mapped to any column
- `transient` — Java semantic: also not serialized

Use `@Transient` for JPA context; `transient` for serialization context. Combining both is common.

---

### `@Enumerated` — The ORDINAL vs STRING Trap

```java
public enum Status { PENDING, ACTIVE, SUSPENDED, DELETED }

@Enumerated(EnumType.ORDINAL) // default — stores 0, 1, 2, 3
private Status status;

@Enumerated(EnumType.STRING)  // stores "PENDING", "ACTIVE", etc.
private Status status;
```

**`ORDINAL` is a production disaster waiting to happen:**

```java
// Original enum:
public enum Status { PENDING, ACTIVE, SUSPENDED, DELETED }
// DB values: 0=PENDING, 1=ACTIVE, 2=SUSPENDED, 3=DELETED

// Developer adds a new status IN THE MIDDLE:
public enum Status { PENDING, REVIEW, ACTIVE, SUSPENDED, DELETED }
// DB values: 0=PENDING, 1=REVIEW, 2=ACTIVE, 3=SUSPENDED, 4=DELETED

// All existing ACTIVE records in DB (value=1) now map to REVIEW
// SILENT DATA CORRUPTION — no exception, no warning
```

**Always use `@Enumerated(EnumType.STRING)`** in production. Or better: use `@Convert` with `AttributeConverter`.

**`AttributeConverter` approach** (most robust):
```java
@Converter(autoApply = true)
public class StatusConverter implements AttributeConverter<Status, String> {
    
    @Override
    public String convertToDatabaseColumn(Status status) {
        return status == null ? null : status.getDbValue();
    }
    
    @Override
    public Status convertToEntityAttribute(String dbValue) {
        return Status.fromDbValue(dbValue);
    }
}

// The enum carries its own DB representation:
public enum Status {
    PENDING("P"), ACTIVE("A"), SUSPENDED("S"), DELETED("D");
    
    private final String dbValue;
    // constructor, getter, fromDbValue()...
}
// DB stores: "P", "A", "S", "D" — compact, stable, immune to reordering
```

---

### `@Temporal` — Legacy Date Handling

```java
// Legacy (pre-Java 8):
@Temporal(TemporalType.DATE)      // maps to java.sql.Date
@Temporal(TemporalType.TIME)      // maps to java.sql.Time
@Temporal(TemporalType.TIMESTAMP) // maps to java.sql.Timestamp
private Date createdAt;

// Modern (Java 8+ — @Temporal NOT needed):
private LocalDate date;           // maps to DATE automatically
private LocalTime time;           // maps to TIME automatically
private LocalDateTime dateTime;   // maps to TIMESTAMP automatically
private Instant instant;          // maps to TIMESTAMP WITH TIME ZONE
private ZonedDateTime zoned;      // maps to TIMESTAMP WITH TIME ZONE
```

**Never use `java.util.Date` in new code.** `LocalDateTime` and `Instant` are the modern replacements. `@Temporal` is only needed for legacy `Date`/`Calendar` fields.

**`Instant` vs `LocalDateTime` trap:**
- `LocalDateTime` has NO timezone info — stores exactly what you put in, reads exactly what's in DB
- `Instant` has UTC semantics — converts to/from UTC for storage
- If your DB column is `TIMESTAMP WITH TIME ZONE` → use `Instant` or `OffsetDateTime`
- If your DB column is `TIMESTAMP` (no timezone) → use `LocalDateTime`

---

### `@Lob` — Large Object Mapping

```java
@Lob
private String description;      // maps to CLOB or TEXT
 
@Lob
private byte[] thumbnail;        // maps to BLOB or BYTEA
```

`@Lob` fields are loaded **eagerly by default** in most dialects. For truly lazy `@Lob` loading, bytecode enhancement is required.

PostgreSQL nuance: `@Lob` on `String` maps to `TEXT` in PostgreSQL (not `OID`-based CLOB as in Oracle). Behavior is dialect-specific.

---

### Naming Strategies — The Hidden Mapping Layer

Naming strategies are how Hibernate derives table/column names from class/field names when no explicit `name` is provided.

Two strategies work in sequence:

**`ImplicitNamingStrategy`** — Determines the **logical name** when no `@Table`/`@Column` is specified.

**`PhysicalNamingStrategy`** — Transforms the **logical name** into the **physical SQL name**.

```
Class name: ProductCategory
  ↓ ImplicitNamingStrategy → logical name: "ProductCategory"
  ↓ PhysicalNamingStrategy → physical name: "product_category" (snake_case)
```

Spring Boot auto-configures `SpringPhysicalNamingStrategy`:
- Converts `camelCase` to `snake_case`
- Converts uppercase to lowercase
- `ProductCategory` → `product_category`
- `createdAt` → `created_at`

This means **you never need `@Column(name="created_at")`** if your field is named `createdAt` — Spring Boot's naming strategy handles it automatically.

**Trap**: If you switch from Spring Boot's naming strategy to Hibernate's default `PhysicalNamingStrategyStandardImpl`, it keeps names as-is. `createdAt` stays `createdAt` → column name `createdAt` → may break on case-sensitive databases.

```properties
# Spring Boot default (recommended):
spring.jpa.hibernate.naming.physical-strategy=
  org.springframework.boot.orm.jpa.hibernate.SpringPhysicalNamingStrategy

# Hibernate default (no transformation):
spring.jpa.hibernate.naming.physical-strategy=
  org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
```

---

## 2️⃣ Code Examples

### Example 1 — Complete Entity with All Mapping Annotations

```java
@Entity
@Table(
    name = "products",
    schema = "inventory",
    uniqueConstraints = @UniqueConstraint(name = "uk_product_sku", columnNames = "sku"),
    indexes = @Index(name = "idx_product_name", columnList = "name")
)
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "product_seq")
    @SequenceGenerator(
        name = "product_seq",
        sequenceName = "seq_products",
        allocationSize = 50
    )
    private Long id;

    @Column(name = "product_name", nullable = false, length = 200)
    private String name;

    @Column(name = "sku", nullable = false, length = 50)
    private String sku;

    @Column(precision = 10, scale = 2)
    private BigDecimal price;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 20)
    private ProductStatus status;

    @Column(name = "description")
    @Lob
    private String description;

    @Column(name = "created_at", insertable = false, updatable = false)
    private LocalDateTime createdAt; // DB default manages this

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @Transient
    private String displayLabel; // computed, never persisted

    protected Product() {} // required by JPA spec — Hibernate uses this

    public Product(String name, String sku, BigDecimal price) {
        this.name = name;
        this.sku = sku;
        this.price = price;
        this.status = ProductStatus.ACTIVE;
    }
    
    // getters/setters...
}
```

---

### Example 2 — GenerationType Comparison and SQL Generated

```java
// IDENTITY strategy:
@Entity
public class OrderIdentity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
}

// SQL on persist():
// INSERT INTO order_identity (name) VALUES (?)
// [immediately — cannot be deferred]

// SEQUENCE strategy with allocationSize=50:
@Entity
public class OrderSequence {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
    @SequenceGenerator(name="order_seq", sequenceName="seq_orders", allocationSize=50)
    private Long id;
}

// SQL on first persist():
// SELECT nextval('seq_orders')  ← only once per 50 entities
// INSERT batch happens at flush time

// SQL on next 49 persists(): NO sequence calls — IDs from in-memory pool
// At flush: INSERT INTO order_sequence VALUES (?,?),(?,?),...  ← batch
```

---

### Example 3 — The `isNew()` Trap with Manual IDs

```java
@Entity
public class Configuration {
    @Id
    private String configKey; // natural key, manually assigned — no @GeneratedValue
    
    private String value;
}

// The save() trap:
configRepository.save(new Configuration("theme", "dark"));
// isNew() check: is id null? NO — id="theme" — not null
// → calls em.merge() instead of em.persist()
// → SELECT for "theme" → not found → INSERT
// Works, but is LESS efficient (extra SELECT)

// Fix: implement Persistable<String>
@Entity
public class Configuration implements Persistable<String> {
    @Id
    private String configKey;
    
    @Transient
    private boolean isNew = true;
    
    @Override public String getId() { return configKey; }
    @Override public boolean isNew() { return isNew; }
    
    @PostPersist @PostLoad
    void markNotNew() { this.isNew = false; }
}
// Now save() correctly calls persist() for new entities
```

---

### Example 4 — `@Enumerated` ORDINAL Disaster Demonstrated

```java
// Version 1 of enum (in production):
public enum OrderStatus { PENDING, PROCESSING, SHIPPED, DELIVERED }
// DB stores: 0, 1, 2, 3

// 3 months later, developer adds PAYMENT_FAILED after PENDING:
public enum OrderStatus { PENDING, PAYMENT_FAILED, PROCESSING, SHIPPED, DELIVERED }
// Now: 0=PENDING, 1=PAYMENT_FAILED, 2=PROCESSING, 3=SHIPPED, 4=DELIVERED

// DB has value=1 for all "PROCESSING" orders → now reads as PAYMENT_FAILED
// DB has value=2 for all "SHIPPED" orders → now reads as PROCESSING
// SILENT DATA CORRUPTION — zero exceptions thrown

// STRING version is immune:
@Enumerated(EnumType.STRING)
private OrderStatus status;
// DB stores: "PENDING", "PROCESSING", "SHIPPED", "DELIVERED"
// Adding PAYMENT_FAILED anywhere in enum → no impact on existing data
```

---

### Example 5 — AttributeConverter Internals

```java
@Converter(autoApply = true) // applies to ALL fields of type Money globally
public class MoneyConverter implements AttributeConverter<Money, Long> {
    
    // Called by Hibernate when writing to DB
    @Override
    public Long convertToDatabaseColumn(Money money) {
        return money == null ? null : money.getAmountInCents();
    }
    
    // Called by Hibernate when reading from DB  
    @Override
    public Money convertToEntityAttribute(Long cents) {
        return cents == null ? null : Money.ofCents(cents);
    }
}

@Entity
public class Invoice {
    @Id @GeneratedValue Long id;
    
    // autoApply=true means no @Convert needed here
    private Money totalAmount; // DB: BIGINT (cents), Java: Money object
    
    // To DISABLE autoApply for a specific field:
    @Convert(disableConversion = true)
    private Money rawAmount;
}
```

**Internal execution point**: `AttributeConverter` is called by Hibernate's `AttributeConverterTypeAdapter` during:
- `ResultSet` reading → `convertToEntityAttribute()`
- `PreparedStatement` binding → `convertToDatabaseColumn()`
- Dirty checking snapshot comparison — the DB column value is what's stored in the snapshot

---

### Example 6 — Naming Strategy Impact

```java
// With Spring Boot's SpringPhysicalNamingStrategy (default):

@Entity
public class ProductCategory {       // table: product_category
    @Id
    private Long id;                  // column: id
    
    private String categoryName;      // column: category_name (auto snake_case)
    private LocalDateTime createdAt;  // column: created_at (auto snake_case)
    
    // @Column(name=...) NOT needed for any of these
}

// Without Spring Boot naming strategy (Hibernate default):
// ProductCategory → table: ProductCategory (kept as-is)
// categoryName → column: categoryName (kept as-is)
// May fail on case-sensitive databases
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ**
What is the difference between `@Entity(name="Foo")` and `@Table(name="foo")`?

A) They are equivalent — both set the table name  
B) `@Entity(name)` sets the JPQL entity name; `@Table(name)` sets the SQL table name  
C) `@Table(name)` sets the JPQL entity name; `@Entity(name)` sets the SQL table name  
D) `@Entity(name)` sets both the JPQL name and SQL table name  

**Answer: B**

---

**Q2 — Select All That Apply**
Which `GenerationType` strategies support JDBC batch inserts? (Select all)

A) `AUTO` (when Hibernate selects SEQUENCE internally)  
B) `IDENTITY`  
C) `SEQUENCE`  
D) `TABLE`  

**Answer: A, C** — IDENTITY forces immediate INSERT per entity (no batching). TABLE uses pessimistic locking and is also effectively non-batchable in practice. SEQUENCE allows deferred INSERT and true batching.

---

**Q3 — Code Output Prediction**
```java
@Entity
public class Item {
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "g")
    @SequenceGenerator(name="g", sequenceName="item_seq", allocationSize=1)
    private Long id;
    
    private String name;
}

// In one transaction:
for (int i = 0; i < 5; i++) {
    Item item = new Item("Item-" + i);
    em.persist(item);
}
em.flush();
```
How many times is `item_seq` queried from the database?

A) 1  
B) 5  
C) 0 — sequences are queried at commit not persist  
D) 6 — one extra for initialization  

**Answer: B** — `allocationSize=1` means NO pooling. Every `persist()` calls `SELECT nextval('item_seq')`. 5 persists = 5 sequence calls. With `allocationSize=50`, it would be 1 call.

---

**Q4 — Compilation/Runtime Error**
```java
@Entity
public final class Customer {
    @Id @GeneratedValue Long id;
    String name;
}
```
What issue arises?

A) Compile error — `@Entity` cannot be on final class  
B) Runs fine — final classes work with JPA  
C) Runtime warning only — proxy generation is skipped  
D) Runtime error — Hibernate cannot create proxy subclass for lazy loading  

**Answer: D** — JPA spec permits it (and some providers handle it), but Hibernate cannot subclass a `final` class to create proxies. Any LAZY association targeting `Customer` will fail with proxy generation error. `Customer` itself works, but associations TO it are problematic.

---

**Q5 — SQL Prediction**
```java
@Enumerated(EnumType.ORDINAL)
private Status status; // enum: PENDING=0, ACTIVE=1, CLOSED=2

// Insert with status=ACTIVE:
product.setStatus(Status.ACTIVE);
em.persist(product);
```
What is stored in the database?

A) "ACTIVE"  
B) 1  
C) "1"  
D) 2  

**Answer: B** — `ORDINAL` stores the zero-based ordinal position. `ACTIVE` is index 1 in the enum → stores integer `1`.

---

**Q6 — Scenario-Based Trap**
```java
@Entity
public class Setting {
    @Id
    private String key; // no @GeneratedValue — manual assignment
    
    private String value;
}

settingRepository.save(new Setting("timeout", "30"));
```
What SQL is executed?

A) `INSERT INTO setting (key, value) VALUES ('timeout', '30')`  
B) `SELECT ... WHERE key='timeout'` then `INSERT` if not found  
C) `SELECT ... WHERE key='timeout'` then `UPDATE` if found, `INSERT` if not  
D) Exception — `@Id` without `@GeneratedValue` is invalid  

**Answer: B** — `SimpleJpaRepository.save()` calls `isNew()`. Key is `"timeout"` (not null) → `isNew()` returns false → calls `em.merge()`. Merge issues a SELECT first. If not found → INSERT. If found → UPDATE. This is a silent performance issue for always-new entities with natural keys.

---

**Q7 — `@Basic(fetch=LAZY)` Trap**
```java
@Entity
public class Document {
    @Id Long id;
    
    @Basic(fetch = FetchType.LAZY)
    @Lob
    private byte[] content; // large binary field
}

Document doc = em.find(Document.class, 1L);
// Is content loaded at this point?
```

A) No — `@Basic(fetch=LAZY)` defers loading until `doc.getContent()` is called  
B) Yes — `@Basic(fetch=LAZY)` is ignored without bytecode enhancement; content is loaded eagerly  
C) Null — lazy basic fields are always null until explicitly fetched  
D) Exception — `@Basic(fetch=LAZY)` is not a valid JPA annotation  

**Answer: B** — Without Hibernate bytecode enhancement plugin, `@Basic(fetch=LAZY)` is silently ignored. The content is loaded eagerly in the same SELECT. This surprises developers who add it for performance expecting lazy loading.

---

**Q8 — Column Mapping Trap**
```java
@Column(nullable = false)
private String email;

// Saving entity with null email:
user.setEmail(null);
userRepository.save(user);
```
When does the error occur?

A) Immediately when `setEmail(null)` is called — JPA validates at assignment  
B) At `save()` call — Spring Data validates nullable constraint  
C) At SQL execution time — database rejects NULL for NOT NULL column  
D) Never — `nullable=false` is DDL-only and doesn't enforce at runtime  

**Answer: C** — `nullable=false` only affects DDL (schema generation). There's no JPA-level runtime check. The `PreparedStatement` is executed with a null value, the database rejects it with a constraint violation, Hibernate wraps it in `ConstraintViolationException`. To get earlier validation: add `@NotNull` from Bean Validation.

---

**Q9 — Naming Strategy**
With Spring Boot's default `SpringPhysicalNamingStrategy`, what table name is generated for this entity?

```java
@Entity
public class UserAccount { }
```

A) `UserAccount`  
B) `user_account`  
C) `useraccount`  
D) `user_accounts`  

**Answer: B** — `SpringPhysicalNamingStrategy` converts CamelCase to snake_case and lowercases. `UserAccount` → `user_account`. Note: no pluralization (Spring does NOT add "s").

---

## 4️⃣ Trick Analysis

**`@Entity(name)` vs `@Table(name)` confusion (Q1)**:
The most frequent basic-level exam trap. `@Entity(name)` is for JPQL. `@Table(name)` is for SQL. They are completely orthogonal. You can have `@Entity(name="Foo") @Table(name="bar")` and JPQL uses `Foo`, SQL uses `bar`.

**IDENTITY breaks batching (Q2)**:
Why? Because Hibernate's batch mechanism works by accumulating `EntityInsertAction` objects in the `ActionQueue` and flushing them all at once via JDBC `addBatch()`. But IDENTITY requires knowing the generated ID immediately after INSERT (to register in identity map). This forces immediate execution, destroying the batch. This is a first-principles understanding question.

**`allocationSize=1` negates pooling (Q3)**:
The purpose of `allocationSize` is to reduce DB roundtrips. Setting it to 1 defeats this entirely and is equivalent to calling `SELECT nextval()` for every entity. Default is 50. In practice, always use a meaningful allocation size (50–100 is typical).

**`final` class proxy problem (Q4)**:
Hibernate generates proxy subclasses using CGLIB (or Byte Buddy in recent versions). You cannot subclass a final class in Java. This is a fundamental Java language constraint, not a JPA limitation. The workaround is to use bytecode enhancement instead of proxy-based lazy loading.

**`isNew()` with natural keys (Q6)**:
This is a subtler production trap. When an entity uses a manually assigned ID (natural key, UUID set by application, String key), `isNew()` returns false because the ID is not null. `save()` calls `merge()` instead of `persist()`. `merge()` does a SELECT first. For entities that are ALWAYS new (e.g., importing thousands of records), this SELECT is pure waste. Fix: implement `Persistable<ID>`.

**`@Basic(fetch=LAZY)` silent failure (Q7)**:
JPA spec says implementations MAY support lazy basic fetching. Hibernate chooses not to without bytecode enhancement. The annotation doesn't throw an error — it's just ignored. Many developers waste time wondering why their LOB fields are still loaded eagerly.

**`nullable=false` runtime behavior (Q8)**:
Bean Validation `@NotNull` = runtime validation at application layer (before SQL).
JPA `@Column(nullable=false)` = DDL hint only (after SQL, at DB layer).
These are categorically different. Many developers assume `nullable=false` provides runtime protection — it does not without Bean Validation.

---

## 5️⃣ Summary Sheet

### `@GeneratedValue` Strategy Comparison

| Strategy | DB Feature Used | SQL on persist() | Supports Batching | Recommended |
|---|---|---|---|---|
| `IDENTITY` | AUTO_INCREMENT/SERIAL | Immediate INSERT | ❌ No | ❌ Avoid for bulk |
| `SEQUENCE` | DB Sequence object | SELECT nextval() | ✅ Yes | ✅ Preferred |
| `TABLE` | Dedicated table + lock | SELECT FOR UPDATE + UPDATE | ❌ No | ❌ Avoid |
| `AUTO` | Provider decides | Depends on provider | Depends | ⚠️ Unpredictable |

### `@Column` Attribute Reference

| Attribute | Affects DDL | Affects Runtime SQL | Default |
|---|---|---|---|
| `name` | ✅ | ✅ | field name via naming strategy |
| `nullable` | ✅ | ❌ | true |
| `unique` | ✅ | ❌ | false |
| `length` | ✅ (VARCHAR only) | ❌ | 255 |
| `precision` | ✅ | ❌ | 0 |
| `scale` | ✅ | ❌ | 0 |
| `insertable` | ❌ | ✅ | true |
| `updatable` | ❌ | ✅ | true |
| `columnDefinition` | ✅ (overrides all) | ❌ | none |

### Sequence allocationSize — DB vs Hibernate Alignment

```
DB sequence INCREMENT BY must equal @SequenceGenerator allocationSize

allocationSize=50, DB INCREMENT BY=50 → IDs: 1,2,3,4,...50,51,...100 ✓
allocationSize=50, DB INCREMENT BY=1  → IDs: 1,51,101,151... (gaps!) ✗
allocationSize=1,  DB INCREMENT BY=1  → IDs: 1,2,3,4... (no gaps, no pooling) ✓
```

### `@Enumerated` Decision Matrix

| Mode | Stored As | Safe to Reorder? | Safe to Add Middle? | Recommended |
|---|---|---|---|---|
| `ORDINAL` | Integer (0,1,2...) | ❌ No | ❌ No | ❌ Never |
| `STRING` | Varchar | ✅ Yes | ✅ Yes | ✅ Yes |
| `AttributeConverter` | Custom | ✅ Yes | ✅ Yes | ✅ Best |

### Entity Class Requirements Checklist

```
✅ @Entity annotation present
✅ No-arg constructor (public or protected)
✅ Class is NOT final
✅ Methods to be proxied are NOT final
✅ Exactly one @Id field (or @EmbeddedId / @IdClass)
✅ @Id type is nullable (Long, not long) — unless using Persistable
✅ jakarta.persistence.* imports (Spring Boot 3.x)
✅ Equals/hashCode based on business key or @Id (for Set/Map correctness)
```

### Key Naming Strategy Rules (Spring Boot Default)

```
CamelCase field    → snake_case column   (categoryName → category_name)
CamelCase class    → snake_case table    (ProductCategory → product_category)
No pluralization   → (Product → product, NOT products)
Override with:     → @Column(name="...") or @Table(name="...")
```

### `isNew()` Resolution Logic in `SimpleJpaRepository.save()`

```
save(entity):
  entity implements Persistable? → use entity.isNew()
  @Id field is null?             → persist()  (new entity)
  @Id field is not null?         → merge()    (assumed existing)
  
Natural key trap:
  String/UUID @Id manually set → isNew()=false → merge() → extra SELECT
  Fix: implement Persistable<ID>
```

---
