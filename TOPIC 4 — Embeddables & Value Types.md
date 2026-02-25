# 🏛️ TOPIC 4 — Embeddables & Value Types

---

## 1️⃣ Conceptual Explanation

### Value Types vs Entities — The Fundamental Semantic Distinction

Before diving into `@Embeddable`, you must understand the **philosophical difference** between a Value Type and an Entity. This distinction drives every design decision in domain modeling and has deep implications for how Hibernate maps, tracks, and manages objects.

**Entity semantics:**
- Has its own **identity** — identified by primary key
- Has its own **lifecycle** — can exist independently
- **Shared** — multiple objects can reference the same entity instance
- Identity survives across transactions — the same DB row is always the same entity
- Example: `Customer`, `Order`, `Product`

**Value Type semantics:**
- Has **no independent identity** — identified only by the owning entity
- **Lifecycle is owned** by the containing entity — lives and dies with it
- **Not shared** — a value type instance belongs to exactly one owner
- Equality defined by **value** (like `String`, `Integer`, `Money`)
- Example: `Address`, `Money`, `DateRange`, `Name`

This maps directly to Domain-Driven Design's distinction between **Entities** and **Value Objects**. JPA's embeddable mechanism is the technical implementation of this DDD concept.

```
Entity:     Customer ──has-a──► Order (Order has its own identity, lifecycle)
Value Type: Customer ──has-a──► Address (Address has no meaning outside Customer)
```

**Why does this matter for mapping?**

Value types are stored **in the same table row** as their owning entity. There is no separate table, no foreign key, no JOIN. The `Address` fields (`street`, `city`, `zip`) become columns in the `customers` table directly.

```sql
-- Entity relationship (two tables, FK, JOIN needed):
SELECT c.name, o.total FROM customers c JOIN orders o ON o.customer_id = c.id

-- Value type (one table, no JOIN):
SELECT c.name, c.street, c.city, c.zip FROM customers c
```

This is both the power and the constraint of embeddables.

---

### `@Embeddable` — Declaring a Value Type

`@Embeddable` marks a class as a value type that can be embedded inside an entity. The embeddable class itself has no `@Id`, no table, no independent existence.

```java
@Embeddable
public class Address {
    
    @Column(name = "street", nullable = false, length = 200)
    private String street;
    
    @Column(name = "city", nullable = false, length = 100)
    private String city;
    
    @Column(name = "state", length = 50)
    private String state;
    
    @Column(name = "zip_code", length = 20)
    private String zipCode;
    
    @Column(name = "country", length = 2)
    private String countryCode;
    
    protected Address() {} // required by Hibernate for instantiation
    
    public Address(String street, String city, String state, 
                   String zipCode, String countryCode) {
        this.street = street;
        this.city = city;
        this.state = state;
        this.zipCode = zipCode;
        this.countryCode = countryCode;
    }
    
    // Value object contract: equality by VALUE, not reference
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Address)) return false;
        Address that = (Address) o;
        return Objects.equals(street, that.street) &&
               Objects.equals(city, that.city) &&
               Objects.equals(zipCode, that.zipCode) &&
               Objects.equals(countryCode, that.countryCode);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(street, city, zipCode, countryCode);
    }
}
```

**Internal Hibernate behavior:**

When Hibernate builds its metamodel for an `@Embeddable` class, it creates a `ComponentType` (the internal representation of value types). Unlike `EntityType`, `ComponentType` has:
- No `EntityKey` — no identity map entry
- No independent dirty checking — dirty checking is done through the owning entity
- No proxy — embeddable instances are always fully loaded (no lazy loading possible at the component level without bytecode enhancement)
- Column mapping is **flattened** into the owning entity's `TableGroup`

**Snapshot for dirty checking:**
The embeddable's fields are included in the owning entity's snapshot array. If `Address.city` changes, Hibernate's dirty check on the `Customer` entity detects it, because the snapshot includes all embedded fields as a flat array.

---

### `@Embedded` — Using a Value Type in an Entity

```java
@Entity
@Table(name = "customers")
public class Customer {
    
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Embedded // optional — presence of @Embeddable type is enough
    private Address homeAddress;
    
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street",      column = @Column(name = "work_street")),
        @AttributeOverride(name = "city",        column = @Column(name = "work_city")),
        @AttributeOverride(name = "state",       column = @Column(name = "work_state")),
        @AttributeOverride(name = "zipCode",     column = @Column(name = "work_zip_code")),
        @AttributeOverride(name = "countryCode", column = @Column(name = "work_country"))
    })
    private Address workAddress;
}
```

