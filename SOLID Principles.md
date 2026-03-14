# SOLID Principles in C++ — Explicit Notes

> Each principle answers **one specific fear** in software design.  
> Format: Problem → Why it hurts → Fix → Code → Class flow diagram.

---

## Quick Cheatsheet

| Letter | Principle | One-line Summary |
|--------|-----------|-----------------|
| **S** | Single Responsibility | A class does **one job** — one reason to change |
| **O** | Open/Closed | Open for **extension**, closed for **modification** |
| **L** | Liskov Substitution | Subclass must be **safely replaceable** for its base |
| **I** | Interface Segregation | Don't force a class to implement what it **doesn't need** |
| **D** | Dependency Inversion | Depend on **abstractions**, not concrete classes |

---

## S — Single Responsibility Principle (SRP)

### The Fear It Solves
> "If I change how invoices are printed, why did my database logic break?"

**Rule:** A class should have **only one reason to change**.  
Every time you add a new responsibility (logging, formatting, saving, emailing) to a single class, you create coupling. A bug in one area can break the other.

---

### ❌ Violation — One class doing too many jobs

```cpp
// Invoice is doing 3 jobs: data, formatting, AND persistence
class Invoice {
public:
    double amount;
    std::string customerName;

    // Job 1: Business logic
    double calculateTotal() { return amount * 1.18; }  // add GST

    // Job 2: Formatting/display — DIFFERENT reason to change
    void printInvoice() {
        std::cout << "Customer: " << customerName << "\n";
        std::cout << "Total: "    << calculateTotal() << "\n";
    }

    // Job 3: Persistence — ANOTHER reason to change
    void saveToDatabase() {
        // DB logic here — if DB changes, this class changes
    }
};
```

**Problem:** 3 independent teams or 3 unrelated bugs can all modify `Invoice`.  
Changing DB schema forces re-testing invoice printing. Not acceptable.

---

### ✅ Fix — Split responsibilities into separate classes

```cpp
// Job 1: Pure data + business logic ONLY
struct Invoice {
    std::string customerName;
    double amount;
    double calculateTotal() const { return amount * 1.18; }  // GST
};

// Job 2: Formatting ONLY — changes only if output format changes
class InvoicePrinter {
public:
    void print(const Invoice& inv) {
        std::cout << "Customer: " << inv.customerName << "\n";
        std::cout << "Total:    " << inv.calculateTotal() << "\n";
    }
};

// Job 3: Persistence ONLY — changes only if DB/storage changes
class InvoiceRepository {
public:
    void save(const Invoice& inv) {
        // Save to database
        std::cout << "Saving invoice for: " << inv.customerName << "\n";
    }
};

// Usage — each collaborates, none owns the other's concern
int main() {
    Invoice inv{"John Doe", 1000.0};

    InvoicePrinter printer;
    printer.print(inv);       // Handles display

    InvoiceRepository repo;
    repo.save(inv);           // Handles persistence
}
```

---

### Class Flow

```
[main]
   |
   |--creates--> [Invoice]          (data + GST calc)
   |
   |--uses-----> [InvoicePrinter]   (print only)
   |                  |
   |                  +--reads----> [Invoice]
   |
   |--uses-----> [InvoiceRepository] (save only)
                       |
                       +--reads----> [Invoice]
```

**Key insight:** `Invoice` is a passive data holder. The other two classes *read* it — they don't own it, don't modify its structure. If you switch from SQL to NoSQL, only `InvoiceRepository` changes. If you switch from console to PDF output, only `InvoicePrinter` changes.

---

## O — Open/Closed Principle (OCP)

### The Fear It Solves
> "Every time I add a new shape/payment method/format, I have to edit existing, tested code."

**Rule:** A class should be **open for extension** (add new behavior) but **closed for modification** (don't touch what's already tested).

The mechanism: use **abstraction (virtual functions / interfaces)** — new types extend the abstraction, old code doesn't change.

---

### ❌ Violation — Adding a new type forces editing existing code

```cpp
// PROBLEM: Every new shape requires editing this function
// What if this function is in a shipped library? You can't touch it.
double totalArea(const std::vector<std::string>& shapes,
                 const std::vector<double>& dims) {
    double total = 0;
    for (size_t i = 0; i < shapes.size(); i++) {
        if (shapes[i] == "circle")    total += 3.14 * dims[i] * dims[i];
        if (shapes[i] == "square")   total += dims[i] * dims[i];
        // Adding Triangle? You MUST edit this function → violates OCP
    }
    return total;
}
```

