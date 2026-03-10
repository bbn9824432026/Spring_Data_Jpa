# 🏛️ TOPIC 20 — Spring Data REST and Repository Exposure

---

## 1️⃣ Conceptual Explanation

### What Spring Data REST Does — The Core Idea

Spring Data REST automatically exposes Spring Data repositories as RESTful HTTP endpoints. A single `@RepositoryRestResource` annotation (or just having a `JpaRepository`) can generate a fully functional CRUD REST API with pagination, sorting, search, and HATEOAS links — with zero controller code.

```
Without Spring Data REST:
  Repository → Service → Controller → HTTP
  (you write all three layers)

With Spring Data REST:
  Repository → HTTP  (framework generates the middle layers)
  
What gets generated automatically:
  GET    /products          → findAll() with pagination
  GET    /products/{id}     → findById()
  POST   /products          → save() (new entity)
  PUT    /products/{id}     → save() (full replace)
  PATCH  /products/{id}     → save() (partial update)
  DELETE /products/{id}     → deleteById()
  GET    /products/search   → lists all exported query methods
  GET    /products/search/findByCategory?category=X → query method
```

---

### HAL — Hypertext Application Language

Spring Data REST responses use **HAL** (Hypertext Application Language) — a JSON format that embeds `_links` objects alongside data:

```json
{
  "id": 1,
  "name": "Widget",
  "price": 9.99,
  "_links": {
    "self": {
      "href": "http://localhost:8080/products/1"
    },
    "product": {
      "href": "http://localhost:8080/products/1"
    },
    "category": {
      "href": "http://localhost:8080/products/1/category"
    }
  }
}
```

**Collection response (HAL):**

```json
{
  "_embedded": {
    "products": [
      { "id": 1, "name": "Widget", "_links": { "self": {...} } },
      { "id": 2, "name": "Gadget", "_links": { "self": {...} } }
    ]
  },
  "_links": {
    "self":  { "href": "http://localhost:8080/products{?page,size,sort}" },
    "first": { "href": "http://localhost:8080/products?page=0&size=20" },
    "next":  { "href": "http://localhost:8080/products?page=1&size=20" },
    "last":  { "href": "http://localhost:8080/products?page=4&size=20" }
  },
  "page": {
    "size": 20,
    "totalElements": 95,
    "totalPages": 5,
    "number": 0
  }
}
```

---

### Setup — Dependencies and Auto-Configuration

```xml
<!-- Maven dependency: -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-rest</artifactId>
</dependency>
```

```yaml
# application.yml configuration:
spring:
  data:
    rest:
      base-path: /api          # prefix all REST endpoints with /api
      default-page-size: 20    # default page size for collections
      max-page-size: 100       # maximum allowed page size
      default-sort: name,asc   # default sort order
      return-body-on-create: true   # return entity in POST response body
      return-body-on-update: true   # return entity in PUT/PATCH response body
      detection-strategy: annotated # only expose @RepositoryRestResource repos
      # default-media-type: application/hal+json
```

**Detection strategies:**

```
ALL:         every repository is exported (default in older versions)
DEFAULT:     repositories not annotated with @NoRepositoryBean are exported
ANNOTATED:   only repositories with @RepositoryRestResource are exported
VISIBILITY:  only public repositories are exported
```

---

### `@RepositoryRestResource` — Controlling Exposure

```java
// Basic exposure with custom path and rel name:
@RepositoryRestResource(
    path = "products",          // URL path: /api/products
    collectionResourceRel = "products",  // _embedded key name
    itemResourceRel = "product"         // individual item rel name
)
public interface ProductRepository extends JpaRepository<Product, Long> {

    // Exported search method — accessible at /api/products/search/findByCategory:
    List<Product> findByCategory(@Param("category") String category);

    // Exported with custom path:
    @RestResource(path = "byPrice", rel = "byPrice")
    List<Product> findByPriceLessThan(@Param("price") BigDecimal price);
    // Accessible at: /api/products/search/byPrice?price=50

    // HIDDEN from REST API — not accessible via HTTP:
    @RestResource(exported = false)
    List<Product> findByInternalCode(String code);

    // All custom methods are exported by default
    // Use exported=false to hide sensitive or internal queries
}

// Completely suppress REST exposure of a repository:
@RepositoryRestResource(exported = false)
public interface InternalRepository extends JpaRepository<InternalEntity, Long> {
    // Entire repository hidden from REST
}
```

---

### `@RestResource` — Method-Level Exposure Control

```java
@RepositoryRestResource(path = "users")
public interface UserRepository extends JpaRepository<User, Long> {

    // Exposed: GET /api/users/search/findByEmail?email=alice@example.com
    @RestResource(path = "findByEmail", rel = "findByEmail")
    Optional<User> findByEmail(@Param("email") String email);

    // Exposed: GET /api/users/search/active
    @RestResource(path = "active")
    List<User> findByActiveTrue();

    // HIDDEN:
    @RestResource(exported = false)
    @Modifying @Transactional
    @Query("DELETE FROM User u WHERE u.lastLoginAt < :cutoff")
    void deleteInactiveUsers(@Param("cutoff") LocalDateTime cutoff);

    // DELETE endpoint can be suppressed at repository level:
    // Override deleteById and mark as not exported
    @Override
    @RestResource(exported = false)
    void deleteById(Long id);
    // Now: DELETE /api/users/{id} returns 405 Method Not Allowed
}
```

