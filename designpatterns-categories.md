# Understanding Design Patterns: Creational, Structural, and Behavioral Categories

## The Problem

Imagine you're building a payment microservice for an e-commerce platform. Your team hardcoded Stripe payment logic throughout 50+ files. Six months later, business wants to add PayPal and Apple Pay support. Now you're facing:

- 200+ code changes across multiple services
- 2-week deployment cycle with high risk
- Broken tests everywhere
- Every new payment gateway = full system refactor

**Design patterns solve exactly this.** With a Factory pattern, adding PayPal takes 2 hours and zero changes to existing code.

In large-scale cloud systems and microservices, codebases often grow messy with duplicated logic, rigid structures, and tangled communications. Design patterns offer proven, reusable solutions to these problems, categorized into three main types by the "Gang of Four" book: **creational**, **structural**, and **behavioral**. This post breaks them down with backend examples to help you recognize and apply them in real projects.

## Quick Pattern Selector

**Don't know which pattern to use? Start here:**

### Ask Yourself:
- **Creating objects and it's complicated?** → Go to **Creational Patterns**
  - Need exactly one instance? → **Singleton**
  - Complex setup with many options? → **Builder**
  - Need different types at runtime? → **Factory Method**
  - Need families of related objects? → **Abstract Factory**

- **Connecting components with different interfaces?** → Go to **Structural Patterns**
  - Incompatible interfaces? → **Adapter**
  - Want to add features without changing code? → **Decorator**
  - Need to simplify complex subsystems? → **Facade**
  - Want to control access? → **Proxy**

- **Managing object behavior and communication?** → Go to **Behavioral Patterns**
  - Multiple objects need to know about changes? → **Observer**
  - Need to swap algorithms at runtime? → **Strategy**
  - Want to queue/undo operations? → **Command**
  - Objects go through states? → **State**

## Creational Patterns

These patterns focus on flexible object creation, abstracting the instantiation process so your code isn't tied to concrete classes.

- They solve issues like complex setup logic or varying object types at runtime, common in services needing different DB clients or configs.
- Key examples: **Singleton** (single config manager), **Factory Method** (plugin-based factories for payment gateways), **Abstract Factory** (families of related SDK objects), **Builder** (step-by-step API request builders), **Prototype** (cloning cache entries).
- Real-world example: In Azure microservices, a Factory creates connection pools dynamically based on env vars, avoiding hardcoded clients.

### Real Code Example: Factory Pattern

**❌ Without Factory (Rigid & Hard to Maintain):**
```java
// Payment service - tightly coupled to Stripe
public class PaymentService {
    private StripeClient stripeClient;

    public PaymentService() {
        this.stripeClient = new StripeClient("sk_live_...");
    }

    public String processPayment(BigDecimal amount, String currency) {
        try {
            Charge charge = stripeClient.charges().create(
                amount,
                currency,
                "tok_visa"
            );
            return charge.getId();
        } catch (StripeException e) {
            return null;
        }
    }
}

// Problem: What if we need PayPal? We'd need to:
// 1. Modify PaymentService class
// 2. Update 50+ files that use it
// 3. Conditional logic everywhere: if (provider.equals("stripe")) else if (provider.equals("paypal"))
// 4. Testing nightmare - can't mock external services
```

