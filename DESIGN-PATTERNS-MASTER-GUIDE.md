# Design Patterns — Unforgettable Master Guide (Java)

> **Goal:** See a problem → name the pattern in 30 seconds → explain why in an interview.

Patterns are **reusable solutions to recurring design problems**. They are not code you copy — they are **vocabulary** for how experienced engineers think.

---

## How to Use This Guide

For each pattern read in order:
1. **Never Forget** — the story that sticks
2. **Why / How / Where** — interview answers
3. **Java** — minimal but complete example
4. **Interview signal** — when the interviewer wants this pattern

---

## Pattern Map (Memorize)

| Category | Patterns | One-line memory hook |
|----------|----------|----------------------|
| **Creational** | Singleton, Factory, Abstract Factory, Builder, Prototype | *How objects are born* |
| **Structural** | Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy | *How objects are assembled* |
| **Behavioral** | Observer, Strategy, State, Command, Template, Iterator, Chain, Mediator, Memento, Visitor | *How objects communicate* |

---

# CREATIONAL PATTERNS

---

## 1. Singleton

### Never Forget
**One boss, one office.** The company has exactly one CEO. Every department asks "the CEO" — same person, same door. You don't `new CEO()` from every floor.

### Why
- Guarantee **exactly one instance** (config, connection pool, logger, cache manager)
- Controlled global access point

### How
- Private constructor + static `getInstance()`
- Variants: eager, lazy, double-checked locking, enum (best in Java), Bill Pugh holder

### Where (real world)
- `Runtime.getRuntime()`, Spring `@Scope("singleton")`, database connection pools, app config, `Logger.getLogger()`

### Interview signal
> "We must ensure only one instance exists across the JVM / application."

```java
// Enum singleton — thread-safe, serialization-safe (Joshua Bloch recommended)
public enum AppConfig {
 INSTANCE;

 private final Properties props = new Properties();

 public String get(String key) {
 return props.getProperty(key);
 }
}

// Usage: AppConfig.INSTANCE.get("db.url");
```

```java
// Lazy holder — loads only when first accessed, thread-safe without sync
public class ConnectionPool {
 private ConnectionPool() {}

 private static class Holder {
 static final ConnectionPool INSTANCE = new ConnectionPool();
 }

 public static ConnectionPool getInstance() {
 return Holder.INSTANCE;
 }
}
```

---

## 2. Factory Method

### Never Forget
**Pizza shop with regional style.** You order "a pizza" — NYC store makes thin crust, Chicago makes deep dish. Same order interface, **subclass decides** which concrete product.

### Why
- Decouple client from concrete class names
- Subclasses choose which object to instantiate

### How
- Abstract creator with `createProduct()` — concrete creators override

### Where
- `Calendar.getInstance()`, `NumberFormat.getInstance()`, `List.of()` vs `ArrayList` internals, framework hooks

### Interview signal
> "Creation logic varies by type/context, but client shouldn't know concrete classes."

```java
// Product
interface Notification {
 void send(String to, String message);
}

class EmailNotification implements Notification {
 public void send(String to, String message) {
 System.out.println("Email to " + to + ": " + message);
 }
}

class PushNotification implements Notification {
 public void send(String to, String message) {
 System.out.println("Push to " + to + ": " + message);
 }
}

// Creator — factory method
abstract class NotificationFactory {
 public void notifyUser(String userId, String msg) {
 Notification n = createNotification(); // subclass decides
 n.send(userId, msg);
 }
 protected abstract Notification createNotification();
}

class EmailFactory extends NotificationFactory {
 protected Notification createNotification() {
 return new EmailNotification();
 }
}
```

---

## 3. Abstract Factory

### Never Forget
**Furniture families.** You pick "Modern" or "Victorian" — you get matching chair + sofa + table. Switch family = **all products change together**. One factory per theme.

### Why
- Create **families of related products** without naming concrete classes
- Ensure products from same family are compatible

### How
- Interface for each product type + interface for factory that creates all products in family

### Where
- UI toolkits (Win/Mac/Linux widgets), cross-platform themes, AWS vs Azure client families