**Resulting SQL table:**
```sql
CREATE TABLE customers (
    id           BIGINT NOT NULL PRIMARY KEY,
    name         VARCHAR(255) NOT NULL,
    -- homeAddress fields (default column names from @Embeddable):
    street       VARCHAR(200) NOT NULL,
    city         VARCHAR(100) NOT NULL,
    state        VARCHAR(50),
    zip_code     VARCHAR(20),
    country      VARCHAR(2),
    -- workAddress fields (overridden column names):
    work_street  VARCHAR(200),
    work_city    VARCHAR(100),
    work_state   VARCHAR(50),
    work_zip_code VARCHAR(20),
    work_country VARCHAR(2)
);
```

The entire structure lives in one table. No JOIN required.

---

### `@AttributeOverride` — The Column Name Conflict Resolver

When the **same embeddable type is embedded twice** in the same entity (like `homeAddress` and `workAddress` above), the column names from `@Embeddable` would clash — both would try to create a column named `street`.

`@AttributeOverrides` + `@AttributeOverride` resolves this by remapping column names per usage:

```java
@AttributeOverride(
    name = "street",                    // field name in @Embeddable class
    column = @Column(name = "work_street") // override to this column name
)
```

**`name` refers to the field name in the embeddable class, not the column name.** This is a common mistake — if your `@Embeddable` has `private String zipCode` and the column is `zip_code`, the `name` in `@AttributeOverride` is `"zipCode"` (the Java field name).

```java
// WRONG — using column name:
@AttributeOverride(name = "zip_code", column = @Column(name = "work_zip")) // ❌

// CORRECT — using Java field name:
@AttributeOverride(name = "zipCode", column = @Column(name = "work_zip")) // ✅
```

---

### Nested Embeddables — Embeddables Inside Embeddables

Embeddables can themselves contain other embeddables. Hibernate flattens the entire structure into the owning entity's table.

```java
@Embeddable
public class GeoCoordinates {
    @Column(name = "latitude", precision = 10, scale = 7)
    private BigDecimal latitude;
    
    @Column(name = "longitude", precision = 10, scale = 7)
    private BigDecimal longitude;
}

@Embeddable
public class Address {
    private String street;
    private String city;
    private String zipCode;
    
    @Embedded // nested embeddable
    private GeoCoordinates coordinates; // flattened into same table
}

@Entity
public class Store {
    @Id @GeneratedValue Long id;
    
    @Embedded
    private Address location;
}
```

**Resulting table:**
```sql
CREATE TABLE store (
    id        BIGINT PRIMARY KEY,
    street    VARCHAR(255),
    city      VARCHAR(255),
    zip_code  VARCHAR(255),
    latitude  DECIMAL(10,7),   -- from GeoCoordinates, flattened
    longitude DECIMAL(10,7)    -- from GeoCoordinates, flattened
);
```

**Overriding nested embeddable columns:**
```java
@Embedded
@AttributeOverrides({
    @AttributeOverride(name = "city", column = @Column(name = "loc_city")),
    // For nested embeddable fields, use dot notation:
    @AttributeOverride(name = "coordinates.latitude",  
                       column = @Column(name = "loc_lat")),
    @AttributeOverride(name = "coordinates.longitude", 
                       column = @Column(name = "loc_lng"))
})
private Address location;
```

The **dot notation** for nested embeddable override is a critical exam point that most developers don't know.

---

### Null Embeddable Semantics — The Null Trap

What happens when `homeAddress` is null?

```java
Customer customer = new Customer("John");
customer.setHomeAddress(null); // embeddable is null
em.persist(customer);
```

**Hibernate behavior**: All embeddable columns are stored as `NULL` in the database.

On reading back:
- If ALL embeddable columns are `NULL` in DB → Hibernate returns `null` for the embeddable field
- If SOME columns are non-null → Hibernate instantiates the embeddable with mixed null/non-null fields

This is the **null reconstruction trap**:

```java
// DB state: street=NULL, city="London", zip=NULL (partial null)
Customer loaded = em.find(Customer.class, 1L);
Address addr = loaded.getHomeAddress();
// addr is NOT null — Hibernate creates Address instance
// addr.getStreet() == null
// addr.getCity() == "London"
// addr.getZipCode() == null
```