**✅ With Factory Pattern (Flexible & Scalable):**
```java
// Define payment processor interface
public interface PaymentProcessor {
    String processPayment(BigDecimal amount, String currency);
}

// Concrete implementations
public class StripeProcessor implements PaymentProcessor {
    private final StripeClient client;

    public StripeProcessor(String apiKey) {
        this.client = new StripeClient(apiKey);
    }

    @Override
    public String processPayment(BigDecimal amount, String currency) {
        try {
            Charge charge = client.charges().create(amount, currency);
            return charge.getId();
        } catch (StripeException e) {
            throw new PaymentProcessingException("Stripe payment failed", e);
        }
    }
}

public class PayPalProcessor implements PaymentProcessor {
    private final PayPalClient client;

    public PayPalProcessor(String clientId, String secret) {
        this.client = new PayPalClient.Builder()
            .mode("live")
            .clientId(clientId)
            .clientSecret(secret)
            .build();
    }

    @Override
    public String processPayment(BigDecimal amount, String currency) {
        Payment payment = new Payment();
        payment.setAmount(amount);
        payment.setCurrency(currency);
        return client.createPayment(payment).getId();
    }
}

public class ApplePayProcessor implements PaymentProcessor {
    @Override
    public String processPayment(BigDecimal amount, String currency) {
        // Apple Pay implementation
        return "apple_pay_transaction_id";
    }
}

// Factory to create the right processor
public class PaymentProcessorFactory {

    public static PaymentProcessor createProcessor(String provider) {
        switch (provider.toLowerCase()) {
            case "stripe":
                return new StripeProcessor(
                    System.getenv("STRIPE_KEY")
                );
            case "paypal":
                return new PayPalProcessor(
                    System.getenv("PAYPAL_CLIENT_ID"),
                    System.getenv("PAYPAL_SECRET")
                );
            case "applepay":
                return new ApplePayProcessor();
            default:
                throw new IllegalArgumentException(
                    "Unknown payment provider: " + provider
                );
        }
    }
}

// Configuration service to manage application settings
public class ConfigService {
    private final Properties properties;

    public ConfigService() {
        this.properties = new Properties();
        loadConfiguration();
    }

    private void loadConfiguration() {
        // Load from environment variables, config file, or database
        String provider = System.getenv("PAYMENT_PROVIDER");
        if (provider != null) {
            properties.setProperty("payment.provider", provider);
        } else {
            // Default to stripe
            properties.setProperty("payment.provider", "stripe");
        }
    }

    public String getPaymentProvider() {
        return properties.getProperty("payment.provider", "stripe");
    }

    public void setPaymentProvider(String provider) {
        properties.setProperty("payment.provider", provider);
    }
}

// Usage - completely decoupled
public class OrderService {
    private final ConfigService config;

    public OrderService(ConfigService config) {
        this.config = config;
    }

    public String checkout(String orderId, BigDecimal amount, String currency) {
        // Get provider from config or user preference
        String provider = config.getPaymentProvider();

        // Factory creates the right processor
        PaymentProcessor processor = PaymentProcessorFactory.createProcessor(provider);

        // Process payment - same interface for all providers!
        String paymentId = processor.processPayment(amount, currency);

        return paymentId;
    }
}

// Example usage with dependency injection
public class Main {
    public static void main(String[] args) {
        // Create config service
        ConfigService config = new ConfigService();

        // Inject into order service
        OrderService orderService = new OrderService(config);

        // Process order
        String paymentId = orderService.checkout(
            "ORDER-123", 
            new BigDecimal("99.99"), 
            "USD"
        );

        System.out.println("Payment processed: " + paymentId);
    }
}
```

**Real-World Impact:**
- ✅ Added PayPal support in **2 hours** (previously 2 weeks)
- ✅ A/B test payment providers by changing config (no deployment needed)
- ✅ Easy to mock processors in unit tests
- ✅ Added Apple Pay in **30 minutes** - just created new class
- ✅ Each team can own their processor independently

| Pattern          | Problem Solved                  | Backend Example              | Use When |
|------------------|---------------------------------|------------------------------|----------|
| Singleton       | Global access to one instance  | Shared Redis cache client    | Need exactly one instance (config manager, connection pool, logger) |
| Builder         | Complex object construction    | HTTP request with auth/headers | >3 constructor params or many optional configurations |
| Factory Method  | Subclass-defined creation      | Different logger types (Console, File, CloudWatch) | Need to select object type at runtime based on conditions |
| Abstract Factory | Create families of related objects | AWS SDK vs Azure SDK clients | Need consistent families of related objects |
| Prototype       | Clone existing objects         | Duplicate user profiles with modifications | Creating objects is expensive, easier to clone |

## Structural Patterns

These deal with class and object composition, making large structures flexible without altering interfaces.

- Purpose: Simplify relationships between entities, ideal for layering services or adapting legacy components.
- Key examples: **Adapter** (wrap third-party APIs), **Composite** (tree of services like nested handlers), **Decorator** (add logging/metrics dynamically), **Facade** (unified cloud client interface), **Proxy** (lazy-loaded or cached DB access).
- Real-world example: A Facade hides Azure DevOps pipeline complexities, letting services call one method instead of juggling multiple APIs.

### Real Code Example: Decorator Pattern

**❌ Without Decorator (Violates Open/Closed Principle):**
```java
// HTTP client with hardcoded features
public class HttpClient {

    public Response get(String url) {
        // Make request
        Response response = executeRequest(url);

        // Hardcoded logging
        System.out.println("GET " + url + " -> " + response.getStatusCode());

        // Hardcoded retry logic
        if (response.getStatusCode() >= 500) {
            try {
                Thread.sleep(1000);
                response = executeRequest(url);  // Retry once
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }

        // Hardcoded metrics
        MetricsCollector.increment("http.requests");

        return response;
    }

    private Response executeRequest(String url) {
        // Actual HTTP request
        return new OkHttpClient().newCall(
            new Request.Builder().url(url).build()
        ).execute();
    }
}

// Problem: Need to modify class for every new feature
// What if we want: caching, auth, rate limiting, circuit breaker?
// Can't enable/disable features dynamically
```