---

### Internal Architecture — Request Processing Pipeline

```
HTTP Request: GET /api/products/1
      │
      ▼
DispatcherServlet
      │
      ▼
RepositoryRestHandlerMapping
  → Matches /api/products/1 to ProductRepository
      │
      ▼
RepositoryEntityController
  → Calls productRepository.findById(1L)
      │
      ▼
PersistentEntityResourceAssembler
  → Wraps Product in PersistentEntityResource
  → Adds _links (self, associations)
      │
      ▼
HAL Jackson Serializer
  → Serializes to HAL JSON
      │
      ▼
HTTP Response: 200 OK with HAL body
```

**Association handling — Spring Data REST navigates relationships:**

```
Product has @ManyToOne Category:

GET /api/products/1           → product data + _links.category
GET /api/products/1/category  → the associated Category resource
PUT /api/products/1/category  → update association (body: URI of new category)
DELETE /api/products/1/category → remove association

Spring Data REST automatically generates association endpoints
for all managed associations (@ManyToOne, @OneToOne, @OneToMany, @ManyToMany)
```

---

### Repository Event Handlers — `@RepositoryEventHandler`

Spring Data REST fires events at key points in the entity lifecycle. You can intercept these with `@RepositoryEventHandler`:

```java
@Component
@RepositoryEventHandler
public class ProductEventHandler {

    @Autowired AuditService auditService;
    @Autowired ValidationService validationService;

    // Fires BEFORE POST (create):
    @HandleBeforeCreate
    public void handleBeforeCreate(Product product) {
        // Validate, set defaults, enrich:
        if (product.getCreatedAt() == null) {
            product.setCreatedAt(LocalDateTime.now());
        }
        validationService.validate(product);
        // Can throw exception to prevent creation
    }

    // Fires AFTER POST (create):
    @HandleAfterCreate
    public void handleAfterCreate(Product product) {
        auditService.logCreation(product);
        // product.getId() is now populated
        searchIndexService.index(product);
    }

    // Fires BEFORE PUT/PATCH (save/update):
    @HandleBeforeSave
    public void handleBeforeSave(Product product) {
        product.setLastModifiedAt(LocalDateTime.now());
    }

    // Fires AFTER PUT/PATCH:
    @HandleAfterSave
    public void handleAfterSave(Product product) {
        cacheService.evict("product:" + product.getId());
    }

    // Fires BEFORE DELETE:
    @HandleBeforeDelete
    public void handleBeforeDelete(Product product) {
        if (product.hasActiveOrders()) {
            throw new RepositoryConstraintViolationException(
                new Errors() {{ ... }});
        }
    }

    // Fires AFTER DELETE:
    @HandleAfterDelete
    public void handleAfterDelete(Product product) {
        searchIndexService.deindex(product);
    }

    // Fires BEFORE linking association:
    @HandleBeforeLinkSave
    public void handleBeforeLinkSave(Product product, Category category) {
        // Fires when PUT /api/products/1/category is called
    }

    // Fires AFTER linking association:
    @HandleAfterLinkSave
    public void handleAfterLinkSave(Product product, Category category) { }

    // Fires BEFORE deleting association link:
    @HandleBeforeLinkDelete
    public void handleBeforeLinkDelete(Product product, Object linkedEntity) { }

    @HandleAfterLinkDelete
    public void handleAfterLinkDelete(Product product, Object linkedEntity) { }
}
```

**Event handler type specificity:**

```java
// Handler for specific entity type:
@RepositoryEventHandler(Product.class)  // ONLY fires for Product events
public class ProductSpecificHandler {
    @HandleBeforeCreate
    public void beforeCreate(Product product) { ... }
}

// Generic handler (fires for ALL entity types):
@RepositoryEventHandler  // no class = fires for all
public class GenericAuditHandler {
    @HandleBeforeCreate
    public void beforeCreate(Object entity) {
        if (entity instanceof Auditable a) {
            a.setCreatedAt(LocalDateTime.now());
        }
    }
}
```

---

### Projections in Spring Data REST — `@Projection`

`@Projection` defines a view of an entity exposed via REST, similar to JPA projections but designed for REST consumption:

```java
// Define projection:
@Projection(
    name = "summary",                   // name used in URL
    types = {Product.class}             // entity type this applies to
)
public interface ProductSummary {
    Long getId();
    String getName();
    BigDecimal getPrice();
    // No description, no internal fields
}

// More detailed projection:
@Projection(name = "detail", types = {Product.class})
public interface ProductDetail {
    Long getId();
    String getName();
    BigDecimal getPrice();
    String getDescription();
    CategoryInfo getCategory();  // nested projection

    interface CategoryInfo {
        Long getId();
        String getName();
    }
}

// Usage in REST calls:
// GET /api/products?projection=summary
// → Returns all products using ProductSummary projection

// GET /api/products/1?projection=detail
// → Returns product 1 using ProductDetail projection

// GET /api/products/search/findByCategory?category=Electronics&projection=summary
// → Returns search results using summary projection
```

**Where to place `@Projection` interfaces:**

