---
title: "Structured concurrency y scoped values en Java 25: sustituir ThreadLocal en pipelines de peticiones"
date: 2026-08-22T00:00:00Z
description: "Una guía práctica de StructuredTaskScope y ScopedValue en Java 25: hacer fan-out de llamadas de forma segura y propagar el contexto de la petición sin ThreadLocal."
---
# Structured concurrency y scoped values en Java 25: sustituir ThreadLocal en pipelines de peticiones

## 1. Introducción

Los virtual threads hacen barato ejecutar muchos tasks. Dos patrones antiguos todavía causan problemas en pipelines de peticiones: iniciar tasks de fan-out sin un scope compartido y guardar el contexto de la petición en `ThreadLocal`.

En este artículo mostraré cómo `StructuredTaskScope` y `ScopedValue` resuelven estos problemas en Java 25. También mostraré cómo pasar de la API preview de Java 21 y qué problemas siguen necesitando atención.

Java 25 (LTS) es la base de este artículo:

| Funcionalidad | Estado en Java 21 | Estado en Java 25 |
|---|---|---|
| Scoped values | Preview (JEP 446) | **Final** (JEP 506) — no necesita flag |
| Structured concurrency | Preview (JEP 453) | Preview, API rediseñada (JEP 505) |
| Pinning de virtual threads con `synchronized` | Sí | **No** — corregido en Java 24 (JEP 491) |

## 2. Contexto: qué aporta la structured concurrency

La structured concurrency usa la misma idea que la programación estructurada. El trabajo empieza dentro de un bloque y termina dentro de ese bloque:

```text
Unstructured (executor)                Structured (StructuredTaskScope)
-----------------------                --------------------------------
submit() escapes the method            forks live inside the try block
failure of one task is invisible       first failure cancels siblings
cancellation is manual bookkeeping     scope close() joins everything
stack traces lose the caller           parent/child relation is explicit
```

La regla es sencilla: **cuando sale el bloque, ningún task forked sigue ejecutándose.** No hay tasks huérfanos ni tasks filtrados.

## 3. Paso 1: hacer fan-out con `StructuredTaskScope`

En Java 25 abro un scope con la factory estática `StructuredTaskScope.open()`. Los constructores públicos y las subclases `ShutdownOnFailure` y `ShutdownOnSuccess` del preview de Java 21 han desaparecido. La factory sin argumentos cubre el caso habitual: esperar a que todos los subtasks terminen bien y cancelar todo ante el primer error.

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

Comparado con el preview de Java 21, han cambiado tres cosas y una parte es más sencilla:

- `new StructuredTaskScope.ShutdownOnFailure()` ha cambiado a `StructuredTaskScope.open()`.
- `scope.join(); scope.throwIfFailed();` ha cambiado a un solo `scope.join()`, que lanza `StructuredTaskScope.FailedException` con la excepción del subtask como causa.
- `scope.joinUntil(instant)` ha cambiado a un timeout en la configuración del scope en el Paso 5.

Sigo teniendo estos comportamientos:

1. Si `userClient.fetch` lanza una excepción, el subtask `orders` recibe una **interrupción inmediata**. Así se evita una llamada downstream innecesaria.
2. `join()` solo devuelve cuando todos los subtasks han terminado. `close()` también espera a todos los threads, así que nada sobrevive al método.
3. Solo llamo a `Subtask::get` **después** de un `join()` correcto. Antes lanza `IllegalStateException`.

Cada fork se ejecuta en su propio virtual thread. Por eso un fan-out de 50 es tan barato como uno de 2.

Gestiono los errores capturando `FailedException` y analizando su causa:

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

## 4. Paso 2: elegir un `Joiner` en lugar de una política de shutdown

La política ya no vive en una subclase. La paso como un `Joiner` a `open`. Cuatro factories cubren la mayoría de los casos:

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

`Joiner.allUntil(Predicate)` es la base para políticas propias. Devuelve todos los subtasks y cancela el scope cuando el predicado devuelve `true`. Si el predicado nunca cancela, puedo filtrar el resultado para conservar los tasks correctos e ignorar el resto:

```java
try (var scope = StructuredTaskScope.open(Joiner.<Quote>allUntil(sub -> false))) {
    providers.forEach(p -> scope.fork(() -> p.quote(req)));
    List<Quote> successes = scope.join()
        .filter(sub -> sub.state() == Subtask.State.SUCCESS)
        .map(Subtask::get)
        .toList();
}
```

