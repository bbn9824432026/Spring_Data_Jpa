# 🏛️ TOPIC 5 — Inheritance Mapping Strategies

---

## 1️⃣ Conceptual Explanation

### The Object-Relational Impedance Mismatch — Inheritance Edition

Inheritance is the sharpest edge of the object-relational impedance mismatch. Java has first-class inheritance — `extends`, `instanceof`, polymorphism, method dispatch. Relational databases have no concept of inheritance whatsoever. Every row in every table is a flat, homogeneous structure.

JPA bridges this gap with four strategies, each making a different tradeoff between:
- **Query performance** (how many JOINs are needed)
- **Data integrity** (nullable columns, constraint enforcement)
- **Storage efficiency** (wasted space, data duplication)
- **Polymorphic query support** (can you query the parent type and get all subtypes?)
- **Schema complexity** (how many tables, how complex are the DDL and queries)

Understanding these tradeoffs at the SQL level — not just the annotation level — is what separates architects from developers.

---

### The Class Hierarchy Used Throughout This Topic

```
          Payment (abstract)
         /        |          \
CreditCard    BankTransfer   CryptoPayment
Payment       Payment        Payment
```

```java
public abstract class Payment {
    private Long id;
    private BigDecimal amount;
    private LocalDateTime createdAt;
    private PaymentStatus status;
}

public class CreditCardPayment extends Payment {
    private String cardNumber;  // last 4 digits
    private String cardHolder;
    private YearMonth expiryDate;
}

public class BankTransferPayment extends Payment {
    private String iban;
    private String bankName;
    private String reference;
}

public class CryptoPayment extends Payment {
    private String walletAddress;
    private String blockchain;
    private BigDecimal exchangeRate;
}
```

---

### Strategy 1 — `SINGLE_TABLE` (Default)

All classes in the hierarchy are mapped to **one single table**. A special **discriminator column** identifies which subtype each row represents.

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(
    name = "payment_type",
    discriminatorType = DiscriminatorType.STRING,
    length = 31
)
@DiscriminatorValue("PAYMENT") // value for Payment itself (if not abstract)
@Table(name = "payments")
public abstract class Payment {
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;
    private BigDecimal amount;
    private LocalDateTime createdAt;
    @Enumerated(EnumType.STRING)
    private PaymentStatus status;
}

@Entity
@DiscriminatorValue("CREDIT_CARD")
public class CreditCardPayment extends Payment {
    private String cardNumber;
    private String cardHolder;
    // expiryDate omitted for clarity
}

@Entity
@DiscriminatorValue("BANK_TRANSFER")
public class BankTransferPayment extends Payment {
    private String iban;
    private String bankName;
    private String reference;
}

@Entity
@DiscriminatorValue("CRYPTO")
public class CryptoPayment extends Payment {
    private String walletAddress;
    private String blockchain;
    private BigDecimal exchangeRate;
}
```

**Generated DDL:**
```sql
CREATE TABLE payments (
    id              BIGINT PRIMARY KEY,
    payment_type    VARCHAR(31) NOT NULL,  -- discriminator
    amount          DECIMAL(19,4),
    created_at      TIMESTAMP,
    status          VARCHAR(50),
    -- CreditCardPayment columns:
    card_number     VARCHAR(255),          -- NULLABLE (not present for others)
    card_holder     VARCHAR(255),          -- NULLABLE
    -- BankTransferPayment columns:
    iban            VARCHAR(255),          -- NULLABLE
    bank_name       VARCHAR(255),          -- NULLABLE
    reference       VARCHAR(255),          -- NULLABLE
    -- CryptoPayment columns:
    wallet_address  VARCHAR(255),          -- NULLABLE
    blockchain      VARCHAR(255),          -- NULLABLE
    exchange_rate   DECIMAL(19,4)          -- NULLABLE
);
```

**SQL generated for queries:**

```sql
-- Find all payments (polymorphic query):
SELECT id, payment_type, amount, created_at, status,
       card_number, card_holder, iban, bank_name, reference,
       wallet_address, blockchain, exchange_rate
FROM payments

-- Find CreditCardPayment only:
SELECT id, payment_type, amount, ..., card_number, card_holder
FROM payments
WHERE payment_type = 'CREDIT_CARD'

-- Find by parent type with filter:
SELECT ... FROM payments WHERE amount > 1000
-- No discriminator filter — returns all subtypes

