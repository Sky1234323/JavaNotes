# Spring Lecture 5 - IoC Container, Beans, DI Types, @Bean, @Primary and @Qualifier

> **How to read these notes**
> - Things from your screenshots and your own code are written as normal text.
> - Anything marked **[GK]** is *General Knowledge* that I added because the screenshots did not show it. Check these against what your lecturer said.
> - **Naming:** I write method names in normal Java style (`pay()`, `placeOrder()`, `createUser()`). Your code has `Pay()`, `PlaceOrder()`, `CreateUser()`. Any name works if it is the same everywhere. For `@Bean` methods, the method name becomes the bean name, so keep that in mind.
> - The lecturer's screenshots use `CardPayment` and `UpiPayment`. Your code uses `Card` and `UPI`. They are the same idea. I use `Card` and `UPI`.

---

## Table of Contents

1. Setup (Spring dependency)
2. The big picture: IoC container and beans
3. Bean definition and the 7 steps of the container
4. What happens behind the scenes in `new AnnotationConfigApplicationContext(Config.class)`
5. Why we pass `OrderService.class`
6. First working example: `@Component` and `@ComponentScan`
7. How `@Autowired` works internally
8. Types of Dependency Injection (constructor, setter, field)
9. Why constructor injection is recommended (with unit testing in depth)
10. `@Bean` annotation: why do we need it? (two conditions)
11. Creating beans with `@Bean` for PaymentService and OrderService
12. Two beans of the same interface: `@Primary` and `@Qualifier`
13. What if we use `@Component` and `@Bean` both on the same class?
14. Why we write `@Bean` in Config.java and not in Main.java
15. BeanFactory vs ApplicationContext
16. `@Component` vs `@Bean` summary
17. Revision checklist and questions

---

## 1. Setup (Spring dependency)

To use the Spring container, add `spring-context` to `pom.xml` (you found it on the Maven Repository website, as in Lecture 3):

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>7.0.9</version>
</dependency>
```

In your run log, the classpath also had `spring-aop`, `spring-beans`, `spring-core`, `spring-expression`, `commons-logging`, `jspecify` and two `micrometer` jars. You never added them. Maven downloaded them because `spring-context` needs them. These are **transitive dependencies** (Lecture 3). All of them are stored in `.m2/repository`, not inside your project.

---

## 2. The big picture: IoC container and beans

**Bean** = an object that is created and managed by the Spring container.

- If you write `new OrderService()` yourself, that object is **not** a bean. Spring does not know about it.
- If Spring creates it, Spring knows it, stores it, and can give it to anyone who needs it. That is a bean.

**IoC container** = the place (a "box" inside your running program) where Spring keeps all the beans.

```
+------------------------------------------------+
|                  IoC CONTAINER                 |
|                                                |
|   +----------------+     +----------------+    |
|   | PaymentService |     |  (other bean)  |    |
|   +----------------+     +----------------+    |
|   +----------------+     +----------------+    |
|   |  (other bean)  |     |  OrderService  |    |
|   +----------------+     +----------------+    |
|                                                |
+------------------------------------------------+
            IoC Container  -->  Beans
```

Why IoC? In Lecture 4 your `Main` created objects and connected them (`new PaymentService()`, `new OrderService(service)`). With many services, `Main` becomes complex. Now **Spring takes the control of creating and connecting objects**. The control is inverted from you to Spring. That is Inversion of Control.

### Two ways to tell Spring about beans (configuration)

```
        How do we configure the container?
                     |
        +------------+-------------+
        |                          |
  1. Annotation based      2. XML configuration based
     (what we use)            (older style, not used here)
```

- **Annotation based:** you mark classes and methods with annotations (`@Component`, `@Bean`, `@Configuration`...). This is what the lecture uses.
- **XML configuration based:** you list beans in an XML file. **[GK]** A tiny example looks like this: `<bean id="paymentService" class="org.example.Card"/>`. Modern projects mostly use annotations.

### Two ways to create beans with annotations

```
                 Beans
                   |
        +----------+----------+
        |                     |
   @Component              @Bean
 (on a CLASS)          (on a METHOD)
```

- `@Component` is put **on a class**. Spring finds it (by scanning) and creates the object.
- `@Bean` is put **on a method** inside a configuration class. Spring calls the method and stores the returned object.

Most of this lecture is about when to use which.

---

## 3. Bean definition and the 7 steps of the container

Before Spring creates an object, it first makes a **bean definition**. This is a "blueprint card" that describes the bean.

```
+------------------------------+
| PaymentService               |
|   Bean name  : paymentService|
|   Bean class : (the class)   |
|   Scope      : (singleton)   |
|   Dependency : no            |
+------------------------------+
        Bean Definition
