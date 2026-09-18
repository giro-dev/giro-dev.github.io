---
title: "Observabilidad para virtual threads: trazas de una petición Spring Boot 3 con Micrometer y OpenTelemetry"
date: 2026-08-24T00:00:00Z
description: "Una guía práctica para mantener trazas, logs y métricas después de pasar un servicio Spring Boot a virtual threads: Micrometer Observation, exportación OTLP, propagación de contexto en fan-out y detección de pinning y starvation."
---
# Observabilidad para virtual threads: trazas de una petición Spring Boot 3 con Micrometer y OpenTelemetry

## 1. Introducción

Una propiedad pasa un servicio Spring Boot a virtual threads:

```properties
spring.threads.virtual.enabled=true
```

Después de este cambio, el throughput puede mejorar. También se puede perder información de observabilidad. Las trazas pueden perder spans hijos, los logs pueden perder sus trace ids y las métricas de los thread pools ya no muestran el límite real.

En este artículo mostraré cómo resolver estos tres problemas. Usaré Micrometer Observation, exportaré los datos a OpenTelemetry, propagaré el contexto en cada salto de thread y añadiré métricas para un runtime de virtual threads.

Los ejemplos usan Java 25 (LTS) y Spring Boot 3.5.x. Todo excepto la sección de structured concurrency también funciona en Java 21.

## 2. Contexto: por qué desaparece el contexto

El bridge de tracing de Micrometer guarda el span actual en un `ThreadLocal`. Esto permite que la librería funcione sin cambios en el código de negocio. Un thread nuevo no tiene ese valor si no lo copio.

```text
Same thread            → context is visible, spans nest correctly
New thread, unwrapped  → context is empty, span becomes a new root
New thread, wrapped    → context is restored, spans nest correctly
```

Con un executor de platform threads, Spring solía envolver el executor. También había pocos threads en el pool. Con virtual threads, cada petición y cada fan-out pueden crear más threads. **Cada salto debe ser explícito.**

La herramienta para hacerlo es `io.micrometer:context-propagation`. Usa `ThreadLocalAccessor`s para capturar los thread locals registrados en un `ContextSnapshot` inmutable y restaurarlos en otro thread. Micrometer registra `ObservationThreadLocalAccessor` para la observation y su span. La configuración de logs de Spring Boot también aporta las entradas MDC.

## 3. Paso 1: dependencias y configuración

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

El bridge `micrometer-tracing-bridge-otel` convierte las observations de Micrometer en spans de OpenTelemetry. El exporter los envía por OTLP. También puedo usar `micrometer-tracing-bridge-brave` con un backend Zipkin. El código de la aplicación no cambia porque usa la Observation API.

```properties
spring.application.name=orders-api
spring.threads.virtual.enabled=true

management.tracing.sampling.probability=1.0
management.otlp.tracing.endpoint=http://localhost:4318/v1/traces
management.otlp.metrics.export.url=http://localhost:4318/v1/metrics
management.endpoints.web.exposure.include=health,info,metrics,prometheus
```

Uso un sampling de `1.0` solo en desarrollo. En producción prefiero una probabilidad head-based baja y tail sampling en el collector. Así conservo las trazas lentas y fallidas sin pagar por cada petición correcta.

Estos flags también ayudan en los pasos de diagnóstico:

```bash
java -XX:StartFlightRecording=filename=app.jfr,settings=profile,maxsize=512m \
     -Djdk.virtualThreadScheduler.parallelism=16 \
     -jar orders-api.jar
```

## 4. Paso 2: instrumentar con la Observation API

No inyecto `Tracer` en el código de negocio. Una `Observation` puede crear un span, un timer y, si hace falta, un contexto de log:

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

La diferencia entre los dos tipos de claves es importante:

- **Claves de baja cardinalidad** — se convierten en tags de métrica, así que uso valores limitados como channel, region u outcome.
- **Claves de alta cardinalidad** — van solo al span, así que los identificadores van aquí.

Poner un order id en una clave de baja cardinalidad puede convertir una métrica de 20 series en una factura de 2 millones de series.

Para instrumentación más general, puedo usar `@Observed` después de registrar su aspect:

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

Para una familia de observations, mantengo los nombres y los tags en una `ObservationConvention`. Así los nombres de los spans siguen estables cuando cambia el nombre de un método.

