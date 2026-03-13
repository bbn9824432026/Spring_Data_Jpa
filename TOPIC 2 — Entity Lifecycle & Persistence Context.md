# 🏛️ TOPIC 2 — Entity Lifecycle & Persistence Context

---

## 1️⃣ Conceptual Explanation

### The Persistence Context — The Heart of JPA

The Persistence Context is the single most important concept in JPA. Every bug related to `LazyInitializationException`, unexpected UPDATE statements, stale data, detached entity exceptions, and identity confusion traces back to a misunderstanding of the Persistence Context.

**Definition**: The Persistence Context is a **first-level cache and identity map** that tracks all managed entity instances within a given scope. It is the runtime environment in which entities live, breathe, and die.

In Hibernate's implementation, the Persistence Context is `StatefulPersistenceContext` — a class that maintains:

```
StatefulPersistenceContext contains:
  ├── entitiesByKey: Map<EntityKey, Object>         ← identity map (PK → entity instance)
  ├── entitiesByUniqueKey: Map<EntityUniqueKey, Object> ← unique constraint cache
  ├── entitySnapshotsByKey: Map<EntityKey, Object[]>  ← deep copy of state at load time
  ├── collectionsByKey: Map<CollectionKey, PersistentCollection> ← tracked collections
  ├── proxiesByKey: Map<EntityKey, Object>           ← Hibernate proxies
  └── actionQueue: ActionQueue                       ← pending SQL operations
```

---

### The Four Entity States

This is the core state machine every JPA developer must internalize completely.

```
                    ┌─────────────────────────────────────┐
                    │           TRANSIENT                  │
                    │  new MyEntity()                      │
                    │  - Not known to any EntityManager    │
                    │  - No database row                   │
                    │  - No ID (or ID manually set)        │
                    └──────────────┬──────────────────────┘
                                   │ em.persist()
                                   ▼
                    ┌─────────────────────────────────────┐
         ┌─────────►│            MANAGED                   │◄──────────────┐
         │          │  - Tracked by Persistence Context    │               │
         │          │  - Dirty checking active             │               │
         │          │  - Changes auto-flushed on commit    │               │
         │          └──┬──────────────┬───────────────────┘               │
         │             │              │                                    │
         │   em.merge()│  em.detach() │ context close /                   │
         │             │  evict()     │ clear() /                         │
         │             │              │ serialization /                    │
         │             ▼              ▼ end of transaction                 │
         │          ┌────────────────────────────────────┐                │
         │          │           DETACHED                  │                │
         │          │  - Was managed, no longer is        │────────────────┘
         │          │  - Has an ID and DB row             │  em.merge()
         │          │  - Changes NOT tracked              │
         │          └────────────────────────────────────┘
                    
         MANAGED state also transitions to:
         
         em.remove() → REMOVED state:
         ┌────────────────────────────────────┐
         │            REMOVED                  │
         │  - Scheduled for deletion           │
         │  - Still in Persistence Context     │
         │  - DELETE SQL on flush/commit       │
         └────────────────────────────────────┘
```

---

### State 1 — TRANSIENT

A transient entity is a plain Java object that has never been associated with any Persistence Context and has no corresponding database row.

```java
Product p = new Product(); // TRANSIENT
p.setName("Widget");
// At this point: no ID, not tracked, Hibernate has zero knowledge of it
// GC will collect it normally if no reference held
```

**Internal reality**: Hibernate has absolutely no awareness of transient objects. They are ordinary Java heap objects. No proxy, no snapshot, no identity map entry.

**Trap**: If you set an ID manually on a transient object, it is still transient. The state is determined by Persistence Context association, not by whether an ID field is set.

---

### State 2 — MANAGED

A managed entity is associated with an open Persistence Context. Hibernate watches it.

**How an entity becomes managed:**
- `em.persist(transientEntity)` — for new entities
- `em.find(Entity.class, id)` — loads from DB, returns managed
- `em.merge(detachedEntity)` — returns a NEW managed instance (critical distinction)
- JPQL query result — results are always managed within the current PC
- Navigation through managed entity associations — traversal can load and manage related entities

**What Hibernate does with managed entities:**

1. **Identity Map Registration**: The entity is stored in `entitiesByKey` under its `EntityKey` (type + primary key).

2. **Snapshot Creation**: Hibernate captures a **deep copy** of the entity's state at load time into `entitySnapshotsByKey`. This is called the "loaded state."

3. **Dirty Checking**: At flush time, Hibernate compares current field values against the snapshot. If any field differs → generates UPDATE SQL.

**The Dirty Checking Engine** — This is a critical internal mechanism:

```
Flush triggered (commit / explicit flush / pre-query):
  For each managed entity in identity map:
    currentState = extract current field values via reflection / bytecode
    loadedState  = retrieve snapshot from entitySnapshotsByKey
    For each field:
      if currentState[i] != loadedState[i]:  // uses TypeDescriptor.equals()
        mark entity as dirty
    If dirty:
      enqueue EntityUpdateAction in ActionQueue
  Execute ActionQueue → SQL
```

The default dirty checking mechanism uses **reflection** to read all field values. This is O(n) per entity per flush for every managed entity in the context — a key performance concern with large Persistence Contexts.

Hibernate 5+ introduced **bytecode enhancement** for dirty checking — instead of comparing snapshots, enhanced entities track their own dirty fields via injected `$$_hibernate_tracker` fields. This reduces flush cost significantly.

---

### State 3 — DETACHED

A detached entity was previously managed but its Persistence Context has closed or it was explicitly detached. It has:
- A valid primary key
- A corresponding database row (unless deleted)
- **NO connection to any Persistence Context** — changes are NOT tracked

**How an entity becomes detached:**
- Persistence Context closes (`em.close()` — end of transaction in most Spring setups)
- `em.detach(entity)` — explicit detachment
- `em.clear()` — detaches ALL entities in the context
- `em.evict(entity)` — Hibernate-specific, same as detach
- Serialization and deserialization (the deserialized copy is detached)
- Passing an entity across transaction boundaries (very common in layered architectures)

**The critical detached entity trap** in Spring:

```java
@Service
public class OrderService {
    
    @Transactional
    public Order loadOrder(Long id) {
        return orderRepository.findById(id).get();
        // Transaction commits here, EntityManager closes
        // Order is now DETACHED
    }
    
    public void processOrder(Long id) {
        Order order = loadOrder(id); // returns DETACHED entity
        
        // This line throws LazyInitializationException if items is LAZY:
        order.getItems().size(); // ← No Persistence Context alive!
    }
}
```

**Merging detached entities** — `em.merge(detachedEntity)`:
- Hibernate checks if the identity map already has an entity with the same key
- If yes: copies detached state into the existing managed instance, returns it
- If no: loads from database, copies detached state into it, returns managed instance
- **The detached instance itself NEVER becomes managed** — merge returns a NEW managed instance