**Consequence**: You cannot reliably use `homeAddress == null` to check "has no address" if the DB can have partial null states. Use a `hasAddress()` method checking all fields, or use a dedicated `boolean` column.

**Best practice**: Use `@NotNull` on `@Column` definitions inside `@Embeddable` to prevent partial null states at the DB level. Or design embeddables so that "all null" and "partially null" are semantically impossible.

---

### Embeddables in Collections — `@ElementCollection`

An embeddable can appear in a collection, stored in a **separate collection table** (not the owning entity's table).

```java
@Entity
public class Customer {
    @Id @GeneratedValue Long id;
    
    @ElementCollection
    @CollectionTable(
        name = "customer_addresses",
        joinColumns = @JoinColumn(name = "customer_id")
    )
    private List<Address> addresses = new ArrayList<>();
    
    @ElementCollection
    @CollectionTable(name = "customer_phone_numbers")
    @Column(name = "phone") // for scalar element collections
    private Set<String> phoneNumbers = new HashSet<>();
}
```

**Resulting schema:**
```sql
-- Main entity table:
CREATE TABLE customers (
    id   BIGINT PRIMARY KEY,
    name VARCHAR(255)
);

-- Collection table for embedded Address list:
CREATE TABLE customer_addresses (
    customer_id BIGINT NOT NULL,  -- FK to customers
    street      VARCHAR(200),
    city        VARCHAR(100),
    zip_code    VARCHAR(20),
    country     VARCHAR(2)
    -- NO primary key on collection table by default!
);

-- Collection table for scalar strings:
CREATE TABLE customer_phone_numbers (
    customer_id BIGINT NOT NULL,
    phone       VARCHAR(255)
);
```

**Critical internal behavior of `@ElementCollection`:**

Element collections have **no identity** — the collection table has no primary key by default. This means Hibernate **cannot update individual elements**. When you modify a collection, Hibernate:

1. **DELETE all rows** for this owner: `DELETE FROM customer_addresses WHERE customer_id = ?`
2. **INSERT all rows** again with new state

```java
@Transactional
public void addPhone(Long customerId, String phone) {
    Customer c = em.find(Customer.class, customerId);
    c.getPhoneNumbers().add(phone);
    // At flush:
    // DELETE FROM customer_phone_numbers WHERE customer_id = ?
    // INSERT INTO customer_phone_numbers VALUES (?, ?)
    // INSERT INTO customer_phone_numbers VALUES (?, ?)
    // ... for ALL phones, not just the new one
}
```

This is extremely inefficient for large collections. Adding one element = delete all + reinsert all.

**`@OrderColumn`** can add an order column to the collection table:
```java
@ElementCollection
@OrderColumn(name = "address_order")
private List<Address> addresses;
// Adds: address_order INT to collection table
// Maintains list order persistently
```

**`@MapKeyColumn`** for Map-keyed element collections:
```java
@ElementCollection
@MapKeyColumn(name = "address_type")
@CollectionTable(name = "customer_addresses")
private Map<String, Address> addressesByType;
// Adds: address_type VARCHAR to collection table as map key
```

---

### Value Type Semantics — Equality Contract

Because embeddables are value types, their `equals()` and `hashCode()` must be based on **field values**, not object identity. This is not just a best practice — it affects correctness of collection operations.

```java
// If Address has no equals()/hashCode() override:
Set<Address> addresses = customer.getAddresses();
Address addr = new Address("123 Main", "NYC", "NY", "10001", "US");
addresses.contains(addr); // FALSE — uses Object.equals() (reference identity)
// Even if the DB has an identical address, this returns false

// With proper value-based equals()/hashCode():
addresses.contains(addr); // TRUE — compares field values
```

For embeddable classes inside `@ElementCollection`, the `equals()` is also used by Hibernate to detect collection changes during dirty checking.

---

### Embeddable with Associations

An `@Embeddable` class **can contain associations** to entities:

```java
@Embeddable
public class ContactInfo {
    
    private String email;
    private String phone;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "preferred_contact_method_id")
    private ContactMethod preferredMethod; // association inside embeddable
}
```

Hibernate handles this by propagating the association FK column into the owning entity's table. The `ContactMethod` entity has its own table; the FK lives in the owner's table.

This is valid but rare. The association is managed through the owning entity — cascade behavior follows the owning entity's cascade settings.

---

### `@Embeddable` vs `@Entity` — Decision Matrix

When should you use an embeddable vs a separate entity?

| Question | Use `@Embeddable` | Use `@Entity` |
|---|---|---|
| Does it need its own identity/PK? | No | Yes |
| Can it be shared between multiple owners? | No | Yes |
| Does it need to be queried independently? | No | Yes |
| Is its lifecycle fully owned by one entity? | Yes | No |
| Does it represent a value/concept, not a thing? | Yes | No |
| Do you need history/audit of the sub-object alone? | No | Yes |

Examples:
- `Address` on a `Customer` — `@Embeddable` (no independent identity, owned)
- `Address` shared between `Customer` and `Supplier` — borderline — could be `@Entity` with FK
- `OrderItem` on an `Order` — `@Entity` (has quantity, price, needs to be queried, managed)
- `Money(amount, currency)` on an `Invoice` — `@Embeddable` (pure value concept)
- `DateRange(start, end)` on a `Contract` — `@Embeddable` (value concept)

---

### JVM and Memory Implications

**Memory**: Embeddable instances are part of the owning entity's object graph. When the entity is in the Persistence Context (identity map), the embeddable instances are also reachable via the entity reference. No separate identity map entry for embeddables.

**Proxy**: Embeddable fields are never proxied by Hibernate's standard proxy mechanism. The entire embeddable is loaded when the entity is loaded (eagerly). There is no lazy loading at the embeddable level without bytecode enhancement.

**Dirty checking**: The `ComponentType` in Hibernate's type system performs deep comparison of embeddable fields during dirty checking. The snapshot array for an entity with an embedded `Address` contains all 5 address fields as individual elements in the snapshot array.

**Thread safety**: Same as entities — not thread-safe. Embeddable instances are not shared across threads.

---

## 2️⃣ Code Examples

### Example 1 — Complete Embeddable Domain Model

```java
// Value type: Money
@Embeddable
public class Money {
    
    @Column(name = "amount", precision = 19, scale = 4, nullable = false)
    private BigDecimal amount;
    
    @Column(name = "currency", length = 3, nullable = false)
    @Enumerated(EnumType.STRING)
    private Currency currency;
    
    protected Money() {}
    
    public Money(BigDecimal amount, Currency currency) {
        if (amount == null || currency == null) 
            throw new IllegalArgumentException("Amount and currency required");
        if (amount.compareTo(BigDecimal.ZERO) < 0)
            throw new IllegalArgumentException("Amount cannot be negative");
        this.amount = amount;
        this.currency = currency;
    }
    
    public Money add(Money other) {
        if (!this.currency.equals(other.currency))
            throw new IllegalArgumentException("Currency mismatch");
        return new Money(this.amount.add(other.amount), this.currency);
    }
    
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Money)) return false;
        Money that = (Money) o;
        return Objects.equals(amount, that.amount) && 
               Objects.equals(currency, that.currency);
    }
    
    @Override public int hashCode() { 
        return Objects.hash(amount, currency); 
    }
}

// Entity using Money twice:
@Entity
@Table(name = "invoices")
public class Invoice {
    
    @Id @GeneratedValue(strategy = GenerationType.SEQUENCE) 
    private Long id;
    
    private String invoiceNumber;
    
    // First Money embedding — uses default column names: amount, currency
    @Embedded
    private Money subtotal;
    
    // Second Money embedding — must override to avoid column clash
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "amount",   
                           column = @Column(name = "tax_amount", precision = 19, scale = 4)),
        @AttributeOverride(name = "currency", 
                           column = @Column(name = "tax_currency", length = 3))
    })
    private Money taxAmount;
    
    // Third embedding
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "amount",   
                           column = @Column(name = "total_amount", precision = 19, scale = 4)),
        @AttributeOverride(name = "currency", 
                           column = @Column(name = "total_currency", length = 3))
    })
    private Money totalAmount;
}
```

**Generated SQL table:**
```sql
CREATE TABLE invoices (
    id             BIGINT PRIMARY KEY,
    invoice_number VARCHAR(255),
    amount         DECIMAL(19,4),    -- subtotal.amount
    currency       VARCHAR(3),       -- subtotal.currency
    tax_amount     DECIMAL(19,4),    -- taxAmount.amount
    tax_currency   VARCHAR(3),       -- taxAmount.currency
    total_amount   DECIMAL(19,4),    -- totalAmount.amount
    total_currency VARCHAR(3)        -- totalAmount.currency
);
```

---

### Example 2 — Nested Embeddable with Dot-Notation Override

```java
@Embeddable
public class Coordinates {
    @Column(name = "lat", precision = 10, scale = 7)
    private BigDecimal latitude;
    
    @Column(name = "lng", precision = 10, scale = 7)
    private BigDecimal longitude;
    
    protected Coordinates() {}
    public Coordinates(BigDecimal lat, BigDecimal lng) { 
        this.latitude = lat; this.longitude = lng; 
    }
    // equals/hashCode...
}

@Embeddable
public class Address {
    private String street;
    private String city;
    
    @Embedded
    private Coordinates coordinates; // nested embeddable
}

@Entity
public class Store {
    @Id @GeneratedValue Long id;
    private String name;
    
    // First address — default columns
    @Embedded
    private Address mainAddress;
    
    // Second address — override including nested embeddable via dot notation
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street", 
                           column = @Column(name = "warehouse_street")),
        @AttributeOverride(name = "city",   
                           column = @Column(name = "warehouse_city")),
        // DOT NOTATION for nested embeddable:
        @AttributeOverride(name = "coordinates.latitude",  
                           column = @Column(name = "warehouse_lat")),
        @AttributeOverride(name = "coordinates.longitude", 
                           column = @Column(name = "warehouse_lng"))
    })
    private Address warehouseAddress;
}
```

---

### Example 3 — ElementCollection Behavior (Delete-All + Re-Insert)

```java
@Entity
public class Employee {
    @Id @GeneratedValue Long id;
    private String name;
    
    @ElementCollection
    @CollectionTable(
        name = "employee_skills",
        joinColumns = @JoinColumn(name = "employee_id")
    )
    @Column(name = "skill")
    private Set<String> skills = new HashSet<>();
}

// Test to observe SQL behavior:
@Transactional
public void demonstrateElementCollectionBehavior(Long employeeId) {
    Employee emp = em.find(Employee.class, employeeId);
    // Assume DB has: skills = {"Java", "SQL", "Spring"}
    
    emp.getSkills().add("Kubernetes");
    // You might expect: INSERT INTO employee_skills VALUES (?, 'Kubernetes')
    
    // ACTUAL SQL at flush:
    // DELETE FROM employee_skills WHERE employee_id = ?    ← removes ALL
    // INSERT INTO employee_skills VALUES (?, 'Java')        ← re-inserts all
    // INSERT INTO employee_skills VALUES (?, 'SQL')
    // INSERT INTO employee_skills VALUES (?, 'Spring')
    // INSERT INTO employee_skills VALUES (?, 'Kubernetes')  ← and the new one
}
```

---

### Example 4 — Null Embeddable Reconstruction

```java
@Transactional
public void nullEmbeddableDemo() {
    Customer c = new Customer("Alice");
    c.setHomeAddress(null); // no address
    em.persist(c);
    em.flush();
    // SQL: INSERT INTO customers (id, name, street, city, zip_code, country)
    //      VALUES (1, 'Alice', NULL, NULL, NULL, NULL)
    
    em.clear(); // detach all
    
    Customer loaded = em.find(Customer.class, 1L);
    // SQL: SELECT id, name, street, city, zip_code, country FROM customers WHERE id=1
    
    Address addr = loaded.getHomeAddress();
    System.out.println(addr); // NULL — all columns were null → Hibernate returns null
    
    // Partial null scenario:
    // Manually update DB: UPDATE customers SET city='London' WHERE id=1
    em.clear();
    Customer loaded2 = em.find(Customer.class, 1L);
    Address addr2 = loaded2.getHomeAddress();
    System.out.println(addr2 == null);          // FALSE — city is non-null
    System.out.println(addr2.getStreet());      // null
    System.out.println(addr2.getCity());        // "London"
    // Partial object — dangerous if your code assumes all-or-nothing
}
```

---

### Example 5 — Dirty Checking on Embedded Fields

```java
@Transactional
public void embeddableDirtyCheck() {
    Customer c = em.find(Customer.class, 1L);
    // Snapshot includes all Address fields:
    // snapshot[...] = ["123 Main", "NYC", "NY", "10001", "US"]
    
    // Mutate the embeddable:
    c.getHomeAddress().setCity("Brooklyn");
    // snapshot[city_index] = "NYC" still
    // current value: "Brooklyn"
    
    // At flush: Hibernate compares all fields including embedded ones
    // Detects city changed → generates UPDATE
    // SQL: UPDATE customers SET city='Brooklyn' WHERE id=1
    
    // Replace entire embeddable:
    c.setHomeAddress(new Address("456 Oak", "Queens", "NY", "11368", "US"));
    // All 5 address fields are different from snapshot → UPDATE all 5
    // SQL: UPDATE customers SET street='456 Oak', city='Queens', 
    //      state='NY', zip_code='11368', country='US' WHERE id=1
}
```

---

### Example 6 — `@Embeddable` with `@AssociationOverride`

```java
@Embeddable
public class Warranty {
    private LocalDate startDate;
    private LocalDate endDate;
    
    @ManyToOne
    @JoinColumn(name = "provider_id") // default FK column name
    private WarrantyProvider provider;
}

@Entity
public class Product {
    @Id @GeneratedValue Long id;
    
    @Embedded
    private Warranty manufacturerWarranty;
    
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "startDate", 
                           column = @Column(name = "ext_warranty_start")),
        @AttributeOverride(name = "endDate",   
                           column = @Column(name = "ext_warranty_end"))
    })
    @AssociationOverrides({
        // Override the FK column for the association inside the embeddable:
        @AssociationOverride(name = "provider",
                             joinColumns = @JoinColumn(name = "ext_provider_id"))
    })
    private Warranty extendedWarranty;
}
```

`@AssociationOverride` is the counterpart of `@AttributeOverride` for associations inside embeddables — a rarely known but exam-tested annotation.

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ**
An `@Embeddable` class is embedded twice in the same `@Entity`. What is required to avoid a mapping exception?

A) Two separate `@Embeddable` classes with different column names  
B) `@AttributeOverride` to remap column names for at least one of the usages  
C) A separate `@Table` annotation on the `@Embeddable`  
D) `@EmbeddedId` annotation on the second usage  