-- Find specific subtype:
SELECT ... FROM payments 
WHERE payment_type IN ('CREDIT_CARD', 'BANK_TRANSFER')
-- Hibernate knows which discriminator values belong to which subtypes
```

**Internal Hibernate mechanics:**

Hibernate uses `SingleTableEntityPersister`. When loading, it reads the discriminator column first and uses it to determine which `EntityPersister` (and therefore which Java class) to instantiate. This mapping is stored in `discriminatorSQLValue → EntityPersister` lookup table built at startup.

For polymorphic queries on the parent type, Hibernate generates a single SELECT on the payments table — no filtering on discriminator. The discriminator is read per-row to determine the concrete class. This is the key performance advantage.

**Pros of SINGLE_TABLE:**
- ✅ **Fastest queries** — single table, no JOINs ever
- ✅ **Simplest schema** — one table per hierarchy
- ✅ **Polymorphic queries** — trivial, no complexity
- ✅ **Best for read-heavy workloads** with polymorphic access

**Cons of SINGLE_TABLE:**
- ❌ **All subtype columns are nullable** — cannot enforce NOT NULL constraints for subtype fields at DB level (a `CreditCardPayment`'s `card_number` cannot be NOT NULL because `BankTransferPayment` rows would violate it)
- ❌ **Wasted space** — each row has many null columns
- ❌ **Table grows wide** as more subtypes are added
- ❌ **Data integrity** is entirely enforced at application level
- ❌ **Reporting is messy** — sparse columns, hard to reason about

---

### Discriminator Column Deep Dive

**`DiscriminatorType`:**
- `STRING` (default) — stores String value (`'CREDIT_CARD'`)
- `CHAR` — stores single character (`'C'`)
- `INTEGER` — stores integer value (`1`, `2`, `3`)

**When `@DiscriminatorColumn` is omitted:** Hibernate uses default `DTYPE VARCHAR(31)`.

**When `@DiscriminatorValue` is omitted:** Hibernate uses the entity class name as the value.

**Discriminator with `NOT NULL`:** The discriminator column should always be `NOT NULL`. If a row has a null discriminator, Hibernate cannot determine the type → runtime exception.

**`@DiscriminatorFormula`** — Hibernate-specific extension allowing a SQL expression as discriminator:
```java
@DiscriminatorFormula("CASE WHEN card_number IS NOT NULL THEN 'CREDIT_CARD' " +
                      "WHEN iban IS NOT NULL THEN 'BANK_TRANSFER' ELSE 'CRYPTO' END")
```
Useful when working with legacy schemas where there's no explicit type column.

---

### Strategy 2 — `TABLE_PER_CLASS`

Each concrete class gets its **own table** containing ALL columns — both inherited and class-specific. No discriminator. No shared table.

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Payment {
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE)
    // WARNING: Must NOT use IDENTITY — see below
    private Long id;
    private BigDecimal amount;
    private LocalDateTime createdAt;
    private PaymentStatus status;
}

@Entity
@Table(name = "credit_card_payments")
public class CreditCardPayment extends Payment {
    private String cardNumber;
    private String cardHolder;
}

@Entity
@Table(name = "bank_transfer_payments")
public class BankTransferPayment extends Payment {
    private String iban;
    private String bankName;
    private String reference;
}
```

**Generated DDL:**
```sql
-- NO payments table at all

CREATE TABLE credit_card_payments (
    id          BIGINT PRIMARY KEY,  -- inherited
    amount      DECIMAL(19,4),       -- inherited
    created_at  TIMESTAMP,           -- inherited
    status      VARCHAR(50),         -- inherited
    card_number VARCHAR(255),        -- subtype-specific
    card_holder VARCHAR(255)         -- subtype-specific
);

CREATE TABLE bank_transfer_payments (
    id          BIGINT PRIMARY KEY,  -- inherited (DUPLICATED column definition)
    amount      DECIMAL(19,4),       -- inherited (DUPLICATED)
    created_at  TIMESTAMP,           -- inherited (DUPLICATED)
    status      VARCHAR(50),         -- inherited (DUPLICATED)
    iban        VARCHAR(255),        -- subtype-specific
    bank_name   VARCHAR(255),        -- subtype-specific
    reference   VARCHAR(255)         -- subtype-specific
);
```

**SQL for polymorphic queries — the UNION problem:**

```sql
-- Find all payments (polymorphic query on Payment type):
SELECT id, amount, created_at, status,
       card_number, card_holder,  -- NULLs for non-CC rows
       iban, bank_name, reference, -- NULLs for non-BT rows
       wallet_address, blockchain, exchange_rate, -- NULLs
       'CREDIT_CARD' as clazz_   -- synthetic discriminator
FROM credit_card_payments
UNION ALL
SELECT id, amount, created_at, status,
       NULL, NULL,
       iban, bank_name, reference,
       NULL, NULL, NULL,
       'BANK_TRANSFER' as clazz_
FROM bank_transfer_payments
UNION ALL
SELECT id, amount, created_at, status,
       NULL, NULL, NULL, NULL, NULL,
       wallet_address, blockchain, exchange_rate,
       'CRYPTO' as clazz_
FROM crypto_payments
```

This `UNION ALL` query is generated **every time** you query the parent `Payment` type. With many subtypes and large tables, this is a catastrophic performance problem.

**ID generation constraint with TABLE_PER_CLASS:**

Because each table has its own PK, IDs must be globally unique across all tables (otherwise `em.find(Payment.class, 5L)` is ambiguous — which table has id=5?). Therefore:
- `IDENTITY` auto-increment per-table generates overlapping IDs → **broken**
- Must use `SEQUENCE` or `TABLE` strategy with a **shared sequence** across all subtype tables
- This is a hard constraint — IDENTITY is incompatible with TABLE_PER_CLASS

**Pros of TABLE_PER_CLASS:**
- ✅ Each subtype table has clean, fully non-nullable columns (good data integrity per table)
- ✅ Subtype-specific queries are fast (single table, no JOINs, no discriminator)
- ✅ Natural schema for subtypes in isolation

