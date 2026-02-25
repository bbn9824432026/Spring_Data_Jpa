# 🏛️ TOPIC 6 — Association Mapping

---

## 1️⃣ Conceptual Explanation

### The Owning Side vs Inverse Side — The Most Misunderstood Concept in JPA

Before examining individual association types, you must internalize the single most important concept in association mapping: **every bidirectional relationship has exactly one owning side and one inverse side**. This is not optional, not configurable — it is a fundamental JPA contract.

**Why does this exist?**

A relational database represents a relationship with a **foreign key column** in exactly one table. A Java bidirectional relationship has references on **both sides** of the object graph. JPA must decide: which side controls the FK column? Which side's state is authoritative for writing to the database?

The answer is: **the owning side**.

```
Database reality:
  orders table has: customer_id FK column  ← one FK, one place

Java reality:
  Customer has: List<Order> orders         ← reference from customer side
  Order has:    Customer customer          ← reference from order side

JPA question: when Hibernate writes customer_id, which Java reference does it read?
JPA answer:   the OWNING SIDE — whichever side does NOT have mappedBy
```

**Rules — absolute and non-negotiable:**

1. The side with `mappedBy` = **inverse side** — Hibernate **ignores** this side for SQL writes
2. The side WITHOUT `mappedBy` = **owning side** — Hibernate reads this side for FK column writes
3. `mappedBy` value = the **field name** on the owning side that maps back to this entity
4. For `@ManyToOne` / `@OneToMany` bidirectional: `@ManyToOne` side is **always** the owning side
5. For `@ManyToMany`: you choose which side owns; the other gets `mappedBy`
6. For `@OneToOne`: you choose; typically the side holding the FK column

**The bidirectional consistency contract:**

Because the inverse side is ignored for writes, you bear the responsibility of keeping both sides in sync in Java memory. If you only set one side, the in-memory object graph is inconsistent even though the database write may be correct.

```java
// WRONG — only sets owning side (Order.customer):
order.setCustomer(customer);
// DB write: correct (customer_id FK is set)
// In-memory: customer.getOrders() does NOT contain this order → inconsistency

// WRONG — only sets inverse side (Customer.orders):
customer.getOrders().add(order);
// DB write: NOTHING (inverse side is ignored by Hibernate!)
// customer_id FK is NOT set → order has null FK in DB

// CORRECT — set both sides:
order.setCustomer(customer);           // owning side → drives DB write
customer.getOrders().add(order);       // inverse side → keeps Java graph consistent
```

The standard pattern is a **helper method** on the entity that sets both sides atomically:

```java
public class Customer {
    private List<Order> orders = new ArrayList<>();
    
    public void addOrder(Order order) {
        orders.add(order);          // inverse side
        order.setCustomer(this);    // owning side
    }
    
    public void removeOrder(Order order) {
        orders.remove(order);       // inverse side
        order.setCustomer(null);    // owning side
    }
}
```

---

### `@ManyToOne` — The Foundation Association

`@ManyToOne` is the most common and most important association. It is **always the owning side** in a bidirectional `@OneToMany` / `@ManyToOne` relationship. It maps the FK column directly.

```java
@Entity
public class Order {
    
    @Id @GeneratedValue Long id;
    
    @ManyToOne(
        fetch = FetchType.LAZY,         // ALWAYS use LAZY (default is EAGER — bad)
        optional = false,               // FK is NOT NULL — customer is mandatory
        cascade = {}                    // typically no cascade on @ManyToOne
    )
    @JoinColumn(
        name = "customer_id",           // FK column name in orders table
        nullable = false,               // NOT NULL constraint (DDL)
        foreignKey = @ForeignKey(name = "fk_order_customer")  // FK constraint name
    )
    private Customer customer;
}
```

**Generated DDL:**
```sql
CREATE TABLE orders (
    id          BIGINT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    CONSTRAINT fk_order_customer FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

**Default fetch type for `@ManyToOne`: `EAGER`**

This is one of JPA's worst defaults. Every `Order` load eagerly loads its `Customer`, which may eagerly load its `Address`, which... This can cause catastrophic chains of eager loading. **Always override to `LAZY`:**

```java
@ManyToOne(fetch = FetchType.LAZY)
```

**Internal proxy mechanics for `@ManyToOne(fetch=LAZY)`:**

When Hibernate loads an `Order`, it does NOT load the `Customer` immediately. Instead, it places a **Hibernate proxy** in the `customer` field. The proxy is a CGLIB/ByteBuddy-generated subclass of `Customer` that:
- Holds only the FK value (customer_id) — already in the `orders` result set
- Intercepts the first method call on the proxy (except `getId()`)
- On first method call: fires SELECT to load the real Customer data
- Replaces itself in the identity map with the real instance

```java
Order order = em.find(Order.class, 1L);
// SQL: SELECT * FROM orders WHERE id=1
// customer field = Hibernate proxy (not yet loaded)