Para otra política, implemento `Joiner` directamente. `onFork` y `onComplete` devuelven un `boolean` que puede cancelar el scope. `result()` crea el valor que devuelve `join()`. Los dos callbacks se ejecutan en threads de subtasks, así que la implementación debe ser thread safe. También creo un `Joiner` nuevo para cada scope.

## 5. Paso 3: configurar timeouts, nombres y thread factories

El `open` de dos argumentos recibe una función sobre la `Configuration` por defecto. Aquí se movió el deadline de `joinUntil` de Java 21. Ahora el timeout cubre todo el fan-out, incluida la cancelación:

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

Poner nombre al scope y a sus threads ayuda cuando leo un thread dump estructurado:
`jcmd <pid> Thread.dump_to_file -format=json dump.json`. El dump agrupa los subtasks bajo su owner. Un nombre hace más fácil leer un bloqueo en producción.

## 6. Paso 4: sustituir `ThreadLocal` con `ScopedValue`

`ScopedValue` es final en Java 25. No necesita el flag `--enable-preview` y es seguro exponerlo en signatures de librería. Un valor queda vinculado durante una llamada, no puede cambiar dentro de esa llamada y se desvincula automáticamente cuando la llamada termina:

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

Puedo vincular varios valores encadenándolos en el carrier. Uso `call` cuando la operación devuelve un valor:

```java
Report report = ScopedValue.where(RequestContext.TENANT, tenantId)
                           .where(RequestContext.CORRELATION_ID, correlationId)
                           .call(() -> reportService.build());
```

La forma de la API es importante. Los helpers estáticos `ScopedValue.runWhere` y `callWhere` de previews anteriores se eliminaron. En Java 25 uso `where(...).run(...)` y `where(...).call(...)`.

Estas son las diferencias importantes frente a `ThreadLocal`:

| | `ThreadLocal` | `ScopedValue` |
|---|---|---|
| Duración | hasta que se elimina (o muere el thread) | el ámbito dinámico de `run`/`call` |
| Mutabilidad | `set()` en cualquier sitio | inmutable dentro del scope |
| Limpieza | `remove()` manual, hay leaks si se olvida | automática al salir del scope |
| Herencia | solo con `InheritableThreadLocal`, copia el valor | heredado por forks de `StructuredTaskScope`, sin copia |
| Coste por thread | una entrada de mapa por thread | binding compartido y de solo lectura |

Cuando un binding no está garantizado, lo leo de forma defensiva:

```java
TenantId tenant = RequestContext.TENANT.orElse(TenantId.SYSTEM);   // Java 25: argument must not be null
if (RequestContext.TENANT.isBound()) { /* ... */ }
```

Una llamada anidada puede vincular otro valor. Solo oculta el valor exterior dentro de esa llamada y no cambia el binding exterior:

```java
ScopedValue.where(RequestContext.TENANT, otherTenant)
           .run(() -> migrationJob.copyFrom());   // outer binding intact afterwards
```

## 7. Paso 5: combinar las dos funcionalidades

Aquí es donde las dos funcionalidades trabajan juntas. Cada subtask forked dentro del scope hereda los scoped values. No hay que copiarlos ni pasar argumentos:

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

Con `ThreadLocal`, esto necesitaba `InheritableThreadLocal` y un executor con task decoration. También se rompía en silencio cuando el trabajo iba a otro pool.

## 8. Paso 6: migrar un pipeline existente

Uso este orden para una migración gradual:

1. **Inventario** los `ThreadLocal` y sus llamadas a `set`:

   ```bash
   grep -rn "ThreadLocal\|InheritableThreadLocal" src/main/java
   ```

2. **Clasifico** cada valor. Un valor con alcance de petición y solo de lectura después del edge puede usar `ScopedValue`. Un acumulador mutable necesita un objeto mutable explícito porque los scoped values no se pueden reasignar.
3. **Vinculo solo en los edges.** Uso un servlet filter, un message listener o el punto de entrada de un job programado. Un solo binding por entry point, nunca en código de negocio.
4. **Convierto los fan-outs.** Sustituyo grupos de `executor.submit(...)` y `Future.get()` por un `StructuredTaskScope`. Elijo el `Joiner` del Paso 4.
5. **Elimino los decorators.** Quito `TaskDecorator`s, wrappers que copian MDC y código de `InheritableThreadLocal` que solo existía para mover contexto entre threads.
6. **Mantengo el MDC conectado** cuando uso SLF4J. Los frameworks de logging siguen leyendo el MDC, así que lo establezco desde el scoped value en el edge:

   ```java
   ScopedValue.where(RequestContext.TENANT, tenantId).run(() -> {
       MDC.put("tenant", tenantId.value());
       try { chain.doFilter(request, response); } finally { MDC.clear(); }
   });
   ```