**Answer: B** — When the same embeddable type is used twice, column names clash. `@AttributeOverride` renames the columns for one (or both) usages.

---

**Q2 — Select All That Apply**
Which statements about `@ElementCollection` are TRUE? (Select all)

A) Element collections are stored in a separate collection table  
B) Adding one element to an `@ElementCollection` Set generates one INSERT  
C) Hibernate deletes all collection rows and re-inserts when a collection is modified  
D) The collection table has a primary key by default  
E) `@CollectionTable` is required to specify the collection table name  

**Answer: A, C**
- B is false: Hibernate deletes all and re-inserts all
- D is false: no PK by default on collection tables
- E is false: `@CollectionTable` is optional (Hibernate generates a default name); recommended for clarity but not required

---

**Q3 — Code Output Prediction**
```java
@Embeddable
public class Range {
    private int min;
    private int max;
    // NO equals()/hashCode() override
}

@Entity
public class Product {
    @Id Long id;
    @Embedded Range priceRange;
}

// In test:
Set<Range> ranges = new HashSet<>();
ranges.add(new Range(1, 100));
ranges.add(new Range(1, 100)); // identical values
System.out.println(ranges.size());
```

A) 1 — embeddables are compared by value  
B) 2 — no `equals()`/`hashCode()` → uses reference identity → two different instances  
C) Compile error  
D) 1 — JPA enforces value equality for embeddables automatically  

