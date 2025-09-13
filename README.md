# The ADAPT Architecture Manifesto

### A Pragmatic Approach to Building Resilient, Context-Driven Software

-----

## Preamble: The Problem with Dogma

Our industry has a deep appreciation for principles and patterns. Frameworks like SOLID have given us a common language, but their dogmatic application in modern distributed systems often leads to over-engineered, brittle, and unnecessarily complex software. We build abstractions for futures that never arrive, couple services in rigid ways, and create systems that are difficult to change and perform poorly under pressure.

***We believe there is a better way.***

This manifesto proposes a shift in mindset: from following rigid rules to embracing **context**. We believe that a truly robust architecture is one that **adapts**—to the business domain, to the operational environment, and to the inevitable changes of the future.

This document outlines the five core principles of the **ADAPT** architecture.

-----

## The 5 Principles of ADAPT

These principles are not rules to be followed blindly, but pillars for a philosophy of pragmatic software design.

### 1\. `A` - Asynchronous Communication

> **Services must be decoupled, communicating through events rather than direct calls to build resilient and scalable systems.**

This principle dictates that services should not be tightly coupled through synchronous calls (like REST or gRPC). Instead, they should communicate via **asynchronous messaging** (e.g., message queues, event streams). This decoupling allows services to evolve independently, improves fault tolerance, and enhances scalability. Having isolated services reduces the blast radius of failures and allows for more flexible deployment strategies.

#### **Implementation Strategies**

  * Use message brokers like **Kafka**, **RabbitMQ**, or cloud-native options like **AWS SNS/SQS**.
  * Design services to be **idempotent**, ensuring that reprocessing the same message multiple times does not lead to inconsistent states.
  * Implement **Dead Letter Queues (DLQs)** for handling failed messages gracefully.
  * Favor event **choreography** over orchestration to maximize service autonomy, except for specific cases where a strict, orchestrated workflow is justified.

#### **Example: Before and After**

##### **Before: The Synchronous Chain**

```cpp
class OrderService {
    void placeOrder(Order order) {
        // Direct calls create a rigid, brittle chain of command.
        inventoryService.reserveStock(order.productId, order.quantity);
        paymentService.processPayment(order.paymentDetails);
        shippingService.scheduleDelivery(order.shippingAddress);
        notificationService.sendOrderConfirmation(order.customerId);
    }
}
```

Here, `OrderService` is a central point of failure. If any downstream service is slow or unavailable, the entire operation grinds to a halt.

##### **After: The Asynchronous Event**

```cpp
class OrderService {
    void placeOrder(Order order) {
        // Publish a fact. Let the rest of the system react.
        Event event = new Event("OrderPlaced", order);
        messageBroker.publish(event);
    }
}
```

Now, `OrderService` simply announces that an order was placed. `Inventory`, `Payment`, and other services subscribe to this event and perform their duties independently. The system is now resilient and easily extendable.

-----

### 2\. `D` - Domain-Cohesive Design

> **The architecture must mirror the business domain, with services organized around business capabilities, not technical layers.**

Your codebase should be a direct reflection of the business it serves. Instead of organizing code by technical concerns (e.g., controllers, services, repositories), we organize it around **business capabilities** (e.g., User Management, Billing, Shipping). Each service encapsulates a complete business capability, owning its data and logic.

#### **Implementation Strategies**

  * Use **Domain-Driven Design (DDD)** principles to identify bounded contexts.
  * Enforce a **"share nothing"** rule for data; services communicate via APIs and events, never by accessing another service's database directly.
  * Favor **event sourcing** and **CQRS** where appropriate to manage complex state changes.

#### **Example: Before and After**

##### **Before: The Layered Monolith**

```cpp
// In a layered architecture, logic is fragmented across technical boundaries.
class UserController { /* Handles HTTP */ }
class UserService { /* Handles business logic */ }
class UserRepository { /* Handles data access */ }
```

Changing a single business rule for user creation might require modifying all three layers, increasing the risk of bugs and making development slow.

##### **After: The Cohesive Service**

```cpp
// In a cohesive design, the service owns the entire business capability.
class UserService {
    void createUser(UserDto userDto) {
        User user = new User(userDto);
        userRepository.save(user); // Manages its own persistence.
        eventBus.publish(new Event("UserCreated", user)); // Notifies the system.
    }
}
```

Now, all the logic for creating a user lives in one place. If we need to add a welcome email, a new `NotificationService` simply listens for the `UserCreated` event.

> **Redefining Responsibility:** In the ADAPT philosophy, a service's **"single responsibility"** is the complete ownership of a **business capability**, from its API down to its data. This is a higher-level view of SRP that decouples **teams**, not just classes.

-----

### 3\. `A` - Abstraction with Purpose

> **Abstraction is a tool to be used with caution, justified by a clear and present need, not a hypothetical future.**