---

### ✅ Fix — Abstract the behavior; new types extend without editing callers

```cpp
// Abstract interface — the "contract"
struct Shape {
    virtual double area() const = 0;  // Each shape computes its own area
    virtual ~Shape() = default;
};

// Existing types — won't be touched when Triangle is added
struct Circle : Shape {
    double radius;
    Circle(double r) : radius(r) {}
    double area() const override { return 3.14159 * radius * radius; }
};

struct Square : Shape {
    double side;
    Square(double s) : side(s) {}
    double area() const override { return side * side; }
};

// NEW TYPE: Added without touching Circle, Square, or totalArea
struct Triangle : Shape {
    double base, height;
    Triangle(double b, double h) : base(b), height(h) {}
    double area() const override { return 0.5 * base * height; }
};

// This function NEVER changes — it's closed for modification
double totalArea(const std::vector<std::unique_ptr<Shape>>& shapes) {
    double total = 0;
    for (const auto& s : shapes) total += s->area();  // Polymorphic call
    return total;
}

// Usage
int main() {
    std::vector<std::unique_ptr<Shape>> shapes;
    shapes.push_back(std::make_unique<Circle>(5.0));
    shapes.push_back(std::make_unique<Square>(4.0));
    shapes.push_back(std::make_unique<Triangle>(3.0, 6.0));  // Just plug in

    std::cout << totalArea(shapes) << "\n";  // Works without editing totalArea
}
```

---

### Class Flow

```
         [Shape]  <--- abstract interface
           /|\
          / | \
         /  |  \
   [Circle] [Square] [Triangle]   <-- extensions (open)
         \   |   /
          \  |  /
       [totalArea()]               <-- closed, never changes
```

**Key insight:** `totalArea()` depends on the *abstraction* `Shape`, not on concrete types. When you add `Triangle`, only `Triangle.cpp` is new code — nothing existing is touched or re-tested.

---

## L — Liskov Substitution Principle (LSP)

### The Fear It Solves
> "I passed a `Square` where a `Rectangle` was expected and my resize broke silently."

**Rule:** If `B` inherits from `A`, you must be able to **replace `A` with `B` everywhere** without breaking behavior.  
The subclass must honor the *contract* (preconditions, postconditions) of the base class.

A subclass that *overrides* behavior in a way that **surprises the caller** violates LSP.

---

### ❌ Violation — Square breaking Rectangle's contract

```cpp
class Rectangle {
protected:
    double w, h;
public:
    virtual void setWidth(double width)   { w = width; }
    virtual void setHeight(double height) { h = height; }
    double area() const { return w * h; }
};

// Square IS-A Rectangle in math — but NOT in behavior here
class Square : public Rectangle {
public:
    // Square forces both sides equal — this BREAKS Rectangle's contract
    void setWidth(double width)  override { w = h = width; }  // sets BOTH
    void setHeight(double h_val) override { w = h = h_val; }  // sets BOTH
};

// Client code that trusts Rectangle's contract
void resizeAndCheck(Rectangle& r) {
    r.setWidth(5);
    r.setHeight(4);
    // Expected: area = 5*4 = 20 (Rectangle contract)
    // Got with Square: area = 4*4 = 16  ← SURPRISE! LSP violated
    assert(r.area() == 20);  // FAILS for Square
}

int main() {
    Rectangle rect;
    resizeAndCheck(rect);   // OK

    Square sq;
    resizeAndCheck(sq);     // CRASH — unexpected behavior
}
```

**Why is this a violation?** The caller set width=5, height=4 and expects area=20. `Square` silently changed width when `setHeight` was called. The *caller had no way to know* — it trusted the base class contract.

---

### ✅ Fix — Don't force IS-A when the contract differs; use composition or a clean hierarchy

```cpp
// Separate abstractions — no forced inheritance
struct Shape {
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

// Rectangle has independent width/height — its own contract
class Rectangle : public Shape {
    double w, h;
public:
    Rectangle(double w, double h) : w(w), h(h) {}
    void setWidth(double width)   { w = width; }
    void setHeight(double height) { h = height; }
    double area() const override  { return w * h; }
};

// Square has one side — its own contract; does NOT inherit Rectangle
class Square : public Shape {
    double side;
public:
    Square(double s) : side(s) {}
    void setSide(double s) { side = s; }
    double area() const override { return side * side; }
};

// Client: now works with Shape — no contract surprise
void printArea(const Shape& s) {
    std::cout << "Area: " << s.area() << "\n";
}

int main() {
    Rectangle r(5, 4);
    Square s(4);
    printArea(r);  // 20
    printArea(s);  // 16 — correct, no surprise
}
```

