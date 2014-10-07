# Motivation.md

### Motivation
Wiring everything together is a tedious part of application development. There are several approaches to connect data, service, and presentation classes to one another. To contrast these approaches, we'll write the billing code for a pizza ordering website:
```java
public interface BillingService {

  /**
   * Attempts to charge the order to the credit card. Both successful and
   * failed transactions will be recorded.
   *
   * @return a receipt of the transaction. If the charge was successful, the
   *      receipt will be successful. Otherwise, the receipt will contain a
   *      decline note describing why the charge failed.
   */
  Receipt chargeOrder(PizzaOrder order, CreditCard creditCard);
}
```
Along with the implementation, we'll write unit tests for our code. In the tests we need a `FakeCreditCardProcessor` to avoid charging a real credit card!

### Direct constructor calls
Here's what the code looks like when we just `new` up the credit card processor and transaction logger:
```java
public class RealBillingService implements BillingService {
  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    CreditCardProcessor processor = new PaypalCreditCardProcessor();
    TransactionLog transactionLog = new DatabaseTransactionLog();

    try {
      ChargeResult result = processor.charge(creditCard, order.getAmount());
      transactionLog.logChargeResult(result);

      return result.wasSuccessful()
          ? Receipt.forSuccessfulCharge(order.getAmount())
          : Receipt.forDeclinedCharge(result.getDeclineMessage());
     } catch (UnreachableException e) {
      transactionLog.logConnectException(e);
      return Receipt.forSystemFailure(e.getMessage());
    }
  }
}
```
This code poses problems for modularity and testability. The direct, compile-time dependency on the real credit card processor means that testing the code will charge a credit card! It's also awkward to test what happens when the charge is declined or when the service is unavailable.

### Factories
A factory class decouples the client and implementing class. A simple factory uses static methods to get and set mock implementations for interfaces. A factory is implemented with some boilerplate code:
```java
public class CreditCardProcessorFactory {
  
  private static CreditCardProcessor instance;
  
  public static void setInstance(CreditCardProcessor creditCardProcessor) {
    instance = creditCardProcessor;
  }

  public static CreditCardProcessor getInstance() {
    if (instance == null) {
      return new SquareCreditCardProcessor();
    }
    
    return instance;
  }
}
```
In our client code, we just replace the `new` calls with factory lookups:
```java
public class RealBillingService implements BillingService {
  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    CreditCardProcessor processor = CreditCardProcessorFactory.getInstance();
    TransactionLog transactionLog = TransactionLogFactory.getInstance();

    try {
      ChargeResult result = processor.charge(creditCard, order.getAmount());
      transactionLog.logChargeResult(result);

      return result.wasSuccessful()
          ? Receipt.forSuccessfulCharge(order.getAmount())
          : Receipt.forDeclinedCharge(result.getDeclineMessage());
     } catch (UnreachableException e) {
      transactionLog.logConnectException(e);
      return Receipt.forSystemFailure(e.getMessage());
    }
  }
}
```
The factory makes it possible to write a proper unit test:
```java
public class RealBillingServiceTest extends TestCase {

  private final PizzaOrder order = new PizzaOrder(100);
  private final CreditCard creditCard = new CreditCard("1234", 11, 2010);

  private final InMemoryTransactionLog transactionLog = new InMemoryTransactionLog();
  private final FakeCreditCardProcessor creditCardProcessor = new FakeCreditCardProcessor();

  @Override public void setUp() {
    TransactionLogFactory.setInstance(transactionLog);
    CreditCardProcessorFactory.setInstance(creditCardProcessor);
  }

  @Override public void tearDown() {
    TransactionLogFactory.setInstance(null);
    CreditCardProcessorFactory.setInstance(null);
  }

  public void testSuccessfulCharge() {
    RealBillingService billingService = new RealBillingService();
    Receipt receipt = billingService.chargeOrder(order, creditCard);

    assertTrue(receipt.hasSuccessfulCharge());
    assertEquals(100, receipt.getAmountOfCharge());
    assertEquals(creditCard, creditCardProcessor.getCardOfOnlyCharge());
    assertEquals(100, creditCardProcessor.getAmountOfOnlyCharge());
    assertTrue(transactionLog.wasSuccessLogged());
  }
}
```
This code is clumsy. A global variable holds the mock implementation, so we need to be careful about setting it up and tearing it down. Should the `tearDown` fail, the global variable continues to point at our test instance. This could cause problems for other tests. It also prevents us from running multiple tests in parallel.

But the biggest problem is that the dependencies are *hidden in the code*. If we add a dependency on a `CreditCardFraudTracker`, we have to re-run the tests to find out which ones will break. Should we forget to initialize a factory for a production service, we don't find out until a charge is attempted. As the application grows, babysitting factories becomes a growing drain on productivity.

Quality problems will be caught by QA or acceptance tests. That may be sufficient, but we can certainly do better.

### Dependency Injection
Like the factory, dependency injection is just a design pattern. The core principal is to *separate behaviour from dependency resolution*. In our example, the `RealBillingService` is not responsible for looking up the `TransactionLog` and `CreditCardProcessor`. Instead, they're passed in as constructor parameters:
```java
public class RealBillingService implements BillingService {
  private final CreditCardProcessor processor;
  private final TransactionLog transactionLog;

  public RealBillingService(CreditCardProcessor processor, 
      TransactionLog transactionLog) {
    this.processor = processor;
    this.transactionLog = transactionLog;
  }

  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    try {
      ChargeResult result = processor.charge(creditCard, order.getAmount());
      transactionLog.logChargeResult(result);

      return result.wasSuccessful()
          ? Receipt.forSuccessfulCharge(order.getAmount())
          : Receipt.forDeclinedCharge(result.getDeclineMessage());
     } catch (UnreachableException e) {
      transactionLog.logConnectException(e);
      return Receipt.forSystemFailure(e.getMessage());
    }
  }
}
```
We don't need any factories, and we can simplify the testcase by removing the `setUp` and `tearDown` boilerplate:
```java
public class RealBillingServiceTest extends TestCase {

  private final PizzaOrder order = new PizzaOrder(100);
  private final CreditCard creditCard = new CreditCard("1234", 11, 2010);

  private final InMemoryTransactionLog transactionLog = new InMemoryTransactionLog();
  private final FakeCreditCardProcessor creditCardProcessor = new FakeCreditCardProcessor();

  public void testSuccessfulCharge() {
    RealBillingService billingService
        = new RealBillingService(creditCardProcessor, transactionLog);
    Receipt receipt = billingService.chargeOrder(order, creditCard);

    assertTrue(receipt.hasSuccessfulCharge());
    assertEquals(100, receipt.getAmountOfCharge());
    assertEquals(creditCard, creditCardProcessor.getCardOfOnlyCharge());
    assertEquals(100, creditCardProcessor.getAmountOfOnlyCharge());
    assertTrue(transactionLog.wasSuccessLogged());
  }
}
```
Now, whenever we add or remove dependencies, the compiler will remind us what tests need to be fixed. The dependency is *exposed in the API signature*.

Unfortunately, now the clients of `BillingService` need to lookup its dependencies. We can fix some of these by applying the pattern again! Classes that depend on it can accept a `BillingService` in their constructor. For top-level classes, it's useful to have a framework. Otherwise you'll need to construct dependencies recursively when you need to use a service:
```java
  public static void main(String[] args) {
    CreditCardProcessor processor = new PaypalCreditCardProcessor();
    TransactionLog transactionLog = new DatabaseTransactionLog();
    BillingService billingService
        = new RealBillingService(creditCardProcessor, transactionLog);
    ...
  }
```

### Dependency Injection with Guice
The dependency injection pattern leads to code that's modular and testable, and Guice makes it easy to write. To use Guice in our billing example, we first need to tell it how to map our interfaces to their implementations. This configuration is done in a Guice module, which is any Java class that implements the `Module` interface:
```java
public class BillingModule extends AbstractModule {
  @Override 
  protected void configure() {
    bind(TransactionLog.class).to(DatabaseTransactionLog.class);
    bind(CreditCardProcessor.class).to(PaypalCreditCardProcessor.class);
    bind(BillingService.class).to(RealBillingService.class);
  }
}
```
We add `@Inject` to `RealBillingService`'s constructor, which directs Guice to use it. Guice will inspect the annotated constructor, and lookup values for each of parameter.
```java
public class RealBillingService implements BillingService {
  private final CreditCardProcessor processor;
  private final TransactionLog transactionLog;

  @Inject
  public RealBillingService(CreditCardProcessor processor,
      TransactionLog transactionLog) {
    this.processor = processor;
    this.transactionLog = transactionLog;
  }

  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    try {
      ChargeResult result = processor.charge(creditCard, order.getAmount());
      transactionLog.logChargeResult(result);

      return result.wasSuccessful()
          ? Receipt.forSuccessfulCharge(order.getAmount())
          : Receipt.forDeclinedCharge(result.getDeclineMessage());
     } catch (UnreachableException e) {
      transactionLog.logConnectException(e);
      return Receipt.forSystemFailure(e.getMessage());
    }
  }
}
```
Finally, we can put it all together. The `Injector` can be used to get an instance of any of the bound classes.
```java
  public static void main(String[] args) {
    Injector injector = Guice.createInjector(new BillingModule());
    BillingService billingService = injector.getInstance(BillingService.class);
    ...
  }
```
[[Getting started|GettingStarted]] explains how this all works.
# GettingStarted.md

_How to start doing dependency injection with Guice._
### Getting Started
With dependency injection, objects accept dependencies in their constructors. To construct an object, you first build its dependencies. But to build each dependency, you need _its_ dependencies, and so on. So when you build an object, you really need to build an *object graph*. 

Building object graphs by hand is labour intensive, error prone, and makes testing difficult. Instead, Guice can build the object graph for you. But first, Guice needs to be configured to build the graph exactly as you want it. 

To illustrate, we'll start the `BillingService` class that accepts its dependent interfaces `CreditCardProcessor` and `TransactionLog` in its constructor. To make it explicit that the `BillingService` constructor is invoked by Guice, we add the `@Inject` annotation:
```java
class BillingService {
  private final CreditCardProcessor processor;
  private final TransactionLog transactionLog;

  @Inject
  BillingService(CreditCardProcessor processor, 
      TransactionLog transactionLog) {
    this.processor = processor;
    this.transactionLog = transactionLog;
  }

  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    ...
  }
}
```
We want to build a `BillingService` using `PaypalCreditCardProcessor` and `DatabaseTransactionLog`. Guice uses *bindings* to map types to their implementations. A *module* is a collection of bindings specified using fluent, English-like method calls:
```java
public class BillingModule extends AbstractModule {
  @Override 
  protected void configure() {

     /*
      * This tells Guice that whenever it sees a dependency on a TransactionLog,
      * it should satisfy the dependency using a DatabaseTransactionLog.
      */
    bind(TransactionLog.class).to(DatabaseTransactionLog.class);

     /*
      * Similarly, this binding tells Guice that when CreditCardProcessor is used in
      * a dependency, that should be satisfied with a PaypalCreditCardProcessor.
      */
    bind(CreditCardProcessor.class).to(PaypalCreditCardProcessor.class);
  }
}
```
The modules are the building blocks of an *injector*, which is Guice's object-graph builder. First we create the injector, and then we can use that to build the `BillingService`:
```java
 public static void main(String[] args) {
    /*
     * Guice.createInjector() takes your Modules, and returns a new Injector
     * instance. Most applications will call this method exactly once, in their
     * main() method.
     */
    Injector injector = Guice.createInjector(new BillingModule());

    /*
     * Now that we've got the injector, we can build objects.
     */
    BillingService billingService = injector.getInstance(BillingService.class);
    ...
  }
```
By building the billingService, we've constructed a small object graph using Guice. The graph contains the billing service and its dependent credit card processor and transaction log.
# Bindings.md

_Overview of bindings in Guice_
### Bindings
The injector's job is to assemble graphs of objects. You request an instance of a given type, and it figures out what to build, resolves dependencies, and wires everything together. To specify how dependencies are resolved, configure your injector with bindings.

#### Creating Bindings
To create bindings, extend `AbstractModule` and override its `configure` method. In the method body, call `bind()` to specify each binding. These methods are type checked so the compiler can report errors if you use the wrong types. Once you've created your modules, pass them as arguments to `Guice.createInjector()` to build an injector.

Use modules to create [[linked bindings|LinkedBindings]], [[instance bindings|InstanceBindings]], [[@Provides methods|ProvidesMethods]], [[provider bindings|ProviderBindings]], [[constructor bindings|ToConstructorBindings]] and [[untargetted bindings|UntargettedBindings]].

#### More Bindings
In addition to the bindings you specify the injector includes [[built-in bindings|BuiltInBindings]]. When a dependency is requested but not found it attempts to create a [[just-in-time binding|JustInTimeBindings]]. The injector also includes bindings for the [[providers|InjectingProviders]] of its other bindings.
# LinkedBindings.md

### Linked Bindings
Linked bindings map a type to its implementation. This example maps the interface `TransactionLog` to the implementation `DatabaseTransactionLog`:
```java
public class BillingModule extends AbstractModule {
  @Override 
  protected void configure() {
    bind(TransactionLog.class).to(DatabaseTransactionLog.class);
  }
}
```
Now, when you call `injector.getInstance(TransactionLog.class)`, or when the injector encounters a dependency on `TransactionLog`, it will use a `DatabaseTransactionLog`. Link from a type to any of its subtypes, such as an implementing class or an extending class. You can even link the concrete `DatabaseTransactionLog` class to a subclass:
```java
    bind(DatabaseTransactionLog.class).to(MySqlDatabaseTransactionLog.class);
```
Linked bindings can also be chained:
```java
public class BillingModule extends AbstractModule {
  @Override 
  protected void configure() {
    bind(TransactionLog.class).to(DatabaseTransactionLog.class);
    bind(DatabaseTransactionLog.class).to(MySqlDatabaseTransactionLog.class);
  }
}
```
In this case, when a `TransactionLog` is requested, the injector will return a `MySqlDatabaseTransactionLog`.
# BindingAnnotations.md

### Binding Annotations
Occasionally you'll want multiple bindings for a same type. For example, you might want both a PayPal credit card processor and a Google Checkout processor. To enable this, bindings support an optional *binding annotation*. The annotation and type together uniquely identify a binding. This pair is called a *key*.

Defining a binding annotation requires two lines of code plus several imports. Put this in its own `.java` file or inside the type that it annotates. 
```java
package example.pizza;

import com.google.inject.BindingAnnotation;
import java.lang.annotation.Target;
import java.lang.annotation.Retention;
import static java.lang.annotation.RetentionPolicy.RUNTIME;
import static java.lang.annotation.ElementType.PARAMETER;
import static java.lang.annotation.ElementType.FIELD;
import static java.lang.annotation.ElementType.METHOD;

@BindingAnnotation @Target({ FIELD, PARAMETER, METHOD }) @Retention(RUNTIME)
public @interface PayPal {}
```
You don't need to understand all of these meta-annotations, but if you're curious:
  * `@BindingAnnotation` tells Guice that this is a binding annotation.  Guice will produce an error if ever multiple binding annotations apply to the same member.
  * `@Target({FIELD, PARAMETER, METHOD})` is a courtesy to your users. It prevents `@PayPal` from being accidentally being applied where it serves no purpose.
  * `@Retention(RUNTIME)` makes the annotation available at runtime.

To depend on the annotated binding, apply the annotation to the injected parameter:
```java
public class RealBillingService implements BillingService {

  @Inject
  public RealBillingService(@PayPal CreditCardProcessor processor,
      TransactionLog transactionLog) {
    ...
  }
```
Lastly we create a binding that uses the annotation. This uses the optional `annotatedWith` clause in the `bind()` statement:
```java
    bind(CreditCardProcessor.class)
        .annotatedWith(PayPal.class)
        .to(PayPalCreditCardProcessor.class);
```


### @Named
Guice comes with a built-in binding annotation `@Named` that uses a string:
```java
public class RealBillingService implements BillingService {

  @Inject
  public RealBillingService(@Named("Checkout") CreditCardProcessor processor,
      TransactionLog transactionLog) {
    ...
  }
```
To bind a specific name, use `Names.named()` to create an instance to pass to `annotatedWith`:
```java
    bind(CreditCardProcessor.class)
        .annotatedWith(Names.named("Checkout"))
        .to(CheckoutCreditCardProcessor.class);
```
Since the compiler can't check the string, we recommend using `@Named` sparingly.