```
IMPORTANT: @Projection interfaces MUST be in the same package as the entity
OR registered manually with RepositoryRestConfigurer:

@Configuration
public class RestConfig implements RepositoryRestConfigurer {
    @Override
    public void configureRepositoryRestConfiguration(
            RepositoryRestConfiguration config,
            CorsRegistry cors) {
        config.getProjectionConfiguration()
              .addProjection(ProductSummary.class);
    }
}
```

---

### `RepositoryRestConfigurer` — Full Configuration

```java
@Configuration
public class RestConfig implements RepositoryRestConfigurer {

    @Override
    public void configureRepositoryRestConfiguration(
            RepositoryRestConfiguration config,
            CorsRegistry cors) {

        // Expose entity IDs in response (hidden by default):
        config.exposeIdsFor(Product.class, User.class, Order.class);
        // Without this: {"name":"Widget","price":9.99} — no id field!
        // With this:    {"id":1,"name":"Widget","price":9.99}

        // Base path:
        config.setBasePath("/api");

        // Pagination:
        config.setDefaultPageSize(20);
        config.setMaxPageSize(100);

        // Return body on create/update:
        config.setReturnBodyOnCreate(true);
        config.setReturnBodyOnUpdate(true);

        // CORS:
        cors.addMapping("/api/**")
            .allowedOrigins("https://myapp.com")
            .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE")
            .allowedHeaders("*")
            .maxAge(3600);

        // Disable PATCH support globally:
        // (use if you want only full PUT updates)
        config.disableDefaultExposureFor(Product.class);
        // Then re-enable only what you want via @RepositoryRestResource

        // Register projections not in entity package:
        config.getProjectionConfiguration()
              .addProjection(ProductSummary.class)
              .addProjection(ProductDetail.class);
    }

    @Override
    public void configureJacksonObjectMapper(ObjectMapper objectMapper) {
        // Customize JSON serialization:
        objectMapper.configure(
            DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        objectMapper.registerModule(new JavaTimeModule());
        objectMapper.disable(
            SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }

    @Override
    public void configureConversionService(
            ConfigurableConversionService conversionService) {
        // Add custom converters for request parameter binding:
        conversionService.addConverter(String.class, Status.class,
            Status::fromCode);
    }
}
```

---

### Entity ID Exposure — The Hidden ID Problem

By default, Spring Data REST does NOT include entity IDs in JSON responses. The `id` field is omitted — only `_links.self` contains the ID (embedded in the URL).

```json
// Default (id NOT in body):
{
  "name": "Widget",
  "price": 9.99,
  "_links": {
    "self": { "href": "/api/products/1" }
  }
}

// After config.exposeIdsFor(Product.class):
{
  "id": 1,
  "name": "Widget",
  "price": 9.99,
  "_links": {
    "self": { "href": "/api/products/1" }
  }
}
```

**Why IDs are hidden by default:**

```
HATEOAS principle: clients should navigate via links, not construct URLs from IDs.
The id=1 value in the body is redundant — it's already in the self link.
For pure HATEOAS clients, exposing IDs is unnecessary.
For practical frontend clients, exposing IDs is usually convenient.
```

---

### Pagination and Sorting in Spring Data REST

```
Collection endpoint: GET /api/products

Pagination parameters:
  page=0          → zero-based page number (default: 0)
  size=20         → page size (default: spring.data.rest.default-page-size)
  
Sort parameters:
  sort=name,asc           → sort by name ascending
  sort=price,desc         → sort by price descending
  sort=name,asc&sort=price,desc  → multiple sorts

Examples:
  GET /api/products?page=2&size=10
  GET /api/products?sort=name,asc&sort=price,desc
  GET /api/products?page=1&size=25&sort=createdAt,desc

Response includes pagination metadata:
  "page": {
    "size": 10,
    "totalElements": 95,
    "totalPages": 10,
    "number": 2
  }
  "_links": {
    "first": ..., "prev": ..., "self": ..., "next": ..., "last": ...
  }
```

---

### Search Endpoints — Exposing Query Methods

```java
@RepositoryRestResource(path = "products")
public interface ProductRepository extends JpaRepository<Product, Long> {

    // GET /api/products/search
    // Lists all exported search methods as _links

    // GET /api/products/search/findByCategory?category=Electronics
    List<Product> findByCategory(@Param("category") String category);

    // GET /api/products/search/findByPriceRange?min=10&max=100
    @RestResource(path = "findByPriceRange")
    List<Product> findByPriceBetween(
        @Param("min") BigDecimal min,
        @Param("max") BigDecimal max);

    // GET /api/products/search/active?page=0&size=10
    @RestResource(path = "active")
    Page<Product> findByActiveTrue(Pageable pageable);
    // Returns paginated results with HAL pagination links

    // @Param is REQUIRED for Spring Data REST parameter binding
    // Without @Param: request parameters cannot be bound to method parameters
}

// GET /api/products/search response:
// {
//   "_links": {
//     "findByCategory": {
//       "href": "http://localhost:8080/api/products/search/findByCategory{?category}"
//     },
//     "findByPriceRange": {
//       "href": "http://localhost:8080/api/products/search/findByPriceRange{?min,max}"
//     },
//     "active": {
//       "href": "http://localhost:8080/api/products/search/active{?page,size,sort}"
//     }
//   }
// }
```

---

### Validators — Request Body Validation

