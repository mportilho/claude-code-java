---
name: jpa-patterns
description: JPA/Hibernate patterns and common pitfalls (N+1, lazy loading, transactions, queries). Use when user has JPA performance issues, LazyInitializationException, or asks about entity relationships and fetching strategies.
---

# JPA Patterns Skill

Best practices and common pitfalls for JPA/Hibernate in Spring Boot 4 /
Hibernate 7.x applications, optimized for modern Java (JDK 21/25).

## When to Use

- User mentions "N+1 problem" / "too many queries"
- LazyInitializationException errors
- Questions about fetch strategies (EAGER vs LAZY)
- Transaction management issues
- Entity relationship design
- Query optimization

---

## Quick Reference: Common Problems

| Problem | Symptom | Solution |
| --- | --- | --- |
| N+1 queries | Many SELECT statements | JOIN FETCH, @EntityGraph |
| LazyInitializationException | Error | DTO projection, service tx, JOIN FETCH |
| Slow | Perf | Pagination |
| Dirty checking overhead | Slow updates | Read-only transactions, DTOs |
| Lost updates | Concurrent modifications | Optimistic locking (@Version) |

---

## N+1 Problem

> The #1 JPA performance killer

### The Problem

```java
// ❌ BAD: N+1 queries
@Entity
public class Author {
    @Id private Long id;
    private String name;

    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    private List<Book> books;
}

// This innocent code...
List<Author> authors = authorRepository.findAll();  // 1 query
for (Author author : authors) {
    System.out.println(author.getBooks().size());   // N queries!
}
// Result: 1 + N queries (if 100 authors = 101 queries)
```

### Solution 1: JOIN FETCH (JPQL)

```java
// ✅ GOOD: Single query with JOIN FETCH
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @Query("SELECT a FROM Author a JOIN FETCH a.books")
    List<Author> findAllWithBooks();
}

// Usage - single query
List<Author> authors = authorRepository.findAllWithBooks();
```

### Solution 2: @EntityGraph

```java
// ✅ GOOD: EntityGraph for declarative fetching
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @EntityGraph(attributePaths = {"books"})
    List<Author> findAll();

    // Or with named graph
    @EntityGraph(value = "Author.withBooks")
    List<Author> findAllWithBooks();
}

// Define named graph on entity
@Entity
@NamedEntityGraph(
    name = "Author.withBooks",
    attributeNodes = @NamedAttributeNode("books")
)
public class Author {
    // ...
}
```

### Solution 3: Batch Fetching

```java
// ✅ GOOD: Batch fetching (Hibernate-specific)
@Entity
public class Author {

    @OneToMany(mappedBy = "author")
    @BatchSize(size = 25)  // Fetch 25 at a time
    private List<Book> books;
}

// Or globally in application.properties
spring.jpa.properties.hibernate.default_batch_fetch_size=25
```

### Detecting N+1

```yaml
# Enable SQL logging to detect N+1
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE
# For older Hibernate versions, BasicBinder may still be used.
```

---

## Lazy Loading

### FetchType Basics

```java
@Entity
public class Order {

    // LAZY: Load only when accessed (default for collections)
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;

    // EAGER: Always load immediately (default for @ManyToOne, @OneToOne)
    @ManyToOne(fetch = FetchType.EAGER)  // ⚠️ Usually bad
    private Customer customer;
}
```

### Best Practice: Default to LAZY

```java
// ✅ GOOD: Always use LAZY, fetch when needed
@Entity
public class Order {

    @ManyToOne(fetch = FetchType.LAZY)  // Override EAGER default
    private Customer customer;

    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;
}
```

### LazyInitializationException

```java
// ❌ BAD: Accessing lazy field outside transaction
@Service
public class OrderService {

    public Order getOrder(Long id) {
        return orderRepository.findById(id).orElseThrow();
    }
}

// In controller (no transaction)
Order order = orderService.getOrder(1L);
order.getItems().size();  // 💥 LazyInitializationException!
```

### Solutions for LazyInitializationException

#### Solution 1: JOIN FETCH in query

```java
// ✅ Fetch needed associations in query
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);
```

`JOIN FETCH` with collections is great to avoid N+1, but avoid combining it
with pagination/limits.

#### Solution 2: @Transactional on service method

```java
// ✅ Keep transaction open while accessing
@Service
public class OrderService {

    @Transactional(readOnly = true)
    public OrderDTO getOrderWithItems(Long id) {
        Order order = orderRepository.findById(id).orElseThrow();
        // Access within transaction
        int itemCount = order.getItems().size();
        return new OrderDTO(order, itemCount);
    }
}
```