### Binding Annotations with Attributes
Guice supports binding annotations that have attribute values. In the rare case that you need such an annotation:
  1. Create the annotation `@interface`.
  2. Create a class that implements the annotation interface. Follow the guidelines for `equals()` and `hashCode()` specified in the [Annotation Javadoc](http://java.sun.com/javase/6/docs/api/java/lang/annotation/Annotation.html). Pass an instance of this to the `annotatedWith()` binding clause.
# InstanceBindings.md

### Instance Bindings
You can bind a type to a specific instance of that type. This is usually only useful only for objects that don't have dependencies of their own, such as value objects:
```java
    bind(String.class)
        .annotatedWith(Names.named("JDBC URL"))
        .toInstance("jdbc:mysql://localhost/pizza");
    bind(Integer.class)
        .annotatedWith(Names.named("login timeout seconds"))
        .toInstance(10);
```
Avoid using `.toInstance` with objects that are complicated to create, since it can slow down application startup. You can use an `@Provides` method instead.
# ProvidesMethods.md

### @Provides Methods
When you need code to create an object, use an `@Provides` method. The method must be defined within a module, and it must have an `@Provides` annotation. The method's return type is the bound type. Whenever the injector needs an instance of that type, it will invoke the method. 
```java
public class BillingModule extends AbstractModule {
  @Override
  protected void configure() {
    ...
  }

  @Provides
  TransactionLog provideTransactionLog() {
    DatabaseTransactionLog transactionLog = new DatabaseTransactionLog();
    transactionLog.setJdbcUrl("jdbc:mysql://localhost/pizza");
    transactionLog.setThreadPoolSize(30);
    return transactionLog;
  }
}
```
If the `@Provides` method has a binding annotation like `@PayPal` or `@Named("Checkout")`, Guice binds the annotated type. Dependencies can be passed in as parameters to the method. The injector will exercise the bindings for each of these before invoking the method.
```java
  @Provides @PayPal
  CreditCardProcessor providePayPalCreditCardProcessor(
      @Named("PayPal API key") String apiKey) {
    PayPalCreditCardProcessor processor = new PayPalCreditCardProcessor();
    processor.setApiKey(apiKey);
    return processor;
  }
```

### Throwing Exceptions
Guice does not allow exceptions to be thrown from Providers.  Exceptions thrown by `@Provides` methods will be wrapped in a `ProvisionException`.  It is bad practice to allow any kind of exception to be thrown -- runtime or checked -- from an `@Provides` method.  If you need to throw an exception for some reason, you may want to use the [[ThrowingProviders extension|ThrowingProviders]] `@CheckedProvides` methods.
# ProviderBindings.md

### Provider Bindings
When your `@Provides` methods start to grow complex, you may consider moving them to a class of their own. The provider class implements Guice's `Provider` interface, which is a simple, general interface for supplying values:
```java
public interface Provider<T> {
  T get();
}
```
Our provider implementation class has dependencies of its own, which it receives via its `@Inject`-annotated constructor. It implements the `Provider` interface to define what's returned with complete type safety:
```java
public class DatabaseTransactionLogProvider implements Provider<TransactionLog> {
  private final Connection connection;

  @Inject
  public DatabaseTransactionLogProvider(Connection connection) {
    this.connection = connection;
  }

  public TransactionLog get() {
    DatabaseTransactionLog transactionLog = new DatabaseTransactionLog();
    transactionLog.setConnection(connection);
    return transactionLog;
  }
}
```
Finally we bind to the provider using the `.toProvider` clause:
```java
public class BillingModule extends AbstractModule {
  @Override
  protected void configure() {
    bind(TransactionLog.class)
        .toProvider(DatabaseTransactionLogProvider.class);
  }
```
If your providers are complex, be sure to test them!

### Throwing Exceptions
Guice does not allow exceptions to be thrown from Providers.  The Provider interface does not allow for checked exception to be thrown.  RuntimeExceptions may be wrapped in a `ProvisionException` or `CreationException` and may prevent your `Injector` from being created.  If you need to throw an exception for some reason, you may want to use the [[ThrowingProviders extension|ThrowingProviders]].
# UntargettedBindings.md

_Creating bindings that don't have targets_
### Untargeted Bindings

You may create bindings without specifying a target. This is most useful for concrete classes and types annotated by either `@ImplementedBy` or `@ProvidedBy`. An untargetted binding informs the injector about a type, so it may prepare dependencies eagerly. Untargetted bindings have no _to_ clause, like so:
```java
    bind(MyConcreteClass.class);
    bind(AnotherConcreteClass.class).in(Singleton.class);
```

When specifying binding annotations, you must still add the target binding, even it is the same concrete class.  For example:
```java
    bind(MyConcreteClass.class)
        .annotatedWith(Names.named("foo"))
        .to(MyConcreteClass.class);
    bind(AnotherConcreteClass.class)
        .annotatedWith(Names.named("foo"))
        .to(AnotherConcreteClass.class)
        .in(Singleton.class);
```
# ToConstructorBindings.md

### Constructor Bindings
_New in Guice 3.0_

Occasionally it's necessary to bind a type to an arbitrary constructor. This comes up when the `@Inject` annotation cannot be applied to the target constructor: either because it is a third party class, or because _multiple_ constructors that participate in dependency injection. [[@Provides methods|ProvidesMethods]] provide the best solution to this problem! By calling your target constructor explicitly, you don't need reflection and its associated pitfalls. But there are limitations of that approach: manually constructed instances do not participate in [[AOP|AOP]].

To address this, Guice has `toConstructor()` bindings. They require you to reflectively select your target constructor and handle the exception if that constructor cannot be found:
```java
public class BillingModule extends AbstractModule {
  @Override 
  protected void configure() {
    try {
      bind(TransactionLog.class).toConstructor(
          DatabaseTransactionLog.class.getConstructor(DatabaseConnection.class));
    } catch (NoSuchMethodException e) {
      addError(e);
    }
  }
}
```
In this example, the `DatabaseTransactionLog` must have a constructor that takes a single `DatabaseConnection` parameter. That constructor does not need an `@Inject` annotation. Guice will invoke that constructor to satisfy the binding.

Each `toConstructor()` binding is scoped independently. If you create multiple singleton bindings that target the same constructor, each binding yields its own instance.
# BuiltInBindings.md

_More bindings that you can use_
### Built-in Bindings
Alongside explicit and [[just-in-time bindings|JustInTimeBindings]] additional bindings are automatically included in the injector. Only the injector can create these bindings and attempting to bind them yourself is an error. 

### Loggers
Guice has a built-in binding for `java.util.logging.Logger`, intended to save some boilerplate. The binding automatically sets the logger's name to the name of the class into which the Logger is being injected..
```java
@Singleton
public class ConsoleTransactionLog implements TransactionLog {

  private final Logger logger;

  @Inject
  public ConsoleTransactionLog(Logger logger) {
    this.logger = logger;
  }

  public void logConnectException(UnreachableException e) {
    /* the message is logged to the "ConsoleTransacitonLog" logger */
    logger.warning("Connect exception failed, " + e.getMessage());
  }
```


### The Injector
In framework code, sometimes you don't know the type you need until runtime. In this rare case you should inject the injector. Code that injects the injector does not self-document its dependencies, so this approach should be done sparingly.

### Providers
For every type Guice knows about, it can also inject a Provider of that type. [[Injecting Providers|InjectingProviders]] describes this in detail.

### TypeLiterals
Guice has complete type information for everything it injects. If you're injecting parameterized types, you can inject a `TypeLiteral<T>` to reflectively tell you the element type.

### The Stage
Guice supports a stage enum to differentiate between development and production runs.

### MembersInjectors
When binding to providers or writing extensions, you may want Guice to inject dependencies into an object that you construct yourself.  To do this, add a dependency on a `MembersInjector<T>` (where T is your object's type), and then call `membersInjector.injectMembers(myNewObject)`.
# JustInTimeBindings.md

_Bindings that are created automatically by Guice_
### Just-in-time Bindings
When the injector needs an instance of a type, it needs a binding. The bindings in a modules are called *explicit bindings*, and the injector uses them whenever they're available. If a type is needed but there isn't an explicit binding, the injector will attempt to create a *Just-In-Time binding*. These are also known as JIT bindings and implicit bindings.

### Eligible Constructors
Guice can create bindings for concrete types by using the type's *injectable constructor*. This is either a non-private, no-arguments constructor, or a constructor with the `@Inject` annotation:
```java
public class PayPalCreditCardProcessor implements CreditCardProcessor {
  private final String apiKey;

  @Inject
  public PayPalCreditCardProcessor(@Named("PayPal API key") String apiKey) {
    this.apiKey = apiKey;
  }
```
Guice will not construct nested classes unless they have the `static` modifier. Inner classes have an implicit reference to their enclosing class that cannot be injected.

### @ImplementedBy
Annotate types tell the injector what their default implementation type is. The `@ImplementedBy` annotation acts like a *linked binding*, specifying the subtype to use when building a type.
```java
@ImplementedBy(PayPalCreditCardProcessor.class)
public interface CreditCardProcessor {
  ChargeResult charge(String amount, CreditCard creditCard)
      throws UnreachableException;
}
```
The above annotation is equivalent to the following `bind()` statement:
```java
    bind(CreditCardProcessor.class).to(PayPalCreditCardProcessor.class);
```
If a type is in both a `bind()` statement (as the first argument) and has the `@ImplementedBy` annotation, the `bind()` statement is used. The annotation suggests a _default implementation_ that can be overridden with a binding. Use `@ImplementedBy` carefully; it adds a compile-time dependency from the interface to its implementation.

### @ProvidedBy
`@ProvidedBy` tells the injector about a `Provider` class that produces instances:
```java
@ProvidedBy(DatabaseTransactionLogProvider.class)
public interface TransactionLog {
  void logConnectException(UnreachableException e);
  void logChargeResult(ChargeResult result);
}
```
The annotation is equivalent to a `toProvider()` binding:
```java
    bind(TransactionLog.class)
        .toProvider(DatabaseTransactionLogProvider.class);
```
Like `@ImplementedBy`, if the type is annotated and used in a `bind()` statement, the `bind()` statement will be used.
# Scopes.md

### Scopes
By default, Guice returns a new instance each time it supplies a value. This behaviour is configurable via *scopes*. Scopes allow you to reuse instances: for the lifetime of an application (`@Singleton`), a session (`@SessionScoped`), or a request (`@RequestScoped`). Guice includes a servlet extension that defines scopes for web apps. [[Custom scopes|CustomScopes]] can be written for other types of applications.


### Applying Scopes
Guice uses annotations to identify scopes. Specify the scope for a type by applying the scope annotation to the implementation class. As well as being functional, this annotation also serves as documentation. For example, `@Singleton` indicates that the class is intended to be threadsafe.
```java
@Singleton
public class InMemoryTransactionLog implements TransactionLog {
  /* everything here should be threadsafe! */
}
```
Scopes can also be configured in `bind` statements:
```java
  bind(TransactionLog.class).to(InMemoryTransactionLog.class).in(Singleton.class);
```
And by annotating `@Provides` methods:
```java
  @Provides @Singleton
  TransactionLog provideTransactionLog() {
    ...
  }
```
If there's conflicting scopes on a type and in a `bind()` statement, the `bind()` statement's scope will be used. If a type is annotated with a scope that you don't want, bind it to `Scopes.NO_SCOPE`.

In linked bindings, scopes apply to the binding source, not the binding target. Suppose we have a class `Applebees` that implements both `Bar` and `Grill` interfaces. These bindings allow for *two* instances of that type, one for `Bar`s and another for `Grill`s:
```java
  bind(Bar.class).to(Applebees.class).in(Singleton.class);
  bind(Grill.class).to(Applebees.class).in(Singleton.class);
```
This is because the scopes apply to the bound type (`Bar`, `Grill`), not the type that satisfies that binding (`Applebees`). To allow only a single instance to be created, use a `@Singleton` annotation on the declaration for that class. Or add another binding:
```java
  bind(Applebees.class).in(Singleton.class);
```
This binding makes the other two `.in(Singleton.class)` clauses above unnecessary.

The `in()` clause accepts either a scoping annotation like `RequestScoped.class` and also `Scope` instances like `ServletScopes.REQUEST`:
```java
  bind(UserPreferences.class)
      .toProvider(UserPreferencesProvider.class)
      .in(ServletScopes.REQUEST);
```
The annotation is preferred because it allows the module to be reused in different types of applications. For example, an `@RequestScoped` object could be scoped to the HTTP request in a web app and to the RPC when it's in an API server.


### Eager Singletons
Guice has special syntax to define singletons that can be constructed eagerly:
```java
  bind(TransactionLog.class).to(InMemoryTransactionLog.class).asEagerSingleton();
```
Eager singletons reveal initialization problems sooner, and ensure end-users get a consistent, snappy experience. Lazy singletons enable a faster edit-compile-run development cycle. Use the `Stage` enum to specify which strategy should be used.

                      | *PRODUCTION* | *DEVELOPMENT* |
----------------------|--------------|---------------|
.asEagerSingleton()   | eager        | eager         |
.in(Singleton.class)  | eager        | lazy          |
.in(Scopes.SINGLETON) | eager        | lazy          |
@Singleton            | eager\*     | lazy          |

\* Guice will only eagerly build singletons for the types it knows about. These are the types mentioned in your modules, plus the transitive dependencies of those types.


### Choosing a scope
If the object is *stateful*, the scoping should be obvious. Per-application is `@Singleton`, per-request is `@RequestScoped`, etc. If the object is *stateless* and *inexpensive to create*, scoping is unnecessary. Leave the binding unscoped and Guice will create new instances as they're required.

Singletons are popular in Java applications but they don't provide much value, especially when dependency injection is involved. Although singletons save object creation (and later garbage collection), initialization of the singleton requires synchronization; getting a handle to the single initialized instance only requires reading a volatile. Singletons are most useful for:
  * stateful objects, such as configuration or counters
  * objects that are expensive to construct or lookup
  * objects that tie up resources, such as a database connection pool.


### Scopes and Concurrency
Classes annotated `@Singleton` and `@SessionScoped` *must be threadsafe*. Everything that's injected into these classes must also be threadsafe. [[Minimize mutability|MinimizeMutability]] to limit the amount of state that requires concurrency protection.

`@RequestScoped` objects do not need to be threadsafe. It is usually an error for a `@Singleton` or `@SessionScoped` object to depend on an `@RequestScoped` one. Should you require an object in a narrower scope, inject a `Provider` of that object.
# Injections.md

_How Guice initializes your objects_
### Injections
The dependency injection pattern separates behaviour from dependency resolution. Rather than looking up dependencies directly or from factories, the pattern recommends that dependencies are passed in. The process of setting dependencies into an object is called *injection*.

#### Constructor Injection
Constructor injection combines instantiation with injection. To use it, annotate the constructor with the `@Inject` annotation. This constructor should accept class dependencies as parameters. Most constructors will then assign the parameters to final fields.
```java
public class RealBillingService implements BillingService {
  private final CreditCardProcessor processorProvider;
  private final TransactionLog transactionLogProvider;

  @Inject
  public RealBillingService(CreditCardProcessor processorProvider,
      TransactionLog transactionLogProvider) {
    this.processorProvider = processorProvider;
    this.transactionLogProvider = transactionLogProvider;
  }
```
If your class has no `@Inject`-annotated constructor, Guice will use a public, no-arguments constructor if it exists. Prefer the annotation, which documents that the type participates in dependency injection.

Constructor injection works nicely with unit testing. If your class accepts all of its dependencies in a single constructor, you won't accidentally forget to set a dependency. When a new dependency is introduced, all of the calling code conveniently breaks! Fix the compile errors and you can be confident that everything is properly wired up.


#### Method Injection
Guice can inject methods that have the `@Inject` annotation. Dependencies take the form of parameters, which the injector resolves before invoking the method. Injected methods may have any number of parameters, and the method name does not impact injection.
```java
public class PayPalCreditCardProcessor implements CreditCardProcessor {
  
  private static final String DEFAULT_API_KEY = "development-use-only";
  
  private String apiKey = DEFAULT_API_KEY;

  @Inject
  public void setApiKey(@Named("PayPal API key") String apiKey) {
    this.apiKey = apiKey;
  }
```


#### Field Injection
Guice injects fields with the `@Inject` annotation. This is the most concise injection, but the least testable.
```java
public class DatabaseTransactionLogProvider implements Provider<TransactionLog> {
  @Inject Connection connection;

  public TransactionLog get() {
    return new DatabaseTransactionLog(connection);
  }
}
```
Avoid using field injection with `final` fields, which has [weak semantics](http://java.sun.com/javase/6/docs/api/java/lang/reflect/Field.html#set(java.lang.Object,%20java.lang.Object)).


#### Optional Injections
Occasionally it's convenient to use a dependency when it exists and to fall back to a default when it doesn't. Method and field injections may be optional, which causes Guice to silently ignore them when the dependencies aren't available. To use optional injection, apply the `@Inject(optional=true)` annotation:
```java
public class PayPalCreditCardProcessor implements CreditCardProcessor {
  private static final String SANDBOX_API_KEY = "development-use-only";

  private String apiKey = SANDBOX_API_KEY;

  @Inject(optional=true)
  public void setApiKey(@Named("PayPal API key") String apiKey) {
    this.apiKey = apiKey;
  }
```
Mixing optional injection and just-in-time bindings may yield surprising results. For example, the following field is always injected even when `Date` is not explicitly bound. This is because `Date` has a public no-arguments constructor that is eligible for just-in-time bindings.
```java
  @Inject(optional=true) Date launchDate;
```


### On-demand Injection
Method and field injection can be used to initialize an existing instance. You can use the `Injector.injectMembers` API:
```java
  public static void main(String[] args) {
    Injector injector = Guice.createInjector(...);
    
    CreditCardProcessor creditCardProcessor = new PayPalCreditCardProcessor();
    injector.injectMembers(creditCardProcessor);
```


### Static Injections
When [migrating an application](http://publicobject.com/2007/07/guice-patterns-1-horrible-static-code.html) from static factories to Guice, it is possible to change incrementally. Static injection is a helpful crutch here. It makes it possible for objects to _partially_ participate in dependency injection, by gaining access to injected types without being injected themselves. Use `requestStaticInjection()` in a module to specify classes to be injected at injector-creation time:
```java
  @Override public void configure() {
    requestStaticInjection(ProcessorFactory.class);
    ...
  }
```
Guice will inject class's static members that have the `@Injected` annotation:
```java
class ProcessorFactory {
  @Inject static Provider<Processor> processorProvider;

  /**
   * @deprecated prefer to inject your processor instead.
   */
  @Deprecated
  public static Processor getInstance() {
    return processorProvider.get();
  }
}
```
Static members will not be injected at instance-injection time. This API is not recommended for general use because it suffers many of the same problems as static factories: it's clumsy to test, it makes dependencies opaque, and it relies on global state.

### Automatic Injection
Guice automatically injects all of the following:
  * instances passed to `toInstance()` in a bind statement
  * provider instances passed to `toProvider()` in a bind statement
The objects will be injected while the injector itself is being created. If they're needed to satisfy other startup injections, Guice will inject them before they're used.
# InjectingProviders.md

### Injecting Providers
With normal dependency injection, each type gets exactly *one instance* of each of its dependent types. The `RealBillingService` gets one `CreditCardProcessor` and one `TransactionLog`. When this flexibility is necessary, Guice binds a provider. Providers produce a value when the `get()` method is invoked:
```java
public interface Provider<T> {
  T get();
}
```
The provider's type is _parameterized_ to differentiate a `Provider<TransactionLog>` from a `Provider<CreditCardProcessor>`. Wherever you inject a value you can inject a provider for that value.
```java
public class RealBillingService implements BillingService {
  private final Provider<CreditCardProcessor> processorProvider;
  private final Provider<TransactionLog> transactionLogProvider;

  @Inject
  public RealBillingService(Provider<CreditCardProcessor> processorProvider,
      Provider<TransactionLog> transactionLogProvider) {
    this.processorProvider = processorProvider;
    this.transactionLogProvider = transactionLogProvider;
  }

  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    CreditCardProcessor processor = processorProvider.get();
    TransactionLog transactionLog = transactionLogProvider.get();

    /* use the processor and transaction log here */
  }
}
```
For every binding, annotated or not, the injector has a built-in binding for its provider.


### Providers for multiple instances
Use providers when you need multiple instances of the same type. Suppose your application saves a summary entry and a details when a pizza charge fails. With providers, you can get a new entry whenever you need one:
```java
public class LogFileTransactionLog implements TransactionLog {

  private final Provider<LogFileEntry> logFileProvider;

  @Inject
  public LogFileTransactionLog(Provider<LogFileEntry> logFileProvider) {
    this.logFileProvider = logFileProvider;
  }

  public void logChargeResult(ChargeResult result) {
    LogFileEntry summaryEntry = logFileProvider.get();
    summaryEntry.setText("Charge " + (result.wasSuccessful() ? "success" : "failure"));
    summaryEntry.save();

    if (!result.wasSuccessful()) {
      LogFileEntry detailEntry = logFileProvider.get();
      detailEntry.setText("Failure result: " + result);
      detailEntry.save();
    }
  }
```


### Providers for lazy loading
If you've got a dependency on a type that is particularly *expensive to produce*, you can use providers to defer that work. This is especially useful when you don't always need the dependency:
```java
public class DatabaseTransactionLog implements TransactionLog {
  
  private final Provider<Connection> connectionProvider;

  @Inject
  public DatabaseTransactionLog(Provider<Connection> connectionProvider) {
    this.connectionProvider = connectionProvider;
  }

  public void logChargeResult(ChargeResult result) {
    /* only write failed charges to the database */
    if (!result.wasSuccessful()) {
      Connection connection = connectionProvider.get();
    }
  }
```

### Providers for Mixing Scopes
It is an error to depend on an object in a _narrower_ scope. Suppose you have a singleton transaction log that needs on the request-scoped current user. Should you inject the user directly, things break because the user changes from request to request. Since providers can produce values on-demand, they enable you to mix scopes safely:
```java
@Singleton
public class ConsoleTransactionLog implements TransactionLog {
  
  private final AtomicInteger failureCount = new AtomicInteger();
  private final Provider<User> userProvider;

  @Inject
  public ConsoleTransactionLog(Provider<User> userProvider) {
    this.userProvider = userProvider;
  }

  public void logConnectException(UnreachableException e) {
    failureCount.incrementAndGet();
    User user = userProvider.get();
    System.out.println("Connection failed for " + user + ": " + e.getMessage());
    System.out.println("Failure count: " + failureCount.incrementAndGet());
  }
```
# AOP.md

_Intercepting methods with Guice_
### Aspect Oriented Programming
To compliment dependency injection, Guice supports *method interception*. This feature enables you to write code that is executed each time a _matching_ method is invoked. It's suited for cross cutting concerns ("aspects"), such as transactions, security and logging. Because interceptors divide a problem into aspects rather than objects, their use is called Aspect Oriented Programming (AOP).

Most developers won't write method interceptors directly; but they may see their use in integration libraries like [Warp Persist](http://www.wideplay.com/guicewebextensions2). Those that do will need to select the matching methods, create an interceptor, and configure it all in a module.

[Matcher](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/matcher/Matcher.html) is a simple interface that either accepts or rejects a value. For Guice AOP, you need two matchers: one that defines which classes participate, and another for the methods of those classes. To make this easy, there's [factory class](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/matcher/Matchers.html) to satisfy the common scenarios. 

[MethodInterceptors](http://aopalliance.sourceforge.net/doc/org/aopalliance/intercept/MethodInterceptor.html) are executed whenever a matching method is invoked. They have the opportunity to inspect the call: the method, its arguments, and the receiving instance. They can perform their cross-cutting logic and then delegate to the underlying method. Finally, they may inspect the return value or exception and return. Since interceptors may be applied to many methods and will receive many calls, their implementation should be efficient and unintrusive.

### Example: Forbidding method calls on weekends
To illustrate how method interceptors work with Guice, we'll forbid calls to our pizza billing system on weekends. The delivery guys only work Monday thru Friday so we'll prevent pizza from being ordered when it can't be delivered! This example is structurally similar to use of AOP for authorization.

To mark select methods as weekdays-only, we define an annotation:
```java
@Retention(RetentionPolicy.RUNTIME) @Target(ElementType.METHOD)
@interface NotOnWeekends {}
```
...and apply it to the methods that need to be intercepted:
```java
public class RealBillingService implements BillingService {

  @NotOnWeekends
  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    ...
  }
}
```
Next, we define the interceptor by implementing the `org.aopalliance.intercept.MethodInterceptor` interface. When we need to call through to the underlying method, we do so by calling `invocation.proceed()`:
```java
public class WeekendBlocker implements MethodInterceptor {
  public Object invoke(MethodInvocation invocation) throws Throwable {
    Calendar today = new GregorianCalendar();
    if (today.getDisplayName(DAY_OF_WEEK, LONG, ENGLISH).startsWith("S")) {
      throw new IllegalStateException(
          invocation.getMethod().getName() + " not allowed on weekends!");
    }
    return invocation.proceed();
  }
}
```
Finally, we configure everything. This is where we create matchers for the classes and methods to be intercepted. In this case we match any class, but only the methods with our `@NotOnWeekends` annotation:
```java
public class NotOnWeekendsModule extends AbstractModule {
  protected void configure() {
    bindInterceptor(Matchers.any(), Matchers.annotatedWith(NotOnWeekends.class), 
        new WeekendBlocker());
  }
}
```
Putting it all together, (and waiting until Saturday), we see the method is intercepted and our order is rejected:
```java
Exception in thread "main" java.lang.IllegalStateException: chargeOrder not allowed on weekends!
	at com.publicobject.pizza.WeekendBlocker.invoke(WeekendBlocker.java:65)
	at com.google.inject.internal.InterceptorStackCallback.intercept(...)
	at com.publicobject.pizza.RealBillingService$$EnhancerByGuice$$49ed77ce.chargeOrder(<generated>)
	at com.publicobject.pizza.WeekendExample.main(WeekendExample.java:47)
```

### Limitations
Behind the scenes, method interception is implemented by generating bytecode at runtime. Guice dynamically creates a subclass that applies interceptors by overriding methods. If you are on a platform that doesn't support bytecode generation (such as Android), you should use [[Guice without AOP support|OptionalAOP]].

This approach imposes limits on what classes and methods can be intercepted:
  * Classes must be public or package-private.
  * Classes must be non-final
  * Methods must be public, package-private or protected
  * Methods must be non-final
  * Instances must be created by Guice by an `@Inject`-annotated or no-argument constructor
It is not possible to use method interception on instances that aren't constructed by Guice.

### Injecting Interceptors
If you need to inject dependencies into an interceptor, use the `requestInjection` API.
```java
public class NotOnWeekendsModule extends AbstractModule {
  protected void configure() {
    WeekendBlocker weekendBlocker = new WeekendBlocker();
    requestInjection(weekendBlocker);
    bindInterceptor(Matchers.any(), Matchers.annotatedWith(NotOnWeekends.class), 
       weekendBlocker);
  }
}
```
Another option is to use Binder.getProvider and pass the dependency in the constructor of the interceptor.
```java
public class NotOnWeekendsModule extends AbstractModule {
  protected void configure() {
    bindInterceptor(any(),
                    annotatedWith(NotOnWeekends.class),
                    new WeekendBlocker(getProvider(Calendar.class)));
  }
}
```
Use caution when injecting interceptors. If your interceptor calls a method that it itself is intercepting, you may receive a `StackOverflowException` due to unending recursion.

### AOP Alliance
The method interceptor API implemented by Guice is a part of a public specification called [AOP Alliance](http://aopalliance.sourceforge.net/). This makes it possible to use the same interceptors across a variety of frameworks.
# MinimizeMutability.md

### Minimize mutability
Wherever possible, use constructor injection to create immutable objects. Immutable objects are simple, shareable, and can be composed. Follow this pattern to define your injectable types:
```java
public class RealPaymentService implements PaymentService { 

   private final PaymentQueue paymentQueue; 
   private final Notifier notifier;  

   @Inject 
   RealPaymentRequestService( 
       PaymentQueue paymentQueue, 
       Notifier notifier) { 
     this.paymentQueue = paymentQueue; 
     this.notifier = notifier; 
   }

   ...
```
All fields of this class are final and initialized by a single `@Inject`-annotated constructor. [Effective Java](http://java.sun.com/docs/books/effective/toc.html) discusses other benefits of immutability.

#### Injecting methods and fields
*Constructor injection* has some limitations:
  * Injected constructors may not be optional.
  * It cannot be used unless objects are created by Guice. This is a dealbreaker for certain frameworks.
  * Subclasses must call `super()` with all dependencies. This makes constructor injection cumbersome, especially as the injected base class changes.

*Method injection* is most useful when you need to initialize an instance that is not constructed by Guice. Extensions like [[AssistedInject]] and Multibinder use method injection to initialize bound objects.

*Field injection* has the most compact syntax, so it shows up frequently on slides and in examples. It is neither encapsulated nor testable. Never inject [final fields](https://github.com/google/guice/issues/245); the JVM doesn't guarantee that the injected value will be visible to all threads.
# InjectOnlyDirectDependencies.md

### Inject only direct dependencies
Avoid injecting an object only as a means to get at another object. For example, don't inject a `Customer` as a means to get at an `Account`:
```java
public class ShowBudgets { 
   private final Account account; 

   @Inject 
   ShowBudgets(Customer customer) { 
     account = customer.getPurchasingAccount(); 
   } 
```
Instead, inject the dependency directly. This makes testing easier; the test case doesn't need to concern itself with the customer. Use an `@Provides` method in your `Module` to create the binding for `Account` that uses the binding for `Customer`:
```java
public class CustomersModule extends AbstractModule { 
  @Override public void configure() {
    ...
  }

  @Provides 
  Account providePurchasingAccount(Customer customer) { 
    return customer.getPurchasingAccount();
  }
```
By injecting the dependency directly, our code is simpler.
```java
public class ShowBudgets { 
   private final Account account; 

   @Inject 
   ShowBudgets(Account account) { 
     this.account = account; 
   } 
```
# CyclicDependencies.md

_Resolving cyclic dependencies_              
### Resolving Cyclic Dependencies 

Say that your application has a few classes including a Store, a Boss, and a Clerk.

```java
public class Store {
  	private final Boss boss;
  	//...

	@Inject public Store(Boss boss) {
 		this.boss = boss;
 		//...
	}

	public void incomingCustomer(Customer customer) {...}
	public Customer getNextCustomer() {...}
}

public class Boss {
	private final Clerk Clerk;
	@Inject public Boss(Clerk Clerk) {
		this.Clerk = Clerk;
	}
}

public class Clerk {
	// Nothing interesting here
}
```

Right now, the dependency chain is all good: constructing a Store results in constructing a Boss, which results in constructing a Clerk. However, to get the Clerk to get a Customer to do his selling, he'll need a reference to the Store to get those customer

```java
public class Store {
  	private final Boss boss;
  	//...

	@Inject public Store(Boss boss) {
 		this.boss = boss;
 		//...
	}
	public void incomingCustomer(Customer customer) {...}
	public Customer getNextCustomer() {...}
}

public class Boss {
	private final Clerk clerk;
	@Inject public Boss(Clerk clerk) {
		this.clerk = clerk;
	}
}

public class Clerk {
	private final Store shop;
	@Inject Clerk(Store shop) {
		this.shop = shop;
	}

	void doSale() {
		Customer sucker = shop.getNextCustomer();
		//...
	}
}
```

which leads to a cycle: Clerk -> Store -> Boss -> Clerk. In trying to construct a Clerk, an Store will be constructed, which needs a Boss, which needs a Clerk again! 

There are a few ways to resolve this cycle:

### Eliminate the cycle (Recommended) 

Cycles often reflect insufficiently granular decomposition.  To eliminate such cycles, extract the Dependency Case into a separate class.

In this example, the work of managing the incoming customers can be extracted into another class, say `CustomerLine`, and that can be injected into the Clerk and Store.

```java
public class Store {
  	private final Boss boss;
  	private final CustomerLine line;
  	//...

	@Inject public Store(Boss boss, CustomerLine line) {
 		this.boss = boss; 
 		this.line = line;
 		//...
	}

	public void incomingCustomer(Customer customer) { line.add(customer); }	
}

public class Clerk {
	private final CustomerLine line;

	@Inject Clerk(CustomerLine line) {
		this.line = line;
	}

	void doSale() {
		Customer sucker = line.getNextCustomer();
		//...
	}
}
```

While both Store and Clerk dependend on the CustomerLine, there's no cycle in the dependency graph (although you may want to make sure that the Store and Clerk both use the same CustomerLine instance). This also means that your Clerk will be able to sell cars when your shop has a big tent sale: just inject a different CustomerLine.

### Break the cycle with a Provider 
[[Injecting a Guice provider|InjectingProviders]] will allow you to add a _seam_ in the dependency graph. The Clerk will still depend on the Store, but the Clerk doesn't look at the Store until he needs one.
```java
public class Clerk {
	private final Provider<Store> shopProvider;
	@Inject Clerk(Provider<Store> shopProvider) {
		this.shopProvider = shopProvider;
	}

	void doSale() {
		Customer sucker = shopProvider.get().getNextCustomer();
		//...
	}
}
```

Note here, that unless Store is bound as a Singleton or in some other scope to be reused, the `shopProvider.get()` call will end up constructing a new Store, which will construct a new Boss, which will construct a new Clerk again!

#### Factory Methods to tie two objects together

When your dependencies are tied together a bit closer, untangling them with the above methods don't work. Situations like this come up when using something like a View/Presenter paradigm:

```java
public class FooPresenter {
	@Inject public FooPresenter(FooView view) {
		//...
	}

	public void doSomething() {
		view.doSomethingCool();
	}
}

public class FooView {
	@Inject public FooView(FooPresenter presenter) {
		//...
	}

	public void userDidSomething() {
		presenter.theyDidSomething();
	}
	//...
}
```

Each of those objects needs the other object. Here, you can use [[AssistedInject]] to get around it:

```java
public class FooPresenter {
	privat final FooView view;
	@Inject public FooPresenter(FooView.Factory viewMaker) {
		view = viewMaker.create(this);
	}

	public void doSomething() {
	//...
		view.doSomethingCool();
	}
}

public class FooView {
	@Inject public FooView(@Assisted FooPresenter presenter) {...}

	public void userDidSomething() {
		presenter.theyDidSomething();
	}

	public static interface Factory {
		FooView create(FooPresenter presenter)
	}
}
```

Such situations also come up when attempting to use Guice to manifest business object models, which may have cycles that reflect different types of relationships.  [[AssistedInject]] is also quite good for such cases.
# AvoidStaticState.md

### Avoid static state
Static state and testability are enemies. Your tests should be fast and free of side-effects. But non-constant values held by static fields are a pain to manage. It's tricky to reliably tear down static singletons that are mocked by tests, and this interferes with other tests. 

`requestStaticInjection()` is a *crutch*. Guice includes this API to ease migration from a statically-configured application to a dependency-injected one. New applications developed with Guice should not use this API.

Although *static state* is bad, there's nothing wrong with the static *keyword*. Static classes are okay (preferred even!) and for pure functions (sorting, math, etc.), static is just fine.
# UseNullable.md

### Use @Nullable
To eliminate `NullPointerExceptions` in your codebase, you must be disciplined about null references. We've been successful at this by following and enforcing a simple rule:
  _Every parameter is non-null unless explicitly specified._
The [Guava: Google Core Libraries for Java](http://code.google.com/p/guava-libraries/) and [JSR-305](http://code.google.com/p/jsr-305/) have simple APIs to get a nulls under control. `Preconditions.checkNotNull` can be used to fast-fail if a null reference is found, and `@Nullable` can be used to annotate a parameter that permits the `null` value:
```java
import static com.google.common.base.Preconditions.checkNotNull;
import static javax.annotation.Nullable;

public class Person {
  ...

  public Person(String firstName, String lastName, @Nullable Phone phone) {
    this.firstName = checkNotNull(firstName, "firstName");
    this.lastName = checkNotNull(lastName, "lastName");
    this.phone = phone;
  }
```
*Guice forbids null by default.* It will refuse to inject `null`, failing with a `ProvisionException` instead. If `null` is permissible by your class, you can annotate the field or parameter with `@Nullable`. Guice recognizes any `@Nullable` annotation, like [edu.umd.cs.findbugs.annotations.Nullable](http://findbugs.sourceforge.net/api/edu/umd/cs/findbugs/annotations/Nullable.html) or [javax.annotation.Nullable](http://code.google.com/p/jsr-305/source/browse/trunk/ri/src/main/java/javax/annotation/Nullable.java?r=24).
# ModulesShouldBeFastAndSideEffectFree.md

### Modules should be fast and side-effect free
Rather than using an external XML file for configuration, Guice modules are written using regular Java code. Java is familiar, works with your IDE, and survives refactoring.

But the full power of the Java language comes at a cost: it's easy to do _too much_ in a module. It's tempting to connect to a database connection or to start an HTTP server in your Guice module. Don't do this! Doing heavy-lifting in a module poses problems:
  * **Modules start up, but they don't shut down.** Should you open a database connection in your module, you won't have any hook to close it. 
  * **Modules should be tested.** If a module opens a database as a course of execution, it becomes difficult to write  unit tests for it.
  * **Modules can be overridden.** Guice modules support [overrides](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/util/Modules.html#override(com.google.inject.Module...)), allowing a production service to be substituted with a lightweight or test one. When the production service is created as a part of module execution, such overrides are ineffective.

Rather than doing work in the module itself, define an interface that can do the work at the proper level of abstraction. In our applications we use this interface:
```java
public interface Service {
  /**
   * Starts the service. This method blocks until the service has completely started.
   */
  void start() throws Exception;

  /**
   * Stops the service. This method blocks until the service has completely shut down.
   */
  void stop();
}
```
After creating the Injector, we finish bootstrapping our application by starting its services. We also add shutdown hooks to cleanly release resources when the application is stopped.
```java
  public static void main(String[] args) throws Exception {
    Injector injector = Guice.createInjector(
        new DatabaseModule(),
        new WebserverModule(),
        ...
    );

    Service databaseConnectionPool = injector.getInstance(
        Key.get(Service.class, DatabaseService.class));
    databaseConnectionPool.start();
    addShutdownHook(databaseConnectionPool);

    Service webserver = injector.getInstance(
        Key.get(Service.class, WebserverService.class));
    webserver.start();
    addShutdownHook(webserver);
  }
```
# BeCarefulAboutIoInProviders.md

### Be careful about I/O in Providers
The `Provider` interface is convenient for the caller, but it lacks semantics:
  * **Provider doesn't declare checked exceptions.** If you're writing code that needs to recover from specific types of failures, you can't catch `TransactionRolledbackException`. `ProvisionException` allows you to recover from general provision failures, and you can [iterate its causes](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/ProvisionException.html#getErrorMessages()), but you can't specify what those causes may be.
  * **Provider doesn't support a timeout.**
  * **Provider doesn't define a retry-strategy.** When a value is unavailable, calling `get()` multiple times may cause multiple failed provisions.

[[ThrowingProviders]] is a Guice extension that implements an exception-throwing provider. It allows failures to be scoped, so a failed lookup only happens once per request or session.
# AvoidConditionalLogicInModules.md

### Avoid conditional logic in modules
It’s tempting to create modules that have moving parts and can be configured to operate differently for different environments:
```java
public class FooModule {
  private final String fooServer;

  public FooModule() {
    this(null);
  }

  public FooModule(@Nullable String fooServer) {
    this.fooServer = fooServer;
  }

  @Override protected void configure() {
    if (fooServer != null) {
      bind(String.class).annotatedWith(named("fooServer")).toInstance(fooServer);
      bind(FooService.class).to(RemoteFooService.class);
    } else {
      bind(FooService.class).to(InMemoryFooService.class);
    }
  }
}
```
Conditional logic in itself isn't too bad. But problems arise when configurations are untested. In this example, the`InMemoryFooService` is used for development and `RemoteFooService` is used in production. But without testing this specific case, it's impossible to be sure that `RemoteFooService` works in the integrated application.

To overcome this, **minimize the number of distinct configurations** in your applications. If you split production and development into distinct modules, it is easier to be sure that the entire production codepath is tested. In this case, we split `FooModule` into `RemoteFooModule` and `InMemoryFooModule`. This also prevents production classes from having a compile-time dependency on test code.
# KeepConstructorsHidden.md

### Keep constructors on Guice-instantiated classes as hidden as possible.
Consider this simple interface:
```java
public interface DataReader {
 
  Data readData(DataSource dataSource);
}
```

It's a common reflex to implement this interface with a public class:
```java
public class DatabaseDataReader implements DataReader {
  
   private final ConnectionManager connectionManager;

   @Inject
   public DatabaseDataReader(
      ConnectionManager connectionManager) {
     this.connectionManager = connectionManager;
   }

   @Override
   public Data readData(DataSource dataSource) {
      // ... read data from the database
      return Data.of(readInData, someMetaData);
   }
}
```

A quick inspection of this code reveals nothing faulty about this implementation.  Unfortunately, such an inspection excludes the dimension of time and the inevitability of an unguarded code base to become more tightly coupled within itself over time.

Similar to the old axiom, [Nothing good happens after midnight](http://www.google.com/webhp#hl=en&q=nothing+good+happens+after+midnight), we also know that Nothing good happens after making a constructor public:  A public constructor _will_ have illicit uses introduced within a code base.  These uses necessarily will:

   * make refactoring more difficult.
   * break the interface-implementation abstraction barrier.   
   * introduce tighter coupling within a codebase.


Perhaps worst of all, any direct use of a constructor circumvents Guice's object instantiation.

As a correction, simply limit the visibility of both your implementation classes, and their constructors.  Typically package private is preferred for both, as this facilitates:

   * binding the class within a `Module` in the same package
   * unit testing the class through means of direct instantiation

As a simple, mnemonic remember that `public` and `@Inject` are like [Elves and Dwarfs](http://en.wikipedia.org/wiki/Dwarf_(Middle-earth)):  they _can_ work together, but in an ideal world, they would coexist independently.
# FrequentlyAskedQuestions.md

### Frequently Asked Questions

##### How do I inject configuration parameters?
You need a binding annotation to identify your parameter. Create an annotation class that defines the parameter:
```java
/**
 * Annotates the URL of the foo server.
 */
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.PARAMETER})
@BindingAnnotation
public @interface FooServerAddress {}
```
Bind the annotation to its value in your module:
```java
public class FooModule extends AbstractModule {
  private final String fooServerAddress;

  /**
   * @param fooServerAddress the URL of the foo server.
   */
  public FooModule(String fooServerAddress) {
    this.fooServerAddress = fooServerAddress;
  }

  @Override public void configure() {
    bindConstant().annotatedWith(FooServerAddress.class).to(fooServerAddress);
    ...
  }
}
```
Finally, inject it into your class:
```java
public class FooClient {

  @Inject
  FooClient(@FooServerAddress String fooServerAddress) {
    ...
  }
```
You may save some keystrokes by using Guice's built-in `@Named` binding annotation rather than creating your own.

##### How do I load configuration properties?
Use [Names.bindProperties()](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/name/Names.html#bindProperties(com.google.inject.Binder,%20java.util.Properties)) to create bindings for each of the properties in a configuration file.

##### How do I pass a parameter when creating an object via Guice?
You can't directly pass a parameter into an injected value. But you can use Guice to create a `Factory`, and use that factory to create your object.
```java
public class Thing {
  // note: no @Inject annotation here
  private Thing(A a, B b) {
    ...
  }

  public static class Factory {
    @Inject
    public Factory(A a) { ... }
    public Thing make(B b) { ... }
  }
}
```

```java
public class Example {
  @Inject
  public Example(Thing.Factory factory) { ... }
}
```
See [[AssistedInject]], which can be used to remove the factory boilerplate.

<a name="RobotLegs"/>
##### How do I build two similar but slightly different trees of objects?
This is commonly called the "robot legs" problem: How to create a robot with a two `Leg` objects, the left one injected with a `LeftFoot`, and the right one with a `RightFoot`. But only one `Leg` class that's reused in both contexts.

There's a [PrivateModules solution](http://docs.google.com/Doc?id=dhfm3hw2_51d2tmv6pc). It uses two separate private modules, a `@Left` one and an `@Right` one. Each has a binding for the unannotated `Foot.class` and `Leg.class`, and exposes a binding for the annotated `Leg.class`:
```java
class LegModule extends PrivateModule {
  private final Class<? extends Annotation> annotation;

  LegModule(Class<? extends Annotation> annotation) {
    this.annotation = annotation;
  }

  @Override protected void configure() {
    bind(Leg.class).annotatedWith(annotation).to(Leg.class);
    expose(Leg.class).annotatedWith(annotation);

    bindFoot();
  }

  abstract void bindFoot();
}
```
```java
  public static void main(String[] args) {
    Injector injector = Guice.createInjector(
        new LegModule(Left.class) {
          @Override void bindFoot() {
            bind(Foot.class).toInstance(new Foot("leftie"));
          }
        },
        new LegModule(Right.class) {
          @Override void bindFoot() {
            bind(Foot.class).toInstance(new Foot("righty"));
          }
        });
  }
```
See also [Alen Vrecko's more complete example](http://pastie.org/368348).

##### How can I inject an inner class?
Guice doesn't support this.  However, you can inject a _nested_ class (sometimes called a "static inner class"):
```java
class Outer {
  static class Nested {
    ...
  }
}
```

##### How to inject class with generic type?
You may need to inject a class with a parameterized type, like `List<String>`:
```java
class Example {
  @Inject
  void setList(List<String> list) {
    ...
  }
}
```
You can use a [TypeLiteral](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/TypeLiteral.html) to create the binding. `TypeLiteral` is a special class that allows you to specify a full parameterized type.
```java
  @Override public void configure() {
    bind(new TypeLiteral<List<String>>() {}).toInstance(new ArrayList<String>());
  }
```

Alternately, you can use an @Provides method.
```java
 @Provides List<String> providesListOfString() {
   return new ArrayList<String>();
 }
```

##### How can I inject optional parameters into a constructor?
Neither constructors nor `@Provides` methods support optional injection. To work-around this, you can create an inner class that holds the optional value:
```java
class Car {
  private final Engine engine;
  private final AirConditioner airConditioner;

  @Inject
  public Car(Engine engine, AirConditionerHolder airConditionerHolder) {
    this.engine = engine;
    this.airConditioner = airConditionerHolder.value;
  }

  static class AirConditionerHolder {
    @Inject(optional=true) AirConditioner value = new NoOpAirconditioner();
  }
}
```
This also allows for a default value for the optional parameter.

##### How do I inject a method interceptor?
In order to inject dependencies in an AOP `MethodInterceptor`, use `requestInjection()` alongside the standard `bindInterceptor()` call.
```java
public class NotOnWeekendsModule extends AbstractModule {
  protected void configure() {
    MethodInterceptor interceptor = new WeekendBlocker();
    requestInjection(interceptor);
    bindInterceptor(any(), annotatedWith(NotOnWeekends.class), interceptor);
  }
}
```

Another option is to use Binder.getProvider and pass the dependency in the constructor of the interceptor.
```java
public class NotOnWeekendsModule extends AbstractModule {
  protected void configure() {
    bindInterceptor(any(),
                    annotatedWith(NotOnWeekends.class),
                    new WeekendBlocker(getProvider(Calendar.class)));
  }
}
```

##### How can I get other questions answered?
Please post to the [google-guice](http://groups.google.com/group/google-guice) discussion group.
# Servlets.md

_Guice Servlet Extensions_

### Introduction

Guice Servlet provides a complete story for use in web applications and servlet containers. Guice's servlet extensions allow you to completely eliminate `web.xml` from your servlet application and take advantage of type-safe, idiomatic Java configuration of your servlet and filter components.

This has advantages not only in being able to use a nicer API for configuring your web applications, but also in tying together dependency injection with web components. Meaning that your servlets and filters benefit from:

  * Constructor injection
  * Type-safe, idiomatic configuration
  * Modularization (package and distribute custom Guice Servlet libraries)
  * Guice AOP

While keeping the benefits of the standard servlet lifecycle.

### Getting Started

Before you begin, you will require the latest version of the *guice-servlet* jar file, which is available along with the full Guice distribution from the [project homepage](http://github.com/google/guice) (or can be built from source using `ant dist distjars`). Once you have this library in your classpath, along with the core guice jar, you ready to go.

Start by placing `GuiceFilter` at the top of your `.web.xml` file:

```xml
  <filter>
    <filter-name>guiceFilter</filter-name>
    <filter-class>com.google.inject.servlet.GuiceFilter</filter-class>
  </filter>

  <filter-mapping>
    <filter-name>guiceFilter</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>
```

This tells the Servlet Container to re-route all requests through `GuiceFilter`. The nice thing about this is that any servlets or JSPs you already have will continue to function as normal, and you can migrate them to Guice Servlet at your own pace.
# ServletModule.md

_Using Guice Servlet and Binding Language_

### Installing a Servlet Module

Once you have [[GuiceFilter|Servlets]] up and running, Guice Servlet is set up. However, you will need to install an instance of `ServletModule` in order to get real use out of Guice Servlet:
```java
   Guice.createInjector(new ServletModule());
```

This module sets up the request and session scopes, and provides a place to configure your filters and servlets from. While you are free to create the injector from any place of your choice, a logical place to do it in is a `ServletContextListener`. 

A `ServletContextListener` is a Java servlet component that is triggered as soon as a web application is deployed, and before any requests begin to arrive. Guice Servlet provides a convenience utility that you can subclass in order to register your own `ServletContextListener`s:

```java
public class MyGuiceServletConfig extends GuiceServletContextListener {

  @Override
  protected Injector getInjector() {
    return Guice.createInjector(new ServletModule());
  }
}
```

Next, add the following to `web.xml` so the servlet container triggers this class when the app is deployed:

```xml
<listener>
  <listener-class>com.example.MyGuiceServletConfig</listener-class>
</listener>
```

You can now use Guice Servlet as per your needs. Note that it is not necessary to use a `ServletContextListener` to use Guice Servlet, as long as you remember to install `ServletModule` when creating your injector.

### The Binding Language

Think of the `ServletModule` as an in-code replacement for the `web.xml` deployment descriptor. Filters and servlets are configured here using normal Java method calls. Here is a typical example of registering a servlet when creating your Guice injector:

```java
   Guice.createInjector(..., new ServletModule() {

     @Override
     protected void configureServlets() {
       serve("*.html").with(MyServlet.class);
     }
   }
```
 
This registers a servlet (subclass of `HttpServlet`) called `MyServlet` to serve any web requests ending in `.html`. You can also use a path-style syntax to register servlets as you would in `web.xml`:
```java
       serve("/my/*").with(MyServlet.class);
```

== Filter Mapping ==

You may also map Servlet Filters using a very similar syntax:
```java
      filter("/*").through(MyFilter.class);
```

This will route every incoming request through `MyFilter`, and then continue to any other matching filters before finally being dispatched to a servlet for processing.

_Note: Every servlet (or filter) is required to be a `@Singleton`. If you cannot annotate the class directly, you must bind it using `bind(..).in(Singleton.class)`, separate to the `filter()` or `servlet()` rules. Mapping under any other scope is an error. This is to maintain consistency with the Servlet specification. 
Guice Servlet does not support the deprecated `SingleThreadModel`._


### Available Injections

Installing the servlet module automatically gives you access to several classes from the servlet framework. These are useful for working with the servlet programming model and are injectable in *any* Guice injected object by default, when you install the ServletModule:

```java
@RequestScoped
class SomeNonServletPojo {

  @Inject
  SomeNonServletPojo(HttpServletRequest request, HttpServletResponse response, HttpSession session) {
    ...
  }

}
```

The request and response are scoped to the current http request. Similarly the http session object is scoped to the current user session. In addition to this you may also inject the current `ServletContext` and a map of the request parameters using the binding annotation `@RequestParameters` as follows:

```java
@Inject @RequestParameters Map<String, String[]> params;
```

This must be a map of strings to arrays of strings because http allows multiple values to be bound to the same request parameter, though typically there is only one. Note that if you want access to any of the request or session scoped classes from singletons or other wider-scoped instances, you should inject a `Provider<T>` instead.

### Dispatch Order

You are free to register as many servlets and filters as you like this way. They will be compared and dispatched in the order in which the rules appear in your `ServletModule`:

```java
   Guice.createInjector(..., new ServletModule() {

     @Override
     protected void configureServlets() {
       filter("/*").through(MyFilter.class);
       filter("*.css").through(MyCssFilter.class);
       // etc..

       serve("*.html").with(MyServlet.class);
       serve("/my/*").with(MyServlet.class);
       // etc..
      }
    }
```
 
This will traverse down the list of rules in lexical order. For example, a url `/my/file.js` will first be compared against the servlet mapping:
```java
       serve("*.html").with(MyServlet.class);
```
 
And failing that, it will descend to the next servlet mapping:
```java
       serve("/my/*").with(MyServlet.class);
```
 
Since this rule matches, Guice Servlet will dispatch the request to `MyServlet` and stop the matching process. The same process is used when deciding the dispatch order of filters (that is, matched to their order of appearance in the binding list).

### Varargs Mapping

The two mapping rules above can also be written in more compact form using varargs syntax:
```java
       serve("*.html", "/my/*").with(MyServlet.class);
```
 
This way you can map several URI patterns to the same servlet. A similar syntax is also available for filter mappings.


### Using RequestScope

Each servlet configured with a `ServletModule` is executed within `RequestScope`.  

By default, there are several elements available to be injected, each of which is bound in RequestScope:
  * `HttpServletRequest` / `ServletRequest`
  * `HttpServletResponse` / `ServletResponse`
  * `RequestParameters Map<String, String[]>`

Remember to use a `Provider` when injecting either of these elements if the injection point is on a class that is created outside of `RequestScope`.  For example, a singleton servlet is created outside of a request, and so it needs to call `Provider.get()` on any `RequestScoped` dependency only after the request is received (typically it its `service()` method.

The most common way to seed a value within `RequestScope` is to add a <a href="http://download.oracle.com/docs/cd/E17802_01/products/products/servlet/2.3/javadoc/javax/servlet/Filter.html">Filter</a> in front of your servlet.  This filter should seed the scope by adding the value as a request attribute.

For example, assume we want to scope the context path of each request as a `String` so that objects involved in processing the request can have this value injected.

The `Filter` to do this might look like this:

```java
   protected Filter createUserIdScopingFilter() {
     return new Filter() {
       @Override public void doFilter(
          ServletRequest request,  ServletResponse response, FilterChain chain)
           throws IOException, ServletException {
         HttpServletRequest httpRequest = (HttpServletRequest) request;
         // ...you'd probably want more sanity checking here
         Integer userId = Integer.valueOf(httpRequest.getParameter("user-id"));
         httpRequest.setAttribute(
             Key.get(Integer.class, Names.named("user-id")).toString(),
             userId);  
         chain.doFilter(request, response);
       }

      @Override public void init(FilterConfig filterConfig) throws ServletException { }

      @Override public void destroy() { }
     };
  } 
```

And the binding might look like this:
```java
  public class YourServletModule extends ServletModule {
     @Override protected void configureServlets() {
         .....
        filter("/process-user*").through(createUserIdScopingFilter());
    }

    @Provides @Named("user-id") @RequestScoped Integer provideUserId() {
      throw new IllegalStateException("user id must be manually seeded");
    }
  }
```

And the servlet to use this might look like this:
```java
@Singleton
class UserProcessingServlet extends HttpServlet {

      private final Provider<Integer> userIdProvider;
      ....
   
      @Inject UserProcessingServlet(
         @Named("user-id") Provider<Integer> userIdProvider,
         .....) {
            this.userIdProvider = userIdProvider;
       }

      ....

     @Override public void doGet(
          HttpServletRequest req,  HttpServletResponse resp)
        throws ServletException, IOException {
             ...
            Integer userId = userIdProvider.get();
       }
}
```


        
# ServletRegexKeyMapping.md

_Advanced topics: regex mapping, init params, etc._

### Regular Expressions

You can also map servlets (or filters) to URIs using regular expressions:
```java
    serveRegex("(.)*ajax(.)*").with(MyAjaxServlet.class)
```
 
This will map any URI containing the text `"ajax"` in it to `MyAjaxServlet`. Such as:
  * `http://www.google.com/`**`ajax`**`.html`
  * `http://www.google.com/content/`**`ajax`**`/index`
  * `http://www.google.com/it/is-totally-`**`ajax`**`y`

### Initialization Parameters

Servlets (and filters) allow you to pass in string values using the `<init-param>` tag in web.xml. Guice Servlet supports this using a `Map<String, String>` of name/value pairs. For example, to initialize `MyServlet` with two parameters `coffee="Espresso"` and `site="google.com"` you could write:
```java
  Map<String, String> params = new HashMap<String, String>();
  params.put("coffee", "Espresso");
  params.put("site", "google.com");

  ...
      serve("/*").with(MyServlet.class, params)
```

And the `ServletConfig` object passed to `MyServlet` will contain the appropriate input parameters when using `getInitParams()`. The same syntax is available with filters.

### Binding Keys

You can also bind keys rather than classes. This lets you hide implementations with package-local visbility and expose them using only a Guice module and an annotation:
```java
      filter("/*").through(Key.get(Filter.class, Fave.class));
```
 
Where `Filter.class` refers to the Servlet API interface `javax.servlet.Filter` and `Fave.class` is a custom binding annotation. Elsewhere (in one of your own modules) you can bind this filter's implementation:
```java
   bind(Filter.class).annotatedWith(Fave.class).to(MyFilterImpl.class);
```
 
See the [UserGuide User's Guide] for more information on binding and annotations.

### Injecting the injector

Once you have a servlet or filter injected for you using `ServletModule`, you can access the injector at any time by simply injecting it directly into any of your classes:

```java
@Singleton
public class MyServlet extends HttpServlet {
  @Inject private Injector injector;
  ...
}

// elsewhere in ServletModule
serve("/myurl").with(MyServlet.class);
```

This is often useful for integrating third party frameworks that need to use the injector themselves.
# ServletExtensionSPI.md

_Using the servlet module SPI_

### ServletModule Extension SPI

_New in Guice 3.0_

Sometimes you want to examine your Modules and/or Bindings to inspect which URLs are serving which servlets or filters.  This is useful for tests or to assert preconditions in your code.  The !ServletModule extension SPI, built on the [[core extension SPI|ExtensionSPI]], lets you do this the same way you would inspect a binding using the normal [[elements SPI|InspectingModules]].

#### Inspecting Servlet Bindings

[ServletModuleTargetVisitor](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/servlet/ServletModuleTargetVisitor.html) is an extension to the core [BindingTargetVisitor](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/BindingTargetVisitor.html).  You can implement this interface and call `binding.acceptTargetVisitor(myVisitor)` to learn details about servlet bindings.

```java
  boolean isBindingForUri(Binding<?> binding, String uri) {
    return binding.acceptTargetVisitor(new Visitor(uri));
  }

  private static class Visitor
      extends DefaultBindingTargetVisitor<Object, Boolean>
      implements ServletModuleTargetVisitor<Object, Boolean> {
    private final String uri;
    
    Visitor(String uri) {
      this.uri = uri;
    }

    @Override boolean visitOther(Binding<?> binding) {
      return false;
    }

    @Override boolean visit(InstanceFilterBinding binding) {
      return matchesUri(binding);
    } 

    @Override boolean visit(InstanceServletBinding binding) {
      return matchesUri(binding);
    }

    @Override boolean visit(LinkedFilterBinding binding) {
      return matchesUri(binding);
    }

    @Override boolean visit(LinkedServletBinding binding) {
      return matchesUri(binding);
    }

    private boolean matchesUri(ServletModuleBinding binding) {
      return binding.matchesUri(uri);
    }
  }
```

These visitors will work both on bindings retrieved from an Injector and bindings retrieved from Elements.
# GoogleAppEngine.md

### Using Guice with Google App Engine

You can use Guice with to write modular applications for [Google App Engine](http://code.google.com/appengine/).

### Supported Builds
Google App Engine support requires Guice 2 (with or without AOP), plus the guice-servlet extension.

### Setup

##### Servlet and Filter Registration
[[Configure servlets and filters|ServletModule]] by subclassing `ServletModule`:
```java
package com.mycompany.myproject;

import com.google.inject.servlet.ServletModule;

class MyServletModule extends ServletModule {
  @Override protected void configureServlets() {
    serve("/*").with(MyServlet.class);
  }
}
```

##### Injector Creation
Construct your Guice injector in the `getInjector()` method of a class that extends GuiceServletContextListener. Be sure to include your application's servlet module in the list of modules.
```java
package com.mycompany.myproject;

import com.google.inject.servlet.ServletModule;
import com.google.inject.servlet.GuiceServletContextListener;
import com.google.inject.Guice;
import com.google.inject.Injector;

public class MyGuiceServletContextListener extends GuiceServletContextListener {

  @Override protected Injector getInjector() {
    return Guice.createInjector(
        new MyServletModule(),
        new BusinessLogicModule());
  }
}
```

##### Web.xml Configuration
You must register both the [[GuiceFilter|Servlets]] and your subclass of [GuiceServletContextListener](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/servlet/GuiceServletContextListener.html) in your application's `web.xml` file. All other servlets and filters may be configured in your servlet module.
```xml
<?xml version="1.0" encoding="ISO-8859-1"?>
<web-app 
   xmlns="http://java.sun.com/xml/ns/javaee" 
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"
   version="2.5"> 
  <display-name>My Project</display-name>
  
 <filter>
    <filter-name>guiceFilter</filter-name>
    <filter-class>com.google.inject.servlet.GuiceFilter</filter-class>
  </filter>

  <filter-mapping>
    <filter-name>guiceFilter</filter-name>
    <url-pattern>/*</url-pattern>
  </filter-mapping>

  <listener>
    <listener-class>com.mycompany.myproject.MyGuiceServletContextListener</listener-class>
  </listener>
</web-app>
```

##### WAR Layout
Ensure the AOP alliance, Guice, and Guice servlet jars are in the `WEB-INF/lib` directory of your `.war` file (or `www` directory):
```xml
  www/
      WEB-INF/
          lib/
              aopalliance.jar
              guice-servlet-snapshot.jar
              guice-snapshot.jar
              ...
          classes/
              ...
          appengine-web.xml
          web.xml
```
# GuicePersist.md

_Getting started with Guice Persist_
_New in Guice 3.0_

### Introduction

Guice Persist provides abstractions for working with datastores and persistence providers in your Guice applications. It works in a normal Java desktop or server application, inside a plain Servlet environment, or even a Java EE container.

Guice Persist supports a simple approach to transactions and various unit-of-work semantics; both with the same declarative, annotation-based API. Switching between unit-of-work strategies is as simple as changing a configuration option.

It also follows the Guice idioms of using simple, expressive configuration and tries to remain type-safe as far as possible. 

##### Support

We currently support the following persistence providers:
  * JPA (Java Persistence API) - with any compliant vendor implementation.

And a simple mechanism to extend to other persistence providers.

##### Getting Started

To begin, you must obtain a copy of `guice-persist.jar` from the Guice 3.0 download.
# JPA.md

_Using JPA with Guice Persist._

### Java Persistence API (JPA)

JPA is a standard released as part of JSR-220 (or EJB3). It is roughly an analog to Hibernate or Oracle TopLink; and they are also the two most prominent vendors of JPA. Guice Persist supports JPA in a vendor-agnostic fashion, so it should be easy to swap between implementations (such as AppEngine's datastore) should you need to.

##### Enabling Persistence Support

To enable persistence support, simply install the JPA module:

```java
Injector injector = Guice.createInjector(..., new JpaPersistModule("myFirstJpaUnit"));
```
 
In JPA, you specify your configuration in a *persistence.xml* file in the `META-INF` folder of that is available on the classpath (if inside a jar, then at the root for example). In this file you declare several _Persistence Units_ that are roughly different profiles (for example, backed by different databases). Here is an example of a simple JPA configuration:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<persistence xmlns="http://java.sun.com/xml/ns/persistence"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://java.sun.com/xml/ns/persistence
	http://java.sun.com/xml/ns/persistence/persistence_1_0.xsd" version="1.0">

	<!-- A JPA Persistence Unit -->
	<persistence-unit name="myFirstJpaUnit" transaction-type="RESOURCE_LOCAL">
		<provider>org.hibernate.ejb.HibernatePersistence</provider>
 
		<!-- JPA entities must be registered here -->
		<class>com.wideplay.warp.jpa.JpaTestEntity</class>

		<properties>
			<!-- vendor-specific properties go here -->
		</properties>
	</persistence-unit>

</persistence>
```

To tell Guice Persist which persistence unit you wish you use, you specify its name when creating your module:

```java
Injector injector = Guice.createInjector(..., new JpaPersistModule("myFirstJpaUnit"));
```

Finally, you must decide when the persistence service is to be started by invoking `start()` on `PersistService`. I typically use a simple initializer class that I can trigger at a time of my choosing:

```java
public class MyInitializer { 
	@Inject MyInitializer(PersistService service) {
		service.start(); 
 
 		// At this point JPA is started and ready.
	} 
}
```

It makes good sense to use Guice's [Service API] to start all services in your application at once. I recommend you think about how it fits into your particular deployment environment when doing so. 

However, in the case of web applications, you do not need to do anything if you simply install the `PersistFilter` (see below).

##### Session-per-transaction strategy

This is a popular strategy, where a JPA `EntityManager` is created and destroyed around each database transaction. Set the transaction-type attribute on your persistence unit's configuration:

```xml
 <persistence-unit name="myFirstJpaUnit" transaction-type="RESOURCE_LOCAL">
```

##### Using the EntityManager inside transactions
Once you have the injector created, you can freely inject and use an `EntityManager` in your transactional services:

```java
import com.google.inject.persist.Transactional;
import javax.persistence.EntityManager; 
 
public class MyService {
	@Inject EntityManager em; 
 
	@Transactional 
	public void createNewPerson() {
		em.persist(new Person(...)); 
	} 
}
```
 
This is known as the _session-per-transaction_ strategy. For more information on transactions and units of work (sessions), see [[this page|Transactions]].

Note that if you make `MyService` a `@Singleton`, then you should inject `Provider<EntityManager>` instead.

##### Web Environments (session-per-http-request)

So far, we've seen the session-per-transaction strategy. In web environments this is atypical, and generally a _session-per-http-request_ is preferred (sometimes also called _open-session-in-view_). To enable this strategy, you first need to add a filter to your `ServletModule`:

```java
public class MyModule extends ServletModule {
  protected void configureServlets() {
    install(new JpaPersistModule("myJpaUnit"));  // like we saw earlier.

    filter("/*").through(PersistFilter.class);
  }
}
```
 
You should typically install this filter before any others that require the `EntityManager` to do their work.
 
Note that with this configuration you can run multiple transactions within the same `EntityManager` (i.e. the same HTTP request).
# Transactions.md

_Transactions with Guice Persist_

### Method-level @Transactional

By default, any method marked with `@Transactional` will have a transaction started before, and committed after it is called. 

```java
  @Transactional
  public void myMethod() { ... }
```

The only restriction is that these methods must be on objects that were created by Guice and they must not be `private`.

### Responding to exceptions

If a transactional method encounters an unchecked exception (any kind of `RuntimeException`), the transaction will be rolled back. Checked exceptions are ignored and the transaction will be committed anyway.

To change this behavior, you can specify your own exceptions (checked or unchecked) on a per-method basis:

```java
  @Transactional(rollbackOn = IOException.class)
  public void myMethod() throws IOException { ... }
```
 
Once you specify a `rollbackOn` clause, only the given exceptions and their subclasses will be considered for rollback. Everything else will be committed. Note that you can specify any combination of exceptions using array literal syntax:

```java
  @Transactional(rollbackOn = { IOException.class, RuntimeException.class, ... })
```
 
It is sometimes necessary to have some general exception types you want to rollback but particular subtypes that are still allowed. This is also possible using the `ignore` clause:

```java
  @Transactional(rollbackOn = IOException.class, ignore = FileNotFoundException.class)
```
 
In the above case, any `IOException` (and any of its subtypes) except FileNotFoundException will trigger a rollback. In the case of `FileNotFoundException`, a commit will be performed and the exception propagated to the caller anyway. 

_Note that you can specify any combination of checked or unchecked exceptions._

### Units of Work 

A unit of work is roughly the lifespan of an `EntityManager` (in JPA). It is the _session_ referred to in the `session-per-*` strategies. We have so far seen how to create a unit of work that spans a single transaction:

http://warp-persist.googlecode.com/svn/wiki/txn_unitofwork.png

...and one that spans an entire HTTP request:

http://warp-persist.googlecode.com/svn/wiki/request_unitofwork.png

Sometimes you need to define a custom unit-of-work that doesn't fit into either requests or transactions. For example, you may want to do some background work in a timer thread, or some initialization work during startup. Or perhaps you are making a desktop app and have some other idea of a unit of work.

http://warp-persist.googlecode.com/svn/wiki/custom_unitofwork.png

To start and end a unit of work arbitrarily, inject the `UnitOfWork` interface:

```java
public class MyBackgroundWorker {
  @Inject private UnitOfWork unitOfWork;
 
  public void doSomeWork() {
    unitOfWork.begin();
    try { 
      // Do transactions, queries, etc...
      //...
    } finally {
      unitOfWork.end(); 
    }
  } 
}
```
 
You are free to call any `@Transactional` methods while a unit of work is in progress this way. When `end()` is called, any existing session is closed and discarded. It is safe to call `begin()` multiple times--if a unit of work is in progress, nothing happens. Similarly, if one is ended calling `end()` returns silently. `UnitOfWork` is threadsafe and can be cached for multiple uses or injected directly into singletons.
# GuicePersistMultiModules.md

_Multiple persistence providers with Guice Persist_

### Multiple Modules

So far we've seen how to use Guice Persist and JPA with a single persistence unit. This typically maps to one data store. Guice Persist supports using multiple persistence modules using the `private modules` API.

This means that you must be careful to install all JPA modules only in individual private modules:

```java
Module one = new PrivateModule() {
  protected void configure() {
    install(new JpaModule("persistenceUnitOne"));
    // other services...
  }
};

Module two = new PrivateModule() {
  protected void configure() {
    install(new JpaModule("persistenceUnitTwo"));
    // other services...
  }
};

Guice.createInjector(one, two);
```

The injector now contains two parallel sets of bindings which are isolated from each other. When you bind services in module `one` those services will get the `EntityManager` from `persistenceUnitOne` and transactions around its database.

Similarly services in module `two` will have their own transactions, `EntityManager`s etc., and these should typically not cross.
# JSR330.md

### JSR-330 Integration

_New in Guice 3.0_

[JSR-330](http://code.google.com/p/atinject/) standardizes annotations like `@Inject` and the `Provider` interfaces for Java platforms. It doesn't currently specify how applications are configured, so it has no analog to Guice's modules.

Guice implements a complete JSR-330 injector. This table summarizes the JSR-330 types and their Guice equivalents.

 *JSR-330*<br>javax.inject | *Guice* <br>com.google.inject |                                    |
---------------------------|-------------------------------|------------------------------------|
 [@Inject](http://atinject.googlecode.com/svn/trunk/javadoc/javax/inject/Inject.html) | [@Inject](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/Inject.html) | Interchangeable, but JSR-330 places additional constraints on injection points. Fields must be non-final. Optional injection is not supported. Methods must be non-abstract and not have type parameters of their own.  Additionally, method overriding differs in a key way:  If a class being injected overrides a method where the superclass' method was annotated with javax.inject.Inject, but the subclass method is not annotated, then the method will not be injected.
 [@Named](http://atinject.googlecode.com/svn/trunk/javadoc/javax/inject/Named.html) | [@Named](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/name/Named.html) | Interchangeable.
 [@Qualifier](http://atinject.googlecode.com/svn/trunk/javadoc/javax/inject/Qualifier.html) | [@BindingAnnotation](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/BindingAnnotation.html) | Interchangeable.
 [@Scope](http://atinject.googlecode.com/svn/trunk/javadoc/javax/inject/Scope.html) | [@ScopeAnnotation](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/ScopeAnnotation.html) | Interchangeable.
 [@Singleton](http://atinject.googlecode.com/svn/trunk/javadoc/javax/inject/Singleton.html) | [@Singleton](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/Singleton.html) | Interchangeable.
 [Provider](http://atinject.googlecode.com/svn/trunk/javadoc/javax/inject/Provider.html) | [Provider](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/Provider.html) | Guice's Provider extends JSR-330's Provider. Use Providers.guicify() to convert a JSR-330 provider into a Guice provider.

### Best Practices
Prefer JSR-330's annotations and Provider interface.
# OSGi.md

_Using Guice in an OSGi container_
### OSGi
[OSGi](http://www.osgi.org/) is a dynamic module system used in many projects including [Eclipse](http://www.eclipse.org) and [GlassFish](https://glassfish.dev.java.net/). A module in OSGi is known as a bundle - basically a JAR with additional metadata. This metadata describes the bundle's identity, what packages it needs, and what it provides. Guice added OSGi metadata in its second release, which means you can now install it as a bundle on any modern OSGi container.

OSGi containers also provide a shared service registry, which supports dynamic collaboration between bundles by letting them publish and consume services on-the-fly in the same JVM. Guice can integrate with the OSGi service registry via the third-party *[peaberry](http://code.google.com/p/peaberry/)* extension.

Can I use Guice in an OSGi environment without peaberry?
Short answer: yes!

Long answer: Guice is a valid OSGi bundle which contains code to transparently support AOP inside OSGi containers. You can share module bindings and injectors between your OSGi bundles and use method interception on public methods, without needing the peaberry extension.

##### What are the benefits of peaberry?
peaberry exposes a fluent API to publish and consume _dynamic services_. It provides integration with the OSGi service registry out of the box and has plug-in support for other registry-based service frameworks. Think of peaberry as the equivalent of [Spring Dynamic-Modules](http://www.springsource.org/osgi), but with the type-safety and performance of Guice, and the ability to work with other service frameworks - not just OSGi.

With peaberry you can inject OSGi services just like any other Guice binding. The bindings remain valid and transparently continue to work as services are updated. You can inject `Export` handles that let you publish and unpublish instances (or bound types) as OSGi services. You can even use a form of _outjection_ to watch for changes in injected services.

Last, but definitely not least, peaberry brings OSGi service support to your existing Guice application without introducing any OSGi specific types. You can switch between a fixed Java instance or an OSGi service for the same injection point just by changing your binding modules.

##### What are the constraints of using Guice's OSGi support?
Guice uses [[bytecode generation|ClassLoading]] for AOP, faster reflection, and to proxy circular dependencies. The generated bytecode must be able to see both the original client type (available from the client classloader) and AOP internals (available from the Guice classloader), but in OSGi these classloaders will be isolated from one another. The client classloader will only see its own types and imports, while Guice will only see its own types and imports. 

To support AOP in OSGi without requiring clients to know about AOP internals, or vice-versa, Guice automatically creates and manages _bridge_ classloaders that bridge between client and Guice classloaders. The bridge classloaders also support eager unloading of proxy classes, which reduces the potential for memory leaks when reloading bundles.

This is only possible when the types involved are *not package-private* (because by definition package-private types are not visible from other classloaders). If you attempt to use AOP on package-private types while running inside an OSGi container you will see the following exception:
```java
 ClassNotFoundException: com.google.inject.internal.cglib.core.ReflectUtils
```
because bridging wasn't possible. If you need to use AOP with package-private types or methods in OSGi then you can always put Guice and the AOP Alliance JAR on the main classpath and use the `org.osgi.framework.bootdelegation` property to expose the Guice packages, like so:
```txt
 "-Dorg.osgi.framework.bootdelegation=org.aopalliance.*,com.google.inject.*"
```
You can still dynamically unload and reload bundles when using bootdelegation with Guice, but you would need to restart the JVM to replace the Guice bundle itself. This is much less likely to happen compared to upgrading client bundles, and it's usually fine to do a full shutdown when a key production service is upgraded. This also avoids potential problems like serialization-compatibility for HTTP session objects.
# Struts2Integration.md

### Struts 2 Integration

To install the Guice Struts 2 plugin with Struts 2.2 or later, simply include `guice-struts2-plugin-3.0.jar` in your web application's classpath, 
create a subclass of GuiceServletContextListener that implements getInjector and adds any servlets and filters (using ServletModule) and the Struts2GuicePluginModule, and a web.xml file that references your GuiceServletContextListener subclass as a listener.  For examples, look at the example [web.xml](https://github.com/google/guice/blob/master/extensions/struts2/example/root/WEB-INF/web.xml) and [GuiceServletContextListener subclass](https://github.com/google/guice/blob/master/extensions/struts2/example/src/com/google/inject/struts2/example/ExampleListener.java).

Guice will inject all of your Struts 2 objects including actions and interceptors. You can even scope your actions.
#### A Counting Example
Say for example that we want to count the number of requests in a session. Define a `Counter` object which will live on the session:
```java
@SessionScoped
public class Counter {

  int count = 0;

  /** Increments the count and returns the new value. */
  public synchronized int increment() {
    return count++;
  }
}
```
Next, we can inject our counter into an action:
```java
public class Count {

  final Counter counter;

  @Inject
  public Count(Counter counter) {
    this.counter = counter;
  }

  public String execute() {
    return SUCCESS;
  }

  public int getCount() {
    return counter.increment();
  }
}
```
Then create a mapping for our action in our struts.xml file:
```xml
<action name="Count"
    class="mypackage.Count">
  <result>/WEB-INF/Counter.jsp</result>
</action>     
```
And a JSP to render the result:
```jsp
<%@ taglib prefix="s" uri="/struts-tags" %>

<html>   
  <body>
    <h1>Counter Example</h1>
    <h3><b>Hits in this session:</b>
      <s:property value="count"/></h3>
  </body>
</html>
```
We actually made this example more complicated than necessary in an attempt to illustrate more concepts. In reality, we could have done away with the separate `Counter` object and applied `@SessionScoped` to our action directly.

#### Struts2 support in earlier versions of Guice
In Guice 1.0, the struts2 extension supports struts 2.0.6 or later, and is included using the `guice-struts2-plugin-1.0.jar`.  The struts2 extension included in Guice 2.0 did not work properly.

You must select Guice as your ObjectFactory implementation in your `struts.xml` file:
```xml
<constant name="struts.objectFactory" value="guice" />
```
You can optionally specify a `Module` for Guice to install in your `struts.xml` file:
```xml
<constant name="guice.module" value="mypackage.MyModule"/>
```
If all of your bindings are implicit, you can get away without defining a module at all.
# CustomInjections.md

_Performing custom injections with type and injection listeners_
### Custom Injections
In addition to the standard `@Inject`-driven injections, Guice includes hooks for custom injections. This enables Guice to host other frameworks that have their own injection semantics or annotations. Most developers won't use custom injections directly; but they may see their use in extensions and third-party libraries. Each custom injection requires a type listener, an injection listener, and registration of each.

[TypeListeners](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/TypeListener.html) get notified of the types that Guice injects. Since type listeners are only notified once-per-type, they are the best place to run potentially slow operations, such as enumerating a type's members. With their inspection complete, type listeners may register instance listeners for values as they're injected.

[MembersInjectors](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/MembersInjector.html) and [InjectionListeners](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/InjectionListener.html) can be used to receive a callback after Guice has injected an instance. The instance is first injected by Guice, then by the custom members injectors, and finally the injection listeners are notified. Since they're notified once-per-instance, these should execute as quickly as possible.

#### Example: Injecting a Log4J Logger
Guice includes built-in support for injecting `java.util.Logger` instances that are named using the type of the injected instance. With the type listener API, you can inject a `org.apache.log4j.Logger` with the same high-fidelity naming. We'll be injecting fields of this format:
```java
import org.apache.log4j.Logger;
import com.publicobject.log4j.InjectLogger;

public class PaymentService {
  @InjectLogger Logger logger;

  ...
}
```
We start by registering our custom type listener `Log4JTypeListener` in a module. We use a matcher to select which types we listen for:
```java
  @Override protected void configure() {
    bindListener(Matchers.any(), new Log4JTypeListener());
  }
```
You can implement the [TypeListener](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/TypeListener.html) to scan through  a type's fields, looking for Log4J loggers. For each logger field that's encountered, we register a `Log4JMembersInjector` on the passed-in [TypeEncounter](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/TypeEncounter.html):
```java
  class Log4JTypeListener implements TypeListener {
    public <T> void hear(TypeLiteral<T> typeLiteral, TypeEncounter<T> typeEncounter) {
      for (Field field : typeLiteral.getRawType().getDeclaredFields()) {
        if (field.getType() == Logger.class
            && field.isAnnotationPresent(InjectLogger.class)) {
          typeEncounter.register(new Log4JMembersInjector<T>(field));
        }
      }
    }
  }
```
Finally, we implement the `Log4JMembersInjector` to set the logger. In this example, we always set the field to the same instance. In your application you may need to compute a value or request one from a provider.
```java
  class Log4JMembersInjector<T> implements MembersInjector<T> {
    private final Field field;
    private final Logger logger;

    Log4JMembersInjector(Field field) {
      this.field = field;
      this.logger = Logger.getLogger(field.getDeclaringClass());
      field.setAccessible(true);
    }

    public void injectMembers(T t) {
      try {
        field.set(t, logger);
      } catch (IllegalAccessException e) {
        throw new RuntimeException(e);
      }
    }
  }
```
# 3rdPartyModules.md

_3rd party Guice addons_
###Third Party Modules

#### Infrastructure
  * **[Automatic-Injection](http://github.com/manzke/guice-automatic-injection)** - classpath scanning for automatic bindings.
  * **[Peaberry](http://code.google.com/p/peaberry/)** - OSGi
  * **[Concurrent Singletons](http://tembrel.blogspot.com/2009/11/concurrently-initialized-singletons-in.html)**
  * **[JBoss Microcontainer](http://anonsvn.jboss.org/repos/jbossas/projects/microcontainer/trunk/guice-int/)**
  * **[GuiceRemoteServiceServlet](http://pavelgj.blogspot.com/2008/02/gwt-remoteserviceservlet-guice.html)** - GWT RemoteServiceServlet
  * **[JCatapult](http://www.jcatapult.org/)** - a modular app framework
  * **[GuiceyFruit](http://code.google.com/p/guiceyfruit/)** - lifecycle, JSR 250, EJB 3, Spring lifecycles, JNDI
  * **[GuiceHttpInvoker](http://code.google.com/p/garbagecollected/wiki/GuiceHttpInvoker)** - The Spring Framework's HttpInvoker
  * **[Dimdwarf](http://dimdwarf.sourceforge.net/)** - lightweight application server
  * **[Gabriel](http://gabriel.codehaus.org/)** - permission-based security
  * **[Rocoto](http://www.99soft.org/projects/rocoto/3.2/)** - expanded properties file parsing

#### Mobile
  * **[RoboGuice](http://code.google.com/p/roboguice/)** - Guice for Android.

#### Testing
  * **[GuiceBerry](http://code.google.com/p/guiceberry/)** - JUnit and integration testing.
  * **[AtUnit](http://code.google.com/p/atunit/)** - JUnit and mocking frameworks (JMock, EasyMock)

#### Persistence
  * **[Warp-Persist](http://www.wideplay.com/guicewebextensions2)** and **[Dynamic Finders](http://www.wideplay.com/dynamicfinders)** - Hibernate, JPA, db4o and [neodatis](http://code.google.com/p/warp-persist-neodatis/). Supports `@Transactional` and query automation.
  * **[Guice-BlazeDs](http://code.google.com/p/guice-blazeds/)** - warp-persist and BlazeDs for Flex

#### Web frameworks
  * **[GIN](http://code.google.com/p/google-gin/)** - [GWT](http://code.google.com/webtoolkit/)
  * **[DWR-Guice Integration](http://dev.priorartisans.com/tim/dwr-guice/index.html)** - [Tim Peierls integrated](http://tembrel.blogspot.com/2007/04/guice-support-in-dwr.html) [DWR](http://getahead.org/dwr) (Direct Web Remoting).
  * **[Wicket](http://herebebeasties.com/2007-06-20/wicket-gets-guicy)** - a simple Java web framework
  * **[Stripes-Guicer](http://code.google.com/p/stripes-guicer/)** - [Stripes](http://www.stripesframework.org/)
  * **[Seam](http://docs.jboss.com/seam/2.1.2/reference/en-US/html/guice.html)** - integrates EJB and JSF. See also [LabsSeamGuice](http://www.jboss.org/community/docs/DOC-11242), [Pawel’s Blog](http://pawel.wrzesz.cz/blog/?p=4).
  * **[Guicesf](http://code.google.com/p/guicesf/)** - more JavaServer Faces
  * **[Guice and JSF](http://notdennisbyrne.blogspot.com/2007/09/integrating-guice-and-jsf.html)** - [JavaServer Faces](http://java.sun.com/javaee/javaserverfaces/) (also see [part 2](http://notdennisbyrne.blogspot.com/2007/10/integrating-guice-and-jsf-part-2.html))
  * **[JRest4Guice](http://code.google.com/p/jrest4guice/)** - a restful web framework
  * **[Restlet](http://tembrel.blogspot.com/2008/07/resource-dependency-injection-in.html)**

#### Other Languages
  * **[Groovy-Guice](http://code.google.com/p/groovy-guice/)** - Groovy
  * **[Ninject](http://ninject.org/)** - .NET
  * **[Snake Guice](http://code.google.com/p/snake-guice/)** - Python
  * **[Smartypants-IOC](http://code.google.com/p/smartypants-ioc/)** - Flex/AS3
  * **[Dawn](http://github.com/sammyt/dawn)** - AS3
# ExtendingGuice.md

_Guice's SPI for authors of tools, extensions and plugins_

The [service provider interface](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/package-summary.html) exposes Guice's internal models to aid in the development in tools, extensions, and plugins. 

### Core Abstractions

|    |    | 
|----|----|
|[InjectionPoint](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/InjectionPoint.html) | A constructor, field or method that can receive injections. Typically this is a member with the `@Inject` annotation. For non-private, no argument constructors, the member may omit the annotation. Each injection point has a collection of dependencies. |
|[Key](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/Key.html) | A type, plus an optional binding annotation.|
|[Dependency](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/Dependency.html) | A key, optionally associated to an injection point. These exist for injectable fields, and for the parameters of injectable methods and constructors. Dependencies know whether they accept null values via an `@Nullable` annotation.|
|[Element](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/Element.html) | A configuration unit, such as a `bind` or `requestInjection` statement. Elements are [visitable](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/ElementVisitor.html). |
|[Module](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/Module.html) | A collection of configuration elements. [Extracting the elements](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/Elements.html#getElements(java.lang.Iterable)) of a module enables **static analysis** and **code-rewriting**. You can inspect, rewrite, and validate these elements, and use them to build new modules. |
|[Injector](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/Injector.html) | Manages the application's object graph, as specified by modules. SPI access to the injector works like **reflection**. It can be used to retrieve the application's bindings and dependency graph. |

The [[Elements SPI|InspectingModules]] page has more information about using these classes.
<p>

### Abstractions for Extension Authors

_(New in Guice 3.0)_

|    |    | 
|----|----|
[@Toolable](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/Toolable.html) | An annotation used on methods also annotated with `@Inject`.  This instructs Guice to inject the method even in `Stage.TOOL`.  Typically used for extensions that need to gather information to implement `HasDependencies` or validate requirements and fail early for easier testing.
[ProviderWithExtensionVisitor](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/ProviderWithExtensionVisitor.html) | An interface that provider instances implement to allow extensions to visit custom subinterfaces of [BindingTargetVisitor](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/BindingTargetVisitor.html).  See [[Extensions SPI|ExtensionSPI]] for more information.

The [[Extensions SPI|ExtensionSPI]] page has more information about writing extensions that expose an SPI

### Examples

Log a warning for each static injection in your Modules:
```java
  public void warnOfStaticInjections(Module... modules) { 
    for (Element element : Elements.getElements(modules)) { 
      element.acceptVisitor(new DefaultElementVisitor<Void>() { 
        @Override 
        public Void visit(StaticInjectionRequest element) { 
          logger.warning("Static injection is fragile! Please fix " 
              + element.getType().getName() + " at " + element.getSource()); 
          return null; 
        } 
      }); 
    } 
  } 
```
# InspectingModules.md

_Inspecting what makes up a Module & examining Bindings_


### Elements SPI

Guice provides a rich service provider interface (SPI) for all the elements that make up a module.  You can use this SPI to learn what bindings are in a module and even rewrite some of those bindings.  [Modules.override](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/util/Modules.html#override(com.google.inject.Module...)) is written entirely using this SPI.

Examining Elements

The [Elements](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/Elements.html) class provides access to each [Element](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/Element.html) within a Module.  You can use this class to get individual elements from a module and take action upon them.  Each Element has an acceptVisitor method that takes an [ElementVisitor](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/ElementVisitor.html).  You can use [DefaultElementVisitor](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/DefaultElementVisitor.html) to simplify writing a visitor.

```java
  void warnAboutStaticInjections(Module... modules) {
    for(Element element : Elements.getElements(modules)) {
      element.acceptVisitor(new DefaultElementVisitor<Void>() {
          @Override public void visit(StaticInjectionRequest request) {
            System.out.println("You shouldn't be using static injection at: " + request.getSource());
          }
      });
    }
  }
```

#### Manipulating Modules

You can use the SPI to do anything from stripping out elements to rewriting elements to adding new elements.  When you've finished wiring your elements, you can turn them back into a module.

```java
  Module stripStaticInjections(Module... modules) {
    final List<Element> noStatics = Lists.newArrayList();
    for(Element element : Elements.getElements(modules)) {
      element.acceptVisitor(new DefaultElementVisitor<Void>() {
        @Override public void visit(StaticInjectionRequest request) {
          // override to not call visitOther
        }

        @Override public void visitOther(Element element) {
          noStatics.add(element);
        }
      });
    }
    return Elements.getModule(noStatics);
  }
```

#### Binding Binoculars

The most common Element is a [[Binding|Bindings]].  Bindings have more configuration than other elements and can be inspected in more ways.  You can visit a binding's scope using [Binding.acceptScopingVisitor](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/Binding.html#acceptScopingVisitor)), or figure out what kind of binding it is using [Binding.acceptTargetVisitor](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/Binding.html#acceptTargetVisitor).  Each of these methods have their own default visitors ([DefaultBindingScopingVisitor](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/DefaultBindingScopingVisitor.html) and [DefaultBindingTargetVisitor](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spi/DefaultBindingTargetVisitor.html), respectively) to make visiting easier.

Bindings can either be "Module bindings" or "Injector bindings".  Injector bindings are bindings retrieved from an injector (see the [Injector javadoc](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/Injector.html)).  Module bindings are bindings retrieved using the Elements SPI.  The [Binding javadoc](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/Binding.html) explains the difference between these in more detail.

#### Extensions

Guice 3.0 adds the ability for extensions to write their own SPI.  See [[ExtensionSPI]] for details.
# ExtensionSPI.md

_Interaction between Guice SPI & Extensions_

### Extensions SPI

_New in Guice 3.0_

In Guice 2.0 we expanded our SPI to [[expose rich details about modules and injectors|InspectingModules]]. The elements API can inspect, extend and even rewrite Guice configuration. It's an essential tool used by GIN, Modules.override(), Grapher, Guiceberry's controllable injection, and others. In Guice 3.0 we expanded the SPI to include extensions!  Extensions such as Multibinder, Assisted Inject, Servlets and others can expose their details through programmatic inspection.

#### How Extensions Participate
An extension author needs to do two things.  First it must create a subinterface of `BindingTargetVisitor` that is specific to its extension.  For example, the servlet extension has `ServletModuleTargetVisitor`, and the Multibinder extension has `MultibindingsTargetVisitor`.  The extension must also bind the extension's provider instance to a `ProviderWithExtensionVisitor`.  The implementation of acceptExtensionVisitor must do an instanceof check on the `BindingTargetVisitor` and call the appropriate visit method.  If the visitor is an instance of the extension's visitor, it must visit the appropriate extension method, otherwise it must visit the normal `ProviderInstanceBinding`.
For example, this is what mapbinder does:
```java
  public <R, B> R acceptExtensionVisitor(BindingTargetVisitor<B, R> visitor,
      ProviderInstanceBinding<? extends B> binding) {
    if (visitor instanceof MultibindingsTargetVisitor) {
      return ((MultibindingsTargetVisitor<Map<K, V>, R>)visitor).visit(this);
    } else {
      return visitor.visit(binding);
    }
  }
```

#### How Users Participate
To visit an extension's custom methods, users must implement the extension's custom visitor interface and call `binding.acceptTargetVisitor(myVisitor)`.  For example:

```java
  public static void main(String[] args) {
    Injector injector = Guice.createInjector(new PorscheModule());

    for (Binding<?> binding : injector.getBindings().values()) {
      System.out.println(binding.acceptTargetVisitor(new MultibindVisitor()));
    }
  }

  static class MultibindVisitor extends DefaultBindingTargetVisitor<?, String>
      implements MultibindingsTargetVisitor<?, String> {
    public String visit(MultibinderBinding<?> multibinder) {
      return "Key: " + multibinder.getSetKey()
          + " uses bindings: " + multibinder.getElements()
          + " permitsDuplicates: " + multibinder.permitsDuplicates();
    }
    public String visit(MapBinderBinding<?> mapbinder) {
      return "Key: " + mapbinder.getMapKey()
          + " uses entries: " + mapbinder.getEntries()
          + " permitsDuplicates: " + mapbinder.permitsDuplicates();
    }
    protected String visitOther(Binding<?> binding) {
      return binding.toString();
    }
  }
```
# AssistedInject.md

_an easier way to get Guice to build auto-wired factories._
### AssistedInject

Factories are a well established pattern for creating value objects, model/domain objects (entities), or objects that combine parameterization and dependencies.  Factories can be brittle and contain a lot of boilerplate.  Guice can eliminate a lot of that boilerplate by auto-generating Factory implementations from simple interfaces.  This process is (possibly misleadingly) known as assisted injection.

##### Factories by Hand
Sometimes a class gets some of its constructor parameters from the Guice Injector and others from the caller:
```java
public class RealPayment implements Payment {
  public RealPayment(
        CreditService creditService,  // from the Injector
        AuthService authService,  // from the Injector
        Date startDate, // from the instance's creator
        Money amount); // from the instance's creator
  }
  ...
}
```
The standard solution to this problem is to write a factory that helps Guice build the objects:
```java
public interface PaymentFactory {
  public Payment create(Date startDate, Money amount);
}
```
```java
public class RealPaymentFactory implements PaymentFactory {
  private final Provider<CreditService> creditServiceProvider;
  private final Provider<AuthService> authServiceProvider;

  @Inject
  public RealPaymentFactory(Provider<CreditService> creditServiceProvider,
      Provider<AuthService> authServiceProvider) {
    this.creditServiceProvider = creditServiceProvider;
    this.authServiceProvider = authServiceProvider;
  }

  public Payment create(Date startDate, Money amount) {
    return new RealPayment(creditServiceProvider.get(),
      authServiceProvider.get(), startDate, amount);
  }
}
```
...and a corresponding binding in the module:
```java
   bind(PaymentFactory.class).to(RealPaymentFactory.class);
```
It's annoying to write the boilerplate factory class each time this situation arises. It's also annoying to update the factories when the implementation class' dependencies change. 


##### Factories by AssistedInject
AssistedInject generates an implementation of the factory class automatically. To use it, annotate the implementation class' constructor and the fields that aren't known by the injector:
```java
public class RealPayment implements Payment {
  @Inject
  public RealPayment(
        CreditService creditService,
        AuthService authService,
        @Assisted Date startDate,
        @Assisted Money amount);
  }
  ...
}
```
Then bind a `Provider<Factory>` in the Guice module:
##### AssistedInject in Guice 2.0
```java
bind(PaymentFactory.class).toProvider(
    FactoryProvider.newFactory(PaymentFactory.class, RealPayment.class));
```

##### AssistedInject in Guice 3.0
Guice 3.0 improves AssistedInject by offering the ability to allow different factory methods to return different types or different constructors from one type.  See the [FactoryModuleBuilder javadoc](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/assistedinject/FactoryModuleBuilder.html) for complete details.
```java
install(new FactoryModuleBuilder()
     .implement(Payment.class, RealPayment.class)
     .build(PaymentFactory.class));
```

##### How & Why

AssistedInject maps the `create()` method's parameters to the corresponding `@Assisted` parameters in the implementation class' constructor. For the other constructor arguments, it asks the regular Injector to provide values. 

With AssistedInject, it's easier to create classes that need extra arguments at construction time:
  1. Annotate the constructor and assisted parameters on the implementation class (such as `RealPayment`) 
  2. Create a factory interface with a `create()` method that takes only the assisted parameters. Make sure they're in the same order as in the constructor
  3. Bind that factory to a provider created by AssistedInject.

##### Inspecting AssistedInject Bindings _(new in Guice 3.0)_

Visiting an assisted inject binding is useful for tests or debugging.  AssistedInject uses the [[extensions SPI|ExtensionSPI]] to let you learn more about the factory binding.  You can visit bindings with an [AssistedInjectTargetVisitor](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/assistedinject/AssistedInjectTargetVisitor.html) to see details about the binding.

```java  
  Binding<PaymentFactory> binding = injector.getBinding(PaymentFactory.class);
  binding.acceptTargetVisitor(new Visitor());

  class Visitor
      extends DefaultBindingTargetVisitor<Object, Void>
      implements AssistedInjectTargetVisitor<Object, Void> {

    @Override void visit(AssistedInjectBinding<?> binding) {
      // Loop over each method in the factory...
      for(AssistedMethod method : binding.getAssistedMethods()) {
        System.out.println("Non-assisted Dependencies: " + method.getDependencies()
                       + ", Factory Method: " + method.getFactoryMethod()
                       + ", Implementation Constructor: " + method.getImplementationConstructor()
                       + ", Implementation Type: " + method.getImplementationType());
      }
    }
  }
```
# Multibindings.md

_Overview of Multibinder and MapBinder extensions_
### Multibindings
[Multibinder](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/multibindings/Multibinder.html) and [MapBinder](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/multibindings/MapBinder.html) are intended for plugin-type architectures, where you've got several modules contributing Servlets, Actions, Filters, Components or even just names. 

#### Using Multibindings to host Plugins
Multibindings make it easy to support plugins in your application. Made popular by [IDEs](http://www.eclipseplugincentral.com) and [browsers](https://addons.mozilla.org/en-US/firefox/), this pattern exposes APIs for extending the behaviour of an application.

Neither the plugin consumer nor the plugin author need write much setup code for extensible applications with Guice. Simply define an interface, bind implementations, and inject sets of implementations! Any module can create a new Multibinder to contribute bindings to a set of implementations.  To illustrate, we'll use plugins to summarize ugly URIs like `http://bit.ly/1mzgW1` into something readable on Twitter.

First, we define an interface that plugin authors can implement. This is usually an interface that lends itself to several implementations. For this example, we would write a different implementation for each website that we could summarize.
```java
interface UriSummarizer {
  /** 
   * Returns a short summary of the URI, or null if this summarizer doesn't
   * know how to summarize the URI.
   */
  String summarize(URI uri);
}
```

Next, we'll get our plugin authors to implement the interface. Here's an implementation that shortens Flickr photo URLs:
```java
class FlickrPhotoSummarizer implements UriSummarizer {
  private static final Pattern PHOTO_PATTERN
      = Pattern.compile("http://www\.flickr\.com/photos/[^/]+/(\d+)/");

  public String summarize(URI uri) {
    Matcher matcher = PHOTO_PATTERN.matcher(uri.toString());
    if (!matcher.matches()) {
      return null;
    } else {
      String id = matcher.group(1);
      Photo photo = lookupPhoto(id);
      return photo.getTitle();
    }
  }
}
```

The plugin author registers their implementation using a multibinder. Some plugins may bind multiple implementations, or implementations of several extension-point interfaces.
```java
public class FlickrPluginModule extends AbstractModule {
  public void configure() {
    Multibinder<UriSummarizer> uriBinder = Multibinder.newSetBinder(binder(), UriSummarizer.class);
    uriBinder.addBinding().to(FlickrPhotoSummarizer.class);

    ... // bind plugin dependencies, such as our Flickr API key
  }
}
```

Now we can consume the services exposed by our plugins. In this case, we're summarizing tweets:
```java
public class TweetPrettifier {

  private final Set<UriSummarizer> summarizers;
  private final EmoticonImagifier emoticonImagifier;

  @Inject TweetPrettifier(Set<UriSummarizer> summarizers,
      EmoticonImagifier emoticonImagifier) {
    this.summarizers = summarizers;
    this.emoticonImagifier = emoticonImagifier;
  }

  public Html prettifyTweet(String tweetMessage) {
    ... // split out the URIs and call prettifyUri() for each
  }

  public String prettifyUri(URI uri) {
    // loop through the implementations, looking for one that supports this URI
    for (UrlSummarizer summarizer : summarizers) {
      String summary = summarizer.summarize(uri);
      if (summary != null) {
        return summary;
      }
    }

    // no summarizer found, just return the URI itself
    return uri.toString();
  }
}
```

_**Note:** The method `Multibinder.newSetBinder(binder, type)` can be confusing.  This operation creates a new binder, but doesn't override any existing bindings.  A binder created this way contributes to the existing Set of implementations for that type.  It would create a new set only if one is not already bound._


Finally we must register the plugins themselves. The simplest mechanism to do so is to list them programatically:
```java
public class PrettyTweets {
  public static void main(String[] args) {
    Injector injector = Guice.createInjector(
        new GoogleMapsPluginModule(),
        new BitlyPluginModule(),
        new FlickrPluginModule()
        ...
    );

    injector.getInstance(Frontend.class).start();
  }
}
```
If it is infeasible to recompile each time the plugin set changes, the list of plugin modules can be loaded from a configuration file.

Note that this mechanism cannot load or unload plugins while the system is running. If you need to hot-swap application components, investigate [[Guice's OSGi|OSGi]].

#### Limitations
When you use PrivateModules with multibindings, all of the elements must be bound in the same environment. You cannot create collections whose elements span private modules. Otherwise injector creation will fail.

#### Inspecting Multibindings or MapBindings _(new in Guice 3.0)_
Sometimes you need to inspect the elements that make up a Multibinder or MapBinder.  For example, you may need a test that strips all elements of a MapBinder out of a series of modules.  You can visit a binding with a [MultibindingTargetVisitor](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/multibindings/MultibindingsTargetVisitor.html) to get details about Multibindings or MapBindings.  After you have an instance of a [MapBinderBinding](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/multibindings/MapBinderBinding.html) or a [MultibinderBinding](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/multibindings/MultibinderBinding.html) you can learn more.

```java
   // Find the MapBinderBinding and use it to remove elements within it.
   Module stripMapBindings(Key<?> mapKey, Module... modules) {
     MapBinderBinding<?> mapBinder = findMapBinder(mapKey, modules);
     List<Element> allElements = Lists.newArrayList(Elements.getElements(modules));
     if (mapBinder != null) {
       List<Element> mapElements = getMapElements(mapBinder, modules);
       allElements.removeAll(mapElements);
     }
     return Elements.getModule(allElements);
  }

  // Look through all Elements in the module and, if the key matches,
  // then use our custom MultibindingsTargetVisitor to get the MapBinderBinding
  // for the matching binding.
  MapBinderBinding<?> findMapBinder(Key<?> mapKey, Module... modules) {
    for(Element element : Elements.getElements(modules)) {
      MapBinderBinding<?> binding =
          element.acceptVisitor(new DefaultElementVisitor<MapBinderBinding<?>>() {
            MapBinderBinding<?> visit(Binding<?> binding) {
              if(binding.getKey().equals(mapKey)) {
                return binding.acceptTargetVisitor(new Visitor());
              }
              return null;
            }
          });
      if (binding != null) {
        return binding;
      }
    }
    return null;
  }

  // Get all elements in the module that are within the MapBinderBinding.
  List<Element> getMapElements(MapBinderBinding<?> binding, Module... modules) {
    List<Element> elements = Lists.newArrayList();
    for(Element element : Elements.getElements(modules)) {
      if(binding.containsElement(elements)) {
        elements.add(element);
      }
    }
    return elements;
  }

  // A visitor that just returns the MapBinderBinding for the binding.
  class Visitor
      extends DefaultBindingTargetVisitor<Object, MapBinderBinding<?>>
      implements MultibindingsTargetVisitor<Object, MapBinderBinding<?>> {
    MapBinderBinding<?> visit(MapBinderBinding<?> mapBinder) {
      return mapBinder;
    }

    MapBinderBinding<?> visit(MultibinderBinding<?> multibinder) {
      return null;
    }
  }
```

# CustomScopes.md

### Custom Scopes
It is generally recommended that users **do not** write their own custom scopes — the built-in scopes should be sufficient for most applications. If you're writing a web application, the `ServletModule` provides simple, well tested scope implementations for HTTP requests and HTTP sessions. 

Creating custom scopes is a multistep process:
  1. Define a scoping annotation
  2. Implementing the `Scope` interface
  3. Attaching the scope annotation to the implementation
  4. Triggering scope entry and exit

#### Defining a scoping annotation
The scoping annotation identifies your scope. You'll use it to annotate Guice-constructed types, `@Provides` methods, and in the `in()` clause of a bind statement. Copy-and-customize this code to define your scoping annotation:
```java
import static java.lang.annotation.ElementType.METHOD;
import static java.lang.annotation.ElementType.TYPE;
import java.lang.annotation.Retention;
import static java.lang.annotation.RetentionPolicy.RUNTIME;
import java.lang.annotation.Target;

@Target({ TYPE, METHOD }) @Retention(RUNTIME) @ScopeAnnotation
public @interface BatchScoped {}
```
**Tip:** If your scope represents a request or session (such as for SOAP requests), consider using the `RequestScoped` and `SessionScoped` annotations from Guice's servlet extension. Otherwise, you may import the wrong annotation by mistake. That problem can be quite frustrating to debug.

#### Implementing Scope
The scope interface ensures there's at most one type instance for each scope instance. `SimpleScope` is a decent starting point for a per-thread implementation. Copy this class into your project and tweak it to suit your needs.
```java
import static com.google.common.base.Preconditions.checkState;
import com.google.common.collect.Maps;
import com.google.inject.Key;
import com.google.inject.OutOfScopeException;
import com.google.inject.Provider;
import com.google.inject.Scope;
import java.util.Map;

/**
 * Scopes a single execution of a block of code. Apply this scope with a
 * try/finally block: <pre><code>
 *
 *   scope.enter();
 *   try {
 *     // explicitly seed some seed objects...
 *     scope.seed(Key.get(SomeObject.class), someObject);
 *     // create and access scoped objects
 *   } finally {
 *     scope.exit();
 *   }
 * </code></pre>
 *
 * The scope can be initialized with one or more seed values by calling
 * <code>seed(key, value)</code> before the injector will be called upon to
 * provide for this key. A typical use is for a servlet filter to enter/exit the
 * scope, representing a Request Scope, and seed HttpServletRequest and
 * HttpServletResponse.  For each key inserted with seed(), you must include a
 * corresponding binding:
 *  <pre><code>
 *   bind(key)
 *       .toProvider(SimpleScope.&lt;KeyClass&gt;seededKeyProvider())
 *       .in(ScopeAnnotation.class);
 * </code></pre>
 *
 * @author Jesse Wilson
 * @author Fedor Karpelevitch
 */
public class SimpleScope implements Scope {

  private static final Provider<Object> SEEDED_KEY_PROVIDER =
      new Provider<Object>() {
        public Object get() {
          throw new IllegalStateException("If you got here then it means that" +
              " your code asked for scoped object which should have been" +
              " explicitly seeded in this scope by calling" +
              " SimpleScope.seed(), but was not.");
        }
      };
  private final ThreadLocal<Map<Key<?>, Object>> values
      = new ThreadLocal<Map<Key<?>, Object>>();

  public void enter() {
    checkState(values.get() == null, "A scoping block is already in progress");
    values.set(Maps.<Key<?>, Object>newHashMap());
  }

  public void exit() {
    checkState(values.get() != null, "No scoping block in progress");
    values.remove();
  }

  public <T> void seed(Key<T> key, T value) {
    Map<Key<?>, Object> scopedObjects = getScopedObjectMap(key);
    checkState(!scopedObjects.containsKey(key), "A value for the key %s was " +
        "already seeded in this scope. Old value: %s New value: %s", key,
        scopedObjects.get(key), value);
    scopedObjects.put(key, value);
  }

  public <T> void seed(Class<T> clazz, T value) {
    seed(Key.get(clazz), value);
  }

  public <T> Provider<T> scope(final Key<T> key, final Provider<T> unscoped) {
    return new Provider<T>() {
      public T get() {
        Map<Key<?>, Object> scopedObjects = getScopedObjectMap(key);

        @SuppressWarnings("unchecked")
        T current = (T) scopedObjects.get(key);
        if (current == null && !scopedObjects.containsKey(key)) {
          current = unscoped.get();

          // don't remember proxies; these exist only to serve circular dependencies
          if (Scopes.isCircularProxy(current)) {
            return current;
          }

          scopedObjects.put(key, current);
        }
        return current;
      }
    };
  }

  private <T> Map<Key<?>, Object> getScopedObjectMap(Key<T> key) {
    Map<Key<?>, Object> scopedObjects = values.get();
    if (scopedObjects == null) {
      throw new OutOfScopeException("Cannot access " + key
          + " outside of a scoping block");
    }
    return scopedObjects;
  }

  /**
   * Returns a provider that always throws exception complaining that the object
   * in question must be seeded before it can be injected.
   *
   * @return typed provider
   */
  @SuppressWarnings({"unchecked"})
  public static <T> Provider<T> seededKeyProvider() {
    return (Provider<T>) SEEDED_KEY_PROVIDER;
  }
}
```

#### Binding the annotation to the implementation
You must attach your scoping annotation to the corresponding scope implementation. Just like bindings, you can configure this in your module's `configure()` method. Usually you'll also bind the scope itself, so interceptors or filters can use it.
```java
public class BatchScopeModule {
  public void configure() {
    SimpleScope batchScope = new SimpleScope();

    // tell Guice about the scope
    bindScope(BatchScoped.class, batchScope);

    // make our scope instance injectable
    bind(SimpleScope.class)
        .annotatedWith(Names.named("batchScope"))
        .toInstance(batchScope);
  }
}
```

#### Triggering the Scope
`SimpleScope` requires that you manually enter and exit the scope. Usually this lives in some low-level infrastructure code, like a filter or interceptor. Be sure to call `exit()` in a `finally` clause, otherwise the scope will be left open when an exception is thrown.
```java
  @Inject @Named("batchScope") SimpleScope scope;

  /**
   * Runs {@code runnable} in batch scope.
   */
  public void scopeRunnable(Runnable runnable) {
    scope.enter();
    try {
      // explicitly seed some seed objects...
      scope.seed(Key.get(SomeObject.class), someObject);

      // create and access scoped objects
      runnable.run();

    } finally {
      scope.exit();
    }
  }
```
# ThrowingProviders.md

_throwing providers extension tutorial_

### Throwing Providers

Guice isn't very good at handling exceptions that occur during provision.
  * Implementers of Provider can only throw RuntimeExceptions.
  * Callers of Provider can't catch the exception they threw, because it may be wrapped in a ProvisionException.
  * Injecting an instance directly rather than a Provider can cause creation of the injected object to fail.
  * Exceptions cannot be advertised in the API.

The ThrowingProviders extension offers an alternative to Providers that allow a checked exception to be thrown.
  * API consistent with Provider.
  * Scopable
  * Standard binding DSL

#### ThrowingProvider _(new in Guice 2.0)_
Guice 2.0 offers the ThrowingProvider interface.  It is similar to Provider, but with a generic Exception type:
```java
public interface ThrowingProvider<T,E extends Exception> {
  T get() throws E;
}
```
For each application exception, create an interface that extends the ThrowingProvider. For our news widget application, we created the FeedProvider interface that throws a FeedUnavailableException:
```java
public interface FeedProvider<T> extends ThrowingProvider<T, FeedUnavailableException> { }
```

#### CheckedProvider _(new in Guice 3.0)_
Guice 3.0 offers the CheckedProvider interface.  It improves upon ThrowingProvider by allowing more than one exception type to be thrown:
```java
public interface CheckedProvider<T> {
  T get() throws Exception;
}
```
For each object where providing may throw an exception, create an interface that extends the CheckedProvider. For our news widget application, we created the FeedProvider interface that throws a FeedUnavailableException and SecurityException:
```java
public interface FeedProvider<T> extends CheckedProvider<T> {
  T get() throws FeedUnavailableException, SecurityException;
}

```

#### Binding the provider
After implementing our WorldNewsFeedProvider and SportsFeedProvider, we bind them in our module using ThrowingProviderBinder:
```java
public static class FeedModule extends AbstractModule {
  protected void configure() {
    ThrowingProviderBinder.create(binder())
        .bind(FeedProvider.class, BbcFeed.class)
        .annotatedWith(WorldNews.class)
        .to(WorldNewsFeedProvider.class)
        .in(HourlyScoped.class);

    ThrowingProviderBinder.create(binder())
        .bind(FeedProvider.class, BbcFeed.class)
        .annotatedWith(Sports.class)
        .to(SportsFeedProvider.class)
        .in(QuarterHourlyScoped.class);
  }
}
```

#### Using @CheckedProvides _(new in Guice 3.0)_
You can also bind checked providers use an @Provides-like syntax: @CheckedProvides.  This has all the benefits of [ProvidesMethods @Provides methods], and also allows you to specify exceptions.
```java
public static class FeedModule extends AbstractModule {
  protected void configure() {
    // create & install a module that uses the @CheckedProvides methods
    install(ThrowingProviderBinder.forModule(this)); 
  }

  @CheckedProvides(FeedProvider.class) // define what interface will provide it
  @HourlyScoped // scoping annotation
  @WorldNews // binding annotation
  BbcFeed provideWorld(FeedFactory factory)
      throws FeedUnavailableException, SecurityException {
    return factory.tryToFeed("bbc"); // may throw an exception
  }

  @CheckedProvides(FeedProvider.class) // define what interface will provide it
  @QuarterlyHourlyScoped // scoping annotation
  @Sports // binding annotation
  BbcFeed provideSports(FeedFactory factory)
      throws FeedUnavailableException, SecurityException {
    return factory.tryToFeed("bbc"); // may throw an exception
  }
```
  
#### Injecting the provider
Finally, we can inject the FeedProviders throughout our application code. Whenever we call get(), the compiler reminds us to handle the FeedUnavailableException and any other exceptions declared in the interface:
```java
public class BbcNewsWidget {
  private final FeedProvider<BbcFeed> worldNewsFeedProvider;
  private final FeedProvider<BbcFeed> sportsFeedProvider;

  @Inject
  public BbcNewsWidget(
      @WorldNews FeedProvider<BbcFeed> worldNewsFeedProvider,
      @Sports FeedProvider<BbcFeed> sportsFeedProvider) {
    this.worldNewsFeedProvider = worldNewsFeedProvider;
    this.sportsFeedProvider = sportsFeedProvider;
  }

  public GxpClosure render() {
    try {
      BbcFeed bbcWorldNews = worldNewsFeedProvider.get();
      BbcFeed bbcSports = sportsFeedProvider.get();
      return NewsWidgetBody.getGxpClosure(bbcWorldNews, bbcSports);
    } catch (FeedUnavailableException e) {
      return UnavailableWidgetBody.getGxpClosure();
    }
  }
}
```
#### Notes on Scoping
Scopes work the same way they do with Providers. Each time get() is called, the returned object will be scoped appropriately. Exceptions are also scoped. For example, when worldNewsFeedProvider.get() throws an exception, the same exception instance will be thrown for all callers within the scope.
# Grapher.md

_Using grapher to visualize Guice applications_
### Grapher
When you've written a sophisticated application, Guice's rich introspection API can describe the object graph in detail. The built-in grapher extension exposes this data as an easily understandable visualization. It can show the bindings and dependencies from several classes in a complex application in a unified diagram.

#### Generating a .dot file
Guice's grapher leans heavily on [GraphViz](http://www.graphviz.org/), an open source graph visualization package. It cleanly separates graph specification from visualization and layout. To produce a graph `.dot` file for an `Injector`, you can use the following code:

#### For Guice 3.0 or earlier
```java
import com.google.inject.Injector;
import com.google.inject.grapher.GrapherModule;
import com.google.inject.grapher.InjectorGrapher;
import com.google.inject.grapher.graphviz.GraphvizModule;
import com.google.inject.grapher.graphviz.GraphvizRenderer;

public class Grapher {
  private void graph(String filename, Injector demoInjector) throws IOException {
    PrintWriter out = new PrintWriter(new File(filename), "UTF-8");

    Injector injector = Guice.createInjector(new GrapherModule(), new GraphvizModule());
    GraphvizRenderer renderer = injector.getInstance(GraphvizRenderer.class);
    renderer.setOut(out).setRankdir("TB");

    injector.getInstance(InjectorGrapher.class)
        .of(demoInjector)
        .graph();
  }
}
```

#### For future releases
```java
import com.google.inject.Injector;
import com.google.inject.grapher.graphviz.GraphvizGrapher;
import com.google.inject.grapher.graphviz.GraphvizModule;

public class Grapher {
  private void graph(String filename, Injector demoInjector) throws IOException {
    PrintWriter out = new PrintWriter(new File(filename), "UTF-8");

    Injector injector = Guice.createInjector(new GraphvizModule());
    GraphvizGrapher grapher = injector.getInstance(GraphvizGrapher.class);
    grapher.setOut(out);
    grapher.setRankdir("TB");
    grapher.graph(demoInjector);
  }
}
```

#### The .dot file
Executing the code above produces a `.dot` file that specifies a graph. Each entry in the file represents either a node or an edge in the graph. Here's a sample `.dot` file:
```dot
digraph injector {
  graph [rankdir=TB];
  k_997fdab [shape=box, label=<<table cellspacing="0" cellpadding="5" cellborder="0" border="0"><tr><td 
      align="left" port="header" bgcolor="#ffffff"><font align="left" color="#000000" point-size="10">@Nuclear<br
      align="left"/></font><font color="#000000">EnergySource<br align="left"/></font></td></tr></table>>,
      style=dashed, margin=0.02,0]
  k_119e4fd8 [shape=box, label=<<table cellspacing="0" cellpadding="5" cellborder="0" border="0"><tr><td
      align="left" port="header" bgcolor="#ffffff"><font align="left" color="#000000" point-size="10">@Driver<br 
      align="left"/></font><font color="#000000">Person<br align="left"/></font></td></tr></table>>, 
      style=dashed, margin=0.02,0]
  k_115c9d69 [shape=box, label=<<table cellspacing="0" cellpadding="5" cellborder="0" border="0"><tr><td 
      align="left" port="header" bgcolor="#ffffff"><font color="#000000">FluxCapacitor<br 
      align="left"/></font></td></tr></table>>, style=dashed, margin=0.02,0]
  i_115c9d69 [shape=box, label=<<table cellspacing="0" cellpadding="5" cellborder="1" border="0"><tr><td 
      align="left" port="header" bgcolor="#aaaaaa"><font align="left" color="#ffffff" point-
      size="10">BackToTheFutureModule.java:42<br align="left"/></font><font 
      color="#ffffff">#provideFluxCapacitor(EnergySource)<br align="left"/></font></td></tr></table>>, style=invis, 
      margin=0.02,0]
  k_1c5031a6 [shape=box, label=<<table cellspacing="0" cellpadding="5" cellborder="1" border="0"><tr><td 
      align="left" port="header" bgcolor="#000000"><font color="#ffffff">Plutonium<br 
      align="left"/></font></td></tr><tr><td align="left" port="m_9c5dfb84">&lt;init&gt;</td></tr></table>>, 
      style=invis, margin=0.02,0]
  k_17b87c3a [shape=box, label=<<table cellspacing="0" cellpadding="5" cellborder="1" border="0"><tr><td 
      align="left" port="header" bgcolor="#000000"><font color="#ffffff">MartyMcFly<br 
      align="left"/></font></td></tr><tr><td align="left" port="m_8b2cda3d">&lt;init&gt;</td></tr></table>>, 
      style=invis, margin=0.02,0]
  k_9acc501 [shape=box, label=<<table cellspacing="0" cellpadding="5" cellborder="0" border="0"><tr><td 
      align="left" port="header" bgcolor="#ffffff"><font color="#000000">EnergySource<br 
      align="left"/></font></td></tr></table>>, style=dashed, margin=0.02,0]
  k_dd456f9 [shape=box, label=<<table cellspacing="0" cellpadding="5" cellborder="1" border="0"><tr><td 
      align="left" port="header" bgcolor="#000000"><font color="#ffffff">EnergySourceProvider<br 
      align="left"/></font></td></tr><tr><td align="left" port="m_f4b5f9f7">&lt;init&gt;</td></tr></table>>, 
      style=invis, margin=0.02,0]
  k_997fdab -> k_1c5031a6 [arrowtail=none, style=dashed, arrowhead=onormal]
  k_119e4fd8 -> k_17b87c3a [arrowtail=none, style=dashed, arrowhead=onormal]
  k_115c9d69 -> i_115c9d69 [arrowtail=none, style=dashed, arrowhead=onormalonormal]
  i_115c9d69:header:e -> k_9acc501 [arrowtail=none, style=solid, arrowhead=normal]
  k_9acc501 -> k_dd456f9 [arrowtail=none, style=dashed, arrowhead=onormalonormal]
}
```

#### Rendering the .dot file
Download a [Graphviz viewer](http://www.graphviz.org/) for your platform, and use it to render the `.dot` file. The rendered graph might take a few minutes to render. Exporting the rendered graph as a PDF or image makes it easier to share.

[<img src="http://google.github.io/guice/user-docs/Grapher_screenshot.png" width="300px" height="300px" />](http://google.github.io/guice/user-docs/Grapher_screenshot.png)

On Linux, you can use the command-line `dot` tool to convert `.dot` files into images.
```shell
  dot -T png my_injector.dot > my_injector.png
```
The tool currently has issues with our font tags. We're still working that out.

#### Graph display
Edges:
   * **Solid edges** represent dependencies from implementations to the types they depend on.
   * **Dashed edges** represent bindings from types to their implementations.
   * **Double arrows** indicate that the binding or dependency is to a `Provider`.

Nodes:
   * Implementation types are given *black backgrounds*.
   * Implementation instances have *gray backgrounds*.

Other behavior:
   * If an injection point requests a `Provider<Foo>`, the grapher will elide the `Provider` and just show a dependency to `Foo`.
   * The grapher will use an instance's `toString()` method to render its title if it has overridden `Object`'s default implementation.
# BoundFields.md

_Automatically bind field's values to their types._

### Bound Fields (New in Guice 4.0)

Test code which uses Guice frequently follows the pattern whereby you create several fields in your test class and bind those fields to their types using a custom Module. `BoundFieldModule` can help simplify this pattern by automatically binding fields annotated with `@Bind`.

#### Binding Fields Manually
Frequently in order to test classes which are injected, you need to inject mocks or dummy values for the tested class's dependencies. In order to do this, a common pattern is the following:
```java
public class TestFoo {
  private Bar barMock;

  // Foo depends on Bar.
  @Inject private Foo foo;

  @Before public void setUp() {
    barMock = ...;
    Guice.createInjector(getTestModule()).injectMembers(this);
  }

  private Module getTestModule() {
    return new AbstractModule() {
      @Override protected void configure() {
        bind(Bar.class).toInstance(barMock);
      }
    };
  }

  @Test public void testBehavior() {
    ...
  }
}
```
This class creates a field for the tested class's dependencies, places in that field a mock and binds the field into Guice with a custom Module.

#### Binding Fields Using `BoundFieldModule`
`BoundFieldModule` scans a given object for fields annotated with `@Bind` and binds those fields automatically. Using this, we can simplify our `TestFoo` class to:
```java
public class TestFoo {
  // bind(Bar.class).toInstance(barMock);
  @Bind private Bar barMock;

  // Foo depends on Bar.
  @Inject private Foo foo;

  @Before public void setUp() {
    barMock = ...;
    Guice.createInjector(BoundFieldModule.of(this)).injectMembers(this);
  }

  @Test public void testBehavior() {
    ...
  }
}
```

This binding occurrs under the following rules:

 * For each `@Bind` annotated field of an object and its superclasses, this module will bind that field's type to that field's value at injector creation time. This includes both instance and static fields.
 * If `@Bind(to = ...)` is specified, the field's value will be bound to the class specified by `to` instead of the field's actual type.
 * If a [BindingAnnotation](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/BindingAnnotation.html) or [Qualifier](http://atinject.googlecode.com/svn/trunk/javadoc/javax/inject/Qualifier.html) is present on the field, that field will be bound using that annotation via [annotatedWith](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/binder/AnnotatedBindingBuilder.html). For example, `bind(Foo.class).annotatedWith(BarAnnotation.class).toInstance(theValue)`. It is an error to supply more than one [BindingAnnotation](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/BindingAnnotation.html) or [Qualifier](http://atinject.googlecode.com/svn/trunk/javadoc/javax/inject/Qualifier.html).
 * If the field is of type [Provider](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/Provider.html), the field's value will be bound as a provider using [toProvider](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/binder/LinkedBindingBuilder.html) to the provider's parameterized type. For example, `Provider<Integer>` binds to `Integer`. Attempting to bind a non-parameterized `Provider` without a `@Bind(to = ...)` clause is an error.

Additional information can be found in the [source](https://github.com/google/guice/blob/master/extensions/testlib/src/com/google/inject/testing/fieldbinder/BoundFieldModule.java). Javadoc link coming soon!
# Bootstrap.md

_Overview of the Injector creation process_
### The Injector Creation Process
Guice builds an injector using configuration modules. If there are errors at the end of any phase, injector creation is halted and a `CreationException` is thrown. 

Phase 1: Static Building
In this phase, Guice interprets elements, creates bindings, and validates the configuration. The only user code executed in this phase is `Module.configure()`.

This is the only phase executed for `Stage.TOOL`.

#### Phase 2: Injection
During this phase, objects will be injected on-demand if necessary. For example, if satisfying a static injection requires a provider instance, the provider will be injected before it is used. If initialized objects are circularly dependent, the order of injection is undefined.

First, **statics** registered via `requestStaticInjection()` are injected. Next, **instances** that are the arguments to `requestInjection()`, `toInstance()` and `toProvider()` are injected.

#### Phase 3: Singleton Preloading
In `Stage.PRODUCTION`, all singletons are created. In `Stage.DEVELOPMENT`, only bindings scoped using `asEagerSingleton()` are created.
# BindingResolution.md

_How the Injector resolves injection requests_
### Binding Resolution
The injector's process of resolving an injection request depends on the bindings and the annotations of the types involved.  Here's how an injection request is resolved:
  1. **Use explicit bindings.**
    * If the binding links to another, follow this resolution algorithm for that.
    * If the binding specifies an instance, return that.
    * If the binding specifies a provider, use that.
  2. **Ask a parent injector.** If this injector has a parent injector, ask that to resolve the binding. If it succeeds, use that. Otherwise proceed.
  3. **Ask child injectors.** If any child injector already has this binding, give up. A blacklist of bindings from child injectors is kept so injectors don't need to maintain references to their child injectors.
  4. **Handle Provider injections.** If the type is `Provider<T>`, resolve `T` instead, using the same binding annotation, if it exists. 
  5. **Convert constants.** If there is a constant string bound with the same annotation, and a `TypeConverter` that supports this type, use the converted String.
  6. **If the dependency has a binding annotation, give up.** Guice will not create default bindings for annotated dependencies.
  7. **If the dependency is an array or enum, give up.**
  8. **Handle TypeLiteral injections**. If the type is `TypeLiteral<T>`, inject that value, using context for the value of the type parameter.
  9. **Use resolution annotations.** If the dependency's type has `@ImplementedBy` or `@ProvidedBy`, lookup the binding for the referenced type and use that.
  10. **If the dependency is abstract or a non-static inner class, give up.**
  11. **Use a single `@Inject` or public no-arguments constructor.**
    1. Validate bindings for all dependencies — the constructor's parameters, plus `@Inject` methods and fields of the type and all supertypes.
    2. Invoke the constructor.
    3. Inject all fields. Supertype fields are injected before subtype fields.
    4. Inject all methods. Supertype methods are injected before subtype methods.
# InjectionPoints.md

_How Guice decides what gets injected_
### Injection Points

#### Constructor Selection
When Guice instantiates a type using its constructor, it decides which constructor to invoke by following these rules:
  1. Find and return an `@Inject`-annotated constructor
    * If any are `@Inject(optional=true)` constructors, record an error. Constructors may not be optional.
    * If there are multiple `@Inject`-annotated constructors, record an error. There may be at most one `@Inject`-annotated constructor.
    * If this constructor has binding annotations (like `@Named`), record an error. Binding annotations on the constructor's _parameters_ are okay, but not on the constructor itself.
  2. Find and return a no-arguments constructor
    * If this constructor is `private` but the class is not `private`, record an error. Private constructors on non-private classes are not injectable without the `@Inject` annotation.
    * If this constructor itself has binding annotations (like `@Named`), record an error. Binding annotations on the constructor's _parameters_ are okay, but not on the constructor itself.
  3. No suitable constructor was found. Record an error.

#### Injecting Fields and Methods
Injections are performed in a specific order. All fields are injected and then all methods. Within the fields, supertype fields are injected before subtype fields. Similarly, supertype methods are injected before subtype methods.

#### What Gets Injected
Guice injects **instance methods** and fields of:
  * All values that are bound using `toInstance()`. These are injected at injector-creation time.
  * All providers that are bound using `toProvider(Provider)`. These are injected at injector-creation time.
  * All instances that are registered for injection using `requestInjection`. These are injected at injector-creation time.
  * The arguments to `Injector.injectMembers` when that method is invoked.
  * All values instantiated by Guice via its injectable constructor, immediately after it is instantiated. Regardless of how the value is scoped, it is only injected once.

Guice injects **static methods** and fields of:
  * All classes that are registered for static injection using `requestStaticInjection`.

#### How Guice Injects
**To inject a field**, Guice will:
  * Lookup the binding, creating a just-in-time binding if necessary. If no just-in-time binding can be created, an error is reported. 
  * The binding is exercised to provide a value. How this works depends on the type of the binding.
  * The binding's value is set into the field. Injecting `final` fields is not recommended because the injected value may not be visible to other threads.
If the field injection is `@Inject(optional=true)` and the binding neither exists, nor can be [[created|JustInTimeBindings]], the field injection is skipped. 

**To inject a method**, Guice will:
  * Lookup bindings for all of the parameters, creating just-in-time bindings if necessary. If any just-in-time binding cannot be created, an error is reported. 
  * The bindings are each exercised to provide a value. How this works depends on the types of the bindings.
  * The method is called, using the bindings' values as arguments.
If the method injection is optional and a parameter's binding neither exists nor can be [[created|JustInTimeBindings]], the method injection will be skipped. 
# ClassLoading.md

_Motivation for and consequences of generating classes in Guice_
### Class Loading
Guice performs code generation and class loading. We use this stuff for faster reflection, method interceptors, and to proxy circular dependencies. 

When loading classes, we need to be careful of:
  * **Memory leaks.** Generated classes need to be garbage collected in long-lived applications. Once an injector and any instances it created can be garbage collected, the corresponding generated classes should be collectable.
  * **Visibility.** Containers like `OSGi` use class loader boundaries to enforce modularity at runtime.

For each generated class, there's multiple class loaders involved:
  * **The target class's class loader.** Every generated class services exactly one user-supplied class. This class loader must be used to access members with private and package visibility.
  * **Guice's class loader.**
  * **Our bridge class loader.** This is a child of the target class's loader. It selectively delegates to either the target class's loader (for user classes) or the Guice class loader (for internal classes that are used by the generated classes). This class loader that owns the classes generated by Guice.


#### The Bridge ClassLoader
Guice uses its bridge class loader unless:
  * A package-private method or constructor is being intercepted
  * A method in a package-private class is being intercepted
  * Any parameters or return types of the method or constructor being intercepted are package-private
  * The target class was loaded by the system class loader
  * It is disabled

Only classes loaded by the bridge class loader will work in an OSGi environment.


#### Disabling Guice's Bridge ClassLoader
Setting the system property `guice.custom.loader` to `false` will cause Guice to use the target class's classloader for all generated classes. Guice's bridge classloader will not be used.


#### Generated Classes and Memory Leaks
Classes can only be garbage collected when all of their instances and the originating classloader are no longer referenced. The bridge class loader is intended to allow generated classes to be garbage collected as quickly as possible. 

To minimize the amount of PermGen memory used by Guice-generated classes, avoid intercepting package-private methods.
# OptionalAOP.md

_Building Guice without AOP support_
### Optional AOP
Guice 1.0 was available in a single version that included AOP.

In Guice 2.0 and later, AOP is optional. If your platform doesn't support bytecode generation, you can download a version of Guice that doesn't include AOP support. This is most useful for mobile platforms like [Android](http://code.google.com/android/). This version also lacks **fast reflection** and **line numbers in error messages**. For this reason, we recommend Guice+AOP even in applications that don't use method interceptors.

#### Feature Comparison

|   | *Guice 1.0* | *Guice 2.0 no AOP* | *Guice 2.0* | *[GIN](http://code.google.com/p/google-gin)* |
|---|-------------|--------------------|-------------|----------------------------------------------|
| Library size | 544KB | 430KB | 652KB | 0KB (code gen) |
| High performance | ✔ | ✔ | ✔ | ✔ |
| Binding EDSL | ✔ | ✔ | ✔ | ✔ |
| Scopes | ✔ | ✔ | ✔ | ✔ |
| Typesafe | ✔ | ✔ | ✔ | ✔ |
| Fast reflection | ✔ ([cglib](http://cglib.sourceforge.net/)) | | ✔ ([cglib](http://cglib.sourceforge.net/)) | ✔ (code gen) |
| Line numbers in error messages  | ✔ |  | ✔ |  |
| Method interceptors  | ✔ | | ✔ ||  |
| Provider Methods |  | ✔ | ✔ | ✔ |
| Binding overrides |  | ✔ | ✔ |  |
| Tool-friendly SPI |  | ✔ | ✔ |  |
| Child Injectors |  | ✔ | ✔ |  |
| Servlet Support | Scopes Only | ✔ | ✔ |  |
| Error Reporting | Good | Better | Best | Best |

#### Building Guice without AOP
Guice has an ant task that creates a modified copy of the Guice source-tree. The copy uses the [munge](http://publicobject.com/2009/02/preprocessing-java-with-munge.html) preprocessor to remove all bytecode-dependent APIs. Use the following commands to build Guice without AOP support:
```bash
ant no_aop
cd build/no_aop/
ant dist 
```

Building with Maven automatically generates a no_aop artifact.
# SpringComparison.md

_How does Guice compare to Spring et al_
### Spring Comparison

The Spring Framework, created by Rod Johnson, blazed the Java dependency injection trail. If not the first dependency injection framework, you can certainly credit Spring with pioneering much of what we know about dependency injection and bringing it into the mainstream. Guice might not exist (at least not this early) if not for Spring's example.

Guice and Spring don't compete directly. While Spring is a comprehensive stack, Guice focuses on dependency injection. Even so, the first question most people ask is, "how does Guice compare to Spring?" Rather than repeat the same spiel N times, we figured it best to answer the inevitable question once.

Let us start by saying, the Guice team has a great deal of respect for the Spring developers, community, and their overall attitude. We're more than willing to work together in any capacity. We even gave a few core Spring developers a sneak peek at Guice six months before the 1.0 release, and they've had access to the source code repository ever since.

Without further ado, how does Guice compare to Spring?

Guice is anything but a rehash of Spring. Guice was born purely out of use cases from one of Google's biggest (in every way) applications: [AdWords](http://adwords.google.com). We sat down and asked ourselves, how do we really want to build this application going forward? Guice enables our answer. 

Guice wholly embraces annotations and generics, thereby enabling you to wire together and test objects with measurably less effort. Guice proves you can use annotations instead of error-prone, refactoring-adverse string identifiers. 

Some new users fear that the number of annotations will get out of hand. Fear not. You won't have nearly as many different annotations in a Guice application as you have string identifiers in a Spring application.
  1. You only need to use annotations when a binding type alone isn't sufficient.
  2. You can reuse the same annotation type across multiple bindings (`@Transactional` for example). 
  3. Annotations can have attributes with values. You can bind each set of values separately to the same type if you like.

Spring supports two polar configuration styles: explicit configuration and auto-wiring. Explicit configuration is verbose but maintainable. Auto-wiring is concise but slow and not well suited to non-trivial applications. If you have 100s of developers and 100s of thousands of lines of code, auto-wiring isn't an option. Guice uses annotations to support a best-of-both-worlds approach which is concise but explicit (and maintainable).

Some critics have tried to find an analog to Spring's XML (or new Java-based configuration) in Guice. Most times, there isn't any, but sometimes you can't use annotations, and you need to manually wire up an object (a 3rd party component for example). When this comes up, you write a custom provider. A custom provider adds one level of indirection to your configuration and is roughly 1:1 with Spring's XML in complexity. The remaining majority of the time, you'll write significantly less code. 

Eric Burke recently [ported a Spring application to Guice](http://stuffthathappens.com/blog/2007/03/09/guicy-good/):
```
  At the end of the day, I compared my (Guice) modules — written in Java — to my 
  Spring XML files. The modules are significantly smaller and easier to read.

  Then I realized about 3/4 of the code in my modules was unnecessary, 
  so I stripped it out. They were reduced to just a few lines of code each.
```
Oftentimes, it doesn't make sense to port perfectly working code which you won't be changing any time soon to Guice, so Guice works with Spring when possible. You can [bind your existing Spring beans](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/spring/SpringIntegration.html). Guice supports AOP Alliance method interceptors which means you can use Spring's popular transaction interceptor.

From our experience, most AOP frameworks are too powerful for most uses and don't carry their own conceptual weight. Guice builds method interception in at the core instead of as an afterthought. This hides the mechanics of AOP from the user and lowers the learning curve. Using one object instead of a separate proxy reduces memory waste and more importantly reduces the chances of invoking a method without interceptors. Under the covers, Guice uses a separate handler for each method minimizing performance overhead. If you intercept just one method in a class, the other methods will not incur overhead.
# Guice40.md

_Guice 4.0 Beta Release_
### Guice 4.0 - Beta 5
Released Sept 24, 2014<br>
_Note: This date and page will be updated upon final release_

#### Downloads

 * Guice: [guice-4.0-beta5.jar](http://search.maven.org/remotecontent?filepath=com/google/inject/guice/4.0-beta5/guice-4.0-beta5.jar)
 * Guice (No AOP): [guice-4.0-beta5-no_aop.jar](http://search.maven.org/remotecontent?filepath=com/google/inject/guice/4.0-beta5/guice-4.0-beta5-no_aop.jar)
 * Guice extensions are all directly downloadable [from this search page](http://search.maven.org/#search%7Cga%7C1%7Cg%3A%22com.google.inject.extensions%22%20AND%20v%3A%224.0-beta5%22).  Just click on the "jar" link for the appropriate extension.

#### Docs

_These are links to the latest documentation, which may have changed since the beta was cut._
  * [API documentation](https://google.github.io/guice/api-docs/latest/javadoc/index.html)
  * [API Changes from 3.0 to 4.0, by JDiff](http://google.github.io/guice/api-docs/latest/api-diffs/changes.html)

#### New Features

  * Better Java8 runtime compatibility
  * _List of changes pending release._

#### Migrating from Guice 3.0
See the [JDiff change report](http://google.github.io/guice/api-docs/latest/api-diffs/changes.html) for complete details.


# Guice30.md

_Guice 3.0 Release_
### Guice 3.0
Released March 24, 2011

#### Downloads
 * **[guice-3.0.zip](https://github.com/google/guice/releases/download/3.0/guice-3.0.zip)** core & extension jars.  866KB
 * **[guice-3.0-src.zip](https://github.com/google/guice/archive/3.0.zip)** sources, tests, dependencies and Javadoc. 23.8MB
 * **[guice-3.0-no_aop.jar](https://github.com/google/guice/releases/download/3.0/guice-3.0-no_aop.jar)** core without AOP, suitable for Android.  470KB

#### Docs
  * [Javadoc API](http://google.github.io/guice/api-docs/3.0/javadoc/index.html)
  * [API Changes from 2.0 to 3.0, by JDiff](http://google.github.io/guice/api-docs/3.0/api-diffs/changes.html)

#### New Features
  * [[JSR 330 support|JSR330]]
  * [[New Persist extension|GuicePersist]]
  * [[ToConstructor Bindings|ToConstructorBindings]]
  * Better OSGi support (and generally improved support for multiple classloaders)
  * New methods in Binder: `requireExplicitBindings` & `disableCircularProxies`
  * Much simpler stack traces when AOP is involved
  * Exact duplicate bindings are ignored (instead of an exception being thrown)
  * Repackaged cglib, asm & guava classes are now hidden from IDE auto-imports
  * Source can be built with Maven
  * General extension & SPI improvements:
     * [[Extensions SPI|ExtensionSPI]], with support from the [[Multibindings]], [[ServletModule]] and [[AssistedInject]] extensions.
     * `@Toolable`, for use by extensions in Stage.TOOL
     * All binding implementations implement `HasDependencies`
  * Scope improvements:
     * `Scopes.isSingleton`
     * toInstance bindings are considered eager singletons in the Scope SPI
     * Singleton providers that return null are now scoped properly.
     * Fail when circularly dependent singletons cannot be proxied (previously two instances may have been created)     
  * Provider Method (@Provides) improvements:
     * Logger dependencies now use named instead of anonymous loggers
     * Exceptions produce a simpler stack 
  * Private Modules improvements:
     * If a single `PrivateModule` is passed to Modules.override, private elements can be overridden.
     * If many modules are passed to Modules.override, exposed keys can be overridden.
  * `Multibinder` & `MapBinder` extension improvements:
     * `permitDuplicates` -- allows `MapBinder` to inject a `Map<K, Set<V>>` and a `Map<K, Set<Provider<V>>` and `Multibinder` to silently ignore duplicate entries
     * Expose dependencies in Stage.TOOL, and return a dependency on Injector (instead of null) when working with Elements.
     * Co-exist nicely with Modules.override
     * Work properly with marker (parameterless) annotations
     * Expose details through the new extensions SPI
  * Servlet extension improvements:
     * Added `with(instance)` for servlet bindings.
     * Support multiple injectors using ServletModule in the same JVM.
     * Support thread-continuations (allow background threads to process an `HttpRequest`)
     * Support manually seeding a `@RequestScope`
     * Expose details through new extensions SPI
     * Fix request forwarding
     * Performance improvements for the filter & servlet pipelines
     * Expose the servlet context (`ServletModule.getServletContext`)
     * Fixed so a single ServletModule instance can be used twice (in, say Element.getElements & creating an Injector)
     * Fix to work with context paths
  * [[AssistedInject]] extension improvements
     * New `FactoryModuleBuilder` for building assisted factories
     * Injection is done through child injectors and can participate in AOP
     * Performance improvements
     * Multiple different implementations can be created from a single factory
     * Incorrect implementations or factories are detected in Stage.TOOL, and more error-checking with better error messages
     * Work better with parameterized types
     * Expose details through new extensions SPI
  * [[ThrowingProviders]] extension improvements
     * Added a new CheckedProviders interface that allows more than one exception to be thrown.
     * Added @CheckedProvides allowing @Provides-like syntax for CheckedProviders.
     * Dependencies are checked at Injector-creation time and during Stage.TOOL, so they can fail early
     * Bindings created by ThrowingProviderBinder now return proper dependencies instead of Injector.
  * [[Struts2|Struts2Integration]]
     * The Struts2 extension now works again! (It worked in Guice 1.0 and was somewhat broken in Guice 2.0)
  * Added to Injector: `getAllBindings`, `getExistingBinding`, `getScopeBindings`, `getTypeConverterBindings`.
  * Added to Key: `hasAttributes`, `ofType`, `withoutAttributes`
  * Various bug fixes / improvements:
     * Prevent child injectors from rebinding a parent just-in-time binding.
     * Non-intercepted methods in a class with other intercepted methods no longer go through cglib (reducing total stack frames).
     * Allow Guice to work on generated classes.
     * Fix calling Provider.get() of an unrelated Provider in a scoping method.
     * `MoreTypes.getRawTypes` returns the proper types for arrays (instead of Object[].class)
     * JIT bindings left in a parent injector when child injectors fail creating a JIT binding are now removed
     * Overriding an @Inject method (if the subclass also annotates the method with @Inject) no longer injects the method twice
     * You can now bind byte constants (using `ConstantBindingBuilder.to(byte)`)
     * Better exceptions when trying to intercept final classes, using keys already bound in child or private module, and many other cases.
     * ConvertedConstantBinding now exposes the TypeConverterBinding used to convert the constant.

#### Migrating from Guice 2.0
See the [JDiff change report](http://google.github.io/guice/api-docs/3.0/api-diffs/changes.html) for complete details.

#### JSR 330
Guice 3.0 now requires JSR 330 on your classpath.  This is the javax.inject.jar included in the guice download.

#### com.google.inject.internal
Many things inside com.google.inject.internal changed and/or moved.  This is especially true for repackaged Guava (formerly Google Collections), cglib, and asm classes.  All these classes are now hidden from IDE auto-import suggestions and are in new locations.  You will have to update your code if you relied on any of these classes.

# Guice20.md

_Guice 2.0 Release_
### Guice 2.0
Released May 19, 2009

##### Downloads
  * **[guice-2.0.zip](https://github.com/google/guice/releases/download/2.0/guice-2.0.zip)** jars and Javadoc. 1.1MB
  * **[guice-2.0-src.zip](https://github.com/google/guice/archive/2.0.zip)** sources, tests, dependencies and Javadoc. 16.5MB
  * **[guice-2.0-no_aop.jar](https://github.com/google/guice/releases/download/2.0/guice-2.0-no_aop.jar)** replaces guice-2.0.jar on platforms that don't support AOP (ie. Android). 430KB

##### Docs
  * [Javadoc API](http://google.github.io/guice/api-docs/2.0/javadoc/index.html)
  * [User's Guide](https://github.com/google/guice/wiki/Motivation)

##### New Features

###### Small Features
  * `Binder.getProvider` and `AbstractModule.requireBinding` allow modules to declare and use dependencies.
  * `Binder.requestInjection` allows modules to register instances for injection at Injector-creation time.
  * `Providers.of()` always provides the same instance, useful for testing.
  * `Scopes.NO_SCOPE` allows you make no-scoping explicit.
  * `Matcher.inSubpackage` matches all classes in any child package of the given one.
  * `Types` utility class for creating implementations of generic types.
  * `AbstractModule.requireBinding` allows modules to document dependencies programatically

###### Provider Methods
Creating custom providers without all of the boilerplate. In any module, annotate a method with `@Provides` to define logic that will be used to provide that type:
```java
class PaymentsModule extends AbstractModule {
  public void configure() {
    ...
  }

  @Provides @Singleton
  PaymentService providePaymentService(CustomerDb database) {
    DatabasePaymentService result = new DatabasePaymentService();
    result.setDatabase(database);
    result.setReplicationLevel(5);
    return result;
  }
```
Parameters to the @Provides method will be injected. It can optionally be annotated with a scoping annotation (like `@Singleton`). The method's returned type is the bound type. Annotate the method with a binding annotation (like `@Named("replicated")`) to create a binding for the annotated type.

###### Binding Overrides
[Override](http://publicobject.com/2008/05/overriding-bindings-in-guice.html) the bindings from one module with another:
```java
Module functionalTestModule 
        = Modules.override(new ProductionModule()).with(new OverridesModule());
```

###### Multibindings, MapBindings
Bind elements of [Sets](http://publicobject.com/2008/05/guice-multibindings-extension-checked.html) and [Maps](http://publicobject.com/2008/05/mapbinder-checked-in.html), then inject the full collection. Bindings are aggregated from all modules.
```java
public class SnacksModule extends AbstractModule {
  protected void configure() {
    MapBinder<String, Snack> mapBinder
        = MapBinder.newMapBinder(binder(), String.class, Snack.class);
    mapBinder.addBinding("twix").toInstance(new Twix());
    mapBinder.addBinding("snickers").toProvider(SnickersProvider.class);
    mapBinder.addBinding("skittles").to(Skittles.class);
  }
}
```

###### Private Modules
[PrivateModules](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/PrivateModule.html) can be used to create bindings that are not externally visible. This makes it easier to encapsulate dependencies and to avoid bind conflicts.

###### Servlets
`ServletModule` now supports programmatic configuration of servlets and filters (get rid of web.xml). 

These servlets may be:
  * constructor injected directly
  * support AOP
  * circular references, and all Guice POJO idioms.

Guice Servlet also supports regex URL mapping and the ability to package and bundle your own servlet libraries (for example, an SOAP web service).

`GuiceServletContextListener` can be used to help bootstrap a Guice application in a servlet container.

###### Child Injectors
[Injector.createChildInjector](http://google.github.io/guice/api-docs/latest/javadoc/com/google/inject/Injector.html#createChildInjector(java.lang.Iterable)) allows you to create child injectors that inherit the bindings, scopes, interceptors and converters of their parent. This API is primarily intended for extensions and tools.

###### Even Better Error Reporting
Exceptions in Guice 1.0 tend to include long chains of 'caused by' exceptions. We've tidied this up! Now a single exception describes concisely what Guice was doing when the problem occurred.

###### Introspection API
Like `java.lang.reflect`, but for Guice. It lets you rewrite a Module, tweaking bindings programatically. It also lets you inspect a created injector, and examine its bindings. This is intended to enable simpler, more powerful extensions and tools for Guice.

###### Custom Injections
[[Type and instance listeners|CustomInjections]] enable Guice to host other frameworks that have their own injection semantics or annotations.

###### Pluggable Type Converters
Constant String bindings can be converted to arbitrary types (such as dates, URLs, or colours) using pluggable type converters.

###### Available without AOP
Guice 2.0 is available [[without AOP|OptionalAOP]] for platforms like [Android](http://code.google.com/android/) that don't support bytecode manipulation.

###### OSGi-friendly AOP
Guice does bytecode generation internally to implement AOP. In version 2.0, generated classes are loaded by a bridge classloader that works in managed environments such as OSGi.

###### Type Resolution
[Parameterized injection points](http://groups.google.com/group/google-guice/browse_thread/thread/1355313a074d8094/88270edbbeae2df8) allow you to inject types like `Reducer<T>` or `Converter<A, B>`. Guice will figure out what `T` is and find the binding for that type. [TypeLiteral injection](http://publicobject.com/2008/11/guice-punches-erasure-in-face.html) means you can inject a `TypeLiteral<T>` into your classes. Use this to reify [Java 5 type erasure](http://java.sun.com/docs/books/tutorial/java/generics/erasure.html). The `TypeLiteral` class now includes library methods for manual type resolution.

###### Migrating from Guice 1.0
Guice 2.0 breaks compatibility with Guice 1.0 as described below. See the [JDiff change report](http://google.github.io/guice/api-docs/2.0/api-diffs/changes.html) for complete details. 

###### Exception Types
In Guice 1.0, when a custom provider throws an unchecked exception, sometimes Guice wraps the exception and sometimes it doesn't. This depends on whether the provider is being called directly (via Provider.get()) or indirectly (such as for injecting into another type).

In Guice 2.0, any time a provider or injection throws, Guice wraps it in a ProvisionException. [This rule is simpler](http://publicobject.com/2008/04/future-guice-providers-that-throw.html), and it makes it easier to write fault-tolerant code with Guice.

Injector creation problems are always reported as CreationException. Runtime configuration problems (ie. programmer errors) are always reported as ConfigurationException.

ProvisionException, ConfigurationException and OutOfScopeException are now public.

###### Abstract Types
Guice doesn't support injecting into abstract types. The messaging around this has been improved since 1.0, and some code that was silently failing now throws exceptions.

###### Inner Classes
Guice used to support constructor injection of nonstatic inner classes. So this used to work, but it won't anymore:
```java
public class FooTest extends TestCase {
  public void testFoo() {
    Foo foo = Guice.createInjector().getInstance(FakeFoo.class)
  }

  class FakeFoo implements Foo {
    @Inject TestFoo() {...}
  }
}
```

###### Keys
Guice now [canonicalizes](http://publicobject.com/2008/06/integerclass-and-intclass-as-guice-keys.html) primitive types (like `int.class`) and array types (like `Integer[].class`) when they're used in Keys. It now supports wildcards like `List<?>` in keys - use `@Provides` to bind these.

###### Annotation Implementations
Guice 2.0 fixes the treatment of equals() and hashCode() for fieldless annotations. Annotation implementations that don't implement equals() and hashCode() may have worked in 1.0 but will be broken in 2.0.

###### Injector.getBinding
Guice 2.0 throws an exception if the binding cannot be resolved. The old version used to return null. To get the old behaviour, use `injector.getBindings().get(key)`.

###### SPI Changes
SourceProviders have been replaced with `Binder.withSource` and `Binder.skipSources`. These new methods are easier to call and test. They don't require static initialization or static dependencies.

# Guice10.md

_Guice 1.0 Release_

### Guice 1.0
Released March 8, 2007.

#### Downloads
  * [guice-1.0.zip](https://github.com/google/guice/releases/download/1.0/guice-1.0.zip)
  * [guice-1.0-src.zip](https://github.com/google/guice/archive/1.0.zip)
  * [guice-struts2-plugin-1.0.1.jar](https://github.com/google/guice/releases/download/1.0/guice-struts2-plugin-1.0.1.jar)


#### Docs
  * [Javadoc API](http://google.github.io/guice/api-docs/1.0/javadoc/index.html)
  * [Guice 1.0 User's Guide.pdf](http://google.github.io/guice/user-docs/Guice-1.0-Users-Guide.pdf)

#### Changes

##### 1.0
  * Changed constant binding API. Replaced `bindConstant(annotation)` with `bindConstant().annotatedWith(annotation)`. 
  * Fixed bug in ordering of instance injection.

##### 1.0rc3
  * Renamed `Container` to `Injector`
  * Renamed "container scope" to "singleton"
  * Guice now blows up if a custom provider returns `null`
  * Added `@ImplementedBy` and `@ProvidedBy`
  * Removed `Binder.link()`. `Binder.bind().to(...)` now links.
  * Further fleshed out binder expression language.
  * Added Spring integration.
  * Added JNDI integration.
  * Made `CreationException` a runtime exception.

##### 1.0rc2 (changes since 1.0rc1)
  * Added `@ScopeAnnotation`. If you forget to bind a scope, you'll get an error.
  * Added `Binder.addError()` so you can record custom error messages from modules.
  * Removed `Scopes.DEFAULT`. We now refer to this as "no scope."
  * Added up front checks to ensure annotations have runtime retention and the proper meta annotations.
  * Added support for injecting private and protected members (cglib fast reflection doesn't support this).
  * Renamed `Locator` to `Provider`
  * Replaced `Factory` with `Provider` also
  * Removed `Context` from public API. We may support this functionality via injection in a future release but we're leaving it out for now.
  * Extracted an interface for `Binding`
  * We now inject members of instances bound using `toInstance()` and `toProvider()` upon container creation.
  * When a binding annotation has attributes, if we can't find a binding to the same set of attribute values, we now look for a binding to just the annotation type.
  * Created a separate jar for the `servlet` package
  * Created a Struts 2 plugin jar

# ExternalDocumentation.md

_Blogs, articles, books on Guice._
### External Documentation

#### Books

##### [Dependency Injection by Dhanji Prasanna](http://www.manning.com/prasanna/)
  In a traditional object-oriented application, a primary program controls secondary pieces of code, such as classes in a module, library, or framework. Dependency Injection (DI) is a technique that inverts this control, using an external mechanism to insert—or inject—a reference to an implementation of a service into an object. This allows you to build complex OO applications in a more testable, maintainable, and business-focused manner.

##### [Google Guice by Robbie Vanbrabant](http://www.apress.com/9781590599976)
  _Google Guice: Agile Lightweight Dependency Injection Framework_ will not only tell you “how,” it will also tell you “why” and “why not,” so that all the knowledge you gain will be as widely applicable as possible. Filled with examples and background information, this book is an invaluable addition to your knowledge of modern agile Java.

#### Videos
##### [Big Modular Java with Guice](http://code.google.com/events/io/sessions/BigModularJavaGuice.html)
An introduction to Guice by Dhanji R. Prasanna and Jesse Wilson, at Google I/O 2009.
  Learn how Google uses the fast, lightweight Guice framework to power some of the largest and most complex applications in the world. Supporting scores of developers, and steep testing and scaling requirements for the web, Guice proves that there is still ample room for a simple, type-safe and dynamic programming model in Java. This session will serve as a simple introduction to Guice, its ecosystem and how we use it at Google.

##### [Introduction to Google Guice: Programming is fun again!](http://developers.sun.com/learning/javaoneonline/sessions/2009/pdf/TS-5434.pdf)
PDF slides from the sold-out technical session at !JavaOne 2009.

#### Articles
##### [Dependency injection with Guice by Nicholas Lesiecki](http://www.ibm.com/developerworks/java/library/j-guice/index.html)
  Guice is a dependency injection (DI) framework. I've suggested for years that developers use DI, because it improves maintainability, testability, and flexibility. By watching engineers react to Guice, I've learned that the best way to convince a programmer to adopt a new technology is to make it really easy. Guice makes DI really easy, and as a result, the practice has taken off at Google. I hope to continue in the same vein in this article by making it really easy for you to learn Guice.


##### [Guicing Up Your Testing by Dick Wall](http://www.developer.com/design/article.php/3684656/Guicing-Up-Your-Testing.htm)
  This article examines the simplest and most obvious use case for the Guice container, for mocking or faking objects in unit tests.

  _Also available in [Portuguese](http://fabiolnm.blogspot.com/2009/11/teste-unitario-com-junit-e-google-guice.html)_

##### [Squeezing More Guice from Your Tests with EasyMock by Dick Wall](http://www.developer.com/design/article.php/3688436/Squeezing-More-Guice-from-Your-Tests-with-EasyMock.htm)
   It should be apparent that making Invoice something that Guice creates directly is an incorrect design decision. Instead, you can mix it with another pattern instead: Factory.


#### Blogs

##### [Guice with GWT](http://stuffthathappens.com/blog/2009/09/14/guice-with-gwt/)
Configure your serverside app to serve up a backend for GWT apps.
  Obtain the Guice JAR files, extend GWT’s `RemoteServiceServlet`, extend Guice’s `GuiceServletContextListener`, extend Guice’s `ServletModule`, set all `RemoteService` relative paths to _GWT.rpc_, and configure `GuiceFilter` and your context listener in _web.xml_.

##### [Refactoring to Guice: Part 1 of N](http://publicobject.com/2007/07/guice-patterns-1-horrible-static-code.html)
Guide for migrating from factories to dependency injection.
  In this N-part series, I'm attempting to document some patterns for improving your code with Guice. In each example, I'll start with sample code, explain what I don't like about it, and then show how I clean it up.

##### [What's a Hierarchical Injector?](http://publicobject.com/2008/06/whats-hierarchical-injector.html)
  The premise is simple. @Inject anything, even stuff you don't know at injector-creation time. So our DeLorean class would look exactly as it would if EnergySource was constant:

##### [TypeResolver tells you what List.get() returns](http://publicobject.com/2008/07/typeresolver-tells-you-what-listget.html)
What the new methods on `TypeLiteral` do.

##### [Simpler service interfaces with Guice](http://publicobject.com/2007/11/simpler-service-interfaces-with-guice.html)
A brief discussion on contextual APIs.

##### [Guice AOP Example](http://sarah-a-happy.livejournal.com/145875.html)
```java
  @Trace("note goes here")
  public void bye() {
    System.out.println("see you later");
  }
```

##### [Alternative way to integrate Guice with Wicket](http://headtoscreencollision.blogspot.com/2010/05/wicket-and-guice-alternate-route.html)
Explains a better way to integrate Guice and Wicket, using Guice 2.0 and ServletModule which allows you to get rid of even more XML in your projects!
# AppsThatUseGuice.md

_a list of applications that use Guice_

  * Google ([AdWords Frontend](http://adwords.google.com/), [Google Wave](http://wave.google.com/) and several others)
  * [Google Sitebricks](http://code.google.com/p/google-sitebricks): a light, fast web app framework
  * [LimeWire](http://www.limewire.com)
  * [3banana](http://3banana.com/)
  * Apache Shindig

See also [[3rdPartyModules|3rdPartyModules]] for frameworks and libraries that use Guice.
# GuiceDiscussions.md

_Links to Guice discussions_

  * http://www.theserverside.com/news/thread.tss?thread_id=44593
  * http://www.oreillynet.com/onjava/blog/2007/03/the_next_spring_google_guice.html
  * http://jroller.com/page/Solomon?entry=ioc_di_alliance
  * http://www.infoq.com/news/2007/03/guice
  * http://crazybob.org/2007/03/guice-10.html
  * http://forum.springframework.org/showthread.php?t=35768
  * http://www.artima.com/forums/flat.jsp?forum=270&thread=198371
  * http://www.kallisti.net.nz/blog/2007/03/dependency-injection-is-my-new-best-friend/
  * http://stuffthathappens.com/blog/2007/03/09/guicy-good/
  * http://jroller.com/page/habuma?entry=guice_vs_spring_javaconfig_a
  * http://programming.reddit.com/info/1cxfe/comments
