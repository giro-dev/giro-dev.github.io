---
title: "Observability for virtual threads: tracing a Spring Boot 3 request with Micrometer and OpenTelemetry"
date: 2026-08-24T00:00:00Z
description: "A hands-on tutorial on keeping traces, logs and metrics intact after switching a Spring Boot service to virtual threads: Micrometer Observation, OTLP export, context propagation across fan-out, and detecting pinning and starvation."
---
# Observability for virtual threads: tracing a Spring Boot 3 request with Micrometer and OpenTelemetry

## 1. Overview

One property moves a Spring Boot service to virtual threads:

```properties
spring.threads.virtual.enabled=true
```

After this change, throughput can improve. Observability can also lose important information. Traces can lose child spans, logs can lose their trace ids, and thread-pool metrics no longer show the real limit.

In this post I will show how to fix these three problems. I will use Micrometer Observation, export data to OpenTelemetry, propagate context across every thread hop, and add metrics for a virtual-thread runtime.

The examples use Java 25 (LTS) and Spring Boot 3.5.x. Everything except the structured-concurrency section also works on Java 21.

## 2. Background: why the context disappears

Micrometer's tracing bridge stores the current span in a `ThreadLocal`. This lets the library work without changes in business code. A new thread does not have that value unless I copy it.

```text
Same thread            → context is visible, spans nest correctly
New thread, unwrapped  → context is empty, span becomes a new root
New thread, wrapped    → context is restored, spans nest correctly
```

With a platform-thread executor, Spring often wrapped the executor. There were also only a few pooled threads. With virtual threads, every request and every fan-out can create more threads. **Every hop must be explicit.**

The tool for this is `io.micrometer:context-propagation`. It uses `ThreadLocalAccessor`s to capture registered thread locals into an immutable `ContextSnapshot` and restore them on another thread. Micrometer registers `ObservationThreadLocalAccessor` for the observation and its span. Spring Boot's logging setup also contributes the MDC entries.

## 3. Step 1: dependencies and configuration

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
<dependency>
  <groupId>io.micrometer</groupId>
  <artifactId>context-propagation</artifactId>
</dependency>
```

The `micrometer-tracing-bridge-otel` bridge turns Micrometer observations into OpenTelemetry spans. The exporter sends them over OTLP. I can use `micrometer-tracing-bridge-brave` for a Zipkin backend instead. The application code stays the same because it uses the Observation API.

```properties
spring.application.name=orders-api
spring.threads.virtual.enabled=true

management.tracing.sampling.probability=1.0
management.otlp.tracing.endpoint=http://localhost:4318/v1/traces
management.otlp.metrics.export.url=http://localhost:4318/v1/metrics
management.endpoints.web.exposure.include=health,info,metrics,prometheus
```

I use a sampling value of `1.0` only in development. In production, I prefer a low head-based probability and tail sampling in the collector. This keeps slow and failed traces without paying for every successful request.

These flags also help with the diagnosis steps later:

```bash
java -XX:StartFlightRecording=filename=app.jfr,settings=profile,maxsize=512m \
     -Djdk.virtualThreadScheduler.parallelism=16 \
     -jar orders-api.jar
```

## 4. Step 2: instrument with the Observation API

I do not inject `Tracer` into business code. One `Observation` can create a span, a timer, and, if needed, a log context:

```java
@Service
class OrderService {

    private final ObservationRegistry registry;

    Order place(OrderRequest request) {
        return Observation.createNotStarted("order.place", registry)
                .lowCardinalityKeyValue("channel", request.channel())     // becomes a metric tag
                .highCardinalityKeyValue("order.id", request.id())        // span attribute only
                .observe(() -> doPlace(request));
    }
}
```

The difference between the two kinds of keys is important:

- **Low-cardinality keys** — these become metric tags, so use bounded values such as channel, region, or outcome.
- **High-cardinality keys** — these go to the span only, so identifiers belong here.

Putting an order id in a low-cardinality key can turn a 20-series metric into a 2-million-series bill.

For coarse-grained instrumentation, I can use `@Observed` after registering its aspect:

```java
@Configuration
class ObservationConfig {
    @Bean
    ObservedAspect observedAspect(ObservationRegistry registry) {
        return new ObservedAspect(registry);
    }
}