**✅ With Decorator Pattern (Open for Extension, Closed for Modification):**
```java
// Base component interface
public interface HttpClient {
    Response get(String url) throws IOException;
}

// Core implementation
public class BasicHttpClient implements HttpClient {
    private final OkHttpClient client;

    public BasicHttpClient() {
        this.client = new OkHttpClient();
    }

    @Override
    public Response get(String url) throws IOException {
        Request request = new Request.Builder()
            .url(url)
            .build();
        return client.newCall(request).execute();
    }
}

// Decorator base class
public abstract class HttpClientDecorator implements HttpClient {
    protected final HttpClient client;

    public HttpClientDecorator(HttpClient client) {
        this.client = client;
    }

    @Override
    public Response get(String url) throws IOException {
        return client.get(url);
    }
}

// Concrete decorators - each adds ONE feature
public class LoggingDecorator extends HttpClientDecorator {
    private static final Logger logger = LoggerFactory.getLogger(LoggingDecorator.class);

    public LoggingDecorator(HttpClient client) {
        super(client);
    }

    @Override
    public Response get(String url) throws IOException {
        logger.info("[LOG] Requesting {}", url);
        Response response = client.get(url);
        logger.info("[LOG] Response: {}", response.code());
        return response;
    }
}

public class RetryDecorator extends HttpClientDecorator {
    private final int maxRetries;

    public RetryDecorator(HttpClient client, int maxRetries) {
        super(client);
        this.maxRetries = maxRetries;
    }

    @Override
    public Response get(String url) throws IOException {
        IOException lastException = null;

        for (int attempt = 0; attempt < maxRetries; attempt++) {
            try {
                Response response = client.get(url);
                if (response.code() < 500) {
                    return response;
                }
            } catch (IOException e) {
                lastException = e;
                if (attempt == maxRetries - 1) {
                    throw e;
                }
            }

            // Exponential backoff
            try {
                Thread.sleep((long) Math.pow(2, attempt) * 1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new IOException("Interrupted during retry", e);
            }
        }

        throw lastException;
    }
}

public class MetricsDecorator extends HttpClientDecorator {
    private final MetricsCollector metrics;

    public MetricsDecorator(HttpClient client, MetricsCollector metrics) {
        super(client);
        this.metrics = metrics;
    }

    @Override
    public Response get(String url) throws IOException {
        long startTime = System.currentTimeMillis();

        try {
            Response response = client.get(url);
            long duration = System.currentTimeMillis() - startTime;

            metrics.recordTiming("http.request.duration", duration);
            metrics.increment("http.status." + response.code());

            return response;
        } catch (IOException e) {
            metrics.increment("http.request.error");
            throw e;
        }
    }
}

public class CachingDecorator extends HttpClientDecorator {
    private final Map<String, CachedResponse> cache = new ConcurrentHashMap<>();
    private final long cacheTtlMillis;

    public CachingDecorator(HttpClient client, long cacheTtlMillis) {
        super(client);
        this.cacheTtlMillis = cacheTtlMillis;
    }

    @Override
    public Response get(String url) throws IOException {
        CachedResponse cached = cache.get(url);

        if (cached != null && !cached.isExpired()) {
            return cached.getResponse();
        }

        Response response = client.get(url);
        cache.put(url, new CachedResponse(response, System.currentTimeMillis() + cacheTtlMillis));

        return response;
    }

    private static class CachedResponse {
        private final Response response;
        private final long expiryTime;

        CachedResponse(Response response, long expiryTime) {
            this.response = response;
            this.expiryTime = expiryTime;
        }

        boolean isExpired() {
            return System.currentTimeMillis() > expiryTime;
        }

        Response getResponse() {
            return response;
        }
    }
}

// Usage - compose features as needed!
public class HttpClientFactory {

    // Production: Full stack
    public static HttpClient createProductionClient() {
        HttpClient client = new BasicHttpClient();
        client = new RetryDecorator(client, 3);
        client = new LoggingDecorator(client);
        client = new MetricsDecorator(client, new MetricsCollector());
        client = new CachingDecorator(client, 600_000); // 10 min cache
        return client;
    }

    // Development: Just logging
    public static HttpClient createDevClient() {
        return new LoggingDecorator(new BasicHttpClient());
    }

    // High-traffic endpoint: Cache + metrics only
    public static HttpClient createFastClient() {
        HttpClient client = new BasicHttpClient();
        client = new CachingDecorator(client, 300_000);
        client = new MetricsDecorator(client, new MetricsCollector());
        return client;
    }
}

// Example usage
public class ApiService {
    public void fetchData() throws IOException {
        HttpClient client = HttpClientFactory.createProductionClient();
        Response response = client.get("https://api.example.com/data");

        System.out.println("Response: " + response.body().string());
    }
}
```

**Real-World Impact:**
- ✅ Added circuit breaker in **1 hour** without touching existing code
- ✅ Enable/disable features per environment (dev vs prod)
- ✅ Each decorator = single responsibility, easy to test
- ✅ Mix and match features: `Retry + Cache + Metrics`

