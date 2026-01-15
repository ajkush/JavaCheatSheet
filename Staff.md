I reformatted the sheet as a clean Markdown file you can copy into your repo. Below is the full file content.

````markdown name=Java_Interview_Cheat_Sheet.md url=https://github.com/ajkush/JavaCheatSheet/blob/8f8074890a5863e9c75eb02d49f1568ec41ee6a0/Staff
# 🎓 Java Backend/Staff Engineer Interview Cheat Sheet (2026)

## 1. STAFF-LEVEL SYSTEM DESIGN PATTERNS

### High-Concurrency Architecture

```java
// Rate Limiting Patterns
Token Bucket: new RateLimiter(permitsPerSecond, warmupPeriod)
Sliding Window Log: Redis Sorted Set + ZREMRANGEBYSCORE
Fixed Window Counter: Redis INCR with EXPIRE

// Circuit Breaker Implementation
@CircuitBreaker(failureThreshold=5, timeout=1000)
public Response callService() { ... }

// Bulkheading Pattern
ExecutorService pool = Executors.newFixedThreadPool(10);
// Separate pools for different service calls
```

### Distributed Systems Essentials
- **CAP Theorem**: Choose 2 of 3 (Consistency, Availability, Partition Tolerance)
- **PACELC**: Extension for latency tradeoffs
- **Consistency Models**: Strong → Sequential → Causal → Eventual

## 2. JVM DEEP DIVE FOR STAFF ENGINEERS

### Memory Optimization Patterns

```java
// Off-Heap Memory (Panama API)
try (Arena arena = Arena.ofConfined()) {
    MemorySegment segment = arena.allocate(1024);
    // Direct memory access without GC pressure
}

// Object Pooling (for expensive objects)
@ResourcePool(size=100, evictionTime=300)
public class ConnectionPool { ... }

// Value Objects (Valhalla Preview)
inline class UserId {
    private final long id;
    // Stored on stack, no heap allocation
}
```

### Garbage Collection Tuning (Production)

```properties
# G1 GC for predictable latency
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:InitiatingHeapOccupancyPercent=45
-XX:G1ReservePercent=10

# ZGC for ultra-low latency
-XX:+UseZGC
-XX:ZAllocationSpikeTolerance=2.0
-XX:ZCollectionInterval=120
-XX:ZUncommitDelay=300

# Shenandoah for balanced throughput
-XX:+UseShenandoahGC
-XX:ShenandoahGuaranteedGCInterval=10000
```

### Container-Aware JVM (Kubernetes)

```bash
# Java 17+ automatically respects container limits
java -XX:+UseContainerSupport \
     -XX:MaxRAMPercentage=75.0 \
     -XX:InitialRAMPercentage=50.0 \
     -XX:MinRAMPercentage=25.0 \
     -jar app.jar

# Critical: Always set Xmx/Xms in containers!
# Wrong: java -jar app.jar
# Right: java -Xmx512m -Xms256m -jar app.jar
```

## 3. PERFORMANCE & SCALABILITY PATTERNS

### Virtual Threads Production Patterns

```java
// Structured Concurrency (Java 21+)
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    Future<User> user = scope.fork(() -> fetchUser(id));
    Future<Order> order = scope.fork(() -> fetchOrder(id));
    
    scope.join();           // Join both forks
    scope.throwIfFailed();  // Propagate exceptions
    
    return new Response(user.resultNow(), order.resultNow());
}

// Virtual Thread Pools (NEVER use cached thread pools!)
ExecutorService vThreadExecutor = Executors.newVirtualThreadPerTaskExecutor();
// Or use limited pool for resource control
ExecutorService limitedVThreadExecutor = 
    Executors.newThreadPerTaskExecutor(Thread.ofVirtual().factory());
```

### Reactive vs Virtual Threads Decision Matrix

Choose Virtual Threads When:
- Blocking I/O operations (DB, HTTP calls)
- Traditional thread-per-request model
- Debuggability and stack traces important
- Migrating from synchronous codebase

Choose Reactive When:
- Streaming data (WebSocket, SSE)
- Backpressure handling required
- Non-blocking I/O already implemented
- High-throughput message processing

Hybrid Approach:

```java
@RestController
public class UserController {
    @VirtualThread  // Spring Boot 3.2+
    public CompletableFuture<User> getUser() {
        return CompletableFuture.supplyAsync(() -> blockingCall());
    }
}
```

### Caching Strategies (Multi-Level)