@Observed(name = "payment.authorize", contextualName = "authorize")
public Authorization authorize(Payment payment) { ... }
```

For one family of observations, I keep naming and tagging in an `ObservationConvention`. This keeps span names stable when a method is renamed.

## 5. Step 3: propagate context across every thread hop

This step fixes the broken traces. Three patterns cover most code.

### 5.1 Executors: wrap once at the bean definition

```java
@Bean
ExecutorService appExecutor(ObservationRegistry registry) {
    var factory = ContextSnapshotFactory.builder()
            .contextRegistry(ContextRegistry.getInstance())
            .build();
    return ContextExecutorService.wrap(
            Executors.newVirtualThreadPerTaskExecutor(),
            factory::captureAll);
}
```

Every task submitted to `appExecutor` now restores the observation and MDC from the submitting thread. The context is cleared after the task. I never inject a raw `Executors.newVirtualThreadPerTaskExecutor()` into application code. The wrapped bean should be the only executor available.

For Spring's `@Async` and `TaskExecutor`, I use a `TaskDecorator`:

```java
@Bean
TaskDecorator contextPropagatingDecorator() {
    var factory = ContextSnapshotFactory.builder().build();
    return runnable -> {
        ContextSnapshot snapshot = factory.captureAll();
        return () -> snapshot.wrap(runnable).run();
    };
}
```

Spring Boot applies a `TaskDecorator` bean to its auto-configured `SimpleAsyncTaskExecutor`, which is used when `spring.threads.virtual.enabled=true`. One bean therefore covers `@Async`, `@Scheduled`, and MVC async dispatches.

### 5.2 `CompletableFuture` chains: capture at the boundary

```java
ContextSnapshot snapshot = ContextSnapshotFactory.builder().build().captureAll();

CompletableFuture
        .supplyAsync(snapshot.wrap(() -> pricing.quote(request)), appExecutor)
        .thenApply(snapshot.wrap(this::applyDiscount));
```

I capture once outside the chain. If I capture inside a lambda that is already on the wrong thread, there is no useful context to capture.

### 5.3 Structured concurrency: bind inside each fork

`ScopedValue` bindings are inherited by subtasks. Thread-local observation context is not. A `StructuredTaskScope` fan-out therefore needs the snapshot passed to each fork:

```java
Dashboard load(long userId) throws InterruptedException {
    ContextSnapshot snapshot = ContextSnapshotFactory.builder().build().captureAll();

    try (var scope = StructuredTaskScope.open()) {
        var user   = scope.fork(snapshot.wrap(() -> userClient.fetch(userId)));
        var orders = scope.fork(snapshot.wrap(() -> orderClient.fetchFor(userId)));

        scope.join();
        return new Dashboard(user.get(), orders.get());
    }
}
```

Each subtask now creates a child span under the request span. Cancellation still works because `wrap` returns the same callable semantics. `StructuredTaskScope` is still a preview API in Java 25, so I keep it behind one helper class.

## 6. Step 4: correlate logs with traces

Spring Boot puts `traceId` and `spanId` in the MDC when a tracer is on the classpath. I make them visible in the log pattern:

```properties
logging.pattern.level=%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]
```

A log line then looks like this:

```text
INFO  [orders-api,3f9a1c0b5e2d4a77,7c1d9e42ab35] c.g.orders.OrderService : order accepted
```

Other business context, such as a tenant or a correlation id from a header, is not part of the trace identity. I bind it as a `ScopedValue` at the edge and mirror it into the MDC there. I use one binding site for each entry point:

```java
@Component
class TenantFilter extends OncePerRequestFilter {

    static final ScopedValue<String> TENANT = ScopedValue.newInstance();

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain) {
        String tenant = req.getHeader("X-Tenant-Id");
        ScopedValue.where(TENANT, tenant).run(() -> {
            MDC.put("tenant", tenant);
            try {
                chain.doFilter(req, res);
            } catch (Exception e) {
                throw new IllegalStateException(e);
            } finally {
                MDC.remove("tenant");
            }
        });
    }
}
```

The `finally` is required. The request thread ends after the request, but `captureAll()` also copies the MDC. A stale entry can therefore reach every forked task.

## 7. Step 5: metrics that still mean something

The old pool gauges are no longer useful. I replace them with signals for the real bottleneck, which is usually not the thread count after the switch.

**In-flight requests** are the closest saturation gauge:

```java
@Bean
MeterBinder inFlightRequests(HttpRequestCounter counter) {   // a LongAdder incremented in a filter
    return registry -> Gauge.builder("http.server.requests.active", counter, HttpRequestCounter::value)
            .description("Requests currently being served")
            .register(registry);
}
```

**Connection pool utilisation** is the real ceiling. HikariCP already publishes it. I alert on `hikaricp.connections.pending > 0` and on `hikaricp.connections.acquire` p99. A virtual-thread service under load queues here, not on the threads.

**Downstream latency and errors** should be measured for each client with `http.client.requests`. A slow dependency no longer appears as thread exhaustion, so I need to show it directly.

**Pinning events** come from JFR, described in Step 8. If I run a JFR-to-metrics bridge, I can export them as a counter.

I remove these two kinds of dashboard and alert:

- `jvm.threads.live` as a saturation signal — it does not include virtual threads, so it stays flat under load.
- `tomcat.threads.busy` and `executor.pool.*` on a virtual-thread runtime — these gauges are absent or describe a scheduler that is not sized per request.

## 8. Step 6: detect pinning and starvation

Since Java 24 (JEP 491), blocking inside `synchronized` no longer pins a carrier thread. The old `-Djdk.tracePinnedThreads` sweep is therefore obsolete, and the property was removed. Native frames such as JNI or some drivers, and blocking during class initialization, can still pin a carrier.

JFR is the supported way to see these events:

```bash
# 1. Record (or use -XX:StartFlightRecording as in Step 1)
jcmd <pid> JFR.start name=vt settings=profile filename=vt.jfr