```

- **Bean name:** the default name is the class name with a small first letter (`PaymentService` becomes `paymentService`).
- **Bean class:** which class Spring will create the object from.
- **Scope:** how many objects Spring makes. **[GK]** The default scope is **singleton**: one object per container, and the same object is given every time.
- **Dependency:** what this bean needs. `PaymentService` needs nothing ("no"). `OrderService` needs a `PaymentService`.

### The 7 steps (from the lecture)

```
Step 1: Spring starts the container
   |
Step 2: Spring reads AppConfig.java (the config class)
   |
Step 3: Spring processes @ComponentScan
   |
Step 4: Spring finds the @Component classes
   |
Step 5: Spring creates bean definitions (blueprints only, no objects yet)
   |
Step 6: Spring starts creating objects (beans)
   |        PaymentService payment = new PaymentService();
   |        OrderService order = new OrderService(payment);
   |
Step 7: Our application uses these beans
```

Important order inside step 6: Spring creates `PaymentService` **first**, because `OrderService` depends on it (the arrow `OrderService -> PaymentService` in the lecture diagram). Spring works out this order itself from the dependencies.

```
+----------------------------------------+
|              IoC Container             |
|   +----------------+   +-------------+ |
|   | PaymentService | <-| OrderService| |
|   +----------------+   +-------------+ |
+----------------------------------------+
   OrderService needs PaymentService, so Payment is created first
```

---

## 4. What happens behind the scenes in `new AnnotationConfigApplicationContext(Config.class)`

```java
ApplicationContext context = new AnnotationConfigApplicationContext(Config.class);
```

### Reading the line piece by piece

| Part | Meaning |
|---|---|
| `ApplicationContext` | The **type** of the variable. It is an interface, and it means "the Spring container". |
| `context` | Your variable name. Later you call `context.getBean(...)` on it. |
| `AnnotationConfigApplicationContext` | The **actual class** being created. "AnnotationConfig" means it reads configuration from annotated Java classes (not from XML). |
| `Config.class` | "Start from this configuration class." (See section 5 for `.class`.) |

### What the constructor does (all inside this one line)

```
new AnnotationConfigApplicationContext(Config.class)
        |
        v
[1] Create an empty container (a BeanFactory inside)          [GK]
        |
        v
[2] Register Config.class as the starting point               [GK]
        |
        v
[3] refresh()  -- the real work starts here                   [GK]
        |
        v
[4] Read Config: it is @Configuration
        |-- @ComponentScan -> scan the package, find @Component classes
        |                     -> make a bean definition for each
        |-- @Bean methods  -> make a bean definition for each method
        v
[5] Register helpers (post processors). One of them is the
    helper that understands @Autowired                        [GK]
        v
[6] Create ALL beans now (objects), in dependency order
        |-- call constructors (for @Component classes)
        |-- call @Bean methods
        |-- inject dependencies (@Autowired / method parameters)
        v
[7] Container is ready. The constructor finishes.
        |
        v
context.getBean(X.class)  -->  only LOOKS UP an object that already exists
```

These match the lecturer's 7 steps in section 3. Steps [4] and [5] are steps 2 to 5 there, and step [6] is his step 6.

### Proof from your own error (very important to understand)

When your `User` class failed, the stack trace had these lines:

```
at ...AnnotationConfigApplicationContext.<init>(...)
at org.example.Main.main(Main.java:10)
```

and before them these:

```
...AbstractApplicationContext.refresh(...)
...finishBeanFactoryInitialization(...)
...preInstantiateSingletons(...)
```

This tells you two things:

1. The error happened on **line 10 of Main**, which is the `new AnnotationConfigApplicationContext(...)` line, not at `getBean`.
2. `preInstantiateSingletons` means Spring creates all the (singleton) beans **at startup**, while the context is being built. So if any bean cannot be created, the **application fails right at startup**. This is good: you find the problem early. **[GK]** (This is the default behaviour, called eager creation.)

---

## 5. Why we pass `OrderService.class`

```java
OrderService order = context.getBean(OrderService.class);
```

`OrderService.class` is a **Class object**. It is the Java description of the class `OrderService`: its name, constructors, methods, annotations. It is **not** an object of `OrderService`.

```
OrderService.class   -->  "the blueprint information about OrderService"
new OrderService()   -->  "an actual OrderService object"
```

So `getBean(OrderService.class)` means: **"Container, find the bean whose type is OrderService and give it to me."**

- Spring searches the container for a bean of that type.
- It returns `OrderService` directly, so you do not need a cast.
- You never write `new OrderService(...)`. You only ask. That is IoC.

There is also a way to ask by **name**:

```java
OrderService order = (OrderService) context.getBean("orderService"); // needs a cast
```

By type is safer because the compiler checks the type for you. **[GK]**

The same idea is used in `new AnnotationConfigApplicationContext(AppConfig.class)`: "Container, use the AppConfig class as your configuration."

---

## 6. First working example: `@Component` and `@ComponentScan`

**PaymentService.java**

```java
@Component
public class PaymentService {
    public void pay() {
        System.out.println("Payment is Done!!!");
    }
}
```

**OrderService.java**

```java
@Component
public class OrderService {

