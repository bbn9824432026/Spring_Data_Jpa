# 🏛️ TOPIC 14 — Transactions in Spring Data JPA

---

## 1️⃣ Conceptual Explanation

### What a Transaction Is — The ACID Contract

A transaction is a unit of work that is either committed entirely or rolled back entirely. The database guarantees four properties:

```
A — Atomicity:   All operations succeed or none do
C — Consistency: DB moves from one valid state to another
I — Isolation:   Concurrent transactions don't interfere (to a configurable degree)
D — Durability:  Committed data survives crashes (written to durable storage)
```

Spring's transaction management is an **abstraction** over the underlying resource (JDBC connection, JPA EntityManager, JMS session). The same `@Transactional` annotation works for JDBC, JPA, and JMS transactions — the `PlatformTransactionManager` implementation handles the resource-specific details.

---

### The Proxy Mechanism — How `@Transactional` Works Internally

`@Transactional` is implemented via Spring AOP proxies. When you call a `@Transactional` method, you are not calling the method directly — you are calling a proxy that wraps the method with transaction management logic.

```
External caller
      │
      ▼
┌─────────────────────────────┐
│  Spring AOP Proxy           │
│  (TransactionInterceptor)   │
│                             │
│  1. TransactionManager.     │
│     getTransaction()        │
│     → open TX or join existing
│                             │
│  2. ──► target.method() ◄── │  ← actual business logic
│                             │
│  3. On success:             │
│     TransactionManager.     │
│     commit()                │
│                             │
│  On RuntimeException:       │
│     TransactionManager.     │
│     rollback()              │
└─────────────────────────────┘
      │
      ▼
  Caller receives return value
  (or exception propagates)
```

**The proxy is created at application startup.** When you inject a `@Service` bean that has `@Transactional` methods, what you receive is a CGLIB proxy (or JDK dynamic proxy if the class implements an interface), not the actual class instance.

```java
@Service
public class OrderService {
    @Transactional
    public void placeOrder(OrderRequest req) { ... }
}

// What Spring injects:
// OrderService$$SpringCGLIB$$0 (proxy)
// NOT: new OrderService()

// The proxy:
// 1. Intercepts placeOrder()
// 2. Begins transaction
// 3. Delegates to real OrderService.placeOrder()
// 4. Commits or rolls back
```

---

### The Self-Invocation Trap — The Most Critical Transaction Trap

When a method within the same class calls another `@Transactional` method on `this`, the call **bypasses the proxy**. The transaction boundary of the called method is completely ignored.

```java
@Service
public class OrderService {

    @Transactional(propagation = Propagation.REQUIRED)
    public void processOrder(Long orderId) {
        // ... some work ...
        this.sendConfirmation(orderId);  // ← BYPASSES PROXY!
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendConfirmation(Long orderId) {
        // Developer expects a NEW transaction here
        // REALITY: runs in the SAME transaction as processOrder()
        // REQUIRES_NEW is completely ignored
        // If this throws, processOrder's transaction rolls back too
    }
}
```

**Why this happens:** `this.sendConfirmation()` calls the real object's method directly — not through the proxy. Only calls that go through the proxy trigger the transaction interceptor.

**Fixes:**

```java
// Fix 1: Inject self (Spring 4.3+):
@Service
public class OrderService {
    @Autowired
    private OrderService self;  // Spring injects the PROXY

    @Transactional
    public void processOrder(Long orderId) {
        self.sendConfirmation(orderId);  // goes through proxy ✓
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendConfirmation(Long orderId) { ... }
}

// Fix 2: Extract to separate bean:
@Service
public class OrderService {
    @Autowired NotificationService notificationService; // separate bean

    @Transactional
    public void processOrder(Long orderId) {
        notificationService.sendConfirmation(orderId); // different proxy ✓
    }
}

// Fix 3: ApplicationContext.getBean() — rare, ugly:
@Autowired ApplicationContext ctx;
OrderService proxy = ctx.getBean(OrderService.class);
proxy.sendConfirmation(orderId);
```

---

### `@Transactional` Propagation Behaviours — Complete Reference

Propagation defines what happens to the transaction when a `@Transactional` method is called — specifically: what relationship it has with any existing transaction in the calling thread.

```
Thread-local transaction state:
  Spring binds the current transaction to the thread via TransactionSynchronizationManager.
  Each thread has its own transaction context.
  Passing entities between threads = passing detached entities (no transaction in new thread).
```

---

#### `REQUIRED` — The Default

```
Rule: Use existing TX if present; create new TX if absent.

Scenario A — No active TX:
  Caller (no TX) → method(@Transactional REQUIRED)
  → Spring creates NEW transaction
  → Method runs in new TX
  → Method commits at end

Scenario B — Active TX exists:
  Caller (@Transactional) → method(@Transactional REQUIRED)
  → Spring JOINS the existing transaction (no new TX created)
  → Method runs in caller's TX
  → No commit at method end (caller's TX continues)
  → Caller's TX commits or rolls back at its boundary

Key trap — rollback in joined TX:
  If the inner method (REQUIRED, joined) throws RuntimeException:
  → Inner TX marks the SHARED transaction as rollback-only
  → Even if caller catches the exception:
    caller's TX is now marked rollback-only
    → commit attempt at caller boundary → UnexpectedRollbackException
```

```java
@Transactional  // REQUIRED (default)
public void outer() {
    repository.save(entityA);
    try {
        inner();  // REQUIRED — joins outer TX
    } catch (RuntimeException e) {
        // We caught the exception — but TX is already rollback-only!
    }
    repository.save(entityB);  // still executes
    // At commit: UnexpectedRollbackException!
    // Because inner() marked TX as rollback-only
}

@Transactional  // REQUIRED — joins outer TX
public void inner() {
    throw new RuntimeException("fail");
    // Marks the SHARED TX as rollback-only
}
```

