# 🏛️ TOPIC 15 — Locking — Optimistic and Pessimistic

---

## 1️⃣ Conceptual Explanation

### Why Locking Exists — The Lost Update Problem

Without locking, concurrent transactions reading and writing the same data produce incorrect results:

```
Time  TX A                          TX B
──────────────────────────────────────────────────────
T1    READ account balance = 1000
T2                                  READ account balance = 1000
T3    balance = 1000 - 200 = 800
T4    WRITE balance = 800
T5                                  balance = 1000 + 500 = 1500
T6                                  WRITE balance = 1500
T7    COMMIT                        COMMIT

Final balance: 1500  (WRONG — should be 1300)
TX A's debit was completely overwritten — the Lost Update
```

Two fundamentally different strategies prevent this:

```
Optimistic Locking:
  "Conflicts are rare — don't lock, detect conflict on write"
  → No DB locks held during read
  → At write time: check if data changed since read
  → If changed: throw OptimisticLockException, caller must retry
  → Best for: low-contention, read-heavy, long user think-time scenarios

Pessimistic Locking:
  "Conflicts are likely — lock the data during read"
  → DB-level lock acquired on SELECT
  → Other transactions blocked until lock released
  → At write time: no conflict detection needed (lock prevented concurrent writes)
  → Best for: high-contention, write-heavy, short operations
```

---

### Optimistic Locking — `@Version` Internals

`@Version` is the JPA mechanism for optimistic locking. A version column is added to the entity table. Every UPDATE increments this version. If two transactions read the same row, the second to write detects the version mismatch and fails.

```java
@Entity
public class Product {
    @Id @GeneratedValue Long id;
    String name;
    BigDecimal price;
    int stock;

    @Version
    Integer version;  // managed entirely by Hibernate — never set manually
}
```

**How `@Version` works internally — SQL trace:**

```
TX A reads Product id=1:
  SELECT id, name, price, stock, version FROM products WHERE id=1
  → Result: version=3

TX B reads Product id=1:
  SELECT id, name, price, stock, version FROM products WHERE id=1
  → Result: version=3

TX A modifies and flushes:
  UPDATE products
  SET name=?, price=?, stock=?, version=4   ← version incremented
  WHERE id=1 AND version=3                  ← version check in WHERE
  → Rows affected: 1 (success)

TX B modifies and flushes (version still 3 in memory):
  UPDATE products
  SET name=?, price=?, stock=?, version=4   ← Hibernate still uses version+1=4
  WHERE id=1 AND version=3                  ← version check in WHERE
  → Rows affected: 0 (TX A already changed version to 4)
  → Hibernate detects 0 rows affected → throws OptimisticLockException
  → TX B rolls back
```

**The version check is in the WHERE clause — not a separate SELECT.**

---

### `@Version` — Supported Types and Rules

```java
// Valid @Version field types:
@Version int version;         // primitive int — starts at 0 (be careful with isNew())
@Version Integer version;     // wrapper Integer — starts at null (null = new entity)
@Version long version;        // primitive long
@Version Long version;        // wrapper Long
@Version short version;
@Version Short version;
@Version Timestamp version;   // java.sql.Timestamp — uses DB timestamp (unreliable)

// RECOMMENDED:
@Version Integer version;     // null = new entity (isNew()=true via @Version null check)
@Version Long version;        // for tables with very high update frequency
```

**Rules:**

```
1. NEVER set @Version field manually in application code
   → Hibernate manages it exclusively
   → Setting it manually breaks optimistic locking semantics

2. Hibernate increments version on EVERY UPDATE (any field change)

3. Hibernate does NOT increment version on:
   → Bulk UPDATE queries (@Modifying @Query)
   → Direct JDBC updates bypassing Hibernate

4. @Version null → isNew() = true → save() calls persist() not merge()
   (This is the recommended way to handle isNew() for entities with @Version)

5. Primitive @Version (int/long) starts at 0
   → save(new entity with int version=0) → isNew() checks @Version null first
   → primitive 0 is NOT null → isNew() may return false → save() calls merge()!
   → Use wrapper types (Integer/Long) to avoid this trap
```

---

### `OptimisticLockException` — Handling and Retry

```java
// The exception hierarchy:
javax.persistence.OptimisticLockException
  → wrapped in Spring's:
org.springframework.orm.jpa.JpaOptimisticLockingFailureException
  → extends:
org.springframework.dao.OptimisticLockingFailureException

// When does it fire?
// At flush time — not at read time, not at write-field time
// Specifically: when Hibernate executes the UPDATE and sees 0 rows affected
```

**Retry pattern:**

```java
@Service
public class ProductService {

    @Retryable(
        retryFor = OptimisticLockingFailureException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2)
    )
    @Transactional
    public void decrementStock(Long productId, int quantity) {
        Product product = productRepo.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));

        if (product.getStock() < quantity) {
            throw new InsufficientStockException();
        }
        product.setStock(product.getStock() - quantity);
        // On flush: UPDATE ... WHERE id=? AND version=?
        // If version mismatch: OptimisticLockException → @Retryable retries
    }
    // Each retry: fresh TX, fresh read, fresh version
}
```

**When NOT to use retry:**