    private PaymentService paymentService;

    @Autowired
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    public void placeOrder() {
        paymentService.pay();
        System.out.println("Order placed");
    }
}
```

**AppConfig.java**

```java
@Configuration
@ComponentScan("in.coderarmy")
public class AppConfig {
    // empty
}
```

**Main.java**

```java
public class Main {
    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

        OrderService order = context.getBean(OrderService.class);
        order.placeOrder();

        PaymentService payment = context.getBean(PaymentService.class);
        payment.pay();
    }
}
```

What each annotation does:

| Annotation | Put on | Meaning |
|---|---|---|
| `@Component` | a class | "Spring, make a bean from this class." |
| `@Configuration` | a class | "This class gives configuration to the container." |
| `@ComponentScan("in.coderarmy")` | a config class | "Search this package (and its sub-packages) for `@Component` classes." |
| `@Autowired` | constructor / setter / field | "Spring, fill this dependency from the container." |

> **[GK]** If you write `@ComponentScan` **without** a value (like in your own `Config.java`), Spring scans the package of the config class itself (`org.example` for you), including its sub-packages.

`Main` is now very small. It only starts the container and asks for beans. It does not connect anything by hand.

---

## 7. How `@Autowired` works internally

Using the example above:

```
new AnnotationConfigApplicationContext(AppConfig.class)
        |
        v
[1] Read AppConfig  ->  @ComponentScan("in.coderarmy")
        |
        v
[2] Scan the package, find classes with @Component
        |        found: PaymentService, OrderService
        v
[3] Make a bean definition for each
        |        paymentService : depends on nothing
        |        orderService   : depends on PaymentService
        v
[4] Create beans. Start with orderService:
        |
        |   Constructor has @Autowired, parameter type = PaymentService
        |        |
        |        v
        |   Is there a PaymentService bean in the container already?
        |        |-- yes -> use it
        |        |-- no  -> create it first: new PaymentService(), store it
        |        v
        |   Call new OrderService(paymentBean)   (Spring calls it, not you)
        |   Store the OrderService bean in the container
        v
[5] Container has both beans
        |
        v
context.getBean(OrderService.class) -> gives the ready object
```

In simple words:

1. `@Component` says "make a bean from me".
2. `@ComponentScan` says "look here for such classes".
3. `@Autowired` on the constructor says "when you create me, fill my parameters from the container".
4. Spring matches the parameter by its **type** (`PaymentService`), finds that bean, and passes it in.

**Before and after:**

```
Lecture 4 (you wire by hand)            Lecture 5 (Spring wires)
Main:                                   Main:
  new PaymentService()                    context.getBean(OrderService.class)
  new OrderService(service)               (the container already did the wiring)
```

> **Why the first version needed `@Autowired`:** Your first `OrderService` had **two** constructors (an empty one and one with `PaymentService`). Without a hint, Spring would use the empty one. `payment` would stay `null` and `placeOrder()` would throw `NullPointerException`. `@Autowired` points to the constructor Spring must use. **[GK]** When a class has only **one** constructor, newer Spring versions use it automatically even without `@Autowired`.

> **Why your `User` class gave an error with no `@Autowired`:** `User` has only one constructor, so Spring automatically tried to use it and wanted to autowire its parameters. That is exactly the problem explained in section 10.

---

## 8. Types of Dependency Injection

There are 3 ways Spring can inject a dependency.

### 1) Constructor injection

```java
@Component
public class OrderService {

    private final PaymentService paymentService;

    @Autowired
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Spring finds the dependency **first**, then calls the constructor with it.

```
find PaymentService bean --> new OrderService(paymentBean) --> object is ready and complete
```

### 2) Setter injection

```java
@Component
public class OrderService {

    private PaymentService paymentService;

    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

Spring creates the object first (empty constructor), then calls the setter.

```
new OrderService()  -->  (object exists, paymentService = null)  -->  setPaymentService(paymentBean)
```

### 3) Field injection

```java
@Component
public class OrderService {