**Cons of TABLE_PER_CLASS:**
- ❌ **UNION ALL for polymorphic queries** — worst-case performance
- ❌ **Column duplication** — inherited columns repeated in every table
- ❌ **FK references to abstract parent** are impossible or awkward (what table does `FK → payments` point to?)
- ❌ Incompatible with IDENTITY generator
- ❌ Generally **avoid in production** for polymorphic access patterns

---

### Strategy 3 — `JOINED` (Normalized)

Each class in the hierarchy gets its own table containing only its **own fields**. The subtype table has a FK to the parent table sharing the same PK.

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@Table(name = "payments")
public abstract class Payment {
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;
    private BigDecimal amount;
    private LocalDateTime createdAt;
    private PaymentStatus status;
}

@Entity
@Table(name = "credit_card_payments")
@PrimaryKeyJoinColumn(name = "payment_id") // FK to payments.id
public class CreditCardPayment extends Payment {
    private String cardNumber;
    private String cardHolder;
}

@Entity
@Table(name = "bank_transfer_payments")
@PrimaryKeyJoinColumn(name = "payment_id")
public class BankTransferPayment extends Payment {
    private String iban;
    private String bankName;
    private String reference;
}
```

**Generated DDL:**
```sql
CREATE TABLE payments (
    id         BIGINT PRIMARY KEY,
    amount     DECIMAL(19,4) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    status     VARCHAR(50) NOT NULL
    -- No discriminator column needed (but optional — can add for performance)
);

CREATE TABLE credit_card_payments (
    payment_id  BIGINT PRIMARY KEY,           -- also FK
    card_number VARCHAR(255) NOT NULL,         -- CAN be NOT NULL now!
    card_holder VARCHAR(255) NOT NULL,
    FOREIGN KEY (payment_id) REFERENCES payments(id)
);

CREATE TABLE bank_transfer_payments (
    payment_id BIGINT PRIMARY KEY,
    iban       VARCHAR(255) NOT NULL,
    bank_name  VARCHAR(255),
    reference  VARCHAR(255),
    FOREIGN KEY (payment_id) REFERENCES payments(id)
);
```

**SQL for queries:**

```sql
-- Find all CreditCardPayments:
SELECT p.id, p.amount, p.created_at, p.status,
       cc.card_number, cc.card_holder
FROM payments p
INNER JOIN credit_card_payments cc ON p.id = cc.payment_id

-- Polymorphic query (all Payments):
SELECT p.id, p.amount, p.created_at, p.status,
       cc.card_number, cc.card_holder,
       bt.iban, bt.bank_name, bt.reference,
       cr.wallet_address, cr.blockchain, cr.exchange_rate
FROM payments p
LEFT OUTER JOIN credit_card_payments cc ON p.id = cc.payment_id
LEFT OUTER JOIN bank_transfer_payments bt ON p.id = bt.payment_id
LEFT OUTER JOIN crypto_payments cr ON p.id = cr.payment_id

-- INSERT for CreditCardPayment:
INSERT INTO payments (amount, created_at, status) VALUES (?, ?, ?)
INSERT INTO credit_card_payments (payment_id, card_number, card_holder) VALUES (?, ?, ?)
-- Two INSERT statements per persist()!
```

**Internal Hibernate mechanics:**

`JoinedSubclassEntityPersister` manages this strategy. For polymorphic queries, it generates `LEFT OUTER JOIN` to each subtype table. Hibernate reads the result and uses the presence of non-null subtype columns (or an optional discriminator) to determine the concrete type.

For INSERT: two statements per entity (parent table + subtype table). For DELETE: two deletes. This doubles write overhead.

**Optional discriminator in JOINED:**

Hibernate supports adding a discriminator column to the parent table even with JOINED strategy, which helps it avoid probing all JOIN columns to determine the type:

```java
@DiscriminatorColumn(name = "payment_type")
// Allows: WHERE payment_type = 'CREDIT_CARD' instead of checking JOINs
```

**Pros of JOINED:**
- ✅ **Best data integrity** — subtype columns CAN be NOT NULL
- ✅ **Normalized schema** — no data duplication
- ✅ **FK references to parent** work cleanly
- ✅ **Adding subtypes** doesn't alter existing tables

**Cons of JOINED:**
- ❌ **JOINs on every query** — including single-entity loads
- ❌ **Two writes per entity** (parent + subtype table)
- ❌ **Polymorphic queries** require `LEFT OUTER JOIN` to all subtype tables
- ❌ Performance degrades as hierarchy depth grows
- ❌ Complex SQL — harder to debug and optimize

---

### Strategy 4 — `@MappedSuperclass`

**Not an inheritance mapping strategy in the JPA sense.** `@MappedSuperclass` is a way to share field definitions and mapping metadata across entity classes without creating any inheritance relationship in the database or in the JPA type system.

```java
@MappedSuperclass
public abstract class BaseEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;
    
    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @Version
    private Long version;
    
    @PrePersist
    protected void prePersist() { this.createdAt = LocalDateTime.now(); }
    
    @PreUpdate
    protected void preUpdate() { this.updatedAt = LocalDateTime.now(); }
}

