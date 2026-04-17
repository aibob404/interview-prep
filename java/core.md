# Core Java

## 1. OOP Principles — Senior Depth

### Encapsulation
More than private fields. It means designing **invariant-preserving boundaries** — no caller should be able to put an object into an illegal state.

**Bad encapsulation:**
```java
public class Order {
    public List<Item> items = new ArrayList<>(); // mutable reference exposed
}
```

**Proper encapsulation:**
```java
public class Order {
    private final List<Item> items = new ArrayList<>();

    public void addItem(Item item) {
        Objects.requireNonNull(item);
        items.add(item);
    }

    public List<Item> getItems() {
        return Collections.unmodifiableList(items); // defensive copy
    }
}
```

### Inheritance vs Composition
> Prefer composition over inheritance (Effective Java, Item 18)

Inheritance creates tight coupling — a subclass is dependent on the implementation details of its superclass. If the parent changes, subclasses silently break.

```java
// Fragile inheritance
public class InstrumentedHashSet<E> extends HashSet<E> {
    private int addCount = 0;

    @Override public boolean add(E e) { addCount++; return super.add(e); }

    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c); // calls add() internally — double-counts!
    }
}

// Robust composition
public class InstrumentedSet<E> {
    private final Set<E> delegate;
    private int addCount = 0;

    public InstrumentedSet(Set<E> s) { this.delegate = s; }

    public boolean add(E e) { addCount++; return delegate.add(e); }
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return delegate.addAll(c);
    }
}
```

### Polymorphism — Runtime vs Compile-time
- **Runtime (dynamic dispatch):** method overriding, resolved by JVM at runtime based on the actual object type.
- **Compile-time:** method overloading, resolved by compiler based on declared parameter types.

**Critical trap:**
```java
class Animal { void speak() { System.out.println("..."); } }
class Dog extends Animal { void speak() { System.out.println("Woof"); } }

Animal a = new Dog();
a.speak(); // "Woof" — runtime dispatch

// Overloading trap
void process(Animal a) { System.out.println("animal"); }
void process(Dog d) { System.out.println("dog"); }

Animal a = new Dog();
process(a); // "animal" — compile-time resolution uses declared type
```

### Abstraction
Design interfaces around *what*, not *how*. Expose behavior, hide implementation.

**Interface segregation (ISP from SOLID):**
```java
// Bad — fat interface forces unused implementations
interface Worker {
    void work();
    void eat(); // Robots don't eat!
}

// Good — segregated
interface Workable { void work(); }
interface Eatable  { void eat();  }
```

---

## 2. SOLID Principles

| Principle | Key idea | Violation sign |
|-----------|----------|----------------|
| **S**ingle Responsibility | One class, one reason to change | Class does DB access + business logic + formatting |
| **O**pen/Closed | Open for extension, closed for modification | Adding a feature requires changing existing code |
| **L**iskov Substitution | Subtypes must be substitutable for base types | Overriding method narrows contract (throws more exceptions) |
| **I**nterface Segregation | Clients shouldn't depend on unused methods | Large interfaces with default no-op throws |
| **D**ependency Inversion | Depend on abstractions, not concretions | `new` inside business logic, `new UserRepository()` |

**Liskov Substitution — the subtle one:**
```java
// Violation: Rectangle → Square
public class Rectangle {
    protected int width, height;
    public void setWidth(int w) { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int area() { return width * height; }
}

public class Square extends Rectangle {
    @Override public void setWidth(int w) { this.width = this.height = w; } // breaks LSP
    @Override public void setHeight(int h) { this.width = this.height = h; }
}

// Consumer breaks with Square:
Rectangle r = new Square();
r.setWidth(5);
r.setHeight(4);
assert r.area() == 20; // FAILS — LSP violated
```

---

## 3. Generics

### Type Erasure
Generic types are **erased at compile time** — `List<String>` becomes `List` at runtime. Consequences:
- Cannot use `instanceof` with generic types: `x instanceof List<String>` is a compile error
- Cannot create generic arrays: `new T[]` is illegal
- Cannot overload methods that differ only by generic parameter

```java
// Both compile to the same bytecode signature — compile error
void process(List<String> list) { }
void process(List<Integer> list) { } // duplicate method!
```