### Interview signal
> "We need multiple related objects (DB + Cache + Queue) that must match the same vendor/environment."

```java
interface Button { void render(); }
interface Checkbox { void render(); }

class WinButton implements Button { public void render() { System.out.println("Win button"); } }
class WinCheckbox implements Checkbox { public void render() { System.out.println("Win checkbox"); } }
class MacButton implements Button { public void render() { System.out.println("Mac button"); } }
class MacCheckbox implements Checkbox { public void render() { System.out.println("Mac checkbox"); } }

interface GUIFactory {
 Button createButton();
 Checkbox createCheckbox();
}

class WindowsFactory implements GUIFactory {
 public Button createButton() { return new WinButton(); }
 public Checkbox createCheckbox() { return new WinCheckbox(); }
}

class MacFactory implements GUIFactory {
 public Button createButton() { return new MacButton(); }
 public Checkbox createCheckbox() { return new MacCheckbox(); }
}
```

---

## 4. Builder

### Never Forget
**Custom burger order.** Bun, patty, cheese, bacon, sauce — 20 optional fields. You don't want a constructor with 20 parameters. You **assemble step by step**, then `build()`.

### Why
- Construct complex objects with many optional fields
- Readable, immutable result, validation at build time

### How
- Builder with fluent methods returning `this`, final `build()` calls private constructor

### Where
- `StringBuilder`, Lombok `@Builder`, Protobuf builders, HTTP request builders, test data builders

### Interview signal
> "Object has many optional parameters / step-by-step construction / immutability required."

```java
public final class Pizza {
 private final String size;
 private final boolean cheese;
 private final boolean pepperoni;

 private Pizza(Builder b) {
 this.size = b.size;
 this.cheese = b.cheese;
 this.pepperoni = b.pepperoni;
 }

 public static class Builder {
 private String size = "medium";
 private boolean cheese = true;
 private boolean pepperoni = false;

 public Builder size(String size) { this.size = size; return this; }
 public Builder cheese(boolean cheese) { this.cheese = cheese; return this; }
 public Builder pepperoni(boolean pepperoni) { this.pepperoni = pepperoni; return this; }
 public Pizza build() { return new Pizza(this); }
 }
}

// Pizza pizza = new Pizza.Builder().size("large").pepperoni(true).build();
```

---

## 5. Prototype

### Never Forget
**Photocopy machine.** Creating from scratch is expensive (DB load, parsing). Clone an existing object, tweak a few fields. `clone()` = instant duplicate.

### Why
- Copy existing instance instead of expensive construction
- Preserve pre-configured state

### How
- `Cloneable` + `clone()` or copy constructor / manual copy method (preferred)

### Where
- Game entities (spawn from template), document templates, Spring `prototype` scope beans

### Interview signal
> "Creating object is costly; we have a prototype to duplicate and modify."

```java
class Resume implements Cloneable {
 String name;
 List<String> skills;

 public Resume(String name, List<String> skills) {
 this.name = name;
 this.skills = new ArrayList<>(skills);
 }

 @Override
 public Resume clone() {
 try {
 Resume copy = (Resume) super.clone();
 copy.skills = new ArrayList<>(this.skills); // deep copy mutable fields
 return copy;
 } catch (CloneNotSupportedException e) {
 throw new AssertionError();
 }
 }
}
```

---

# STRUCTURAL PATTERNS

---

## 6. Adapter

### Never Forget
**Power plug adapter abroad.** Your US laptop plug doesn't fit EU socket. Adapter **wraps** one interface to look like another. Client unchanged.

### Why
- Make incompatible interfaces work together
- Integrate legacy/third-party code

### How
- Adapter class implements target interface, holds adaptee, delegates + transforms

### Where
- `Arrays.asList()`, `InputStreamReader`, JDBC drivers, REST client wrapping SOAP

### Interview signal
> "We have existing class but client expects a different interface."

