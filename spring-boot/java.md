# Java Essentials Before Spring Boot

Spring Boot leans on Java's type system harder than most frameworks — repositories are interfaces
Spring implements _for_ you at runtime, entities are plain classes, base behavior is shared via
abstract classes, and almost every controller/service method passes data around as a `List`,
`Map`, `Optional`, or a `Stream` pipeline. This note covers the Java you need solid before any of
that clicks.

```
Part A — Core Java syntax (the everyday building blocks)
Part B — Class vs. Abstract Class vs. Interface (the deep dive)
Part C — The Collections Framework (the deep dive)
Part D — Where this all shows up in a real Spring Boot app
Part E — Self-check before you start building
```

---

## Part A — Core Java Syntax Essentials

### A.1 Primitives vs. reference types

```java
int age = 30;            // primitive — stored directly, has a default value (0), can't be null
double price = 19.99;
boolean active = true;

String username = "alice";   // reference type — an object, CAN be null
User user = new User();       // reference type
```

This distinction matters constantly in Spring: a `Long id` field can't be `null` and silently
defaults to `0`, while a `Long id` (capital-L, the **wrapper** class) can be `null` — which is
exactly why JPA entity IDs are declared as `Long`, not `long`: before the row is saved and an ID
is assigned, the field needs to legitimately be `null`.

### A.2 Access modifiers — who can see what

| Modifier                | Same class | Same package | Subclass (other package) | Everywhere |
| ----------------------- | ---------- | ------------ | ------------------------ | ---------- |
| `private`               | ✅         | ❌           | ❌                       | ❌         |
| _(default, no keyword)_ | ✅         | ✅           | ❌                       | ❌         |
| `protected`             | ✅         | ✅           | ✅                       | ❌         |
| `public`                | ✅         | ✅           | ✅                       | ✅         |

```java
public class User {
    private Long id;          // only accessible inside User itself — typical for entity fields
    private String username;

    public Long getId() { return id; }              // public getters expose state safely
    public void setUsername(String u) { this.username = u; }
}
```

Spring convention: fields are almost always `private`, with `public` getters/setters (or none at
all if you use Lombok's `@Data`/`@Getter`/`@Setter` to generate them) — direct field access from
outside the class is avoided even when technically possible.

### A.3 Static vs. instance members

```java
public class MathUtils {
    static final double PI = 3.14159;            // belongs to the CLASS, not any instance
    static double square(double x) { return x * x; } // called as MathUtils.square(4), no `new` needed
}

public class User {
    private String username; // INSTANCE field — every User object has its own copy
    String getUsername() { return username; } // instance method — needs an actual object to call
}
```

Spring beans (the objects Spring creates and manages — `@Service`, `@Repository`, `@Controller`
classes) are **instances**, not static utility classes — this is part of why Spring can swap
implementations, inject dependencies, and manage object lifecycles, none of which is possible with
purely `static` code.

### A.4 Constructors

```java
public class User {
    private Long id;
    private String username;

    public User() { }                          // no-arg constructor — required by JPA entities

    public User(String username) {              // parameterized constructor
        this.username = username;               // `this.username` (the field) vs `username` (the param)
    }

    public User(Long id, String username) {
        this(username);                          // `this(...)` calls another constructor in the same class
        this.id = id;
    }
}
```

> ⚠️ JPA entities **require** a no-argument constructor (even if `protected`, not necessarily
> `public`) — Hibernate instantiates entities via reflection and populates fields afterward. If
> you add a custom parameterized constructor and forget the no-arg one, entity loading breaks.

### A.5 `final`

```java
final double TAX_RATE = 0.08;       // can't be reassigned after initialization
final class ImmutablePoint { }        // can't be subclassed
public final void validate() { }      // can't be overridden by a subclass
```

### A.6 Generics — the `<T>` you'll see in almost every Spring type signature

```java
public class Box<T> {
    private T value;
    public T get() { return value; }
    public void set(T value) { this.value = value; }
}

Box<String> stringBox = new Box<>();
```

You don't need to write much generic code yourself early on, but you need to **read** it fluently
— `List<User>`, `Optional<User>`, `ResponseEntity<UserDTO>`, and
`JpaRepository<User, Long>` (entity type, ID type) are generics you'll type daily in Spring Boot.

### A.7 Annotations — what they actually are

```java
@Override
public String toString() { return username; }

@Deprecated
public void oldMethod() { }
```

An annotation is **metadata attached to code** — it doesn't run anything by itself. `@Override` is
checked by the compiler; most Spring annotations (`@Service`, `@Autowired`, `@RestController`,
`@GetMapping`) are read by **Spring at startup/runtime via reflection**, and Spring's framework
code decides what to do based on which annotations it finds. Understanding "annotations are just
readable metadata, not magic syntax" demystifies a huge amount of how Spring Boot actually works.

### A.8 Exception handling

```java
try {
    int result = 10 / divisor;
} catch (ArithmeticException e) {
    System.out.println("Can't divide by zero: " + e.getMessage());
} finally {
    System.out.println("Always runs, success or failure");
}

// custom exception — the pattern you'll use for "not found", "invalid input", etc.
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

throw new ResourceNotFoundException("User " + id + " not found");
```

**Checked vs. unchecked**, the one Java-specific distinction worth knowing: checked exceptions
(`extends Exception`, e.g. `IOException`) **must** be either caught or declared with `throws` —
the compiler enforces it. Unchecked exceptions (`extends RuntimeException`) don't require this.
Spring Boot's own conventions (and most custom exceptions you'll write, like the example above)
favor `RuntimeException` subclasses specifically so they don't force `throws` declarations
through every layer of the call stack — they're typically caught centrally by a
`@ControllerAdvice`/`@ExceptionHandler` instead, covered in Part D.