```java
// L1: Caffeine (In-Process)
Cache<Key, Value> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .recordStats()
    .build();

// L2: Redis (Distributed)
@Cacheable(value="users", cacheManager="redisCacheManager")
public User getUser(Long id) { ... }

// L3: Database (Source of Truth)
// With Cache-Aside Pattern
public User getUserWithCache(Long id) {
    User user = cache.get(id);
    if (user == null) {
        user = database.get(id);
        cache.put(id, user);
    }
    return user;
}
```

## 4. MICROSERVICES & CLOUD-NATIVE PATTERNS

### Service Communication Patterns

```java
// Synchronous (REST/gRPC)
@FeignClient(name="order-service")
public interface OrderClient {
    @GetMapping("/orders/{id}")
    Order getOrder(@PathVariable Long id);
}

// Asynchronous (Kafka/Event-Driven)
@KafkaListener(topics="order-events")
public void handleOrderEvent(OrderEvent event) {
    eventProcessor.process(event);
}

// Saga Pattern for Distributed Transactions
@Saga
public class OrderSaga {
    @StartSaga
    @SagaEventHandler(associationProperty="orderId")
    public void handle(OrderCreated event) { ... }
    
    @SagaEventHandler(associationProperty="orderId")
    public void handle(PaymentCompleted event) { ... }
}
```

### Resilience Patterns (Spring Cloud Circuit Breaker)

```java
@RestController
@CircuitBreaker(name="inventoryService", fallbackMethod="fallback")
@RateLimiter(name="inventoryService")
@Retry(name="inventoryService", fallbackMethod="fallback")
@Bulkhead(name="inventoryService", type=Bulkhead.Type.THREADPOOL)
public class InventoryController {
    
    @GetMapping("/inventory/{id}")
    public Inventory getInventory(@PathVariable String id) {
        return inventoryService.getInventory(id);
    }
    
    public Inventory fallback(String id, Throwable t) {
        return Inventory.cached(id);
    }
}
```

### Observability Stack (Staff-Level)

```yaml
# Micrometer Configuration
management:
  metrics:
    export:
      prometheus:
        enabled: true
  tracing:
    sampling:
      probability: 0.1
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus

# Custom Metrics
@Timed(value="api.call", description="API call duration")
@Counted(value="api.requests", description="API request count")
@Metered
public Response apiCall() { ... }

# Distributed Tracing
@NewSpan
public void processOrder(Order order) {
    // Automatically traced
}
```

## 5. DATABASE PATTERNS & OPTIMIZATION

### Connection Pool Tuning (HikariCP)

```properties
spring.datasource.hikari:
  maximumPoolSize: 20
  minimumIdle: 10
  connectionTimeout: 30000
  idleTimeout: 600000
  maxLifetime: 1800000
  connectionTestQuery: SELECT 1
  poolName: MainPool
  
# Formula: connections = (core_count * 2) + effective_spindle_count
# For 16-core with SSD: (16 * 2) + 1 = 33 connections
```

### JPA/Hibernate Performance

```java
// N+1 Problem Solutions
@EntityGraph(attributePaths = {"orders", "orders.items"})
@Query("SELECT u FROM User u WHERE u.id = :id")
User findWithOrders(@Param("id") Long id);

// Batch Processing
spring.jpa.properties.hibernate.jdbc.batch_size=50
spring.jpa.properties.hibernate.order_inserts=true
spring.jpa.properties.hibernate.order_updates=true

// Read-Write Splitting
@Configuration
@EnableTransactionManagement
public class DatabaseConfig {
    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.master")
    public DataSource masterDataSource() { ... }
    
    @Bean
    @ConfigurationProperties("spring.datasource.replica")
    public DataSource replicaDataSource() { ... }
}
```

### Time-Series Data Pattern

```java
// Time-based partitioning
@Table(name = "metrics")
@Partition(valueBy = "{ 'ts': 1 }", partitionBy = "MONTH")
public class Metric {
    @Id
    private String id;
    private Instant ts;
    private double value;
    
    // Index strategy
    @Index(name = "idx_ts_type", def = "{'ts': 1, 'type': 1}")
    private String type;
}
```

## 6. STAFF-LEVEL INTERVIEW QUESTIONS & ANSWERS

### System Design Questions

Q: Design a distributed rate limiter for 1M requests/second.
```
A: "Use Redis Cluster with token bucket algorithm, 
   sliding window counters in Lua scripts for atomicity,
   local caching with caffeine for 95% hit rate,
   and fallback to degraded mode during Redis failures."
```

Q: How would you migrate from monolith to microservices?
```
A: "Strangler Fig pattern: Identify bounded contexts,
   create anti-corruption layers, extract modules horizontally,
   implement feature toggles, use canary deployments,
   and maintain backward compatibility for 2 release cycles."
```