```java
// Adaptee — third-party / legacy
class LegacyPayment {
 void payInCents(int cents) { System.out.println("Paid " + cents + " cents"); }
}

// Target — what our app expects
interface PaymentProcessor {
 void pay(double dollars);
}

// Adapter
class LegacyPaymentAdapter implements PaymentProcessor {
 private final LegacyPayment legacy;

 LegacyPaymentAdapter(LegacyPayment legacy) { this.legacy = legacy; }

 public void pay(double dollars) {
 legacy.payInCents((int) (dollars * 100));
 }
}
```

---

## 7. Bridge

### Never Forget
**Remote control + devices.** Remote has buttons; TV or Radio **implements** "device API." Change remote logic OR device independently — two hierarchies bridged by composition, not explosion of subclasses (RemoteForSonyTV, RemoteForSamsungTV...).

### Why
- Split abstraction from implementation so both evolve separately
- Avoid cartesian product of subclasses

### How
- Abstraction holds reference to Implementor interface; concrete abstractions call implementor

### Where
- JDBC (Connection abstraction + driver implementation), messaging (sender + platform)

### Interview signal
> "Two dimensions of variation — we don't want N×M subclasses."

```java
interface Renderer { void renderCircle(float r); }

class VectorRenderer implements Renderer {
 public void renderCircle(float r) { System.out.println("Vector circle r=" + r); }
}

class RasterRenderer implements Renderer {
 public void renderCircle(float r) { System.out.println("Raster circle r=" + r); }
}

abstract class Shape {
 protected Renderer renderer;
 Shape(Renderer renderer) { this.renderer = renderer; }
 abstract void draw();
}

class Circle extends Shape {
 private float radius;
 Circle(Renderer renderer, float radius) { super(renderer); this.radius = radius; }
 void draw() { renderer.renderCircle(radius); }
}
```

---

## 8. Composite

### Never Forget
**File system tree.** File and Folder both are "things you can get size of." Folder contains files AND folders. Client treats **leaf and container the same** — `getSize()` works on both.

### Why
- Uniform treatment of individual objects and compositions
- Tree structures (UI, org chart, menus)

### How
- Component interface; Leaf and Composite implement; Composite holds children

### Where
- Java `java.awt.Container`, DOM nodes, org hierarchies, menu systems

### Interview signal
> "Part-whole tree — treat single item and group identically."

```java
interface FileSystemNode {
 int getSize();
}

class File implements FileSystemNode {
 private int size;
 File(int size) { this.size = size; }
 public int getSize() { return size; }
}

class Directory implements FileSystemNode {
 private List<FileSystemNode> children = new ArrayList<>();
 void add(FileSystemNode node) { children.add(node); }
 public int getSize() {
 return children.stream().mapToInt(FileSystemNode::getSize).sum();
 }
}
```

---

## 9. Decorator

### Never Forget
**Coffee add-ons.** Base espresso. Wrap with milk → Mocha decorator. Wrap again with whip → still a `Coffee`, price stacks. **Wrap objects** to add behavior without subclass explosion.

### Why
- Add responsibilities dynamically at runtime
- Alternative to subclassing for every combination

### How
- Decorator implements same interface as component, holds wrapped component, delegates + extends

### Where
- Java I/O (`BufferedInputStream` wraps `FileInputStream`), Spring `@Transactional` wrappers, HTTP middleware

### Interview signal
> "Add features in layers at runtime — logging, caching, auth on top of core service."

```java
interface Coffee {
 double cost();
 String description();
}

class SimpleCoffee implements Coffee {
 public double cost() { return 2.0; }
 public String description() { return "Coffee"; }
}

abstract class CoffeeDecorator implements Coffee {
 protected Coffee wrapped;
 CoffeeDecorator(Coffee wrapped) { this.wrapped = wrapped; }
 public double cost() { return wrapped.cost(); }
 public String description() { return wrapped.description(); }
}

class MilkDecorator extends CoffeeDecorator {
 MilkDecorator(Coffee wrapped) { super(wrapped); }
 public double cost() { return super.cost() + 0.5; }
 public String description() { return super.description() + ", milk"; }
}

// Coffee c = new MilkDecorator(new SimpleCoffee());
```