7. **Activo el preview solo para structured concurrency.** `ScopedValue` no lo necesita:

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

   Las clases preview tienen un marcador de versión menor. Por eso los tests y el runtime deben usar la misma versión de funcionalidad del JDK. Si no puedo distribuir `--enable-preview`, puedo separar la migración: adoptar `ScopedValue` ahora y mantener executors para el fan-out.

### Hoja de migración: preview de Java 21 → Java 25

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

## 9. Problemas habituales y cómo detectarlos

- **No saques un `Subtask` fuera del scope.** Llamar a `get()` después del bloque `try` o guardar un subtask en un campo rompe el modelo. Lee los resultados dentro del bloque y devuelve valores normales.
- **Haz que los subtasks respondan a interrupciones.** La cancelación es cooperativa. Un subtask en una llamada no interrumpible puede retrasar `close()` indefinidamente porque `close()` siempre espera a sus threads. Usa clientes downstream interrumpibles y dales sus propios timeouts.
- **Usa el scope solo desde su thread owner.** El thread owner debe llamar a `fork`, `join` y `close`. `join` solo puede ejecutarse una vez y `fork` no puede ejecutarse después de `join`. No guardes un scope en un campo de un bean; créalo para cada llamada.
- **Crea un `Joiner` nuevo para cada scope.** Un `Joiner` tiene estado, así que no lo compartas ni lo guardes en cache.
- **No esperes que `ScopedValue` sea mutable.** No existe `set()`. Pasa un holder mutable explícito o reorganiza el código para devolver valores.
- **Gestiona lecturas sin binding en jobs de fondo.** `get()` en un scoped value sin binding lanza `NoSuchElementException`. Los jobs programados y los listeners Kafka son entry points separados y necesitan su propio binding. Prueba este camino:

  ```java
  assertThatThrownBy(() -> RequestContext.TENANT.get())
      .isInstanceOf(NoSuchElementException.class);
  ```

  `orElse(null)` también se rechaza en Java 25. Usa `isBound()` cuando la ausencia sea válida.
- **No asumas que el pinning sigue siendo el problema principal.** Desde Java 24 (JEP 491), bloquear en `synchronized` ya no fija un virtual thread. Los frames nativos y el bloqueo durante la inicialización de clases todavía pueden fijarlo. Vigila el evento JFR `jdk.VirtualThreadPinned` en lugar de la propiedad del sistema eliminada.
- **El backpressure sigue siendo necesario.** El fan-out estructurado facilita crear concurrencia. Los pools de conexiones y los rate limits downstream siguen siendo el límite. Usa un `Semaphore` o un pool limitado cuando sea necesario.
- **Espera cambios en la API preview.** La structured concurrency está en su quinto preview en Java 25 y se espera un sexto en Java 26. Pon el fan-out detrás de un helper interno pequeño como `Parallel.all(...)`, para que una actualización cambie una clase y no muchos call sites.

## 10. Conclusión

Las dos funcionalidades resuelven partes diferentes del mismo problema. `StructuredTaskScope` limita la concurrencia por scope: los errores cancelan a los hermanos, un timeout cubre todo el fan-out y ningún task sobrevive al método. `ScopedValue`, que es final en Java 25, hace que el contexto sea inmutable y limitado. Los metadatos de la petición llegan a los subtasks forked sin decorators y sin estado por thread para cada virtual thread.

Antes de adoptarlas, uso esta lista:

1. Paso a Java 25 (LTS). `ScopedValue` no necesita flag y `synchronized` ya no fija carriers.
2. Paso los `ThreadLocal` de petición y solo de lectura a `ScopedValue`, vinculados solo en entry points.
3. Sustituyo grupos de `executor.submit` y `Future.get()` por `StructuredTaskScope.open(...)` y un `Joiner` explícito.
4. Paso los timeouts por llamada a un `withTimeout` del scope y doy nombre a los scopes para que los thread dumps sigan siendo legibles.
5. Conecto el MDC de logs en el edge y elimino los hacks de herencia y los task decorators.
6. Pruebo el camino sin binding para jobs y listeners, y pruebo la cancelación.
7. Mantengo la API preview de structured concurrency detrás de una sola clase helper para que las actualizaciones sean pequeñas.

Los virtual threads hacen barata la concurrencia. La structured concurrency y los scoped values la hacen más segura. Los uso cuando una petición hace fan-out y vinculo el contexto explícitamente en cada entry point.
