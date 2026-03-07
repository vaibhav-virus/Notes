# Design Patterns in C++ — Quick Reference

> Short, unambiguous examples. Each pattern under 40 lines.

---

## Creational Patterns

### 1. Singleton — One instance, globally accessible

```cpp
class Logger {
    Logger() {}                                    // Private constructor prevents direct creation
    Logger(const Logger&) = delete;                // Prevent copy
    Logger& operator=(const Logger&) = delete;     // Prevent assignment

    static Logger* instance;
    static std::mutex mutex_;

public:
    static Logger& getInstance() {
        std::lock_guard<std::mutex> lock(mutex_);  // Thread-safe in C++11
        if (!instance) instance = new Logger();
        return *instance;
    }

    void log(const std::string& msg) { std::cout << msg << "\n"; }
};

// Usage
Logger::getInstance().log("Hello");   // Access globally
Logger::getInstance().log("World");   // Same instance every time
```

---

### 2. Factory Method — Subclass decides which object to create

```cpp
// Product interface
struct Shape { virtual void draw() = 0; virtual ~Shape() = default; };
struct Circle  : Shape { void draw() override { std::cout << "Circle\n";  } };
struct Square  : Shape { void draw() override { std::cout << "Square\n";  } };

// Factory — one function decides type at runtime
std::unique_ptr<Shape> createShape(const std::string& type) {
    if (type == "circle") return std::make_unique<Circle>();
    if (type == "square") return std::make_unique<Square>();
    return nullptr;
}

// Usage: caller doesn't know concrete type
auto s = createShape("circle");
s->draw();   // Circle
```

---

### 3. Abstract Factory — Factory of related factories

```cpp
// Families of products
struct Button { virtual void click() = 0; virtual ~Button() = default; };
struct Checkbox { virtual void check() = 0; virtual ~Checkbox() = default; };

struct WinButton  : Button  { void click() override { std::cout << "Win click\n";  } };
struct WinCheckbox: Checkbox{ void check() override { std::cout << "Win check\n";  } };
struct MacButton  : Button  { void click() override { std::cout << "Mac click\n";  } };
struct MacCheckbox: Checkbox{ void check() override { std::cout << "Mac check\n";  } };

// Abstract factory interface
struct UIFactory {
    virtual std::unique_ptr<Button>   makeButton()   = 0;
    virtual std::unique_ptr<Checkbox> makeCheckbox() = 0;
    virtual ~UIFactory() = default;
};

struct WinFactory : UIFactory {
    std::unique_ptr<Button>   makeButton()   override { return std::make_unique<WinButton>();   }
    std::unique_ptr<Checkbox> makeCheckbox() override { return std::make_unique<WinCheckbox>(); }
};

// Usage
std::unique_ptr<UIFactory> factory = std::make_unique<WinFactory>();
auto btn = factory->makeButton();   // Win button — caller doesn't know concrete type
btn->click();
```

---

### 4. Builder — Step-by-step object construction

```cpp
struct Car {
    std::string engine, color;
    int wheels = 4;
    void show() { std::cout << engine << " | " << color << " | " << wheels << " wheels\n"; }
};

class CarBuilder {
    Car car;
public:
    CarBuilder& setEngine(std::string e) { car.engine = e; return *this; }  // Return *this = fluent API
    CarBuilder& setColor(std::string c)  { car.color  = c; return *this; }
    CarBuilder& setWheels(int w)         { car.wheels = w; return *this; }
    Car build() { return car; }
};

// Usage — readable chained construction
Car myCar = CarBuilder()
                .setEngine("V8")
                .setColor("Red")
                .setWheels(4)
                .build();
myCar.show();  // V8 | Red | 4 wheels
```

---

### 5. Prototype — Clone an existing object

```cpp
struct Config {
    std::string host;
    int port;
    virtual std::unique_ptr<Config> clone() const {  // Each class knows how to copy itself
        return std::make_unique<Config>(*this);
    }
};

// Usage: create variations of a base config cheaply
Config base{"localhost", 8080};
auto configA = base.clone();  // Copy
configA->port = 9090;         // Modify clone — base unchanged
```

---

## Structural Patterns

### 6. Adapter — Make incompatible interfaces work together