@Entity
@Table(name = "customers")
public class Customer extends BaseEntity {
    private String name;
    private String email;
    // inherits id, createdAt, updatedAt, version from BaseEntity
}

@Entity
@Table(name = "products")
public class Product extends BaseEntity {
    private String name;
    private BigDecimal price;
    // inherits id, createdAt, updatedAt, version from BaseEntity
}
```

**Generated DDL:**
```sql
-- NO base_entity table — @MappedSuperclass has no table

CREATE TABLE customers (
    id         BIGINT PRIMARY KEY,     -- from BaseEntity
    created_at TIMESTAMP,              -- from BaseEntity
    updated_at TIMESTAMP,              -- from BaseEntity
    version    BIGINT,                 -- from BaseEntity
    name       VARCHAR(255),
    email      VARCHAR(255)
);

CREATE TABLE products (
    id         BIGINT PRIMARY KEY,     -- from BaseEntity (duplicated definition)
    created_at TIMESTAMP,              -- from BaseEntity (duplicated)
    updated_at TIMESTAMP,              -- from BaseEntity (duplicated)
    version    BIGINT,                 -- from BaseEntity (duplicated)
    name       VARCHAR(255),
    price      DECIMAL(19,4)
);
```

**Critical distinctions from inheritance strategies:**

| Aspect | `@MappedSuperclass` | `@Inheritance` strategies |
|---|---|---|
| Is a JPA managed type? | ❌ No | ✅ Yes |
| Has its own table? | ❌ No | Depends on strategy |
| Can be used in JPQL `FROM`? | ❌ No | ✅ Yes |
| `em.find(BaseEntity.class, id)` | ❌ Throws exception | ✅ Works (for `@Inheritance`) |
| Polymorphic associations | ❌ Cannot target | ✅ Can target |
| Purpose | Code reuse / DRY | Type hierarchy modeling |

```java
// THIS IS ILLEGAL — @MappedSuperclass is not a managed type:
em.find(BaseEntity.class, 1L); // IllegalArgumentException

// This query is ILLEGAL:
em.createQuery("SELECT b FROM BaseEntity b"); // not a queryable type

// These are LEGAL with @Inheritance:
em.find(Payment.class, 1L); // returns appropriate subtype
em.createQuery("SELECT p FROM Payment p"); // polymorphic query
```

**`@MappedSuperclass` with `@AttributeOverride`:**

Subclasses can override column names from the superclass:
```java
@Entity
@Table(name = "premium_customers")
@AttributeOverride(name = "createdAt", column = @Column(name = "registered_at"))
public class PremiumCustomer extends BaseEntity {
    private String tier;
}
```

---

### Polymorphic Query Internals — How Hibernate Resolves Types

When you execute `SELECT p FROM Payment p` with SINGLE_TABLE:
1. Single SQL SELECT on payments table
2. Per-row: read discriminator column value
3. Look up discriminator → `EntityPersister` map
4. Instantiate correct Java class
5. Hydrate fields (subtype-specific columns may be null for other types)

When you execute `SELECT p FROM Payment p` with JOINED:
1. SQL with LEFT OUTER JOIN to all subtype tables
2. Per-row: check which JOIN columns are non-null (or read discriminator if present)
3. Instantiate correct Java class
4. Hydrate parent fields from `payments` columns, subtype fields from subtype JOIN

When you execute `SELECT p FROM Payment p` with TABLE_PER_CLASS:
1. UNION ALL across all subtype tables
2. Each sub-query adds synthetic `clazz_` column
3. Per-row: read `clazz_` column to determine type
4. Instantiate correct Java class
5. Hydrate fields

---

### `instanceof` in JPQL — `TREAT` and `TYPE`

JPA provides operators for working with types in JPQL:

```java
// TYPE() — filter by concrete type:
"SELECT p FROM Payment p WHERE TYPE(p) = CreditCardPayment"
// SINGLE_TABLE: WHERE payment_type = 'CREDIT_CARD'
// JOINED: WHERE EXISTS (SELECT 1 FROM credit_card_payments WHERE payment_id = p.id)

// TREAT() — downcast in JPQL:
"SELECT TREAT(p AS CreditCardPayment).cardNumber FROM Payment p 
 WHERE TYPE(p) = CreditCardPayment"
// Allows accessing subtype fields in a polymorphic query

// instanceof equivalent:
"SELECT p FROM Payment p WHERE p INSTANCEOF CreditCardPayment"
// Hibernate-specific (not standard JPQL)
```

---

### Inheritance Strategy — Enterprise Performance Decision Matrix

```
Hierarchy size:   small (2-4 subtypes)  →  any strategy viable
                  large (5+ subtypes)   →  avoid TABLE_PER_CLASS

Query pattern:    mostly polymorphic    →  SINGLE_TABLE
                  mostly subtype-specific → JOINED or TABLE_PER_CLASS
                  mixed                →  SINGLE_TABLE (usually wins)

Data integrity:   strict NOT NULL needed → JOINED
                  application-enforced OK → SINGLE_TABLE

Write frequency:  high insert/update    →  SINGLE_TABLE (one write)
                  moderate             →  JOINED (two writes tolerable)

