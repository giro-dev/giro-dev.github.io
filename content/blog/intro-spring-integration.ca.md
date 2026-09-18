---
title: "Un exemple senzill de Spring Integration"
date: 2026-09-18T00:00:00Z
description: "Un exemple d’introducció per fer servir Spring Integration en una aplicació hello world senzilla."
---
# Un exemple senzill de Spring Integration

## 1. Introducció

Spring Integration és un framework que ajuda a connectar sistemes dins d’una aplicació Spring. Usa les mateixes idees que els Enterprise Integration Patterns: missatges, canals i endpoints. Si ja coneixes Spring Boot, resulta familiar al cap de pocs minuts.

En aquest article mostraré un exemple mínim de «hello world». Enviarem un nom a través d’un gateway, el passarem per un canal i obtindrem una salutació.

## 2. Què és Spring Integration

En lloc de cridar mètodes directament, envies missatges. Un missatge té un payload i headers. El payload són les dades reals. Els headers són metadades, com un id de correlació o una marca de temps.

Tres peces principals:

- **Message**: l’element que viatja. Té un payload i headers.
- **Channel**: el conducte on els missatges esperen o es mouen. Pensa-hi com una `Queue` o una `List`.
- **Endpoint**: la peça que fa alguna cosa amb el missatge. Per exemple, un `ServiceActivator` és un endpoint que executa un mètode.

## 3. Dependències

Afegeix `spring-integration-core` al projecte. Si fas servir Spring Boot, el starter ja el porta quan inclous els mòduls correctes. Per a un exemple sense més:

```xml
<dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-core</artifactId>
</dependency>
```

Si fas servir Gradle:

```groovy
implementation 'org.springframework.integration:spring-integration-core'
```

## 4. L’exemple Hello World

Volem cridar `sayHello("World")` i obtenir `"Hello, World!"`. La crida passa per un gateway, un canal i un service activator.

### 4.1 La configuració

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.annotation.IntegrationComponentScan;
import org.springframework.integration.annotation.MessagingGateway;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.integration.config.EnableIntegration;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.messaging.MessageChannel;

@Configuration
@EnableIntegration
@IntegrationComponentScan
public class HelloIntegrationConfig {

    @Bean
    public MessageChannel helloChannel() {
        return new DirectChannel();
    }

    @ServiceActivator(inputChannel = "helloChannel")
    public String handle(String name) {
        return "Hello, " + name + "!";
    }

    @MessagingGateway
    public interface HelloGateway {
        String sayHello(String name);
    }
}
```

Què està passant aquí:

- `helloChannel()` crea un `DirectChannel`. És un canal senzill en memòria que passa el missatge al següent endpoint immediatament.
- `handle` és un `ServiceActivator`. Escolta `helloChannel`, rep el payload com a `String` i retorna un `String` nou.
- `HelloGateway` és un `MessagingGateway`. Spring en crea una implementació. Envia l’entrada al canal i retorna el que rep.

### 4.2 Com flueix el missatge

1. Crides `gateway.sayHello("World")`.
2. El gateway embolcalla `"World"` en un `Message` i el posa a `helloChannel`.
3. El `ServiceActivator` agafa el payload i executa `handle("World")`.
4. El valor de retorn `"Hello, World!"` es converteix en un missatge de resposta.
5. El gateway retorna el payload de la resposta a qui ha fet la crida.

El codi que el fa servir no sembla diferent d’una crida normal a un mètode:

```java
HelloIntegrationConfig.HelloGateway gateway = ...;
String greeting = gateway.sayHello("World");
System.out.println(greeting); // Hello, World!
```

## 5. Provar l’exemple

Un test petit amb JUnit 5 i Spring:

```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.integration.test.context.SpringIntegrationTest;
import org.springframework.test.context.junit.jupiter.SpringJUnitConfig;

import static org.junit.jupiter.api.Assertions.assertEquals;

@SpringJUnitConfig(HelloIntegrationConfig.class)
class HelloIntegrationTest {

    private HelloIntegrationConfig.HelloGateway gateway = new HelloIntegrationConfig.HelloGateway();

    @Test
    void shouldReturnGreeting() {
        String result = gateway.sayHello("World");
        assertEquals("Hello, World!", result);
    }
}
```

Si les dependències de test no estan disponibles al projecte, també pots carregar el context amb `@SpringBootTest` o `AnnotationConfigApplicationContext`.

## 6. Algunes coses a tenir en compte

- `DirectChannel` és síncron. Qui fa la crida espera que acabi el service activator. Això va bé per a exemples senzills.
- Si vols gestió asíncrona, fes servir un `QueueChannel`.
- El tipus del payload al service activator pot ser un `String`, un `Message<String>` o el teu propi objecte. Spring ho resol.
- `@MessagingGateway` evita haver de construir i enviar missatges manualment.

## 7. Conclusió

Aquest és l’exemple útil més petit de Spring Integration. Tens un gateway, un canal i un service activator. El gateway és l’entrada, el canal és el conducte i el service activator fa la feina. Quan això queda clar, és fàcil afegir transformers, filters, routers i adapters.

En el proper article mostraré com afegir un transformer i un router al mateix flux.
