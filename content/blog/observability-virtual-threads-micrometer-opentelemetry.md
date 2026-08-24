---
title: "Observability for virtual threads: tracing a Spring Boot 3 request with Micrometer and OpenTelemetry"
date: 2026-08-24T00:00:00Z
description: "A hands-on tutorial on keeping traces, logs and metrics intact after switching a Spring Boot service to virtual threads: Micrometer Observation, OTLP export, context propagation across fan-out, and detecting pinning and starvation."
---

## Problem

Flipping one property moves a Spring Boot service onto virtual threads:

```properties
spring.threads.virtual.enabled=true
```

Throughput improves, and then observability quietly degrades in three ways:

1. **Traces lose spans.** Anything that hands work to another thread — an `ExecutorService`, a `CompletableFuture`, a reactive `publishOn` — no longer sees the current span, because the span lives in a `ThreadLocal` that the new thread does not have. The parent span closes with no children, or worse, a child span shows up as its own trace root.
2. **Logs lose correlation.** `traceId`/`spanId` reach the log line through the SLF4J MDC, another `ThreadLocal`. Same failure mode: the log line from a forked task carries an empty trace id, so a stack trace can no longer be joined to the request that produced it.
3. **Metrics lose their saturation signal.** The Tomcat thread-pool gauges that used to answer "are we out of threads?" become meaningless: there is no pool. `ThreadMXBean.getThreadCount()` does not count virtual threads either, so the dashboards look permanently healthy while requests queue up behind a connection pool or a pinned carrier.

This post is a step-by-step guide to fixing all three: instrument with the Micrometer Observation API, export to OpenTelemetry, propagate context explicitly across every thread hop, and add the metrics that actually reveal saturation on a virtual-thread runtime.

Baseline: Java 25 (LTS), Spring Boot 3.5.x. Everything except the structured-concurrency section works on Java 21 too.

## Background: why the context disappears

Micrometer's tracing bridge stores the current span in a `ThreadLocal`. That is not a flaw — it is the only way a library can be transparent to your code. The consequence is a simple rule:

```text
Same thread            → context is visible, spans nest correctly
New thread, unwrapped  → context is empty, span becomes a new root
New thread, wrapped    → context is restored, spans nest correctly
```

With platform threads and a pooled executor, "wrapped" was usually somebody else's problem: Spring wrapped the executor for you, and pooled threads were few enough that a leaked MDC entry was survivable. On virtual threads, every request creates a thread and every fan-out creates more, so **every hop must be explicit**.

The tool for that is `io.micrometer:context-propagation`: a registry of `ThreadLocalAccessor`s that can *capture* all registered thread locals into an immutable `ContextSnapshot` and *restore* them on another thread. Micrometer registers `ObservationThreadLocalAccessor` (the observation and its span) and Spring Boot's logging setup contributes the MDC entries.

## Step 1 — Dependencies and configuration

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

The bridge (`micrometer-tracing-bridge-otel`) turns Micrometer observations into OpenTelemetry spans; the exporter ships them over OTLP. Swap the bridge for `micrometer-tracing-bridge-brave` if your backend speaks Zipkin — the application code below does not change, which is the whole point of the Observation API.

```properties
spring.application.name=orders-api
spring.threads.virtual.enabled=true

management.tracing.sampling.probability=1.0
management.otlp.tracing.endpoint=http://localhost:4318/v1/traces
management.otlp.metrics.export.url=http://localhost:4318/v1/metrics
management.endpoints.web.exposure.include=health,info,metrics,prometheus
```

Sample at `1.0` in dev only. In production, prefer a low head-based probability plus tail sampling in the collector, so you keep the slow and failed traces without paying for the happy path.

Two flags worth setting from day one, because they make the later diagnosis steps possible:

```bash
java -XX:StartFlightRecording=filename=app.jfr,settings=profile,maxsize=512m \
     -Djdk.virtualThreadScheduler.parallelism=16 \
     -jar orders-api.jar
```

## Step 2 — Instrument with the Observation API, not the tracer