### Wildcards and PECS
**PECS: Producer Extends, Consumer Super**

```java
// Producer (you READ from it) → use extends
public double sumList(List<? extends Number> list) {
    return list.stream().mapToDouble(Number::doubleValue).sum();
}

// Consumer (you WRITE to it) → use super
public void addNumbers(List<? super Integer> list) {
    list.add(1);
    list.add(2);
}

// Both: use two parameters
public <T> void copy(List<? extends T> src, List<? super T> dst) {
    dst.addAll(src);
}
```

### Bounded Type Parameters
```java
// Upper bound
public <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) >= 0 ? a : b;
}

// Multiple bounds
public <T extends Serializable & Comparable<T>> void store(T obj) { ... }
```

### Generic Methods vs Wildcards
Prefer generic methods when the type relationship matters between parameters:
```java
// Wildcard — loses type info
void swap(List<?> list, int i, int j) {
    Object tmp = list.get(i);
    // list.set(i, list.get(j)); // compile error — can't set into List<?>
}

// Generic method — type-safe
<T> void swap(List<T> list, int i, int j) {
    T tmp = list.get(i);
    list.set(i, list.get(j));
    list.set(j, tmp);
}
```

---

## 4. Lambdas and Streams

### Functional Interfaces
```java
@FunctionalInterface
interface Transformer<T, R> {
    R transform(T input);
    // Can have default/static methods — still functional
    default Transformer<T, R> andLog() {
        return input -> {
            R result = this.transform(input);
            System.out.println(input + " → " + result);
            return result;
        };
    }
}
```

Built-in functional interfaces:

| Interface | Method | Use |
|-----------|--------|-----|
| `Function<T,R>` | `R apply(T t)` | Transform |
| `Predicate<T>` | `boolean test(T t)` | Filter |
| `Consumer<T>` | `void accept(T t)` | Side effect |
| `Supplier<T>` | `T get()` | Lazy evaluation |
| `BiFunction<T,U,R>` | `R apply(T t, U u)` | Two inputs |
| `UnaryOperator<T>` | `T apply(T t)` | Same type in/out |

### Stream Internals — Laziness
Intermediate operations create a **pipeline** — no work is done until a terminal operation triggers evaluation.

```java
List<String> result = IntStream.range(0, 1_000_000)
    .mapToObj(i -> {
        System.out.println("mapping " + i); // only runs for matching elements
        return "item-" + i;
    })
    .filter(s -> s.endsWith("7"))
    .limit(3) // pipeline stops after 3 matches
    .collect(Collectors.toList());
```

### Collectors
```java
// Grouping
Map<Department, List<Employee>> byDept =
    employees.stream().collect(Collectors.groupingBy(Employee::getDepartment));

// Grouping with downstream collector
Map<Department, Long> countByDept =
    employees.stream().collect(
        Collectors.groupingBy(Employee::getDepartment, Collectors.counting()));

// Partitioning
Map<Boolean, List<Employee>> partitioned =
    employees.stream().collect(
        Collectors.partitioningBy(e -> e.getSalary() > 100_000));

// Custom joining
String names = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.joining(", ", "[", "]"));

// toMap — watch for duplicate key exception
Map<String, Employee> byName = employees.stream()
    .collect(Collectors.toMap(
        Employee::getName,
        Function.identity(),
        (a, b) -> a)); // merge function for duplicates
```

### Parallel Streams — When to Use
```java
// GOOD: CPU-bound, large dataset, no shared mutable state
long sum = LongStream.rangeClosed(1, 10_000_000)
    .parallel()
    .sum();

// BAD: I/O-bound (default pool is shared with all parallel streams)
// BAD: Small datasets (overhead > benefit, typically < 10k elements)
// BAD: Ordered operations on parallel streams (forEachOrdered serializes)
// BAD: Stateful operations (sorted, distinct) — require buffering all elements
```

---

## 5. String

### Why String is Immutable
1. **String pool** — identical literals share the same object
2. **Thread safety** — immutable objects need no synchronization
3. **Security** — class names, network hosts passed as String can't be changed mid-operation
4. **HashMap keys** — hashCode can be cached since value never changes