Schema ownership: greenfield           →  any
                  legacy schema        →  often TABLE_PER_CLASS or custom

Reporting:        BI/SQL reporting     →  JOINED (cleaner schema)
                  ORM-only access      →  SINGLE_TABLE (simpler)
```

---

## 2️⃣ Code Examples

### Example 1 — SINGLE_TABLE Complete Setup with SQL Trace

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "dtype", discriminatorType = DiscriminatorType.STRING, length = 31)
@Table(name = "notifications")
public abstract class Notification {
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE) Long id;
    private String recipient;
    private String subject;
    private LocalDateTime sentAt;
    private boolean delivered;
}

@Entity @DiscriminatorValue("EMAIL")
public class EmailNotification extends Notification {
    private String emailAddress;
    private String htmlBody;
}

@Entity @DiscriminatorValue("SMS")
public class SmsNotification extends Notification {
    private String phoneNumber;
    private String messageText; // 160 char limit
}

@Entity @DiscriminatorValue("PUSH")
public class PushNotification extends Notification {
    private String deviceToken;
    private String payload;
    private String clickAction;
}

// Repository:
public interface NotificationRepository 
    extends JpaRepository<Notification, Long> {
    
    // Polymorphic — returns all subtypes:
    List<Notification> findByRecipient(String recipient);
    // SQL: SELECT * FROM notifications WHERE recipient = ?
    
    // Subtype-specific:
    List<EmailNotification> findByEmailAddress(String email);
    // SQL: SELECT * FROM notifications 
    //      WHERE dtype='EMAIL' AND email_address = ?
}

// Usage:
@Transactional
public void demo() {
    EmailNotification email = new EmailNotification();
    email.setRecipient("alice");
    email.setEmailAddress("alice@example.com");
    email.setHtmlBody("<h1>Hello</h1>");
    em.persist(email);
    // SQL: INSERT INTO notifications 
    //      (id, dtype, recipient, subject, sent_at, delivered, 
    //       email_address, html_body, 
    //       phone_number, message_text,   ← NULL
    //       device_token, payload, click_action)  ← NULL
    //      VALUES (1, 'EMAIL', 'alice', ...)
}
```

---

### Example 2 — JOINED Strategy with Dual INSERT/DELETE

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@Table(name = "vehicles")
public abstract class Vehicle {
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE) Long id;
    private String licensePlate;
    private String manufacturer;
    private int year;
}

@Entity
@Table(name = "cars")
@PrimaryKeyJoinColumn(name = "vehicle_id")
public class Car extends Vehicle {
    private int numberOfDoors;
    private String fuelType;
}

@Entity
@Table(name = "motorcycles")
@PrimaryKeyJoinColumn(name = "vehicle_id")
public class Motorcycle extends Vehicle {
    private String engineType;
    private boolean hasSidecar;
}

@Transactional
public void persistJoined() {
    Car car = new Car();
    car.setLicensePlate("ABC-123");
    car.setManufacturer("Toyota");
    car.setYear(2023);
    car.setNumberOfDoors(4);
    car.setFuelType("Hybrid");
    
    em.persist(car);
    em.flush();
    
    // SQL executed:
    // INSERT INTO vehicles (license_plate, manufacturer, year) 
    //   VALUES ('ABC-123', 'Toyota', 2023)      ← parent table
    // INSERT INTO cars (vehicle_id, number_of_doors, fuel_type) 
    //   VALUES (1, 4, 'Hybrid')                 ← subtype table
    
    em.remove(car);
    em.flush();
    
    // SQL executed:
    // DELETE FROM cars WHERE vehicle_id = 1     ← child first (FK constraint)
    // DELETE FROM vehicles WHERE id = 1         ← parent second
}

@Transactional
public void queryJoined() {
    // Subtype query:
    List<Car> cars = em.createQuery("SELECT c FROM Car c", Car.class)
                       .getResultList();
    // SQL:
    // SELECT v.id, v.license_plate, v.manufacturer, v.year,
    //        c.number_of_doors, c.fuel_type
    // FROM vehicles v
    // INNER JOIN cars c ON v.id = c.vehicle_id
    
    // Polymorphic query:
    List<Vehicle> all = em.createQuery("SELECT v FROM Vehicle v", Vehicle.class)
                          .getResultList();
    // SQL:
    // SELECT v.id, v.license_plate, v.manufacturer, v.year,
    //        c.number_of_doors, c.fuel_type,
    //        m.engine_type, m.has_sidecar
    // FROM vehicles v
    // LEFT OUTER JOIN cars c ON v.id = c.vehicle_id
    // LEFT OUTER JOIN motorcycles m ON v.id = m.vehicle_id
}
```

---

### Example 3 — TABLE_PER_CLASS and the UNION Query

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Shape {
    // MUST use SEQUENCE — IDENTITY is incompatible
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "shape_seq")
    @SequenceGenerator(name = "shape_seq", sequenceName = "seq_shapes", allocationSize = 50)
    private Long id;
    private String color;
    private double area;
}

@Entity @Table(name = "circles")
public class Circle extends Shape {
    private double radius;
}

@Entity @Table(name = "rectangles")
public class Rectangle extends Shape {
    private double width;
    private double height;
}

// Polymorphic query generates UNION:
List<Shape> shapes = em.createQuery("SELECT s FROM Shape s", Shape.class)
                        .getResultList();
// SQL:
// SELECT id, color, area, radius, NULL as width, NULL as height, 
//        1 as clazz_
// FROM circles
// UNION ALL
// SELECT id, color, area, NULL as radius, width, height,
//        2 as clazz_
// FROM rectangles

// Subtype query — efficient:
List<Circle> circles = em.createQuery("SELECT c FROM Circle c", Circle.class)
                          .getResultList();
// SQL: SELECT id, color, area, radius FROM circles
// No UNION — single table scan
```