Do not inject `Tracer` in business code. An `Observation` produces a span *and* a timer *and* (optionally) a log context from a single instrumentation point:

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

The cardinality distinction is the part teams get wrong. Low-cardinality keys end up as **metric tags** — keep them to bounded enumerations (channel, region, outcome). High-cardinality keys are attached to the **span only**, which is where identifiers belong. Putting an order id in a low-cardinality key is how you turn a 20-series metric into a 2-million-series bill.

For coarse-grained instrumentation, `@Observed` is equivalent once the aspect is registered:

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

Custom naming and tagging for a whole family of observations belongs in an `ObservationConvention`, not scattered across call sites — that keeps span names stable when the method is renamed.

## Step 3 — Propagate context across every thread hop

This is the step that actually fixes broken traces. Three patterns cover almost all code.

**3a. Executors: wrap once, at the bean definition.**

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

Every task submitted to `appExecutor` now runs with the submitting thread's observation and MDC restored, and cleared afterwards. Never inject a raw `Executors.newVirtualThreadPerTaskExecutor()` into application code — make the wrapped bean the only one available.

For Spring's own `@Async` and `TaskExecutor`, the equivalent is a `TaskDecorator`:

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

Spring Boot applies a `TaskDecorator` bean to its auto-configured `SimpleAsyncTaskExecutor` (the one used when `spring.threads.virtual.enabled=true`), so a single bean covers `@Async`, `@Scheduled` and MVC async dispatches.

**3b. `CompletableFuture` chains: capture at the boundary.**

```java
ContextSnapshot snapshot = ContextSnapshotFactory.builder().build().captureAll();

CompletableFuture
        .supplyAsync(snapshot.wrap(() -> pricing.quote(request)), appExecutor)
        .thenApply(snapshot.wrap(this::applyDiscount));
```

Capture *once*, outside the chain: capturing inside a lambda that already runs on the wrong thread captures nothing useful.

**3c. Structured concurrency: bind explicitly inside each fork.**

`ScopedValue` bindings are inherited by subtasks; `ThreadLocal`-based observation context is not. So a `StructuredTaskScope` fan-out needs the snapshot passed in:

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

Each subtask now produces a child span under the request span, and cancellation still works because `wrap` returns the same callable semantics. (`StructuredTaskScope` is still a preview API in Java 25 — keep it behind one helper class.)

## Step 4 — Correlate logs with traces

Spring Boot puts `traceId` and `spanId` in the MDC as soon as a tracer is on the classpath. Make them visible:

```properties
logging.pattern.level=%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]
```

A log line then reads:

```text
INFO  [orders-api,3f9a1c0b5e2d4a77,7c1d9e42ab35] c.g.orders.OrderService : order accepted
```

Business context that is not part of the trace identity (tenant, correlation id from a header) is best bound as a `ScopedValue` at the edge and mirrored into the MDC there — one binding site per entry point:

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

The `finally` is not optional. On virtual threads the thread dies with the request, so a leak is bounded — but the MDC is also copied by `captureAll()`, and a stale entry propagates into every forked task.

## Step 5 — Metrics that still mean something

The pool gauges are gone. Replace them with signals that describe *the actual bottleneck*, which after the switch is almost never the thread count.

**In-flight requests** — the closest thing to a saturation gauge:

```java
@Bean
MeterBinder inFlightRequests(HttpRequestCounter counter) {   // a LongAdder incremented in a filter
    return registry -> Gauge.builder("http.server.requests.active", counter, HttpRequestCounter::value)
            .description("Requests currently being served")
            .register(registry);
}
```

**Connection pool utilisation** — the real ceiling. HikariCP already publishes it; alert on `hikaricp.connections.pending > 0` and on `hikaricp.connections.acquire` p99. A virtual-thread service under load queues *here*, not on threads.

**Downstream latency and errors** per client, from `http.client.requests` — with virtual threads, a slow dependency no longer shows up as thread exhaustion, so it must be visible directly.

**Pinning events**, via JFR (see Step 6), exported as a counter if you run a JFR-to-metrics bridge.

