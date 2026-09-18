---
title: "Observabilitat per a virtual threads: traçar una petició Spring Boot 3 amb Micrometer i OpenTelemetry"
date: 2026-08-24T00:00:00Z
description: "Una guia pràctica per mantenir traces, logs i mètriques després de passar un servei Spring Boot a virtual threads: Micrometer Observation, exportació OTLP, propagació de context en fan-out i detecció de pinning i starvation."
---
# Observabilitat per a virtual threads: traçar una petició Spring Boot 3 amb Micrometer i OpenTelemetry

## 1. Introducció

Una propietat passa un servei Spring Boot a virtual threads:

```properties
spring.threads.virtual.enabled=true
```

Després d’aquest canvi, el throughput pot millorar. També es pot perdre informació d’observabilitat. Les traces poden perdre spans fills, els logs poden perdre els seus trace ids i les mètriques dels thread pools ja no mostren el límit real.

En aquest article mostraré com resoldre aquests tres problemes. Faré servir Micrometer Observation, exportaré les dades a OpenTelemetry, propagaré el context a cada salt de thread i afegiré mètriques per a un runtime de virtual threads.

Els exemples fan servir Java 25 (LTS) i Spring Boot 3.5.x. Tot excepte la secció de structured concurrency també funciona a Java 21.

## 2. Context: per què desapareix el context

El bridge de tracing de Micrometer guarda el span actual en un `ThreadLocal`. Això permet que la llibreria funcioni sense canvis al codi de negoci. Un thread nou no té aquest valor si no el copio.

```text
Same thread            → context is visible, spans nest correctly
New thread, unwrapped  → context is empty, span becomes a new root
New thread, wrapped    → context is restored, spans nest correctly
```

Amb un executor de platform threads, Spring sovint embolcallava l’executor. També hi havia pocs threads al pool. Amb virtual threads, cada petició i cada fan-out poden crear més threads. **Cada salt ha de ser explícit.**

L’eina per fer-ho és `io.micrometer:context-propagation`. Fa servir `ThreadLocalAccessor`s per capturar els thread locals registrats en un `ContextSnapshot` immutable i restaurar-los en un altre thread. Micrometer registra `ObservationThreadLocalAccessor` per a l’observation i el seu span. La configuració de logs de Spring Boot també aporta les entrades MDC.

## 3. Pas 1: dependències i configuració

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

El bridge `micrometer-tracing-bridge-otel` converteix les observations de Micrometer en spans d’OpenTelemetry. L’exporter els envia per OTLP. En lloc d’això puc fer servir `micrometer-tracing-bridge-brave` amb un backend Zipkin. El codi de l’aplicació no canvia perquè fa servir l’Observation API.

```properties
spring.application.name=orders-api
spring.threads.virtual.enabled=true

management.tracing.sampling.probability=1.0
management.otlp.tracing.endpoint=http://localhost:4318/v1/traces
management.otlp.metrics.export.url=http://localhost:4318/v1/metrics
management.endpoints.web.exposure.include=health,info,metrics,prometheus
```

Faig servir un sampling d’`1.0` només en desenvolupament. En producció prefereixo una probabilitat head-based baixa i tail sampling al collector. Això conserva les traces lentes i fallides sense pagar per cada petició correcta.

Aquests flags també ajuden en els passos de diagnòstic:

```bash
java -XX:StartFlightRecording=filename=app.jfr,settings=profile,maxsize=512m \
     -Djdk.virtualThreadScheduler.parallelism=16 \
     -jar orders-api.jar
```

## 4. Pas 2: instrumentar amb l’Observation API

No injecto `Tracer` al codi de negoci. Una `Observation` pot crear un span, un timer i, si cal, un context de log:

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

La diferència entre els dos tipus de claus és important:

- **Claus de baixa cardinalitat** — es converteixen en tags de mètrica, així que faig servir valors limitats com channel, region o outcome.
- **Claus d’alta cardinalitat** — van només al span, així que els identificadors van aquí.

Posar un order id en una clau de baixa cardinalitat pot convertir una mètrica de 20 sèries en una factura de 2 milions de sèries.

Per a instrumentació més general, puc fer servir `@Observed` després de registrar el seu aspect:

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

Per a una família d’observations, mantinc els noms i els tags en una `ObservationConvention`. Això manté estables els noms dels spans quan canvia el nom d’un mètode.