---

### Example 4 — `@MappedSuperclass` for DRY Auditing

```java
@MappedSuperclass
public abstract class AuditableEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;
    
    @Column(name = "created_at", updatable = false, nullable = false)
    private LocalDateTime createdAt;
    
    @Column(name = "created_by", updatable = false, length = 100)
    private String createdBy;
    
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;
    
    @Column(name = "updated_by", length = 100)
    private String updatedBy;
    
    @Version
    @Column(name = "version", nullable = false)
    private Long version;
    
    @PrePersist
    protected void onPersist() {
        this.createdAt = LocalDateTime.now();
        this.createdBy = SecurityContext.currentUser(); // example
    }
    
    @PreUpdate
    protected void onUpdate() {
        this.updatedAt = LocalDateTime.now();
        this.updatedBy = SecurityContext.currentUser();
    }
    
    // No @Table, no persistence — just metadata inheritance
}

@Entity
@Table(name = "orders")
public class Order extends AuditableEntity {
    // Gets: id, created_at, created_by, updated_at, updated_by, version
    private BigDecimal total;
    private String customerId;
}

@Entity
@Table(name = "products")
@AttributeOverride(name = "createdAt", 
                   column = @Column(name = "listed_at", updatable = false))
public class Product extends AuditableEntity {
    // Gets: id, listed_at (overridden), created_by, updated_at, updated_by, version
    private String name;
    private BigDecimal price;
}
```

---

### Example 5 — Strategy Comparison: Same Query, Different SQL

```java
// Same JPQL: "SELECT p FROM Payment p WHERE p.amount > 1000"

// SINGLE_TABLE SQL:
SELECT id, dtype, amount, created_at, status,
       card_number, card_holder, iban, bank_name, 
       reference, wallet_address, blockchain, exchange_rate
FROM payments
WHERE amount > 1000
-- ✅ Single table scan, no JOINs, fastest

// JOINED SQL:
SELECT p.id, p.amount, p.created_at, p.status,
       cc.card_number, cc.card_holder,
       bt.iban, bt.bank_name, bt.reference,
       cr.wallet_address, cr.blockchain, cr.exchange_rate
FROM payments p
LEFT OUTER JOIN credit_card_payments cc ON p.id = cc.payment_id
LEFT OUTER JOIN bank_transfer_payments bt ON p.id = bt.payment_id
LEFT OUTER JOIN crypto_payments cr ON p.id = cr.payment_id
WHERE p.amount > 1000
-- ⚠️ 3 LEFT JOINs — moderate cost

// TABLE_PER_CLASS SQL:
SELECT id, amount, created_at, status,
       card_number, card_holder, NULL, NULL, NULL, NULL, NULL, NULL, 'CC'
FROM credit_card_payments WHERE amount > 1000
UNION ALL
SELECT id, amount, created_at, status,
       NULL, NULL, iban, bank_name, reference, NULL, NULL, NULL, 'BT'
FROM bank_transfer_payments WHERE amount > 1000
UNION ALL
SELECT id, amount, created_at, status,
       NULL, NULL, NULL, NULL, NULL, wallet_address, blockchain, exchange_rate, 'CR'
FROM crypto_payments WHERE amount > 1000
-- ❌ 3 table scans + UNION ALL — most expensive for polymorphic queries
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ**
Which inheritance strategy stores all subclass data in one table with a type discriminator column?

A) `JOINED`  
B) `TABLE_PER_CLASS`  
C) `SINGLE_TABLE`  
D) `@MappedSuperclass`  

**Answer: C**

---

**Q2 — Select All That Apply**
Which statements about `TABLE_PER_CLASS` inheritance are TRUE? (Select all)

A) Each concrete class has its own table  
B) Polymorphic queries require `UNION ALL`  
C) `GenerationType.IDENTITY` is compatible with this strategy  
D) Inherited columns are duplicated in each subclass table  
E) `@DiscriminatorColumn` is required  

**Answer: A, B, D**
- C is false: IDENTITY generates non-unique IDs across tables → broken
- E is false: TABLE_PER_CLASS has no discriminator column in the schema; a synthetic discriminator is added to UNION queries by Hibernate internally

---

**Q3 — SQL Prediction**
```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@Table(name = "animals")
public abstract class Animal { @Id @GeneratedValue Long id; String name; }

@Entity @Table(name = "dogs")
@PrimaryKeyJoinColumn(name = "animal_id")
public class Dog extends Animal { String breed; }

