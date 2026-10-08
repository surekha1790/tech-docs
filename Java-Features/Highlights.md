# Java 8 → 27 — Useful Features (Interview Notes)

**LTS versions:** 8, 11, 17, 21, 25 (these matter most in interviews).
**Final** = ready for production. **Preview** = needs `--enable-preview`, may still change.

---

## Java 8 (LTS, 2014) — the big one

**Lambdas + Functional Interfaces**
```java
Runnable r = () -> System.out.println("Hi");
Function<Integer, Integer> square = x -> x * x;
Predicate<String> isEmpty = String::isEmpty;   // method reference
```

**Streams**
```java
List<String> names = users.stream()
    .filter(u -> u.age() > 18)
    .map(User::name)
    .sorted()
    .collect(Collectors.toList());
```

**Optional** — avoid null checks
```java
String city = Optional.ofNullable(user)
    .map(User::address)
    .map(Address::city)
    .orElse("Unknown");
```

**Default & static methods in interfaces**
```java
interface Greeter {
    default void greet() { System.out.println("Hello"); }
    static Greeter create() { return new Greeter() {}; }
}
```

**New Date/Time API (`java.time`)** — immutable, thread-safe
```java
LocalDate today = LocalDate.now();
LocalDate next = today.plusDays(7);
Duration d = Duration.ofMinutes(90);
```

**CompletableFuture** — async programming
```java
CompletableFuture.supplyAsync(() -> fetchPrice())
    .thenApply(p -> p * 1.19)
    .thenAccept(System.out::println);
```

---

## Java 9 (2017)

**Collection factory methods** (immutable)
```java
List<String> list = List.of("a", "b");
Map<String, Integer> map = Map.of("x", 1, "y", 2);
```

**Private methods in interfaces**
```java
interface Logger {
    default void info(String m) { log("INFO", m); }
    private void log(String lvl, String m) { System.out.println(lvl + ": " + m); }
}
```

**Stream improvements**
```java
Stream.of(1, 2, 3, 4, 1).takeWhile(n -> n < 3);   // 1, 2
Stream.iterate(1, n -> n < 10, n -> n * 2);       // 1, 2, 4, 8
```

**Optional improvements**
```java
opt.ifPresentOrElse(System.out::println, () -> System.out.println("empty"));
```

**Others:** Module system (JPMS, `module-info.java`), JShell (REPL).

---

## Java 10 (2018)

**`var`** — local type inference
```java
var list = new ArrayList<String>();   // type is ArrayList<String>
```

**Unmodifiable copies**
```java
List<String> copy = List.copyOf(original);
```

---

## Java 11 (LTS, 2018)

**New String methods**
```java
"  ".isBlank();          // true
" hi ".strip();          // "hi" (Unicode-aware trim)
"ab".repeat(3);          // "ababab"
"a\nb".lines().count();  // 2
```

**File helpers**
```java
String text = Files.readString(Path.of("data.txt"));
Files.writeString(Path.of("out.txt"), "Hello");
```

**Standard HttpClient**
```java
HttpClient client = HttpClient.newHttpClient();
HttpRequest req = HttpRequest.newBuilder(URI.create("https://api.example.com")).build();
String body = client.send(req, HttpResponse.BodyHandlers.ofString()).body();
```

**Others:** `var` in lambda params, run a single file with `java Hello.java`.

---

## Java 12 – 13

**`Collectors.teeing`** (12) — two collectors, one result
```java
double avg = nums.stream().collect(Collectors.teeing(
    Collectors.summingInt(i -> i), Collectors.counting(),
    (sum, count) -> (double) sum / count));
```

**`String.indent()` / `transform()`** (12)

---

## Java 14 (2020)

**Switch expressions** (final)
```java
String type = switch (day) {
    case SATURDAY, SUNDAY -> "Weekend";
    default -> "Weekday";
};
```

**Helpful NullPointerExceptions** — tells you *which* variable was null.

---

## Java 15 (2020)

**Text blocks**
```java
String json = """
    {
      "name": "Surekha",
      "role": "Developer"
    }
    """;
```

**Others:** ZGC and Shenandoah GC ready for production.

---

## Java 16 (2021)

**Records** — immutable data classes
```java
record User(String name, int age) { }
User u = new User("Ana", 30);
u.name();   // getter auto-generated, plus equals/hashCode/toString
```

**Pattern matching for `instanceof`**
```java
if (obj instanceof String s) {
    System.out.println(s.length());   // no cast needed
}
```

**`Stream.toList()`**
```java
List<String> result = stream.toList();   // unmodifiable
```

---

## Java 17 (LTS, 2021)

**Sealed classes** — control who can extend
```java
sealed interface Shape permits Circle, Square { }
record Circle(double r) implements Shape { }
record Square(double side) implements Shape { }
```

**Others:** New `RandomGenerator` API, strong encapsulation of JDK internals.

---

## Java 18 – 20

- **18:** UTF-8 is the default charset; simple web server (`jwebserver`).
- **19–20:** Previews of virtual threads, record patterns, structured concurrency.

---

## Java 21 (LTS, 2023) — very important

**Virtual Threads** — millions of lightweight threads
```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> callRemoteService());
}
Thread.startVirtualThread(() -> System.out.println("Hi"));
```

**Pattern matching for switch + Record patterns**
```java
double area = switch (shape) {
    case Circle(double r)   -> Math.PI * r * r;
    case Square(double s)   -> s * s;
};   // no default needed – sealed type is exhaustive
```

**Sequenced Collections**
```java
List<String> list = List.of("a", "b", "c");
list.getFirst();   // "a"
list.getLast();    // "c"
list.reversed();
```

**Others:** Generational ZGC.

---

## Java 22 (2024)