```java
// Standard Bean Validation (applied automatically):
@Entity
public class Product {
    @Id @GeneratedValue Long id;

    @NotBlank @Size(min=2, max=100)
    String name;

    @NotNull @DecimalMin("0.01")
    BigDecimal price;

    @Min(0)
    Integer stock;
}
// POST /api/products with invalid data → 400 Bad Request with validation errors

// Custom validator for Spring Data REST:
@Component("beforeCreateProductValidator")  // naming convention: before{Create/Save}{EntityName}Validator
public class ProductCreateValidator implements Validator {

    @Override
    public boolean supports(Class<?> clazz) {
        return Product.class.isAssignableFrom(clazz);
    }

    @Override
    public void validate(Object target, Errors errors) {
        Product product = (Product) target;
        if (product.getName() != null
                && product.getName().contains("BANNED")) {
            errors.rejectValue("name", "name.banned",
                "Product name contains banned content");
        }
    }
}

// Register validator explicitly (if naming convention doesn't auto-register):
@Configuration
public class ValidatorConfig implements RepositoryRestConfigurer {

    @Autowired ProductCreateValidator productValidator;

    @Override
    public void configureValidatingRepositoryEventListener(
            ValidatingRepositoryEventListener listener) {
        listener.addValidator("beforeCreate", productValidator);
        listener.addValidator("beforeSave", productValidator);
    }
}
```

---

### Security — Protecting Repository Endpoints

```java
@Configuration
@EnableMethodSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
            .requestMatchers(HttpMethod.GET, "/api/**").permitAll()
            .requestMatchers(HttpMethod.POST, "/api/**").hasRole("ADMIN")
            .requestMatchers(HttpMethod.PUT, "/api/**").hasRole("ADMIN")
            .requestMatchers(HttpMethod.PATCH, "/api/**").hasRole("ADMIN")
            .requestMatchers(HttpMethod.DELETE, "/api/**").hasRole("ADMIN")
            .anyRequest().authenticated()
        );
        return http.build();
    }
}

// Method-level security on repository:
@RepositoryRestResource(path = "users")
public interface UserRepository extends JpaRepository<User, Long> {

    @PreAuthorize("hasRole('ADMIN')")
    @Override
    <S extends User> S save(S entity);

    @PreAuthorize("hasRole('ADMIN')")
    @Override
    void deleteById(Long id);

    @PreAuthorize("isAuthenticated()")
    Optional<User> findByEmail(@Param("email") String email);

    // Public search — no security:
    List<User> findByActiveTrue();
}
```

---

### ALPS — Application-Level Profile Semantics

Spring Data REST automatically generates an ALPS metadata document describing the API:

```
GET /api/profile
→ Lists all resource types

GET /api/profile/products
→ Describes Product resource:
  - Properties and their types
  - Available operations (GET, POST, PUT, PATCH, DELETE)
  - Search operations and their parameters
  - Projections available
```

```json
{
  "alps": {
    "version": "1.0",
    "descriptor": [
      {
        "id": "product-representation",
        "href": "http://localhost:8080/api/profile/products",
        "descriptor": [
          { "name": "name",  "type": "SEMANTIC" },
          { "name": "price", "type": "SEMANTIC" },
          { "name": "category", "type": "SAFE",
            "rt": "#category-representation" }
        ]
      }
    ]
  }
}
```

---

### Excerpts — Default Embedded Projections

An excerpt projection is automatically applied when listing collections (without `?projection=` parameter):

```java
@Projection(name = "summary", types = {Product.class})
public interface ProductSummary {
    Long getId();
    String getName();
    BigDecimal getPrice();
}

@RepositoryRestResource(
    path = "products",
    excerptProjection = ProductSummary.class  // applied to collections by default
)
public interface ProductRepository extends JpaRepository<Product, Long> { }

// GET /api/products
// → Uses ProductSummary projection for each item in _embedded
// → Only id, name, price returned (not full entity)
// GET /api/products/1
// → Returns full entity (excerpt only applies to collection lists)
// GET /api/products?projection=detail
// → Uses specified projection instead of excerpt
```

---

## 2️⃣ Code Examples

### Example 1 — Complete Spring Data REST Setup