---

## 10. Facade

### Never Forget
**Hotel concierge.** You don't call plumber, chef, and cleaner separately. One desk: "I need dinner and towels." Facade **simplifies** a messy subsystem behind one friendly API.

### Why
- Simple interface to complex subsystem
- Reduce coupling for clients

### How
- Facade class delegates to multiple subsystem classes

### Where
- SLF4J over Log4j/JUL, Spring `JdbcTemplate`, AWS SDK high-level clients

### Interview signal
> "Subsystem is complex; clients need one simple entry point."

```java
class VideoDecoder { void decode(String file) {} }
class AudioMixer { void mix(String track) {} }
class VideoEncoder { void encode() {} }

class VideoConverterFacade {
 private final VideoDecoder decoder = new VideoDecoder();
 private final AudioMixer mixer = new AudioMixer();
 private final VideoEncoder encoder = new VideoEncoder();

 public void convert(String input, String output) {
 decoder.decode(input);
 mixer.mix("default");
 encoder.encode();
 System.out.println("Converted " + input + " -> " + output);
 }
}
```

---

## 11. Flyweight

### Never Forget
**Forest of trees.** Millions of trees share same **texture** and **mesh** (intrinsic state). Only position/size differ (extrinsic). Cache shared flyweights — memory saved.

### Why
- Share common intrinsic state across many fine-grained objects
- Reduce memory when many similar objects exist

### How
- Factory/cache maps key → flyweight; client passes extrinsic state to methods

### Where
- String intern pool, character formatting in editors, game particle systems, `Integer.valueOf(-128..127)`

### Interview signal
> "Huge number of similar objects — separate shared vs unique state."

```java
class TreeType { // flyweight — shared
 final String name;
 final String color;
 final String texture;
 TreeType(String name, String color, String texture) {
 this.name = name; this.color = color; this.texture = texture;
 }
}

class TreeFactory {
 private static final Map<String, TreeType> cache = new HashMap<>();
 static TreeType get(String name, String color, String texture) {
 String key = name + color + texture;
 return cache.computeIfAbsent(key, k -> new TreeType(name, color, texture));
 }
}

class Tree { // context — extrinsic x,y
 int x, y;
 TreeType type;
 Tree(int x, int y, TreeType type) { this.x = x; this.y = y; this.type = type; }
}
```

---

## 12. Proxy

### Never Forget
**Celebrity's assistant.** You don't call the star directly. Assistant checks access, logs request, maybe returns cached autograph photo. **Same interface** as celebrity, controls access/ adds behavior.

### Why
- Control access (lazy load, security, remote)
- Add behavior without changing real object

### How
- Proxy implements same interface, holds reference to real subject

### Where
- Spring AOP proxies, Hibernate lazy loading, RMI, `java.lang.reflect.Proxy`, caching proxies

### Interview signal
> "Control access to expensive/remote object — lazy init, auth, logging."

```java
interface Image {
 void display();
}

class RealImage implements Image {
 private final String filename;
 RealImage(String filename) {
 this.filename = filename;
 loadFromDisk();
 }
 private void loadFromDisk() { System.out.println("Loading " + filename); }
 public void display() { System.out.println("Displaying " + filename); }
}

class ImageProxy implements Image {
 private final String filename;
 private RealImage real;
 ImageProxy(String filename) { this.filename = filename; }
 public void display() {
 if (real == null) real = new RealImage(filename); // lazy load
 real.display();
 }
}
```

---

# BEHAVIORAL PATTERNS

---

## 13. Observer

### Never Forget
**YouTube subscribe.** You subscribe to a channel. New video drops → **all subscribers notified** automatically. Subject doesn't know subscriber details — just broadcasts.

### Why
- One-to-many dependency: when state changes, dependents auto-update
- Decouple publisher from subscribers

### How
- Subject maintains observer list; `attach/detach/notify`; observers implement `update()`

### Where
- `java.util.Observable` (deprecated), Spring `ApplicationEventPublisher`, RxJava, Kafka consumers, MVC