Two anti-patterns to retire from dashboards and alerts:

- `jvm.threads.live` as a saturation signal — it excludes virtual threads, so it stays flat under any load.
- `tomcat.threads.busy` / `executor.pool.*` on a virtual-thread runtime — the gauges are either absent or describe a scheduler you do not size per request.

## Step 6 — Detect pinning and starvation

Since Java 24 (JEP 491), blocking inside `synchronized` no longer pins a carrier thread, so the historical `-Djdk.tracePinnedThreads` sweep is obsolete (the property was removed). What is left pins rarely but hurts: native frames (JNI, some drivers) and blocking during class initialization.

JFR is the supported way to see it:

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

And for a stalled service, the structured thread dump is the fastest read, because it groups subtasks under the scope that forked them:

```bash
jcmd <pid> Thread.dump_to_file -format=json dump.json
```

Interpretation guide:

| Symptom | Likely cause | Where to look |
|---|---|---|
| p99 latency up, CPU low, few pinning events | Downstream or connection-pool queueing | `hikaricp.connections.pending`, `http.client.requests` |
| Frequent `jdk.VirtualThreadPinned` with native frames | JNI or a driver blocking a carrier | Stack in the JFR event; raise `jdk.virtualThreadScheduler.parallelism` as a stopgap |
| `jdk.VirtualThreadSubmitFailed` | Carrier pool saturated or starved | Pinning first, then parallelism |
| Traces with orphan roots | An unwrapped thread hop | Step 3 — find the raw executor |
| Log lines with empty `traceId` | Same, on the MDC side | Step 3a decorator coverage |

## Step 7 — Test the instrumentation

Broken propagation is a silent failure, so assert on it. `micrometer-observation-test` makes the observation graph assertable:

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

And with the OpenTelemetry bridge in place, an integration test can assert the trace actually has one root:

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

The `distinct().hasSize(1)` assertion is the regression test for Step 3: it fails the moment somebody submits work to an unwrapped executor.

## Common pitfalls

- **Wrapping the executor at the call site instead of the bean.** One forgotten call site breaks a subset of traces, which is far harder to notice than all of them.
- **Capturing the snapshot lazily.** `captureAll()` must run on the thread that *has* the context. Inside the task, it captures emptiness.
- **Leaving `sampling.probability=1.0` in production.** Full sampling on a virtual-thread service — which happily runs far more concurrent requests than before — is how you discover your collector's rate limit.
- **Instrumenting with `Tracer` directly.** You get a span but no timer, no metric tags, and no way to switch backends. Use `Observation`.
- **High-cardinality keys as low-cardinality ones.** Ids, URLs with path variables and stack messages must never become metric tags.
- **Alerting on thread counts.** Move the alert to the connection pool, the in-flight gauge and downstream latency.
- **Assuming `@Async` is covered.** It is only covered if a `TaskDecorator` bean exists; check it with a test that asserts the trace id inside the async method.
- **Forgetting the non-HTTP entry points.** Kafka listeners and scheduled jobs start with no context; they need their own observation opened at the entry point, or every span they produce is an orphan.

## Outcome

Virtual threads do not break observability — `ThreadLocal`-based context plus implicit thread hops do. The fix is mechanical:

1. Instrument with the **Observation API**, keeping low- and high-cardinality keys separate.
2. Export over **OTLP**, sampling conservatively in production.
3. Make **every thread hop explicit**: wrapped executor beans, a `TaskDecorator`, `snapshot.wrap` for futures and `StructuredTaskScope` forks.
4. Put `traceId`/`spanId` in the log pattern and bind business context once per entry point, with cleanup.
5. Replace pool gauges with **in-flight requests, connection-pool pressure and downstream latency**.
6. Watch `jdk.VirtualThreadPinned` and `jdk.VirtualThreadSubmitFailed` in JFR instead of the removed pinning property.
7. **Assert propagation in tests** — one trace id per request is a cheap, high-value invariant.

Do these seven and the switch to virtual threads becomes what it should be: a throughput change, not a visibility change.
