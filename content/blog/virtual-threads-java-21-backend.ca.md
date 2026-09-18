---
title: "Virtual threads a Java 21: guia pràctica per a plataformes backend"
date: 2026-07-24T00:00:00Z
description: "Una guia pràctica dels virtual threads de Java 21: com funcionen, com activar-los a Spring Boot i quins problemes cal evitar."
---
# Virtual threads a Java 21: guia pràctica per a plataformes backend

## 1. Introducció

Les aplicacions backend sovint fan servir thread pools limitats. Una petició que espera una base de dades o un altre servei HTTP manté ocupat un platform thread. Quan el pool s’omple, la latència creix.

En aquest article mostraré com funcionen els virtual threads, com activar-los, com comprovar-los i quins problemes cal evitar. Java 21 els converteix en una funcionalitat estable, però no són un interruptor general de rendiment.

## 2. Context: com funcionen els virtual threads

Un virtual thread és un thread lleuger gestionat per la JVM. La JVM l’executa en un pool petit de platform threads anomenats carrier threads. Quan un virtual thread espera I/O, la JVM el treu del seu carrier. Un altre task pot fer servir aquell carrier. Quan l’I/O està preparat, el virtual thread torna a executar-se en un carrier disponible.

```text
Platform threads (classic)          Virtual threads (Java 21)
------------------------            -------------------------
1 request  -> 1 OS thread           1 request  -> 1 virtual thread
blocked I/O holds the OS thread     blocked I/O unmounts the vthread
throughput bound by pool size       throughput bound by concurrent ops
```

El canvi principal és com penso en els threads:

- Amb platform threads, comparteixo un pool petit i intento evitar el bloqueig.
- Amb virtual threads, creo un thread per task i puc bloquejar durant l’I/O.

El throughput queda limitat pel nombre d’operacions concurrents, no només pel nombre de threads del sistema operatiu.

## 3. Pas 1: crear un virtual thread

L’API de baix nivell és a `Thread`:

```java
// Start a single virtual thread
Thread.startVirtualThread(() -> System.out.println("hello from " + Thread.currentThread()));

// Or with the builder, for naming and lifecycle control
Thread t = Thread.ofVirtual().name("worker-1").start(() -> doWork());
t.join();
```

Per a molts tasks, fes servir l’executor específic. Crea un **virtual thread nou per task**. No és un pool fix:

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

## 4. Pas 2: activar-los a Spring Boot

A Spring Boot 3.2 i versions posteriors, activa els virtual threads amb una propietat:

```properties
spring.threads.virtual.enabled=true
```

Això canvia l’executor de peticions del servlet de Tomcat. Cada petició HTTP entrant s’executa en el seu propi virtual thread. Per a `@Async` i altres executors, en puc configurar un directament:

```java
@Bean
public AsyncTaskExecutor applicationTaskExecutor() {
    return new TaskExecutorAdapter(Executors.newVirtualThreadPerTaskExecutor());
}
```

## 5. Pas 3: comprovar que funciona

Puc imprimir el thread actual des d’un request handler i després executar una prova de càrrega. Un virtual thread té un format com `VirtualThread[#NN]/runnable@ForkJoinPool-1-worker-M`:

```java
@GetMapping("/whoami")
String whoami() {
    return Thread.currentThread().toString();
    // => VirtualThread[#42]/runnable@ForkJoinPool-1-worker-3
}
```

També puc enviar moltes peticions lentes, per exemple peticions que dormen 500ms. Amb platform threads, el throughput s’atura a la mida del pool. Amb virtual threads, el servei hauria de gestionar milers de peticions lentes concurrents amb pocs carriers. Hauria de superar molt el límit anterior de `server.tomcat.threads.max`.

## 6. Problemes habituals i com detectar-los

- **Pinning.** Un virtual thread que es bloqueja dins d’un bloc `synchronized` o d’una crida nativa es queda al seu carrier. No es pot desmuntar i el pool de carriers es pot quedar sense capacitat. En camins calents, prefereix `ReentrantLock` a `synchronized`. A Java 21 LTS, fes servir aquesta ordre per trobar pinning:

  ```bash
  java -Djdk.tracePinnedThreads=full -jar app.jar
  ```

  JDK 24 relaxa la majoria del pinning de `synchronized`, però encara afecta Java 21.

- **No facis pool de virtual threads.** Són barats i han de ser curts i creats per task. Un pool de mida fixa torna a introduir la contenció que els virtual threads havien de treure. Fes servir `newVirtualThreadPerTaskExecutor()`.

- **Treball limitat per CPU.** Els virtual threads ajuden quan la major part del temps s’espera. El treball limitat per CPU encara necessita un pool limitat segons els cores disponibles. Un milió de virtual threads fent càlculs només afegeix cost de planificació.

- **El backpressure es mou.** Amb concurrència sense límit, el pool de connexions, un rate limit del servei extern o un `Semaphore` explícit es converteix en el límit real. He de dimensionar aquests límits perquè el nombre de threads ja no els protegeix.

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

- **Memòria de ThreadLocal.** Milions de virtual threads amb estat pesat a `ThreadLocal` poden consumir massa memòria. Per al context de la petició, prefereixo scoped values o propagació explícita.

## 7. Conclusió

Els virtual threads milloren l’escalabilitat de la concurrència. No són una acceleració directa per a qualsevol càrrega. Ajuden els serveis Spring Boot que passen gran part del temps esperant I/O, perquè el servei ja no depèn de pools grans de platform threads.

No milloren el treball limitat per CPU. També mouen el problema del backpressure als pools de connexions i als límits dels serveis externs.

Abans d’activar-los, faig servir aquesta llista:

1. Confirmo que la càrrega és limitada per I/O.
2. Activo `spring.threads.virtual.enabled=true` o configuro l’executor.
3. Comprovo que els handlers s’executen en `VirtualThread[...]`.
4. Reviso els camins calents per trobar pinning de `synchronized` o codi natiu amb `-Djdk.tracePinnedThreads=full`.
5. Dimensiono els pools de connexions i afegeixo backpressure explícit quan cal.

Faig servir virtual threads quan les peticions esperen sobretot. Primer comprovo el pinning i tracto cada dependència externa com un límit de capacitat.