### A.9 Lambda expressions & functional interfaces (Java 8+)

```java
// a functional interface: exactly ONE abstract method
@FunctionalInterface
interface Validator { boolean isValid(String input); }

Validator notEmpty = input -> input != null && !input.isEmpty(); // lambda implementing it

// built-in functional interfaces you'll use constantly with Streams (Part C):
Function<User, String> getUsername = u -> u.getUsername();   // takes T, returns R
Predicate<User> isActive = u -> u.isActive();                  // takes T, returns boolean
Consumer<User> printUser = u -> System.out.println(u);          // takes T, returns nothing
Supplier<User> defaultUser = () -> new User("guest");           // takes nothing, returns T
```

Lambdas are the syntax that makes the Streams API (Part C.6) readable — `users.stream().filter(u
-> u.isActive())` is a lambda passed as a `Predicate<User>`.

---

## Part B — Class vs. Abstract Class vs. Interface

The four OOP pillars in one line each, since this section is really about how Java expresses them:
**encapsulation** (private fields + public methods, A.2), **abstraction** (hiding implementation
behind a simpler contract — abstract classes and interfaces, below), **inheritance** (`extends`),
**polymorphism** (treating different concrete types through one shared type — exactly what lets
Spring inject _any_ implementation of an interface wherever that interface is expected).

### B.1 Concrete class

```java
public class User {
    private Long id;
    private String username;

    public User(Long id, String username) {
        this.id = id;
        this.username = username;
    }

    public String getUsername() { return username; }
}

User u = new User(1L, "alice"); // instantiable — this is the defining trait of a plain class
```

### B.2 Abstract class

```java
public abstract class BaseEntity {
    protected Long id;
    protected LocalDateTime createdAt = LocalDateTime.now();

    public BaseEntity() { }                 // abstract classes CAN have constructors —
                                              // called via super() from a subclass, never directly

    public abstract String describe();       // no body — every subclass MUST implement this

    public Long getId() { return id; }        // concrete method — shared as-is by every subclass
}

public class Post extends BaseEntity {
    private String caption;

    public Post(String caption) { this.caption = caption; }

    @Override
    public String describe() {               // required — Post would fail to compile without this
        return "Post: " + caption;
    }
}

// new BaseEntity();   ❌ compile error — cannot instantiate an abstract class
BaseEntity entity = new Post("hello");        // ✅ fine — reference the abstract TYPE, hold a concrete instance
```

An abstract class is for **closely related subclasses that share real state and some shared
behavior**, while leaving specific pieces for each subclass to fill in.

### B.3 Interface

```java
public interface UserService {
    User createUser(String username, String email, String password); // implicitly public abstract
    Optional<User> findById(Long id);

    default boolean isValidEmail(String email) {   // default method (Java 8+) — has a body,
        return email != null && email.contains("@"); // inherited by implementers unless overridden
    }

    static UserService noOp() {                     // static method (Java 8+) — called on the interface itself
        return null;
    }
}

public class UserServiceImpl implements UserService {
    @Override
    public User createUser(String username, String email, String password) { /* ... */ return null; }

    @Override
    public Optional<User> findById(Long id) { /* ... */ return Optional.empty(); }
    // isValidEmail is inherited for free — no need to re-implement it
}

// a class can implement MULTIPLE interfaces — this is Java's answer to multiple inheritance
public class Post implements Serializable, Comparable<Post> {
    @Override
    public int compareTo(Post other) { return this.createdAt.compareTo(other.createdAt); }
}
```