---

#### `REQUIRES_NEW` — Always Independent Transaction

```
Rule: Always create a NEW transaction. Suspend existing TX if present.

Scenario A — No active TX:
  → Same as REQUIRED: new TX created

Scenario B — Active TX exists:
  → Existing TX is SUSPENDED
  → Brand new independent TX created
  → Method runs in new TX
  → Method's TX commits/rolls back independently
  → Suspended TX is RESUMED after method returns
  → Caller's TX continues regardless of inner TX outcome

Use case: audit logging, notification sending — must commit regardless
  of whether the outer business TX rolls back
```

```java
@Transactional  // outer TX
public void placeOrder() {
    orderRepo.save(order);
    auditService.logOrderPlaced(order);  // REQUIRES_NEW — independent TX
    // Even if placeOrder() rolls back: audit log is COMMITTED
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void logOrderPlaced(Order order) {
    auditRepo.save(new AuditEntry(...));
    // Commits independently when this method returns
    // If outer placeOrder() rolls back: this audit entry STAYS committed
}

// Resource implication: REQUIRES_NEW needs a SECOND DB connection
// from the connection pool (outer TX holds one, inner needs another)
// With pool size=1: DEADLOCK
```

---

#### `SUPPORTS` — Optional Transaction

```
Rule: Join TX if present; execute without TX if absent.

Scenario A — No active TX: runs without transaction (no ACID guarantees)
Scenario B — Active TX: joins and runs within it

Use case: read-only methods that can work both transactionally and non-transactionally
Warning: if called without TX, JPA operations may fail (no EntityManager TX context)
```

---

#### `NOT_SUPPORTED` — Suspend if Present

```
Rule: Execute WITHOUT a transaction. Suspend existing TX if present.

Scenario A — No active TX: runs without TX
Scenario B — Active TX: SUSPENDS existing TX, runs without TX, resumes TX after

Use case: operations that must not participate in a transaction
  (e.g., calling a remote service where TX semantics don't apply)
```

---

#### `MANDATORY` — Must Have Existing Transaction

```
Rule: MUST run within an existing transaction. Throw if no TX active.

Scenario A — No active TX: IllegalTransactionStateException immediately
Scenario B — Active TX: joins and runs in it

Use case: methods that should NEVER be entry points (always called from within TX)
  Enforces a programming contract at runtime
```

```java
@Transactional(propagation = Propagation.MANDATORY)
public void updateBalance(Long accountId, BigDecimal amount) {
    // Will throw if called without an active transaction
    // Ensures this method is always called from within a TX
}
```

---

#### `NEVER` — Must Not Have Transaction

```
Rule: Must NOT run within a transaction. Throw if TX is active.

Scenario A — No active TX: runs fine
Scenario B — Active TX: IllegalTransactionStateException

Use case: methods that must never run in a TX (e.g., long-running batch reads)
```

---

#### `NESTED` — Savepoint-Based Sub-Transaction

```
Rule: Create a NESTED transaction using a savepoint within the existing TX.

Scenario A — No active TX: behaves like REQUIRED (creates new TX)
Scenario B — Active TX:
  → SAVEPOINT created within the outer TX
  → Inner method runs within same physical TX
  → If inner throws: ROLLBACK TO SAVEPOINT (partial rollback)
  → Outer TX continues with its state restored to the savepoint point
  → Outer TX can still commit

Key difference from REQUIRES_NEW:
  REQUIRES_NEW: separate connection, separate TX, independent commit
  NESTED: same connection, same TX, savepoint-based partial rollback

Limitation: NESTED requires JDBC savepoint support
  → Most major DBs support it (PostgreSQL, MySQL/InnoDB, Oracle)
  → Not all JPA providers expose savepoints via @Transactional
  → Spring's DataSourceTransactionManager supports it
  → JpaTransactionManager support is limited — check your provider
```

```java
@Transactional
public void processOrderBatch(List<Order> orders) {
    for (Order order : orders) {
        try {
            processOrder(order);  // NESTED
        } catch (Exception e) {
            // NESTED rolled back to savepoint
            // This order failed, but outer TX continues
            // Next order can still be processed
        }
    }
    // Outer TX commits all successfully processed orders
}

@Transactional(propagation = Propagation.NESTED)
public void processOrder(Order order) {
    orderRepo.save(order);
    // SAVEPOINT here — if exception: rollback to this savepoint only
}
```

---

### Propagation Comparison Matrix

```
┌─────────────────┬─────────────────────────────┬────────────────────────────┐
│ Propagation     │ If TX exists                │ If no TX exists            │
├─────────────────┼─────────────────────────────┼────────────────────────────┤
│ REQUIRED        │ Join existing TX            │ Create new TX              │
│ REQUIRES_NEW    │ Suspend + new independent TX│ Create new TX              │
│ SUPPORTS        │ Join existing TX            │ No TX (non-transactional)  │
│ NOT_SUPPORTED   │ Suspend + no TX             │ No TX                      │
│ MANDATORY       │ Join existing TX            │ IllegalTransactionStateEx  │
│ NEVER           │ IllegalTransactionStateEx   │ No TX                      │
│ NESTED          │ Savepoint sub-transaction   │ Create new TX              │
└─────────────────┴─────────────────────────────┴────────────────────────────┘
```

---

### Isolation Levels — Concurrent Transaction Phenomena

Isolation controls what concurrent transactions can see from each other. Higher isolation = fewer anomalies = more locking overhead = lower throughput.

**The four phenomena that isolation prevents:**