| Pattern     | Problem Solved                | Backend Example                  | Use When |
|-------------|-------------------------------|----------------------------------|----------|
| Adapter    | Incompatible interfaces      | Wrap legacy DB to match new ORM interface | Integrating third-party libraries or legacy systems |
| Decorator  | Extend without subclassing   | Add retry/logging/caching to HTTP client | Need to add features dynamically at runtime |
| Facade     | Simplify subsystems          | Unified API for Azure services (Storage, Queue, Cache) | Complex system with many moving parts |
| Proxy      | Control access to objects    | Lazy-load database connections or add access control | Need lazy initialization, caching, or access control |
| Composite  | Tree structures              | Organization hierarchy, file system structure | Part-whole hierarchies (menus, trees) |

## Behavioral Patterns

These manage object interactions and responsibilities, promoting loose coupling in algorithms and communication.

- They shine in event-driven or pluggable systems, distributing behavior across objects.
- Key examples: **Observer** (pub/sub for status changes), **Strategy** (swap sorting/routing algos), **Command** (queueable async tasks), **Iterator** (traverse query results), **State** (workflow state machines).
- Real-world example: Observer notifies multiple services when order status changes via Event Bus, decoupling order service from inventory, shipping, and notification services.

### Real Code Example: Strategy Pattern

**❌ Without Strategy (Rigid Conditional Logic):**
```java
// Pricing service with hardcoded rules
public class PricingService {

    public BigDecimal calculatePrice(Order order, String customerType) {
        BigDecimal basePrice = order.getSubtotal();

        // Nightmare of if-else statements
        if ("regular".equals(customerType)) {
            if (basePrice.compareTo(new BigDecimal("100")) > 0) {
                return basePrice.multiply(new BigDecimal("0.95")); // 5% discount
            } else {
                return basePrice;
            }
        } 
        else if ("premium".equals(customerType)) {
            if (basePrice.compareTo(new BigDecimal("50")) > 0) {
                return basePrice.multiply(new BigDecimal("0.90")); // 10% discount
            } else {
                return basePrice.multiply(new BigDecimal("0.95"));
            }
        } 
        else if ("enterprise".equals(customerType)) {
            if (order.getItemsCount() > 100) {
                return basePrice.multiply(new BigDecimal("0.75")); // 25% discount
            } else if (basePrice.compareTo(new BigDecimal("1000")) > 0) {
                return basePrice.multiply(new BigDecimal("0.80"));
            } else {
                return basePrice.multiply(new BigDecimal("0.85"));
            }
        } 
        else if ("black_friday".equals(customerType)) {
            return basePrice.multiply(new BigDecimal("0.70"));
        } 
        else {
            return basePrice;
        }
    }
}

// Problems:
// - Adding new customer type = modify this method
// - Can't test individual pricing strategies
// - Can't enable/disable strategies at runtime
// - Violates Open/Closed Principle
```

**✅ With Strategy Pattern (Flexible & Testable):**
```java
// Strategy interface
public interface PricingStrategy {
    BigDecimal calculatePrice(Order order);
}

// Concrete strategies - each encapsulates ONE algorithm
public class RegularCustomerPricing implements PricingStrategy {
    @Override
    public BigDecimal calculatePrice(Order order) {
        BigDecimal subtotal = order.getSubtotal();
        if (subtotal.compareTo(new BigDecimal("100")) > 0) {
            return subtotal.multiply(new BigDecimal("0.95"));
        }
        return subtotal;
    }
}

public class PremiumCustomerPricing implements PricingStrategy {
    @Override
    public BigDecimal calculatePrice(Order order) {
        BigDecimal subtotal = order.getSubtotal();
        if (subtotal.compareTo(new BigDecimal("50")) > 0) {
            return subtotal.multiply(new BigDecimal("0.90"));
        }
        return subtotal.multiply(new BigDecimal("0.95"));
    }
}

public class EnterpriseCustomerPricing implements PricingStrategy {
    @Override
    public BigDecimal calculatePrice(Order order) {
        BigDecimal subtotal = order.getSubtotal();

        if (order.getItemsCount() > 100) {
            return subtotal.multiply(new BigDecimal("0.75"));
        } else if (subtotal.compareTo(new BigDecimal("1000")) > 0) {
            return subtotal.multiply(new BigDecimal("0.80"));
        }
        return subtotal.multiply(new BigDecimal("0.85"));
    }
}

public class BlackFridayPricing implements PricingStrategy {
    @Override
    public BigDecimal calculatePrice(Order order) {
        return order.getSubtotal().multiply(new BigDecimal("0.70"));
    }
}

public class NewCustomerPricing implements PricingStrategy {
    /** Added in 10 minutes - zero changes to existing code! */
    @Override
    public BigDecimal calculatePrice(Order order) {
        // First order gets 20% off
        return order.getSubtotal().multiply(new BigDecimal("0.80"));
    }
}

// Context that uses strategies
public class PricingService {
    private final Map<String, PricingStrategy> strategies = new HashMap<>();

    public PricingService() {
        // Strategy registry - can be loaded from database or config
        strategies.put("regular", new RegularCustomerPricing());
        strategies.put("premium", new PremiumCustomerPricing());
        strategies.put("enterprise", new EnterpriseCustomerPricing());
        strategies.put("black_friday", new BlackFridayPricing());
        strategies.put("new_customer", new NewCustomerPricing());
    }

    public BigDecimal calculatePrice(Order order, String customerType) {
        PricingStrategy strategy = strategies.getOrDefault(
            customerType, 
            new RegularCustomerPricing()  // Default strategy
        );

        return strategy.calculatePrice(order);
    }

    public void addStrategy(String name, PricingStrategy strategy) {
        /** Add new pricing strategy at runtime */
        strategies.put(name, strategy);
    }
}

// Usage
public class CheckoutService {
    private final PricingService pricingService;
    private final FeatureFlagService featureFlags;

    public CheckoutService(PricingService pricingService, FeatureFlagService featureFlags) {
        this.pricingService = pricingService;
        this.featureFlags = featureFlags;
    }

    public BigDecimal checkout(Order order, Customer customer) {
        // Calculate price using strategy
        BigDecimal price = pricingService.calculatePrice(order, customer.getType());

        // A/B Testing: Swap strategies dynamically
        if (featureFlags.isEnabled("new_pricing_v2")) {
            pricingService.addStrategy("premium", new PremiumCustomerPricingV2());
        }

        return price;
    }
}

// Testing individual strategies
@Test
public void testPremiumPricing() {
    PricingStrategy strategy = new PremiumCustomerPricing();
    Order order = new Order(new BigDecimal("75"));

    BigDecimal price = strategy.calculatePrice(order);

    assertEquals(new BigDecimal("67.50"), price);
}
```