### Interview signal
> "When X happens, many listeners must react — event-driven, pub/sub."

```java
import java.util.*;

interface EventListener {
 void update(String event);
}

class NewsAgency {
 private final List<EventListener> listeners = new ArrayList<>();
 void subscribe(EventListener l) { listeners.add(l); }
 void publish(String news) {
 for (EventListener l : listeners) l.update(news);
 }
}

class NewsChannel implements EventListener {
 private final String name;
 NewsChannel(String name) { this.name = name; }
 public void update(String event) {
 System.out.println(name + " received: " + event);
 }
}
```

---

## 14. Strategy

### Never Forget
**Navigation app — route strategy.** Same "get directions" button. Driving vs Walking vs Transit = **different algorithms**, swapped at runtime. Context doesn't care which strategy — interface is the same.

### Why
- Encapsulate interchangeable algorithms
- Open/closed: add strategies without changing context

### How
- Strategy interface + concrete strategies; context holds strategy reference

### Where
- `Comparator`, payment methods, compression algorithms, auth providers (OAuth, SAML)

### Interview signal
> "Multiple ways to do the same job — pick at runtime."

```java
interface PaymentStrategy {
 void pay(int amount);
}

class CreditCardPayment implements PaymentStrategy {
 public void pay(int amount) { System.out.println("Paid $" + amount + " via card"); }
}

class PayPalPayment implements PaymentStrategy {
 public void pay(int amount) { System.out.println("Paid $" + amount + " via PayPal"); }
}

class ShoppingCart {
 private PaymentStrategy strategy;
 void setPaymentStrategy(PaymentStrategy strategy) { this.strategy = strategy; }
 void checkout(int total) { strategy.pay(total); }
}
```

---

## 15. State

### Never Forget
**Vending machine moods.** Idle → accept coin → HasMoney → select product → Dispensing → Idle. **Same machine**, behavior changes with internal state. State objects replace giant `if/else`.

### Why
- Object behavior changes when internal state changes
- Replace conditional logic with polymorphism

### How
- State interface with methods; concrete states; context delegates to current state

### Where
- TCP connection states, workflow engines, order status (Pending → Shipped → Delivered), media players

### Interview signal
> "Object acts differently in different states — lots of state-dependent conditionals."

```java
interface VendingState {
 void insertCoin(VendingMachine machine);
 void dispense(VendingMachine machine);
}

class IdleState implements VendingState {
 public void insertCoin(VendingMachine m) {
 System.out.println("Coin inserted");
 m.setState(new HasCoinState());
 }
 public void dispense(VendingMachine m) { System.out.println("Insert coin first"); }
}

class HasCoinState implements VendingState {
 public void insertCoin(VendingMachine m) { System.out.println("Already has coin"); }
 public void dispense(VendingMachine m) {
 System.out.println("Dispensing...");
 m.setState(new IdleState());
 }
}

class VendingMachine {
 private VendingState state = new IdleState();
 void setState(VendingState state) { this.state = state; }
 void insertCoin() { state.insertCoin(this); }
 void dispense() { state.dispense(this); }
}
```

---

## 16. Command

### Never Forget
**Restaurant order ticket.** Waiter writes order on slip — doesn't cook. Slip goes to kitchen (execute later). **Undo** = cancel order. Queue orders. Log for replay.

### Why
- Encapsulate request as object — execute, queue, undo, log
- Decouple invoker from receiver

### How
- Command interface `execute()` / `undo()`; concrete commands hold receiver

### Where
- Job queues, undo/redo in editors, transactional systems, `Runnable`, macro recording

### Interview signal
> "Actions as first-class objects — undo, queue, audit trail."

```java
interface Command {
 void execute();
 void undo();
}

class Light {
 void on() { System.out.println("Light on"); }
 void off() { System.out.println("Light off"); }
}

class LightOnCommand implements Command {
 private final Light light;
 LightOnCommand(Light light) { this.light = light; }
 public void execute() { light.on(); }
 public void undo() { light.off(); }
}

class RemoteControl {
 private final Deque<Command> history = new ArrayDeque<>();
 void press(Command cmd) { cmd.execute(); history.push(cmd); }
 void undo() { if (!history.isEmpty()) history.pop().undo(); }
}
```