    @Autowired
    private PaymentService paymentService;
}
```

No constructor, no setter. Spring creates the object and then puts the dependency directly into the private field. **[GK]** Spring does this using reflection (it can reach private fields).

```
new OrderService()  -->  (paymentService = null)  -->  Spring sets the private field directly
```

### Comparison

| | Constructor | Setter | Field |
|---|---|---|---|
| Dependency set when | At object creation | After creation | After creation |
| Can use `final` | Yes | No | No |
| Object can be half-ready (null dependency) | No | Yes | Yes |
| Easy to unit test without Spring | Yes | Possible | Hard |
| Code length | A bit longer | Medium | Shortest |
| Recommended | **Yes** | For optional dependencies **[GK]** | Avoid in real code |

---

## 9. Why constructor injection is recommended

The lecture gives 3 reasons:

1. Dependency gets wired at the time of object creation.
2. `final` can be used.
3. Easy to test the class.

### Reason 1: Wired at the time of object creation

With constructor injection, you **cannot create** the object without giving its dependency:

```java
OrderService order = new OrderService(paymentService);   // dependency is mandatory
```

So the object is always in a complete, ready state.

With setter or field injection, the object can exist while `paymentService` is still `null`:

```java
OrderService order = new OrderService();   // allowed
order.placeOrder();                        // NullPointerException (paymentService is null)
```

Also, the constructor shows all dependencies in one place. If a constructor has 8 parameters, you can see that this class is doing too much. **[GK]**

### Reason 2: `final` can be used

```java
private final PaymentService paymentService;
```

A `final` field must be set once in the constructor and can never change later. This gives:

- The dependency cannot be replaced by mistake.
- The object is safe to share between threads. **[GK]**

Setter and field injection need the field to be changeable after creation, so `final` is not possible.

### Reason 3: Easy to test the class (in depth)

**What is a unit test?** A unit test checks **one class alone**. For `OrderService`, we want to test only the logic of `OrderService`. We do **not** want a real payment to happen. So we replace the real `PaymentService` with a **fake** one that just records "pay was called".

To do that, we must be able to **hand our own fake to the class**. Constructor injection makes this simple.

**The class (constructor injection):**

```java
public class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    public void placeOrder() {
        paymentService.pay();
        System.out.println("Order placed");
    }
}
```

**The test (no Spring, no container):** **[GK]** (uses JUnit 5; the idea is the important part)

```java
// A fake payment, written by us
class FakePaymentService implements PaymentService {
    boolean payCalled = false;

    @Override
    public void pay() {
        payCalled = true;        // no real payment, just remember it was called
    }
}

class OrderServiceTest {

    @Test
    void placeOrder_callsPay() {
        FakePaymentService fake = new FakePaymentService();

        OrderService order = new OrderService(fake);   // we hand over the fake

        order.placeOrder();

        assertTrue(fake.payCalled);   // proves OrderService really called pay()
    }
}
```

Why this works so easily:

```
Test:  new FakePaymentService()
          |
          v
       new OrderService(fake)       <- plain Java, one line
          |
          v
       placeOrder() -> calls fake.pay() -> test checks it
```

- No Spring is started, so the test runs in milliseconds.
- No real payment, database or network is used.
- This works because `PaymentService` is an interface (the loose coupling from Lecture 4). Any class that implements it can be given.
- **[GK]** If you do not want to write a fake, a library like Mockito can create one: `PaymentService fake = Mockito.mock(PaymentService.class);`.

**Now the same class with field injection:**

```java
public class OrderService {

    @Autowired
    private PaymentService paymentService;     // private, no constructor, no setter

    public void placeOrder() {
        paymentService.pay();
        System.out.println("Order placed");
    }
}
```

```java
OrderService order = new OrderService();   // the only way to create it
order.placeOrder();                        // NullPointerException: paymentService is null
```

There is **no normal way to give the fake** to the private field. You are forced to:

- use reflection tricks or special test tools to set the private field, **[GK]** or
- start the whole Spring container just to test one small class (slow and heavy).

```
Constructor injection test :  new OrderService(fake)          -> simple, fast
Field injection test       :  new OrderService() -> null field -> needs tricks or Spring
```

**Summary of reason 3:** constructor injection lets you create the class with any dependency you like using plain Java. So the class can be tested alone, quickly, without Spring.

---

## 10. `@Bean` annotation: why do we need it? (two conditions)

`@Component` is very easy, but it **cannot be used in every situation**. There are two common situations where we need `@Bean`.

### Condition 1: The class needs values that Spring cannot provide (the `User` problem)

Your class:

```java
@Component
public class User {
    private String name;
    private int age;

    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
    // getters, setters, display() ...
}
```

**The error you got:**

```
UnsatisfiedDependencyException: Error creating bean with name 'user' ...
Unsatisfied dependency expressed through constructor parameter 0:
No qualifying bean of type 'java.lang.String' available:
expected at least 1 bean which qualifies as autowire candidate.
```

**Reading the error in simple words:**

| Part of the error | Meaning |
|---|---|
| `Error creating bean with name 'user'` | Spring tried to create the bean named `user` (from the class `User`). |
| `constructor parameter 0` | The first constructor parameter, `String name`, is the problem. |
| `No qualifying bean of type 'java.lang.String'` | Spring searched its container for a bean of type `String` and found none. |

**Why it happens (step by step):**

```
@Component on User
      |
      v