**Real-World Impact:**
- ✅ Added new pricing tiers in **minutes** instead of hours
- ✅ A/B test pricing algorithms without deployment
- ✅ Each strategy independently testable
- ✅ Business analysts can configure strategies in database

| Pattern     | Problem Solved                   | Backend Example                | Use When |
|-------------|----------------------------------|--------------------------------|----------|
| Observer   | One-to-many dependencies        | Event notifications (Order placed → Inventory, Shipping, Email) | Objects need to notify multiple listeners of changes |
| Strategy   | Interchangeable algorithms      | Pricing rules per customer tier, compression algorithms | Need to swap algorithms at runtime |
| Command    | Encapsulate requests            | Retryable job queue, undo/redo operations | Need to queue operations, support undo, or log requests |
| State      | Object behavior changes with state | Order workflow (Pending → Paid → Shipped → Delivered) | Object behavior varies based on internal state |
| Template Method | Define algorithm skeleton | Data processing pipeline with customizable steps | Algorithm structure is fixed, but steps vary |

## How Patterns Work Together

Real-world patterns are rarely used in isolation. Here's how they compose in an actual Azure microservice:

### Example: Payment Processing Microservice Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Payment Service                          │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Config Manager (Singleton)                                  │
│      ↓ loads payment provider config                         │
│                                                               │
│  Payment Processor Factory (Factory)                         │
│      ↓ creates StripeProcessor or PayPalProcessor           │
│                                                               │
│  HTTP Client (Decorator Stack)                               │
│      • RetryDecorator                                        │
│      • LoggingDecorator                                      │
│      • MetricsDecorator                                      │
│      • CachingDecorator                                      │
│      ↓ wrapped around BasicHttpClient                        │
│                                                               │
│  External Payment API (Adapter)                              │
│      ↓ converts Stripe/PayPal API to common interface       │
│                                                               │
│  Pricing Calculator (Strategy)                               │
│      ↓ applies customer-specific pricing rules              │
│                                                               │
│  Event Publisher (Observer)                                  │
│      → notifies Order Service                                │
│      → notifies Inventory Service                            │
│      → notifies Notification Service                         │
│      → notifies Analytics Service                            │
└─────────────────────────────────────────────────────────────┘
```

### Code Implementation:

```java
// 1. Singleton - Config Manager
public class ConfigManager {
    private static ConfigManager instance;
    private String paymentProvider;
    private int retryCount;

    private ConfigManager() {
        loadConfig();
    }

    public static synchronized ConfigManager getInstance() {
        if (instance == null) {
            instance = new ConfigManager();
        }
        return instance;
    }

    private void loadConfig() {
        this.paymentProvider = System.getenv().getOrDefault("PAYMENT_PROVIDER", "stripe");
        this.retryCount = Integer.parseInt(System.getenv().getOrDefault("RETRY_COUNT", "3"));
    }

    public String getPaymentProvider() {
        return paymentProvider;
    }

    public int getRetryCount() {
        return retryCount;
    }
}

// 2. Factory - Create payment processors
ConfigManager config = ConfigManager.getInstance();
PaymentProcessor processor = PaymentProcessorFactory.createProcessor(
    config.getPaymentProvider()
);

