---
title: "Virtual threads en Java 21: guía práctica para plataformas backend"
date: 2026-07-24T00:00:00Z
description: "Una guía práctica de los virtual threads de Java 21: cómo funcionan, cómo activarlos en Spring Boot y qué problemas evitar."
---
# Virtual threads en Java 21: guía práctica para plataformas backend

## 1. Introducción

Las aplicaciones backend suelen usar thread pools limitados. Una petición que espera a una base de datos o a otro servicio HTTP mantiene ocupado un platform thread. Cuando el pool se llena, la latencia crece.

En este artículo mostraré cómo funcionan los virtual threads, cómo activarlos, cómo comprobarlos y qué problemas evitar. Java 21 los convierte en una funcionalidad estable, pero no son un interruptor general de rendimiento.

## 2. Contexto: cómo funcionan los virtual threads

Un virtual thread es un thread ligero gestionado por la JVM. La JVM lo ejecuta en un pool pequeño de platform threads llamados carrier threads. Cuando un virtual thread espera I/O, la JVM lo quita de su carrier. Otro task puede usar ese carrier. Cuando el I/O está listo, el virtual thread vuelve a ejecutarse en un carrier disponible.

```text
Platform threads (classic)          Virtual threads (Java 21)
------------------------            -------------------------
1 request  -> 1 OS thread           1 request  -> 1 virtual thread
blocked I/O holds the OS thread     blocked I/O unmounts the vthread
throughput bound by pool size       throughput bound by concurrent ops
```

El cambio principal está en cómo pienso en los threads:

- Con platform threads, comparto un pool pequeño e intento evitar el bloqueo.
- Con virtual threads, creo un thread por task y puedo bloquear durante el I/O.

El throughput queda limitado por el número de operaciones concurrentes, no solo por el número de threads del sistema operativo.

## 3. Paso 1: crear un virtual thread

La API de bajo nivel está en `Thread`:

```java
// Start a single virtual thread
Thread.startVirtualThread(() -> System.out.println("hello from " + Thread.currentThread()));

// Or with the builder, for naming and lifecycle control
Thread t = Thread.ofVirtual().name("worker-1").start(() -> doWork());
t.join();
```

Para muchos tasks, usa el executor específico. Crea un **virtual thread nuevo por task**. No es un pool fijo:

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    List<Future<String>> futures = ids.stream()
        .map(id -> executor.submit(() -> fetchUser(id)))  // each runs on its own vthread
        .toList();
    for (var f : futures) {
        System.out.println(f.get());
    }
} // try-with-resources waits for all tasks to finish
```

## 4. Paso 2: activarlos en Spring Boot

En Spring Boot 3.2 y posteriores, activa los virtual threads con una propiedad:

```properties
spring.threads.virtual.enabled=true
```

Esto cambia el executor de peticiones del servlet de Tomcat. Cada petición HTTP entrante se ejecuta en su propio virtual thread. Para `@Async` y otros executors, puedo configurar uno directamente:

```java
@Bean
public AsyncTaskExecutor applicationTaskExecutor() {
    return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
}
```

## 5. Paso 3: comprobar que funciona

Puedo imprimir el thread actual desde un request handler y después ejecutar una prueba de carga. Un virtual thread tiene un formato como `VirtualThread[#NN]/runnable@ForkJoinPool-1-worker-M`:

```java
@GetMapping("/whoami")
String whoami() {
    return Thread.currentThread().toString();
    // => VirtualThread[#42]/runnable@ForkJoinPool-1-worker-3
}
```

También puedo enviar muchas peticiones lentas, por ejemplo peticiones que duermen 500ms. Con platform threads, el throughput se detiene en el tamaño del pool. Con virtual threads, el servicio debería gestionar miles de peticiones lentas concurrentes con pocos carriers. Debería superar claramente el límite anterior de `server.tomcat.threads.max`.

## 6. Problemas habituales y cómo detectarlos

- **Pinning.** Un virtual thread que se bloquea dentro de un bloque `synchronized` o una llamada nativa se queda en su carrier. No puede desmontarse y el pool de carriers puede quedarse sin capacidad. En los caminos calientes, prefiere `ReentrantLock` a `synchronized`. En Java 21 LTS, usa este comando para encontrar pinning:

  ```bash
  java -Djdk.tracePinnedThreads=full -jar app.jar
  ```

  JDK 24 relaja la mayor parte del pinning de `synchronized`, pero sigue afectando a Java 21.

- **No hagas pool de virtual threads.** Son baratos y deben ser cortos y crearse por task. Un pool de tamaño fijo vuelve a introducir la contención que los virtual threads debían eliminar. Usa `newVirtualThreadPerTaskExecutor()`.

- **Trabajo limitado por CPU.** Los virtual threads ayudan cuando la mayor parte del tiempo se espera. El trabajo limitado por CPU todavía necesita un pool limitado según los cores disponibles. Un millón de virtual threads haciendo cálculos solo añade coste de planificación.

- **El backpressure se mueve.** Con concurrencia sin límite, el pool de conexiones, un rate limit del servicio externo o un `Semaphore` explícito se convierte en el límite real. Debo dimensionar esos límites porque el número de threads ya no los protege.

  ```java
  Semaphore limit = new Semaphore(100); // cap concurrent downstream calls
  try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
      for (var task : tasks) {
          executor.submit(() -> {
              limit.acquire();
              try { callDownstream(task); } finally { limit.release(); }
          });
      }
  }
  ```

- **Memoria de ThreadLocal.** Millones de virtual threads con estado pesado en `ThreadLocal` pueden consumir demasiada memoria. Para el contexto de la petición, prefiero scoped values o propagación explícita.

## 7. Conclusión

Los virtual threads mejoran la escalabilidad de la concurrencia. No son una aceleración directa para cualquier carga. Ayudan a los servicios Spring Boot que pasan gran parte del tiempo esperando I/O, porque el servicio ya no depende de pools grandes de platform threads.

No mejoran el trabajo limitado por CPU. También trasladan el problema del backpressure a los pools de conexiones y a los límites de los servicios externos.

Antes de activarlos, uso esta lista:

1. Confirmo que la carga está limitada por I/O.
2. Activo `spring.threads.virtual.enabled=true` o configuro el executor.
3. Compruebo que los handlers se ejecutan en `VirtualThread[...]`.
4. Reviso los caminos calientes para encontrar pinning de `synchronized` o código nativo con `-Djdk.tracePinnedThreads=full`.
5. Dimensiono los pools de conexiones y añado backpressure explícito cuando hace falta.

Uso virtual threads cuando las peticiones esperan sobre todo. Primero compruebo el pinning y trato cada dependencia externa como un límite de capacidad.