# 2. Pinning: every occurrence, with the stack that caused it
jfr print --events jdk.VirtualThreadPinned vt.jfr

# 3. Starvation: the scheduler could not accept work
jfr print --events jdk.VirtualThreadSubmitFailed vt.jfr

# 4. How many virtual threads are actually alive
jfr summary vt.jfr | grep VirtualThread
```

For a stalled service, I use a structured thread dump. It groups subtasks under the scope that created them:

```bash
jcmd <pid> Thread.dump_to_file -format=json dump.json
```

This table gives a starting point:

| Symptom | Likely cause | Where to look |
|---|---|---|
| p99 latency up, CPU low, few pinning events | Downstream or connection-pool queueing | `hikaricp.connections.pending`, `http.client.requests` |
| Frequent `jdk.VirtualThreadPinned` with native frames | JNI or a driver blocking a carrier | Stack in the JFR event; raise `jdk.virtualThreadScheduler.parallelism` as a stopgap |
| `jdk.VirtualThreadSubmitFailed` | Carrier pool saturated or starved | Pinning first, then parallelism |
| Traces with orphan roots | An unwrapped thread hop | Step 3 — find the raw executor |
| Log lines with empty `traceId` | Same, on the MDC side | Step 3a decorator coverage |

## 9. Step 7: test the instrumentation

Context propagation can fail without an obvious error, so I assert on it. `micrometer-observation-test` makes the observation graph testable:

```java
@Test
void fan_out_creates_child_observations() {
    TestObservationRegistry registry = TestObservationRegistry.create();
    var service = new DashboardService(registry, userClient, orderClient);

    service.load(42L);

    assertThat(registry)
            .hasObservationWithNameEqualTo("dashboard.load")
            .that()
            .hasLowCardinalityKeyValue("channel", "web")
            .hasBeenStarted()
            .hasBeenStopped();
}
```

With the OpenTelemetry bridge, an integration test can also check that the trace has one root:

```java
@SpringBootTest(properties = "management.tracing.sampling.probability=1.0")
class TracePropagationTest {

    @Autowired InMemorySpanExporter exporter;   // registered as a test-only SpanProcessor

    @Test
    void forked_calls_stay_in_the_same_trace() {
        dashboardService.load(42L);

        var spans = exporter.getFinishedSpanItems();
        assertThat(spans).hasSize(3);
        assertThat(spans.stream().map(SpanData::getTraceId).distinct()).hasSize(1);
        assertThat(spans).filteredOn(s -> s.getParentSpanId().equals("0000000000000000")).hasSize(1);
    }
}
```

The `distinct().hasSize(1)` assertion checks Step 5. It fails when somebody submits work to an executor that is not wrapped.

## 10. Common pitfalls

- **Wrap the executor at the bean, not at each call site.** One forgotten call site then breaks only some traces, which is difficult to find.
- **Capture the snapshot at the right time.** `captureAll()` must run on a thread that has the context. Inside the task, it captures an empty context.
- **Do not leave `sampling.probability=1.0` in production.** A virtual-thread service can run many more concurrent requests. Full sampling can hit the collector's rate limit.
- **Use Observation instead of `Tracer` directly.** Observation gives a span, a timer, metric tags, and a way to change tracing backends.
- **Keep high-cardinality keys out of low-cardinality keys.** Ids, URLs with path variables, and stack messages must not become metric tags.
- **Do not alert on thread counts.** Alert on the connection pool, the in-flight gauge, and downstream latency.
- **Check that `@Async` is covered.** It is covered only when a `TaskDecorator` bean exists. Add a test that checks the trace id inside the async method.
- **Cover non-HTTP entry points.** Kafka listeners and scheduled jobs start without a context. Open an observation at each entry point or their spans become orphan roots.

## 11. Conclusion

Virtual threads do not break observability by themselves. The problem is thread-local context combined with thread hops that do not copy it.

I use these steps:

1. Instrument with the **Observation API** and keep low- and high-cardinality keys separate.
2. Export over **OTLP** and sample carefully in production.
3. Make **every thread hop explicit** with wrapped executor beans, a `TaskDecorator`, and `snapshot.wrap` for futures and `StructuredTaskScope` forks.
4. Put `traceId` and `spanId` in the log pattern. Bind business context once for each entry point and clean it up.
5. Replace pool gauges with **in-flight requests, connection-pool pressure, and downstream latency**.
6. Watch `jdk.VirtualThreadPinned` and `jdk.VirtualThreadSubmitFailed` in JFR instead of the removed pinning property.
7. **Test propagation** by checking that one request has one trace id.

With these steps, switching to virtual threads changes throughput without changing what I can see.