---

## 17. Template Method

### Never Forget
**Recipe skeleton.** Make beverage: boil water → brew → pour → add condiments. Coffee and Tea **share steps**, differ only in `brew()` and `addCondiments()`. Parent defines algorithm; child fills hooks.

### Why
- Define algorithm skeleton; subclasses override specific steps
- Code reuse without duplication

### How
- Abstract class with `final templateMethod()` calling abstract/overridable hooks

### Where
- `HttpServlet.service()`, JUnit `setUp/test/tearDown`, Spring `JdbcTemplate`, ETL pipelines

### Interview signal
> "Fixed sequence of steps, some steps vary by subclass."

```java
abstract class DataMiner {
 public final void mine() { // template — cannot override order
 openConnection();
 extract();
 parse();
 closeConnection();
 }
 abstract void extract();
 abstract void parse();
 void openConnection() { System.out.println("DB connected"); }
 void closeConnection() { System.out.println("DB closed"); }
}

class UserMiner extends DataMiner {
 void extract() { System.out.println("Extract users"); }
 void parse() { System.out.println("Parse users"); }
}
```

---

## 18. Iterator

### Never Forget
**TV remote channel scan.** You press "next" — don't care if channels live in array or linked list. **Uniform way** to walk a collection without exposing internals.

### Why
- Traverse collection without exposing representation
- Multiple traversals, polymorphic iteration

### How
- Iterator interface `hasNext()/next()`; aggregate provides `iterator()`

### Where
- Java `Iterator`, `for-each`, database cursors, pagination APIs

### Interview signal
> "Traverse diverse collections the same way."

```java
interface PlaylistIterator {
 boolean hasNext();
 String next();
}

class SongPlaylist implements PlaylistIterator {
 private final List<String> songs;
 private int index = 0;
 SongPlaylist(List<String> songs) { this.songs = songs; }
 public boolean hasNext() { return index < songs.size(); }
 public String next() { return songs.get(index++); }
}
```

---

## 19. Chain of Responsibility

### Never Forget
**Support escalation ladder.** Bot handles FAQ. Can't? → Agent. Still stuck? → Manager. Request passes **chain** until someone handles it or it falls through.

### Why
- Pass request along handler chain until one processes it
- Decouple sender from receivers

### How
- Handler interface with `setNext()` and `handle()`; each forwards or processes

### Where
- Servlet filters, logging pipelines, auth middleware, exception handling chains

### Interview signal
> "Multiple handlers might process request — order matters, sender doesn't know which."

```java
abstract class SupportHandler {
 private SupportHandler next;
 void setNext(SupportHandler next) { this.next = next; }
 void handle(String issue, int severity) {
 if (canHandle(severity)) resolve(issue);
 else if (next != null) next.handle(issue, severity);
 else System.out.println("Escalation failed: " + issue);
 }
 abstract boolean canHandle(int severity);
 abstract void resolve(String issue);
}

class Level1Support extends SupportHandler {
 boolean canHandle(int s) { return s <= 1; }
 void resolve(String issue) { System.out.println("L1 fixed: " + issue); }
}
```

---

## 20. Mediator

### Never Forget
**Air traffic control.** Planes don't talk to each other directly — tower coordinates. Mediator **central hub** reduces spaghetti between many objects.

### Why
- Reduce chaotic many-to-many dependencies
- Centralize complex communication

### How
- Mediator interface; colleagues communicate only through mediator

### Where
- Chat rooms, dialog coordinators in UI, event buses (careful — can become god object)

### Interview signal
> "Many components interact — direct links would be unmaintainable."

