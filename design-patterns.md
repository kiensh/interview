# Design Patterns Interview Questions & Answers

---

## Table of Contents

### [Creational Patterns](#creational-patterns)
- [Q1: What is the Singleton pattern?](#q1)
- [Q2: What is the Factory pattern?](#q2)
- [Q3: What is the Abstract Factory pattern?](#q3)
- [Q4: What is the Builder pattern?](#q4)
- [Q5: What is the Prototype pattern?](#q5)

### [Structural Patterns](#structural-patterns)
- [Q6: What is the Adapter pattern?](#q6)
- [Q7: What is the Decorator pattern?](#q7)
- [Q8: What is the Facade pattern?](#q8)
- [Q9: What is the Proxy pattern?](#q9)
- [Q10: What is the Composite pattern?](#q10)

### [Behavioral Patterns](#behavioral-patterns)
- [Q11: What is the Observer pattern?](#q11)
- [Q12: What is the Strategy pattern?](#q12)
- [Q13: What is the Template Method pattern?](#q13)
- [Q14: What is the Chain of Responsibility pattern?](#q14)
- [Q15: What is the Command pattern?](#q15)

### [Concurrency Patterns](#concurrency-patterns)
- [Q16: What is the Thread Pool pattern?](#q16)
- [Q17: What is the Future/Promise pattern?](#q17)
- [Q18: What is the Producer-Consumer pattern?](#q18)

### [Architectural Patterns](#architectural-patterns)
- [Q19: What is the MVC pattern?](#q19)
- [Q20: What is Clean Architecture?](#q20)
- [Q21: What is the Repository pattern?](#q21)

---

## Creational Patterns

<a id="q1"></a>
### Q1: What is the Singleton pattern?
**Answer:**
Singleton ensures a class has only one instance and provides global access to it.

```java
// Thread-safe Singleton using enum (recommended)
public enum DatabaseConnection {
    INSTANCE;
    
    private Connection connection;
    
    DatabaseConnection() {
        // Initialize connection
        this.connection = createConnection();
    }
    
    public Connection getConnection() {
        return connection;
    }
}

// Usage
Connection conn = DatabaseConnection.INSTANCE.getConnection();
```

```java
// Double-checked locking (classic approach)
public class Singleton {
    private static volatile Singleton instance;
    
    private Singleton() { }  // Private constructor
    
    public static Singleton getInstance() {
        if (instance == null) {                    // First check (no locking)
            synchronized (Singleton.class) {
                if (instance == null) {            // Second check (with lock)
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

```java
// Bill Pugh Singleton (lazy initialization)
public class Singleton {
    private Singleton() { }
    
    private static class Holder {
        private static final Singleton INSTANCE = new Singleton();
    }
    
    public static Singleton getInstance() {
        return Holder.INSTANCE;  // Loaded only when called
    }
}
```

**When to use:**
- Database connections
- Configuration managers
- Logger instances
- Thread pools

**Drawbacks:**
- Global state (hard to test)
- Hidden dependencies
- Tight coupling

<a id="q2"></a>
### Q2: What is the Factory pattern?
**Answer:**
Factory Method defines an interface for creating objects but lets subclasses decide which class to instantiate.

```java
// Product interface
public interface Notification {
    void send(String message);
}

// Concrete products
public class EmailNotification implements Notification {
    public void send(String message) {
        System.out.println("Email: " + message);
    }
}

public class SmsNotification implements Notification {
    public void send(String message) {
        System.out.println("SMS: " + message);
    }
}

public class PushNotification implements Notification {
    public void send(String message) {
        System.out.println("Push: " + message);
    }
}

// Simple Factory
public class NotificationFactory {
    public static Notification create(String type) {
        return switch (type.toLowerCase()) {
            case "email" -> new EmailNotification();
            case "sms" -> new SmsNotification();
            case "push" -> new PushNotification();
            default -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}

// Usage
Notification notification = NotificationFactory.create("email");
notification.send("Hello!");
```

```java
// Factory Method pattern (with inheritance)
public abstract class NotificationCreator {
    
    // Factory method
    public abstract Notification createNotification();
    
    // Template method using factory
    public void notifyUser(String message) {
        Notification notification = createNotification();
        notification.send(message);
    }
}

public class EmailNotificationCreator extends NotificationCreator {
    @Override
    public Notification createNotification() {
        return new EmailNotification();
    }
}

public class SmsNotificationCreator extends NotificationCreator {
    @Override
    public Notification createNotification() {
        return new SmsNotification();
    }
}
```

<a id="q3"></a>
### Q3: What is the Abstract Factory pattern?
**Answer:**
Abstract Factory provides an interface for creating families of related objects without specifying their concrete classes.

```java
// Abstract products
public interface Button { void render(); }
public interface Checkbox { void render(); }
public interface TextField { void render(); }

// Concrete products - Windows family
public class WindowsButton implements Button {
    public void render() { System.out.println("Windows Button"); }
}
public class WindowsCheckbox implements Checkbox {
    public void render() { System.out.println("Windows Checkbox"); }
}

// Concrete products - Mac family
public class MacButton implements Button {
    public void render() { System.out.println("Mac Button"); }
}
public class MacCheckbox implements Checkbox {
    public void render() { System.out.println("Mac Checkbox"); }
}

// Abstract Factory
public interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

// Concrete factories
public class WindowsUIFactory implements UIFactory {
    public Button createButton() { return new WindowsButton(); }
    public Checkbox createCheckbox() { return new WindowsCheckbox(); }
}

public class MacUIFactory implements UIFactory {
    public Button createButton() { return new MacButton(); }
    public Checkbox createCheckbox() { return new MacCheckbox(); }
}

// Client code
public class Application {
    private Button button;
    private Checkbox checkbox;
    
    public Application(UIFactory factory) {
        this.button = factory.createButton();
        this.checkbox = factory.createCheckbox();
    }
    
    public void render() {
        button.render();
        checkbox.render();
    }
}

// Usage
UIFactory factory = System.getProperty("os.name").contains("Windows") 
    ? new WindowsUIFactory() 
    : new MacUIFactory();
Application app = new Application(factory);
```

<a id="q4"></a>
### Q4: What is the Builder pattern?
**Answer:**
Builder separates the construction of a complex object from its representation, allowing the same construction process to create different representations.

```java
public class User {
    private final String firstName;      // Required
    private final String lastName;       // Required
    private final String email;          // Optional
    private final String phone;          // Optional
    private final String address;        // Optional
    private final int age;               // Optional
    
    private User(Builder builder) {
        this.firstName = builder.firstName;
        this.lastName = builder.lastName;
        this.email = builder.email;
        this.phone = builder.phone;
        this.address = builder.address;
        this.age = builder.age;
    }
    
    public static class Builder {
        // Required parameters
        private final String firstName;
        private final String lastName;
        
        // Optional parameters with defaults
        private String email = "";
        private String phone = "";
        private String address = "";
        private int age = 0;
        
        public Builder(String firstName, String lastName) {
            this.firstName = firstName;
            this.lastName = lastName;
        }
        
        public Builder email(String email) {
            this.email = email;
            return this;
        }
        
        public Builder phone(String phone) {
            this.phone = phone;
            return this;
        }
        
        public Builder address(String address) {
            this.address = address;
            return this;
        }
        
        public Builder age(int age) {
            this.age = age;
            return this;
        }
        
        public User build() {
            // Validation
            if (age < 0) throw new IllegalArgumentException("Age cannot be negative");
            return new User(this);
        }
    }
}

// Usage - fluent API
User user = new User.Builder("John", "Doe")
    .email("john@email.com")
    .phone("123-456-7890")
    .age(30)
    .build();
```

**Lombok @Builder:**
```java
@Builder
@Getter
public class User {
    private final String firstName;
    private final String lastName;
    @Builder.Default
    private final String email = "";
}

// Usage
User user = User.builder()
    .firstName("John")
    .lastName("Doe")
    .build();
```

<a id="q5"></a>
### Q5: What is the Prototype pattern?
**Answer:**
Prototype creates new objects by copying an existing object (prototype).

```java
public abstract class Shape implements Cloneable {
    protected String color;
    
    public abstract void draw();
    
    @Override
    public Shape clone() {
        try {
            return (Shape) super.clone();  // Shallow copy
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException(e);
        }
    }
}

public class Circle extends Shape {
    private int radius;
    
    public Circle(int radius, String color) {
        this.radius = radius;
        this.color = color;
    }
    
    @Override
    public void draw() {
        System.out.println("Circle: " + radius + ", " + color);
    }
    
    @Override
    public Circle clone() {
        Circle clone = (Circle) super.clone();
        return clone;
    }
}

// Prototype registry
public class ShapeRegistry {
    private Map<String, Shape> shapes = new HashMap<>();
    
    public ShapeRegistry() {
        // Pre-populate with prototypes
        shapes.put("large-red-circle", new Circle(100, "red"));
        shapes.put("small-blue-circle", new Circle(10, "blue"));
    }
    
    public Shape get(String key) {
        return shapes.get(key).clone();
    }
}

// Usage
ShapeRegistry registry = new ShapeRegistry();
Shape circle1 = registry.get("large-red-circle");  // Clone of prototype
Shape circle2 = registry.get("large-red-circle");  // Another clone
```

---

## Structural Patterns

<a id="q6"></a>
### Q6: What is the Adapter pattern?
**Answer:**
Adapter converts the interface of a class into another interface clients expect.

```java
// Target interface (what client expects)
public interface MediaPlayer {
    void play(String filename);
}

// Adaptee (incompatible interface)
public class VlcPlayer {
    public void playVlc(String filename) {
        System.out.println("Playing VLC: " + filename);
    }
}

public class Mp4Player {
    public void playMp4(String filename) {
        System.out.println("Playing MP4: " + filename);
    }
}

// Adapter
public class MediaAdapter implements MediaPlayer {
    private VlcPlayer vlcPlayer;
    private Mp4Player mp4Player;
    
    public MediaAdapter(String audioType) {
        if (audioType.equalsIgnoreCase("vlc")) {
            vlcPlayer = new VlcPlayer();
        } else if (audioType.equalsIgnoreCase("mp4")) {
            mp4Player = new Mp4Player();
        }
    }
    
    @Override
    public void play(String filename) {
        if (vlcPlayer != null) {
            vlcPlayer.playVlc(filename);
        } else if (mp4Player != null) {
            mp4Player.playMp4(filename);
        }
    }
}

// Client
public class AudioPlayer implements MediaPlayer {
    @Override
    public void play(String filename) {
        String extension = getExtension(filename);
        
        if (extension.equals("mp3")) {
            System.out.println("Playing MP3: " + filename);
        } else if (extension.equals("vlc") || extension.equals("mp4")) {
            MediaAdapter adapter = new MediaAdapter(extension);
            adapter.play(filename);
        }
    }
}
```

<a id="q7"></a>
### Q7: What is the Decorator pattern?
**Answer:**
Decorator dynamically adds responsibilities to objects without affecting other objects of the same class.

```java
// Component interface
public interface Coffee {
    String getDescription();
    double getCost();
}

// Concrete component
public class SimpleCoffee implements Coffee {
    public String getDescription() { return "Simple Coffee"; }
    public double getCost() { return 1.0; }
}

// Decorator base class
public abstract class CoffeeDecorator implements Coffee {
    protected Coffee decoratedCoffee;
    
    public CoffeeDecorator(Coffee coffee) {
        this.decoratedCoffee = coffee;
    }
    
    public String getDescription() {
        return decoratedCoffee.getDescription();
    }
    
    public double getCost() {
        return decoratedCoffee.getCost();
    }
}

// Concrete decorators
public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public String getDescription() {
        return decoratedCoffee.getDescription() + ", Milk";
    }
    
    @Override
    public double getCost() {
        return decoratedCoffee.getCost() + 0.5;
    }
}

public class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) {
        super(coffee);
    }
    
    @Override
    public String getDescription() {
        return decoratedCoffee.getDescription() + ", Sugar";
    }
    
    @Override
    public double getCost() {
        return decoratedCoffee.getCost() + 0.2;
    }
}

// Usage - stack decorators
Coffee coffee = new SimpleCoffee();
coffee = new MilkDecorator(coffee);
coffee = new SugarDecorator(coffee);
coffee = new SugarDecorator(coffee);  // Double sugar!

System.out.println(coffee.getDescription());  // Simple Coffee, Milk, Sugar, Sugar
System.out.println(coffee.getCost());         // 1.0 + 0.5 + 0.2 + 0.2 = 1.9
```

**Real-world example: Java I/O:**
```java
// Decorators in Java I/O
InputStream is = new BufferedInputStream(
    new FileInputStream("file.txt")
);

// Or with more decorators
DataInputStream dis = new DataInputStream(
    new BufferedInputStream(
        new FileInputStream("file.txt")
    )
);
```

<a id="q8"></a>
### Q8: What is the Facade pattern?
**Answer:**
Facade provides a simplified interface to a complex subsystem.

```java
// Complex subsystem classes
public class CPU {
    public void freeze() { System.out.println("CPU freeze"); }
    public void jump(long position) { System.out.println("CPU jump to " + position); }
    public void execute() { System.out.println("CPU execute"); }
}

public class Memory {
    public void load(long position, byte[] data) {
        System.out.println("Memory load at " + position);
    }
}

public class HardDrive {
    public byte[] read(long lba, int size) {
        System.out.println("HardDrive read");
        return new byte[size];
    }
}

// Facade
public class ComputerFacade {
    private CPU cpu;
    private Memory memory;
    private HardDrive hardDrive;
    
    public ComputerFacade() {
        this.cpu = new CPU();
        this.memory = new Memory();
        this.hardDrive = new HardDrive();
    }
    
    // Simple interface hiding complexity
    public void start() {
        cpu.freeze();
        memory.load(0, hardDrive.read(0, 1024));
        cpu.jump(0);
        cpu.execute();
    }
}

// Client - simple usage
ComputerFacade computer = new ComputerFacade();
computer.start();  // Hides all the complexity
```

**Real-world example:**
```java
// Without facade - complex
EntityManagerFactory emf = Persistence.createEntityManagerFactory("myPU");
EntityManager em = emf.createEntityManager();
em.getTransaction().begin();
User user = em.find(User.class, 1L);
em.getTransaction().commit();
em.close();

// With Spring Data (facade)
User user = userRepository.findById(1L).orElseThrow();
```

<a id="q9"></a>
### Q9: What is the Proxy pattern?
**Answer:**
Proxy provides a surrogate or placeholder for another object to control access to it.

**Types:**
| Type | Purpose |
|------|---------|
| Virtual Proxy | Lazy loading |
| Protection Proxy | Access control |
| Remote Proxy | Local representative of remote object |
| Logging Proxy | Log operations |
| Caching Proxy | Cache results |

```java
// Subject interface
public interface Image {
    void display();
}

// Real subject (expensive to create)
public class RealImage implements Image {
    private String filename;
    
    public RealImage(String filename) {
        this.filename = filename;
        loadFromDisk();  // Expensive operation
    }
    
    private void loadFromDisk() {
        System.out.println("Loading image: " + filename);
    }
    
    @Override
    public void display() {
        System.out.println("Displaying: " + filename);
    }
}

// Virtual Proxy (lazy loading)
public class ProxyImage implements Image {
    private RealImage realImage;
    private String filename;
    
    public ProxyImage(String filename) {
        this.filename = filename;
        // Don't load yet!
    }
    
    @Override
    public void display() {
        if (realImage == null) {
            realImage = new RealImage(filename);  // Load only when needed
        }
        realImage.display();
    }
}

// Usage
Image image = new ProxyImage("photo.jpg");  // Not loaded yet
// ... later ...
image.display();  // Now it loads
image.display();  // Uses cached real image
```

**Protection Proxy:**
```java
public class SecuredDocument implements Document {
    private RealDocument document;
    private User currentUser;
    
    public SecuredDocument(RealDocument document, User user) {
        this.document = document;
        this.currentUser = user;
    }
    
    @Override
    public void view() {
        if (currentUser.hasPermission("VIEW")) {
            document.view();
        } else {
            throw new AccessDeniedException("No view permission");
        }
    }
    
    @Override
    public void edit() {
        if (currentUser.hasPermission("EDIT")) {
            document.edit();
        } else {
            throw new AccessDeniedException("No edit permission");
        }
    }
}
```

<a id="q10"></a>
### Q10: What is the Composite pattern?
**Answer:**
Composite composes objects into tree structures to represent part-whole hierarchies.

```java
// Component
public interface FileSystemItem {
    String getName();
    long getSize();
    void print(String indent);
}

// Leaf
public class File implements FileSystemItem {
    private String name;
    private long size;
    
    public File(String name, long size) {
        this.name = name;
        this.size = size;
    }
    
    @Override
    public String getName() { return name; }
    
    @Override
    public long getSize() { return size; }
    
    @Override
    public void print(String indent) {
        System.out.println(indent + "📄 " + name + " (" + size + " bytes)");
    }
}

// Composite
public class Directory implements FileSystemItem {
    private String name;
    private List<FileSystemItem> children = new ArrayList<>();
    
    public Directory(String name) {
        this.name = name;
    }
    
    public void add(FileSystemItem item) {
        children.add(item);
    }
    
    public void remove(FileSystemItem item) {
        children.remove(item);
    }
    
    @Override
    public String getName() { return name; }
    
    @Override
    public long getSize() {
        return children.stream()
            .mapToLong(FileSystemItem::getSize)
            .sum();
    }
    
    @Override
    public void print(String indent) {
        System.out.println(indent + "📁 " + name);
        for (FileSystemItem child : children) {
            child.print(indent + "  ");
        }
    }
}

// Usage
Directory root = new Directory("root");
Directory docs = new Directory("documents");
docs.add(new File("resume.pdf", 1024));
docs.add(new File("report.docx", 2048));

Directory pics = new Directory("pictures");
pics.add(new File("photo.jpg", 4096));

root.add(docs);
root.add(pics);
root.add(new File("readme.txt", 100));

root.print("");
System.out.println("Total size: " + root.getSize());
```

---

## Behavioral Patterns

<a id="q11"></a>
### Q11: What is the Observer pattern?
**Answer:**
Observer defines a one-to-many dependency between objects so that when one object changes state, all its dependents are notified.

```java
// Observer interface
public interface Observer {
    void update(String message);
}

// Subject interface
public interface Subject {
    void attach(Observer observer);
    void detach(Observer observer);
    void notifyObservers();
}

// Concrete subject
public class NewsAgency implements Subject {
    private List<Observer> observers = new ArrayList<>();
    private String news;
    
    @Override
    public void attach(Observer observer) {
        observers.add(observer);
    }
    
    @Override
    public void detach(Observer observer) {
        observers.remove(observer);
    }
    
    @Override
    public void notifyObservers() {
        for (Observer observer : observers) {
            observer.update(news);
        }
    }
    
    public void setNews(String news) {
        this.news = news;
        notifyObservers();
    }
}

// Concrete observers
public class NewsChannel implements Observer {
    private String name;
    
    public NewsChannel(String name) {
        this.name = name;
    }
    
    @Override
    public void update(String news) {
        System.out.println(name + " received news: " + news);
    }
}

// Usage
NewsAgency agency = new NewsAgency();
agency.attach(new NewsChannel("CNN"));
agency.attach(new NewsChannel("BBC"));

agency.setNews("Breaking: Important event!");
// Both channels receive the news
```

**Java built-in (PropertyChangeListener):**
```java
public class Stock {
    private PropertyChangeSupport support = new PropertyChangeSupport(this);
    private double price;
    
    public void addPropertyChangeListener(PropertyChangeListener listener) {
        support.addPropertyChangeListener(listener);
    }
    
    public void setPrice(double newPrice) {
        double oldPrice = this.price;
        this.price = newPrice;
        support.firePropertyChange("price", oldPrice, newPrice);
    }
}
```

<a id="q12"></a>
### Q12: What is the Strategy pattern?
**Answer:**
Strategy defines a family of algorithms, encapsulates each one, and makes them interchangeable.

```java
// Strategy interface
public interface PaymentStrategy {
    void pay(double amount);
}

// Concrete strategies
public class CreditCardPayment implements PaymentStrategy {
    private String cardNumber;
    
    public CreditCardPayment(String cardNumber) {
        this.cardNumber = cardNumber;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " with credit card " + cardNumber);
    }
}

public class PayPalPayment implements PaymentStrategy {
    private String email;
    
    public PayPalPayment(String email) {
        this.email = email;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " via PayPal: " + email);
    }
}

public class CryptoPayment implements PaymentStrategy {
    private String walletAddress;
    
    public CryptoPayment(String walletAddress) {
        this.walletAddress = walletAddress;
    }
    
    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " in crypto to " + walletAddress);
    }
}

// Context
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();
    private PaymentStrategy paymentStrategy;
    
    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }
    
    public void checkout() {
        double total = items.stream().mapToDouble(Item::getPrice).sum();
        paymentStrategy.pay(total);
    }
}

// Usage - strategy can be changed at runtime
ShoppingCart cart = new ShoppingCart();
cart.addItem(new Item("Book", 29.99));

cart.setPaymentStrategy(new CreditCardPayment("1234-5678"));
cart.checkout();

cart.setPaymentStrategy(new PayPalPayment("user@email.com"));
cart.checkout();
```

<a id="q13"></a>
### Q13: What is the Template Method pattern?
**Answer:**
Template Method defines the skeleton of an algorithm in a method, deferring some steps to subclasses.

```java
// Abstract class with template method
public abstract class DataProcessor {
    
    // Template method - defines algorithm skeleton
    public final void process() {
        readData();
        processData();
        writeData();
        if (shouldNotify()) {  // Hook method
            notify();
        }
    }
    
    // Abstract methods - must be implemented
    protected abstract void readData();
    protected abstract void processData();
    protected abstract void writeData();
    
    // Hook method - optional override
    protected boolean shouldNotify() {
        return false;
    }
    
    protected void notify() {
        System.out.println("Processing complete");
    }
}

// Concrete implementations
public class CSVProcessor extends DataProcessor {
    @Override
    protected void readData() {
        System.out.println("Reading CSV file");
    }
    
    @Override
    protected void processData() {
        System.out.println("Processing CSV data");
    }
    
    @Override
    protected void writeData() {
        System.out.println("Writing to database");
    }
    
    @Override
    protected boolean shouldNotify() {
        return true;  // Override hook
    }
}

public class XMLProcessor extends DataProcessor {
    @Override
    protected void readData() {
        System.out.println("Parsing XML");
    }
    
    @Override
    protected void processData() {
        System.out.println("Transforming XML data");
    }
    
    @Override
    protected void writeData() {
        System.out.println("Writing XML output");
    }
}

// Usage
DataProcessor csvProcessor = new CSVProcessor();
csvProcessor.process();  // Follows template, notifies

DataProcessor xmlProcessor = new XMLProcessor();
xmlProcessor.process();  // Follows template, no notification
```

<a id="q14"></a>
### Q14: What is the Chain of Responsibility pattern?
**Answer:**
Chain of Responsibility passes a request along a chain of handlers, where each handler decides either to process the request or pass it to the next handler.

```java
// Handler interface
public abstract class SupportHandler {
    protected SupportHandler nextHandler;
    
    public void setNext(SupportHandler handler) {
        this.nextHandler = handler;
    }
    
    public abstract void handle(SupportTicket ticket);
    
    protected void passToNext(SupportTicket ticket) {
        if (nextHandler != null) {
            nextHandler.handle(ticket);
        } else {
            System.out.println("No handler available for: " + ticket.getIssue());
        }
    }
}

// Concrete handlers
public class Level1Support extends SupportHandler {
    @Override
    public void handle(SupportTicket ticket) {
        if (ticket.getSeverity() == Severity.LOW) {
            System.out.println("Level 1 handled: " + ticket.getIssue());
        } else {
            passToNext(ticket);
        }
    }
}

public class Level2Support extends SupportHandler {
    @Override
    public void handle(SupportTicket ticket) {
        if (ticket.getSeverity() == Severity.MEDIUM) {
            System.out.println("Level 2 handled: " + ticket.getIssue());
        } else {
            passToNext(ticket);
        }
    }
}

public class Level3Support extends SupportHandler {
    @Override
    public void handle(SupportTicket ticket) {
        if (ticket.getSeverity() == Severity.HIGH) {
            System.out.println("Level 3 handled: " + ticket.getIssue());
        } else {
            passToNext(ticket);
        }
    }
}

// Usage
SupportHandler level1 = new Level1Support();
SupportHandler level2 = new Level2Support();
SupportHandler level3 = new Level3Support();

// Build chain
level1.setNext(level2);
level2.setNext(level3);

// Handle tickets
level1.handle(new SupportTicket("Password reset", Severity.LOW));    // Level 1
level1.handle(new SupportTicket("Server slow", Severity.MEDIUM));    // Level 2
level1.handle(new SupportTicket("System down!", Severity.HIGH));     // Level 3
```

**Real-world: Spring Security Filter Chain**

<a id="q15"></a>
### Q15: What is the Command pattern?
**Answer:**
Command encapsulates a request as an object, allowing parameterization, queuing, logging, and undo operations.

```java
// Command interface
public interface Command {
    void execute();
    void undo();
}

// Receiver
public class TextEditor {
    private StringBuilder text = new StringBuilder();
    
    public void write(String str) {
        text.append(str);
    }
    
    public void delete(int length) {
        int start = text.length() - length;
        if (start >= 0) {
            text.delete(start, text.length());
        }
    }
    
    public String getText() {
        return text.toString();
    }
}

// Concrete commands
public class WriteCommand implements Command {
    private TextEditor editor;
    private String text;
    
    public WriteCommand(TextEditor editor, String text) {
        this.editor = editor;
        this.text = text;
    }
    
    @Override
    public void execute() {
        editor.write(text);
    }
    
    @Override
    public void undo() {
        editor.delete(text.length());
    }
}

// Invoker with history
public class CommandInvoker {
    private Stack<Command> history = new Stack<>();
    
    public void execute(Command command) {
        command.execute();
        history.push(command);
    }
    
    public void undo() {
        if (!history.isEmpty()) {
            Command command = history.pop();
            command.undo();
        }
    }
}

// Usage
TextEditor editor = new TextEditor();
CommandInvoker invoker = new CommandInvoker();

invoker.execute(new WriteCommand(editor, "Hello "));
invoker.execute(new WriteCommand(editor, "World!"));
System.out.println(editor.getText());  // "Hello World!"

invoker.undo();
System.out.println(editor.getText());  // "Hello "

invoker.undo();
System.out.println(editor.getText());  // ""
```

---

## Concurrency Patterns

<a id="q16"></a>
### Q16: What is the Thread Pool pattern?
**Answer:**
Thread Pool manages a pool of worker threads to execute tasks, reducing the overhead of thread creation.

```java
// Using ExecutorService (built-in thread pool)
ExecutorService executor = Executors.newFixedThreadPool(4);

// Submit tasks
for (int i = 0; i < 10; i++) {
    final int taskId = i;
    executor.submit(() -> {
        System.out.println("Task " + taskId + " executed by " + 
            Thread.currentThread().getName());
    });
}

executor.shutdown();  // Don't forget to shutdown!
```

```java
// Custom ThreadPoolExecutor for more control
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    2,                      // Core pool size
    4,                      // Maximum pool size
    60L, TimeUnit.SECONDS,  // Keep-alive time for idle threads
    new LinkedBlockingQueue<>(100),  // Work queue
    new ThreadPoolExecutor.CallerRunsPolicy()  // Rejection policy
);

// Different rejection policies:
// - AbortPolicy: throws RejectedExecutionException
// - CallerRunsPolicy: runs in caller's thread
// - DiscardPolicy: silently discards
// - DiscardOldestPolicy: discards oldest in queue
```

<a id="q17"></a>
### Q17: What is the Future/Promise pattern?
**Answer:**
Future represents a result of an asynchronous computation that may not be available yet.

```java
// Basic Future
ExecutorService executor = Executors.newSingleThreadExecutor();

Future<String> future = executor.submit(() -> {
    Thread.sleep(1000);
    return "Result";
});

// Do other work...

String result = future.get();  // Blocks until complete
// Or with timeout
String result = future.get(2, TimeUnit.SECONDS);
```

```java
// CompletableFuture (more powerful)
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return fetchDataFromService();
});

// Chaining operations
CompletableFuture<Integer> processed = future
    .thenApply(data -> parse(data))           // Transform
    .thenApply(parsed -> calculate(parsed))    // Transform again
    .exceptionally(ex -> {                     // Handle errors
        log.error("Error", ex);
        return 0;
    });

// Combining futures
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "World");

CompletableFuture<String> combined = future1.thenCombine(future2, 
    (s1, s2) -> s1 + " " + s2);

// Wait for all
CompletableFuture.allOf(future1, future2).join();

// Wait for any
CompletableFuture.anyOf(future1, future2).join();
```

<a id="q18"></a>
### Q18: What is the Producer-Consumer pattern?
**Answer:**
Producer-Consumer decouples production of data from its consumption using a shared buffer.

```java
public class ProducerConsumer {
    private final BlockingQueue<Integer> queue;
    
    public ProducerConsumer(int capacity) {
        this.queue = new LinkedBlockingQueue<>(capacity);
    }
    
    // Producer
    public void produce(int item) throws InterruptedException {
        queue.put(item);  // Blocks if queue is full
        System.out.println("Produced: " + item);
    }
    
    // Consumer
    public int consume() throws InterruptedException {
        int item = queue.take();  // Blocks if queue is empty
        System.out.println("Consumed: " + item);
        return item;
    }
}

// Usage
ProducerConsumer pc = new ProducerConsumer(10);

// Producer thread
new Thread(() -> {
    for (int i = 0; i < 100; i++) {
        try {
            pc.produce(i);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}).start();

// Consumer threads
for (int i = 0; i < 3; i++) {
    new Thread(() -> {
        while (true) {
            try {
                pc.consume();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }
    }).start();
}
```

---

## Architectural Patterns

<a id="q19"></a>
### Q19: What is the MVC pattern?
**Answer:**
MVC (Model-View-Controller) separates an application into three components:

```
┌─────────────────────────────────────────────────┐
│                    User                          │
└────────────────────┬────────────────────────────┘
                     │ Interacts
                     ▼
┌─────────────────────────────────────────────────┐
│              Controller                          │
│  - Handles user input                           │
│  - Updates Model                                │
│  - Selects View                                 │
└────────┬────────────────────────────┬───────────┘
         │ Updates                    │ Selects
         ▼                            ▼
┌─────────────────┐         ┌─────────────────────┐
│     Model       │         │       View          │
│  - Business     │────────►│  - Displays data    │
│    logic        │ Notifies│  - UI rendering     │
│  - Data         │         │                     │
└─────────────────┘         └─────────────────────┘
```

```java
// Model
public class User {
    private String name;
    private String email;
    // getters, setters
}

// View (in Spring MVC, this could be a Thymeleaf template)
public class UserView {
    public void displayUser(User user) {
        System.out.println("Name: " + user.getName());
        System.out.println("Email: " + user.getEmail());
    }
}

// Controller (Spring MVC)
@Controller
@RequestMapping("/users")
public class UserController {
    
    @Autowired
    private UserService userService;
    
    @GetMapping("/{id}")
    public String getUser(@PathVariable Long id, Model model) {
        User user = userService.findById(id);
        model.addAttribute("user", user);  // Pass to view
        return "user/detail";              // View name
    }
    
    @PostMapping
    public String createUser(@ModelAttribute User user) {
        userService.save(user);
        return "redirect:/users/" + user.getId();
    }
}
```

<a id="q20"></a>
### Q20: What is Clean Architecture?
**Answer:**
Clean Architecture organizes code into concentric layers with dependencies pointing inward.

```
┌─────────────────────────────────────────────────────────┐
│                   Frameworks & Drivers                   │
│   (Web, UI, DB, External Services)                      │
│  ┌───────────────────────────────────────────────────┐  │
│  │              Interface Adapters                    │  │
│  │   (Controllers, Gateways, Presenters)             │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │           Application Business Rules         │  │  │
│  │  │   (Use Cases / Application Services)        │  │  │
│  │  │  ┌───────────────────────────────────────┐  │  │  │
│  │  │  │     Enterprise Business Rules         │  │  │  │
│  │  │  │   (Entities / Domain Models)          │  │  │  │
│  │  │  └───────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘

Dependencies point INWARD only!
```

```java
// Domain Layer (innermost) - no dependencies
public class Order {
    private OrderId id;
    private List<OrderItem> items;
    private OrderStatus status;
    
    public void addItem(Product product, int quantity) {
        // Business logic
    }
}

// Use Case Layer
public interface CreateOrderUseCase {
    OrderId execute(CreateOrderRequest request);
}

public class CreateOrderInteractor implements CreateOrderUseCase {
    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    
    @Override
    public OrderId execute(CreateOrderRequest request) {
        Order order = new Order(request.getCustomerId());
        // Business logic
        orderRepository.save(order);
        return order.getId();
    }
}

// Interface Adapters Layer
@RestController
public class OrderController {
    private final CreateOrderUseCase createOrderUseCase;
    
    @PostMapping("/orders")
    public ResponseEntity<OrderResponse> createOrder(@RequestBody OrderRequest request) {
        OrderId id = createOrderUseCase.execute(toCreateOrderRequest(request));
        return ResponseEntity.ok(new OrderResponse(id));
    }
}

// Infrastructure Layer (outermost)
@Repository
public class JpaOrderRepository implements OrderRepository {
    @Autowired
    private OrderJpaRepository jpaRepository;
    
    @Override
    public void save(Order order) {
        jpaRepository.save(toEntity(order));
    }
}
```

<a id="q21"></a>
### Q21: What is the Repository pattern?
**Answer:**
Repository mediates between the domain and data mapping layers, acting like an in-memory collection of domain objects.

```java
// Repository interface (in Domain layer)
public interface UserRepository {
    User findById(UserId id);
    List<User> findByEmail(String email);
    void save(User user);
    void delete(User user);
    List<User> findAll();
}

// Implementation (in Infrastructure layer)
@Repository
public class JpaUserRepository implements UserRepository {
    
    @Autowired
    private UserJpaRepository jpaRepository;
    
    @Autowired
    private UserMapper mapper;
    
    @Override
    public User findById(UserId id) {
        return jpaRepository.findById(id.getValue())
            .map(mapper::toDomain)
            .orElseThrow(() -> new UserNotFoundException(id));
    }
    
    @Override
    public void save(User user) {
        UserEntity entity = mapper.toEntity(user);
        jpaRepository.save(entity);
    }
    
    @Override
    public List<User> findByEmail(String email) {
        return jpaRepository.findByEmail(email).stream()
            .map(mapper::toDomain)
            .toList();
    }
}

// Spring Data JPA Repository (infrastructure detail)
public interface UserJpaRepository extends JpaRepository<UserEntity, Long> {
    List<UserEntity> findByEmail(String email);
}
```

**Benefits:**
- Decouples domain from persistence
- Enables easy testing with in-memory implementations
- Centralizes data access logic
- Hides query complexity from domain

---

Good luck with your interview! 🚀