#### Solution 3: DTO Projection (recommended)

```java
// ✅ BEST: Return only what you need
public interface OrderSummary {
    Long getId();
    String getStatus();
    int getItemCount();
}

@Query("SELECT o.id as id, o.status as status, SIZE(o.items) as itemCount " +
       "FROM Order o WHERE o.id = :id")
Optional<OrderSummary> findOrderSummary(@Param("id") Long id);
```

#### Solution 4: Open Session in View (NOT recommended, mostly with Virtual Threads)

```yaml
# Keeps session open during view rendering
# ⚠️ Highly dangerous with JDK 21+ Virtual Threads:
# Virtual threads blocked on view rendering will keep the JDBC connection
# pinned, rapidly exhausting DB pool and bringing
# down the application.
spring:
  jpa:
    # Disable explicitly (Spring Boot 4 may default to false, but be safe)
    open-in-view: false
```

---

## Transactions

### Basic Transaction Management

```java
@Service
public class OrderService {

    // Read-only: optimization hint (provider-dependent), often reduces flush
    // and dirty-check overhead
    @Transactional(readOnly = true)
    public Order findById(Long id) {
        return orderRepository.findById(id).orElseThrow();
    }

    // Write: Full transaction with dirty checking
    @Transactional
    public Order createOrder(CreateOrderRequest request) {
        Order order = new Order();
        // ... set properties
        return orderRepository.save(order);
    }

    // Explicit rollback
    @Transactional(rollbackFor = Exception.class)
    public void processPayment(Long orderId) throws PaymentException {
        // Rolls back on any exception, not just RuntimeException
    }
}
```

### Transaction Propagation

```java
@Service
public class OrderService {

    @Autowired
    private PaymentService paymentService;

    @Transactional
    public void placeOrder(Order order) {
        orderRepository.save(order);

        // REQUIRED (default): Uses existing or creates new
        paymentService.processPayment(order);

        // If paymentService throws, entire order is rolled back
    }
}

@Service
public class PaymentService {

    // REQUIRES_NEW: Always creates new transaction
    // If this fails, order can still be saved
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void processPayment(Order order) {
        // Independent transaction
    }

    // MANDATORY: Must run within existing transaction
    @Transactional(propagation = Propagation.MANDATORY)
    public void updatePaymentStatus(Order order) {
        // Throws if no transaction exists
    }
}
```

### Common Transaction Mistakes

```java
// ❌ BAD: Calling @Transactional method from same class
@Service
public class OrderService {

    public void processOrder(Long id) {
        updateOrder(id);  // @Transactional is IGNORED!
    }

    @Transactional
    public void updateOrder(Long id) {
        // Transaction not started because called internally
    }
}

// ✅ GOOD: Move transactional method to a separate service
@Service
public class OrderService {

    @Autowired
    private OrderWriteService orderWriteService;

    public void processOrder(Long id) {
        orderWriteService.updateOrder(id);
    }
}

@Service
public class OrderWriteService {

    @Autowired
    private OrderRepository orderRepository;

    @Transactional
    public void updateOrder(Long id) {
        Order order = orderRepository.findById(id).orElseThrow();
        // update fields...
    }
}
```

---

## Entity Relationships

### OneToMany / ManyToOne

```java
// ✅ GOOD: Bidirectional with proper mapping
@Entity
public class Author {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Book> books = new ArrayList<>();

    // Helper methods for bidirectional sync
    public void addBook(Book book) {
        books.add(book);
        book.setAuthor(this);
    }

    public void removeBook(Book book) {
        books.remove(book);
        book.setAuthor(null);
    }
}

@Entity
public class Book {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private Author author;
}
```

### ManyToMany

```java
// ✅ GOOD for simple association tables: ManyToMany with Set (not List)
@Entity
public class Student {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();

    public void addCourse(Course course) {
        courses.add(course);
        course.getStudents().add(this);
    }

    public void removeCourse(Course course) {
        courses.remove(course);
        course.getStudents().remove(this);
    }
}

@Entity
public class Course {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToMany(mappedBy = "courses")
    private Set<Student> students = new HashSet<>();
}
```

### Preferred for Most Cases: Link Entity Instead of Direct ManyToMany

```java
// ✅ PREFERRED: Model join table as an entity when relationship can evolve
@Entity
public class Enrollment {
    @EmbeddedId
    private EnrollmentId id;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("studentId")
    private Student student;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("courseId")
    private Course course;

    @Column(nullable = false)
    private LocalDate enrolledOn;
}

@Embeddable
public class EnrollmentId implements Serializable {
    private Long studentId;
    private Long courseId;
}

@Entity
public class Student {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "student", cascade = CascadeType.ALL)
    private Set<Enrollment> enrollments = new HashSet<>();
}

@Entity
public class Course {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "course", cascade = CascadeType.ALL, orphanRemoval = true)
    private Set<Enrollment> enrollments = new HashSet<>();
}
```

