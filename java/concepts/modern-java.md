# Modern Java (8 → 21) — Features That Appear in Interviews

**Topic:** [[java/topics/core-language]]
**Related:** [[java/concepts/collections-deep-dive]], [[java/concepts/generics]]

## Java 8 Features

### Lambdas and Functional Interfaces

```java
// Functional interface: exactly one abstract method
@FunctionalInterface
interface Transformer<T, R> {
    R transform(T input);
}

// Lambda = anonymous implementation of a functional interface
Transformer<String, Integer> length = s -> s.length();

// Built-in functional interfaces (java.util.function)
Function<T, R>       // T → R
BiFunction<T, U, R>  // T, U → R
Predicate<T>         // T → boolean
Consumer<T>          // T → void
Supplier<T>          // () → T
UnaryOperator<T>     // T → T (extends Function<T,T>)
BinaryOperator<T>    // (T, T) → T
```

### Streams API

```java
List<Employee> employees = ...;

// The three parts: source → intermediate ops (lazy) → terminal op (triggers evaluation)
double avgSalary = employees.stream()             // source
    .filter(e -> e.getDept().equals("Engineering"))  // intermediate: lazy
    .mapToDouble(Employee::getSalary)                // intermediate: lazy
    .average()                                       // terminal: triggers everything
    .orElse(0.0);

// Common intermediate operations
.filter(predicate)           // keep matching elements
.map(function)               // transform elements
.flatMap(function)           // transform each element to stream, flatten result
.distinct()                  // remove duplicates (uses equals/hashCode)
.sorted()                    // sort (natural or comparator)
.limit(n)                    // take first n
.skip(n)                     // skip first n
.peek(consumer)              // inspect without consuming (debugging)

// Terminal operations
.collect(Collectors.toList())    // collect to list
.collect(Collectors.toMap(...))  // collect to map
.count()                         // count elements
.anyMatch(predicate)             // short-circuit: true if any matches
.allMatch(predicate)             // short-circuit: false if any doesn't match
.findFirst()                     // first element (Optional)
.reduce(identity, accumulator)   // fold
.forEach(consumer)               // consume each
```

### Advanced Collectors

```java
// groupingBy — the most-asked collector
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept));

// groupingBy with downstream collector
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept, Collectors.counting()));

Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept,
             Collectors.averagingDouble(Employee::getSalary)));

// partitioningBy — split into true/false
Map<Boolean, List<Employee>> seniorJunior = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getYearsExp() >= 5));

// joining
String names = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.joining(", ", "[", "]"));  // delimiter, prefix, suffix

// toUnmodifiableMap
Map<String, Integer> nameToId = employees.stream()
    .collect(Collectors.toUnmodifiableMap(Employee::getName, Employee::getId));

// collectingAndThen — wrap result
List<String> unmodifiable = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList));
```

**Interview trap:** Streams are lazy — intermediate operations are NOT executed until a terminal operation is called. This means: `stream().filter().map()` with no terminal op does nothing.

### Optional

```java
// Creation
Optional<String> opt = Optional.of("value");        // throws NPE if null
Optional<String> opt = Optional.ofNullable(value);  // empty if null
Optional<String> opt = Optional.empty();

// Using Optional safely (never use get() without isPresent())
String result = opt.orElse("default");               // default value
String result = opt.orElseGet(() -> compute());      // lazy default
String result = opt.orElseThrow(NoSuchElementException::new);

opt.ifPresent(s -> System.out.println(s));           // consume if present
opt.ifPresentOrElse(s -> use(s), () -> handleEmpty()); // Java 9

Optional<String> upper = opt.map(String::toUpperCase);  // transform if present
Optional<String> filtered = opt.filter(s -> s.length() > 3);

// Chaining Optionals (flatMap)
Optional<String> streetName = Optional.ofNullable(user)
    .map(User::getAddress)
    .map(Address::getStreet)
    .filter(s -> !s.isEmpty());
```

**Interview rule:** `Optional` is for return types — never use as a field type, method parameter, or collection element. Its purpose is to signal "this method may have no result."

---

## Java 9–11 Features

### `var` (Java 10 — Local Variable Type Inference)
```java
var list = new ArrayList<String>();  // inferred as ArrayList<String>
var map = Map.of("a", 1, "b", 2);   // inferred as Map<String, Integer>

// Cannot use: fields, method params, return types, lambda params (mostly)
// Cannot use: when type is not obvious from context
var result = someMethod();  // OK but makes code harder to read
```

### Factory Methods for Collections (Java 9)
```java
List<String> list = List.of("a", "b", "c");           // immutable
Set<Integer> set = Set.of(1, 2, 3);                   // immutable, no nulls
Map<String, Integer> map = Map.of("a", 1, "b", 2);    // immutable

// Mutable copy:
List<String> mutable = new ArrayList<>(List.of("a", "b", "c"));
```

### String Methods (Java 11)
```java
" hello ".strip();          // trim using Unicode whitespace (better than trim())
"".isBlank();               // true if empty or only whitespace
"a\nb\nc".lines().toList(); // Stream<String> of lines
"na".repeat(3);             // "nanana"
```

---

## Java 14–16 Features

### Records (Java 16)
Records are **immutable data carriers**. The compiler auto-generates: constructor, getters, `equals()`, `hashCode()`, `toString()`.