---

### Class Flow

```
❌ BAD hierarchy:
   [Rectangle] <---- [Square]       (Square breaks Rectangle's contract)

✅ GOOD hierarchy:
        [Shape]
        /     \
[Rectangle]  [Square]               (independent contracts, both substitutable for Shape)
```

**The LSP test:** For every function that accepts a `Base*`, ask: *"If I pass a `Derived*`, does it still behave correctly from the caller's point of view?"*  
If NO — your hierarchy is wrong.

---

## I — Interface Segregation Principle (ISP)

### The Fear It Solves
> "My `Robot` class is forced to implement `eat()` and `sleep()` even though robots don't eat or sleep."

**Rule:** Don't force classes to implement interfaces they don't use.  
Split fat interfaces into small, focused ones. A class only implements what it actually needs.

---

### ❌ Violation — Fat interface forces irrelevant implementations

```cpp
// One big interface — forces ALL implementers to handle ALL methods
struct IWorker {
    virtual void work()  = 0;
    virtual void eat()   = 0;  // Only human workers eat
    virtual void sleep() = 0;  // Only human workers sleep
    virtual ~IWorker() = default;
};

class HumanWorker : public IWorker {
public:
    void work()  override { std::cout << "Human working\n";  }
    void eat()   override { std::cout << "Human eating\n";   }
    void sleep() override { std::cout << "Human sleeping\n"; }
};

// Robot is forced to implement eat() and sleep() — MEANINGLESS
class RobotWorker : public IWorker {
public:
    void work()  override { std::cout << "Robot working\n"; }
    void eat()   override { /* Does nothing — forced stub! */ }  // ← ISP violation
    void sleep() override { /* Does nothing — forced stub! */ }  // ← ISP violation
};
```

**Problem:** `RobotWorker` now has two dead, misleading methods. Any caller that calls `robot.eat()` gets silent no-ops, leading to bugs that are hard to trace. The interface is lying.

---

### ✅ Fix — Split into focused interfaces; each class implements only what it needs

```cpp
// Small, focused interfaces
struct IWorkable {
    virtual void work() = 0;
    virtual ~IWorkable() = default;
};

struct IFeedable {
    virtual void eat() = 0;
    virtual ~IFeedable() = default;
};

struct ISleepable {
    virtual void sleep() = 0;
    virtual ~ISleepable() = default;
};

// Human implements all three — makes sense
class HumanWorker : public IWorkable, public IFeedable, public ISleepable {
public:
    void work()  override { std::cout << "Human working\n";  }
    void eat()   override { std::cout << "Human eating\n";   }
    void sleep() override { std::cout << "Human sleeping\n"; }
};

// Robot only implements what it actually does — clean
class RobotWorker : public IWorkable {
public:
    void work() override { std::cout << "Robot working\n"; }
    // NO eat(), NO sleep() — because robots don't do those things
};

// Callers only depend on what they need
void manageWork(IWorkable& w)  { w.work(); }
void feedWorker(IFeedable& f)  { f.eat();  }

int main() {
    HumanWorker human;
    RobotWorker robot;

    manageWork(human);   // OK
    manageWork(robot);   // OK
    feedWorker(human);   // OK
    // feedWorker(robot) — won't compile! Robot doesn't implement IFeedable.
    //                      Compile-time safety, not silent runtime no-op.
}
```

---

### Class Flow

```
[IWorkable]   [IFeedable]   [ISleepable]
     |              |              |
     |           [HumanWorker] ----+-------+   (implements all 3)
     |
     +-------- [RobotWorker]              (implements only IWorkable)
```

**Key insight:** ISP makes dependencies explicit and compile-time verified. If `RobotWorker` can't be fed, the compiler stops you from calling `feedWorker(robot)` — no silent failures.

---

## D — Dependency Inversion Principle (DIP)

### The Fear It Solves
> "I can't unit test my high-level business logic because it directly creates a `MySQLDatabase` object inside."