**Answer: B** — Without `equals()`/`hashCode()` override, `Object.equals()` is used (reference identity). Two distinct instances with identical values are considered different. The `Set` contains 2 elements. JPA does NOT inject equals/hashCode behavior.

---

**Q4 — `@AttributeOverride` name trap**
```java
@Embeddable
public class PhoneNumber {
    @Column(name = "phone_number")
    private String phoneNumber; // field name: phoneNumber
}

@Entity
public class Contact {
    @Embedded
    @AttributeOverride(
        name = "phone_number", // ← using column name, not field name
        column = @Column(name = "mobile_phone")
    )
    private PhoneNumber mobilePhone;
}
```

What happens at runtime?

A) Works correctly — name refers to column name  
B) `MappingException` or the override is silently ignored — name must be the Java field name  
C) Compile error  
D) Works but generates two columns: `phone_number` and `mobile_phone`  

**Answer: B** — `@AttributeOverride(name=...)` must reference the **Java field name** (`"phoneNumber"`), not the column name (`"phone_number"`). Using the column name results in the override being silently ignored or a mapping exception. The column `phone_number` is created with the original name, not overridden.

---

**Q5 — Null Embeddable Scenario**
```java
@Entity
public class User {
    @Id Long id;
    @Embedded
    private Address address; // nullable
}

// User saved with null address → DB: all address columns are NULL
// DB manually updated: only 'city' column set to 'Paris', others NULL

User u = em.find(User.class, 1L);
System.out.println(u.getAddress() == null);
```