```cpp
// Old interface (third-party, can't change)
class OldPrinter {
public:
    void printText(const char* text) { printf("%s", text); }
};

// New interface (what our code expects)
struct Printer {
    virtual void print(const std::string& text) = 0;
    virtual ~Printer() = default;
};

// Adapter: wraps OldPrinter, exposes Printer interface
class PrinterAdapter : public Printer {
    OldPrinter old;
public:
    void print(const std::string& text) override {
        old.printText(text.c_str());  // Translate new call to old call
    }
};

// Usage: client only knows Printer interface
std::unique_ptr<Printer> p = std::make_unique<PrinterAdapter>();
p->print("hello");  // Works!
```

---

### 7. Decorator — Add behavior without changing the original class

```cpp
struct Coffee {
    virtual std::string getDescription() = 0;
    virtual double getCost() = 0;
    virtual ~Coffee() = default;
};

struct SimpleCoffee : Coffee {
    std::string getDescription() override { return "Coffee"; }
    double getCost() override { return 1.0; }
};

// Decorator base: wraps a Coffee, adds to it
struct CoffeeDecorator : Coffee {
    std::unique_ptr<Coffee> wrapped;
    CoffeeDecorator(std::unique_ptr<Coffee> c) : wrapped(std::move(c)) {}
};

struct MilkDecorator : CoffeeDecorator {
    MilkDecorator(std::unique_ptr<Coffee> c) : CoffeeDecorator(std::move(c)) {}
    std::string getDescription() override { return wrapped->getDescription() + " + Milk"; }
    double getCost() override { return wrapped->getCost() + 0.5; }
};

// Usage: stack decorators at runtime
auto coffee = std::make_unique<SimpleCoffee>();
auto withMilk = std::make_unique<MilkDecorator>(std::move(coffee));
std::cout << withMilk->getDescription(); // Coffee + Milk
std::cout << withMilk->getCost();        // 1.5
```

---

### 8. Facade — Simplified interface over complex subsystem

```cpp
class VideoDecoder  { public: void decode()  { /* complex */ } };
class AudioDecoder  { public: void decode()  { /* complex */ } };
class Renderer      { public: void render()  { /* complex */ } };

// Facade: one simple function hides all complexity
class VideoPlayer {
    VideoDecoder vd;
    AudioDecoder ad;
    Renderer     r;
public:
    void play(const std::string& file) {  // Simple API for client
        vd.decode();
        ad.decode();
        r.render();
    }
};

// Usage: client only interacts with VideoPlayer
VideoPlayer player;
player.play("movie.mp4");   // No knowledge of decoders/renderer needed
```

---

### 9. Composite — Treat individual objects and groups uniformly

```cpp
struct FileSystemItem {
    virtual void show(int depth = 0) = 0;
    virtual ~FileSystemItem() = default;
};

struct File : FileSystemItem {
    std::string name;
    File(std::string n) : name(n) {}
    void show(int depth) override { std::cout << std::string(depth*2,' ') << name << "\n"; }
};

struct Folder : FileSystemItem {
    std::string name;
    std::vector<std::unique_ptr<FileSystemItem>> children;
    Folder(std::string n) : name(n) {}
    void add(std::unique_ptr<FileSystemItem> item) { children.push_back(std::move(item)); }
    void show(int depth) override {
        std::cout << std::string(depth*2,' ') << "[" << name << "]\n";
        for (auto& c : children) c->show(depth + 1);  // Recursive — uniform treatment
    }
};

// Usage
auto root = std::make_unique<Folder>("root");
root->add(std::make_unique<File>("readme.txt"));
auto src = std::make_unique<Folder>("src");
src->add(std::make_unique<File>("main.cpp"));
root->add(std::move(src));
root->show();
// [root]
//   readme.txt
//   [src]
//     main.cpp
```

---

## Behavioral Patterns

### 10. Observer — Notify subscribers on events

```cpp
struct Observer {
    virtual void update(const std::string& event) = 0;
    virtual ~Observer() = default;
};

class EventSystem {
    std::vector<Observer*> observers;
public:
    void subscribe(Observer* o)   { observers.push_back(o); }
    void unsubscribe(Observer* o) { observers.erase(std::remove(observers.begin(), observers.end(), o), observers.end()); }
    void notify(const std::string& event) {
        for (auto* o : observers) o->update(event);  // Broadcast to all
    }
    void fireEvent() { notify("OnClick"); }
};

struct Logger : Observer {
    void update(const std::string& e) override { std::cout << "LOG: " << e << "\n"; }
};

struct Analytics : Observer {
    void update(const std::string& e) override { std::cout << "ANALYTICS: " << e << "\n"; }
};

// Usage
EventSystem es;
Logger      log;
Analytics   ana;
es.subscribe(&log);
es.subscribe(&ana);
es.fireEvent();   // Both Logger and Analytics notified
```

