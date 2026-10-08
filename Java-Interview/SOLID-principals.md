# SOLID Principles — Interview Notes

SOLID = 5 design principles for writing code that is **easy to change, test, and extend**.

| Letter | Principle | One-line meaning |
|---|---|---|
| **S** | Single Responsibility | A class should have only one reason to change |
| **O** | Open/Closed | Open for extension, closed for modification |
| **L** | Liskov Substitution | Subclasses must work wherever the parent works |
| **I** | Interface Segregation | Many small interfaces are better than one big one |
| **D** | Dependency Inversion | Depend on abstractions, not concrete classes |

---

## S — Single Responsibility Principle (SRP)

**Idea:** One class = one job. If a class changes for many different reasons, split it.

❌ **Bad**
```java
class OrderService {
    void placeOrder(Order o) { /* business logic */ }
    void saveToDb(Order o)   { /* SQL */ }
    void sendEmail(Order o)  { /* email */ }
}
```

✅ **Good**
```java
class OrderService     { void placeOrder(Order o) { ... } }
class OrderRepository  { void save(Order o) { ... } }
class EmailNotifier    { void sendConfirmation(Order o) { ... } }
```

**Interview line:** "SRP keeps classes small and focused, so a change in email logic doesn't risk breaking order logic."

---

## O — Open/Closed Principle (OCP)

**Idea:** Add new behaviour by adding new code, not by editing existing, tested code.

❌ **Bad** — every new payment type means editing this method
```java
class PaymentProcessor {
    void pay(String type) {
        if (type.equals("CARD")) { ... }
        else if (type.equals("UPI")) { ... }
    }
}
```

✅ **Good** — new payment = new class
```java
interface PaymentMethod { void pay(double amount); }

class CardPayment implements PaymentMethod { public void pay(double a) { ... } }
class UpiPayment  implements PaymentMethod { public void pay(double a) { ... } }

class PaymentProcessor {
    void process(PaymentMethod method, double amount) { method.pay(amount); }
}
```

**Interview line:** "I use interfaces / Strategy pattern so new features are plug-ins, not edits."

---

## L — Liskov Substitution Principle (LSP)

**Idea:** If `B extends A`, you should be able to use `B` anywhere `A` is used **without surprises**.

❌ **Bad** — classic Rectangle/Square problem
```java
class Rectangle {
    void setWidth(int w)  { ... }
    void setHeight(int h) { ... }
}
class Square extends Rectangle {
    void setWidth(int w) { /* also changes height! */ }
}
// Code expecting a Rectangle breaks when given a Square
```

Another common smell:
```java
class Bird { void fly() { ... } }
class Penguin extends Bird {
    void fly() { throw new UnsupportedOperationException(); } // ❌ breaks LSP
}
```

✅ **Good** — model behaviour correctly
```java
interface Bird { }
interface FlyingBird extends Bird { void fly(); }

class Sparrow implements FlyingBird { public void fly() { ... } }
class Penguin implements Bird { }
```

**Signs of LSP violation:** throwing `UnsupportedOperationException`, `instanceof` checks, overridden methods that do nothing.

---

## I — Interface Segregation Principle (ISP)

**Idea:** Don't force a class to implement methods it doesn't need.

❌ **Bad**
```java
interface Worker {
    void work();
    void eat();
}
class Robot implements Worker {
    public void work() { ... }
    public void eat()  { /* robots don't eat! */ }
}
```

✅ **Good**
```java
interface Workable { void work(); }
interface Eatable  { void eat(); }

class Human implements Workable, Eatable { ... }
class Robot implements Workable { ... }
```

**Interview line:** "Small, role-based interfaces keep implementations clean and avoid empty methods."

---

## D — Dependency Inversion Principle (DIP)

**Idea:** High-level code should not depend on low-level details. Both depend on an **interface**.

❌ **Bad** — tightly coupled to MySQL
```java
class UserService {
    private MySqlUserRepository repo = new MySqlUserRepository();
}
```

✅ **Good** — depends on abstraction, injected from outside
```java
interface UserRepository { User findById(Long id); }

class PostgresUserRepository implements UserRepository { ... }

class UserService {
    private final UserRepository repo;
    UserService(UserRepository repo) { this.repo = repo; } // constructor injection
}
```

**Spring connection:** Spring's Dependency Injection (`@Autowired`, constructor injection) is DIP in practice. Easy to swap implementations and mock in unit tests.

> **DIP vs DI:** DIP is the *principle* (depend on abstractions). DI is the *technique* (pass dependencies in from outside).

---

## Quick Real-World Mapping (Spring Boot)

| Principle | Where you see it |
|---|---|
| SRP | Controller → Service → Repository layers |
| OCP | Strategy pattern, Spring beans implementing a common interface |
| LSP | Proper inheritance; avoiding `UnsupportedOperationException` |
| ISP | Small repository / port interfaces instead of a "god" interface |
| DIP | Constructor injection of interfaces; mocking in tests |

---

## Common Interview Questions

1. **Why SOLID?** → Lower coupling, higher cohesion, easier testing, safer changes.
2. **Which principle is most important?** → Often SRP — it drives the others. (Any reasoned answer is fine.)
3. **Can SOLID be overused?** → Yes. Too many tiny classes/interfaces = over-engineering. Apply when change is likely.
4. **Relation to design patterns?** → Strategy, Factory, Decorator, Adapter all help achieve OCP/DIP.
5. **Give a real example from your project.** → Prepare one per principle from your own work.

---

## 30-Second Summary

- **S** — One class, one job.
- **O** — Extend, don't edit.
- **L** — Child must behave like parent.
- **I** — Small focused interfaces.
- **D** — Depend on interfaces, inject dependencies.