String customerName = order.getCustomer().getName();
// PROXY IS TRIGGERED HERE → SQL: SELECT * FROM customers WHERE id=?
// Now customer is fully loaded
```

**`getId()` on proxy — the special case:**

Calling `proxy.getId()` does NOT trigger loading because Hibernate knows the ID value from the FK column. This is important:

```java
Long customerId = order.getCustomer().getId(); // NO SQL — proxy already has ID
String name = order.getCustomer().getName();   // SQL fired here
```

---

### `@OneToMany` — The Inverse Side and the Join Table Trap

`@OneToMany` is almost always the **inverse side** in a bidirectional relationship (using `mappedBy`). It can also be unidirectional — but unidirectional `@OneToMany` without `mappedBy` has a serious hidden behavior.

**Bidirectional `@OneToMany` (correct pattern):**

```java
@Entity
public class Customer {
    
    @Id @GeneratedValue Long id;
    
    @OneToMany(
        mappedBy = "customer",      // field name in Order that owns the relationship
        fetch = FetchType.LAZY,     // LAZY is the default for collections — keep it
        cascade = CascadeType.ALL,  // cascade operations to orders
        orphanRemoval = true        // delete Order if removed from collection
    )
    private List<Order> orders = new ArrayList<>();
}
```

**Unidirectional `@OneToMany` — THE JOIN TABLE TRAP:**

```java
@Entity
public class Customer {
    
    @OneToMany  // NO mappedBy — unidirectional
    private List<Order> orders;
}
```

When no `mappedBy` is present on `@OneToMany`, Hibernate assumes you want a **join table** — even though there's no `@JoinTable` annotation. This is a silent, catastrophic default:

```sql
-- Hibernate creates a SURPRISE join table:
CREATE TABLE customer_orders (
    customer_id BIGINT NOT NULL,
    orders_id   BIGINT NOT NULL    -- FK to orders table
);
```

You now have a join table that nobody asked for. The `Order` table has no `customer_id` FK column. Every relationship is managed through this surprise join table. This is almost never what developers want.

**Fix options:**

Option 1 — Add `@JoinColumn` to make it a FK-based unidirectional:
```java
@OneToMany
@JoinColumn(name = "customer_id") // tells Hibernate: use FK in orders table, not join table
private List<Order> orders;
// Now: no join table, orders.customer_id FK is used
// But: this FK is managed by Customer (unusual — FK in child managed by parent)
```

Option 2 — Make it bidirectional with `mappedBy` (recommended):
```java
// In Customer:
@OneToMany(mappedBy = "customer")
private List<Order> orders;

// In Order:
@ManyToOne
@JoinColumn(name = "customer_id")
private Customer customer;
```

**Unidirectional `@OneToMany` with `@JoinColumn` — inefficient SQL:**

Even with `@JoinColumn`, unidirectional `@OneToMany` generates inefficient SQL:

```java
// Adding an Order to Customer's orders list:
customer.getOrders().add(newOrder);
// SQL: INSERT INTO orders (id, ...) VALUES (...)  ← inserts with null FK
// SQL: UPDATE orders SET customer_id=? WHERE id=? ← then sets FK separately

// Removing an Order:
customer.getOrders().remove(order);
// SQL: UPDATE orders SET customer_id=NULL WHERE id=? ← nullifies FK
// SQL: DELETE FROM orders WHERE id=? ← if orphanRemoval=true
```

The bidirectional approach (owning side on `@ManyToOne`) generates a single INSERT with the FK set correctly. This is why bidirectional is almost always preferred.

---

### `@OneToOne` — Shared PK vs FK Strategies

`@OneToOne` has two common mapping strategies, each with different tradeoffs.

**Strategy 1 — FK-based (most common):**

```java
@Entity
public class User {
    @Id @GeneratedValue Long id;
    String username;
    
    @OneToOne(
        mappedBy = "user",    // UserProfile owns the relationship
        fetch = FetchType.LAZY,
        cascade = CascadeType.ALL,
        optional = true
    )
    private UserProfile profile;
}