// 3. Decorator - Build resilient HTTP client
HttpClient httpClient = new BasicHttpClient();
httpClient = new RetryDecorator(httpClient, config.getRetryCount());
httpClient = new LoggingDecorator(httpClient);
httpClient = new MetricsDecorator(httpClient, new MetricsCollector());

// 4. Adapter - Wrap external API
PaymentAPIAdapter paymentAdapter = new PaymentAPIAdapter(processor, httpClient);

// 5. Strategy - Calculate price based on customer type
PricingService pricingService = new PricingService();
BigDecimal price = pricingService.calculatePrice(order, customer.getType());

// 6. Observer - Notify interested services
EventBus eventBus = new EventBus();
eventBus.subscribe("payment.completed", new OrderService());
eventBus.subscribe("payment.completed", new InventoryService());
eventBus.subscribe("payment.completed", new NotificationService());
eventBus.publish("payment.completed", paymentData);
```

**Benefits of This Architecture:**
- ✅ Config changes don't require code changes (Singleton)
- ✅ Add new payment providers in minutes (Factory)
- ✅ Add/remove features dynamically (Decorator)
- ✅ Integration changes don't affect business logic (Adapter)
- ✅ Business rules change independently (Strategy)
- ✅ Services loosely coupled (Observer)

## Common Pitfalls ⚠️

### 1. Don't Overuse Singleton
**❌ Bad:** Making every service a Singleton creates hidden dependencies and testing nightmares.
```java
// BAD: Everything is Singleton
DatabaseService.getInstance();  // Can't mock in tests!
CacheService.getInstance();     // Global state hell
EmailService.getInstance();     // Tightly coupled
ConfigService.getInstance();    // Thread safety issues
```

**✅ Good:** Use dependency injection instead:
```java
// GOOD: Inject dependencies
public class OrderService {
    private final DatabaseService db;
    private final CacheService cache;
    private final EmailService email;

    public OrderService(DatabaseService db, CacheService cache, EmailService email) {
        this.db = db;
        this.cache = cache;
        this.email = email;
    }

    public void createOrder(Order order) {
        db.save(order);
        cache.put(order.getId(), order);
        email.sendConfirmation(order);
    }
}

// Easy to test with mocks!
@Test
public void testOrderService() {
    DatabaseService mockDb = Mockito.mock(DatabaseService.class);
    CacheService mockCache = Mockito.mock(CacheService.class);
    EmailService mockEmail = Mockito.mock(EmailService.class);

    OrderService service = new OrderService(mockDb, mockCache, mockEmail);

    Order order = new Order("12345", new BigDecimal("100"));
    service.createOrder(order);

    verify(mockDb).save(order);
    verify(mockCache).put(order.getId(), order);
    verify(mockEmail).sendConfirmation(order);
}
```

**When Singleton IS appropriate:**
- Configuration managers
- Logger instances
- Thread pools
- Database connection pools

### 2. Don't Over-Decorate
**❌ Bad:** 10+ decorator layers = debugging nightmare
```java
// BAD: Too many layers
HttpClient client = new BasicHttpClient();
client = new RetryDecorator(client);
client = new LoggingDecorator(client);
client = new MetricsDecorator(client, metrics);
client = new CachingDecorator(client, 300000);
client = new AuthDecorator(client, authToken);
client = new RateLimitDecorator(client, 100);
client = new CircuitBreakerDecorator(client);
client = new TimeoutDecorator(client, 5000);
client = new CompressionDecorator(client);
client = new ValidationDecorator(client);  // Where's the actual HTTP call?!
```

**✅ Good:** Group related concerns:
```java
// GOOD: Compose strategically
public class ResilientHttpClient implements HttpClient {
    private final HttpClient client;

    /** Pre-configured client with common production features */
    public ResilientHttpClient() {
        HttpClient baseClient = new BasicHttpClient();
        baseClient = new RetryDecorator(baseClient, 3);
        baseClient = new CircuitBreakerDecorator(baseClient);
        baseClient = new MetricsDecorator(baseClient, new MetricsCollector());
        this.client = baseClient;
    }

    @Override
    public Response get(String url) throws IOException {
        return client.get(url);
    }
}
```

### 3. Don't Force Patterns Just Because
**❌ Bad:** "Let me use Visitor pattern because it sounds cool"

**✅ Good:** Use patterns to solve actual problems:
- Code is hard to change? → Consider patterns
- Code is simple and working? → Don't add complexity
- Follow the rule: **YAGNI** (You Aren't Gonna Need It)

### 4. Don't Mix Responsibilities
**❌ Bad:** Factory that also does validation, logging, and caching
```java
// BAD: God factory
public class PaymentProcessorFactory {
    private static final Logger logger = LoggerFactory.getLogger(PaymentProcessorFactory.class);
    private static final Map<String, PaymentProcessor> cache = new HashMap<>();