```
Dirty Read:
  TX A reads data written by TX B (not yet committed).
  If TX B rolls back: TX A has read data that never existed.

Non-Repeatable Read:
  TX A reads a row. TX B updates+commits that row.
  TX A reads the same row again: different value.
  (Same row, different reads within one TX)

Phantom Read:
  TX A reads a set of rows matching a condition.
  TX B inserts a new row matching that condition + commits.
  TX A re-executes the same query: gets a new row it didn't see before.
  (Different rows in result set between two reads of the same query)

Lost Update:
  TX A reads value X. TX B reads value X.
  TX A writes X+1. TX B writes X+1.
  Result: X+1 instead of X+2 (TX A's update is lost).
  (Not in the SQL standard phenomena but a real concurrency issue)
```

**The four isolation levels:**

```
READ_UNCOMMITTED (lowest isolation):
  Allows: Dirty reads, Non-repeatable reads, Phantom reads
  Prevents: nothing
  Use: Rarely — approximate counters, reports where stale is OK

READ_COMMITTED (PostgreSQL/Oracle default):
  Prevents: Dirty reads
  Allows: Non-repeatable reads, Phantom reads
  Use: Most OLTP workloads — good balance

REPEATABLE_READ (MySQL InnoDB default):
  Prevents: Dirty reads, Non-repeatable reads
  Allows: Phantom reads (in theory; MySQL InnoDB prevents phantoms too)
  Use: Financial calculations within one transaction

SERIALIZABLE (highest isolation):
  Prevents: Dirty reads, Non-repeatable reads, Phantom reads
  Mechanism: Full range locking or optimistic multi-version
  Use: Accounting, where no anomaly is acceptable
  Cost: Highest locking overhead, lowest throughput
```

**Spring isolation configuration:**

```java
@Transactional(isolation = Isolation.READ_COMMITTED)   // most common
@Transactional(isolation = Isolation.REPEATABLE_READ)
@Transactional(isolation = Isolation.SERIALIZABLE)
@Transactional(isolation = Isolation.DEFAULT)  // use DB default (common choice)
```

**DB defaults:**
```
PostgreSQL: READ_COMMITTED
MySQL (InnoDB): REPEATABLE_READ
Oracle: READ_COMMITTED
SQL Server: READ_COMMITTED
H2: READ_COMMITTED
```

---

### Rollback Rules — What Triggers a Rollback

By default, Spring rolls back on **unchecked exceptions** (`RuntimeException` and `Error`) and commits on **checked exceptions**.

```java
// Default behavior:
@Transactional
public void process() throws IOException {
    // RuntimeException → ROLLBACK
    throw new IllegalArgumentException("bad input");  // → rollback

    // Error → ROLLBACK
    throw new OutOfMemoryError();  // → rollback

    // Checked exception → COMMIT (!)
    throw new IOException("file not found");  // → COMMIT (surprising!)
}
```

**Custom rollback rules:**

```java
// Rollback on specific checked exception:
@Transactional(rollbackFor = IOException.class)
public void process() throws IOException {
    throw new IOException();  // → ROLLBACK (explicitly configured)
}

// Rollback on any exception (checked or unchecked):
@Transactional(rollbackFor = Exception.class)
public void safeProcess() throws Exception { ... }

// Do NOT rollback on specific runtime exception:
@Transactional(noRollbackFor = OptimisticLockingFailureException.class)
public void retryableProcess() {
    // OptimisticLockingFailureException → COMMIT (caller will retry)
    // Other RuntimeExceptions → ROLLBACK (default)
}

// Combined:
@Transactional(
    rollbackFor = {IOException.class, SQLException.class},
    noRollbackFor = InsufficientStockException.class
)
public void complexProcess() throws Exception { ... }
```

**Rollback rule resolution priority:**

```
Most specific exception match wins:
  If IOException is configured for rollback AND Exception is configured for noRollback:
  IOException is more specific → rollback wins

  Spring walks up the exception hierarchy and takes the first match
```

---

### `readOnly = true` — What It Actually Does

```java
@Transactional(readOnly = true)
public List<User> findAllUsers() { ... }
```

**What `readOnly=true` actually does (not what most developers think):**

```
1. Sets FlushMode to MANUAL on the EntityManager:
   → Dirty checking is DISABLED
   → Hibernate does NOT compare snapshots at flush time
   → Performance improvement: no snapshot creation, no comparison

2. Passes hint to JDBC Connection:
   → connection.setReadOnly(true)
   → Some JDBC drivers and connection pools use this:
     → Route to read replica (HikariCP, some proxies)
     → Some DBs optimize query plans for read-only connections

3. Does NOT enforce that only reads happen:
   → You CAN call save() inside a readOnly=true transaction
   → Hibernate will NOT flush (FlushMode.MANUAL)
   → No exception is thrown at the Spring layer
   → The save() is effectively silently ignored
   → (The entity enters PC as managed but never flushes)

4. Does NOT prevent writes at the DB level:
   → readOnly=true is a HINT, not an enforcement
   → The DB does not reject writes from a readOnly=true transaction
```

**Performance impact of `readOnly=true`:**

```
Without readOnly=true:
  1. Hibernate takes a deep snapshot of each loaded entity
  2. At flush: compares every field with snapshot (dirty checking)
  3. For 1000 entities with 20 fields: 20,000 comparisons

With readOnly=true:
  1. No snapshots taken (FlushMode.MANUAL, no flush will happen)
  2. No dirty checking at transaction end
  3. Memory savings: no snapshot objects in heap
  4. CPU savings: no comparison work
  5. Significant improvement for read-heavy, large-result-set queries
```

---

### `@Transactional` on Class vs Method — Precedence

```java
@Service
@Transactional(readOnly = true)          // class-level default
public class ProductService {

    // Inherits class-level: readOnly=true
    public List<Product> findAll() { ... }

    // Overrides class-level: read-write
    @Transactional                        // readOnly=false (default)
    public Product save(Product p) { ... }

    // Overrides class-level with specific config:
    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        timeout = 30
    )
    public void processExpensiveOperation() { ... }
}

// Precedence rule: METHOD-level @Transactional overrides CLASS-level completely
// Not merged — complete replacement
// If method has @Transactional, class-level is ignored for that method
```