```java
// ── 1. Entity ─────────────────────────────────────────────────────────────
@Entity
@Table(name = "books")
public class Book {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    @NotBlank @Size(max = 200)
    String title;

    @NotBlank @Size(max = 13)
    @Column(unique = true)
    String isbn;

    @DecimalMin("0.00")
    BigDecimal price;

    @NotNull
    LocalDate publishedDate;

    boolean available;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    Author author;

    @ManyToMany
    @JoinTable(name = "book_genres",
        joinColumns = @JoinColumn(name = "book_id"),
        inverseJoinColumns = @JoinColumn(name = "genre_id"))
    Set<Genre> genres;
}

// ── 2. Projections ────────────────────────────────────────────────────────
@Projection(name = "summary", types = {Book.class})
public interface BookSummary {
    Long getId();
    String getTitle();
    String getIsbn();
    BigDecimal getPrice();
}

@Projection(name = "detail", types = {Book.class})
public interface BookDetail {
    Long getId();
    String getTitle();
    String getIsbn();
    BigDecimal getPrice();
    LocalDate getPublishedDate();
    boolean isAvailable();
    AuthorInfo getAuthor();

    interface AuthorInfo {
        Long getId();
        String getName();
        String getBiography();
    }
}

// ── 3. Repository ─────────────────────────────────────────────────────────
@RepositoryRestResource(
    path = "books",
    collectionResourceRel = "books",
    itemResourceRel = "book",
    excerptProjection = BookSummary.class   // default for collection listings
)
public interface BookRepository extends JpaRepository<Book, Long> {

    // GET /api/books/search/findByIsbn?isbn=978-3-16-148410-0
    @RestResource(path = "findByIsbn")
    Optional<Book> findByIsbn(@Param("isbn") String isbn);

    // GET /api/books/search/findByAuthor?authorId=42
    @RestResource(path = "findByAuthor")
    List<Book> findByAuthorId(@Param("authorId") Long authorId);

    // GET /api/books/search/available?page=0&size=10
    @RestResource(path = "available")
    Page<Book> findByAvailableTrue(Pageable pageable);

    // GET /api/books/search/findByTitle?title=Spring
    @RestResource(path = "findByTitle")
    List<Book> findByTitleContainingIgnoreCase(@Param("title") String title);

    // HIDDEN — internal method:
    @RestResource(exported = false)
    List<Book> findByPriceLessThan(BigDecimal price);
}

// ── 4. Event handler ──────────────────────────────────────────────────────
@Component
@RepositoryEventHandler(Book.class)
public class BookEventHandler {

    @HandleBeforeCreate
    public void handleBeforeCreate(Book book) {
        book.setAvailable(true);
        // Validate ISBN format:
        if (!isValidIsbn(book.getIsbn())) {
            throw new IllegalArgumentException(
                "Invalid ISBN format: " + book.getIsbn());
        }
    }

    @HandleAfterCreate
    public void handleAfterCreate(Book book) {
        log.info("Book created: {} (id={})", book.getTitle(), book.getId());
        notificationService.notifyNewBook(book);
    }

    @HandleBeforeSave
    public void handleBeforeSave(Book book) {
        book.setLastModified(LocalDateTime.now());
    }

    @HandleBeforeDelete
    public void handleBeforeDelete(Book book) {
        if (orderService.hasActiveOrders(book.getId())) {
            throw new RepositoryConstraintViolationException(
                Errors.of("book", "has.active.orders",
                    "Cannot delete book with active orders"));
        }
    }

    private boolean isValidIsbn(String isbn) {
        return isbn != null && isbn.matches(
            "\\d{3}-\\d-\\d{2}-\\d{6}-\\d");
    }
}

// ── 5. Configuration ──────────────────────────────────────────────────────
@Configuration
public class DataRestConfig implements RepositoryRestConfigurer {

    @Override
    public void configureRepositoryRestConfiguration(
            RepositoryRestConfiguration config,
            CorsRegistry cors) {
        config.setBasePath("/api");
        config.setDefaultPageSize(20);
        config.setMaxPageSize(100);
        config.exposeIdsFor(Book.class, Author.class, Genre.class);
        config.setReturnBodyOnCreate(true);
        config.setReturnBodyOnUpdate(true);

        cors.addMapping("/api/**")
            .allowedOrigins("*")
            .allowedMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS");
    }
}
```

---

### Example 2 — Association Management via REST

```java
// Entities:
@Entity
public class Author {
    @Id @GeneratedValue Long id;
    String name;
    String biography;

    @OneToMany(mappedBy = "author")
    List<Book> books;
}

// Repository:
@RepositoryRestResource(path = "authors")
public interface AuthorRepository extends JpaRepository<Author, Long> {
    List<Author> findByNameContaining(@Param("name") String name);
}

// Association REST interactions:

// 1. GET /api/books/1/author
//    → Returns the Author associated with Book 1
//    → HTTP 200 with Author HAL representation

// 2. PUT /api/books/1/author
//    Content-Type: text/uri-list
//    Body: http://localhost:8080/api/authors/42
//    → Associates Book 1 with Author 42
//    → HTTP 204 No Content

// 3. DELETE /api/books/1/author
//    → Removes the association (sets author=null on book)
//    → HTTP 204 No Content

// 4. GET /api/books/1/genres
//    → Returns all genres for Book 1 (collection)

// 5. POST /api/books/1/genres
//    Content-Type: text/uri-list
//    Body: http://localhost:8080/api/genres/5
//          http://localhost:8080/api/genres/7
//    → Adds genres 5 and 7 to book 1's genres set

// Testing with curl:
/*
curl -X GET  http://localhost:8080/api/books/1/author
curl -X PUT  http://localhost:8080/api/books/1/author \
     -H "Content-Type: text/uri-list" \
     -d "http://localhost:8080/api/authors/42"
curl -X DELETE http://localhost:8080/api/books/1/author
*/
```

---

### Example 3 — Security and Validation