@Entity
public class UserProfile {
    @Id @GeneratedValue Long id;
    String bio;
    
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", unique = true) // unique = true enforces 1:1 at DB
    private User user; // owning side — has FK column
}
```

**Generated DDL:**
```sql
CREATE TABLE user_profiles (
    id      BIGINT PRIMARY KEY,
    bio     VARCHAR(255),
    user_id BIGINT UNIQUE,  -- unique constraint enforces 1:1
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

**Strategy 2 — Shared Primary Key:**

```java
@Entity
public class UserProfile {
    
    @Id  // no @GeneratedValue — ID comes from User
    private Long id;
    
    @OneToOne(fetch = FetchType.LAZY)
    @MapsId  // user's PK becomes this entity's PK
    @JoinColumn(name = "id")
    private User user;
    
    String bio;
}
```

**Generated DDL:**
```sql
CREATE TABLE user_profiles (
    id  BIGINT PRIMARY KEY,  -- same value as users.id
    bio VARCHAR(255),
    FOREIGN KEY (id) REFERENCES users(id)
);
```

**The `@OneToOne` lazy loading problem — a critical Hibernate limitation:**

For `@OneToOne` associations, lazy loading on the **inverse side** (`mappedBy` side) is unreliable without bytecode enhancement. Here's why:

Hibernate needs to decide: should it put a proxy in `user.profile`, or null? To know whether to create a proxy, it must know whether a `UserProfile` with this FK exists — which requires a SELECT. So Hibernate fires the SELECT anyway, defeating the purpose of lazy loading.

```java
User user = em.find(User.class, 1L);
// Expected: profile field is a proxy (lazy)
// Actual: Hibernate fires SELECT to check if profile exists
//         → if exists: loads it (not lazy!)
//         → if not exists: sets null

// Fix: Bytecode enhancement or use @LazyToOne(LazyToOneOption.NO_PROXY)
```

This is a known Hibernate limitation for `@OneToOne` inverse side. The owning side (`UserProfile.user`) lazy loads correctly because Hibernate knows the FK value without extra queries.

---

### `@ManyToMany` — Join Table, Owning Side, Cascade Trap

`@ManyToMany` always uses a join table. You choose which side is the owning side.

```java
@Entity
public class Student {
    @Id @GeneratedValue Long id;
    String name;
    
    @ManyToMany
    @JoinTable(
        name = "student_courses",
        joinColumns = @JoinColumn(name = "student_id"),        // FK to this entity
        inverseJoinColumns = @JoinColumn(name = "course_id")   // FK to other entity
    )
    private Set<Course> courses = new HashSet<>(); // owning side
}

@Entity
public class Course {
    @Id @GeneratedValue Long id;
    String title;
    
    @ManyToMany(mappedBy = "courses") // inverse side
    private Set<Student> students = new HashSet<>();
}
```

**Generated DDL:**
```sql
CREATE TABLE student_courses (
    student_id BIGINT NOT NULL,
    course_id  BIGINT NOT NULL,
    PRIMARY KEY (student_id, course_id),  -- composite PK
    FOREIGN KEY (student_id) REFERENCES students(id),
    FOREIGN KEY (course_id)  REFERENCES courses(id)
);
```

**The `@ManyToMany` cascade trap — the most dangerous cascade mistake:**

```java
@ManyToMany(cascade = CascadeType.ALL)  // EXTREMELY DANGEROUS
private Set<Course> courses;

// What happens when you delete a Student:
em.remove(student);
// Cascade ALL → removes all Course entities the student was enrolled in
// Even if other students are still enrolled in those courses!
// This deletes courses that are still in use → data corruption
```

**Rule**: Never use `CascadeType.REMOVE` or `CascadeType.ALL` on `@ManyToMany`. Use only `PERSIST` and `MERGE` if needed, or no cascade at all.

**`@ManyToMany` with extra join table columns — must use intermediate entity:**

The standard `@ManyToMany` join table can only hold the two FK columns. If you need extra data (e.g., enrollment date, grade), you must replace `@ManyToMany` with an intermediate entity:

```java
// Instead of @ManyToMany:
@Entity
public class Enrollment {   // the join table becomes an entity
    
    @EmbeddedId
    private EnrollmentId id;  // composite PK
    
    @ManyToOne @MapsId("studentId")
    private Student student;
    
    @ManyToOne @MapsId("courseId")
    private Course course;
    
    // Extra columns on the join:
    private LocalDate enrolledAt;
    private String grade;
}

@Embeddable
public class EnrollmentId implements Serializable {
    private Long studentId;
    private Long courseId;
    // equals/hashCode required
}
```

---

### `@JoinColumn` Deep Internals

```java
@JoinColumn(
    name = "customer_id",           // FK column name in the owning table
    referencedColumnName = "id",    // PK column in the referenced table (default: PK)
    nullable = false,               // NOT NULL (DDL)
    unique = false,                 // for @OneToOne: set true
    insertable = true,              // include in INSERT
    updatable = true,               // include in UPDATE
    foreignKey = @ForeignKey(
        name = "fk_order_customer",
        value = ConstraintMode.CONSTRAINT  // CONSTRAINT, NO_CONSTRAINT, PROVIDER_DEFAULT
    )
)
```

**`referencedColumnName`** — Almost always the PK. Use non-PK only for natural key joins:
```java
@JoinColumn(name = "customer_code", referencedColumnName = "code")
// Joins on customer.code (natural key) instead of customer.id (PK)
```

**`ConstraintMode.NO_CONSTRAINT`** — Creates the FK column but no database FK constraint:
```java
@ForeignKey(value = ConstraintMode.NO_CONSTRAINT)
// Useful for: performance (no FK validation on insert), 
//             cross-schema FKs, soft-delete scenarios
```

---

### Cascade Types — Deep Semantics

```java
public enum CascadeType {
    ALL,      // all of the below
    PERSIST,  // em.persist() cascades to associated entities
    MERGE,    // em.merge() cascades to associated entities
    REMOVE,   // em.remove() cascades to associated entities
    REFRESH,  // em.refresh() cascades to associated entities
    DETACH    // em.detach() cascades to associated entities
}
```

**`CascadeType.PERSIST`** — When you persist the parent, also persist new children:
```java
Order order = new Order();
OrderItem item = new OrderItem(order, "Widget", 2);
order.getItems().add(item);

em.persist(order); // CascadeType.PERSIST → also persists item
// No need to call em.persist(item) separately
```

**`CascadeType.MERGE`** — When you merge a detached parent, also merge detached children:
```java
// order is detached, order.items contains detached items
Order managed = em.merge(order);
// CascadeType.MERGE → also merges all items
// Returns managed copies of all
```

**`CascadeType.REMOVE`** — When you remove the parent, also remove children:
```java
em.remove(order); // CascadeType.REMOVE → also removes all order.items
// SQL: DELETE FROM order_items WHERE order_id=?
//      DELETE FROM orders WHERE id=?
```

**`CascadeType.REMOVE` vs `orphanRemoval`:**

| Aspect | `CascadeType.REMOVE` | `orphanRemoval = true` |
|---|---|---|
| Trigger | `em.remove(parent)` | Removing child from collection |
| Scope | Cascades remove operation | Removes orphaned (unlinked) children |
| Example | Delete order → delete items | Remove item from order.items list → delete item |

```java
// CascadeType.REMOVE:
em.remove(order); → deletes order AND all items

// orphanRemoval:
order.getItems().remove(item); → deletes item (it's now an orphan)
// This works WITHOUT calling em.remove(item)

// Both can be combined:
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
// persist/merge/remove all cascade + orphan removal active
```

**`orphanRemoval` trap — collection replacement:**
```java
// With orphanRemoval = true:
order.setItems(new ArrayList<>()); // replaces the entire collection
// Hibernate detects all old items are now orphans
// DELETE for every old item
// This is intentional behavior but surprises developers
```

---

### Fetch Type Defaults — The Hidden EAGER Dangers

| Association | Default FetchType | Recommendation |
|---|---|---|
| `@ManyToOne` | **EAGER** | Always override to LAZY |
| `@OneToOne` | **EAGER** | Always override to LAZY |
| `@OneToMany` | LAZY | Keep as LAZY |
| `@ManyToMany` | LAZY | Keep as LAZY |

The two `EAGER` defaults (`@ManyToOne`, `@OneToOne`) are JPA's most consequential design mistakes. They cause N+1 queries, unnecessary data loading, and cartesian product explosions in complex entity graphs.

**Global EAGER fetch — the permanent N+1:**

```java
@ManyToOne(fetch = FetchType.EAGER)  // global setting
private Customer customer;

// Every order load eagerly loads customer:
List<Order> orders = em.createQuery("FROM Order", Order.class).getResultList();
// SQL: SELECT * FROM orders
// For each order: SELECT * FROM customers WHERE id=?  ← N+1!
// 100 orders = 101 SQL queries
```

EAGER in query overrides: you can add `JOIN FETCH` or `@EntityGraph` to load eagerly per-query even if the mapping is LAZY. The reverse — making EAGER lazy per-query — is **not reliably possible** in JPA.

---

### `mappedBy` Internals — How Hibernate Resolves It

`mappedBy = "customer"` tells Hibernate:
1. "This side (`@OneToMany`) does NOT control the FK"
2. "Go look at field named `customer` in the other entity (`Order.customer`)"
3. "That `@ManyToOne` field's `@JoinColumn` defines the FK column"
4. "For loading this collection: `SELECT * FROM orders WHERE customer_id = ?`"

**`mappedBy` validation at startup:**
Hibernate validates that the named field exists on the other entity. If you misspell `mappedBy = "custumer"`, you get a `MappingException` at startup — not at runtime query time.

```java
@OneToMany(mappedBy = "custumer") // typo
// Startup: MappingException: "mappedBy reference an unknown target entity property"
```

---

### Association Best Practices — Architect Level

**1. Always specify fetch type explicitly:**
```java
@ManyToOne(fetch = FetchType.LAZY)   // never rely on EAGER default
@OneToOne(fetch = FetchType.LAZY)
```

**2. Use `Set` instead of `List` for `@ManyToMany`:**
```java
// List with @ManyToMany causes Hibernate to delete all + reinsert on any change
// Set uses individual INSERTs and DELETEs
@ManyToMany
private Set<Course> courses = new HashSet<>(); // preferred
```

**3. Avoid `@ManyToMany` when extra join columns needed — use intermediate entity.**

**4. Never cascade REMOVE on `@ManyToMany`.**

**5. Use `@JoinColumn(nullable=false)` to express mandatory relationships at schema level.**

**6. Implement `equals()`/`hashCode()` for entities in collections** — especially important for `Set`-based associations. Use business key, not generated ID (ID may be null before persist).

---

## 2️⃣ Code Examples

### Example 1 — Bidirectional `@OneToMany` / `@ManyToOne` Complete

```java
@Entity
@Table(name = "orders")
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)
    private Long id;
    
    @Column(nullable = false)
    private LocalDateTime orderDate;
    
    // OWNING SIDE — has FK column, no mappedBy
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "customer_id", nullable = false,
                foreignKey = @ForeignKey(name = "fk_order_customer"))
    private Customer customer;
    
    // OWNING SIDE of items relationship
    @OneToMany(
        mappedBy = "order",
        cascade = CascadeType.ALL,
        orphanRemoval = true,
        fetch = FetchType.LAZY
    )
    private List<OrderItem> items = new ArrayList<>();
    
    // Helper methods for bidirectional consistency:
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }
    
    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }
}

@Entity
@Table(name = "customers")
public class Customer {
    
    @Id @GeneratedValue Long id;
    String name;
    
    // INVERSE SIDE — mappedBy, ignored for writes
    @OneToMany(
        mappedBy = "customer",
        cascade = {CascadeType.PERSIST, CascadeType.MERGE},
        fetch = FetchType.LAZY
    )
    private List<Order> orders = new ArrayList<>();
    
    public void addOrder(Order order) {
        orders.add(order);       // inverse side — in-memory consistency
        order.setCustomer(this); // owning side — drives DB write
    }
}

@Entity
@Table(name = "order_items")
public class OrderItem {
    
    @Id @GeneratedValue Long id;
    int quantity;
    BigDecimal unitPrice;
    
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order; // owning side
    
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product; // owning side
}
```

---

### Example 2 — Demonstrating `mappedBy` Ignore Behavior

```java
@Transactional
public void owningVsInverseTrap() {
    Customer customer = em.find(Customer.class, 1L);
    Order order = new Order();
    order.setOrderDate(LocalDateTime.now());
    
    // MISTAKE 1: Only setting inverse side:
    customer.getOrders().add(order); // inverse side — IGNORED by Hibernate
    em.persist(order);
    em.flush();
    // SQL: INSERT INTO orders (id, order_date, customer_id) 
    //      VALUES (?, ?, NULL)  ← customer_id is NULL!
    // Hibernate reads Order.customer (owning side) which is null
    
    // MISTAKE 2: Only setting owning side:
    order.setCustomer(customer); // owning side — CORRECT for DB
    // customer.getOrders() still doesn't contain order → in-memory inconsistency
    // If you call customer.getOrders().size() → doesn't include this order
    // Within same transaction (no flush/reload), this is a consistency bug
    
    // CORRECT: Set both sides:
    order.setCustomer(customer);        // owning side → drives customer_id FK
    customer.getOrders().add(order);    // inverse side → in-memory consistency
    em.persist(order);
    // SQL: INSERT INTO orders (id, order_date, customer_id) VALUES (?, ?, 1)
}
```

---

### Example 3 — `orphanRemoval` Behavior

```java
@Entity
public class Order {
    @OneToMany(mappedBy = "order", 
               cascade = CascadeType.ALL, 
               orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();
}

@Transactional
public void orphanRemovalDemo() {
    Order order = em.find(Order.class, 1L);
    // Assume: order has 3 items [A, B, C]
    
    // Remove one item from collection:
    OrderItem itemToRemove = order.getItems().get(0); // item A
    order.getItems().remove(itemToRemove);
    // orphanRemoval = true → item A is now orphaned
    // At flush: DELETE FROM order_items WHERE id = ? (item A's id)
    
    // Replace entire collection:
    order.setItems(new ArrayList<>()); // items B and C become orphans
    // At flush: DELETE FROM order_items WHERE id=? (B)
    //           DELETE FROM order_items WHERE id=? (C)
    
    // Contrast with just CascadeType.REMOVE (no orphanRemoval):
    // Removing from collection does NOT delete → items become "orphans" 
    // with null FK or remain with old FK → orphanRemoval = true is needed
}
```

---

### Example 4 — `@ManyToMany` Correct Setup with Set

```java
@Entity
public class Article {
    @Id @GeneratedValue Long id;
    String title;
    
    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    // NO CascadeType.REMOVE — never cascade remove on @ManyToMany
    @JoinTable(
        name = "article_tags",
        joinColumns = @JoinColumn(name = "article_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    private Set<Tag> tags = new HashSet<>(); // Set prevents duplicate join table rows
    
    public void addTag(Tag tag) {
        tags.add(tag);
        tag.getArticles().add(this);
    }
    
    public void removeTag(Tag tag) {
        tags.remove(tag);
        tag.getArticles().remove(this);
    }
}

@Entity
public class Tag {
    @Id @GeneratedValue Long id;
    String name;
    
    @ManyToMany(mappedBy = "tags")
    private Set<Article> articles = new HashSet<>();
    
    // equals/hashCode based on 'name' (business key):
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Tag)) return false;
        return Objects.equals(name, ((Tag) o).name);
    }
    @Override public int hashCode() { return Objects.hash(name); }
}

@Transactional
public void manyToManyDemo() {
    Article article = em.find(Article.class, 1L);
    Tag tag = em.find(Tag.class, 5L);
    
    article.addTag(tag);
    // At flush: INSERT INTO article_tags (article_id, tag_id) VALUES (1, 5)
    // One INSERT — not delete-all + reinsert (because we're using @ManyToMany not @ElementCollection)
    
    article.removeTag(tag);
    // At flush: DELETE FROM article_tags WHERE article_id=1 AND tag_id=5
    // One DELETE — targeted removal because Set knows which element was removed
}
```

---

### Example 5 — `@OneToOne` Shared PK with `@MapsId`

```java
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue Long id;
    String username;
    String email;
    
    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL,
              fetch = FetchType.LAZY, optional = true)
    private UserProfile profile; // inverse side — lazy loading unreliable!
}

@Entity
@Table(name = "user_profiles")
public class UserProfile {
    
    @Id
    private Long id; // no @GeneratedValue — shares User's PK
    
    String bio;
    String avatarUrl;
    LocalDate birthDate;
    
    @OneToOne(fetch = FetchType.LAZY)
    @MapsId   // maps User's PK to this entity's @Id
    @JoinColumn(name = "id")
    private User user; // owning side — FK column is the PK itself
    
    public UserProfile(User user, String bio) {
        this.user = user;
        this.bio = bio;
        // id is automatically set via @MapsId
    }
    
    protected UserProfile() {}
}

@Transactional
public void sharedPkDemo() {
    User user = em.find(User.class, 1L);
    
    UserProfile profile = new UserProfile(user, "Java developer");
    em.persist(profile);
    // SQL: INSERT INTO user_profiles (id, bio, avatar_url, birth_date)
    //      VALUES (1, 'Java developer', NULL, NULL)
    //      ↑ id=1 comes from user.id via @MapsId
    
    // Loading profile by user ID:
    UserProfile loaded = em.find(UserProfile.class, 1L);
    // SQL: SELECT * FROM user_profiles WHERE id=1
    // No join needed — shared PK means same value
}
```

---

### Example 6 — Cascade Trap Demonstration

```java
@Entity
public class Department {
    @Id @GeneratedValue Long id;
    String name;
    
    // DANGEROUS cascade ALL on @ManyToMany:
    @ManyToMany(cascade = CascadeType.ALL) // ← WRONG
    private Set<Employee> employees = new HashSet<>();
}

@Transactional
public void cascadeTrap() {
    Department dept = em.find(Department.class, 1L);
    // dept has employees: [Alice, Bob, Charlie]
    // Alice is ALSO in another department
    
    em.remove(dept);
    // CascadeType.REMOVE cascades → removes Alice, Bob, Charlie
    // Alice is deleted even though she's still in another department!
    // → ConstraintViolationException or silent data loss
    
    // CORRECT: no cascade REMOVE on @ManyToMany
    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    private Set<Employee> employees;
    // Now: removing dept only deletes from join table, not the employees themselves
}
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ**
In a bidirectional `@OneToMany` / `@ManyToOne` relationship, which side controls the foreign key column in the database?

A) The `@OneToMany` side  
B) The side with `mappedBy`  
C) The `@ManyToOne` side (the side without `mappedBy`)  
D) Both sides contribute equally  

**Answer: C** — The `@ManyToOne` side is the owning side. It has the `@JoinColumn` and controls the FK write. The `@OneToMany` side with `mappedBy` is the inverse — ignored for writes.

---

**Q2 — Select All That Apply**
You have a bidirectional `@OneToMany(mappedBy="order")` on `Customer.orders` and `@ManyToOne` on `Order.customer`. Which operations cause the `customer_id` FK to be written to the database? (Select all)

A) `customer.getOrders().add(order)`  
B) `order.setCustomer(customer)`  
C) Both A and B together  
D) `em.persist(order)` after setting `order.setCustomer(customer)`  
E) `em.persist(order)` after only adding to `customer.getOrders()`  

**Answer: B, D** — Only the owning side (`Order.customer`) drives the FK write. A alone (inverse side only) → FK is NULL in DB. E alone → FK is NULL in DB. B or C (sets owning side) → FK is written correctly.

---

**Q3 — SQL Prediction**
```java
@Entity public class Author {
    @Id @GeneratedValue Long id;
    
    @OneToMany  // no mappedBy, no @JoinTable
    private List<Book> books = new ArrayList<>();
}
@Entity public class Book {
    @Id @GeneratedValue Long id;
    String title;
}
```
What schema does Hibernate generate?

A) `books` table with `author_id` FK column  
B) Separate `author_books` join table with `author_id` and `books_id` columns  
C) Both tables merged into one  
D) `MappingException` — `@JoinColumn` is required  

**Answer: B** — Unidirectional `@OneToMany` without `mappedBy` and without `@JoinColumn` generates a surprise join table. Hibernate creates `author_books (author_id, books_id)`.

---

**Q4 — Code Output Prediction**
```java
@Transactional
public void test() {
    Customer c = em.find(Customer.class, 1L);
    Order o = new Order();
    o.setOrderDate(LocalDateTime.now());
    
    // Only set inverse side:
    c.getOrders().add(o);
    em.persist(o);
    em.flush();
    
    em.clear();
    Order reloaded = em.find(Order.class, o.getId());
    System.out.println(reloaded.getCustomer());
}
```

A) Prints the `Customer` object  
B) Prints `null` — inverse side was set but FK column is null  
C) Throws `NullPointerException`  
D) Throws `LazyInitializationException`  

**Answer: B** — Only the inverse side (`c.getOrders().add(o)`) was set. The owning side (`o.setCustomer(c)`) was never set. Hibernate reads the owning side for the FK write → `customer_id = NULL`. After reload, `reloaded.getCustomer()` returns null.

---

**Q5 — Select All That Apply**
Which cascade types are safe to use on a `@ManyToMany` relationship? (Select all)

A) `CascadeType.PERSIST`  
B) `CascadeType.MERGE`  
C) `CascadeType.REMOVE`  
D) `CascadeType.REFRESH`  
E) `CascadeType.ALL`  

**Answer: A, B, D** — `CascadeType.REMOVE` and `ALL` (which includes REMOVE) are dangerous on `@ManyToMany` because they delete shared entities that may still be referenced by other owners. `PERSIST`, `MERGE`, and `REFRESH` are generally safe.

---

**Q6 — orphanRemoval Trap**
```java
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderItem> items;

@Transactional
public void replaceItems(Long orderId) {
    Order order = em.find(Order.class, orderId);
    List<OrderItem> newItems = List.of(new OrderItem(...), new OrderItem(...));
    order.setItems(new ArrayList<>(newItems)); // replace entire list
}
```

What SQL is generated at flush?

A) DELETE old items, INSERT new items  
B) UPDATE existing items, INSERT new ones  
C) Only INSERT new items — old ones remain  
D) No SQL — list replacement is not tracked  

**Answer: A** — With `orphanRemoval = true`, replacing the entire collection makes all old items orphans. Hibernate DELETEs all old items. New items (transient) are inserted via `CascadeType.PERSIST` (included in ALL). Result: DELETE all old + INSERT all new.

---

**Q7 — `@OneToOne` Lazy Loading**
```java
@Entity public class User {
    @Id Long id;
    
    @OneToOne(mappedBy = "user", fetch = FetchType.LAZY)
    private UserProfile profile; // inverse side
}

User user = em.find(User.class, 1L);
// How many SQL queries were executed?
```

A) 1 — `User` loaded, `profile` is a proxy (truly lazy)  
B) 2 — `User` loaded, then Hibernate checks if `UserProfile` exists (fires extra SELECT)  
C) 0 — entity is cached  
D) Depends on whether `optional=true` or `optional=false`  

**Answer: B** — For `@OneToOne` inverse side, Hibernate cannot know whether to put a proxy or null without querying the DB. It fires an extra SELECT to check existence. True lazy loading for `@OneToOne` inverse side requires bytecode enhancement or `@LazyToOne(LazyToOneOption.NO_PROXY)` (Hibernate-specific).

---

**Q8 — `@JoinColumn` vs `mappedBy`**
```java
@Entity public class Employee {
    @OneToOne
    @JoinColumn(name = "parking_spot_id")
    private ParkingSpot spot;
}

@Entity public class ParkingSpot {
    @OneToOne(mappedBy = "spot")
    private Employee employee;
}
```
Which table contains the `parking_spot_id` column?

A) `parking_spot` table  
B) `employee` table  
C) A separate join table  
D) Both tables  

**Answer: B** — `@JoinColumn` on `Employee.spot` means the FK column (`parking_spot_id`) lives in the `employee` table. The owning side always holds the FK column.

---

**Q9 — Fetch Default Trap**
```java
@Entity public class Invoice {
    @Id Long id;
    
    @ManyToOne  // no fetch specified
    private Customer customer;
    
    @OneToMany(mappedBy = "invoice")  // no fetch specified
    private List<LineItem> items;
}

Invoice inv = em.find(Invoice.class, 1L);
// How many SQL queries are executed?
```

A) 1 — everything is lazy by default  
B) 2 — `Invoice` + `Customer` (EAGER default for `@ManyToOne`)  
C) 3 — `Invoice` + `Customer` + `LineItem`s  
D) 1 + N for line items  

**Answer: B** — `@ManyToOne` default is `EAGER` → Customer is loaded immediately with Invoice (either via JOIN or separate SELECT depending on Hibernate version). `@OneToMany` default is `LAZY` → items are NOT loaded. Result: 2 queries (Invoice + Customer join or separate SELECT).

---

## 4️⃣ Trick Analysis

**The inverse side is completely ignored for writes (Q2, Q4)**:
This is the most important JPA fact to internalize. Setting only `customer.getOrders().add(order)` does nothing to the database. The `mappedBy` annotation is Hibernate's signal: "ignore this side for FK writes." The FK column value comes exclusively from the owning side field value at flush time.

**Unidirectional `@OneToMany` join table surprise (Q3)**:
JPA spec mandates that `@OneToMany` without `mappedBy` uses a join table by default. This surprises every developer who writes `@OneToMany List<Order> orders` thinking they get a simple FK. The solution: always use `mappedBy` for bidirectional, or add `@JoinColumn` for unidirectional FK-based (with the understanding of its INSERT/UPDATE inefficiency).

**`orphanRemoval` on collection replacement (Q6)**:
Developers expect `order.setItems(newList)` to just swap references. With `orphanRemoval = true`, Hibernate monitors what was in the collection and what left. Replacing the collection with a new list instance means everything in the old list is gone → orphaned → deleted. This is a common source of accidental mass deletes.

**`@OneToOne` inverse lazy loading failure (Q7)**:
This is a genuine Hibernate limitation. The proxy mechanism works by knowing what to proxy — for a proxy to exist, Hibernate must know the associated entity exists. For `@ManyToOne` and `@OneToMany`, Hibernate knows from the FK column value. For `@OneToOne` inverse side, there's no FK column on this side → Hibernate must query to check → defeats lazy loading. Bytecode enhancement is the only true fix.

**`@ManyToOne` EAGER default (Q9)**:
JPA spec chose EAGER as the default for `@ManyToOne` and `@OneToOne`, probably reasoning that loading a single related entity is cheap. This reasoning fails at scale: 100 orders → 100 eager customer loads. **Always specify `fetch = FetchType.LAZY` explicitly on every `@ManyToOne` and `@OneToOne`.**

**Cascade REMOVE on `@ManyToMany` (Q5)**:
The reason it's dangerous: entities in a `@ManyToMany` are shared. A `Tag` may belong to 100 `Article`s. If you cascade REMOVE from one Article, the Tag is deleted, breaking the other 99 Articles. The join table row is deleted (correct), but the Tag entity itself is also deleted (catastrophic). Only cascade PERSIST/MERGE on `@ManyToMany` — never REMOVE.

---

## 5️⃣ Summary Sheet

### Owning Side vs Inverse Side — Complete Reference

```
Association          Owning Side              Inverse Side
──────────────────────────────────────────────────────────────
@ManyToOne /         @ManyToOne (always)      @OneToMany + mappedBy
@OneToMany

@OneToOne            Side with @JoinColumn    Side with mappedBy
(bidirectional)

@ManyToMany          Side with @JoinTable     Side with mappedBy
(bidirectional)      (your choice)

Rules:
  - Owning side: NO mappedBy, HAS @JoinColumn or @JoinTable
  - Inverse side: HAS mappedBy = "fieldNameOnOwningSide"
  - Hibernate writes FK based on owning side ONLY
  - Both sides must be set in Java for in-memory consistency
```

### Fetch Type Defaults and Recommendations

```
Association     JPA Default    Recommendation    Risk if EAGER
──────────────────────────────────────────────────────────────
@ManyToOne      EAGER          LAZY              N+1 queries
@OneToOne       EAGER          LAZY              N+1 queries
@OneToMany      LAZY           LAZY (keep)       Cartesian product
@ManyToMany     LAZY           LAZY (keep)       Cartesian product
```

### Cascade Type Safety Matrix

```
CascadeType     @OneToMany    @ManyToOne    @OneToMany    @ManyToMany
                (parent→      (child→       @OneToOne
                children)     parent)

PERSIST         ✅ Common     ⚠️ Rare       ✅ Common     ✅ OK
MERGE           ✅ Common     ⚠️ Rare       ✅ Common     ✅ OK
REMOVE          ✅ OK         ❌ Dangerous  ✅ OK         ❌ Dangerous
REFRESH         ✅ OK         ✅ OK         ✅ OK         ✅ OK
DETACH          ✅ OK         ✅ OK         ✅ OK         ✅ OK
ALL             ✅ Common     ❌ Dangerous  ✅ OK         ❌ Dangerous
```

### `orphanRemoval` vs `CascadeType.REMOVE`

```
CascadeType.REMOVE:
  Triggered by:    em.remove(parentEntity)
  Effect:          Child is removed from DB
  Collection op:   Does NOT trigger on collection.remove(child)

orphanRemoval=true:
  Triggered by:    child removed from collection OR collection replaced
  Effect:          Child is removed from DB (DELETE)
  em.remove:       Also works (subsumes REMOVE behavior)

Combined (cascade=ALL, orphanRemoval=true):
  Most complete — handles all scenarios
  Use for: fully-owned, lifecycle-dependent children (@OneToMany parent-child)
  Avoid for: @ManyToMany (shared entities)
```

### Association Mapping Decision Flow

```
Relationship between A and B:

Many A → One B?
  → @ManyToOne on A (owning), @OneToMany(mappedBy) on B (inverse)

One A → One B?
  → @OneToOne; FK side = owning; consider @MapsId for shared PK

Many A ↔ Many B?
  → @ManyToMany; choose owning side; use Set<> not List<>
  → Need extra columns on join? → Use intermediate @Entity

Extra join table columns needed?
  → Replace @ManyToMany with intermediate @Entity + two @ManyToOne

@JoinColumn key rules:
  name     = FK column name in OWNING entity's table
  nullable = false for mandatory relationships
  unique   = true for @OneToOne
```

### SQL Generated per Association Operation

```
@ManyToOne owning side set:
  persist(child) → INSERT INTO child_table (..., fk_col) VALUES (..., ?)

@OneToMany inverse side only set:
  persist(child) → INSERT INTO child_table (..., fk_col) VALUES (..., NULL) ← BUG

@ManyToMany add:
  Set: individual INSERT INTO join_table
  List with @ManyToMany: DELETE all + INSERT all (like @ElementCollection)

orphanRemoval triggered:
  DELETE FROM child_table WHERE id = ? (per orphaned child)

CascadeType.REMOVE:
  em.remove(parent) → DELETE children first, then parent
  (respects FK constraint order)
```

---