```java
// String pool
String a = "hello";
String b = "hello";
a == b; // true — same object from pool

String c = new String("hello");
a == c; // false — new object on heap
a.equals(c); // true
```

### StringBuilder vs StringBuffer
- `StringBuilder` — not thread-safe, use in single-threaded contexts (preferred)
- `StringBuffer` — thread-safe (synchronized), use when shared across threads
- String concatenation in loops compiles to `StringBuilder`, but in **pre-Java 9** each `+` in a loop created a new `StringBuilder`

### String.intern()
Forces a string into the pool — useful when you have many duplicate strings from external sources (e.g., JSON parsing). Use with care — the pool is in Metaspace (Java 8+) and overcrowding causes GC pressure.

---

## 6. Records (Java 16+)

Records are **transparent, immutable data carriers**:
```java
public record Point(int x, int y) {
    // Compact canonical constructor — validation
    public Point {
        if (x < 0 || y < 0) throw new IllegalArgumentException("Negative coordinates");
    }

    // Custom methods are allowed
    public double distanceTo(Point other) {
        return Math.hypot(this.x - other.x, this.y - other.y);
    }
}
```

Auto-generated: constructor, getters (`x()`, `y()`), `equals()`, `hashCode()`, `toString()`.

**Limitations:** cannot extend other classes (implicitly extends `Record`), fields are always `final`, cannot add instance fields outside the header.

---

## 7. Sealed Classes (Java 17+)

Restrict which classes can extend/implement a type — exhaustive pattern matching becomes possible:

```java
public sealed interface Shape permits Circle, Rectangle, Triangle {}

public record Circle(double radius) implements Shape {}
public record Rectangle(double w, double h) implements Shape {}
public record Triangle(double a, double b, double c) implements Shape {}

// Exhaustive switch — compiler checks all cases
double area = switch (shape) {
    case Circle c      -> Math.PI * c.radius() * c.radius();
    case Rectangle r   -> r.w() * r.h();
    case Triangle t    -> heronArea(t.a(), t.b(), t.c());
};
```

---

## 8. Exceptions

### Checked vs Unchecked
- **Checked** (`Exception` subclasses, not `RuntimeException`) — compiler forces handling. Use for recoverable conditions where callers need to take action.
- **Unchecked** (`RuntimeException`, `Error`) — programming errors or unrecoverable conditions.

> Effective Java: avoid checked exceptions when callers can't reasonably recover. Overuse of checked exceptions pollutes APIs and forces boilerplate try-catch.

### Exception Hierarchy
```
Throwable
├── Error          (OutOfMemoryError, StackOverflowError — don't catch)
└── Exception
    ├── IOException, SQLException, ...  (checked)
    └── RuntimeException
        ├── NullPointerException
        ├── IllegalArgumentException
        ├── IllegalStateException
        └── ...
```

### Best Practices
```java
// 1. Prefer specific exceptions
throw new IllegalArgumentException("Amount must be positive: " + amount);

// 2. Exception chaining — preserve root cause
try {
    doSomething();
} catch (IOException e) {
    throw new ServiceException("Failed to process request", e); // chain it
}

// 3. Try-with-resources for AutoCloseable
try (Connection conn = ds.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql)) {
    // both closed automatically, even if an exception occurs
}

// 4. Don't swallow exceptions
catch (Exception e) {
    log.error("Error", e); // always log before swallowing
    // or rethrow
}

// 5. Don't use exceptions for control flow
// Bad:
try {
    int value = Integer.parseInt(input);
} catch (NumberFormatException e) {
    // this is actually used for normal flow — don't do this
}
// Good:
if (isNumeric(input)) { ... }
```

---

## 9. equals() and hashCode()

**Contract:**
1. If `a.equals(b)` → `a.hashCode() == b.hashCode()`
2. `equals()` must be reflexive, symmetric, transitive, consistent, and `null`-safe
3. If `hashCode()` is not overridden when `equals()` is, objects won't work correctly as HashMap keys

```java
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money m)) return false; // pattern matching
        return amount.compareTo(m.amount) == 0 // use compareTo for BigDecimal
            && currency.equals(m.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount.stripTrailingZeros(), currency);
    }
}
```