```java
// Secured repository:
@RepositoryRestResource(path = "orders")
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Anyone can read:
    @RestResource(path = "findByStatus")
    List<Order> findByStatus(@Param("status") String status);

    // Only authenticated users can see their own orders:
    @PreAuthorize("isAuthenticated()")
    @RestResource(path = "myOrders")
    List<Order> findByCustomerId(@Param("customerId") Long customerId);

    // Admin only — full access:
    @PreAuthorize("hasRole('ADMIN')")
    @Override
    <S extends Order> S save(S entity);

    @PreAuthorize("hasRole('ADMIN')")
    @Override
    void deleteById(Long id);
}

// Bean Validation on entity:
@Entity
public class Order {
    @Id @GeneratedValue Long id;

    @NotBlank
    String customerId;

    @NotNull @DecimalMin("0.01")
    BigDecimal total;

    @NotNull
    @Enumerated(EnumType.STRING)
    OrderStatus status;

    @NotNull @Future  // delivery date must be in the future
    LocalDate deliveryDate;
}

// Custom event handler with validation:
@Component
@RepositoryEventHandler(Order.class)
public class OrderEventHandler {

    @HandleBeforeCreate
    public void validateBeforeCreate(Order order) {
        if (order.getTotal().compareTo(new BigDecimal("10000")) > 0) {
            // Large orders need manual approval:
            order.setStatus(OrderStatus.PENDING_APPROVAL);
        } else {
            order.setStatus(OrderStatus.CONFIRMED);
        }
    }

    @HandleBeforeSave
    public void validateBeforeSave(Order order) {
        if (order.getStatus() == OrderStatus.SHIPPED
                && order.getTrackingNumber() == null) {
            throw new IllegalStateException(
                "Shipped orders must have a tracking number");
        }
    }
}
```

---

### Example 4 — Projection with Computed Fields

```java
// Entity:
@Entity
public class Product {
    @Id @GeneratedValue Long id;
    String name;
    BigDecimal price;
    Integer stock;
    String category;
    LocalDateTime createdAt;
}

// Projection with @Value SpEL:
@Projection(name = "withDiscount", types = {Product.class})
public interface ProductWithDiscount {
    Long getId();
    String getName();
    BigDecimal getPrice();

    @Value("#{target.price * 0.9}")
    BigDecimal getDiscountedPrice();

    @Value("#{target.stock > 0 ? 'Available' : 'Out of Stock'}")
    String getAvailability();

    @Value("#{target.price * 0.9 < 10.0 ? 'BARGAIN' : 'STANDARD'}")
    String getPriceCategory();
}

// Flat projection combining associated data:
@Projection(name = "withCategoryDetails", types = {Product.class})
public interface ProductWithCategoryDetails {
    Long getId();
    String getName();
    BigDecimal getPrice();

    // Navigate @ManyToOne association:
    @Value("#{target.category != null ? target.category.name : 'Uncategorized'}")
    String getCategoryName();

    @Value("#{target.category != null ? target.category.description : ''}")
    String getCategoryDescription();
}

// Usage:
// GET /api/products?projection=withDiscount
// GET /api/products/1?projection=withCategoryDetails
// GET /api/products/search/findByCategory?category=Electronics&projection=withDiscount
```

---

## 3️⃣ Exam-Style Questions

**Q1 — MCQ: Default ID Exposure**
```java
@RepositoryRestResource(path = "products")
public interface ProductRepository extends JpaRepository<Product, Long> { }
```

`GET /api/products/1` returns:

A) `{"id":1, "name":"Widget", "price":9.99, "_links":{...}}`  
B) `{"name":"Widget", "price":9.99, "_links":{"self":{"href":"/api/products/1"}}}`  
C) `{"id":1, "name":"Widget", "price":9.99}` (no `_links`)  
D) `404 Not Found` — ID not in body means it cannot be retrieved  

**Answer: B** — By default, Spring Data REST does NOT include the entity's `id` field in the JSON body. The ID is available only via the `self` link URL. To include `id` in the body, call `config.exposeIdsFor(Product.class)` in `RepositoryRestConfigurer`.

---

**Q2 — Select All That Apply: `@Filter` in Spring Data REST**

A repository method is annotated with `@RestResource(exported = false)`. Which statements are TRUE?

A) The method is NOT accessible via HTTP  
B) The method can still be called programmatically in Java code  
C) The method is removed from the repository interface at compile time  
D) `GET /api/products/search` does NOT list this method in `_links`  
E) Spring Data REST throws an exception if the method is called via HTTP  

**Answer: A, B, D**
- C is false: `exported=false` is a runtime HTTP exposure flag, not a compile-time change. The method exists and works normally in Java code.
- E is false: accessing a non-exported endpoint returns `404 Not Found` or `405 Method Not Allowed`, not an exception.

---

**Q3 — HAL Response Structure**

`GET /api/books` returns a collection. Where are the actual book objects in the HAL response?

A) Directly in the root JSON object as a `content` array  
B) Inside `_embedded.books` (using the `collectionResourceRel` name)  
C) Inside `_links.books`  
D) In a `data` array at the root level  

**Answer: B** — HAL collections are placed inside `_embedded`, keyed by the `collectionResourceRel` value. If `@RepositoryRestResource(collectionResourceRel = "books")`, the response is `{"_embedded": {"books": [...]}, "_links": {...}, "page": {...}}`.

---

**Q4 — Event Handler Timing**
```java
@HandleBeforeCreate
public void beforeCreate(Product product) {
    product.setCreatedAt(LocalDateTime.now());
    product.setSku(generateSku());
}

@HandleAfterCreate
public void afterCreate(Product product) {
    System.out.println("ID: " + product.getId());
}
```

What is `product.getId()` in `afterCreate`?