```java
// Do NOT retry blindly — re-read and re-validate business rules:
@Transactional
public void retryWrong() {
    // WRONG retry approach:
    for (int i = 0; i < 3; i++) {
        try {
            product.setStock(product.getStock() - 1); // same stale entity!
            productRepo.save(product);
            break;
        } catch (OptimisticLockException e) {
            // Entity is stale AND now detached (TX rolled back)
            // Retrying same entity = always fails
        }
    }
}

// CORRECT: each retry = new TX + fresh read:
@Transactional
public void retryCorrect(Long id, int qty) {
    // Fresh read every time this method is called (each retry is a new @Transactional call)
    Product fresh = productRepo.findById(id).get();
    fresh.setStock(fresh.getStock() - qty);
    // flush: UPDATE ... WHERE id=? AND version=?
}
```

---

### `LockModeType` — Complete Reference

JPA defines seven lock modes, split into optimistic and pessimistic categories:

```
OPTIMISTIC GROUP:
  NONE                   — no lock
  OPTIMISTIC             — version check on commit (same as having @Version)
  OPTIMISTIC_FORCE_INCREMENT — version INCREMENTED even on read-only access

PESSIMISTIC GROUP:
  PESSIMISTIC_READ       — shared lock (SELECT ... FOR SHARE)
  PESSIMISTIC_WRITE      — exclusive lock (SELECT ... FOR UPDATE)
  PESSIMISTIC_FORCE_INCREMENT — exclusive lock + version incremented
  READ                   — alias for OPTIMISTIC (legacy)
  WRITE                  — alias for OPTIMISTIC_FORCE_INCREMENT (legacy)
```

---

### `OPTIMISTIC` Lock Mode — Version Check on Commit

```java
// Explicitly request optimistic locking even without @Version annotation:
Product product = em.find(Product.class, id, LockModeType.OPTIMISTIC);
// OR with repository:
productRepo.findById(id, LockModeType.OPTIMISTIC); // not standard in Spring Data

// Equivalent to having @Version — performs version check UPDATE WHERE version=?
// Useful when you want to ensure no concurrent modification
// even for entities you read but don't explicitly modify
```

---

### `OPTIMISTIC_FORCE_INCREMENT` — Increment on Read

```java
// Increments @Version even if entity is NOT modified:
Product product = em.find(Product.class, id,
                           LockModeType.OPTIMISTIC_FORCE_INCREMENT);

// SQL at flush:
// UPDATE products SET version=N+1 WHERE id=? AND version=N
// → Even if NO fields changed

// Use case: Parent entity aggregates child state
// Reading parent and modifying child should "dirty" the parent version
// to prevent concurrent readers from seeing stale aggregate state

// Example:
@Entity
public class ShoppingCart {
    @Id Long id;
    @Version Integer version;
    @OneToMany List<CartItem> items;
}
// When adding an item to cart, increment cart's version
// even though Cart entity itself wasn't modified:
ShoppingCart cart = em.find(ShoppingCart.class, cartId,
                             LockModeType.OPTIMISTIC_FORCE_INCREMENT);
cartItemRepo.save(new CartItem(cart, product, qty));
// Cart version bumped → concurrent reads of cart see conflict
```

---

### `PESSIMISTIC_READ` — Shared Lock

```java
// Acquires a SHARED lock — multiple readers allowed, no writers:
Product product = em.find(Product.class, id, LockModeType.PESSIMISTIC_READ);

// SQL (PostgreSQL/MySQL):
// SELECT p.* FROM products p WHERE p.id=? FOR SHARE
// or: FOR SHARE NOWAIT, FOR SHARE SKIP LOCKED

// Semantics:
// → Multiple transactions CAN hold PESSIMISTIC_READ simultaneously
// → A transaction wanting PESSIMISTIC_WRITE MUST WAIT until all PESSIMISTIC_READ released
// → Reading data is allowed for others, writing is blocked

// Use case: "I'm reading this data and I want to guarantee nobody writes it
//            while I'm computing something based on it"
```

---

### `PESSIMISTIC_WRITE` — Exclusive Lock

```java
// Acquires an EXCLUSIVE lock — no other readers or writers:
Product product = em.find(Product.class, id, LockModeType.PESSIMISTIC_WRITE);

// SQL:
// SELECT p.* FROM products p WHERE p.id=? FOR UPDATE

// Semantics:
// → Only ONE transaction can hold PESSIMISTIC_WRITE at a time
// → ALL other transactions wanting READ or WRITE must WAIT
// → Guarantees exclusive access to the row

// Common use case: inventory decrement in high-contention scenario:
@Transactional
public void decrementStock(Long productId, int qty) {
    Product product = productRepo.findByIdWithLock(productId);
    // Lock acquired — no other TX can modify until this TX commits
    if (product.getStock() < qty) throw new InsufficientStockException();
    product.setStock(product.getStock() - qty);
    // UPDATE at flush — guaranteed no version conflict (lock prevented it)
}

// Repository method for pessimistic write:
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Optional<Product> findByIdWithLock(@Param("id") Long id);
```

---

### `PESSIMISTIC_FORCE_INCREMENT` — Exclusive Lock + Version Bump

```java
// Exclusive lock AND increments @Version:
Product product = em.find(Product.class, id,
                           LockModeType.PESSIMISTIC_FORCE_INCREMENT);

// SQL:
// SELECT p.* FROM products p WHERE p.id=? FOR UPDATE
// Then at flush (even if entity unchanged):
// UPDATE products SET version=N+1 WHERE id=? AND version=N

// Use case: When you need to signal to optimistic locking readers
// that this entity was exclusively accessed — even if unchanged
// Combines physical lock (FOR UPDATE) with version signal
```

---