## 5. Pas 3: propagar el context a cada salt de thread

Aquest pas arregla les traces trencades. Tres patrons cobreixen la major part del codi.

### 5.1 Executors: embolcallar una vegada en la definició del bean

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

Cada task enviat a `appExecutor` restaura l’observation i l’MDC del thread que l’envia. El context es neteja després del task. No injecto mai un `Executors.newVirtualThreadPerTaskExecutor()` sense embolcallar al codi de l’aplicació. El bean embolcallat ha de ser l’únic executor disponible.

Per a `@Async` i `TaskExecutor` de Spring, faig servir un `TaskDecorator`:

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

Spring Boot aplica un bean `TaskDecorator` al seu `SimpleAsyncTaskExecutor` auto-configurat, que es fa servir quan `spring.threads.virtual.enabled=true`. Un sol bean cobreix `@Async`, `@Scheduled` i els dispatches asíncrons de MVC.

### 5.2 Cadenes `CompletableFuture`: capturar al límit

```java
ContextSnapshot snapshot = ContextSnapshotFactory.builder().build().captureAll();

CompletableFuture
        .supplyAsync(snapshot.wrap(() -> pricing.quote(request)), appExecutor)
        .thenApply(snapshot.wrap(this::applyDiscount));
```

Capturo una vegada fora de la cadena. Si capturo dins d’un lambda que ja corre en el thread equivocat, no hi ha context útil per capturar.

### 5.3 Structured concurrency: vincular dins de cada fork

Les binding de `ScopedValue` s’hereten als subtasks. El context d’observació basat en thread locals no. Per tant, un fan-out amb `StructuredTaskScope` necessita passar el snapshot a cada fork:

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

Ara cada subtask crea un child span sota el request span. La cancel·lació continua funcionant perquè `wrap` retorna la mateixa semàntica de callable. `StructuredTaskScope` encara és una API preview a Java 25, així que la mantinc darrere d’una sola classe helper.

## 6. Pas 4: correlacionar logs i traces

Spring Boot posa `traceId` i `spanId` a l’MDC quan hi ha un tracer al classpath. Els faig visibles en el patró de logs:

```properties
logging.pattern.level=%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]
```

Una línia de log té aquest aspecte:

```text
INFO  [orders-api,3f9a1c0b5e2d4a77,7c1d9e42ab35] c.g.orders.OrderService : order accepted
```

Un altre context de negoci, com un tenant o un correlation id d’un header, no forma part de la identitat de la trace. El vinculo com a `ScopedValue` al límit i el copio a l’MDC allà. Faig servir un sol punt de binding per a cada entry point:

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

El `finally` és necessari. El thread de la petició acaba després de la petició, però `captureAll()` també copia l’MDC. Una entrada antiga pot arribar a tots els tasks forked.

## 7. Pas 5: mètriques que encara tenen sentit

Les mètriques antigues dels pools ja no són útils. Les substitueixo per senyals del coll d’ampolla real, que després del canvi normalment no és el nombre de threads.

**Peticions en curs** són la mètrica de saturació més propera:

```java
@Bean
MeterBinder inFlightRequests(HttpRequestCounter counter) {   // a LongAdder incremented in a filter
    return registry -> Gauge.builder("http.server.requests.active", counter, HttpRequestCounter::value)
            .description("Requests currently being served")
            .register(registry);
}
```

**L’ús del pool de connexions** és el límit real. HikariCP ja el publica. Alerto quan `hikaricp.connections.pending > 0` i en el p99 de `hikaricp.connections.acquire`. Un servei amb virtual threads sota càrrega espera aquí, no als threads.

**La latència i els errors downstream** s’han de mesurar per client amb `http.client.requests`. Una dependència lenta ja no apareix com esgotament de threads, així que l’he de mostrar directament.

**Els esdeveniments de pinning** venen de JFR, que es descriu al Pas 8. Si faig servir un bridge de JFR a mètriques, els puc exportar com a counter.

Trec aquests dos tipus de mètriques dels dashboards i alertes:

- `jvm.threads.live` com a senyal de saturació — no inclou virtual threads i es manté plana amb càrrega.
- `tomcat.threads.busy` i `executor.pool.*` en un runtime de virtual threads — aquestes mètriques no hi són o descriuen un scheduler que no dimensiono per petició.

## 8. Pas 6: detectar pinning i starvation