### Identifier Strategy: TSID / ULID (Modern Applications)

```java
// ❌ BAD for scalability: Standard UUIDs (GenerationType.UUID)
// Standard UUIDs cause severe B-Tree index fragmentation and massive database
// pages overhead.

// ✅ GOOD: Use TSID (Time-Sorted ID) or ULID
// They combine a timestamp with random data: lexicographically sortable,
// cluster-friendly, avoids fragmentation.
@Entity
public class PurchaseOrder {
    @Id
    // Requires hypersistence-utils or similar library, best practice JDK 21+
    @Tsid
    private Long id;
    
    // OR if using String/UUID format:
    // private String ulid;
}
```

### equals() and hashCode() for Entities

Implementing `equals()` and `hashCode()` for JPA entities is tricky because of
database-generated IDs and Hibernate Proxies.

#### 1. Using a Natural / Business Key

If your entity has a unique, immutable business key, this is the easiest and
safest approach.

```java
// ✅ GOOD: Use an immutable business key (Natural ID)
@Entity
public class Book {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NaturalId  // Hibernate annotation for business key
    @Column(unique = true, nullable = false, updatable = false)
    private String isbn;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Book book)) return false;
        return isbn != null && isbn.equals(book.isbn);
    }

    @Override
    public int hashCode() {
        return Objects.hash(isbn);
    }
}
```

#### 2. Using a DB-Generated ID

When using a DB-generated ID (`@GeneratedValue`), the ID is `null` before
persistence. Also, Hibernate wraps entities in proxies (`HibernateProxy`),
which breaks standard `instanceof` and `getClass()` comparisons.

To fix this, we must compare the *effective class* (unwrapping the proxy if
present) and use a constant hash code (so the hash bucket doesn't change
after persistence).

```java
// ✅ GOOD: Robust equals/hashCode for DB-generated IDs
@Entity
public class PurchaseOrder {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Override
    public final boolean equals(Object o) {
        if (this == o) return true;
        if (o == null) return false;
        
        // Unwrap proxy to get the real entity class
        Class<?> oEffectiveClass = o instanceof HibernateProxy 
            ? ((HibernateProxy) o).getHibernateLazyInitializer()
                .getPersistentClass() 
            : o.getClass();
        Class<?> thisEffectiveClass = this instanceof HibernateProxy 
            ? ((HibernateProxy) this).getHibernateLazyInitializer()
                .getPersistentClass() 
            : this.getClass();
            
        if (thisEffectiveClass != oEffectiveClass) return false;
        
        PurchaseOrder other = (PurchaseOrder) o;
        return getId() != null && Objects.equals(getId(), other.getId());
    }

    @Override
    public final int hashCode() {
        // Keeps hash code consistent before and after persistence
        return this instanceof HibernateProxy 
            ? ((HibernateProxy) this).getHibernateLazyInitializer()
                .getPersistentClass().hashCode()
            : getClass().hashCode();
    }
}
```

---

## Query Optimization

### Pagination (Page vs Slice)

```java
// ❌ BAD: Using Page<T> when exact total count is not needed
// Page<T> executes an expensive COUNT() query, scaling poorly on large tables.

// ✅ GOOD: Always paginate large result sets using Slice<T> if you only
// need "Next/Previous" buttons
public interface OrderRepository extends JpaRepository<Order, Long> {

    Slice<Order> findByStatus(OrderStatus status, Pageable pageable);

    // If you absolutely need the total count:
    @Query("SELECT o FROM Order o WHERE o.status = :status")
    Page<Order> findByStatusWithCount(
        @Param("status") OrderStatus status,
        Pageable pageable
    );
}

// Usage
Pageable pageable = PageRequest.of(0, 20, Sort.by("createdAt").descending());
Slice<Order> orders = orderRepository.findByStatus(OrderStatus.PENDING, pageable);
```

For collection relationships, avoid `JOIN FETCH` + pagination in a single query.
Prefer paging root IDs first, then fetching associations in a second query.

### DTO Projections