### Spring Data `@Lock` Annotation

Spring Data provides `@Lock` to apply lock modes in repository methods:

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Pessimistic write on findById equivalent:
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);

    // Pessimistic write on derived query:
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    Optional<Product> findBySku(String sku);
    // SQL: SELECT p.* FROM products WHERE sku=? FOR UPDATE

    // Pessimistic read:
    @Lock(LockModeType.PESSIMISTIC_READ)
    @Query("SELECT p FROM Product p WHERE p.category = :cat")
    List<Product> findByCategoryForShare(@Param("cat") String cat);

    // Optimistic force increment:
    @Lock(LockModeType.OPTIMISTIC_FORCE_INCREMENT)
    Optional<Product> findByIdAndActiveTrue(Long id);

    // Lock on findAll — rarely useful in practice:
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    List<Product> findAll();
    // SQL: SELECT p.* FROM products FOR UPDATE
    // DANGER: locks entire table effectively
}
```

---

### Lock Timeout — `@QueryHints`

Pessimistic locks can specify a timeout after which waiting transactions fail instead of blocking indefinitely:

```java
// Timeout 0 — NOWAIT: fail immediately if lock unavailable:
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints({
    @QueryHint(name = "jakarta.persistence.lock.timeout", value = "0")
})
@Query("SELECT p FROM Product p WHERE p.id = :id")
Optional<Product> findByIdNowait(@Param("id") Long id);
// SQL (PostgreSQL): SELECT ... FOR UPDATE NOWAIT
// If lock unavailable: LockTimeoutException immediately

// Timeout N milliseconds — wait up to N ms then fail:
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints({
    @QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000")
})
@Query("SELECT p FROM Product p WHERE p.id = :id")
Optional<Product> findByIdWithTimeout(@Param("id") Long id);
// SQL (PostgreSQL): SELECT ... FOR UPDATE

// SKIP LOCKED — skip rows that are already locked:
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints({
    @QueryHint(name = "jakarta.persistence.lock.timeout", value = "-2")
})
@Query("SELECT p FROM Product p WHERE p.status = 'PENDING'")
List<Product> findPendingSkipLocked();
// SQL: SELECT ... FOR UPDATE SKIP LOCKED
// Returns only rows NOT locked by other transactions
// Perfect for job queue processing
```

**Timeout values:**

```
0     → NOWAIT (immediate failure if locked)
-1    → wait indefinitely (default behavior)
-2    → SKIP LOCKED (skip already-locked rows)
N > 0 → wait up to N milliseconds (dialect-dependent support)
```

---

### Lock Scope — `EXTENDED` vs `NORMAL`

```java
// Lock scope controls whether the lock extends to related tables/joins:
@QueryHints({
    @QueryHint(name = "jakarta.persistence.lock.scope",
               value = "EXTENDED")  // or "NORMAL"
})

// NORMAL (default): lock only the entity's primary table
// EXTENDED: lock entity table AND joined tables (for JOINED inheritance)

// Example with JOINED inheritance:
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public class Payment { @Id Long id; @Version Integer version; }

@Entity
public class CreditCardPayment extends Payment { String cardNumber; }

// NORMAL lock: locks payments table only
// EXTENDED lock: locks payments AND credit_card_payments tables
```

---

### Deadlock Patterns and Prevention

**Deadlock pattern 1 — Reversed acquisition order:**

```
TX A: Lock Product 1, then lock Product 2
TX B: Lock Product 2, then lock Product 1

T1: TX A locks Product 1 (success)
T2: TX B locks Product 2 (success)
T3: TX A tries to lock Product 2 (waits for TX B)
T4: TX B tries to lock Product 1 (waits for TX A)
→ DEADLOCK
```

**Prevention: Always acquire locks in the same order:**

```java
@Transactional
public void swapStock(Long id1, Long id2, int qty) {
    // Always lock lower ID first:
    Long first  = Math.min(id1, id2);
    Long second = Math.max(id1, id2);

    Product p1 = productRepo.findByIdForUpdate(first);
    Product p2 = productRepo.findByIdForUpdate(second);
    // Consistent order prevents deadlock
}
```

**Deadlock pattern 2 — Upgrade lock escalation:**

```
TX A holds PESSIMISTIC_READ on row X
TX B holds PESSIMISTIC_READ on row X
TX A tries to upgrade to PESSIMISTIC_WRITE → waits for TX B
TX B tries to upgrade to PESSIMISTIC_WRITE → waits for TX A
→ DEADLOCK
```

**Prevention: Acquire PESSIMISTIC_WRITE directly if write is intended:**

```java
// Don't upgrade locks — acquire at the highest level needed from the start:
// WRONG:
Product p = findById(id);                  // implicit PESSIMISTIC_READ or none
p = em.lock(p, LockModeType.PESSIMISTIC_WRITE); // upgrade → deadlock risk

// RIGHT:
Product p = findByIdForUpdate(id);         // PESSIMISTIC_WRITE from the start
```

---

### Optimistic vs Pessimistic — Decision Framework

```
Choose OPTIMISTIC when:
  ✓ Low contention (rare concurrent writes on same row)
  ✓ Long user think-time (user reads data, edits form, submits)
  ✓ Read-heavy workloads (most operations are reads)
  ✓ Distributed / microservice (no shared DB connection)
  ✓ Web applications with user-facing edit forms
  ✓ Acceptable to retry on conflict