---

### Transaction Timeout and Exceptions

```java
@Transactional(timeout = 30)  // 30 seconds
public void longRunningOperation() {
    // If this method takes > 30 seconds:
    // TransactionTimedOutException is thrown
    // Transaction is rolled back automatically
}

// timeout = -1: no timeout (default)
// timeout is in SECONDS
// Implemented at the Connection level: statement timeout on the DB side
```

---

### `TransactionTemplate` — Programmatic Transactions

When annotation-based transactions aren't sufficient (e.g., you need fine-grained control within a loop):

```java
@Service
public class BatchProcessingService {

    @Autowired
    TransactionTemplate txTemplate;  // or inject PlatformTransactionManager

    public void processBatch(List<Item> items) {

        // Execute with result:
        List<Item> saved = txTemplate.execute(status -> {
            // status: TransactionStatus
            List<Item> result = new ArrayList<>();
            for (Item item : items) {
                try {
                    result.add(itemRepo.save(item));
                } catch (Exception e) {
                    status.setRollbackOnly();  // mark for rollback
                    return null;
                }
            }
            return result;
        });

        // Execute without result (void):
        txTemplate.executeWithoutResult(status -> {
            auditRepo.save(new AuditEntry("batch processed"));
        });
    }
}

// TransactionTemplate configuration:
@Bean
public TransactionTemplate transactionTemplate(PlatformTransactionManager txMgr) {
    TransactionTemplate template = new TransactionTemplate(txMgr);
    template.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRES_NEW);
    template.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
    template.setTimeout(30);
    template.setReadOnly(false);
    return template;
}
```

---

### EntityManager and Transaction Binding

Spring binds the EntityManager to the current transaction using `TransactionSynchronizationManager`:

```
TX starts:
  → EntityManager created (or retrieved from ThreadLocal)
  → EntityManager bound to current TX (via EntityManagerHolder in ThreadLocal)
  → All @PersistenceContext-injected EM references use this shared TX-bound EM

TX commits:
  → em.flush() called (unless readOnly)
  → JDBC commit executed
  → EntityManager closed (or returned to scope)
  → ThreadLocal cleared

TX rolls back:
  → JDBC rollback executed
  → EntityManager discarded (not returned to scope)
  → All managed entities become detached (they were managed by closed EM)
```

**Why two `@Transactional` methods in the same thread share the same EntityManager:**

```java
@Transactional
public void outer() {
    User user = userRepo.findById(1L).get();
    inner();  // REQUIRED — joins outer TX
    // user is still MANAGED (same EM, same TX)
}

@Transactional  // REQUIRED — joins outer TX
public void inner() {
    User user2 = userRepo.findById(1L).get();
    // user2 == user (same instance — same EM, same identity map!)
}
// Both methods share ONE EntityManager bound to the outer TX
```

---

### `@TransactionalEventListener` — Transaction-Aware Events

```java
@Service
public class OrderService {
    @Autowired ApplicationEventPublisher publisher;

    @Transactional
    public void placeOrder(Order order) {
        orderRepo.save(order);
        publisher.publishEvent(new OrderPlacedEvent(order));
        // Event published but NOT handled yet
    }
}

@Component
public class OrderEventHandler {

    // Default: AFTER_COMMIT — handler runs after TX commits:
    @TransactionalEventListener
    public void handleOrderPlaced(OrderPlacedEvent event) {
        // Runs AFTER orderService.placeOrder() TX commits
        // Order is now durably saved in DB
        emailService.sendConfirmation(event.getOrder());
    }

    // Other phases:
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void beforeCommit(OrderPlacedEvent event) { ... }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void onRollback(OrderPlacedEvent event) { ... }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
    public void afterCompletion(OrderPlacedEvent event) { ... }
    // AFTER_COMPLETION fires regardless of commit or rollback
}

// CRITICAL: @TransactionalEventListener AFTER_COMMIT runs OUTSIDE the original TX
// If handler needs its own TX: add @Transactional on handler method
// (creates new TX via REQUIRED propagation — no TX active at that point)
```

---

## 2️⃣ Code Examples

### Example 1 — Propagation Deep Dive with Trace

```java
@Service
public class AccountService {

    @Transactional  // REQUIRED
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        // TX A begins here

        Account from = accountRepo.findById(fromId).get(); // SELECT
        Account to   = accountRepo.findById(toId).get();   // SELECT

        debit(from, amount);   // REQUIRED — joins TX A
        credit(to, amount);    // REQUIRED — joins TX A
        auditTransfer();       // REQUIRES_NEW — suspends TX A, creates TX B

        // TX A commits here (includes debit + credit)
    }

    @Transactional(propagation = Propagation.REQUIRED)
    public void debit(Account account, BigDecimal amount) {
        // Joined TX A — no new TX
        if (account.getBalance().compareTo(amount) < 0)
            throw new InsufficientFundsException();
        account.setBalance(account.getBalance().subtract(amount));
        // Dirty checking: UPDATE fired at TX A flush
    }

    @Transactional(propagation = Propagation.REQUIRED)
    public void credit(Account account, BigDecimal amount) {
        // Joined TX A — no new TX
        account.setBalance(account.getBalance().add(amount));
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void auditTransfer() {
        // TX A SUSPENDED
        // TX B begins (new connection from pool)
        auditRepo.save(new AuditRecord("TRANSFER", LocalDateTime.now()));
        // TX B commits
        // TX A RESUMED
    }
}
```

---

### Example 2 — Self-Invocation Trap and All Fixes