This is the #1 `merge()` trap on exams:

```java
Product detached = ...; // detached entity
Product managed = em.merge(detached);

// detached is still DETACHED
// managed is the managed copy

em.persist(managed); // correct
em.persist(detached); // EntityExistsException or no-op depending on state
```

---

### State 4 — REMOVED

A removed entity is scheduled for deletion. It is still in the Persistence Context but marked for DELETE.

```java
Product p = em.find(Product.class, 1L); // MANAGED
em.remove(p); // REMOVED — DELETE enqueued in ActionQueue
// p is still in the identity map but marked as deleted
// Until flush: no SQL executed
// At flush: DELETE FROM products WHERE id = 1

// After flush/commit: p becomes TRANSIENT (no longer in PC, no DB row)
```

**Trap**: Calling `em.persist(removedEntity)` before flush reschedules it for insert — cancels the remove. This is a valid but confusing operation.

---

### `persist()` Internal Mechanics

```java
em.persist(entity)
```

Hibernate's internal sequence:
1. Check entity is not already in removed state
2. Check for `@Id` — is it generated or manually assigned?
3. If SEQUENCE/TABLE generator: may fetch next sequence value NOW (pre-insert ID allocation)
4. If IDENTITY generator: INSERT must happen immediately (can't batch — breaks ActionQueue batching)
5. Register entity in identity map
6. Create loaded state snapshot
7. Enqueue `EntityInsertAction` in ActionQueue
8. Cascade `CascadeType.PERSIST` to associated entities

**Critical**: `persist()` does NOT immediately execute SQL (except IDENTITY strategy — see Topic 24). The INSERT is deferred to flush time.

---

### `merge()` Internal Mechanics

`merge()` is the most complex operation in JPA. Full internal sequence:

```
em.merge(entity):

1. Is entity null? → IllegalArgumentException
2. Is entity REMOVED? → IllegalArgumentException  
3. Is entity MANAGED? → return entity as-is (no-op, already tracked)
4. Is entity DETACHED (has ID)?
   a. Check identity map: is there already a managed instance with same key?
      - YES: copy detached state onto managed instance → return managed
      - NO:  load from DB (SELECT), copy detached state onto loaded → return managed
5. Is entity TRANSIENT (no ID or new entity)?
   → treat like persist: create new managed copy, enqueue INSERT
6. Cascade CascadeType.MERGE to associations
```

---

### `remove()` Internal Mechanics

```java
em.remove(entity)
```

1. Entity MUST be managed — if detached → `IllegalArgumentException`
2. Mark entity as deleted in identity map
3. Enqueue `EntityDeleteAction`
4. Cascade `CascadeType.REMOVE` to associations

**Trap**: You cannot `remove()` a detached entity directly. You must first merge it (to get a managed instance) then remove:

```java
Product detached = ...;
Product managed = em.merge(detached);
em.remove(managed); // correct

em.remove(detached); // IllegalArgumentException
```

---

### `refresh()` Internal Mechanics

```java
em.refresh(entity)
```

1. Entity must be managed
2. Hibernate executes SELECT against the database
3. Overwrites current in-memory state with database state
4. Discards all pending changes to this entity
5. Re-creates loaded state snapshot from fresh DB data

Use case: when another process has modified the database row and you need the latest state, discarding your in-memory changes.

---

### Flush Modes — When Does SQL Execute?

Flush is the process of synchronizing the Persistence Context with the database — translating all pending ActionQueue operations into SQL.

| FlushMode | When SQL is executed |
|---|---|
| `AUTO` (default) | Before each query execution (to ensure query sees latest state) + at commit |
| `COMMIT` | Only at transaction commit |
| `ALWAYS` | Before every query, always |
| `MANUAL` | Only when `em.flush()` is explicitly called |

**`FlushMode.AUTO` deep behavior** — this is subtle and exam-critical:

With `AUTO`, before Hibernate executes a JPQL query, it checks: "does any pending `EntityInsertAction` or `EntityUpdateAction` affect the tables queried?" If yes → flush first (so the query sees the new data). If no → skip flush.

```java
@Transactional
public void example() {
    Product p = new Product("Widget");
    em.persist(p); // INSERT enqueued, not yet executed
    
    // FlushMode.AUTO: Hibernate detects pending INSERT on products table
    // before this query touches products table → flush first → INSERT executes
    List<Product> all = em.createQuery("FROM Product", Product.class).getResultList();
    // p is now in the result list
}
```

With `FlushMode.COMMIT`:
```java
@Transactional
public void example() {
    Product p = new Product("Widget");
    em.persist(p); // INSERT enqueued
    
    // FlushMode.COMMIT: NO pre-query flush
    List<Product> all = em.createQuery("FROM Product", Product.class).getResultList();
    // p is NOT in the result list (INSERT not yet sent to DB)
    // p IS included only because the identity map returns it directly for findById
    
    // At commit: INSERT executes
}
```

`readOnly=true` transactions in Spring set `FlushMode.MANUAL` on the Session — this means Hibernate never flushes, which also disables dirty checking. This is the performance benefit of `@Transactional(readOnly=true)`.

---

### Extended vs Transaction-Scoped Persistence Context

**Transaction-Scoped** (default in Spring):
- PC lives exactly as long as the transaction
- Created when transaction begins, destroyed when transaction ends
- All loaded entities become DETACHED after method returns

**Extended Persistence Context** (rare in Spring, common in EJB):
- PC survives transaction boundaries
- Entity stays MANAGED across multiple transactions
- Used with `@PersistenceContext(type = PersistenceContextType.EXTENDED)`
- In Spring: only in `@Stateful`-equivalent patterns (not common)
- Memory risk: PC grows indefinitely, never cleared

```java
// Transaction-scoped (Spring default)
@PersistenceContext
private EntityManager em; // proxy — new PC per transaction

// Extended — Spring requires special scope
@PersistenceContext(type = PersistenceContextType.EXTENDED)
private EntityManager em; // same PC across transactions
```

---

### Identity Map Pattern — Entity Identity Guarantee

The identity map guarantees: **within one Persistence Context, for a given primary key, there is exactly one entity instance**.

```java
@Transactional
public void identityGuarantee() {
    Product p1 = em.find(Product.class, 1L); // SELECT → creates instance, puts in map
    Product p2 = em.find(Product.class, 1L); // Cache HIT → returns SAME instance
    
    System.out.println(p1 == p2); // TRUE — same Java object reference
    // Second find() generates ZERO SQL queries
}
```

This has profound implications:
- No duplicate instances within one PC
- Thread safety is NOT provided by this — PCs are NOT thread-safe
- The identity map is PER EntityManager — two different EMs can have two different instances of the same DB row

---

### The ActionQueue — Deferred SQL Execution

All state changes are deferred into the `ActionQueue`:

```
ActionQueue contents (in execution order):
  1. OrphanRemovalAction       ← orphan collection elements
  2. EntityInsertAction        ← persist() calls
  3. EntityUpdateAction        ← dirty entity changes
  4. CollectionRemoveAction    ← removed collection elements
  5. CollectionUpdateAction    ← updated collections
  6. CollectionRecreateAction  ← recreated collections
  7. EntityDeleteAction        ← remove() calls
```

The ordering matters for foreign key constraints. Hibernate inserts parents before children, deletes children before parents. This ordering is managed automatically but can be overridden with `@ConstraintMode`.

---

## 2️⃣ Code Examples

### Example 1 — All Four States Demonstrated

```java
@Service
public class EntityStateDemo {
    
    @PersistenceContext
    private EntityManager em;
    
    @Transactional
    public void demonstrateStates() {
        
        // ── STATE 1: TRANSIENT ──────────────────────────────
        Product p = new Product();
        p.setName("Widget");
        // Not in identity map. Not tracked. Just a Java object.
        
        // ── STATE 2: MANAGED ───────────────────────────────
        em.persist(p);
        // Now: in identity map, snapshot created, INSERT enqueued
        // p.getId() is populated IF using SEQUENCE generator
        // p.getId() is null still IF using IDENTITY generator (flush not yet happened)
        
        p.setName("Super Widget"); // dirty — Hibernate will detect this at flush
        // No SQL yet. Just a field mutation on a tracked object.
        
        // ── FLUSH ───────────────────────────────────────────
        em.flush();
        // Now: INSERT executed (Product), then UPDATE executed (name change)
        // SQL: INSERT INTO products (name) VALUES ('Widget')
        // SQL: UPDATE products SET name='Super Widget' WHERE id=1
        
        // ── STATE 4: REMOVED ───────────────────────────────
        em.remove(p);
        // DELETE enqueued. p is still in PC but marked deleted.
        
        em.flush();
        // SQL: DELETE FROM products WHERE id=1
        
        // After transaction commits: p is TRANSIENT again
    }
    
    @Transactional
    public Product loadManaged(Long id) {
        return em.find(Product.class, id); // MANAGED within this transaction
    }
    
    // Called outside @Transactional:
    public void workWithDetached(Long id) {
        Product p = loadManaged(id); // returns DETACHED (transaction ended)
        
        p.setName("Changed"); // NOT tracked — no PC alive
        // This change will NEVER reach the database unless you merge
        
        // ── STATE 3: DETACHED → RE-MANAGED via merge ────────
        reattach(p);
    }
    
    @Transactional
    public void reattach(Product detached) {
        Product managed = em.merge(detached);
        // managed is the tracked copy
        // SQL: SELECT to load current DB state, then copy detached.name onto it
        // At commit: UPDATE executes if state differs from DB
        
        // detached is STILL detached — this is the key trap
        System.out.println(em.contains(detached)); // FALSE
        System.out.println(em.contains(managed));  // TRUE
    }
}
```

---

### Example 2 — Dirty Checking in Action

```java
@Transactional
public void dirtyCheckingDemo() {
    // SQL: SELECT * FROM products WHERE id=1
    Product p = em.find(Product.class, 1L);
    // Snapshot: {name="Widget", price=9.99, stock=100}
    
    p.setPrice(14.99);
    // No SQL yet. Just a Java mutation.
    
    // At commit, Hibernate compares:
    // current:  {name="Widget", price=14.99, stock=100}
    // snapshot: {name="Widget", price=9.99,  stock=100}
    // Dirty fields: [price]
    // SQL: UPDATE products SET price=14.99 WHERE id=1
    
    // If no changes made:
    // current == snapshot → NO UPDATE SQL generated
}

@Transactional
public void noUpdateIfUnchanged() {
    Product p = em.find(Product.class, 1L);
    // Make no changes
    // Commit: zero SQL executed for this entity — just the SELECT
}
```

---

### Example 3 — Identity Map Guarantee

```java
@Transactional
public void identityMapDemo() {
    Product a = em.find(Product.class, 1L);
    // SQL: SELECT * FROM products WHERE id=1
    
    Product b = em.find(Product.class, 1L);
    // NO SQL — identity map hit
    
    Product c = em.createQuery(
        "SELECT p FROM Product p WHERE p.id = 1", Product.class)
        .getSingleResult();
    // SQL executes (JPQL queries bypass identity map check for execution)
    // But after results are loaded, Hibernate checks identity map:
    // "Already have Product with id=1?" → YES → returns existing instance
    
    System.out.println(a == b); // TRUE
    System.out.println(a == c); // TRUE — same instance!
}
```

---

### Example 4 — Flush Mode Traps

```java
@Transactional
public void flushModeTrap() {
    Product p = new Product("Widget");
    em.persist(p); // INSERT enqueued
    
    // DEFAULT FlushMode.AUTO:
    // Hibernate sees query will touch 'products' table
    // Flushes BEFORE executing query → INSERT happens first
    long count = em.createQuery("SELECT COUNT(p) FROM Product p", Long.class)
                   .getSingleResult();
    // count includes p ✓
    
    // Now with COMMIT mode:
    em.setFlushMode(FlushModeType.COMMIT);
    
    Product p2 = new Product("Gadget");
    em.persist(p2);
    
    long count2 = em.createQuery("SELECT COUNT(p) FROM Product p", Long.class)
                    .getSingleResult();
    // count2 does NOT include p2 — INSERT not yet flushed
    // p2 is in identity map but NOT in DB yet
}
```

---

### Example 5 — The LazyInitializationException Scenario

```java
// This is the single most common Spring Data JPA production bug:

@Service
public class OrderService {
    
    @Transactional
    public Order findOrder(Long id) {
        return orderRepository.findById(id).orElseThrow();
        // Transaction commits here, EntityManager closed
        // Order entity becomes DETACHED
        // order.items is a Hibernate proxy, NOT initialized
    }
}

@RestController
public class OrderController {
    
    @Autowired OrderService orderService;
    
    @GetMapping("/orders/{id}")
    public OrderDto getOrder(@PathVariable Long id) {
        Order order = orderService.findOrder(id);
        // order is DETACHED here
        
        // BOOM: LazyInitializationException
        // "could not initialize proxy - no Session"
        order.getItems().forEach(item -> ...);
    }
}
```

Solutions (covered in depth in Topics 7, 8):
- `@Transactional` on controller (anti-pattern)
- `JOIN FETCH` in repository query
- `@EntityGraph`
- DTO projection
- `spring.jpa.open-in-view=true` (the Open Session in View pattern — an anti-pattern that masks the problem)

---

### Example 6 — merge() vs persist() Identity Trap

```java
@Transactional
public void mergeIdentityTrap() {
    // Simulating a detached entity (e.g., came from another transaction)
    Product detached = new Product();
    detached.setId(1L); // manually set ID — simulates detached state
    detached.setName("Updated");
    
    // Option A: persist()
    // em.persist(detached);
    // → EntityExistsException if row with id=1 exists
    // → Or: INSERT attempt → DB constraint violation
    
    // Option B: merge() — correct for detached entities
    Product managed = em.merge(detached);
    // SQL: SELECT * FROM products WHERE id=1 (loads current state)
    // Then: copies detached.name onto loaded entity
    // managed is the tracked copy
    // At commit: UPDATE products SET name='Updated' WHERE id=1
    
    System.out.println(detached == managed); // FALSE — different instances
    System.out.println(em.contains(detached)); // FALSE
    System.out.println(em.contains(managed));  // TRUE
}
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ**
What happens when you call `em.find(Product.class, 1L)` twice within the same transaction?

A) Two SELECT queries are executed; two separate instances are returned  
B) One SELECT query is executed; the same instance is returned both times  
C) One SELECT query is executed; two separate but equal instances are returned  
D) Two SELECT queries are executed; the same instance is returned both times  

**Answer: B** — Identity map returns the cached instance on the second call. Zero SQL for the second call.

---

**Q2 — Select All That Apply**
Which operations cause an entity to transition to the MANAGED state? (Select all)

A) `em.persist(transientEntity)`  
B) `em.merge(detachedEntity)` — the detached instance itself  
C) `em.merge(detachedEntity)` — the returned value  
D) `em.find(Entity.class, id)` result  
E) JPQL query results  
F) `em.detach(entity)`  

**Answer: A, C, D, E**
- B is wrong: the detached instance stays detached; only the RETURN VALUE of merge is managed
- F is wrong: detach causes transition TO detached, not to managed

---

**Q3 — Code Output Prediction**
```java
@Transactional
public void test() {
    Product p = em.find(Product.class, 1L); // name="Widget" in DB
    p.setName("Gadget");
    em.refresh(p);
    System.out.println(p.getName());
}
```
What is printed?

A) "Gadget"  
B) "Widget"  
C) Throws exception — cannot refresh after mutation  
D) null  

**Answer: B** — `refresh()` re-reads from database, overwriting the in-memory change. "Widget" is printed.

---

**Q4 — Scenario-Based**
```java
@Transactional
public void scenario() {
    Product p = new Product("Widget");
    em.persist(p);
    em.remove(p); // remove immediately after persist
    // What SQL is executed at flush?
}
```

A) INSERT then DELETE  
B) No SQL — ActionQueue optimizes away the insert+delete  
C) Just DELETE  
D) Exception — cannot remove a newly persisted entity  

**Answer: B** — Hibernate's ActionQueue is smart enough to cancel paired INSERT+DELETE for the same entity within the same flush cycle. No SQL is executed (for SEQUENCE-generated IDs; IDENTITY may behave differently).

---

**Q5 — Transaction Boundary Trap**
```java
@Service
public class ProductService {
    
    @Transactional
    public Product loadProduct(Long id) {
        return productRepository.findById(id).get();
    }
    
    public void updateProduct(Long id, String newName) {
        Product p = loadProduct(id); // transaction ended after this
        p.setName(newName);
        // Does this update reach the database?
    }
}
```

A) Yes — Spring tracks all entity mutations regardless of transaction  
B) No — entity is detached after `loadProduct` returns; change is lost  
C) Yes — but only at application shutdown  
D) Yes — Spring saves changes via `@Transactional` on `updateProduct`  
E) Exception is thrown when setting name on detached entity  

**Answer: B** — `loadProduct` has its own `@Transactional` which commits when it returns. The returned entity is detached. `setName()` works (no exception) but the change is never persisted since there's no active PC tracking it.

---

**Q6 — FlushMode Trap**
```java
@Transactional
public void flushModeTest() {
    em.setFlushMode(FlushModeType.COMMIT);
    
    Category c = new Category("Electronics");
    em.persist(c);
    
    List<Category> result = em.createQuery("FROM Category", Category.class)
                               .getResultList();
    
    System.out.println(result.contains(c));
}
```

What is printed?

A) true — identity map guarantees c is in results  
B) false — COMMIT flush mode means INSERT hasn't happened yet when query runs  
C) true — Spring always flushes before queries  
D) Exception — cannot query with uncommitted data  

**Answer: B** — With `FlushMode.COMMIT`, Hibernate does NOT flush before the query. The INSERT for `c` has not reached the database. The JPQL query returns only rows that exist in DB → `c` is not in results.

Note: `c` IS in the identity map, but `List.contains()` uses `equals()`, not identity map lookup. And the query result is built from DB rows, not the identity map (though Hibernate re-uses existing identity map instances for matching rows).

---

**Q7 — SQL Prediction**
```java
@Transactional
public void sqlPrediction() {
    Product p = em.find(Product.class, 1L);
    p.setName("A");
    p.setName("B");
    p.setName("C");
    // Transaction commits
}
```
How many UPDATE statements are executed?

A) 3 — one per setName() call  
B) 2 — Hibernate batches consecutive updates  
C) 1 — dirty checking compares initial snapshot to final state  
D) 0 — name field needs @Column(updatable=true) to trigger UPDATE  

**Answer: C** — Dirty checking compares the **snapshot** (name="original") against the **final current state** (name="C"). The intermediate mutations to "A" and "B" are irrelevant. One UPDATE: `SET name='C'`. This is why dirty checking is "end-state" based, not "event" based.

---

**Q8 — remove() Trap**
```java
@Transactional
public void removeTrap() {
    Product detached = new Product();
    detached.setId(5L);
    // This entity was loaded in a previous transaction — now detached
    
    em.remove(detached);
}
```

What happens?

A) DELETE FROM products WHERE id=5 is executed  
B) `IllegalArgumentException` — cannot remove detached entity  
C) Entity is removed from identity map only, no DB DELETE  
D) `EntityNotFoundException`  

**Answer: B** — `em.remove()` requires a MANAGED entity. Passing a detached entity (even one with a valid ID) throws `IllegalArgumentException`. You must `merge()` first to get a managed instance, then `remove()`.

---

**Q9 — Extended Persistence Context**
```java
@PersistenceContext(type = PersistenceContextType.EXTENDED)
private EntityManager em;

public Product load(Long id) {
    return em.find(Product.class, id);
}

public void update(Product p, String name) {
    p.setName(name);
    // No @Transactional here
}
```

In a bean with extended PC, what happens when `update()` is called?

A) `LazyInitializationException`  
B) Change is tracked; persisted on next transaction  
C) `TransactionRequiredException`  
D) Change is silently discarded  

**Answer: B** — Extended PC survives across transaction boundaries. The entity remains MANAGED. The change is tracked and will be flushed on the next transaction that touches this EntityManager.

---

## 4️⃣ Trick Analysis

**The `merge()` return value trap** (Q2, Q6 in Examples):
The most common misconception. `merge()` never promotes the passed object to managed state. It returns a different object (or the same object if it was already managed). Code that calls `merge(detached)` and then uses `detached` thinking it's now tracked is silently broken.

**The dirty checking "final state" trap** (Q7):
Developers assume Hibernate fires an UPDATE for every `set()` call. Hibernate has zero event listeners on setters (unless using bytecode enhancement with `@DynamicUpdate`). It compares snapshots at flush time. Three `setName()` calls produce one UPDATE reflecting only the final value.

**The `FlushMode.COMMIT` + query visibility trap** (Q6):
Persisted but not-yet-flushed entities are NOT visible to JPQL queries with `COMMIT` flush mode. They ARE accessible via `em.find()` and identity map hits. This asymmetry (JPQL doesn't see them, but findById does) is a genuine source of bugs.

**The `em.refresh()` destroys changes trap** (Q3):
Developers call `refresh()` thinking it merges DB state with in-memory state. It doesn't — it overwrites completely. Any unsaved changes are lost.

**The ActionQueue INSERT+DELETE optimization** (Q4):
A niche but exam-tested behavior. Hibernate recognizes that persisting and then removing the same entity within one flush cycle is a no-op and emits no SQL. This only applies reliably for SEQUENCE-based IDs (IDENTITY strategy has to INSERT immediately to get the generated ID).

**The self-invocation transaction trap** (Q5 extended thinking):
If `updateProduct()` had `@Transactional`, calling it from the same bean without going through a proxy would bypass the transaction. But the Q5 scenario is simpler — `updateProduct` has no `@Transactional`, so no transaction is started.

**Detached entity exception confusion**:
`em.remove(detached)` → `IllegalArgumentException`
`em.persist(detached that has ID)` → potentially `EntityExistsException`
`em.refresh(detached)` → `IllegalArgumentException` or `EntityNotFoundException`
These are different exceptions for different operations on detached entities.

---

## 5️⃣ Summary Sheet

### Entity State Transition Rules

```
new Object()           → TRANSIENT
persist(T)             → MANAGED         (T: transient)
persist(D)             → EntityExistsException (D: detached with ID)
merge(T)               → MANAGED copy    (T stays transient)
merge(D)               → MANAGED copy    (D stays detached!)
merge(M)               → M (no-op)       (M: already managed)
remove(M)              → REMOVED
remove(D)              → IllegalArgumentException
refresh(M)             → MANAGED (re-read from DB)
refresh(D)             → IllegalArgumentException
detach(M)              → DETACHED
clear()                → ALL → DETACHED
close()                → ALL → DETACHED
find(id) → hit DB      → MANAGED
JPQL result            → MANAGED
```

### Dirty Checking Mechanics

```
Snapshot captured at:    em.find() / em.persist() / em.merge() result / JPQL result
Comparison happens at:   flush() / commit / pre-query (FlushMode.AUTO)
What's compared:         snapshot vs current field values
What's generated:        UPDATE only for changed fields (with @DynamicUpdate)
                         or UPDATE all fields (default Hibernate behavior)
```

### Flush Mode Comparison Table

| Mode | Pre-query flush | Pre-commit flush | When to use |
|---|---|---|---|
| `AUTO` | Yes (if needed) | Yes | Default — safest |
| `COMMIT` | No | Yes | Performance optimization — risky |
| `ALWAYS` | Always | Yes | Aggressive — rarely needed |
| `MANUAL` | No | No | Batch processing, read-only |

Note: `@Transactional(readOnly=true)` sets `MANUAL` → disables dirty checking entirely.

### State Behavior Matrix

| Operation | Transient | Managed | Detached | Removed |
|---|---|---|---|---|
| `persist()` | → Managed | No-op | EntityExistsException | reschedules |
| `merge()` | → Managed copy | Returns self | → Managed copy | IllegalArgument |
| `remove()` | IllegalArgument | → Removed | IllegalArgument | No-op |
| `refresh()` | IllegalArgument | Re-reads DB | IllegalArgument | IllegalArgument |
| `detach()` | No-op | → Detached | No-op | → Detached |
| `contains()` | false | true | false | true (until flush) |

### Key Architecture Facts

```
Persistence Context = Identity Map + Action Queue + Snapshots
Identity Map key    = EntityKey(EntityClass + PrimaryKey)
Identity guarantee  = ONE instance per PK per EntityManager
PC scope (default)  = transaction boundary (Spring)
PC scope (extended) = bean lifecycle (rare in Spring)
ActionQueue order   = INSERT → UPDATE → CollectionOps → DELETE
Flush triggers      = commit / explicit flush() / pre-query (AUTO mode)
```

### Common Production Pitfalls

```
1. LazyInitializationException    → Accessing lazy collections after PC closes
2. Lost updates                   → Modifying detached entity, forgetting to merge
3. Unexpected UPDATEs             → Loading entity and Persistence Context flush
4. N+1 queries                    → Lazy associations in loops
5. Detach after merge confusion   → Using old detached reference post-merge
6. Open Session in View           → spring.jpa.open-in-view=true hides PC problems
7. Large PC memory pressure       → Many entities in one long transaction
8. IDENTITY generator batching    → Breaks JDBC batch inserts
```

---

---

## The Persistence Context Internals

**Q1. What are the three main data structures inside `StatefulPersistenceContext`? What is the purpose of each?**

**Identity Map (`entitiesByKey`)** — A `Map<EntityKey, Object>` where the key is (entity type + primary key). Guarantees that within one PC, there is exactly one Java instance per DB row. Every `find()` checks here first before hitting the DB.

**Snapshots (`entitySnapshotsByKey`)** — A deep copy of each entity's field values captured at load time. Used by dirty checking at flush time — Hibernate compares current field values against this snapshot to decide whether to generate an UPDATE.

**ActionQueue** — Holds all pending SQL operations (inserts, updates, deletes) that haven't been sent to the DB yet. Flushed as a batch at commit time or when explicitly triggered.

---

**Q2. What is the "loaded state snapshot" and when exactly is it created?**

The snapshot is a deep copy of the entity's field values captured at the moment the entity enters the Persistence Context. It is created at:

- `em.find()` — after loading from DB
- `em.persist()` — after registering a new entity
- `em.merge()` — on the returned managed instance
- JPQL query results — after hydrating each result row

At flush time, Hibernate compares current field values against this snapshot. If anything differs → UPDATE is generated. If nothing changed → no SQL.

---

**Q3. What is the `ActionQueue`? What types of actions does it hold, and in what order?**

The ActionQueue is a buffer of pending SQL operations accumulated during a transaction. Execution order:

```
1. OrphanRemovalAction      ← orphan collection elements
2. EntityInsertAction       ← from persist()
3. EntityUpdateAction       ← from dirty checking
4. CollectionRemoveAction   ← removed collection elements
5. CollectionUpdateAction   ← updated collections
6. CollectionRecreateAction ← recreated collections
7. EntityDeleteAction       ← from remove()
```

Inserts first, deletes last. This order is deliberate — it respects foreign key constraints automatically.

---

**Q4. Why does the ActionQueue execute INSERTs before DELETEs?**

Foreign key constraints. If a child row references a parent row, the parent must exist before the child is inserted. Conversely, children must be deleted before parents.

If Hibernate executed DELETEs first, it might try to delete a parent row that still has child rows referencing it — causing a foreign key constraint violation. By inserting parents before children and deleting children before parents, Hibernate avoids these violations automatically.

---

## The Four Entity States

**Q5. What are the four entity states? Describe each.**

**Transient** — A plain Java object with no association to any PC and no DB row. Hibernate has zero knowledge of it. GC collects it normally.

**Managed** — Associated with an open PC. Hibernate tracks it via the identity map, holds a snapshot, and will auto-flush changes at commit.

**Detached** — Was previously managed but the PC closed or it was explicitly detached. Has a valid ID and DB row, but changes are NOT tracked.

**Removed** — Was managed, `em.remove()` was called. DELETE is enqueued in ActionQueue. Still in the PC but marked for deletion. Becomes transient after flush/commit.

---

**Q6. You create `Product p = new Product()` and manually set `p.setId(1L)`. Is `p` transient or detached?**

**Transient.** Entity state is determined by Persistence Context association — not by whether an ID is set. `p` has never been associated with any PC, so it is transient regardless of the ID value.

This matters because `em.persist(p)` on a transient entity with a manually set ID may cause an `EntityExistsException` if a row with that ID already exists.

---

**Q7. List every way an entity can transition from MANAGED to DETACHED.**

1. Transaction commits → PC closes → all managed entities become detached
2. `em.detach(entity)` — explicit detachment of one entity
3. `em.clear()` — detaches ALL entities in the PC
4. `em.close()` — closes the PC entirely
5. Serialization and deserialization — the deserialized copy has no PC reference
6. Passing entity across transaction boundaries in layered Spring architecture

---

**Q8. What happens to an entity's state after `em.flush()` followed by transaction commit?**

After flush + commit, the entity becomes **DETACHED** (in a transaction-scoped PC, which is the Spring default). The PC is destroyed at transaction end. The entity object still exists in memory with its field values intact, but it is no longer tracked by any PC.

---

## `persist()` Internals

**Q9. Does `em.persist()` immediately execute an INSERT SQL statement?**

No — with one exception. `persist()` registers the entity in the identity map, creates a snapshot, and enqueues an `EntityInsertAction` in the ActionQueue. The actual INSERT SQL is deferred until flush time (commit or explicit `em.flush()`).

---

**Q10. What is the one exception to the deferred INSERT rule, and why?**

`GenerationType.IDENTITY`. With IDENTITY, the DB generates the ID at INSERT time. Hibernate needs that ID immediately after persist (to register the entity in the identity map under the correct key). So the INSERT fires immediately upon `persist()` — it cannot be deferred. This is exactly why IDENTITY breaks JDBC batching.

---

**Q11. What happens if you call `em.persist()` on an already MANAGED entity?**

It is a no-op. The entity is already tracked, already in the identity map, already has a snapshot. Hibernate ignores the call. No exception is thrown.

---

**Q12. What happens if you call `em.persist()` on a DETACHED entity?**

`EntityExistsException` (or `PersistenceException` wrapping a constraint violation). The detached entity has an ID that already exists in the DB. Hibernate either detects this and throws immediately, or attempts an INSERT which fails at the DB level with a duplicate key violation.

For detached entities you should use `em.merge()`, not `em.persist()`.

---

## `merge()` Internals

**Q13. Walk through `em.merge()` on a detached entity whose ID exists in the DB.**

```
1. Is entity null?          → IllegalArgumentException
2. Is entity REMOVED?       → IllegalArgumentException
3. Is entity MANAGED?       → return it as-is (no-op)
4. Is entity DETACHED (has ID)?
   a. Check identity map: managed instance with same key exists?
      YES → copy detached state onto existing managed instance → return it
      NO  → SELECT from DB to load current state
           → copy detached state onto loaded instance
           → register loaded instance in identity map
           → return managed instance
5. Cascade CascadeType.MERGE to associations
6. At flush: UPDATE if state differs from snapshot
```

---

**Q14. What is wrong with this code?**
```java
Product detached = getDetachedProduct();
em.merge(detached);
em.persist(detached);
```

Two problems. First, `em.persist(detached)` after merge is wrong — `detached` is still detached after `merge()`. The managed copy is the **return value** of `merge()`, which is being ignored here.

Second, `em.persist(detached)` on a detached entity (has ID) causes `EntityExistsException`.

Correct code:
```java
Product managed = em.merge(detached); // use the return value
// managed is now tracked — no need to persist
```

---

**Q15. What does `merge()` do with a completely TRANSIENT entity (no ID)?**

Treats it like `persist()` — creates a new managed copy, enqueues an INSERT, returns the managed instance. The original transient object remains transient.

```java
Product transient = new Product("Widget"); // no ID
Product managed = em.merge(transient);
// managed is a new tracked instance
// INSERT enqueued
// transient is still transient
```

---

**Q16. What does `merge()` do with an already MANAGED entity?**

Returns the same entity as-is. It's already tracked, already in the identity map. No SELECT, no copy, no extra work. Complete no-op.

---

**Q17. Does the detached instance itself ever become managed after `merge()`?**

Never. `merge()` always returns a different managed instance (or the same one if already managed). The detached object passed in remains detached permanently. This is the most common merge misconception.

```java
Product detached = ...;
Product managed = em.merge(detached);

em.contains(detached); // always FALSE
em.contains(managed);  // TRUE
detached == managed;   // FALSE (different objects)
```

---

## `remove()` and `refresh()` Internals

**Q18. You want to delete a detached entity. What sequence of operations is needed?**

```java
// WRONG:
em.remove(detached); // IllegalArgumentException

// CORRECT:
Product managed = em.merge(detached); // get managed copy
em.remove(managed);                   // now remove the managed copy
// At flush: DELETE FROM products WHERE id=?
```

`remove()` requires a MANAGED entity. Detached entities have no PC association, so Hibernate cannot schedule their deletion.

---

**Q19. What happens if you call `em.persist()` on a REMOVED entity before flush?**

The removal is cancelled — the entity is rescheduled for INSERT. It transitions back to MANAGED. This is valid JPA behavior, though rarely intentional.

Practical scenario: you remove an entity inside a conditional block, then realize the condition was wrong and call persist() to undo it. Works, but confusing code.

---

**Q20. What does `em.refresh()` do to pending unsaved changes?**

Completely overwrites them. `refresh()` fires a SELECT against the DB and replaces the entire in-memory state with the fresh DB data. All pending changes are silently discarded. The snapshot is also reset to the fresh DB state.

```java
Product p = em.find(Product.class, 1L); // name="Widget"
p.setName("Gadget"); // pending change
em.refresh(p);       // SELECT fires, name back to "Widget"
// "Gadget" is gone forever
```

---

**Q21. What exception does `em.refresh()` throw if called on a detached entity?**

`IllegalArgumentException` — because refresh requires a MANAGED entity. If the entity's DB row doesn't exist anymore (deleted by another process), it throws `EntityNotFoundException`.

---

## Dirty Checking

**Q22. How many UPDATE statements does this produce?**
```java
Product p = em.find(Product.class, 1L);
p.setName("A");
p.setName("B");
p.setName("C");
```

**One UPDATE.** Dirty checking is snapshot-based, not event-based. Hibernate does not listen to setter calls. At flush time it compares the original snapshot (name="Widget") against the final current state (name="C"). Only one UPDATE: `SET name='C'`. The intermediate values "A" and "B" are completely irrelevant.

---

**Q23. If you load an entity, make no changes, and commit — does Hibernate execute an UPDATE?**

No. Dirty checking compares snapshot against current state. If they are identical, no dirty fields are detected and no UPDATE is generated. This is a key performance feature — read-only loads within a transaction produce zero write SQL.

---

**Q24. What is the performance concern with dirty checking in large Persistence Contexts?**

Dirty checking is O(n) per flush — Hibernate iterates over every managed entity in the identity map, reads all field values via reflection, and compares against snapshots. With hundreds or thousands of managed entities in one transaction, this becomes expensive.

Solutions: use `em.clear()` periodically in batch jobs, use `StatelessSession` for bulk operations, or use `@Transactional(readOnly=true)` which disables dirty checking entirely via `FlushMode.MANUAL`.

---

**Q25. What is bytecode enhancement dirty checking vs snapshot-based?**

**Snapshot-based (default):** Hibernate stores a copy of the loaded state. At flush, it compares current values against that copy using reflection. Requires storing two copies of every entity's state in memory.

**Bytecode enhancement:** Hibernate's build plugin rewrites compiled `.class` files to inject a `$$_hibernate_tracker` field into the entity. Every setter call automatically marks that field as dirty. At flush, Hibernate checks the tracker — no reflection, no comparison needed. Faster flush, less memory (no snapshot copy needed).

The tradeoff: bytecode enhancement requires build tooling setup and makes your compiled classes less portable.

---

## Identity Map

**Q26. You call `em.find(Product.class, 1L)` three times in one transaction. How many SELECTs execute?**

**One.** The first `find()` hits the DB and stores the result in the identity map. The second and third calls find the existing entry in the identity map and return the same instance immediately. Zero SQL for calls two and three.

---

**Q27. A JPQL query returns Product id=1. You previously loaded it with `em.find()`. Same instance?**

Yes, same instance. JPQL queries always execute SQL (they bypass the identity map for the query itself), but after the ResultSet is loaded, Hibernate checks the identity map: "do I already have a managed instance for this key?" If yes → returns the existing instance, discards the freshly loaded data. The identity map guarantee holds even for JPQL results.

---

**Q28. Two different EntityManager instances each load Product id=1. Same instance?**

No. The identity map is per-EntityManager (per-PC). Each EM has its own separate map. Two different EMs can have two different Java objects representing the same DB row. This is why PCs are not thread-safe — sharing one EM across threads can cause identity map corruption.

---

**Q29. Is the Persistence Context thread-safe?**

No. The PC (`StatefulPersistenceContext`) and `EntityManager` are not thread-safe. In Spring, this is handled by the transactional proxy — each thread gets its own EntityManager bound to its own transaction via `ThreadLocal`. Never share an EntityManager across threads.

---

## Flush Modes

**Q30. With `FlushMode.AUTO`, does Hibernate always flush before a query?**

No — only when necessary. Hibernate checks whether any pending ActionQueue operations affect the tables the query is about to read. If a pending INSERT touches the `products` table and the query reads `products` → flush first. If the pending operations affect different tables → skip the flush.

This is an optimization to avoid unnecessary flushes while still ensuring query consistency.

---

**Q31. FlushMode.COMMIT — you persist a new entity, then run a JPQL count query. Does count include it?**

No. With `FlushMode.COMMIT`, Hibernate does not flush before queries. The INSERT for the new entity is still in the ActionQueue, not in the DB. The JPQL query reads from DB only → new entity is not counted.

---

**Q32. After persisting (FlushMode.COMMIT), does `em.find()` return the new entity?**

Yes — but for a different reason than JPQL. `em.find()` checks the identity map first. The newly persisted entity is already registered there (persist() adds it to the identity map immediately). So `find()` returns it from the map without touching the DB.

This is the asymmetry: JPQL queries the DB (doesn't see the new entity), but `find()` checks the identity map first (does see it). A genuine source of confusing bugs.

---

**Q33. What flush mode does `@Transactional(readOnly=true)` set? What's the benefit?**

Sets `FlushMode.MANUAL`. This means Hibernate never flushes automatically — ever. The benefit is twofold: no pre-query flushes, and **dirty checking is completely disabled**. Hibernate doesn't bother comparing snapshots at commit because it knows no writes should happen. This reduces both CPU cost (no reflection comparison) and memory pressure (snapshots are still held but never consulted).

---

## Transaction Boundaries and Detachment

**Q34. What is wrong with this design?**
```java
@Transactional
public Order getOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
}