Choose PESSIMISTIC when:
  ✓ High contention (frequent concurrent writes on same row)
  ✓ Short operations (lock held for milliseconds)
  ✓ Cannot retry (side effects already occurred)
  ✓ Financial operations where conflict must never succeed
  ✓ Inventory reservation (overselling must be impossible)
  ✓ Job queue claiming (SKIP LOCKED pattern)
  ✓ Strict consistency required, no retry acceptable
```

---

### `@Version` and Bulk Operations — The Invisible Gap

```java
// Bulk UPDATE bypasses @Version:
@Modifying
@Transactional
@Query("UPDATE Product p SET p.price = p.price * 1.1 WHERE p.category = :cat")
int increasePrices(@Param("cat") String category);

// This UPDATE does NOT increment @Version
// Entities loaded in PC still have old version values
// Subsequent optimistic lock checks on those entities will SUCCEED
// even though the DB was modified — version didn't change

// This is a silent consistency hole:
// 1. Load product (version=3)
// 2. Bulk UPDATE price (version still 3 in DB)
// 3. Modify product.name (entity in PC still has version=3)
// 4. Flush: UPDATE ... WHERE version=3 → rows affected=1 → no exception
// But DB now has price from bulk update + name from entity update
// The two operations are not coordinated
```

---

## 2️⃣ Code Examples

### Example 1 — `@Version` Full Lifecycle

```java
// Entity with version:
@Entity
public class BankAccount {
    @Id @GeneratedValue Long id;
    String owner;
    BigDecimal balance;

    @Version
    Integer version;  // null for new, auto-incremented by Hibernate

    // NO setter for version — Hibernate manages it
    public Integer getVersion() { return version; }
}

// Repository:
public interface BankAccountRepository
        extends JpaRepository<BankAccount, Long> {

    @Lock(LockModeType.OPTIMISTIC_FORCE_INCREMENT)
    @Query("SELECT a FROM BankAccount a WHERE a.id = :id")
    Optional<BankAccount> findByIdForVersionIncrement(@Param("id") Long id);
}

// Service:
@Service
public class BankAccountService {

    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        BankAccount from = accountRepo.findById(fromId)
            .orElseThrow(AccountNotFoundException::new);
        BankAccount to = accountRepo.findById(toId)
            .orElseThrow(AccountNotFoundException::new);

        // SQL: SELECT id, owner, balance, version FROM accounts WHERE id=?
        // from.version = 5, to.version = 3

        if (from.getBalance().compareTo(amount) < 0)
            throw new InsufficientFundsException();

        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));

        // At flush:
        // UPDATE accounts SET balance=800, version=6 WHERE id=1 AND version=5
        // UPDATE accounts SET balance=1200, version=4 WHERE id=2 AND version=3
        // If either version check fails → OptimisticLockException → TX rolls back
    }

    // Demonstrating version value in response:
    @Transactional(readOnly = true)
    public AccountDto findAccount(Long id) {
        BankAccount account = accountRepo.findById(id).get();
        return new AccountDto(account.getId(),
                              account.getBalance(),
                              account.getVersion()); // expose version to client
    }

    // Client sends version back for conflict detection:
    @Transactional
    public void updateWithVersion(Long id, BigDecimal newBalance,
                                  Integer clientVersion) {
        BankAccount account = accountRepo.findById(id).get();
        if (!account.getVersion().equals(clientVersion)) {
            throw new ConcurrentModificationException(
                "Account was modified. Reload and retry.");
        }
        account.setBalance(newBalance);
        // Hibernate's version check in UPDATE WHERE clause provides
        // the real protection — clientVersion check is extra defense
    }
}
```

---

### Example 2 — Pessimistic Locking Patterns

```java
// Repository:
public interface TicketRepository extends JpaRepository<Ticket, Long> {

    // Exclusive lock — only one TX can reserve at a time:
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT t FROM Ticket t WHERE t.id = :id")
    Optional<Ticket> findByIdForReservation(@Param("id") Long id);

    // Skip locked — for job queue / batch processing:
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints(@QueryHint(
        name = "jakarta.persistence.lock.timeout", value = "-2"))
    @Query("SELECT t FROM Ticket t WHERE t.status = 'AVAILABLE'" +
           " ORDER BY t.id ASC")
    List<Ticket> findAvailableTicketsSkipLocked(Pageable pageable);

    // Nowait — fail fast if locked:
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints(@QueryHint(
        name = "jakarta.persistence.lock.timeout", value = "0"))
    @Query("SELECT t FROM Ticket t WHERE t.id = :id")
    Optional<Ticket> findByIdNowait(@Param("id") Long id);
}

// Service:
@Service
public class TicketService {

    // Exclusive reservation — no overselling:
    @Transactional
    public Reservation reserveTicket(Long ticketId, String userId) {
        // FOR UPDATE: blocks concurrent reservations of same ticket
        Ticket ticket = ticketRepo.findByIdForReservation(ticketId)
            .orElseThrow(() -> new TicketNotFoundException(ticketId));
        // SQL: SELECT ... FROM tickets WHERE id=? FOR UPDATE

        if (ticket.getStatus() != TicketStatus.AVAILABLE) {
            throw new TicketAlreadyReservedException(ticketId);
        }

        ticket.setStatus(TicketStatus.RESERVED);
        ticket.setReservedBy(userId);
        ticket.setReservedAt(LocalDateTime.now());
        // Lock released at TX commit

        return reservationRepo.save(new Reservation(ticket, userId));
    }