**Rule:**
1. **High-level modules** (business logic) should NOT depend on **low-level modules** (DB, file I/O, network).  
2. Both should depend on **abstractions** (interfaces).  
3. Abstractions should NOT depend on details; **details depend on abstractions**.

The mechanism: inject dependencies (via constructor / function parameter), don't create them inside.

---

### ❌ Violation — High-level logic directly depends on a concrete low-level class

```cpp
// Low-level module
class MySQLDatabase {
public:
    void saveOrder(const std::string& order) {
        std::cout << "MySQL: Saving order: " << order << "\n";
    }
};

// High-level module — directly creates and uses MySQLDatabase
// PROBLEM: To switch to PostgreSQL, you must EDIT OrderService.
// PROBLEM: To unit test OrderService, you must spin up a real MySQL DB.
class OrderService {
    MySQLDatabase db;  // ← hardcoded concrete dependency
public:
    void placeOrder(const std::string& item) {
        // business logic...
        db.saveOrder(item);  // tightly coupled
    }
};
```

**Why this is fatal for testing:** You cannot mock `MySQLDatabase` because `OrderService` creates it internally. Unit tests then become integration tests — slow, brittle, requiring real infrastructure.

---

### ✅ Fix — Depend on an abstraction; inject the concrete implementation

```cpp
// STEP 1: Define the abstraction (the interface)
struct IDatabase {
    virtual void saveOrder(const std::string& order) = 0;
    virtual ~IDatabase() = default;
};

// STEP 2: Low-level modules implement the abstraction
class MySQLDatabase : public IDatabase {
public:
    void saveOrder(const std::string& order) override {
        std::cout << "MySQL: Saving: " << order << "\n";
    }
};

class PostgreSQLDatabase : public IDatabase {
public:
    void saveOrder(const std::string& order) override {
        std::cout << "PostgreSQL: Saving: " << order << "\n";
    }
};

// For unit tests — a fast in-memory mock
class MockDatabase : public IDatabase {
public:
    std::vector<std::string> savedOrders;  // Inspectable in tests
    void saveOrder(const std::string& order) override {
        savedOrders.push_back(order);      // No real DB needed
    }
};

// STEP 3: High-level module depends on abstraction ONLY
//         Receives its dependency — never creates it
class OrderService {
    IDatabase& db;  // ← abstraction, not concrete type
public:
    OrderService(IDatabase& database) : db(database) {}  // Injected from outside

    void placeOrder(const std::string& item) {
        // business logic...
        db.saveOrder(item);  // calls through interface — works with ANY IDatabase
    }
};

// Usage — wire up concretions OUTSIDE of OrderService
int main() {
    // Production: use real DB
    MySQLDatabase mysql;
    OrderService  prodService(mysql);
    prodService.placeOrder("Laptop");   // MySQL: Saving: Laptop

    // Swap DB without touching OrderService
    PostgreSQLDatabase pg;
    OrderService pgService(pg);
    pgService.placeOrder("Phone");      // PostgreSQL: Saving: Phone

    // Unit test: use mock — no real DB
    MockDatabase mock;
    OrderService testService(mock);
    testService.placeOrder("Book");
    assert(mock.savedOrders[0] == "Book");  // Verify behavior in-process
}
```

---

### Class Flow

```
            [IDatabase]  <--- abstraction
            /     |     \
           /      |      \
[MySQL]  [PgSQL]  [MockDB]         <--- low-level details implement abstraction
                                        (they depend on IDatabase, not the other way)

[OrderService] ---uses---> [IDatabase]  <--- high-level depends on abstraction
                                              (receives via constructor injection)

[main / composition root]
    - creates MySQL / Mock
    - injects into OrderService
    - owns the wiring
```

**Key insight:** `OrderService` doesn't know what database it's using — and it never should. The wiring (which concrete type to use) happens at the *composition root* (`main`, a DI container, or a factory). High-level logic is now testable, reusable, and swappable.

---

## Putting It All Together — A Real Example

> Scenario: A **report generator** that reads data from a source, formats it, and exports it.  
> Apply all 5 SOLID principles simultaneously.