public String getSummary(Long id) {
    Order order = getOrder(id);
    return order.getItems().stream()...;
}
```

`getOrder()` has its own `@Transactional`. When it returns, the transaction commits and the PC closes. The returned `Order` is now DETACHED. `order.getItems()` is a lazy collection — accessing it requires an open Session, which no longer exists. Result: `LazyInitializationException: could not initialize proxy - no Session`.

---

**Q35. Four solutions to `LazyInitializationException`, with tradeoffs.**

**1. JOIN FETCH in query:**
```java
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
```
Loads everything in one SQL. Best performance, but couples query to use case.

**2. `@EntityGraph`:**
```java
@EntityGraph(attributePaths = {"items"})
Order findById(Long id);
```
Cleaner than JOIN FETCH, declarative, still one SQL.

**3. Extend the transaction scope:**
Move `@Transactional` to the caller so the PC stays open during collection access. Risk: long transactions, lazy loading scattered across layers.

**4. DTO projection:**
```java
@Query("SELECT new OrderDto(o.id, i.name) FROM Order o JOIN o.items i")
```
Never loads entities at all. Best for read-only use cases, eliminates the problem entirely.

**Anti-pattern to avoid:** `spring.jpa.open-in-view=true` — keeps a Session open for the entire HTTP request. "Fixes" lazy loading but causes N+1 queries silently and holds DB connections for the full request duration.

---

**Q36. What is `spring.jpa.open-in-view=true` and why is it an anti-pattern?**

It keeps a Hibernate Session open from the start of the HTTP request until the response is written — even after the `@Transactional` method returns. This allows lazy collections to be initialized during serialization (e.g., in JSON serializers).

Why it's an anti-pattern:
- Holds a DB connection for the entire HTTP request lifecycle
- Lazy loading happens silently during serialization — invisible N+1 queries
- Masks architectural problems instead of solving them
- Under load, connection pool exhaustion becomes likely

Spring Boot enables this by default (and logs a warning about it).

---

## Extended vs Transaction-Scoped Persistence Context

**Q37. Difference between transaction-scoped and extended PC.**

**Transaction-scoped (default in Spring):** PC is created when the transaction starts and destroyed when it commits or rolls back. All entities become detached after the transaction ends.

**Extended PC:** PC survives transaction boundaries. The same PC (and all its managed entities) persists across multiple transactions for the lifetime of the bean. Entities stay MANAGED even when no transaction is active.

---

**Q38. Extended PC — load entity, close transaction, mutate, start new transaction. What happens?**

The entity remains MANAGED in the extended PC even after the transaction closes. The mutation is tracked. When the new transaction starts, the PC is already associated with it. At commit of the new transaction, the dirty checking runs and the UPDATE is flushed. The change reaches the DB.

---

**Q39. What is the memory risk of extended Persistence Contexts?**

The PC accumulates entities indefinitely. Every entity loaded during the bean's lifetime stays in the identity map and snapshot store. With no transaction boundary to trigger a clear, the PC grows unboundedly. In long-lived beans handling many requests, this causes memory pressure and increasingly slow dirty checking at flush time (more entities = more snapshot comparisons).

---

## ActionQueue Optimizations

**Q40. `em.persist(p)` then immediately `em.remove(p)` before flush. What SQL executes?**

No SQL. Hibernate's ActionQueue recognizes the paired INSERT + DELETE for the same entity within one flush cycle and cancels them both out. The net effect is zero — the entity never existed from the DB's perspective.

---

**Q41. Why does the answer to Q40 differ for IDENTITY vs SEQUENCE?**

With **SEQUENCE**, the INSERT is deferred in the ActionQueue. Hibernate can inspect the queue, find the paired INSERT+DELETE, and cancel both before any SQL is sent.

With **IDENTITY**, `persist()` fires the INSERT immediately (because the ID must be generated by the DB right away). By the time `remove()` is called, the INSERT has already executed. So Hibernate must now also execute the DELETE. Result: one INSERT + one DELETE in the DB.

---

**Q42. Walk through this scenario step by step:**
```java
@Transactional
public void test() {
    Product p = em.find(Product.class, 1L); // name="Widget"
    p.setName("Gadget");
    em.refresh(p);
    p.setName("SuperWidget");
}
```

```
Step 1: em.find(Product.class, 1L)
  → SELECT * FROM products WHERE id=1
  → p.name = "Widget"
  → Snapshot: {name="Widget"}
  → p is MANAGED

