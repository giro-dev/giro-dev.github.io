---
title: "Structured concurrency i scoped values a Java 25: substituir ThreadLocal en pipelines de peticions"
date: 2026-08-22T00:00:00Z
description: "Una guia pràctica de StructuredTaskScope i ScopedValue a Java 25: fer fan-out de crides de manera segura i propagar el context de la petició sense ThreadLocal."
---
# Structured concurrency i scoped values a Java 25: substituir ThreadLocal en pipelines de peticions

## 1. Introducció

Els virtual threads fan barat executar molts tasks. Dos patrons antics encara causen problemes en pipelines de peticions: iniciar tasks de fan-out sense un scope compartit i guardar el context de la petició en `ThreadLocal`.

En aquest article mostraré com `StructuredTaskScope` i `ScopedValue` resolen aquests problemes a Java 25. També mostraré com passar de l’API preview de Java 21 i quins problemes encara cal tenir en compte.

Java 25 (LTS) és la base d’aquest article:

| Funcionalitat | Estat a Java 21 | Estat a Java 25 |
|---|---|---|
| Scoped values | Preview (JEP 446) | **Final** (JEP 506) — no cal cap flag |
| Structured concurrency | Preview (JEP 453) | Preview, API redissenyada (JEP 505) |
| Pinning de virtual threads amb `synchronized` | Sí | **No** — corregit a Java 24 (JEP 491) |

## 2. Context: què aporta la structured concurrency

La structured concurrency fa servir la mateixa idea que la programació estructurada. El treball comença dins d’un bloc i acaba dins d’aquest bloc:

```text
Unstructured (executor)                Structured (StructuredTaskScope)
-----------------------                --------------------------------
submit() escapes the method            forks live inside the try block
failure of one task is invisible       first failure cancels siblings
cancellation is manual bookkeeping     scope close() joins everything
stack traces lose the caller           parent/child relation is explicit
```

La regla és senzilla: **quan surt el bloc, cap task forked continua executant-se.** No hi ha tasks orfes ni tasks filtrats.

## 3. Pas 1: fer fan-out amb `StructuredTaskScope`

A Java 25 obro un scope amb la factory estàtica `StructuredTaskScope.open()`. Els constructors públics i les subclasses `ShutdownOnFailure` i `ShutdownOnSuccess` del preview de Java 21 han desaparegut. La factory sense arguments cobreix el cas habitual: esperar que tots els subtasks tinguin èxit i cancel·lar-ho tot al primer error.

```java
record Dashboard(User user, List<Order> orders) {}

Dashboard loadDashboard(long userId) throws InterruptedException {
    try (var scope = StructuredTaskScope.open()) {
        Subtask<User> user = scope.fork(() -> userClient.fetch(userId));
        Subtask<List<Order>> orders = scope.fork(() -> orderClient.fetchFor(userId));

        scope.join();   // throws FailedException if either subtask failed

        return new Dashboard(user.get(), orders.get());
    }
}
```

Comparat amb el preview de Java 21, han canviat tres coses i una part s’ha simplificat:

- `new StructuredTaskScope.ShutdownOnFailure()` ha canviat a `StructuredTaskScope.open()`.
- `scope.join(); scope.throwIfFailed();` ha canviat a un sol `scope.join()`, que llança `StructuredTaskScope.FailedException` amb l’excepció del subtask com a causa.
- `scope.joinUntil(instant)` ha canviat a un timeout en la configuració del scope al Pas 5.

Encara tinc aquests comportaments:

1. Si `userClient.fetch` llança una excepció, el subtask `orders` rep **interrupció immediata**. Això evita una crida downstream innecessària.
2. `join()` només retorna quan tots els subtasks han acabat. `close()` també espera tots els threads, així que res sobreviu al mètode.
3. Només crido `Subtask::get` **després** d’un `join()` correcte. Abans llança `IllegalStateException`.

Cada fork s’executa en el seu propi virtual thread. Per això un fan-out de 50 és tan barat com un fan-out de 2.

Gestiono els errors capturant `FailedException` i analitzant-ne la causa:

```java
try (var scope = StructuredTaskScope.open()) {
    // ...
} catch (StructuredTaskScope.FailedException e) {
    switch (e.getCause()) {
        case IOException ioe -> throw new DownstreamUnavailable(ioe);
        default -> throw e;
    }
}
```

## 4. Pas 2: triar un `Joiner` en lloc d’una política de shutdown

La política ja no viu en una subclass. La passo com un `Joiner` a `open`. Quatre factories cobreixen la majoria de casos:

```java
// "AND": all results required, stream of completed subtasks, fail fast
try (var scope = StructuredTaskScope.open(Joiner.<Quote>allSuccessfulOrThrow())) {
    providers.forEach(p -> scope.fork(() -> p.quote(req)));
    List<Quote> quotes = scope.join().map(Subtask::get).toList();
    return cheapest(quotes);
}

// "AND", heterogeneous subtasks: no scope result, read via Subtask::get
try (var scope = StructuredTaskScope.open(Joiner.awaitAllSuccessfulOrThrow())) { ... }

// "OR" / race: first success wins, losers are cancelled
try (var scope = StructuredTaskScope.open(Joiner.<Quote>anySuccessfulResultOrThrow())) {
    scope.fork(() -> providerA.quote(req));
    scope.fork(() -> providerB.quote(req));
    return scope.join();          // fastest successful quote
}

// "best effort": wait for everything, never throw, inspect each outcome
try (var scope = StructuredTaskScope.open(Joiner.<Quote>awaitAll())) { ... }
```

`Joiner.allUntil(Predicate)` és la base per a polítiques pròpies. Retorna tots els subtasks i cancel·la el scope quan el predicat retorna `true`. Si el predicat no cancel·la mai, puc filtrar el resultat per quedar-me amb els tasks correctes i ignorar la resta:

```java
try (var scope = StructuredTaskScope.open(Joiner.<Quote>allUntil(sub -> false))) {
    providers.forEach(p -> scope.fork(() -> p.quote(req)));
    List<Quote> successes = scope.join()
        .filter(sub -> sub.state() == Subtask.State.SUCCESS)
        .map(Subtask::get)
        .toList();
}
```

Per a una altra política, implemento `Joiner` directament. `onFork` i `onComplete` retornen un `boolean` que pot cancel·lar el scope. `result()` crea el valor que retorna `join()`. Tots dos callbacks s’executen en threads de subtasks, així que la implementació ha de ser thread safe. També creo un `Joiner` nou per a cada scope.

## 5. Pas 3: configurar timeouts, noms i thread factories

L’`open` de dos arguments rep una funció sobre la `Configuration` per defecte. Aquí és on ha anat el deadline de `joinUntil` de Java 21. Ara el timeout cobreix tot el fan-out, inclosa la cancel·lació:

```java
try (var scope = StructuredTaskScope.open(
        Joiner.awaitAllSuccessfulOrThrow(),
        cf -> cf.withName("dashboard")                       // shows up in thread dumps
                .withTimeout(Duration.ofMillis(300))         // cancels the scope on expiry
                .withThreadFactory(Thread.ofVirtual().name("dash-", 0).factory()))) {

    var a = scope.fork(() -> serviceA.call());
    var b = scope.fork(() -> serviceB.call());

    scope.join();            // TimeoutException (as the cause) if the budget is blown
    return merge(a.get(), b.get());
}
```

Posar nom al scope i als seus threads ajuda quan llegeixo un thread dump estructurat:
`jcmd <pid> Thread.dump_to_file -format=json dump.json`. El dump agrupa els subtasks sota el seu owner. Un nom fa més fàcil llegir un bloqueig en producció.

## 6. Pas 4: substituir `ThreadLocal` amb `ScopedValue`

`ScopedValue` és final a Java 25. No necessita el flag `--enable-preview` i és segur exposar-lo en signatures de llibreria. Un valor queda vinculat durant una crida, no pot canviar dins d’aquella crida i es desvincula automàticament quan la crida retorna:

```java
public final class RequestContext {
    public static final ScopedValue<TenantId> TENANT = ScopedValue.newInstance();
    private RequestContext() {}
}

// At the edge (filter / interceptor): bind for the request
ScopedValue.where(RequestContext.TENANT, tenantId)
           .run(() -> chain.doFilter(request, response));

// Deep in the call stack: read it
TenantId tenant = RequestContext.TENANT.get();
```

Puc vincular diversos valors encadenant-los al carrier. Faig servir `call` quan l’operació retorna un valor:

```java
Report report = ScopedValue.where(RequestContext.TENANT, tenantId)
                           .where(RequestContext.CORRELATION_ID, correlationId)
                           .call(() -> reportService.build());
```

La forma de l’API és important. Els helpers estàtics `ScopedValue.runWhere` i `callWhere` dels previews anteriors s’han eliminat. A Java 25 faig servir `where(...).run(...)` i `where(...).call(...)`.

Aquestes són les diferències importants respecte de `ThreadLocal`:

| | `ThreadLocal` | `ScopedValue` |
|---|---|---|
| Durada | fins que s’elimina (o mor el thread) | l’abast dinàmic de `run`/`call` |
| Mutabilitat | `set()` a qualsevol lloc | immutable dins del scope |
| Neteja | `remove()` manual, hi ha leaks si s’oblida | automàtica en sortir del scope |
| Herència | només amb `InheritableThreadLocal`, copia el valor | heretat pels forks de `StructuredTaskScope`, sense còpia |
| Cost per thread | una entrada de mapa per thread | binding compartit i només de lectura |

Quan un binding no està garantit, el llegeixo de manera defensiva:

```java
TenantId tenant = RequestContext.TENANT.orElse(TenantId.SYSTEM);   // Java 25: argument must not be null
if (RequestContext.TENANT.isBound()) { /* ... */ }
```

Una crida niada pot vincular un altre valor. Només amaga el valor exterior dins d’aquella crida i no canvia el binding exterior:

```java
ScopedValue.where(RequestContext.TENANT, otherTenant)
           .run(() -> migrationJob.copyFrom());   // outer binding intact afterwards
```

## 7. Pas 5: combinar les dues funcionalitats

Aquí és on les dues funcionalitats treballen juntes. Cada subtask forked dins del scope hereta els scoped values. No cal copiar-los ni passar arguments:

```java
Report buildReport(TenantId tenantId) throws InterruptedException {
    return ScopedValue.where(RequestContext.TENANT, tenantId).call(() -> {
        try (var scope = StructuredTaskScope.open()) {
            var sales = scope.fork(this::loadSales);       // sees TENANT
            var costs = scope.fork(this::loadCosts);       // sees TENANT

            scope.join();
            return new Report(sales.get(), costs.get());
        }
    });
}

private Sales loadSales() {
    TenantId tenant = RequestContext.TENANT.get();   // inherited, no argument passing
    return salesRepository.findFor(tenant);
}
```

Amb `ThreadLocal`, això necessitava `InheritableThreadLocal` i un executor amb task decoration. També es trencava en silenci quan la feina anava a un altre pool.

## 8. Pas 6: migrar un pipeline existent

Faig servir aquest ordre per a una migració gradual:

1. **Inventario** els `ThreadLocal` i les seves crides a `set`:

   ```bash
   grep -rn "ThreadLocal\|InheritableThreadLocal" src/main/java
   ```

2. **Classifico** cada valor. Un valor amb abast de petició i només de lectura després de l’edge pot fer servir `ScopedValue`. Un acumulador mutable necessita un objecte mutable explícit perquè els scoped values no es poden reassignar.
3. **Vinculo només als edges.** Faig servir un servlet filter, un message listener o el punt d’entrada d’un job programat. Un únic binding per entry point, mai al codi de negoci.
4. **Converteixo els fan-outs.** Substitueixo grups d’`executor.submit(...)` i `Future.get()` per un `StructuredTaskScope`. Trio el `Joiner` del Pas 4.
5. **Elimino els decorators.** Trec `TaskDecorator`s, wrappers que copien MDC i codi d’`InheritableThreadLocal` que només existia per moure context entre threads.
6. **Mantinc l’MDC connectat** quan faig servir SLF4J. Els frameworks de logging encara llegeixen l’MDC, així que el poso des del scoped value a l’edge:

   ```java
   ScopedValue.where(RequestContext.TENANT, tenantId).run(() -> {
       MDC.put("tenant", tenantId.value());
       try { chain.doFilter(request, response); } finally { MDC.clear(); }
   });
   ```

7. **Activo el preview només per a structured concurrency.** `ScopedValue` no el necessita:

   ```xml
   <plugin>
     <artifactId>maven-compiler-plugin</artifactId>
     <configuration>
       <release>25</release>
       <compilerArgs><arg>--enable-preview</arg></compilerArgs>
     </configuration>
   </plugin>
   ```

   ```bash
   java --enable-preview -jar app.jar
   ```

   Les classes preview tenen un marcador de versió menor. Per tant, els tests i el runtime han de fer servir la mateixa versió de funcionalitat del JDK. Si no puc distribuir `--enable-preview`, puc separar la migració: adoptar `ScopedValue` ara i mantenir executors per al fan-out.

### Full de migració: preview de Java 21 → Java 25