```java
// ── BROKEN: self-invocation bypasses proxy ───────────────────────────────
@Service
public class EmailService_BROKEN {

    @Transactional
    public void sendBatch(List<Email> emails) {
        for (Email email : emails) {
            this.sendSingle(email);  // ← calls real object, not proxy
            // REQUIRES_NEW is completely ignored!
        }
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendSingle(Email email) {
        emailRepo.save(email);
        smtpClient.send(email);
        // Developer expects independent TX per email
        // REALITY: runs in outer TX — if one fails, ALL fail
    }
}

// ── FIX 1: Self-injection (Spring injects proxy) ─────────────────────────
@Service
public class EmailService_Fix1 {

    @Lazy @Autowired
    private EmailService_Fix1 self;  // @Lazy breaks circular dependency

    @Transactional
    public void sendBatch(List<Email> emails) {
        for (Email email : emails) {
            try {
                self.sendSingle(email);  // calls proxy ✓
            } catch (Exception e) {
                log.warn("Failed to send email {}", email.getId());
            }
        }
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendSingle(Email email) {
        emailRepo.save(email);
        smtpClient.send(email);
        // Truly independent TX — fails independently ✓
    }
}

// ── FIX 2: Extract to separate Spring bean ───────────────────────────────
@Service
public class EmailBatchService {
    @Autowired EmailSenderService emailSenderService; // separate bean

    @Transactional
    public void sendBatch(List<Email> emails) {
        for (Email email : emails) {
            try {
                emailSenderService.sendSingle(email); // different proxy ✓
            } catch (Exception e) {
                log.warn("Failed: {}", email.getId());
            }
        }
    }
}

@Service
public class EmailSenderService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendSingle(Email email) {
        emailRepo.save(email);
        smtpClient.send(email);
    }
}
```

---

### Example 3 — Rollback Rules and Exception Hierarchy

```java
// Custom exceptions:
public class BusinessRuleException extends RuntimeException { ... } // unchecked
public class PaymentException extends Exception { ... }            // checked
public class FraudException extends PaymentException { ... }       // checked

@Service
public class PaymentService {

    // Default: rollback on RuntimeException, commit on checked:
    @Transactional
    public void processDefault() throws PaymentException {
        throw new PaymentException("declined"); // → COMMITS (checked, no rollback config)
        // Payment "fails" but TX commits with potentially partial state
        // This is often a bug!
    }

    // Explicit rollback on checked exception:
    @Transactional(rollbackFor = PaymentException.class)
    public void processSafe() throws PaymentException {
        throw new PaymentException("declined"); // → ROLLBACK ✓
    }

    // Rollback on all:
    @Transactional(rollbackFor = Exception.class)
    public void processAll() throws Exception {
        throw new IOException("network"); // → ROLLBACK ✓
    }

    // noRollbackFor — commit despite RuntimeException:
    @Transactional(noRollbackFor = BusinessRuleException.class)
    public void processWithLogging() {
        try {
            orderRepo.save(order);
        } catch (BusinessRuleException e) {
            // Log the rule violation but COMMIT the order record anyway
            auditRepo.save(new AuditRecord("RULE_VIOLATION", e.getMessage()));
            throw e; // re-throw — but TX commits because noRollbackFor
        }
    }

    // FraudException extends PaymentException:
    // rollbackFor = PaymentException.class → also rolls back FraudException ✓
    @Transactional(rollbackFor = PaymentException.class)
    public void detectFraud() throws FraudException {
        throw new FraudException("fraud detected"); // → ROLLBACK ✓ (subclass match)
    }
}
```

---

### Example 4 — REQUIRED Propagation Rollback-Only Trap

```java
@Service
public class OrderService {

    @Transactional  // TX A begins
    public void processOrders(List<Long> orderIds) {
        for (Long id : orderIds) {
            try {
                processOne(id);  // REQUIRED — joins TX A
            } catch (RuntimeException e) {
                // We caught the exception!
                // BUT: processOne() already marked TX A as rollback-only
                log.error("Order {} failed: {}", id, e.getMessage());
                // Continuing the loop...
            }
        }
        // At commit: Spring detects TX is rollback-only
        // → UnexpectedRollbackException thrown HERE
        // ALL orders are rolled back, even the successful ones!
    }

    @Transactional  // REQUIRED — joins TX A
    public void processOne(Long orderId) {
        orderRepo.findById(orderId).ifPresent(order -> {
            order.setStatus(OrderStatus.PROCESSING);
        });
        externalService.validate(orderId); // throws RuntimeException
        // → TX A marked as rollback-only
    }
}

// CORRECT fix — use REQUIRES_NEW for independent per-order TX:
@Service
public class OrderService_Fixed {

    @Transactional  // outer TX (may be readOnly or minimal)
    public void processOrders(List<Long> orderIds) {
        for (Long id : orderIds) {
            try {
                processOne(id);  // REQUIRES_NEW — independent TX per order
            } catch (RuntimeException e) {
                log.error("Order {} failed independently", id);
                // Outer TX NOT affected ✓
            }
        }
        // Outer TX commits cleanly
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void processOne(Long orderId) { ... }
}
```

---

### Example 5 — `readOnly=true` Performance Demonstration