```cpp
#include <iostream>
#include <string>
#include <vector>
#include <memory>

// (D) Abstractions — high-level does not depend on low-level details
struct IDataSource {
    virtual std::vector<std::string> fetchData() = 0;
    virtual ~IDataSource() = default;
};

struct IExporter {
    virtual void exportReport(const std::string& content) = 0;
    virtual ~IExporter() = default;
};

// (O) Open for extension: add new sources/exporters without touching existing classes

// Low-level: Database source
class DatabaseSource : public IDataSource {
public:
    std::vector<std::string> fetchData() override {
        return {"Row1: Sales=500", "Row2: Sales=800"};  // Simulated DB result
    }
};

// Low-level: CSV source — new type, zero changes elsewhere
class CSVSource : public IDataSource {
public:
    std::vector<std::string> fetchData() override {
        return {"CSV_Row1: Sales=300", "CSV_Row2: Sales=600"};
    }
};

// Low-level: PDF exporter
class PDFExporter : public IExporter {
public:
    void exportReport(const std::string& content) override {
        std::cout << "[PDF] " << content << "\n";
    }
};

// Low-level: HTML exporter — new type, zero changes elsewhere
class HTMLExporter : public IExporter {
public:
    void exportReport(const std::string& content) override {
        std::cout << "<html>" << content << "</html>\n";
    }
};

// (S) Single Responsibility: ReportFormatter ONLY formats — does NOT fetch or export
class ReportFormatter {
public:
    std::string format(const std::vector<std::string>& rows) const {
        std::string report = "=== REPORT ===\n";
        for (const auto& row : rows) report += "  " + row + "\n";
        return report;
    }
};

// (I) Interface Segregation: ReportGenerator depends ONLY on what it needs
//     IDataSource for fetching, IExporter for exporting — not one fat interface
// (D) High-level module — depends on abstractions, injected via constructor
class ReportGenerator {
    IDataSource&     source;      // abstraction only
    IExporter&       exporter;    // abstraction only
    ReportFormatter  formatter;   // (S) formatter has its own SRP class
public:
    ReportGenerator(IDataSource& src, IExporter& exp)
        : source(src), exporter(exp) {}

    void generate() {
        auto rows    = source.fetchData();        // fetch through interface
        auto content = formatter.format(rows);    // format
        exporter.exportReport(content);           // export through interface
    }
};

// Composition root: wiring happens here, not inside business logic
int main() {
    DatabaseSource db;
    PDFExporter    pdf;
    ReportGenerator gen1(db, pdf);
    gen1.generate();
    // [PDF] === REPORT ===
    //   Row1: Sales=500
    //   Row2: Sales=800

    CSVSource   csv;
    HTMLExporter html;
    ReportGenerator gen2(csv, html);
    gen2.generate();
    // <html>=== REPORT ===
    //   CSV_Row1: Sales=300</html>
}
```

### Full Class Flow

```
[main / composition root]
    |
    |--creates--> [DatabaseSource]  implements [IDataSource]
    |--creates--> [PDFExporter]     implements [IExporter]
    |
    |--creates--> [ReportGenerator]
                        |
                        |--depends on--> [IDataSource]  (D: abstraction)
                        |--depends on--> [IExporter]    (D: abstraction)
                        |--owns-------> [ReportFormatter] (S: single job)
                        |
                        |    generate()
                        |        |
                        |        +--calls--> source.fetchData()   → [DatabaseSource]
                        |        +--calls--> formatter.format()   → [ReportFormatter]
                        |        +--calls--> exporter.export()    → [PDFExporter]

   To add a new source/exporter:
   - Create new class implementing [IDataSource] or [IExporter]   (O: open)
   - Wire it in main                                              (D: inversion)
   - ZERO changes to ReportGenerator, ReportFormatter, or others
```

---

## Common Interview Traps & Answers

| Question | Key Answer |
|----------|-----------|
| SRP vs High Cohesion? | SRP is about **reasons to change**, High Cohesion is about **relatedness of members**. SRP leads to high cohesion. |
| OCP vs modifying for bugs? | Bug fixes are exception. OCP targets **feature additions** — you shouldn't need to edit stable, tested code to add new behavior. |
| LSP: is Square a Rectangle? | Mathematically yes, **behaviorally no** if setters exist. LSP is about behavior contracts, not mathematical IS-A. |
| ISP vs one-method interfaces? | Don't split mindlessly. Split when **different clients use different subsets**. One method is fine if that's the natural boundary. |
| DIP vs Dependency Injection? | DI is the *mechanism* (constructor/setter injection). DIP is the *principle* that motivates why you inject: to depend on abstractions. |
| Can all 5 conflict? | Yes. ISP can lead to many tiny interfaces that feel fragmented. Balance with cohesion — group methods that conceptually belong together. |