All fields on an interface are implicitly `public static final` (constants) — interfaces hold no
**instance state**, which is the core structural difference from an abstract class.

### B.4 Side-by-side comparison

|                            | Concrete Class                              | Abstract Class                                                           | Interface                                                                             |
| -------------------------- | ------------------------------------------- | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------- |
| **Instantiable?**          | Yes (`new X()`)                             | No                                                                       | No                                                                                    |
| **Constructors?**          | Yes                                         | Yes (called via `super()` from a subclass)                               | No                                                                                    |
| **Instance fields/state?** | Yes, any modifier                           | Yes, any modifier                                                        | No — only `public static final` constants                                             |
| **Method bodies?**         | All methods have bodies                     | Can mix abstract (no body) + concrete                                    | Java 8+: `default`/`static`/`private` methods can have bodies; others remain abstract |
| **Inheritance**            | `extends` one class                         | `extends` one abstract class                                             | A class `implements` **multiple** interfaces                                          |
| **Interface-to-interface** | —                                           | —                                                                        | An interface can `extends` multiple other interfaces                                  |
| **Typical use**            | A concrete "thing" you instantiate directly | Shared state + partial shared behavior across closely related subclasses | A contract/capability — "can do X" — implementable by otherwise-unrelated classes     |

### B.5 Why Spring leans so heavily on interfaces

This is the part that makes the difference matter practically, not just academically.

```java
public interface PostRepository extends JpaRepository<Post, Long> {
    List<Post> findByUserId(Long userId); // YOU write only this signature...
}
```

...and Spring Data JPA **generates the implementation class for you at runtime**, by parsing the
method name (`findByUserId` → `WHERE user_id = ?`) and creating a dynamic proxy. You never write
or even see the implementing class. This only works _because_ `PostRepository` is an interface —
Spring needs a contract it can implement on your behalf, not a concrete class whose body is
already fixed.

The same shape repeats in the service layer, motivated by a different benefit — **swappable
implementations and easy testing**:

```java
public interface PaymentService {
    boolean charge(Long userId, BigDecimal amount);
}

@Service
public class StripePaymentService implements PaymentService {
    public boolean charge(Long userId, BigDecimal amount) { /* real Stripe call */ return true; }
}

@Service
public class MockPaymentService implements PaymentService {
    public boolean charge(Long userId, BigDecimal amount) { return true; } // used in tests
}
```

Any class that depends on `PaymentService` (via constructor injection) doesn't know or care which
implementation it's actually holding — production wiring injects `StripePaymentService`, a test
can inject `MockPaymentService`, and the dependent code is unchanged either way. This is
**polymorphism** doing real architectural work, not just a textbook definition.

**Quick rule of thumb for which to reach for:**

- Need to actually create objects of this type? → **class**
- A family of closely related types that share real fields and some shared logic, but each needs
  to fill in specific behavior? → **abstract class**
- Defining "any class that can do X" as a contract, possibly across totally unrelated classes,
  or you need Spring to generate/inject an implementation for you? → **interface**

---

## Part C — The Collections Framework

### C.1 The hierarchy

```
Iterable
  └── Collection
        ├── List   (ordered, duplicates allowed)        → ArrayList, LinkedList
        ├── Set    (no duplicates)                        → HashSet, LinkedHashSet, TreeSet
        └── Queue  (FIFO / priority processing)            → ArrayDeque, PriorityQueue

Map  (separate hierarchy — NOT a Collection, key→value)   → HashMap, LinkedHashMap, TreeMap
```

### C.2 `List` — ordered, allows duplicates, indexable

```java
List<String> tags = new ArrayList<>();    // resizable array underneath — DEFAULT choice
tags.add("java");
tags.add("spring");
String first = tags.get(0);                // fast, O(1) random access

List<String> linked = new LinkedList<>();   // doubly-linked list underneath
linked.addFirst("urgent");                   // fast O(1) insert at the ends
```