A) true — if any column is null, the entire embeddable is null  
B) false — Hibernate creates an Address instance because 'city' is non-null  
C) true — Hibernate requires ALL columns non-null to instantiate embeddable  
D) NullPointerException during find()  

**Answer: B** — Hibernate creates an embeddable instance if ANY column is non-null. The result is an `Address` with `city="Paris"` and all other fields null. This partial state can be dangerous.

---

**Q6 — ElementCollection Performance**
```java
@ElementCollection
private List<String> tags; // currently has 1000 tags
```
You add one tag: `product.getTags().add("newTag")`. How many SQL statements are generated at flush?

A) 1 INSERT  
B) 1 DELETE + 1 INSERT  
C) 1 DELETE + 1001 INSERTs  
D) 1 DELETE (all) + 1001 INSERTs (all tags including new one)  

**Answer: D** — Hibernate cannot update individual element collection rows (no PK to target). It deletes ALL 1000 rows, then re-inserts all 1001. This is why large element collections with frequent updates are a serious performance problem.

---

**Q7 — Nested Embeddable Override**
```java
@Embeddable class Inner { @Column(name="val") String value; }
@Embeddable class Outer { @Embedded Inner inner; String name; }

@Entity class MyEntity {
    @Embedded
    @AttributeOverride(name = "inner.value", 
                       column = @Column(name = "overridden_val"))
    private Outer outer;
}
```
Is the `@AttributeOverride` for the nested embeddable field correct?