Spring creates a User bean using its constructor
      |
      v
User has ONE constructor: User(String name, int age)
Spring treats it like @Autowired and must fill both parameters
      |
      v
Spring looks in the container:  "Do I have a bean of type String?"
      |
      v
No.  -->  cannot call the constructor  -->  UnsatisfiedDependencyException
      |
      v
Failure happens at startup (new AnnotationConfigApplicationContext), the program stops
```

The main idea: **Spring injects only beans that exist in its container.** "Akash" and `20` are plain values, not beans. Spring does not invent a name or an age for you. It does not know which values to use. (After `String`, it would also complain about `int age`.)

**Fix with `@Bean`:** we create the object ourselves, with our own values, and give it to Spring.

```java
@Configuration
@ComponentScan
public class Config {

    @Bean
    public User createUser() {
        return new User("Akash", 20);
    }
}
```

```
Spring reads Config
      |
      v
Finds @Bean createUser()
      |
      v
Spring calls createUser()          <- Spring calls it, we do not
      |
      v
Inside, WE write: new User("Akash", 20)    <- we give the values
      |
      v
Returned User object is stored in the container as bean "createUser"
```

Spring no longer needs to find a `String` bean, because we are the ones who call the constructor.

> **Very important:** after using `@Bean` for `User`, **remove `@Component` from the `User` class.** If `@Component` stays, component scan still tries to create the `user` bean on its own and fails with the same error again. (Section 13 explains what happens when both are used.)

> **[GK]** Spring also has `@Value` to inject plain values into a constructor. It was not part of this lecture, so it is not explained here.

### Condition 2: The class comes from a library (JAR) and has no annotation

Your second project, in its own JAR (`CartService` in `CoreDemo3`):

```java
package org.example;

public class CartService {
    public void addToCart() {
        System.out.println("Added to Cart!!!");
    }
}
```

You ran `mvn install` on it and added it as a dependency in the main project (Lecture 3). **[GK]** The path in your run log, `in\strikes\CoreDemo3\1.0-SNAPSHOT`, shows the dependency was:

```xml
<dependency>
    <groupId>in.strikes</groupId>
    <artifactId>CoreDemo3</artifactId>
    <version>1.0-SNAPSHOT</version>
</dependency>
```

**Problem:** the class is in the project, but Spring does not manage it.

```
@ComponentScan looks for classes marked with @Component (or similar)
      |
      v
CartService has NO @Component
      |
      v
Spring never makes a bean definition for it
      |
      v
context.getBean(CartService.class)  -->  NoSuchBeanDefinitionException   [GK]
```

**Why not just add `@Component` to `CartService`?**

- In real life, library classes come from other people or companies (for example a database client, a JSON mapper). You **cannot edit their code**. Their JAR is already compiled.
- A general library should not depend on Spring annotations anyway. **[GK]**

**Fix with `@Bean`:** we tell Spring how to create it, in our own config class.

```java
@Configuration
@ComponentScan
public class Config {

    @Bean
    public CartService createCartService() {
        return new CartService();
    }
}
```

```
Config has @Bean createCartService()
      |
      v
Spring calls it, gets a CartService object, stores it as a bean
      |
      v
context.getBean(CartService.class).addToCart();   // works
```

### Summary: when do we need `@Bean`?

```
Can I add @Component to the class AND can Spring create it alone?
      |
      |-- Yes -> use @Component
      |
      |-- No  -> use @Bean
              Reasons:
                1) the class needs values/special setup (like User("Akash", 20))
                2) the class is from a library/JAR and I cannot edit it (like CartService)
```

---

## 11. Creating beans with `@Bean` for PaymentService and OrderService

Setup: `PaymentService` is now an **interface**. `Card` and `UPI` implement it. In this version these classes have **no `@Component`** and `OrderService` has **no `@Component`**. Spring creates all of them through the `@Bean` methods in `Config`.

```java
public interface PaymentService {
    void pay();
}

public class Card implements PaymentService {
    @Override
    public void pay() { System.out.println("Via Card"); }
}

public class UPI implements PaymentService {
    @Override
    public void pay() { System.out.println("Via UPI"); }
}

public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    public void placeOrder() {
        paymentService.pay();
        System.out.println("Order placed");
    }
}
```

**Config.java (from the lecture screenshot, using one payment first):**

```java
@Configuration
@ComponentScan
public class Config {

    @Bean
    public PaymentService createCardPayment() {
        return new Card();
    }