## 5. Paso 3: propagar el contexto en cada salto de thread

Este paso arregla las trazas rotas. Tres patrones cubren la mayor parte del código.

### 5.1 Executors: envolver una vez en la definición del bean

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

Cada task enviado a `appExecutor` restaura la observation y el MDC del thread que lo envía. El contexto se limpia después del task. Nunca inyecto un `Executors.newVirtualThreadPerTaskExecutor()` sin envolver en el código de la aplicación. El bean envuelto debe ser el único executor disponible.

Para `@Async` y `TaskExecutor` de Spring, uso un `TaskDecorator`:

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

Spring Boot aplica un bean `TaskDecorator` a su `SimpleAsyncTaskExecutor` auto-configurado, que se usa cuando `spring.threads.virtual.enabled=true`. Un solo bean cubre `@Async`, `@Scheduled` y los dispatches asíncronos de MVC.

### 5.2 Cadenas `CompletableFuture`: capturar en el límite

```java
ContextSnapshot snapshot = ContextSnapshotFactory.builder().build().captureAll();

CompletableFuture
        .supplyAsync(snapshot.wrap(() -> pricing.quote(request)), appExecutor)
        .thenApply(snapshot.wrap(this::applyDiscount));
```

Capturo una vez fuera de la cadena. Si capturo dentro de un lambda que ya está en el thread equivocado, no hay contexto útil que capturar.

### 5.3 Structured concurrency: vincular dentro de cada fork

Los bindings de `ScopedValue` se heredan a los subtasks. El contexto de observación basado en thread locals no. Por eso un fan-out con `StructuredTaskScope` necesita pasar el snapshot a cada fork:

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

Ahora cada subtask crea un child span bajo el request span. La cancelación sigue funcionando porque `wrap` devuelve la misma semántica de callable. `StructuredTaskScope` todavía es una API preview en Java 25, así que la mantengo detrás de una sola clase helper.

## 6. Paso 4: correlacionar logs y trazas

Spring Boot pone `traceId` y `spanId` en el MDC cuando hay un tracer en el classpath. Los hago visibles en el patrón de logs:

```properties
logging.pattern.level=%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]
```

Una línea de log tiene este aspecto:

```text
INFO  [orders-api,3f9a1c0b5e2d4a77,7c1d9e42ab35] c.g.orders.OrderService : order accepted
```

Otro contexto de negocio, como un tenant o un correlation id de un header, no forma parte de la identidad de la traza. Lo vinculo como `ScopedValue` en el límite y lo copio al MDC allí. Uso un único punto de binding para cada entry point:

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

El `finally` es necesario. El thread de la petición termina después de la petición, pero `captureAll()` también copia el MDC. Una entrada antigua puede llegar a todos los tasks forked.

## 7. Paso 5: métricas que siguen teniendo sentido

Las métricas antiguas de los pools ya no son útiles. Las sustituyo por señales del cuello de botella real, que después del cambio normalmente no es el número de threads.

**Peticiones en curso** son la métrica de saturación más cercana:

```java
@Bean
MeterBinder inFlightRequests(HttpRequestCounter counter) {   // a LongAdder incremented in a filter
    return registry -> Gauge.builder("http.server.requests.active", counter, HttpRequestCounter::value)
            .description("Requests currently being served")
            .register(registry);
}
```

**El uso del pool de conexiones** es el límite real. HikariCP ya lo publica. Alerto cuando `hikaricp.connections.pending > 0` y sobre el p99 de `hikaricp.connections.acquire`. Un servicio con virtual threads bajo carga espera aquí, no en los threads.

**La latencia y los errores downstream** deben medirse para cada cliente con `http.client.requests`. Una dependencia lenta ya no aparece como agotamiento de threads, así que debo mostrarla directamente.

**Los eventos de pinning** vienen de JFR, descrito en el Paso 8. Si uso un bridge de JFR a métricas, puedo exportarlos como un counter.

Elimino estos dos tipos de métricas de dashboards y alertas:

- `jvm.threads.live` como señal de saturación — no incluye virtual threads y se mantiene plana con carga.
- `tomcat.threads.busy` y `executor.pool.*` en un runtime de virtual threads — estas métricas no existen o describen un scheduler que no dimensiono por petición.

## 8. Paso 6: detectar pinning y starvation