    // Job queue — workers claim without collisions:
    @Transactional
    public List<Ticket> claimBatchForProcessing(int batchSize) {
        Pageable limit = PageRequest.of(0, batchSize);
        List<Ticket> tickets =
            ticketRepo.findAvailableTicketsSkipLocked(limit);
        // SQL: SELECT ... FROM tickets WHERE status='AVAILABLE'
        //      ORDER BY id ASC FOR UPDATE SKIP LOCKED
        // → Returns rows NOT locked by other workers
        // → Each worker gets a unique batch

        tickets.forEach(t -> t.setStatus(TicketStatus.PROCESSING));
        return tickets;
    }

    // Nowait — immediate feedback:
    @Transactional
    public void tryReserveOrFail(Long ticketId, String userId) {
        try {
            Ticket ticket = ticketRepo.findByIdNowait(ticketId)
                .orElseThrow(TicketNotFoundException::new);
            // SQL: SELECT ... FOR UPDATE NOWAIT
            // If locked: LockTimeoutException immediately (no waiting)

            ticket.setStatus(TicketStatus.RESERVED);
        } catch (LockTimeoutException e) {
            throw new TicketCurrentlyUnavailableException(
                "Ticket is being processed by another request. Try again.");
        }
    }
}
```

---

### Example 3 — `@Version` with `save()` isNew() Trap

```java
// Entity with WRAPPER @Version (correct):
@Entity
public class Document {
    @Id @GeneratedValue Long id;
    String title;

    @Version Integer version;  // null for new, non-null after persist
}

// Entity with PRIMITIVE @Version (problematic):
@Entity
public class DocumentBad {
    @Id @GeneratedValue Long id;
    String title;

    @Version int version;  // 0 for new — isNew() check broken for @Id-based check
}

@Service
public class DocumentService {

    @Transactional
    public Document createDocument(String title) {
        Document doc = new Document();
        doc.setTitle(title);
        // doc.version = null (Integer wrapper)
        // save() → isNew() → @Version is null → isNew() = true → em.persist()
        return documentRepo.save(doc);
        // SQL: INSERT INTO documents (title, version) VALUES (?, 0)
        // After persist: doc.version = 0
    }

    @Transactional
    public Document updateDocument(Long id, String title) {
        Document doc = documentRepo.findById(id).get();
        // doc.version = 5 (loaded from DB)
        doc.setTitle(title);
        return documentRepo.save(doc);
        // save() → isNew() → @Version = 5 (non-null) → isNew() = false → merge()
        // merge() might SELECT first, but entity is already managed → no SELECT
        // flush: UPDATE documents SET title=?, version=6 WHERE id=? AND version=5
    }

    // The PRIMITIVE trap:
    @Transactional
    public DocumentBad createBadDocument(String title) {
        DocumentBad doc = new DocumentBad();
        doc.setTitle(title);
        // doc.version = 0 (primitive int)
        // save() → isNew() logic:
        //   1. Implements Persistable? No
        //   2. Has @Version? Yes → version == null? NO (0 != null for int)
        //   3. Falls through to @Id check: id == null? Yes → isNew() = true
        // Actually works here because @Id is null
        // BUT: if you manually set an @Id with primitive @Version:
        //   @Id Long id = 100L; int version = 0;
        //   isNew() → @Version not null (0) → @Id not null → isNew() = false → merge()!
        return docRepo.save(doc);
    }
}
```

---

### Example 4 — Conflict Detection in REST API (Optimistic Concurrency)

```java
// REST pattern: client gets version, sends it back with updates
// Server uses version to detect concurrent modifications

@RestController
@RequestMapping("/api/articles")
public class ArticleController {

    // GET returns version in response:
    @GetMapping("/{id}")
    public ArticleResponse getArticle(@PathVariable Long id) {
        Article article = articleService.findById(id);
        return new ArticleResponse(
            article.getId(),
            article.getTitle(),
            article.getContent(),
            article.getVersion()  // version sent to client
        );
    }

    // PUT requires version in request (ETag / version field):
    @PutMapping("/{id}")
    public ArticleResponse updateArticle(
            @PathVariable Long id,
            @RequestBody ArticleUpdateRequest request) {
        // request contains: title, content, version

        try {
            Article updated = articleService.update(id, request);
            return ArticleResponse.from(updated);
        } catch (OptimisticLockingFailureException e) {
            throw new ResponseStatusException(
                HttpStatus.CONFLICT,
                "Article was modified by another user. " +
                "Please reload and resubmit your changes.");
        }
    }
}

// Service:
@Service
public class ArticleService {

    @Transactional
    public Article update(Long id, ArticleUpdateRequest request) {
        Article article = articleRepo.findById(id)
            .orElseThrow(ArticleNotFoundException::new);

        // Optional extra check (in addition to @Version):
        if (!article.getVersion().equals(request.getVersion())) {
            throw new OptimisticLockingFailureException(
                "Stale version: expected " + article.getVersion() +
                " but client sent " + request.getVersion());
        }

        article.setTitle(request.getTitle());
        article.setContent(request.getContent());
        return article;
        // @Version in UPDATE WHERE clause is the real protection
        // Manual version check above is extra safety
    }
}

// Request/Response DTOs:
public record ArticleUpdateRequest(
    String title,
    String content,
    Integer version  // client must send back the version it read
) {}
```

---

### Example 5 — Optimistic vs Pessimistic Side-by-Side

```java
public interface SeatRepository extends JpaRepository<Seat, Long> {