em.persist(new Dog("Rex", "Labrador"));
```
How many INSERT statements are executed?

A) 1 — JOINED uses one INSERT  
B) 2 — one for `animals`, one for `dogs`  
C) 3 — one for each hierarchy level plus discriminator  
D) 0 — insert is deferred until explicit flush  

**Answer: B** — JOINED always uses two INSERTs per entity: parent table first, then subtype table. (D is partially correct — deferred until flush, but the question asks about count.)

---

**Q4 — MCQ**
What happens when you call `em.find(BaseEntity.class, 1L)` where `BaseEntity` is annotated with `@MappedSuperclass`?

A) Returns the correct subtype entity  
B) Returns a `BaseEntity` instance with only shared fields populated  
C) Throws `IllegalArgumentException` — `@MappedSuperclass` is not a managed type  
D) Returns null — `@MappedSuperclass` entities are not stored  

**Answer: C** — `@MappedSuperclass` classes are NOT JPA managed types. They cannot be used in `em.find()`, JPQL, or as association targets.

---

**Q5 — Discriminator Trap**
```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@Table(name = "events")
public class Event {
    @Id @GeneratedValue Long id;
    String title;
}

@Entity
public class ConferenceEvent extends Event {
    String venue;
}
```
No `@DiscriminatorColumn` or `@DiscriminatorValue` is specified. What discriminator value does Hibernate use for `ConferenceEvent`?

A) `"ConferenceEvent"` — the simple class name  
B) `"com.example.ConferenceEvent"` — the fully qualified class name  
C) `"1"` — an auto-generated integer  
D) `null` — no discriminator is used  

**Answer: A** — When `@DiscriminatorValue` is omitted, Hibernate defaults to the **simple class name** of the entity (`"ConferenceEvent"`). The default `@DiscriminatorColumn` name is `"DTYPE"` with `VARCHAR(31)` length.

---

**Q6 — Architecture Question**
A financial system has a `Transaction` hierarchy: `DebitTransaction`, `CreditTransaction`, `TransferTransaction`. Requirements:
- Transaction records are NEVER deleted
- Most queries are `SELECT * FROM transactions WHERE account_id = ?` (polymorphic)  
- Regulatory reports require `SELECT count(*) FROM debit_transactions` (subtype specific)
- `amount` must be NOT NULL enforced at DB level

Which strategy is the best choice?

A) `SINGLE_TABLE` — fastest polymorphic queries but `amount` cannot be NOT NULL  
B) `JOINED` — normalized schema, subtype columns can be NOT NULL, supports both query patterns  
C) `TABLE_PER_CLASS` — best for subtype-specific queries but UNION for polymorphic  
D) `@MappedSuperclass` — cleanest approach for shared fields  

**Answer: B** — `amount` is on the parent class (`Transaction`) and can be NOT NULL in JOINED strategy (the `transactions` table has `amount NOT NULL`). Both polymorphic (JOIN) and subtype-specific (subtype table scan) queries are supported. `SINGLE_TABLE` cannot enforce NOT NULL on inherited columns when they are actually nullable for different subtypes. `@MappedSuperclass` doesn't support polymorphic queries.

---

**Q7 — JPQL Validity**
```java
@MappedSuperclass
public abstract class BaseEvent { @Id Long id; String name; }

@Entity public class SportEvent extends BaseEvent { String sport; }
@Entity public class CulturalEvent extends BaseEvent { String category; }
```

Which JPQL query is valid?

A) `SELECT b FROM BaseEvent b`  
B) `SELECT s FROM SportEvent s`  
C) `SELECT b FROM BaseEvent b WHERE TYPE(b) = SportEvent`  
D) `em.find(BaseEvent.class, 1L)`  

**Answer: B** — Only `SportEvent` and `CulturalEvent` are managed types. `BaseEvent` cannot appear in JPQL `FROM` clause or `em.find()`. `TYPE()` operator also cannot be used with `@MappedSuperclass` base.

---

**Q8 — Code Output Prediction**
```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type")
public class Vehicle { @Id @GeneratedValue Long id; String color; }

@Entity @DiscriminatorValue("CAR")
public class Car extends Vehicle { int doors; }

@Entity @DiscriminatorValue("BIKE")  
public class Bike extends Vehicle { boolean electric; }