**Unnamed variables `_`**
```java
try { ... } catch (Exception _) { log("failed"); }
map.forEach((_, value) -> System.out.println(value));
```

**Others:** Foreign Function & Memory API (final) — call C code safely, replaces JNI.

---

## Java 23 (2024)

**Markdown in Javadoc**
```java
/// Returns the **sum** of `a` and `b`.
int add(int a, int b) { return a + b; }
```

---

## Java 24 (2025)

**Stream Gatherers** — custom intermediate operations
```java
Stream.of(1, 2, 3, 4, 5)
    .gather(Gatherers.windowFixed(2))
    .toList();   // [[1, 2], [3, 4], [5]]
```

**Others:**
- Virtual threads no longer "pin" on `synchronized` blocks — big fix.
- Class-File API (final).
- Post-quantum crypto: ML-KEM and ML-DSA.

---

## Java 25 (LTS, 2025) — latest LTS

**Compact source files & instance `main`**
```java
void main() {
    IO.println("Hello, Java 25!");
}
```

**Flexible constructor bodies** — code before `super()`
```java
class Employee extends Person {
    Employee(int age) {
        if (age < 18) throw new IllegalArgumentException();  // before super()
        super(age);
    }
}
```

**Module import declarations**
```java
import module java.base;   // imports all packages of the module
```

**Scoped Values** (final) — safer alternative to `ThreadLocal`, works well with virtual threads
```java
static final ScopedValue<String> USER = ScopedValue.newInstance();
ScopedValue.where(USER, "admin").run(() -> handleRequest());
```

**Others:** Compact object headers (smaller memory footprint), Key Derivation Function API.

---

## Java 26 (March 2026)

- **HTTP/3 support** in `HttpClient`.
- **"Prepare to make final mean final"** — warnings when reflection changes `final` fields.
- **Applet API removed.**
- AOT object caching works with any GC (faster startup).
- Previews continue: Lazy Constants, Structured Concurrency, primitive patterns, PEM encodings.

```java
HttpClient client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_3)
    .build();
```

---

## Java 27 (September 2026) — latest release

A "quiet" release: 9 JEPs, mostly runtime and security, no big language changes.

**Final:**
- **G1 is the default GC in all environments** (JEP 523).
- **Compact Object Headers on by default** (JEP 534) — less memory, no code change.
- **Post-Quantum Hybrid Key Exchange for TLS 1.3** (JEP 527).
- **JFR in-process data redaction** (JEP 536) — hides secrets in Flight Recorder data.

**Still preview / incubator:**
- Lazy Constants (3rd preview)
- Structured Concurrency (7th preview)
- Primitive types in patterns, `instanceof`, `switch` (5th preview)
- PEM encodings (3rd preview)
- Vector API (12th incubator)

**Preview examples:**
```java
// Structured Concurrency – treat related tasks as one unit
try (var scope = StructuredTaskScope.open()) {
    var user  = scope.fork(() -> findUser());
    var order = scope.fork(() -> findOrder());
    scope.join();
    return new Response(user.get(), order.get());
}

// Primitive patterns in switch
String size = switch (count) {
    case 0 -> "none";
    case int n when n < 10 -> "small";
    case int n -> "large";
};
```

**Coming in Java 28 (March 2027):** Value Objects (Project Valhalla, preview), Simple JSON API (incubator).

---

## Top Interview Questions

1. **Why upgrade from 8 to 21/25?** → Records, sealed classes, pattern matching, virtual threads, better GC and performance.
2. **Virtual vs platform threads?** → Virtual threads are cheap, managed by the JVM; great for blocking I/O. Not faster for CPU-heavy work.
3. **Record vs Lombok `@Data`?** → Records are immutable and built into the language; no setters.
4. **`map` vs `flatMap`?** → `map` = 1-to-1; `flatMap` = 1-to-many, then flattens.
5. **`Optional` misuse?** → Don't use it for fields or method parameters; use it for return values.
6. **Sealed classes benefit?** → Exhaustive `switch` without `default`, closed hierarchies.
7. **ScopedValue vs ThreadLocal?** → ScopedValue is immutable, bounded in scope, cheaper with virtual threads.

---

## 30-Second Summary

| Version | Remember it for |
|---|---|
| 8 | Lambdas, Streams, Optional, java.time |
| 9 | `List.of`, modules |
| 10 | `var` |
| 11 | String methods, HttpClient |
| 14 | Switch expressions |
| 15 | Text blocks |
| 16 | Records, `instanceof` patterns |
| 17 | Sealed classes |
| 21 | Virtual threads, switch patterns, sequenced collections |
| 22 | Unnamed `_`, FFM API |
| 24 | Stream Gatherers, no pinning |
| 25 | Simple `main`, flexible constructors, Scoped Values |
| 26 | HTTP/3, Applets removed |
| 27 | G1 default everywhere, compact headers default, post-quantum TLS |

Java 8
 ├── Streams
 ├── Functional interfaces
 ├── Optional
 └── CompletableFuture

Java 11
 ├── HTTP Client
 └── important API improvements

Java 14-17
 ├── Switch expressions
 ├── Records
 ├── Pattern matching instanceof
 └── Sealed classes

Java 21
 ├── Virtual Threads ⭐⭐⭐⭐⭐
 ├── Pattern matching switch ⭐⭐⭐⭐⭐
 ├── Record patterns
 └── Sequenced collections

Java 25
 ├── Scoped Values ⭐⭐⭐⭐⭐
 ├── Structured Concurrency ⭐⭐⭐⭐⭐
 ├── JFR improvements ⭐⭐⭐⭐
 ├── Compact Object Headers ⭐⭐⭐
 └── Generational Shenandoah ⭐⭐⭐