    // ── OPTIMISTIC approach ──────────────────────────────────────────────
    Optional<Seat> findById(Long id);
    // Standard find — @Version annotation on Seat does the conflict detection

    // ── PESSIMISTIC approach ─────────────────────────────────────────────
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT s FROM Seat s WHERE s.id = :id")
    Optional<Seat> findByIdForUpdate(@Param("id") Long id);
}

@Service
public class SeatBookingService {

    // ── OPTIMISTIC: low contention, retry-able ──────────────────────────
    @Retryable(retryFor = OptimisticLockingFailureException.class,
               maxAttempts = 3)
    @Transactional
    public void bookSeatOptimistic(Long seatId, String userId) {
        Seat seat = seatRepo.findById(seatId).get();
        // SELECT id, row, number, status, version FROM seats WHERE id=?

        if (seat.getStatus() != SeatStatus.AVAILABLE) {
            throw new SeatUnavailableException();
        }
        seat.setStatus(SeatStatus.BOOKED);
        seat.setBookedBy(userId);
        // At flush: UPDATE seats SET status='BOOKED', booked_by=?, version=N+1
        //           WHERE id=? AND version=N
        // If conflict: OptimisticLockException → @Retryable retries
    }

    // ── PESSIMISTIC: high contention, no retry needed ───────────────────
    @Transactional
    public void bookSeatPessimistic(Long seatId, String userId) {
        Seat seat = seatRepo.findByIdForUpdate(seatId).get();
        // SELECT ... FROM seats WHERE id=? FOR UPDATE
        // Other transactions BLOCKED here until this TX commits

        if (seat.getStatus() != SeatStatus.AVAILABLE) {
            throw new SeatUnavailableException();
        }
        seat.setStatus(SeatStatus.BOOKED);
        seat.setBookedBy(userId);
        // No version conflict possible — lock prevented concurrent writes
        // UPDATE at flush — guaranteed success (unless DB error)
    }
    // Lock released at TX commit
}
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ: Version Check SQL**
```java
@Entity
public class Counter {
    @Id Long id;
    int value;
    @Version Integer version;  // currently version=7 in DB
}
```

What SQL does Hibernate generate when a managed Counter entity is flushed after `counter.setValue(counter.getValue() + 1)`?

A) `UPDATE counters SET value=? WHERE id=?`  
B) `UPDATE counters SET value=?, version=? WHERE id=? AND version=?`  
C) `UPDATE counters SET value=?, version=? WHERE id=?`  
D) `SELECT version FROM counters WHERE id=?` then `UPDATE counters SET value=? WHERE id=?`  

**Answer: B** — Hibernate generates `UPDATE counters SET value=?, version=8 WHERE id=? AND version=7`. Both the new version value AND the version check in WHERE are part of the single UPDATE statement. No separate SELECT. If 0 rows affected (version mismatch), `OptimisticLockException` is thrown.

---

**Q2 — Select All That Apply: `@Version` Rules**

Which statements about `@Version` are TRUE?

A) The `@Version` field should never be set manually in application code  
B) `@Version` is incremented by bulk `@Modifying` `@Query` UPDATE statements  
C) Using primitive `int` for `@Version` may cause `isNew()` detection issues  
D) `@Version` field is incremented at flush time, not when the setter is called  
E) `@Version null` means the entity is new (supports `isNew()` detection)  
F) `OptimisticLockException` is thrown when 0 rows are affected by the version-checked UPDATE  

**Answer: A, C, D, E, F**
- B is false: bulk JPQL UPDATE bypasses entity lifecycle and does NOT increment `@Version`

---

**Q3 — Lock Mode Identification**

Which `LockModeType` acquires a database-level exclusive lock AND increments the `@Version` field even if no entity fields are modified?

A) `PESSIMISTIC_WRITE`  
B) `OPTIMISTIC_FORCE_INCREMENT`  
C) `PESSIMISTIC_FORCE_INCREMENT`  
D) `PESSIMISTIC_READ`  

**Answer: C** — `PESSIMISTIC_FORCE_INCREMENT` acquires a `SELECT ... FOR UPDATE` (exclusive lock) AND increments the `@Version` even if the entity is not otherwise modified. `PESSIMISTIC_WRITE` acquires the lock but does NOT force a version increment. `OPTIMISTIC_FORCE_INCREMENT` forces version increment but does NOT acquire a DB-level lock.

---

**Q4 — SKIP LOCKED Timeout Value**

Which `jakarta.persistence.lock.timeout` hint value causes the query to use `FOR UPDATE SKIP LOCKED`?

A) `0`  
B) `-1`  
C) `-2`  
D) `Integer.MAX_VALUE`  

**Answer: C** — Timeout value `-2` maps to `SKIP LOCKED` semantics. Value `0` = `NOWAIT` (fail immediately if locked). Value `-1` = wait indefinitely (default). Value `N > 0` = wait up to N milliseconds.

---

**Q5 — Optimistic Lock Exception Timing**
```java
@Transactional
public void update(Long id) {
    Product p = productRepo.findById(id).get(); // version=3 in DB
    p.setName("Updated");
    // At this point, concurrent TX has already updated version to 4
    log.info("About to save");
    productRepo.save(p);  // Line A
    log.info("Saved");    // Line B
}
```

Where does `OptimisticLockException` occur?