---

### 11. Strategy — Swap algorithms at runtime

```cpp
struct SortStrategy {
    virtual void sort(std::vector<int>& v) = 0;
    virtual ~SortStrategy() = default;
};

struct BubbleSort : SortStrategy {
    void sort(std::vector<int>& v) override {
        for (size_t i = 0; i < v.size(); i++)
            for (size_t j = 0; j < v.size()-1; j++)
                if (v[j] > v[j+1]) std::swap(v[j], v[j+1]);
    }
};

struct STLSort : SortStrategy {
    void sort(std::vector<int>& v) override { std::sort(v.begin(), v.end()); }
};

class Sorter {
    std::unique_ptr<SortStrategy> strategy;
public:
    Sorter(std::unique_ptr<SortStrategy> s) : strategy(std::move(s)) {}
    void setStrategy(std::unique_ptr<SortStrategy> s) { strategy = std::move(s); }
    void sort(std::vector<int>& v) { strategy->sort(v); }  // Delegates to strategy
};

// Usage: swap algorithm without changing Sorter
Sorter sorter(std::make_unique<STLSort>());
std::vector<int> data{3,1,4,1,5};
sorter.sort(data);
sorter.setStrategy(std::make_unique<BubbleSort>()); // Switch at runtime
sorter.sort(data);
```

---

### 12. Command — Encapsulate a request as an object (undo support)

```cpp
struct Command {
    virtual void execute() = 0;
    virtual void undo()    = 0;
    virtual ~Command() = default;
};

class TextEditor {
    std::string text;
public:
    void addText(const std::string& t) { text += t; }
    void removeText(size_t n) { if (n <= text.size()) text.resize(text.size() - n); }
    const std::string& getText() const { return text; }
};

class TypeCommand : public Command {
    TextEditor& editor;
    std::string typed;
public:
    TypeCommand(TextEditor& e, std::string t) : editor(e), typed(t) {}
    void execute() override { editor.addText(typed); }
    void undo()    override { editor.removeText(typed.size()); }  // Undo = remove what was typed
};

// Usage
TextEditor editor;
std::stack<std::unique_ptr<Command>> history;

auto cmd = std::make_unique<TypeCommand>(editor, "Hello");
cmd->execute();
history.push(std::move(cmd));
// editor.getText() = "Hello"

history.top()->undo();
history.pop();
// editor.getText() = ""
```

---

### 13. Template Method — Define skeleton, subclasses fill in steps

```cpp
class DataMiner {
public:
    void mine() {           // Template method — defines the algorithm skeleton
        openFile();
        extractData();       // Step varies per subclass
        parseData();         // Step varies per subclass
        closeFile();
    }
private:
    void openFile()  { std::cout << "Opening file\n";  }
    void closeFile() { std::cout << "Closing file\n"; }
    virtual void extractData() = 0;
    virtual void parseData()   = 0;
};

class CSVMiner : public DataMiner {
    void extractData() override { std::cout << "Read CSV rows\n";   }
    void parseData()   override { std::cout << "Parse CSV tokens\n"; }
};

class JSONMiner : public DataMiner {
    void extractData() override { std::cout << "Read JSON nodes\n";  }
    void parseData()   override { std::cout << "Parse JSON fields\n"; }
};

// Usage
CSVMiner csv;
csv.mine();  // Opening file → Read CSV rows → Parse CSV tokens → Closing file
```

---

### 14. CRTP (Curiously Recurring Template Pattern) — Static Polymorphism

```cpp
// No virtual → no vtable overhead → compile-time polymorphism
template<typename Derived>
class Animal {
public:
    void speak() {
        static_cast<Derived*>(this)->speakImpl(); // Dispatch at compile time
    }
};

class Dog : public Animal<Dog> {
public:
    void speakImpl() { std::cout << "Woof!\n"; }
};

class Cat : public Animal<Cat> {
public:
    void speakImpl() { std::cout << "Meow!\n"; }
};

template<typename T>
void makeItSpeak(Animal<T>& animal) { animal.speak(); }

Dog d; makeItSpeak(d);  // Woof! — resolved at compile time
Cat c; makeItSpeak(c);  // Meow!
```
