---
title: "Un ejemplo sencillo de Spring Integration"
date: 2026-09-18T00:00:00Z
description: "Un ejemplo de introducción para usar Spring Integration en una aplicación hello world sencilla."
---
# Un ejemplo sencillo de Spring Integration

## 1. Introducción

Spring Integration es un framework que ayuda a conectar sistemas dentro de una aplicación Spring. Usa las mismas ideas que los Enterprise Integration Patterns: mensajes, canales y endpoints. Si ya conoces Spring Boot, resulta familiar después de unos minutos.

En este artículo mostraré un ejemplo mínimo de «hello world». Enviaremos un nombre a través de un gateway, lo pasaremos por un canal y obtendremos un saludo.

## 2. Qué es Spring Integration

En lugar de llamar a métodos directamente, envías mensajes. Un mensaje tiene un payload y headers. El payload son los datos reales. Los headers son metadatos, como un id de correlación o una marca de tiempo.

Tres piezas principales:

- **Message**: el elemento que viaja. Tiene un payload y headers.
- **Channel**: el conducto donde los mensajes esperan o se mueven. Piensa en él como una `Queue` o una `List`.
- **Endpoint**: la pieza que hace algo con el mensaje. Por ejemplo, un `ServiceActivator` es un endpoint que ejecuta un método.

## 3. Dependencias

Añade `spring-integration-core` al proyecto. Si usas Spring Boot, el starter ya lo incluye cuando añades los módulos correctos. Para un ejemplo simple:

```xml
<dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-core</artifactId>
</dependency>
```

Si usas Gradle:

```groovy
implementation 'org.springframework.integration:spring-integration-core'
```

## 4. El ejemplo Hello World

Queremos llamar a `sayHello("World")` y obtener `"Hello, World!"`. La llamada pasa por un gateway, un canal y un service activator.

### 4.1 La configuración

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

Qué está pasando aquí:

- `helloChannel()` crea un `DirectChannel`. Es un canal sencillo en memoria que pasa el mensaje al siguiente endpoint inmediatamente.
- `handle` es un `ServiceActivator`. Escucha `helloChannel`, recibe el payload como `String` y devuelve un `String` nuevo.
- `HelloGateway` es un `MessagingGateway`. Spring crea una implementación. Envía la entrada al canal y devuelve lo que recibe.

### 4.2 Cómo fluye el mensaje

1. Llamas a `gateway.sayHello("World")`.
2. El gateway envuelve `"World"` en un `Message` y lo pone en `helloChannel`.
3. El `ServiceActivator` toma el payload y ejecuta `handle("World")`.
4. El valor devuelto `"Hello, World!"` se convierte en un mensaje de respuesta.
5. El gateway devuelve el payload de respuesta al llamador.

El código que lo usa no parece diferente de una llamada normal a un método:

```java
HelloIntegrationConfig.HelloGateway gateway = ...;
String greeting = gateway.sayHello("World");
System.out.println(greeting); // Hello, World!
```

## 5. Probar el ejemplo

Un test pequeño con JUnit 5 y Spring:

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

Si las dependencias de test no están disponibles en tu proyecto, también puedes cargar el contexto con `@SpringBootTest` o `AnnotationConfigApplicationContext`.

## 6. Algunas cosas a tener en cuenta

- `DirectChannel` es síncrono. El llamador espera hasta que termina el service activator. Esto está bien para ejemplos sencillos.
- Si quieres gestión asíncrona, usa un `QueueChannel`.
- El tipo del payload en el service activator puede ser un `String`, un `Message<String>` o tu propio objeto. Spring lo resuelve.
- `@MessagingGateway` evita tener que construir y enviar mensajes manualmente.

## 7. Conclusión

Este es el ejemplo útil más pequeño de Spring Integration. Tienes un gateway, un canal y un service activator. El gateway es la entrada, el canal es el conducto y el service activator hace el trabajo. Cuando esto está claro, es fácil añadir transformers, filters, routers y adapters.

En el próximo artículo mostraré cómo añadir un transformer y un router al mismo flujo.