Des de Java 24 (JEP 491), bloquejar dins de `synchronized` ja no fixa un carrier. Per això l’antic escaneig amb `-Djdk.tracePinnedThreads` ha quedat obsolet i la propietat s’ha eliminat. Els frames natius, com JNI o alguns drivers, i el bloqueig durant la inicialització de classes encara poden fixar un carrier.

JFR és la manera suportada de veure aquests esdeveniments:

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

Per a un servei aturat, faig servir un thread dump estructurat. Agrupa els subtasks sota el scope que els ha creat:

```bash
jcmd <pid> Thread.dump_to_file -format=json dump.json
```

Aquesta taula és un bon punt de partida:

| Símptoma | Causa probable | On mirar |
|---|---|---|
| p99 de latència alt, CPU baixa, pocs esdeveniments de pinning | Cua downstream o del pool de connexions | `hikaricp.connections.pending`, `http.client.requests` |
| Molts `jdk.VirtualThreadPinned` amb frames natius | JNI o un driver que bloqueja un carrier | Stack de l’esdeveniment JFR; augmentar `jdk.virtualThreadScheduler.parallelism` temporalment |
| `jdk.VirtualThreadSubmitFailed` | Pool de carriers saturat o sense recursos | Primer el pinning, després el parallelism |
| Traces amb arrels orfes | Un salt de thread sense embolcallar | Pas 3 — trobar l’executor directe |
| Línies de log amb `traceId` buit | El mateix, en la part de l’MDC | Cobertura del decorator del Pas 5.1 |

## 9. Pas 7: provar la instrumentació

La propagació de context pot fallar sense cap error evident, així que faig assertions. `micrometer-observation-test` permet provar el graf d’observacions:

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

Amb el bridge d’OpenTelemetry, un test d’integració també pot comprovar que la trace té una sola arrel:

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

L’assertion `distinct().hasSize(1)` comprova el Pas 5. Falla quan algú envia feina a un executor que no està embolcallat.

## 10. Problemes habituals

- **Embolcalla l’executor al bean, no a cada call site.** Un call site oblidat trenca només algunes traces i és difícil de trobar.
- **Captura el snapshot en el moment correcte.** `captureAll()` ha de córrer en un thread que tingui el context. Dins del task captura un context buit.
- **No deixis `sampling.probability=1.0` en producció.** Un servei amb virtual threads pot gestionar moltes més peticions concurrents. El sampling complet pot arribar al rate limit del collector.
- **Fes servir Observation en lloc de `Tracer` directament.** Observation dona un span, un timer, tags de mètrica i una manera de canviar de backend de tracing.
- **Mantén les claus d’alta cardinalitat fora de les de baixa cardinalitat.** Ids, URLs amb variables de path i missatges de stack no poden convertir-se en tags de mètrica.
- **No alertis pels nombres de threads.** Alerta pel pool de connexions, la mètrica de peticions en curs i la latència downstream.
- **Comprova que `@Async` està cobert.** Només ho està quan existeix un bean `TaskDecorator`. Afegeix un test que comprovi el trace id dins del mètode asíncron.
- **Cobreix els entry points que no són HTTP.** Els listeners Kafka i els jobs programats comencen sense context. Obre una observation en cada entry point o els seus spans seran arrels orfes.

## 11. Conclusió

Els virtual threads no trenquen l’observabilitat per si sols. El problema és el context basat en thread locals combinat amb salts de thread que no el copien.

Faig servir aquests passos:

1. Instrumento amb l’**Observation API** i separo les claus de baixa i alta cardinalitat.
2. Exporto per **OTLP** i faig sampling amb cura en producció.
3. Faig explícit **cada salt de thread** amb beans d’executors embolcallats, un `TaskDecorator` i `snapshot.wrap` per a futures i forks de `StructuredTaskScope`.
4. Poso `traceId` i `spanId` en el patró de logs. Vinculo el context de negoci una vegada per entry point i el netejo.
5. Substitueixo les mètriques dels pools per **peticions en curs, pressió del pool de connexions i latència downstream**.
6. Vigilo `jdk.VirtualThreadPinned` i `jdk.VirtualThreadSubmitFailed` a JFR en lloc de la propietat de pinning eliminada.
7. **Provo la propagació** comprovant que una petició té un sol trace id.

Amb aquests passos, canviar a virtual threads canvia el throughput sense canviar el que puc veure.