A) No — dot notation is invalid; must use `@Embedded` on the Outer field separately  
B) Yes — dot notation is the correct way to reference nested embeddable fields  
C) Compile error — `@AttributeOverride` doesn't support dot notation  
D) No — you must repeat the full `@AttributeOverrides` on `Inner` inside `Outer`  

**Answer: B** — JPA specification supports dot notation in `@AttributeOverride(name=...)` for referencing fields in nested embeddables. `"inner.value"` correctly targets the `value` field inside the nested `Inner` embeddable.

---

**Q8 — Embeddable vs Entity Design**
A system must track `Address` objects that are shared between `Customer` and `Supplier` entities, and addresses must be queryable independently (e.g., "find all entities at this zip code"). Should `Address` be `@Embeddable` or `@Entity`?

A) `@Embeddable` — it's an address, always a value type  
B) `@Entity` — it needs independent queryability and is shared across entity types  
C) `@Embeddable` with `@ElementCollection`  
D) Either works — the choice is only about preference  

**Answer: B** — When a concept needs: (1) independent queries, (2) shared references across multiple owner types, it needs its own identity → `@Entity`. `@Embeddable` is stored within the owner's table and cannot be shared or queried independently.

---

## 4️⃣ Trick Analysis

**The `@AttributeOverride(name=...)` field vs column trap (Q4)**:
This is consistently tested because it's counter-intuitive. The `@Column` annotation on the `@Embeddable` field specifies the column name. The `@AttributeOverride` name refers to the **Java field** inside the embeddable, not the column it maps to. Developers confuse these because they think of column names, not field names. Rule: `name` in `@AttributeOverride` = Java field name in `@Embeddable`.

**The `equals()`/`hashCode()` value type contract (Q3)**:
JPA/Hibernate does NOT magically implement `equals()`/`hashCode()` for embeddables. Without override, `Object.equals()` (reference identity) is used. This breaks `Set`/`Map` collections containing embeddables and `List.contains()` checks. Always implement `equals()`/`hashCode()` in `@Embeddable` classes based on field values.