```diff
- try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
+ try (var scope = StructuredTaskScope.open()) {
      var a = scope.fork(() -> callA());
      var b = scope.fork(() -> callB());
-     scope.join();
-     scope.throwIfFailed();
+     scope.join();                        // throws FailedException
      return merge(a.get(), b.get());
  }

- new StructuredTaskScope.ShutdownOnSuccess<Quote>()
+ StructuredTaskScope.open(Joiner.<Quote>anySuccessfulResultOrThrow())

- scope.joinUntil(Instant.now().plusMillis(300));
+ StructuredTaskScope.open(joiner, cf -> cf.withTimeout(Duration.ofMillis(300)));

- override handleComplete(Subtask) in a scope subclass
+ implement Joiner.onComplete(Subtask) / result(), or use Joiner.allUntil(predicate)
```

## 9. Problemes habituals i com detectar-los

- **No treguis un `Subtask` fora del scope.** Cridar `get()` després del bloc `try` o guardar un subtask en un camp trenca el model. Llegeix els resultats dins del bloc i retorna valors normals.
- **Fes que els subtasks responguin a interrupcions.** La cancel·lació és cooperativa. Un subtask en una crida no interruptible pot retardar `close()` indefinidament perquè `close()` sempre espera els seus threads. Fes servir clients downstream interruptibles i dona’ls els seus propis timeouts.
- **Fes servir el scope només des del seu thread owner.** El thread owner ha de cridar `fork`, `join` i `close`. `join` només pot executar-se una vegada i `fork` no pot executar-se després de `join`. No guardis un scope en un camp d’un bean; crea’l per a cada crida.
- **Crea un `Joiner` nou per a cada scope.** Un `Joiner` té estat, així que no el comparteixis ni el guardis en cache.
- **No esperis que `ScopedValue` sigui mutable.** No hi ha `set()`. Passa un holder mutable explícit o reorganitza el codi perquè retorni valors.
- **Gestiona lectures sense binding en jobs de fons.** `get()` en un scoped value sense binding llança `NoSuchElementException`. Els jobs programats i els listeners Kafka són entry points separats i necessiten el seu propi binding. Prova aquest camí:

  ```java
  assertThatThrownBy(() -> RequestContext.TENANT.get())
      .isInstanceOf(NoSuchElementException.class);
  ```

  `orElse(null)` també es rebutja a Java 25. Fes servir `isBound()` quan l’absència sigui vàlida.
- **No assumeixis que el pinning continua sent el problema principal.** Des de Java 24 (JEP 491), bloquejar en `synchronized` ja no fixa un virtual thread. Els frames natius i el bloqueig en inicialització de classes encara poden fixar-lo. Vigila l’esdeveniment JFR `jdk.VirtualThreadPinned` en lloc de la propietat del sistema eliminada.
- **El backpressure continua sent necessari.** El fan-out estructurat fa fàcil crear concurrència. Els pools de connexions i els rate limits downstream continuen sent el límit. Fes servir un `Semaphore` o un pool limitat quan calgui.
- **Espera canvis en l’API preview.** La structured concurrency és al cinquè preview a Java 25 i se’n preveu un sisè per a Java 26. Posa el fan-out darrere d’un helper intern petit com `Parallel.all(...)`, perquè una actualització canviï una classe i no molts call sites.

## 10. Conclusió

Les dues funcionalitats resolen parts diferents del mateix problema. `StructuredTaskScope` limita la concurrència per scope: els errors cancel·len els germans, un timeout cobreix tot el fan-out i cap task sobreviu al mètode. `ScopedValue`, que és final a Java 25, fa que el context sigui immutable i limitat. Les metadades de la petició arriben als subtasks forked sense decorators i sense estat per thread per a cada virtual thread.

Abans d’adoptar-les, faig servir aquesta llista:

1. Passo a Java 25 (LTS). `ScopedValue` no necessita flag i `synchronized` ja no fixa carriers.
2. Passo els `ThreadLocal` de petició i només de lectura a `ScopedValue`, vinculats només als entry points.
3. Substitueixo grups d’`executor.submit` i `Future.get()` per `StructuredTaskScope.open(...)` i un `Joiner` explícit.
4. Passo els timeouts per crida a un `withTimeout` del scope i poso nom als scopes perquè els thread dumps siguin llegibles.
5. Connecto l’MDC de logs a l’edge i elimino els hacks d’herència i els task decorators.
6. Provo el camí sense binding per a jobs i listeners, i provo la cancel·lació.
7. Mantinc l’API preview de structured concurrency darrere d’una sola classe helper perquè les actualitzacions siguin petites.

Els virtual threads fan barata la concurrència. La structured concurrency i els scoped values la fan més segura. Els faig servir quan una petició fa fan-out i vinculo el context explícitament a cada entry point.