A) `null` — ID is not assigned until flush  
B) The generated ID value — entity has been persisted by the time `@HandleAfterCreate` fires  
C) `0` — default value for Long  
D) Throws `NullPointerException`  

**Answer: B** — `@HandleAfterCreate` fires AFTER the entity has been saved (persisted and flushed). The generated ID is available. `@HandleBeforeCreate` fires before save — ID is null at that point.

---

**Q5 — `@Param` Requirement**
```java
@RepositoryRestResource(path = "users")
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByDepartment(String department);  // No @Param
}

// HTTP call: GET /api/users/search/findByDepartment?department=Engineering
```

What happens?

A) Works correctly — Spring Data REST binds `department` request parameter automatically  
B) `400 Bad Request` — `@Param("department")` is required for parameter binding  
C) Returns all users — parameter is ignored without `@Param`  
D) `500 Internal Server Error`  

**Answer: B** — `@Param` is required for Spring Data REST to bind HTTP query parameters to method parameters. Without `@Param("department")`, Spring Data REST cannot map the `?department=Engineering` query parameter to the `String department` method argument. Result: `400 Bad Request` or the parameter is ignored depending on the version.

---

**Q6 — `excerptProjection` Scope**
```java
@RepositoryRestResource(
    path = "products",
    excerptProjection = ProductSummary.class
)
public interface ProductRepository extends JpaRepository<Product, Long> { }
```

`GET /api/products/1` (single item) — is `ProductSummary` applied?

A) Yes — `excerptProjection` applies to both collection and individual item endpoints  
B) No — `excerptProjection` applies ONLY to collection listings, not single item lookups  
C) Yes — unless `?projection=` is explicitly specified  
D) No — `excerptProjection` is ignored; full entity is always returned  

**Answer: B** — `excerptProjection` is applied only when the entity appears as part of a collection (embedded in `_embedded`). Single item endpoints (`GET /api/products/1`) return the FULL entity representation by default. To get a projection on a single item, explicitly add `?projection=summary` to the request.

---

**Q7 — Association Update via REST**

How do you update a `@ManyToOne` association via Spring Data REST?

A) `PUT /api/books/1/author` with JSON body `{"id": 42}`  
B) `PUT /api/books/1/author` with `Content-Type: text/uri-list` and body `http://localhost:8080/api/authors/42`  
C) `PATCH /api/books/1` with JSON body `{"author": {"id": 42}}`  
D) `POST /api/books/1/author` with `Content-Type: application/json`  

**Answer: B** — Spring Data REST manages associations via `text/uri-list` content type. The body is one or more URIs pointing to the associated resources (one per line for collections). The `PUT` method replaces the association entirely. `PATCH` with JSON body `{"author": {"id": 42}}` may work for some configurations but `text/uri-list` is the canonical Spring Data REST approach.

---

**Q8 — `@RepositoryEventHandler` Specificity**
```java
@Component
@RepositoryEventHandler
public class GenericHandler {
    @HandleBeforeCreate
    public void before(Object entity) {
        System.out.println("Generic: " + entity.getClass().getSimpleName());
    }
}

@Component
@RepositoryEventHandler(Product.class)
public class ProductHandler {
    @HandleBeforeCreate
    public void before(Product product) {
        System.out.println("Product-specific");
    }
}
```

A `Product` is created via `POST /api/products`. What is printed?

A) Only `"Product-specific"`  
B) Only `"Generic: Product"`  
C) Both `"Generic: Product"` and `"Product-specific"` (both handlers fire)  
D) Neither — only one handler can be registered per entity type  

**Answer: C** — Both handlers fire. The generic handler (`@RepositoryEventHandler` with no class) fires for ALL entity types. The specific handler fires only for `Product`. When a `Product` is created, Spring fires both: the specific handler AND the generic handler. Order is not guaranteed unless you implement `Ordered`.

---

## 4️⃣ Trick Analysis

**ID not in response body by default (Q1)**:
This surprises nearly every developer new to Spring Data REST. The HATEOAS philosophy says clients should use `_links.self` to identify resources, not parse IDs from the body. In practice, most frontends expect `id` in the JSON. Always call `config.exposeIdsFor(...)` in `RepositoryRestConfigurer` for every entity type that frontend clients need to reference by ID. Missing this causes null/undefined ID values in frontend code with no obvious error message.

**`exported=false` is HTTP-only — Java still works (Q2)**:
`@RestResource(exported=false)` controls HTTP exposure only. The repository method still exists, compiles, and executes normally when called from Java code. This is important for internal service methods that should not be accessible via the REST API. A common pattern: hide dangerous bulk-delete methods from HTTP while keeping them available for scheduled jobs.

**`excerptProjection` is collection-only (Q6)**:
`excerptProjection` is designed for collection listings where you want a compact representation. When you GET a single resource directly, Spring Data REST returns the full entity regardless of `excerptProjection`. This means your collection and single-item representations differ by default. To get a compact single-item response, add `?projection=summary` explicitly in the request.

**`@Param` is mandatory for REST search methods (Q5)**:
In regular Spring Data JPA, `@Param` is optional with `-parameters` compiler flag. In Spring Data REST, `@Param` is mandatory for HTTP parameter binding. Spring Data REST resolves query method parameters by the `@Param` name to match HTTP query parameters. Without it, the parameter cannot be matched and the request fails. Always use `@Param` on all exported query method parameters.