A) At `findById()` — concurrent modification detected on read  
B) At `p.setName()` — field modification triggers version check  
C) At `productRepo.save()` — Hibernate flushes and detects version mismatch  
D) At transaction commit — after all method logic completes  

**Answer: C (or D — depending on flush timing)**

More precisely: `OptimisticLockException` fires when Hibernate executes the UPDATE and sees 0 rows affected. This happens during **flush**. `save()` on a managed entity is a no-op (entity is already in PC) — but it may trigger a flush. More accurately, flush happens either explicitly or at commit time. If `FlushMode.AUTO` and a query follows `save()`, flush happens at that query. Otherwise, at commit.

**Exam answer: D** — At transaction commit (or flush before a query). Not at `save()` itself. Not at field modification. Not at read time.

---

**Q6 — Deadlock Scenario**
```java
// Thread 1:
@Transactional
public void thread1() {
    productRepo.findByIdForUpdate(1L); // Lock product 1
    Thread.sleep(100);
    productRepo.findByIdForUpdate(2L); // Lock product 2
}

// Thread 2:
@Transactional
public void thread2() {
    productRepo.findByIdForUpdate(2L); // Lock product 2
    Thread.sleep(100);
    productRepo.findByIdForUpdate(1L); // Lock product 1
}
```

What happens when both threads execute concurrently?

A) Thread 2 waits for Thread 1 to release lock on product 2 — no deadlock  
B) Deadlock — Thread 1 waits for product 2 (held by Thread 2), Thread 2 waits for product 1 (held by Thread 1)  
C) `OptimisticLockException` on one thread  
D) Both threads succeed — `FOR UPDATE` allows concurrent readers  

**Answer: B** — Classic deadlock from reversed lock acquisition order. Prevention: always lock resources in a consistent canonical order (e.g., always lowest ID first).

---

**Q7 — `@Version` and Bulk Update**
```java
// DB: product id=5, version=10
Product product = productRepo.findById(5L).get(); // version=10 in PC

// Bulk update:
productRepo.updateAllPrices(new BigDecimal("1.1")); // @Modifying query, no version bump

// Modify the loaded entity:
product.setName("New Name");
productRepo.save(product);
// flush: UPDATE products SET name=?, version=11 WHERE id=5 AND version=10
```

Does this UPDATE succeed?

A) No — `OptimisticLockException` because bulk update changed the row  
B) Yes — bulk UPDATE did not change `version`, so WHERE version=10 still matches  
C) No — Hibernate detects the bulk update and invalidates the PC entry  
D) Yes — Hibernate re-reads version before flushing  

**Answer: B** — Bulk JPQL UPDATE bypasses entity lifecycle and does NOT increment `@Version`. The DB row still has `version=10`. The version-checked UPDATE `WHERE version=10` succeeds. This is the **silent consistency gap** with bulk operations — they bypass optimistic locking protection entirely.

---

**Q8 — `OPTIMISTIC_FORCE_INCREMENT` Use Case**
```java
@Entity
public class Order {
    @Id Long id;
    @Version Integer version;
    @OneToMany(mappedBy="order") List<OrderItem> items;
}
```

Why might you use `OPTIMISTIC_FORCE_INCREMENT` when adding an `OrderItem`?

A) To prevent anyone from reading the Order while items are being added  
B) To increment Order's version when its state changes indirectly via item addition, enabling conflict detection for readers of the Order aggregate  
C) To lock the OrderItem table  
D) To ensure the OrderItem INSERT uses the version check  

**Answer: B** — Adding an `OrderItem` modifies child rows but not the `Order` entity itself — Order's version would not normally increment. `OPTIMISTIC_FORCE_INCREMENT` on the Order causes its version to increment even though Order fields didn't change. Concurrent transactions that read Order at version N and try to commit at version N+something will detect the conflict, ensuring aggregate consistency is preserved.

---

## 4️⃣ Trick Analysis

**Version check is in the UPDATE WHERE clause, not a separate SELECT (Q1)**:
Many developers assume Hibernate does a `SELECT version FROM table WHERE id=?` before the UPDATE to check for conflicts. It does not. The version check `AND version=?` is embedded in the UPDATE itself. Hibernate knows the version was stale only when it checks `rows affected = 0`. This means: no lock is held between the initial read and the final write — another transaction can slip in. This is the defining characteristic of optimistic locking: it detects conflicts rather than preventing them.

**Bulk JPQL UPDATE silently bypasses `@Version` (Q7)**:
This is a silent, dangerous gap. A `@Modifying @Query` UPDATE runs directly against the DB without touching entity lifecycle. It does not increment `@Version`. Entities loaded in the Persistence Context remain at the old version. When those entities are later flushed, the UPDATE `WHERE version=old_version` succeeds — even though the bulk operation modified the row. Both operations commit, potentially with inconsistent combined state. Always use `clearAutomatically=true` after bulk updates and be aware that optimistic locking provides no protection against bulk operations.

**`OptimisticLockException` fires at flush, not at `save()` (Q5)**:
`save()` on a managed entity is effectively a no-op — the entity is already tracked by the Persistence Context. `OptimisticLockException` fires when Hibernate executes the UPDATE statement and gets back `rows affected = 0`. This happens at flush time — either before a query with FlushMode.AUTO, or at transaction commit. The exception is thrown INSIDE the `@Transactional` boundary, causing rollback. This is why retries require a completely new transaction (new read, new version snapshot).