```java
// ✅ GOOD: Fetch only needed columns

// Interface-based projection
public interface OrderSummary {
    Long getId();
    String getCustomerName();
    BigDecimal getTotal();
}

@Query("SELECT o.id as id, o.customer.name as customerName, o.total as total " +
       "FROM Order o WHERE o.status = :status")
List<OrderSummary> findOrderSummaries(@Param("status") OrderStatus status);

// Class-based projection (DTO)
public record OrderDTO(Long id, String customerName, BigDecimal total) {}

@Query("SELECT new com.example.dto.OrderDTO(o.id, o.customer.name, o.total) " +
       "FROM Order o WHERE o.status = :status")
List<OrderDTO> findOrderDTOs(@Param("status") OrderStatus status);
```

### Bulk Operations

```java
// ✅ GOOD: Bulk update instead of loading entities
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Modifying
    @Query("UPDATE Order o SET o.status = :status WHERE o.createdAt < :date")
    int updateOldOrdersStatus(
        @Param("status") OrderStatus status,
        @Param("date") LocalDateTime date
    );

    @Modifying
    @Query("DELETE FROM Order o WHERE o.status = :status AND o.createdAt < :date")
    int deleteOldOrders(
        @Param("status") OrderStatus status,
        @Param("date") LocalDateTime date
    );
}

// Usage
@Transactional
public void archiveOldOrders() {
    LocalDateTime threshold = LocalDateTime.now().minusYears(1);
    int updated = orderRepository.updateOldOrdersStatus(
        OrderStatus.ARCHIVED,
        threshold
    );
    log.info("Archived {} orders", updated);
}
```

---

## Optimistic Locking

### Prevent Lost Updates

```java
// ✅ GOOD: Use @Version for optimistic locking
@Entity
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Version
    private Long version;

    private OrderStatus status;
    private BigDecimal total;
}

// When two users update same order:
// User 1: loads order (version=1), modifies, saves → version becomes 2
// User 2: loads order (version=1), modifies, saves → OptimisticLockException!
```

### Handling OptimisticLockException

```java
@Service
public class OrderService {

    @Transactional
    public Order updateOrder(Long id, UpdateOrderRequest request) {
        try {
            Order order = orderRepository.findById(id).orElseThrow();
            order.setStatus(request.getStatus());
            return orderRepository.save(order);
        } catch (OptimisticLockException e) {
            throw new ConcurrentModificationException(
                "Order was modified by another user. Please refresh and try again."
            );
        }
    }

    // Or with retry
    @Retryable(value = OptimisticLockException.class, maxAttempts = 3)
    @Transactional
    public Order updateOrderWithRetry(Long id, UpdateOrderRequest request) {
        Order order = orderRepository.findById(id).orElseThrow();
        order.setStatus(request.getStatus());
        return orderRepository.save(order);
    }
}
```

---

## Common Mistakes

### 1. Cascade Misuse

```java
// ❌ BAD: CascadeType.ALL on @ManyToOne
@Entity
public class Book {
    @ManyToOne(cascade = CascadeType.ALL)  // Dangerous!
    private Author author;
}
// Deleting a book could delete the author!

// ✅ GOOD: Cascade only from parent to child
@Entity
public class Author {
    @OneToMany(mappedBy = "author", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Book> books;
}
```

### 2. Missing Index

```java
// ❌ BAD: Frequent queries on non-indexed column
@Query("SELECT o FROM Order o WHERE o.customerEmail = :email")
List<Order> findByCustomerEmail(@Param("email") String email);

// ✅ GOOD: Add index
@Entity
@Table(indexes = @Index(name = "idx_order_customer_email", columnList = "customerEmail"))
public class Order {
    private String customerEmail;
}
```

### 3. toString() with Lazy Fields

```java
// ❌ BAD: toString includes lazy collection
@Entity
public class Author {
    @OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
    private List<Book> books;

    @Override
    public String toString() {
        return "Author{id=" + id + ", books=" + books + "}";  // Triggers lazy load!
    }
}

// ✅ GOOD: Exclude lazy fields from toString
@Override
public String toString() {
    return "Author{id=" + id + ", name='" + name + "'}";
}
```

---

## Performance Checklist

When reviewing JPA code, check:

- [ ] No N+1 queries (use JOIN FETCH or @EntityGraph)
- [ ] LAZY fetch by default (especially @ManyToOne)
- [ ] Avoid JOIN FETCH + pagination for collections
- [ ] Pagination for large result sets
- [ ] DTO projections for read-only queries
- [ ] Bulk operations for batch updates/deletes
- [ ] Prefer link entity over direct @ManyToMany in non-trivial relationships
- [ ] @Version for entities with concurrent access
- [ ] Indexes on frequently queried columns
- [ ] No lazy fields in toString()
- [ ] Read-only transactions where applicable

---

## Related Skills

- `spring-boot-patterns` - Spring Boot controller/service patterns
- `java-code-review` - General code review checklist
- `clean-code` - Code quality principles