Desde Java 24 (JEP 491), bloquear dentro de `synchronized` ya no fija un carrier. Por eso el antiguo análisis con `-Djdk.tracePinnedThreads` está obsoleto y la propiedad se eliminó. Los frames nativos, como JNI o algunos drivers, y el bloqueo durante la inicialización de clases todavía pueden fijar un carrier.

JFR es la forma soportada de ver estos eventos:

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

Para un servicio atascado, uso un thread dump estructurado. Agrupa los subtasks bajo el scope que los creó:

```bash
jcmd <pid> Thread.dump_to_file -format=json dump.json
```

Esta tabla es un buen punto de partida:

| Síntoma | Causa probable | Dónde mirar |
|---|---|---|
| p99 de latencia alta, CPU baja, pocos eventos de pinning | Cola downstream o del pool de conexiones | `hikaricp.connections.pending`, `http.client.requests` |
| Muchos `jdk.VirtualThreadPinned` con frames nativos | JNI o un driver que bloquea un carrier | Stack del evento JFR; aumentar `jdk.virtualThreadScheduler.parallelism` temporalmente |
| `jdk.VirtualThreadSubmitFailed` | Pool de carriers saturado o sin recursos | Primero el pinning, después el parallelism |
| Trazas con raíces huérfanas | Un salto de thread sin envolver | Paso 3 — encontrar el executor directo |
| Líneas de log con `traceId` vacío | Lo mismo, en la parte del MDC | Cobertura del decorator del Paso 5.1 |

## 9. Paso 7: probar la instrumentación

La propagación de contexto puede fallar sin un error evidente, así que hago assertions. `micrometer-observation-test` permite probar el grafo de observaciones:

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

Con el bridge de OpenTelemetry, un test de integración también puede comprobar que la traza tiene una sola raíz:

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

La assertion `distinct().hasSize(1)` comprueba el Paso 5. Falla cuando alguien envía trabajo a un executor que no está envuelto.

## 10. Problemas habituales

- **Envuelve el executor en el bean, no en cada call site.** Un call site olvidado rompe solo algunas trazas y es difícil de encontrar.
- **Captura el snapshot en el momento correcto.** `captureAll()` debe ejecutarse en un thread que tenga el contexto. Dentro del task captura un contexto vacío.
- **No dejes `sampling.probability=1.0` en producción.** Un servicio con virtual threads puede gestionar muchas más peticiones concurrentes. El sampling completo puede alcanzar el rate limit del collector.
- **Usa Observation en lugar de `Tracer` directamente.** Observation proporciona un span, un timer, tags de métrica y una forma de cambiar de backend de tracing.
- **Mantén las claves de alta cardinalidad fuera de las de baja cardinalidad.** Ids, URLs con variables de path y mensajes de stack no deben convertirse en tags de métrica.
- **No alertes por los números de threads.** Alerta sobre el pool de conexiones, la métrica de peticiones en curso y la latencia downstream.
- **Comprueba que `@Async` está cubierto.** Solo lo está cuando existe un bean `TaskDecorator`. Añade un test que compruebe el trace id dentro del método asíncrono.
- **Cubre los entry points que no son HTTP.** Los listeners Kafka y los jobs programados empiezan sin contexto. Abre una observation en cada entry point o sus spans serán raíces huérfanas.

## 11. Conclusión

Los virtual threads no rompen la observabilidad por sí solos. El problema es el contexto basado en thread locals combinado con saltos de thread que no lo copian.

Uso estos pasos:

1. Instrumento con la **Observation API** y separo las claves de baja y alta cardinalidad.
2. Exporto por **OTLP** y hago sampling con cuidado en producción.
3. Hago explícito **cada salto de thread** con beans de executors envueltos, un `TaskDecorator` y `snapshot.wrap` para futures y forks de `StructuredTaskScope`.
4. Pongo `traceId` y `spanId` en el patrón de logs. Vinculo el contexto de negocio una vez por entry point y lo limpio.
5. Sustituyo las métricas de los pools por **peticiones en curso, presión del pool de conexiones y latencia downstream**.
6. Vigilo `jdk.VirtualThreadPinned` y `jdk.VirtualThreadSubmitFailed` en JFR en lugar de la propiedad de pinning eliminada.
7. **Pruebo la propagación** comprobando que una petición tiene un solo trace id.

Con estos pasos, cambiar a virtual threads cambia el throughput sin cambiar lo que puedo ver.