```java
// Declaration
record Point(double x, double y) {}

// Usage
Point p = new Point(1.0, 2.0);
p.x();         // accessor (not getX())
p.y();

// Compact constructor (validation)
record Range(int min, int max) {
    Range {  // compact constructor — parameters already set
        if (min > max) throw new IllegalArgumentException("min > max");
    }
}

// Records can:
// - implement interfaces
// - have static fields and methods
// - have additional instance methods (but no additional instance fields)
// Records CANNOT:
// - extend classes (implicitly extend Record)
// - have mutable fields (all fields are final)
// - be extended by other classes (implicitly final)
```

**Interview angle:** Records vs Lombok `@Value`. Records are a language feature (no dependency), more limited (no builder pattern), but cleaner for pure data carriers. Use records for DTOs, value objects, and return types from multi-value methods.

---

## Java 17 Features

### Sealed Classes
Sealed classes restrict which classes can extend or implement them. Enables **exhaustive pattern matching**.

```java
// Sealed class hierarchy
sealed interface Shape permits Circle, Rectangle, Triangle {}

record Circle(double radius) implements Shape {}
record Rectangle(double width, double height) implements Shape {}
record Triangle(double base, double height) implements Shape {}

// The compiler knows all subtypes → exhaustive switch
double area(Shape shape) {
    return switch (shape) {
        case Circle c    -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Triangle t  -> 0.5 * t.base() * t.height();
        // No default needed — compiler verifies exhaustiveness
    };
}
```

**Interview angle:** Sealed classes enable the **algebraic data type** pattern from functional languages. They're perfect for modeling sum types (e.g., `Result = Success | Failure`, `Event = LoginEvent | LogoutEvent | PurchaseEvent`).

### Pattern Matching for `instanceof`
```java
// Old (Java < 16)
if (obj instanceof String) {
    String s = (String) obj;  // redundant cast
    System.out.println(s.length());
}

// New (Java 16+)
if (obj instanceof String s) {
    System.out.println(s.length());  // s is scoped to the if block
}

// In switch (Java 21 — standard):
String describe(Object obj) {
    return switch (obj) {
        case Integer i -> "int: " + i;
        case String s  -> "string: " + s.length();
        case null      -> "null";
        default        -> "other";
    };
}
```

---

## Java 21 Features

### Sequenced Collections
New interfaces: `SequencedCollection`, `SequencedSet`, `SequencedMap`. Adds `getFirst()`, `getLast()`, `addFirst()`, `addLast()`, `reversed()` to the appropriate collections.

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));
list.getFirst();   // "a" — no more list.get(0)
list.getLast();    // "c" — no more list.get(list.size()-1)
list.reversed();   // reversed view (not a copy)
```

### String Templates (Preview)
```java
String name = "Alice";
int age = 30;
String msg = STR."Hello, \{name}. You are \{age} years old.";
// "Hello, Alice. You are 30 years old."
```

### Virtual Threads (Standard) — see `[[java/concepts/concurrency-advanced]]`

---

## Effective Java Patterns (Bloch)

### Prefer Composition Over Inheritance
```java
// Fragile: InstrumentedSet extends HashSet (increments may double-count)
// Robust: wrapper/decorator pattern
class InstrumentedSet<E> implements Set<E> {
    private final Set<E> s;  // delegate all Set operations
    private int addCount = 0;

    public boolean add(E e) {
        addCount++;
        return s.add(e);
    }
    public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return s.addAll(c);   // doesn't call our add() → correct count!
    }
    // ... delegate rest to s
}
```

### Item 17: Minimize Mutability
- Declare all fields `final`
- Make the class `final` or use static factory methods
- Never share mutable objects (defensive copies in constructor and accessors)
- For mutable state: `CopyOnWrite`, `volatile`, or `synchronized`

```java
// Immutable class template
public final class Money {
    private final long cents;  // internal representation
    private final Currency currency;

    public Money(long cents, Currency currency) {
        this.cents = cents;
        this.currency = Objects.requireNonNull(currency);
    }

    // Returns new object instead of mutating
    public Money plus(Money other) {
        if (!currency.equals(other.currency)) throw new IllegalArgumentException();
        return new Money(cents + other.cents, currency);
    }
}
```

---

## Interview Questions

**Q: What is the difference between `map()` and `flatMap()` in streams?**
`map()` transforms each element to one value — one-to-one. `flatMap()` transforms each element to a stream of values and flattens the result — one-to-many. Example: `sentences.stream().flatMap(s -> Arrays.stream(s.split(" ")))` produces a stream of all words from all sentences.

**Q: What are sealed classes and why are they useful?**
Sealed classes restrict which classes can extend them (declared with `permits`). This enables exhaustive switch expressions — the compiler knows all possible subtypes and verifies coverage without a `default`. They model sum types: `Result = Ok | Err`, `Shape = Circle | Rectangle`. Without sealed classes, you can't safely switch on a type hierarchy.

**Q: When would you use a record vs a regular class?**
Records are best for pure data carriers with no mutable state, no custom business logic, and no inheritance requirements. They auto-generate equals/hashCode/toString correctly. Use a regular class when you need: mutable fields, custom constructor logic beyond validation, extension points, or Builder pattern.

**Q: What is the difference between `Optional.orElse()` and `Optional.orElseGet()`?**
`orElse(value)` always evaluates the default value expression — even if the Optional is non-empty. `orElseGet(supplier)` only evaluates the supplier lambda when the Optional is empty (lazy). Use `orElseGet` when the default value is expensive to compute.

## Sources
- [[java/concepts/generics]]
- [[java/concepts/collections-deep-dive]]