    @Bean
    public OrderService createOrderService(PaymentService paymentService) {
        return new OrderService(paymentService);
    }
}
```

### The confusing part: how is `OrderService` created?

Two questions usually come to mind:

1. "Who gives the value of `paymentService` in `createOrderService(PaymentService paymentService)`?"
2. "Who calls `createOrderService`? I never call it."

**Answers:**

1. **Spring** gives it. A parameter of a `@Bean` method is filled from the container, by type, just like `@Autowired`.
2. **Spring** calls it. You never call `@Bean` methods yourself. Spring calls each one once and stores what it returns.

**Step by step:**

```
Step 1: Spring reads Config and finds 2 @Bean methods.
        It makes 2 bean definitions (blueprints):

        +-----------------------------+   +---------------------------------------+
        | bean name : createCardPayment|  | bean name : createOrderService         |
        | type      : PaymentService   |   | type      : OrderService               |
        | needs     : nothing          |   | needs     : a PaymentService           |
        +-----------------------------+   +---------------------------------------+

Step 2: Spring wants to create createOrderService.
        It sees the parameter (PaymentService) and thinks:
        "I need a PaymentService bean first."

Step 3: Spring creates createCardPayment first:
        it calls createCardPayment()  ->  returns new Card()  ->  stored in the container.

Step 4: Spring calls createOrderService( <the Card bean> )
        The parameter paymentService now holds the Card object.
        Inside the method:  return new OrderService(paymentService);
        The returned OrderService is stored in the container.

Step 5: Container holds:  [ Card as PaymentService ]  and  [ OrderService (holding the Card) ]
```

```
                createCardPayment()                 createOrderService(paymentService)
                       |                                        ^
                       | returns new Card()                     | Spring passes the Card bean here
                       v                                        |
                  +-----------+  ---- found by TYPE ---->  +-------------+
                  | Card bean |                            | OrderService|
                  +-----------+                            +-------------+
```

**Easy way to remember:** `Config` is the Lecture 4 `Main` moved into a class that **Spring runs**:

```
Lecture 4 Main (you ran it):           Lecture 5 Config (Spring runs it):
  PaymentService p = new Card();         @Bean createCardPayment()  { return new Card(); }
  OrderService o = new OrderService(p);  @Bean createOrderService(p) { return new OrderService(p); }
```

Even if you write `createOrderService` above `createCardPayment` in the file, Spring still creates the payment first, because it works out the order from the dependencies. **[GK]**

**Using it in Main:**

```java
ApplicationContext context = new AnnotationConfigApplicationContext(Config.class);

OrderService order = context.getBean(OrderService.class);
order.placeOrder();
```

Output:

```
Via Card
Order placed
```

> **Naming:** the bean names here are the method names: `createCardPayment` and `createOrderService`. **[GK]** A bean created by `@Bean` is also a singleton by default.

> **Note on `OrderService` itself:** we did not have to use `@Bean` for it (it is our own class and `@Component` would work). The lecturer shows it with `@Bean` to show how a bean that **depends on another bean** is built. Just remember: a class should be created **one way only**, either `@Component` or `@Bean` (section 13).

---

## 12. Two beans of the same interface: `@Primary` and `@Qualifier`

Now add a second payment type:

```java
@Configuration
@ComponentScan
public class Config {

    @Bean
    public PaymentService createCardPayment() {
        return new Card();
    }

    @Bean
    public PaymentService createUpiPayment() {
        return new UPI();
    }

    @Bean
    public OrderService createOrderService(PaymentService paymentService) {   // red line in the IDE!
        return new OrderService(paymentService);
    }
}
```

The IDE shows a red underline on `paymentService` (as in your screenshot). Reason:

```
createOrderService needs ONE PaymentService
        |
        v
Container has TWO beans of type PaymentService:
        |-- createCardPayment  (Card)
        |-- createUpiPayment   (UPI)
        v
Spring cannot decide which one to pass  -->  confusion
        v
NoUniqueBeanDefinitionException: expected single matching bean but found 2:
createCardPayment, createUpiPayment                                     [GK]
```

The same problem happens with `@Component`: if both `Card` and `UPI` have `@Component` and implement `PaymentService`, then `OrderService(PaymentService)` is ambiguous.

We can solve this in two ways.

### Solution 1: `@Primary` = "the default choice"

```java
@Bean
@Primary
public PaymentService createCardPayment() {
    return new Card();
}

@Bean
public PaymentService createUpiPayment() {
    return new UPI();
}
```

```
Two candidates: Card (marked @Primary), UPI
        |
        v
Spring: "Card is primary, use it when nobody asks for a specific one"
        |
        v
OrderService gets Card
```

With `@Component` classes, put `@Primary` on the class:

```java
@Component
@Primary
public class Card implements PaymentService { ... }
```

(In your `UPI.java` screenshot, `@Primary` was written and then commented out. That is the lecturer trying this.)

**Only one bean should be `@Primary` for a type.** **[GK]** If two are primary, Spring is confused again.

### Solution 2: `@Qualifier` = "choose by name"

**Step 1: give each bean a name (label).**
**Step 2: at the place where you inject, say which name you want.**

```java
@Bean
@Qualifier("card")
public PaymentService createCardPayment() {
    return new Card();
}