**Both generic and specific event handlers fire (Q8)**:
This is a source of unexpected double-processing bugs. If you have a generic `@RepositoryEventHandler` for audit logging AND a specific `@RepositoryEventHandler(Product.class)` for product-specific logic, BOTH fire for every Product event. Ensure generic handlers are truly generic (audit logging, timestamp setting) and don't conflict with specific handlers. If order matters, implement `Ordered` on the handler beans.

**Association updates require `text/uri-list` content type (Q7)**:
This is one of the most non-obvious aspects of Spring Data REST. Associations are not updated by embedding entity JSON — they're updated by sending URIs. The `Content-Type: text/uri-list` header is mandatory. Sending `application/json` to an association endpoint behaves differently and may fail. Each line in the `text/uri-list` body is one resource URI. For `@ManyToOne`, one URI. For `@ManyToMany`, multiple URIs (one per line).

---

## 5️⃣ Summary Sheet

### Auto-Generated HTTP Endpoints

| HTTP Method | URL | Repository Method | Notes |
|---|---|---|---|
| `GET` | `/api/products` | `findAll(Pageable)` | HAL collection with pagination |
| `GET` | `/api/products/{id}` | `findById(id)` | Single item or 404 |
| `POST` | `/api/products` | `save(entity)` | Create new, 201 Created |
| `PUT` | `/api/products/{id}` | `save(entity)` | Full replace, 200/204 |
| `PATCH` | `/api/products/{id}` | `save(entity)` | Partial update |
| `DELETE` | `/api/products/{id}` | `deleteById(id)` | 204 No Content |
| `GET` | `/api/products/search` | — | Lists exported query methods |
| `GET` | `/api/products/search/findByX` | `findByX(...)` | Search endpoint |
| `GET` | `/api/products/{id}/category` | — | Association endpoint |
| `PUT` | `/api/products/{id}/category` | — | Update association (text/uri-list) |
| `DELETE` | `/api/products/{id}/category` | — | Remove association |
| `GET` | `/api/profile` | — | ALPS metadata |
| `GET` | `/api/profile/products` | — | Product ALPS descriptor |

### Event Handler Annotations

```
@HandleBeforeCreate  → fires before POST (new entity)
@HandleAfterCreate   → fires after POST (entity has ID)
@HandleBeforeSave    → fires before PUT/PATCH (update)
@HandleAfterSave     → fires after PUT/PATCH
@HandleBeforeDelete  → fires before DELETE
@HandleAfterDelete   → fires after DELETE
@HandleBeforeLinkSave   → fires before association update (PUT on association)
@HandleAfterLinkSave    → fires after association update
@HandleBeforeLinkDelete → fires before association removal (DELETE on association)
@HandleAfterLinkDelete  → fires after association removal
```

### Key Configuration Reference

```java
// RepositoryRestConfigurer methods:
config.setBasePath("/api");
config.setDefaultPageSize(20);
config.setMaxPageSize(100);
config.exposeIdsFor(Product.class);       // expose id field in JSON body
config.setReturnBodyOnCreate(true);       // return body in POST response
config.setReturnBodyOnUpdate(true);       // return body in PUT/PATCH response
config.disableDefaultExposureFor(X.class); // suppress default CRUD endpoints

// Repository annotations:
@RepositoryRestResource(path="products",
    collectionResourceRel="products",
    itemResourceRel="product",
    excerptProjection=Summary.class,
    exported=false)                        // hide entire repository

// Method annotations:
@RestResource(path="customPath",
    rel="customRel",
    exported=false)                        // hide method from HTTP

// Always on search method parameters:
@Param("paramName")  // REQUIRED for HTTP parameter binding
```

### Key Rules

```
1.  Entity IDs NOT in response body by default — use config.exposeIdsFor()
2.  @Param is REQUIRED on all exported search method parameters
3.  excerptProjection applies to collections only, NOT single item GET
4.  @RestResource(exported=false): hides from HTTP, Java calls still work
5.  Association updates use Content-Type: text/uri-list (not JSON)
6.  Both generic and type-specific event handlers fire for matching entity type
7.  @HandleBeforeCreate: entity has no ID yet; @HandleAfterCreate: ID is populated
8.  @Projection interfaces must be in entity's package OR registered manually
9.  Detection strategy ANNOTATED: only @RepositoryRestResource repos exposed
10. DELETE on association endpoint removes the link, not the associated entity
11. @RepositoryEventHandler with no class = fires for ALL entity types
12. Search endpoint GET /api/x/search lists all exported query methods as _links
13. HAL collections: data in _embedded.{collectionResourceRel}
14. text/uri-list for @ManyToOne: one URI; for @ManyToMany: multiple URIs (one per line)
15. ALPS metadata: GET /api/profile (all resources), /api/profile/{resource} (specific)
16. Spring Data REST uses DispatcherServlet — compatible with Spring Security filters
17. @HandleBeforeDelete can throw exception to prevent deletion
18. excerptProjection != projection parameter — excerpt is default, projection overrides
19. config.disableDefaultExposureFor() removes auto-generated CRUD; re-expose selectively
20. Base path in application.yml: spring.data.rest.base-path=/api
```

---