```java
interface ChatMediator {
 void send(String from, String to, String msg);
}

class ChatRoom implements ChatMediator {
 private final Map<String, User> users = new HashMap<>();
 void register(User u) { users.put(u.getName(), u); }
 public void send(String from, String to, String msg) {
 User recipient = users.get(to);
 if (recipient != null) recipient.receive(from, msg);
 }
}

class User {
 private final String name;
 private final ChatMediator mediator;
 User(String name, ChatMediator m) { this.name = name; this.mediator = m; }
 String getName() { return name; }
 void send(String to, String msg) { mediator.send(name, to, msg); }
 void receive(String from, String msg) { System.out.println(name + " <- " + from + ": " + msg); }
}
```

---

## 21. Memento

### Never Forget
**Save game checkpoint.** Play → save snapshot → die → **load snapshot**, back to exact state. Memento stores state; originator restores; caretaker holds history (can't peek inside).

### Why
- Capture and restore object state without exposing internals
- Undo snapshots

### How
- Memento stores state; Originator creates/restores from memento

### Where
- Undo in editors, game saves, transaction rollback snapshots

### Interview signal
> "Need to restore previous state — undo, snapshots."

```java
class EditorMemento {
 private final String content;
 EditorMemento(String content) { this.content = content; }
 String getContent() { return content; }
}

class Editor {
 private String content = "";
 void type(String text) { content += text; }
 EditorMemento save() { return new EditorMemento(content); }
 void restore(EditorMemento m) { this.content = m.getContent(); }
 String getContent() { return content; }
}
```

---

## 22. Visitor

### Never Forget
**Tax auditor visits every room.** New operation (calculate tax) on many room types without adding methods to every room class. **Double dispatch**: visitor visits, element accepts visitor.

### Why
- Add new operations to object structure without modifying element classes
- When structure stable but operations change often

### How
- Visitor interface with `visit(ConcreteElement)` per type; elements `accept(visitor)`

### Where
- AST interpreters, compiler passes, export serializers (PDF/XML) over document tree

### Interview signal
> "Many types, new operation to apply to all — avoid polluting every class."

```java
interface ShapeVisitor {
 void visit(Circle c);
 void visit(Rectangle r);
}

interface Shape {
 void accept(ShapeVisitor v);
}

class Circle implements Shape {
 double radius;
 Circle(double r) { this.radius = r; }
 public void accept(ShapeVisitor v) { v.visit(this); }
}

class AreaCalculator implements ShapeVisitor {
 private double total = 0;
 public void visit(Circle c) { total += Math.PI * c.radius * c.radius; }
 public void visit(Rectangle r) { total += r.width * r.height; }
 double getTotal() { return total; }
}
```

---

# BONUS — Interview-Favorite Patterns

## Repository
**Never Forget:** Librarian desk — you ask for "a User by id", not "run SQL on table users." Hides persistence.

```java
interface UserRepository {
 Optional<User> findById(Long id);
 void save(User user);
}
```

## Dependency Injection
**Never Forget:** Restaurant brings food to your table — you don't walk into the kitchen (`new`). Container wires dependencies.

```java
@Service
public class OrderService {
 private final PaymentProcessor payment;
 public OrderService(PaymentProcessor payment) { this.payment = payment; }
}
```

---

# Quick Interview Cheat Sheet

| Problem smell | Pattern |
|---------------|---------|
| One instance only | Singleton |
| Create without naming class | Factory |
| Family of related products | Abstract Factory |
| Many optional constructor args | Builder |
| Clone expensive object | Prototype |
| Incompatible interface | Adapter |
| Add behavior in layers | Decorator |
| Simple API to complex system | Facade |
| Share memory across many objects | Flyweight |
| Lazy load / access control | Proxy |
| Notify many on change | Observer |
| Swap algorithm at runtime | Strategy |
| Behavior depends on state | State |
| Undo / queue actions | Command |
| Fixed steps, varying hooks | Template Method |
| Pass request along chain | Chain of Responsibility |

---

## Study Plan (2 weeks)

| Week | Focus |
|------|-------|
| 1 | Creational + Structural (1–12) — 2 patterns/day + whiteboard |
| 2 | Behavioral (13–22) + mock: "which pattern for X?" |

**Daily drill:** Read "Never Forget" → cover Java → explain Why/Where out loud in 60 seconds.