Step 2: p.setName("Gadget")
  → p.name = "Gadget" in memory
  → Snapshot still: {name="Widget"}
  → No SQL yet

Step 3: em.refresh(p)
  → SELECT * FROM products WHERE id=1 (re-read from DB)
  → p.name = "Widget" (DB value overwrites "Gadget")
  → Snapshot reset: {name="Widget"}
  → "Gadget" is permanently discarded

Step 4: p.setName("SuperWidget")
  → p.name = "SuperWidget"
  → Snapshot: {name="Widget"}

Step 5: Transaction commits → dirty check
  → current: {name="SuperWidget"} vs snapshot: {name="Widget"}
  → Dirty → UPDATE products SET name='SuperWidget' WHERE id=1
```

Final SQL: one SELECT (find) + one SELECT (refresh) + one UPDATE.

---

## State Transition Matrix

**Q43. Complete state transition matrix:**

| Operation | Transient | Managed | Detached | Removed |
|---|---|---|---|---|
| `persist()` | → Managed | No-op | EntityExistsException | → Managed (reschedules) |
| `merge()` | → Managed copy | Returns self | → Managed copy | IllegalArgumentException |
| `remove()` | IllegalArgumentException | → Removed | IllegalArgumentException | No-op |
| `refresh()` | IllegalArgumentException | Re-reads DB, stays Managed | IllegalArgumentException | IllegalArgumentException |
| `contains()` | false | true | false | true (until flush) |

---

## Mixed / Tricky Scenarios

**Q44. `@Transactional` method calls another `@Transactional` on the same bean. Does the inner method get its own PC?**

No — and this is the Spring self-invocation trap. `@Transactional` works via a proxy that wraps your bean. When you call `this.innerMethod()` directly, you bypass the proxy entirely — the transaction interceptor never runs. The inner method joins the outer transaction and uses the same PC.

If called through the proxy (from another bean), the default propagation is `REQUIRED` — inner method joins the existing transaction and shares the same PC.

---

**Q45. Difference between `em.detach(entity)` and `em.clear()`.**

`em.detach(entity)` — removes one specific entity from the PC. All other managed entities remain tracked.

`em.clear()` — removes ALL entities from the PC at once. The entire identity map and all snapshots are discarded. All previously managed entities become detached simultaneously.

Use `clear()` in batch processing to prevent memory buildup after processing each chunk.

---

**Q46. You call `em.clear()` mid-transaction after making several changes. What happens?**

All pending changes are lost. `clear()` discards the entire identity map and all snapshots. Any dirty entities are simply forgotten — their `EntityUpdateAction` entries in the ActionQueue are also dropped. No UPDATE SQL will be generated for those changes at commit.

```java
Product p = em.find(Product.class, 1L);
p.setName("Changed"); // pending change
em.clear();           // p is now DETACHED, change is gone
// At commit: zero UPDATE SQL — change never persisted
```

---

**Q47. An entity is MANAGED. You serialize and deserialize it. What state is the deserialized copy?**

**Detached.** Serialization writes the entity's field values to a byte stream. Deserialization reconstructs a new Java object from those bytes. The new object has no reference to any PC (the PC is a runtime, in-memory construct — it doesn't serialize). So the deserialized copy is detached — it has a valid ID and field values but no PC association.

---

**Q48. Why does `contains()` return `true` for a REMOVED entity until flush?**

Because a REMOVED entity is still in the Persistence Context — it's still registered in the identity map, just marked with a "deleted" flag. `contains()` checks whether the entity is associated with the PC, not whether it's going to survive the flush. Until flush actually executes the DELETE and the entity is evicted from the PC, `contains()` returns `true`.

After flush/commit, the entity is evicted from the identity map and `contains()` returns `false`.

---