**Primitive `@Version` isNew() interaction (Q2, C)**:
`SimpleJpaRepository.isNew()` checks `@Version` field first. For wrapper `Integer version`, `null` means new. For primitive `int version`, the value `0` is NOT null — it is a valid non-null primitive. If the entity also has a null `@Id`, Hibernate falls through to the `@Id` null check. But if BOTH `@Id` and `int version` are non-null (which can happen with manually assigned IDs), `isNew()` returns false regardless of intent. Always use wrapper types (`Integer`, `Long`) for `@Version`.

**`PESSIMISTIC_FORCE_INCREMENT` vs `OPTIMISTIC_FORCE_INCREMENT` (Q3)**:
These two are tested as a pair. `OPTIMISTIC_FORCE_INCREMENT` increments version at commit but acquires NO database lock — other transactions can still read and write the row concurrently (they just get an OptimisticLockException if they conflict). `PESSIMISTIC_FORCE_INCREMENT` acquires a `FOR UPDATE` database lock (blocking concurrent writes) AND increments version. The word "Pessimistic" always means a DB-level lock is involved.

**SKIP LOCKED timeout value is -2 (Q4)**:
The three special values are easy to confuse: `0` = NOWAIT, `-1` = wait forever, `-2` = SKIP LOCKED. SKIP LOCKED is extremely useful for implementing reliable job queues in SQL — multiple workers can each call the same query and get non-overlapping batches. Each worker gets the rows not currently locked by another worker. This is a production-grade pattern for task distribution.

---

## 5️⃣ Summary Sheet

### `@Version` Rules Reference

```
Type:         Use wrapper types: Integer, Long (NOT int, long)
              null = new entity (supports isNew() detection)
              0 = first persisted version (after INSERT)

Management:   NEVER set manually — Hibernate manages exclusively
              Incremented by: all entity-level UPDATEs at flush
              NOT incremented by: bulk @Modifying @Query UPDATEs

SQL pattern:  UPDATE table SET col1=?, version=N+1 WHERE id=? AND version=N
              rows affected = 0 → OptimisticLockException

Exception:    OptimisticLockException (JPA)
              → wrapped in JpaOptimisticLockingFailureException (Spring)
              → extends OptimisticLockingFailureException (Spring DAO)
              → fires at FLUSH time, not at read or setField time

Retry:        Each retry MUST be a new transaction with a fresh entity read
              Never retry with a stale/detached entity instance
```

### `LockModeType` Complete Reference

| LockModeType | DB Lock? | Version Incremented? | SQL |
|---|---|---|---|
| `NONE` | No | No | Standard SELECT |
| `OPTIMISTIC` | No | On commit (via version check) | Standard SELECT |
| `OPTIMISTIC_FORCE_INCREMENT` | No | YES — on commit even if no change | Standard SELECT + forced version UPDATE |
| `PESSIMISTIC_READ` | Shared (FOR SHARE) | No | SELECT ... FOR SHARE |
| `PESSIMISTIC_WRITE` | Exclusive (FOR UPDATE) | No | SELECT ... FOR UPDATE |
| `PESSIMISTIC_FORCE_INCREMENT` | Exclusive (FOR UPDATE) | YES | SELECT ... FOR UPDATE + version UPDATE |

### Lock Timeout Values

```
0    → NOWAIT: fail immediately if lock unavailable
-1   → wait indefinitely (default)
-2   → SKIP LOCKED: skip already-locked rows
N>0  → wait up to N milliseconds (dialect-dependent)

QueryHint: "jakarta.persistence.lock.timeout"
```

### Optimistic vs Pessimistic Decision Matrix

| Criterion | Optimistic | Pessimistic |
|---|---|---|
| Contention level | Low | High |
| Operation duration | Long (user think-time) | Short (sub-second) |
| On conflict | Retry (new TX) | Block (wait for lock) |
| DB connections held | Not during user wait | Yes — while thinking/waiting |
| Throughput (low contention) | High | Lower (lock overhead) |
| Throughput (high contention) | Low (many retries) | Higher (queue, no retry) |
| Overselling risk | Possible (if retry limit hit) | Impossible (lock prevents) |
| Distributed systems | Works (version in response) | Difficult (no shared lock mgr) |

### Key Rules

```
1.  @Version check is in UPDATE WHERE clause — not a separate SELECT
2.  OptimisticLockException fires at FLUSH time (not at read, not at setField)
3.  Bulk JPQL UPDATE bypasses @Version — silent consistency gap
4.  Always use wrapper Integer/Long for @Version (not primitive int/long)
5.  Never set @Version field manually — Hibernate owns it
6.  Retry requires new @Transactional + fresh entity read — never retry stale entity
7.  PESSIMISTIC_WRITE = FOR UPDATE (exclusive, blocks all others)
8.  PESSIMISTIC_READ = FOR SHARE (shared, blocks writers only)
9.  PESSIMISTIC_FORCE_INCREMENT = FOR UPDATE + version increment
10. OPTIMISTIC_FORCE_INCREMENT = no DB lock + forced version increment at commit
11. Lock timeout 0 = NOWAIT | -1 = wait forever | -2 = SKIP LOCKED
12. REQUIRES_NEW + PESSIMISTIC_WRITE = second DB connection needed (deadlock if pool=1)
13. Deadlock prevention: always acquire locks in consistent canonical order
14. Lock upgrade (READ → WRITE) is deadlock-prone — acquire at highest level from start
15. @Lock on repository method applies LockModeType to that query
16. SKIP LOCKED enables reliable job queue: workers get non-overlapping row sets
```

---