Abstractions are powerful but can lead to unnecessary complexity and performance hazards if used indiscriminately. We resist creating abstractions (interfaces, base classes) prematurely based on what we *might* need tomorrow (**YAGNI** - You Ain't Gonna Need It).

An abstraction is only introduced when the pain of code duplication or modification is demonstrably high. Every abstraction must pass a rigorous test: **Does it make the system simpler *right now*?** If not, we prefer concrete, simple code.

#### **The Abstraction Necessity Test**

This is a formal 3-step process to avoid hasty decisions.

##### **Step 1: Justification (The "Why?")**

  * **Concrete Problem:** What current and proven problem does this abstraction solve?
  * **Rule of Three:** Is this logic already duplicated in at least three distinct places, and have these duplications required coordinated changes in the past?
  * **Concept Stability:** Is the concept I'm abstracting stable and well-understood? Abstracting a moving target is a recipe for failure.
  * **Net Simplicity Benefit:** Will the code be manifestly simpler for a new developer to understand? (***Warning: "shorter" does not mean "simpler"***).
  * **Cost of Failure:** If this abstraction is wrong, what is the cost to remove or refactor it?

##### **Step 2: Definition (The "What?")**

  * **Naming:** Does the abstraction have a clear, unambiguous name that reflects its business intention?
  * **Boundaries:** What are the exact limits of its responsibility? What does it do, and more importantly, what does it **not** do?
  * **Usage Example:** Write pseudo-code showing how to use it. If the example is complex, the design is flawed.

##### **Step 3: Validation (The "How?")**

  * **Testability:** Is it easier to test in isolation than the code it replaces?
  * **Performance:** Is its performance impact measured and acceptable?
  * **Peer Review:** Does a teammate understand its purpose without a long verbal explanation?

-----

### 4\. `P` - Piloted by Configuration

> **A service's behavior must be directed by external configuration, designing the code as a flexible engine rather than a rigid script.**

The compiled artifact of a service should be stable and environment-agnostic. Its runtime behavior—feature flags, connection strings, business rules (like retry counts)—should be **injected through configuration**. This separates the "what the code can do" from "what the code is currently doing," enabling safer experimentation and quicker responses to changing requirements.

#### **Implementation Strategies**

  * Follow the **Twelve-Factor App** methodology for configuration management.
  * Use **Configuration as Code (CaC)** to manage configurations in version control.
  * Leverage **Feature Flags** to enable or disable features without code changes.

#### **Example: Before and After**

##### **Before: Hardcoded Behavior**

```cpp
class PaymentService {
    void processPayment(PaymentDetails details) {
        // Hardcoded retry logic is rigid and requires a redeploy to change.
        int retries = 3;
        // ... retry loop ...
    }
}
```

##### **After: Configurable Behavior**

```cpp
class PaymentService {
    private final int retries;
    // The number of retries is injected from an external source at startup.
    PaymentService(Config config) {
        this.retries = config.getInt("payment.retries", 3);
    }
    // ... retry loop uses this.retries ...
}
```

Now, the number of retries can be adjusted per environment or even in real-time without modifying the deployed code.

| **Configuration Type** | **Example** | **Where to Store It** | **Reasoning** |
| --------------------------- | ---------------------------- | ---------------------------------------------- | ------------------------------------------ |
| **Environment Config** | `DB_URL=jdbc:mysql://...`    | Environment Variables, Secrets Manager         | Sensitive data that varies by environment. |
| **Service Config** | `payment.retries=5`          | Config files (`.yaml`, `.properties`), ConfigMap | Business rules that may change frequently. |
| **Feature Flags** | `feature.newCheckout=true`   | Feature flag service, Environment Variables    | Toggle features without code changes.      |

> **Warning:** Not everything should be configurable. Over-configuration can lead to its own complexity. Expose only settings that are likely to change based on environment or operational needs.

-----

### 5\. `T` - Transparency through Contracts

> **The data that flows between services is a formal contract that must be explicit, versioned, and reliable.**

In an event-driven system, the structure of events and messages **is the API**. We use **schema-first design** (with tools like Avro, Protobuf, or JSON Schema) to define these data contracts. By making data contracts the single source of truth, the entire system's potential interactions become **visible, auditable, and understandable** to everyone involved.

#### **Implementation Strategies**

  * Use **schema registries** to manage and version event schemas.
  * Enforce **backward and forward compatibility** in your schemas to allow services to evolve independently.
  * Use **contract testing** (e.g., Pact) to verify that services adhere to the defined contracts.

#### **Example: Before and After**

##### **Before: Implicit Contracts**

An event is just a class in a shared library. If Service A changes the class, Service B might break at runtime in unexpected ways. The contract is fragile and based on convention.

##### **After: Explicit Contracts with a Schema**

```json
{
  "type": "record",
  "name": "OrderPlacedEvent",
  "fields": [
    {"name": "orderId", "type": "string"},
    {"name": "productId", "type": "string"},
    {"name": "quantity", "type": "int"},
    {"name": "customerId", "type": ["null", "string"], "default": null} 
  ]
}
```

With a formal schema, changes are deliberate. The schema registry can enforce compatibility rules, preventing breaking changes from ever being deployed. The system's communication is now robust and transparent.

-----

## Join the Movement

This is a living document, owned by the community that uses it. If these principles resonate with you, here is how you can get involved:

* ⭐ **Star this repository** to show your support and help it gain visibility.
* 🤔 **Start a discussion** by [opening an Issue](https://github.com/Mideky-hub/adapt-architecture-principle/issues) to debate a principle or propose a clarification.
* ✍️ **Contribute** by fixing a typo or improving the wording via a Pull Request.

### Show Your Support

Add this badge to your project's `README.md` to show that you are building your architecture with the ADAPT principles in mind.

[](https://github.com/Mideky-hub/adapt-architecture-principle/)

#### Copy this markdown snippet:

```markdown
[![ADAPT Architecture](https://img.shields.io/badge/Architecture-ADAPT-blueviolet)](https://github.com/Mideky-hub/adapt-architecture-principle/)
```