// Repository method:
List<Vehicle> all = vehicleRepo.findAll();
```

How many SQL statements are executed?

A) 1 per entity instance in the result  
B) 2 — one for `Car`, one for `Bike`  
C) 1 — single SELECT on the `vehicles` table  
D) 3 — one per hierarchy level  

**Answer: C** — `SINGLE_TABLE` uses one table. `findAll()` on the parent repository generates one SELECT from the single table. Hibernate uses the discriminator column per row to determine the concrete type. Zero JOINs, zero UNIONs.

---

## 4️⃣ Trick Analysis

**The IDENTITY generator incompatibility with TABLE_PER_CLASS (Q2)**:
This is a deep mechanics trap. IDENTITY relies on database auto-increment per table. `circles` table generates id=1, `rectangles` table also generates id=1 independently. Now `em.find(Shape.class, 1L)` is ambiguous — which table? Hibernate has no way to know. SEQUENCE with a shared sequence across all tables ensures global uniqueness.

**`@MappedSuperclass` vs `@Inheritance` root (Q4, Q7)**:
The critical distinction: `@MappedSuperclass` is invisible to the JPA type system at runtime. It provides only **compile-time code reuse**. The JPA metamodel has no entry for it. No `em.find()`, no JPQL, no polymorphic associations. `@Inheritance` root entities ARE in the JPA metamodel and support all of these.

**Default discriminator value = simple class name (Q5)**:
Developers often forget that omitting `@DiscriminatorValue` doesn't mean no discriminator — Hibernate uses the simple class name. This causes subtle bugs when classes are renamed or moved to different packages (the class name changes, DB has old values → Hibernate can't resolve the type → exception on load). Always specify `@DiscriminatorValue` explicitly.

**JOINED not-null capability (Q6)**:
In `SINGLE_TABLE`, all subtype-specific columns exist in the same table. A row for `BankTransferPayment` has `NULL` for `card_number`. Therefore `card_number` CANNOT be `NOT NULL`. In `JOINED`, `credit_card_payments.card_number` only exists for `CreditCardPayment` rows — it CAN be `NOT NULL`. This is the critical data integrity advantage of JOINED.

**SINGLE_TABLE findAll generates ONE SQL (Q8)**:
Many developers assume a polymorphic query on a hierarchy generates multiple SQL statements. For `SINGLE_TABLE`, it never does — always one query, one table. The polymorphic resolution is done at the Java level by reading the discriminator column. This is why SINGLE_TABLE has the best read performance for polymorphic queries.

**`@DiscriminatorFormula` — rarely known, sometimes tested**:
When inheriting a legacy schema with no type column, `@DiscriminatorFormula` allows a SQL expression to derive the discriminator value. Knowing it exists and what it does is enough for exam purposes.

---

## 5️⃣ Summary Sheet

### Strategy Comparison Master Table

| Aspect | SINGLE_TABLE | JOINED | TABLE_PER_CLASS | @MappedSuperclass |
|---|---|---|---|---|
| Tables created | 1 | 1 per class | 1 per concrete class | 0 |
| Discriminator column | ✅ Required | Optional | ❌ None | N/A |
| Polymorphic query SQL | Single SELECT | SELECT + LEFT JOINs | UNION ALL | ❌ Not possible |
| Subtype NOT NULL | ❌ Impossible | ✅ Possible | ✅ Possible | N/A |
| INSERTs per entity | 1 | 2 (parent+child) | 1 | N/A |
| IDENTITY compatible | ✅ Yes | ✅ Yes | ❌ No | ✅ Yes |
| FK to parent type | ✅ Yes | ✅ Yes | ❌ Ambiguous | ❌ No |
| JPA managed type | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No |
| `em.find(Parent.class)` | ✅ Works | ✅ Works | ✅ Works | ❌ Exception |
| JPQL on parent | ✅ Works | ✅ Works | ✅ Works (UNION) | ❌ Exception |
| Column duplication | ❌ None | ❌ None | ✅ Yes | ✅ Yes |
| Best for | Read-heavy, polymorphic | Integrity, normalized | Rare subtype queries | Code reuse only |

### SQL Generated Per Operation

```
SINGLE_TABLE:
  persist()  → 1 INSERT (all cols, NULLs for other subtypes)
  find()     → 1 SELECT (single table)
  remove()   → 1 DELETE
  findAll()  → 1 SELECT (no discriminator filter)

JOINED:
  persist()  → 2 INSERTs (parent table, then subtype table)
  find()     → 1 SELECT + INNER JOIN (parent + subtype)
  remove()   → 2 DELETEs (child table first, then parent)
  findAll()  → 1 SELECT + N LEFT JOINs (N = number of subtypes)

TABLE_PER_CLASS:
  persist()  → 1 INSERT (into subtype table only)
  find()     → 1 SELECT (from correct subtype table)
  remove()   → 1 DELETE (from subtype table only)
  findAll()  → UNION ALL across all subtype tables

@MappedSuperclass:
  All operations → operates on concrete entity tables only
  No polymorphic operations possible
```

### Key Rules to Memorize

```
1. SINGLE_TABLE = one table, discriminator, nullable subtype columns
2. JOINED = N+1 tables, no discriminator needed, 2 writes per entity
3. TABLE_PER_CLASS = N tables (concrete only), UNION ALL for polymorphic, NO IDENTITY
4. @MappedSuperclass = 0 tables, NOT a JPA managed type, code reuse only
5. Default @DiscriminatorColumn name = "DTYPE", length = 31
6. Default @DiscriminatorValue = simple class name
7. JOINED subtype columns CAN be NOT NULL (key integrity advantage)
8. TABLE_PER_CLASS requires SEQUENCE generator (shared across tables)
9. @MappedSuperclass cannot be used in em.find(), JPQL FROM, or associations
10. SINGLE_TABLE is almost always fastest for polymorphic read queries
```

### Decision Flowchart

```
Need polymorphic queries?
  NO  → TABLE_PER_CLASS or @MappedSuperclass
  YES → Continue:
        Need NOT NULL on subtype columns?
          YES → JOINED
          NO  → SINGLE_TABLE (default choice)
                Exception: hierarchy too wide (10+ subtypes, 50+ columns)?
                → Consider JOINED for schema manageability
```

---