    public static PaymentProcessor createProcessor(String provider) {
        // Logging
        logger.info("Creating processor for: " + provider);

        // Validation
        if (!isValidProvider(provider)) {
            throw new IllegalArgumentException("Invalid provider");
        }

        // Caching
        if (cache.containsKey(provider)) {
            return cache.get(provider);
        }

        // Metrics
        MetricsCollector.increment("processor.created");

        // Actual creation (buried in noise!)
        PaymentProcessor processor;
        if ("stripe".equals(provider)) {
            processor = new StripeProcessor();
        } else {
            processor = new PayPalProcessor();
        }

        cache.put(provider, processor);
        return processor;
    }
}
```

**✅ Good:** Single Responsibility Principle
```java
// GOOD: Factory does ONE thing - creates objects
public class PaymentProcessorFactory {

    public static PaymentProcessor createProcessor(String provider) {
        switch (provider.toLowerCase()) {
            case "stripe":
                return new StripeProcessor();
            case "paypal":
                return new PayPalProcessor();
            default:
                throw new IllegalArgumentException("Unknown provider: " + provider);
        }
    }
}

// Separate concerns into dedicated classes
public class CachedPaymentProcessorFactory {
    private final Map<String, PaymentProcessor> cache = new ConcurrentHashMap<>();
    private final PaymentProcessorFactory factory = new PaymentProcessorFactory();