|                            | `ArrayList`                                                  | `LinkedList`                                                                        |
| -------------------------- | ------------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| Random access (`get(i)`)   | O(1) — fast                                                  | O(n) — slow, must walk the list                                                     |
| Insert/remove at start/end | O(n) (start), amortized O(1) (end)                           | O(1)                                                                                |
| Typical use in Spring      | The default for almost everything — DTO lists, query results | Rare in typical web apps; only when you genuinely need frequent head/tail insertion |

### C.3 `Set` — no duplicates

```java
Set<String> roles = new HashSet<>();        // no order guarantee, O(1) average add/contains
roles.add("ADMIN");
roles.add("ADMIN");                          // silently ignored — already present

Set<String> ordered = new LinkedHashSet<>(); // preserves insertion order, same uniqueness guarantee
Set<Integer> sortedIds = new TreeSet<>();     // always sorted, O(log n) operations
```

Use a `Set` whenever "no duplicates" is part of the actual requirement (a user's set of role
names, a set of tag IDs) rather than just habit — reaching for `List` and manually checking
`contains()` before adding is the same idea, done worse.

### C.4 `Map` — key/value pairs

```java
Map<Long, User> userById = new HashMap<>();
userById.put(1L, user);
User found = userById.get(1L);
boolean exists = userById.containsKey(1L);

Map<String, Long> roleCounts = new HashMap<>();
roleCounts.merge("ADMIN", 1L, Long::sum);     // increment-or-initialize in one call

for (Map.Entry<Long, User> entry : userById.entrySet()) {
    System.out.println(entry.getKey() + " -> " + entry.getValue());
}
```

`HashMap` (no order guarantee) is the default; `LinkedHashMap` preserves insertion order;
`TreeMap` keeps keys sorted. This shape — grouping or caching by ID — is extremely common in
service-layer code, often built directly from a Stream (C.6).

### C.5 Iterating safely

```java
for (User u : users) { ... }                          // for-each — most common
users.forEach(u -> System.out.println(u));               // lambda equivalent

Iterator<User> it = users.iterator();
while (it.hasNext()) {
    User u = it.next();
    if (!u.isActive()) it.remove();                       // the ONLY safe way to remove while iterating
}
```

> ⚠️ Calling `list.remove(...)` directly inside a `for-each` loop throws
> `ConcurrentModificationException` — use `Iterator.remove()`, or build a filtered copy instead
> (which is what the Streams approach below does naturally).

### C.6 The Streams API — the pattern you'll write constantly in Spring services

```java
List<UserDTO> dtos = users.stream()
    .filter(User::isActive)                              // keep matching elements
    .map(u -> new UserDTO(u.getId(), u.getUsername()))     // transform each element
    .sorted(Comparator.comparing(UserDTO::getUsername))     // sort
    .collect(Collectors.toList());                          // materialize back into a List

long activeCount = users.stream().filter(User::isActive).count();

Map<String, List<User>> byRole = users.stream()
    .collect(Collectors.groupingBy(User::getRole));          // group into a Map<role, List<User>>
```

**This Entity → DTO conversion is the single most common line of code in a Spring service or
controller layer** — pulling a `List<Entity>` from a repository and mapping it to a
`List<ResponseDTO>` before returning it from a `@RestController` method. Method references
(`User::isActive`, `User::getUsername`) are shorthand for a lambda that just calls that method —
`u -> u.isActive()` and `User::isActive` are equivalent.

### C.7 `Optional<T>` — how Spring Data avoids returning `null`

```java
Optional<User> maybeUser = userRepository.findById(id); // never null itself; may be EMPTY

User user = maybeUser.orElseThrow(() ->
    new ResourceNotFoundException("User " + id + " not found"));

// the idiomatic one-line service method built from this:
public User getUser(Long id) {
    return userRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("User " + id + " not found"));
}

maybeUser.ifPresent(u -> System.out.println("Found: " + u.getUsername()));
String name = maybeUser.map(User::getUsername).orElse("unknown");
```

Every `JpaRepository.findById(...)` call returns `Optional<T>`, specifically so callers are forced
to consciously handle "not found" instead of risking a `NullPointerException` three layers away
from where the missing value originated.

### C.8 Sorting: `Comparable` vs. `Comparator`

```java
// Comparable — ONE natural ordering, defined INSIDE the class itself
public class User implements Comparable<User> {
    public int compareTo(User other) {
        return this.username.compareTo(other.username);
    }
}
Collections.sort(users); // uses compareTo automatically

// Comparator — defined OUTSIDE the class, as many different orderings as you want
users.sort(Comparator.comparing(User::getUsername));
users.sort(Comparator.comparing(User::getCreatedAt).reversed());
users.sort(Comparator.comparing(User::getRole).thenComparing(User::getUsername)); // multi-key sort
```