**The null reconstruction partial state (Q5)**:
Many developers assume "if address is null in the entity, all DB columns are null, and if all DB columns are null, the embedded object is null." The reverse path is asymmetric: Hibernate returns null only if ALL columns are null. One non-null column → non-null embeddable with other fields null. This creates unpredictable object states.

**ElementCollection delete-all behavior (Q6, Q7 in examples)**:
The root cause is the absence of a primary key on the collection table. Without a PK, Hibernate cannot issue `UPDATE WHERE pk=?` or `DELETE WHERE pk=? AND element=?`. It must use the FK (owner ID) to target the entire set of rows. Adding `@OrderColumn` gives Hibernate a position column that enables slightly smarter updates for lists (not full delete-reinsert in all cases), but sets remain delete-all + reinsert.

**Embeddable sharing impossibility (Q8)**:
An `@Embeddable` instance's columns exist in the owner's table. If `Address` is embedded in both `customers` and `suppliers` tables, there are two separate copies of the data — no sharing. Modifying a `Customer`'s address doesn't affect a `Supplier` with the same data. True sharing requires an `@Entity` with a foreign key relationship.

**The `@Basic(fetch=LAZY)` inapplicability to embeddables**:
Just as individual basic fields cannot be lazily loaded without bytecode enhancement, embeddable fields are also loaded eagerly as part of the entity SELECT. You cannot lazy-load an embeddable through standard proxy mechanisms.

---

## 5️⃣ Summary Sheet

### Embeddable vs Entity Decision Checklist

```
Ask these questions:
  1. Does it have independent identity (own PK)?         YES → @Entity
  2. Is it shared between multiple owners?               YES → @Entity  
  3. Must it be queried independently?                   YES → @Entity
  4. Does it have its own lifecycle (exist without owner)?  YES → @Entity
  5. Is it a pure value concept (Money, Address, Range)?  YES → @Embeddable
  6. Does its lifecycle fully belong to one owner?        YES → @Embeddable
```

### `@AttributeOverride` Reference

```
@AttributeOverride(
    name   = "javaFieldName",        // field name in @Embeddable (NOT column name)
    column = @Column(name = "col")   // target SQL column name
)

For nested embeddable:
    name = "nestedFieldName.innerFieldName"  // dot notation

For associations inside embeddable:
    Use @AssociationOverride(name = "assocFieldName", joinColumns = @JoinColumn(...))
```

### Embeddable Null Behavior

```
All columns NULL in DB    → null embeddable returned
ANY column non-NULL in DB → embeddable instance created, other fields null
Embeddable field is null  → INSERT/UPDATE with all NULL for those columns
```

### `@ElementCollection` SQL Behavior

```
Operation on collection    →  SQL Generated
─────────────────────────────────────────────────────
add(element)               →  DELETE all + INSERT all (including new)
remove(element)            →  DELETE all + INSERT all (minus removed)
clear()                    →  DELETE all
replace entire collection  →  DELETE all + INSERT all new
setCollection(null)        →  DELETE all
```

### Embeddable Architecture Summary

```
@Embeddable class:
  - No @Id
  - No separate table
  - No identity map entry
  - No proxy
  - Always loaded eagerly (without bytecode enhancement)
  - Dirty checking via owning entity's snapshot
  - Must implement equals()/hashCode() based on values
  - Must have no-arg constructor

@Embedded usage:
  - Columns flattened into owner's table
  - @AttributeOverride required when same type used multiple times
  - Dot notation for nested embeddable overrides

@ElementCollection:
  - Stored in separate collection table
  - No PK on collection table (by default)
  - Full delete+reinsert on any modification
  - @CollectionTable specifies table name and FK
  - @OrderColumn to persist List ordering
```

### Comparison Table — Mapping Strategies for Sub-Objects

| Aspect | `@Embeddable` | `@ElementCollection` | `@OneToMany` `@Entity` |
|---|---|---|---|
| Table structure | Same table as owner | Separate collection table | Separate entity table |
| Has own PK | ❌ | ❌ (by default) | ✅ |
| Independently queryable | ❌ | ❌ | ✅ |
| Shareable across owners | ❌ | ❌ | ✅ |
| Update granularity | Full entity UPDATE | Delete all + reinsert | UPDATE individual rows |
| Lifecycle | Owned by entity | Owned by entity | Independent |
| Lazy loading | ❌ (without bytecode) | ✅ (lazy by default) | ✅ (configurable) |

---