**Why `BigDecimal.equals()` is dangerous:** `new BigDecimal("1.0").equals(new BigDecimal("1.00"))` returns `false` because they have different scales. Use `compareTo()` instead.

---

## 10. Common Interview Questions

| Question | Answer |
|----------|--------|
| What is the difference between `==` and `equals()`? | `==` compares object references (memory addresses). `equals()` compares logical content as defined by the class. For `String`, `==` may return false even for identical text because two `new String("a")` calls create separate objects. Always use `equals()` for value comparison; use `==` only when you intentionally check reference identity. |
| What happens to generic type parameters at runtime due to type erasure? | The compiler replaces all generic type parameters with their bounds (usually `Object`) and inserts casts where needed. At runtime, `List<String>` and `List<Integer>` are the same `List` class. Consequences: you cannot create generic arrays (`new T[10]`), cannot use `instanceof` with generic types, and cannot overload methods that differ only in generic type parameter. The compiler generates synthetic bridge methods to preserve polymorphism. |
| Why is `String` immutable in Java? | Four reasons: (1) String pool — identical literals share one object, saving memory; (2) caching — `hashCode()` is computed once and cached, since the value can never change; (3) security — file paths, URLs, and credentials passed as strings cannot be modified by untrusted code; (4) thread safety — an immutable object can be shared across threads without synchronization. |
| What is the contract between `hashCode()` and `equals()`? | If two objects are equal according to `equals()`, they must have the same `hashCode()`. The reverse is not required — two objects with the same hash may or may not be equal (hash collision). Breaking this contract (e.g., overriding `equals()` without `hashCode()`) causes `HashMap` and `HashSet` to malfunction: an object can be put in but never found. |
| What is the difference between an abstract class and an interface? | An abstract class can have state (instance fields), constructors, and both abstract and concrete methods. A class can extend only one abstract class. An interface defines a contract — it can have abstract methods, `default` methods (since Java 8), `static` methods, and constants, but no instance state. A class can implement multiple interfaces. Use abstract class for shared implementation; use interface for shared capability / multiple inheritance of type. |
| What are `default` methods in interfaces and why were they added? | `default` methods provide a method body directly in an interface. They were added in Java 8 to allow evolving existing interfaces (e.g., adding `forEach` to `Collection`) without breaking all implementing classes. If two interfaces define a `default` method with the same signature, the implementing class must explicitly override it to resolve the ambiguity — this is how Java avoids the classic diamond problem. |
| What are the four kinds of method references in Java? | (1) `Class::staticMethod` — reference to a static method; (2) `instance::method` — reference to an instance method of a specific object; (3) `Class::instanceMethod` — reference to an instance method where the first lambda argument is the receiver; (4) `Class::new` — constructor reference. They are syntactic sugar for lambdas and are interchangeable wherever a functional interface is expected. |
| What is autoboxing and what are its pitfalls? | Autoboxing is the automatic conversion between primitives and their wrapper types (`int` ↔ `Integer`). Pitfalls: (1) `==` on `Integer` compares references, not values — only integers from -128 to 127 are cached, so `Integer a = 200; Integer b = 200; a == b` is `false`; (2) unboxing a `null` wrapper causes a `NullPointerException`; (3) excessive autoboxing in loops creates many short-lived objects and increases GC pressure. |
| What is a covariant return type? | An overriding method may declare a more specific (narrower) return type than the method in the parent class. For example, if `Animal clone()` is defined in a base class, a `Dog` subclass can override it as `Dog clone()`. This is valid because a `Dog` is-a `Animal`. It avoids unnecessary casting at call sites and is commonly used in builder patterns and the `clone()` method. |
| What is the difference between `Comparable` and `Comparator`? | `Comparable` defines the natural ordering of a class — the class implements it itself via `compareTo()`. `Comparator` is an external strategy object that defines an ordering separate from the class. Use `Comparable` for the default sort order; use `Comparator` when you need multiple orderings or cannot modify the class. `Collections.sort(list)` uses `Comparable`; `list.sort(comparator)` or `Comparator.comparing(...)` use `Comparator`. |