@Bean
@Qualifier("upi")
public PaymentService createUpiPayment() {
    return new UPI();
}

@Bean
public OrderService createOrderService(@Qualifier("upi") PaymentService paymentService) {
    return new OrderService(paymentService);   // gets UPI
}
```

```
createOrderService( @Qualifier("upi") PaymentService p )
        |
        v
Spring filters the 2 candidates by the label "upi"
        |
        v
Only createUpiPayment matches  -->  p = UPI bean
```

With `@Component` classes:

```java
@Component
@Qualifier("card")
public class Card implements PaymentService { ... }

@Component
@Qualifier("upi")
public class UPI implements PaymentService { ... }
```

and with constructor injection:

```java
@Autowired
public OrderService(@Qualifier("upi") PaymentService paymentService) {
    this.paymentService = paymentService;
}
```

> **Note about your screenshots:** `@Qualifier` was written **without a name** on `Card` and `UPI`. A qualifier with no name cannot tell the two apart. The name inside the brackets is what makes it work: `@Qualifier("card")`.

> **[GK]** If you do not label the beans, `@Qualifier` can also match the **bean name**, for example `@Qualifier("createUpiPayment")`. You can also name a component directly with `@Component("upi")`.

### `@Primary` vs `@Qualifier`

| | `@Primary` | `@Qualifier` |
|---|---|---|
| Meaning | "This is the **default** choice" | "I want **this specific** one" |
| Decided at | The bean (once) | The place where it is injected |
| Good when | One option is used most of the time | Different places need different options |
| If both are used | `@Qualifier` **wins** over `@Primary` **[GK]** | |

```
Many beans of the same type?
      |
      |-- One is the usual choice          -> @Primary on it
      |-- Each place needs a different one -> @Qualifier("name") where it is injected
```

---

## 13. What if we use `@Component` and `@Bean` both on the same class?

Example: `Card` has `@Component`, and `Config` also has a `@Bean` method that returns `new Card()`.

```java
@Component
public class Card implements PaymentService { ... }
```

```java
@Configuration
@ComponentScan
public class Config {

    @Bean
    public PaymentService createCardPayment() {
        return new Card();
    }
}
```

**What happens inside (Spring does both jobs):**

```
                    Container starts
                          |
        +-----------------+------------------+
        |                                    |
  @ComponentScan                       @Bean method
  finds Card with @Component           createCardPayment() in Config
        |                                    |
        v                                    v
  bean definition:                     bean definition:
  name = "card"                        name = "createCardPayment"
        |                                    |
        v                                    v
  Card object #1                       Card object #2

        Container now has TWO beans of type PaymentService
```

- The two beans have **different names** (`card` and `createCardPayment`), so Spring treats them as two separate beans.
- You get **two Card objects**. This wastes memory and is confusing.

**Will it execute?**

- It **starts** without an error if nobody asks for a `PaymentService` by type.
- But as soon as something needs **one** `PaymentService` (for example `createOrderService(PaymentService p)`), Spring finds 2 candidates and throws `NoUniqueBeanDefinitionException` (same problem as section 12). **[GK]**
- You could hide the problem with `@Primary` or `@Qualifier`, but the real mistake is creating the same class in two ways.

> **[GK]** If you deliberately give both the **same bean name** (for example `@Component("card")` and a `@Bean` method named `card`), then they are not two beans: one definition replaces the other. Which one wins is hard to predict, and Spring Boot refuses this by default. Do not rely on it.

**Rule to remember:**

```
One class  ->  ONE way of creating its bean.
   Your own class that Spring can create alone  -> @Component
   Library class / needs custom values          -> @Bean
```

This is also why, in section 10, `@Component` must be removed from `User` once `@Bean` creates it.

---

## 14. Why we write `@Bean` in Config.java and not in Main.java

**[GK]** The screenshots do not show the lecturer's exact explanation. These are the standard reasons.

**Reason 1: `Main` runs before the container exists. It is the one that creates the container.**

```
Main.main() starts
    |
    v
new AnnotationConfigApplicationContext(Config.class)   <- container is created here, reads Config
    |
    v