Use `Comparable` for the one "obvious default" ordering a type has; use `Comparator` for every
other situational ordering — most real sorting in a service layer ends up being `Comparator`,
applied inline where the sort is actually needed.

### C.9 Immutable collections

```java
List<String> roles = List.of("ADMIN", "USER");           // immutable — throws on add()/remove()
Set<String> tags = Set.of("java", "spring");
List<String> readOnlyView = Collections.unmodifiableList(mutableList); // wraps an existing list
```

Returning `List.of(...)` (or wrapping with `unmodifiableList`) from a method that hands out
internal state is a small but real defensive habit — it stops a caller from mutating a collection
they were never supposed to own.

---

## Part D — Where This All Shows Up in a Real Spring Boot App

A condensed, realistic slice tying every concept above together:

```java
// 1) Entity — a concrete CLASS, uses access modifiers + a no-arg constructor (A.2, A.4)
@Entity
public class Post extends BaseAuditingEntity {     // 2) extends an ABSTRACT CLASS for shared fields
    @Id @GeneratedValue
    private Long id;
    private String caption;
    @ManyToOne
    private User user;

    protected Post() { }                              // required no-arg constructor for JPA
    public Post(String caption, User user) {
        this.caption = caption;
        this.user = user;
    }
    // getters omitted for brevity
}

// the shared abstract base every entity extends
public abstract class BaseAuditingEntity {
    @CreationTimestamp
    protected LocalDateTime createdAt;                 // shared STATE
    public LocalDateTime getCreatedAt() { return createdAt; } // shared, concrete behavior
}

// 3) Repository — an INTERFACE; Spring generates the implementation at runtime (Part B.5)
public interface PostRepository extends JpaRepository<Post, Long> {
    List<Post> findByUserId(Long userId);
}

// 4) Service — INTERFACE + IMPL CLASS, for swappability and testability (Part B.5)
public interface PostService {
    List<PostDTO> getPostsForUser(Long userId);
}

@Service
public class PostServiceImpl implements PostService {
    private final PostRepository postRepository;

    public PostServiceImpl(PostRepository postRepository) {  // constructor injection
        this.postRepository = postRepository;
    }

    @Override
    public List<PostDTO> getPostsForUser(Long userId) {
        return postRepository.findByUserId(userId).stream()   // 5) COLLECTIONS + STREAMS (Part C)
            .map(p -> new PostDTO(p.getId(), p.getCaption(), p.getCreatedAt()))
            .collect(Collectors.toList());
    }
}

// 6) Controller — returns a List<DTO>, built entirely from the pipeline above
@RestController
@RequestMapping("/posts")
public class PostController {
    private final PostService postService;

    public PostController(PostService postService) {
        this.postService = postService;
    }

    @GetMapping("/user/{userId}")
    public List<PostDTO> getUserPosts(@PathVariable Long userId) {
        return postService.getPostsForUser(userId);
    }
}

// 7) Exception handling — a custom RuntimeException (Part A.8) caught centrally
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<String> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(ex.getMessage());
    }
}
```

Every numbered comment above maps directly back to a section in this note — that's deliberate:
this _is_ what "Java essentials for Spring Boot" cashes out to in practice.

---

## Part E — Self-Check Before You Start Building

- [ ] Explain why a JPA entity needs a `private` ID field, a no-arg constructor, and public
      getters — and what would break if any were missing
- [ ] State the three-way difference between a class, an abstract class, and an interface without
      looking at the table — instantiability, state, and multiple inheritance specifically
- [ ] Explain why `PostRepository` can be an interface with no implementation written anywhere in
      your code, and still work at runtime
- [ ] Convert a `List<Entity>` to a `List<DTO>` using `.stream().map(...).collect(...)` from memory
- [ ] Explain what `Optional<T>` is protecting against, and write `.orElseThrow(...)` correctly
- [ ] Pick `ArrayList` vs `LinkedList`, and `HashSet`/`HashMap` vs their `Tree`/`Linked` variants,
      and justify the choice in one sentence each
- [ ] Explain why a custom exception typically extends `RuntimeException` rather than `Exception`
      in a Spring Boot codebase

If any of these feel shaky, re-read that section with an editor open — Part B and Part D
especially are best understood by typing the examples out, not just reading them.