    public PaymentProcessor createProcessor(String provider) {
        return cache.computeIfAbsent(provider, factory::createProcessor);
    }
}
```

## Complete List of Design Patterns

### 1. Creational Design Patterns
Focus on flexible object creation mechanisms.

- **Factory Pattern** - Create objects based on conditions
- **Abstract Factory Pattern** - Create families of related objects
- **Singleton Pattern** - Ensure single instance exists
- **Prototype Pattern** - Clone existing objects
- **Builder Pattern** - Construct complex objects step-by-step
- **Object Pool Pattern** - Reuse expensive-to-create objects

### 2. Structural Design Patterns
Deal with object composition and relationships.

- **Adapter Pattern** - Convert interface to another interface
- **Bridge Pattern** - Separate abstraction from implementation
- **Composite Pattern** - Treat individual objects and compositions uniformly
- **Decorator Pattern** - Add responsibilities dynamically
- **Facade Pattern** - Provide unified interface to subsystem
- **Flyweight Pattern** - Share common state to save memory
- **Proxy Pattern** - Control access to objects

### 3. Behavioral Design Patterns
Manage algorithms and object responsibilities.

- **Chain of Responsibility Pattern** - Pass requests along handler chain
- **Command Pattern** - Encapsulate requests as objects
- **Interpreter Pattern** - Define grammar and interpret sentences
- **Iterator Pattern** - Access elements sequentially
- **Mediator Pattern** - Define object interaction in a mediator
- **Memento Pattern** - Capture and restore object state
- **Observer Pattern** - Notify dependents of state changes
- **State Pattern** - Change behavior based on internal state
- **Strategy Pattern** - Define interchangeable algorithms
- **Template Method Pattern** - Define algorithm skeleton, defer steps
- **Visitor Pattern** - Add operations without modifying classes

## Progressive Learning Path

Don't try to learn all patterns at once. Follow this proven path:

### 🎯 Week 1-2: Essential Patterns (80% of use cases)

**Start with these three - they solve most real-world problems:**

1. **Singleton** - Configuration managers, loggers, connection pools
   - Practice: Refactor your config loading to use Singleton
   - Warning: Don't make everything a Singleton!

2. **Factory** - Plugin systems, multi-tenant features, provider selection
   - Practice: Convert your if-else object creation to Factory
   - Real example: Payment processors, database clients, cloud providers

3. **Strategy** - Interchangeable algorithms, business rules
   - Practice: Extract your conditional logic into Strategy classes
   - Real example: Pricing rules, sorting algorithms, validation rules

### 📚 Week 3-4: Add These When Needed

4. **Builder** - Complex object construction with many parameters
   - Use when: Constructor has >3 parameters or many optional configs
   - Practice: Build an HTTP request builder or query builder

5. **Decorator** - Add features without modifying existing code
   - Use when: Want to add cross-cutting concerns (logging, retry, caching)
   - Practice: Wrap your HTTP client with retry + logging decorators

6. **Observer** - Event-driven systems, pub/sub messaging
   - Use when: Multiple objects need to react to changes
   - Practice: Implement an event bus for microservice communication

### 🚀 Week 5-6: Advanced Patterns

7. **Abstract Factory** - Families of related objects
   - Use when: Need consistent object families (AWS SDK vs Azure SDK)

8. **Adapter** - Integrate third-party or legacy systems
   - Use when: External APIs don't match your interface

9. **Facade** - Simplify complex subsystems
   - Use when: Multiple APIs to accomplish one business task

10. **Command** - Queueable operations, undo/redo
    - Use when: Need transactional operations or job queues

### 📝 Weekly Practice Plan

**Monday:** Read pattern documentation + examples  
**Tuesday-Wednesday:** Identify pattern opportunities in your codebase  
**Thursday-Friday:** Refactor one class using the pattern  
**Weekend:** Write blog post or teach teammate what you learned

## Interview Success Formula

### ❌ Don't Just List Patterns
**Bad Answer:** "I know Singleton, Factory, Observer, Strategy, Decorator, Adapter..."

**Why it's bad:** Shows memorization, not understanding or experience.

### ✅ Tell a Problem-Solution-Result Story
**Good Answer:** 
"In our e-commerce payment system, we initially hardcoded Stripe integration throughout 50+ files. When the business wanted to add PayPal support, we faced a 2-week refactoring project.

I proposed the Factory pattern. We created a `PaymentProcessor` interface and concrete implementations for each provider. The Factory selected the right processor based on configuration.

**Result:** Adding PayPal took 2 hours instead of 2 weeks. We later added Apple Pay in 30 minutes. The pattern also enabled A/B testing payment providers by just changing a config flag, which increased conversion by 3%.

**Trade-off:** We added one abstraction layer, but it was worth it for the flexibility. We also had to educate the team on the pattern, but now they apply it elsewhere."

### 🎯 Interview Story Template

Use this structure for any pattern discussion:

1. **Context:** "We had [specific problem] in [system/component]"
2. **Pattern Applied:** "We used [pattern name] because [reason]"
3. **Implementation:** "We created [interfaces/classes] that [how it works]"
4. **Result:** "[Quantifiable improvement] - time saved, bugs reduced, flexibility gained"
5. **Trade-offs:** "We accepted [cost] for [benefit] because [business reason]"

### 📊 Pattern Comparison Questions

**Q: "When would you use Factory vs Abstract Factory?"**

**Good Answer:**
- "Use **Factory** when you need to create one type of object based on conditions. Example: Creating different payment processors (Stripe, PayPal, Apple Pay) based on user preference.

- Use **Abstract Factory** when you need families of related objects. Example: Creating cloud provider SDKs where each provider (AWS, Azure, GCP) needs a family of clients (storage, compute, database) that work together.

  In our system, we used Factory for payment processors because we only needed one object type. If we were building a multi-cloud platform, Abstract Factory would be better."

### 🔥 Advanced Interview Topics

**Q: "What are the downsides of design patterns?"**

**Good Answer:**
- "Design patterns add abstraction layers, which can increase complexity if overused
- They require team understanding - not everyone knows all patterns
- Premature pattern application can over-engineer simple problems
- Some patterns (like Singleton) can make testing harder
- They're tools, not rules - force-fitting patterns hurts maintainability

Example: In a startup MVP, we skipped patterns initially to ship fast. Once we had product-market fit and started scaling, we refactored using patterns. Timing matters."

## Why Categorize Patterns?

Understanding these three groups helps you pick the right tool fast:

- **Creational** patterns → Solve object instantiation headaches
- **Structural** patterns → Provide architecture glue between components  
- **Behavioral** patterns → Manage dynamic behavior and communication

**Mental Model:**
- Building objects is complex? → **Creational**
- Connecting incompatible parts? → **Structural**
- Objects need to communicate? → **Behavioral**

In interviews, categorization shows deeper understanding: *"This is a creational problem because we need flexible instantiation. Factory pattern fits because we're selecting object types at runtime."*

## Further Reading & Resources

### 📚 Essential Books
- **[Design Patterns: Elements of Reusable Object-Oriented Software](https://en.wikipedia.org/wiki/Design_Patterns)** (Gang of Four) - The original classic
- **Head First Design Patterns** - Visual, easy-to-digest format
- **Refactoring to Patterns** - When and how to apply patterns

### 🌐 Online Resources
- **[Refactoring Guru](https://refactoring.guru/design-patterns)** - Visual diagrams and examples in multiple languages
- **[SourceMaking](https://sourcemaking.com/design-patterns)** - Clear explanations with code samples
- **[Azure Architecture Patterns](https://docs.microsoft.com/en-us/azure/architecture/patterns/)** - Cloud-specific design patterns

### 💡 Practice Projects
1. Build a **multi-payment gateway service** (Factory + Strategy + Adapter)
2. Create an **HTTP client library** (Decorator + Singleton + Builder)
3. Design an **event-driven order system** (Observer + Command + State)
4. Implement a **cloud storage abstraction** (Abstract Factory + Facade + Proxy)

### 🎯 Next Steps
**Week 1:** Pick ONE pattern from "Essential Patterns"  
**Week 2:** Identify where it applies in your current project  
**Week 3:** Refactor one component using the pattern  
**Week 4:** Document your learnings and share with team  
**Repeat:** Move to next pattern

---

**Remember:** Patterns are tools, not goals. Use them to solve real problems, not to show off knowledge. The best code is often the simplest code that solves the problem effectively.

**Start applying one pattern per week in your code.** 🚀