Q: Design a real-time inventory system.
```
A: "CQRS pattern: Write to PostgreSQL with optimistic locking,
   stream changes via CDC to Kafka, materialize views in Redis,
   serve reads from Redis with cache warming,
   and use CRDTs for conflict resolution in distributed updates."
```

### JVM Deep Dive Questions

Q: How does ZGC achieve sub-millisecond pauses?
```
A: "ZGC uses colored pointers with load barriers,
   concurrent marking and relocation,
   region-based heap layout,
   and NUMA-aware allocation.
   No stop-the-world phases except root scanning."
```

Q: Explain memory ordering in Java's memory model.
```
A: "Happens-before relationships guarantee:
   Program order, volatile writes, monitor locks,
   thread start/join, and final fields.
   Use VarHandle with acquire/release semantics for low-level control."
```

Q: How do virtual threads reduce memory consumption?
```
A: "Virtual threads use continuation objects (~400 bytes)
   instead of OS thread stacks (1-2MB).
   They're multiplexed onto carrier threads,
   with stack chunking for deep recursion."
```

### Performance Optimization Questions

Q: Your service has 99th percentile latency spikes.
```
A: "1. Profile with Async Profiler/Java Flight Recorder
   2. Check GC logs for stop-the-world pauses
   3. Review thread dumps for contention
   4. Examine network/DB connection pools
   5. Implement request queuing with virtual threads"
```

Q: How would you optimize a batch processing job?
```
A: "1. Use parallel streams with custom ForkJoinPool
   2. Implement chunk processing with savepoints
   3. Use batching for database operations
   4. Off-heap memory for large datasets
   5. Consider ETL framework (Spring Batch) with partitioning"
```

Q: Optimize Spring Boot startup time.
```
A: "1. Use Spring Boot 3+ with AOT compilation
   2. Implement lazy initialization
   3. Reduce classpath scanning with explicit @ComponentScan
   4. Use compile-time DI (Micronaut/Quarkus for critical paths)
   5. Profile with Spring Boot Actuator startup endpoint"
```

## 7. PRODUCTION READINESS CHECKLIST

### Health Checks & Probes

```yaml
# Kubernetes Configuration
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 60
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 5

startupProbe:
  httpGet:
    path: /actuator/health/startup
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

### Monitoring Alerts (Staff Level)

```yaml
Critical Alerts:
  - GC Pause > 1s for 5 minutes
  - Memory > 85% for 10 minutes
  - CPU > 90% for 5 minutes
  - Error rate > 1% for 2 minutes
  - 99th percentile latency > 500ms

Business Alerts:
  - Transaction volume drop > 50%
  - Revenue per minute < threshold
  - User signup failure rate > 5%
```

### Disaster Recovery Patterns

```java
// Circuit Breaker with Exponential Backoff
@CircuitBreaker(
    failureThreshold = 5,
    delay = 1000,
    maxDelay = 30000,
    multiplier = 2.0
)

// Bulkhead Pattern for Resource Isolation
@Bulkhead(
    value = 10, 
    waitingTaskQueue = 100,
    type = Bulkhead.Type.THREADPOOL
)

// Retry with Jitter
@Retry(
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, maxDelay = 1000, multiplier = 2, random = true)
)
```

## 8. QUICK REFERENCE - ONE LINERS

### Architecture Decisions
- "Event-driven over request-response for decoupling"
- "Eventual consistency when strong consistency isn't required"
- "CQRS when read/write patterns differ significantly"
- "Saga over 2PC for long-running transactions"
- "API Gateway for cross-cutting concerns"

### Performance Optimization
- "Profile before optimizing - use JFR/Async Profiler"
- "Measure, don't guess - metrics for every decision"
- "Cache at multiple levels - L1, L2, L3"
- "Batch small operations - reduce round trips"
- "Precompute expensive operations - materialized views"

### Team Leadership (Staff Engineer)
- "Document architecture decisions (ADRs)"
- "Define clear service boundaries (DDD)"
- "Establish coding standards and patterns"
- "Implement gradual rollouts with feature flags"
- "Build for observability from day one"

---

**Last Updated:** March 2026  
**Target Roles:** Senior/Staff/Principal Backend Engineer  
**Focus:** System Design, Performance, Cloud-Native Java  

Copy this entire text and paste it into your GitHub repository as a `.md` file (e.g., `Java_Interview_Cheat_Sheet.md`). GitHub will render it beautifully with syntax highlighting and proper formatting.
````