container ready
```

Spring is told to read **`Config.class`**. It reads the annotations (`@Configuration`, `@ComponentScan`, `@Bean`) from the class you give it. `Main` is just the starting point of the Java program, and Spring is never told to read it.

**Reason 2: Main would become complex again.** The whole reason we moved to Spring was to keep `Main` small (Lecture 4's problem). If all bean-creating code sits in `Main`, we are back to a crowded `Main`.

**Reason 3: Separation of responsibilities.**

```
Main    -> starts the application, asks for beans
Config  -> describes which beans exist and how to create them
```

**Reason 4: Reuse and testing.** A separate `Config` class can be swapped. For example, a test can use a different config with fake beans, while the real app uses the real config. Config can also be split into many classes as the project grows.

---

## 15. BeanFactory vs ApplicationContext

The lecture ends with this drawing: a small circle (**BeanFactory**) inside a bigger circle (**ApplicationContext**).

```
        +--------------------------------+
        |        ApplicationContext      |
        |     +--------------------+     |
        |     |    BeanFactory     |     |
        |     | (basic container)  |     |
        |     +--------------------+     |
        |   + extra features on top      |
        +--------------------------------+
```

**[GK]** (details are not in the screenshots)

| | BeanFactory | ApplicationContext |
|---|---|---|
| What it is | The basic container: creates beans and does DI (`getBean`) | Built on top of BeanFactory, with more features |
| Extra features | Few | Reads annotation config, events, messages, resource loading, easy integration with other Spring parts |
| Creates beans | When they are first asked for (lazy) | At startup (eager, as seen in section 4) |
| Used in practice | Rarely directly | **Yes, always** |

So `ApplicationContext` = BeanFactory + more. In your code, `AnnotationConfigApplicationContext` is a kind of `ApplicationContext`.

---

## 16. `@Component` vs `@Bean` summary

| | `@Component` | `@Bean` |
|---|---|---|
| Placed on | A **class** | A **method** (inside a `@Configuration` class) |
| Who creates the object | Spring, using the constructor | **You**, inside the method (`new ...`) |
| How Spring finds it | By **scanning** (`@ComponentScan`) | By reading the config class |
| Dependencies | `@Autowired` (constructor/setter/field) | Method **parameters** (filled by Spring) |
| Use for | Your own classes that Spring can create alone | Library/JAR classes, classes needing custom values, special setup |
| Bean name (default) | Class name with a small first letter (`orderService`) | The **method name** (`createOrderService`) |
| Example | `PaymentService`, `OrderService` | `User("Akash", 20)`, `CartService` from the JAR |

---

## 17. Revision checklist and questions

### Quick points

- A **bean** is an object created and managed by the Spring container.
- The **IoC container** stores beans and wires them together.
- The container reads configuration from annotations (or XML). We use annotations.
- Ways to create a bean: `@Component` (on a class) and `@Bean` (on a method).
- Spring first makes **bean definitions** (name, class, scope, dependency), then creates the objects.
- `new AnnotationConfigApplicationContext(Config.class)` starts the container and creates all beans at startup.
- `X.class` is a Class object that says "this type". `getBean(X.class)` gets the bean of that type.
- `@Autowired` tells Spring to fill a dependency from the container, matching by type.
- 3 DI types: constructor, setter, field. **Constructor injection is recommended.**
- Constructor injection: wired at creation, `final` possible, easy to unit test with a fake.
- No bean of a needed type means `NoSuchBeanDefinitionException` / `UnsatisfiedDependencyException`. Two beans of the same type means confusion (`NoUniqueBeanDefinitionException`).
- Fix the confusion with `@Primary` (default) or `@Qualifier("name")` (specific).
- Use `@Bean` when the class needs custom values or comes from a library you cannot edit.
- Never create the same class with both `@Component` and `@Bean`.
- `@Bean` goes in a `@Configuration` class, not in `Main`.
- `ApplicationContext` is `BeanFactory` plus more features.

### Questions for revision

1. What is a bean, and what is the IoC container?
2. What are the two ways to configure the container? Which one do we use?
3. What is a bean definition? Name the 4 things in it.
4. Write the 7 steps Spring follows when the container starts.
5. What happens inside `new AnnotationConfigApplicationContext(Config.class)`? At which moment are beans created?
6. What does `OrderService.class` mean? Why is it passed to `getBean`?
7. How does `@Autowired` find what to inject?
8. Name the 3 types of DI and show each with code.
9. Give 3 reasons why constructor injection is recommended. Explain the unit testing reason with an example.
10. Why did the `User` class (with `String name, int age`) fail with `@Component`? How does `@Bean` fix it?
11. Why can't Spring manage `CartService` from a JAR automatically, and how do we fix it?
12. In `createOrderService(PaymentService paymentService)`, who calls the method and who fills the parameter?
13. What happens if two `PaymentService` beans exist? How do `@Primary` and `@Qualifier` solve it? Which one wins if both are used?
14. What happens if a class has both `@Component` and a `@Bean` method? Will it run?
15. Why do we write `@Bean` methods in `Config` and not in `Main`?
16. What is the difference between `BeanFactory` and `ApplicationContext`?

### Next topics

Send Lecture 6 screenshots when you are ready.