```java
@Service
public class ReportService {

    // WITHOUT readOnly — Hibernate creates snapshots for dirty checking:
    @Transactional
    public List<OrderSummary> generateReport_SLOW(LocalDateTime from) {
        List<Order> orders = orderRepo.findByCreatedAtAfter(from);
        // Hibernate creates deep snapshots of ALL loaded orders
        // At TX end: compares 10,000 orders × 15 fields = 150,000 comparisons
        // Memory: 10,000 extra snapshot objects in heap

        return orders.stream().map(OrderSummary::from).collect(toList());
        // Entities not modified, snapshots wasted
    }

    // WITH readOnly — no snapshots, no dirty checking:
    @Transactional(readOnly = true)
    public List<OrderSummary> generateReport_FAST(LocalDateTime from) {
        List<Order> orders = orderRepo.findByCreatedAtAfter(from);
        // FlushMode.MANUAL: no snapshots created
        // No dirty checking at TX end
        // Significantly less memory and CPU

        return orders.stream().map(OrderSummary::from).collect(toList());
    }

    // Common pattern — class-level readOnly, override for writes:
    @Service
    @Transactional(readOnly = true)
    public static class ProductService {

        public List<Product> findAll() { ... }           // readOnly=true (inherited)
        public Optional<Product> findById(Long id) { ... } // readOnly=true (inherited)

        @Transactional  // overrides class-level: readOnly=false
        public Product save(Product p) { return productRepo.save(p); }

        @Transactional  // overrides: readOnly=false
        public void delete(Long id) { productRepo.deleteById(id); }
    }
}
```

---

### Example 6 — `@TransactionalEventListener` Pattern

```java
// Events:
public record OrderPlacedEvent(Order order) {}
public record OrderFailedEvent(Order order, String reason) {}

// Publisher (in TX):
@Service
public class OrderService {
    @Autowired ApplicationEventPublisher publisher;

    @Transactional
    public Order placeOrder(OrderRequest req) {
        Order order = orderRepo.save(new Order(req));
        // Event is QUEUED — not dispatched until TX completes:
        publisher.publishEvent(new OrderPlacedEvent(order));
        return order;
        // TX commits → event dispatched AFTER commit
    }
}

// Listeners:
@Component
public class OrderEventHandlers {

    // Runs AFTER commit — order is durably saved:
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    // ↑ Creates NEW TX for email sending (no TX active at AFTER_COMMIT point)
    public void sendConfirmationEmail(OrderPlacedEvent event) {
        emailService.send(event.order().getCustomer().getEmail(),
                         "Order #" + event.order().getId() + " confirmed");
        emailLogRepo.save(new EmailLog(event.order().getId(), "SENT"));
    }

    // Runs BEFORE commit — can still roll back outer TX:
    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void validateBeforeCommit(OrderPlacedEvent event) {
        if (!fraudDetector.isSafe(event.order())) {
            throw new FraudDetectedException(); // → rolls back ORDER TX
        }
    }

    // Runs after rollback — compensating action:
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void handleOrderFailed(OrderPlacedEvent event) {
        notificationService.notifyAdmin("Order failed: " + event.order().getId());
    }

    // Always runs — cleanup regardless of outcome:
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMPLETION)
    public void cleanup(OrderPlacedEvent event) {
        cacheService.evict("order:" + event.order().getId());
    }
}
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ: Self-Invocation**
```java
@Service
public class InventoryService {

    @Transactional
    public void adjustStock(Long productId, int delta) {
        Product p = productRepo.findById(productId).get();
        p.setStock(p.getStock() + delta);
        this.logAdjustment(productId, delta);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAdjustment(Long productId, int delta) {
        auditRepo.save(new AuditLog(productId, delta));
    }
}
```

If `adjustStock()` rolls back, what happens to the audit log?

A) Audit log is saved (REQUIRES_NEW creates independent TX)  
B) Audit log is rolled back (self-invocation bypasses proxy — runs in same TX)  
C) `IllegalTransactionStateException` thrown when `logAdjustment` tries to create new TX  
D) Audit log is saved, then rolled back because of parent TX coordination  

**Answer: B** — `this.logAdjustment()` bypasses the Spring proxy. `REQUIRES_NEW` is completely ignored. Both methods share the same transaction. When `adjustStock()` rolls back, the audit log rolls back too.

---

**Q2 — Select All That Apply: Propagation REQUIRED**

Which statements about `REQUIRED` propagation are TRUE?

A) If an active TX exists, the method joins it  
B) If no TX exists, a new TX is created  
C) If the inner method throws and marks TX rollback-only, catching the exception in the outer method prevents rollback  
D) Two methods joined in the same TX share the same EntityManager instance  
E) If the inner method completes normally, it commits its portion of work independently  
F) `UnexpectedRollbackException` can occur when the caller catches an exception from a REQUIRED inner method  

**Answer: A, B, D, F**
- C is false: catching the exception doesn't un-mark the TX as rollback-only → `UnexpectedRollbackException` at commit
- E is false: there is no partial commit in REQUIRED; the shared TX commits as one unit when the outermost method returns

---

**Q3 — Propagation Identification**
```java
@Transactional(propagation = Propagation.???)
public void criticalOperation() {
    // This method MUST run within an existing transaction.
    // If called without an active transaction,
    // an exception should be thrown immediately.
}
```

Which propagation type satisfies the requirement?

A) `REQUIRED`  
B) `REQUIRES_NEW`  
C) `MANDATORY`  
D) `SUPPORTS`  

**Answer: C** — `MANDATORY` throws `IllegalTransactionStateException` if no active transaction is present. It enforces the programming contract that this method should never be called as a transaction entry point.

---

**Q4 — `readOnly=true` Behavior**
```java
@Transactional(readOnly = true)
public void updateUserName(Long id, String newName) {
    User user = userRepo.findById(id).get();
    user.setName(newName);
    // No explicit save() call
}
```

What happens?

A) `TransactionRequiredException` — writes not allowed with `readOnly=true`  
B) The UPDATE is executed because dirty checking still runs  
C) The UPDATE is NOT executed — `readOnly=true` sets `FlushMode.MANUAL`, no flush occurs  
D) Exception at commit — Spring detects illegal write in readOnly transaction  

**Answer: C** — `readOnly=true` sets `FlushMode.MANUAL`. The entity is managed (in PC) and its `name` field changes, but Hibernate never flushes. No UPDATE SQL is generated. No exception is thrown. The change is silently lost when the TX ends. This is a common bug when developers forget `readOnly=true` on service methods that shouldn't have it.

---

**Q5 — Rollback Rule**
```java
@Transactional(rollbackFor = PaymentException.class)
public void processPayment() throws PaymentException {
    try {
        externalGateway.charge();  // throws IOException (not PaymentException)
    } catch (IOException e) {
        throw new PaymentException("gateway failure", e);  // wrap and throw
    }
}
```

What happens?

A) COMMIT — `IOException` is checked, triggers commit; wrapping doesn't change it  
B) ROLLBACK — `PaymentException` is configured for rollback and is what Spring sees  
C) ROLLBACK — `IOException` is the cause, which overrides the wrapper  
D) `UnexpectedRollbackException` because two exceptions occurred  

**Answer: B** — Spring evaluates rollback rules against the **thrown exception**, which is `PaymentException`. Since `rollbackFor = PaymentException.class` matches, the TX rolls back. The wrapped `IOException` cause is irrelevant to rollback rule evaluation.

---

**Q6 — Isolation Level**

A banking application runs two concurrent transactions:
- TX A reads account balance: 1000
- TX B reads account balance: 1000
- TX B updates balance to 1500 and commits
- TX A reads account balance again: sees 1000 (not 1500)

What isolation level does TX A use?

A) `READ_UNCOMMITTED`  
B) `READ_COMMITTED`  
C) `REPEATABLE_READ`  
D) `SERIALIZABLE`  

**Answer: C** — `REPEATABLE_READ` prevents non-repeatable reads. Once TX A has read the balance as 1000, subsequent reads within the same TX return 1000 — even after TX B commits 1500. `READ_COMMITTED` would return 1500 on the second read (allows non-repeatable reads).

---

**Q7 — `REQUIRES_NEW` Resource Implication**
```java
@Transactional  // TX A uses connection C1
public void outer() {
    // Do some work...
    inner();  // REQUIRES_NEW
    // Continue...
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void inner() {
    // TX A suspended, TX B needs connection C2
}
```

What happens if the connection pool has only **one** connection?

A) TX B reuses TX A's connection (same thread)  
B) TX B waits for a connection and eventually proceeds  
C) TX B deadlocks waiting for a connection TX A holds  
D) Spring automatically increases the pool size  

**Answer: C** — TX A holds connection C1. TX B (REQUIRES_NEW) needs a second connection C2. With pool size=1, C2 is unavailable. TX B waits for C2. C1 is never released (TX A is suspended, not committed). Deadlock: TX B waits for a connection that TX A holds and TX A waits for TX B to return. Always size the connection pool to accommodate the maximum nesting depth of `REQUIRES_NEW` transactions.

---

**Q8 — `@TransactionalEventListener` Phase**
```java
@TransactionalEventListener  // default phase
public void onOrderPlaced(OrderPlacedEvent event) {
    paymentProcessor.charge(event.order());
}
```

When is `onOrderPlaced` executed?

A) When `publisher.publishEvent()` is called (synchronously during TX)  
B) After the publishing transaction commits  
C) After the publishing transaction commits or rolls back  
D) In a new thread after the publishing transaction completes  

**Answer: B** — The default phase is `AFTER_COMMIT`. The listener is invoked synchronously (in the same thread) immediately after the transaction that published the event successfully commits. If the TX rolls back, the listener is NOT called (it's bound to AFTER_COMMIT).

---

**Q9 — NESTED vs REQUIRES_NEW**

Which statement correctly distinguishes `NESTED` from `REQUIRES_NEW`?

A) `NESTED` uses a new JDBC connection; `REQUIRES_NEW` uses the same connection  
B) `NESTED` uses savepoints in the same TX; `REQUIRES_NEW` creates an independent TX with its own commit/rollback  
C) `NESTED` and `REQUIRES_NEW` behave identically; the difference is only at the Spring API level  
D) `NESTED` rolls back the outer TX on failure; `REQUIRES_NEW` isolates failure to the inner TX  

**Answer: B** — `NESTED` creates a savepoint within the SAME physical transaction. Rolling back the nested portion rolls back to the savepoint; the outer TX continues. `REQUIRES_NEW` creates a completely independent transaction with its own connection, commit, and rollback — fully isolated from the outer TX.

---

## 4️⃣ Trick Analysis

**Self-invocation is the #1 transaction trap (Q1)**:
Every Spring transaction beginner encounters this. `this.method()` calls the real object, not the Spring proxy. The proxy is what intercepts method calls to apply transaction management. An inner method annotated with `REQUIRES_NEW` called via `this.` runs in exactly the same transaction as the outer method — the annotation is completely ineffective. The exam tests this in multiple forms: "what propagation behaviour applies when method A calls B on `this`?" — answer: always the outer TX's behaviour, inner annotation irrelevant.

**REQUIRED rollback marks entire shared TX (Q2, C and F)**:
When two methods share a TX via REQUIRED, they share one physical transaction object. If the inner method marks it `rollback-only` (by throwing a RuntimeException that propagates to Spring's interceptor), that flag persists even if the caller catches the exception. The outer method continues executing but when it tries to commit, Spring detects the rollback-only flag and throws `UnexpectedRollbackException`. Catching the exception in the outer method does NOT save the TX — only `REQUIRES_NEW` or `NESTED` can isolate failures.

**`readOnly=true` is a hint, not enforcement (Q4)**:
This is deeply misunderstood. `readOnly=true` does NOT throw if you call `save()`. It does NOT lock the DB against writes. It sets `FlushMode.MANUAL` — dirty checking is disabled and no flush will happen. A `save()` call in a `readOnly=true` transaction puts the entity in the PC as managed but it is never flushed. Changes are silently discarded. The most dangerous form: a service method marked `readOnly=true` that calls a method which modifies an entity field without calling `save()` — developer assumes dirty checking will persist it, but `readOnly=true` disabled dirty checking.

**Rollback rule evaluation uses the thrown exception class (Q5)**:
Spring evaluates rollback rules against the exception type actually thrown at the method boundary — not the cause chain. If you configure `rollbackFor = PaymentException.class` and throw a `PaymentException` (even wrapping another exception), the TX rolls back. Conversely, if only `rollbackFor = IOException.class` is set and you throw `PaymentException` wrapping `IOException`, the TX commits (because `PaymentException` is what Spring sees, not `IOException`).

**`REQUIRES_NEW` deadlock with small connection pool (Q7)**:
`REQUIRES_NEW` requires a second database connection while the outer TX holds the first. With a pool of size N, you can safely nest up to N levels of `REQUIRES_NEW`. With pool size=1, the very first `REQUIRES_NEW` call causes a deadlock. In production, this manifests as all threads eventually hanging waiting for connections. Always configure connection pool size ≥ 2 × (maximum nesting depth of REQUIRES_NEW).

**`@TransactionalEventListener` default phase is AFTER_COMMIT not synchronous (Q8)**:
Developers often assume `publishEvent()` dispatches synchronously. With `@TransactionalEventListener`, the event is queued and dispatched only after the TX commits. If the TX rolls back, the event is never dispatched (with `AFTER_COMMIT` phase). For operations that must always run (commit or rollback), use `AFTER_COMPLETION`. For operations that must run only on commit and need their own TX, combine `@TransactionalEventListener` with `@Transactional(propagation = REQUIRES_NEW)` on the handler.

---

## 5️⃣ Summary Sheet

### Propagation Complete Reference

| Propagation | Existing TX | No TX | Use Case |
|---|---|---|---|
| `REQUIRED` | Join existing | Create new | Default — most operations |
| `REQUIRES_NEW` | Suspend + new independent | Create new | Audit, notification, independent commit |
| `SUPPORTS` | Join existing | No TX | Optional TX reads |
| `NOT_SUPPORTED` | Suspend + no TX | No TX | Non-transactional operations |
| `MANDATORY` | Join existing | Exception | Enforce: must have TX |
| `NEVER` | Exception | No TX | Enforce: must NOT have TX |
| `NESTED` | Savepoint sub-TX | Create new | Per-item retry in batch |

### Isolation Level vs Phenomena

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Default DB |
|---|---|---|---|---|
| `READ_UNCOMMITTED` | Possible | Possible | Possible | — |
| `READ_COMMITTED` | Prevented | Possible | Possible | PG, Oracle, MSSQL |
| `REPEATABLE_READ` | Prevented | Prevented | Possible | MySQL InnoDB |
| `SERIALIZABLE` | Prevented | Prevented | Prevented | — |

### Rollback Rules

```
Default:
  RuntimeException / Error → ROLLBACK
  Checked Exception        → COMMIT (dangerous!)

Override with:
  rollbackFor = SomeCheckedException.class    → ROLLBACK on that exception
  rollbackFor = Exception.class               → ROLLBACK on any exception
  noRollbackFor = SomeRuntimeException.class  → COMMIT on that exception

Exception hierarchy: most specific match wins
Evaluation: against the thrown exception class, NOT the cause chain
```

### `readOnly=true` — What It Does and Does Not Do

```
DOES:
  - Sets FlushMode.MANUAL → disables dirty checking
  - Passes readOnly hint to JDBC Connection
  - Saves memory (no snapshots) and CPU (no comparison)
  - May route to read replica (pool-dependent)

DOES NOT:
  - Prevent save() calls (no exception)
  - Prevent DB writes at DB level
  - Enforce read-only at Java layer
  - throw if entity is modified (changes silently discarded)
```

### Key Rules

```
1.  @Transactional works via AOP proxy — self-invocation (this.method()) bypasses it
2.  REQUIRED: joining inner method marks shared TX rollback-only on exception
    → catching exception in outer does NOT prevent rollback → UnexpectedRollbackException
3.  REQUIRES_NEW: suspends outer TX, needs SECOND DB connection → deadlock if pool size = 1
4.  NESTED: savepoint within same TX → partial rollback possible without new connection
5.  MANDATORY: exception if no active TX → use to enforce programming contracts
6.  readOnly=true: FlushMode.MANUAL → no dirty checking, no flush, changes silently lost
7.  readOnly=true on class level inherited by all methods; @Transactional on method overrides completely
8.  Checked exceptions → COMMIT by default (not rollback!) — use rollbackFor to override
9.  Rollback rule matched against thrown exception class, not cause chain
10. @Transactional(propagation) on inner method is ignored in self-invocation
11. All REQUIRED-joined methods share ONE EntityManager instance in the thread
12. @TransactionalEventListener default = AFTER_COMMIT — not dispatched on rollback
13. AFTER_COMMIT handler has NO active TX — add @Transactional(REQUIRES_NEW) if needed
14. timeout is in SECONDS; timeout = -1 means no timeout
15. Method-level @Transactional completely overrides class-level (not merged)
```

### Self-Invocation Fix Summary

```
Problem: this.method() bypasses proxy → @Transactional ignored

Fix 1: @Lazy @Autowired private MyService self;
       → Spring injects the proxy reference, call via self.method()

Fix 2: Extract to separate @Service bean
       → Different bean = different proxy = @Transactional honoured

Fix 3: ApplicationContext.getBean(MyService.class)
       → Gets proxy from context (ugly, not recommended)

Fix 4: AspectJ load-time weaving (LTW)
       → Eliminates proxy entirely; weaves @Transactional into bytecode
       → Works for self-invocation but complex setup
```

---
