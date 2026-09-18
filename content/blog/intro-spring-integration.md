---
title: "A Simple Spring Integration Example"
date: 2026-09-18T00:00:00Z
description: "An introduction example of how to use Spring Integration to build a simple hello world application."
---
# A Simple Spring Integration Example

## 1. Overview

Spring Integration is a framework that helps you connect systems inside a Spring application. It uses the same ideas as Enterprise Integration Patterns: messages, channels, and endpoints. If you already know Spring Boot, it feels familiar after a few minutes.

In this post I will show a minimal "hello world" example. We will send a name through a gateway, pass it over a channel, and get a greeting back.

## 2. What Is Spring Integration

Instead of calling methods directly, you send messages. A message has a payload and some headers. The payload is the actual data. The headers are metadata such as a correlation id or a timestamp.

Three core pieces:

- **Message**: the thing that travels. It has a payload and headers.
- **Channel**: the pipe where messages wait or move. Think of it as a `Queue` or a `List`.
- **Endpoint**: the piece that does something with the message. For example a `ServiceActivator` is an endpoint that runs a method.

## 3. Dependencies

Add `spring-integration-core` to your project. If you use Spring Boot, the starter already brings it if you include the right modules. For a plain example:

```xml
<dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-core</artifactId>
</dependency>
```

If you use Gradle:

```groovy
implementation 'org.springframework.integration:spring-integration-core'
```

## 4. The Hello World Example

We want to call `sayHello("World")` and get back `"Hello, World!"`. The call goes through a gateway, a channel, and a service activator.

### 4.1 The Configuration

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

What is going on here:

- `helloChannel()` creates a `DirectChannel`. This is a simple in-memory channel that passes the message to the next endpoint immediately.
- `handle` is a `ServiceActivator`. It listens on `helloChannel`, receives the payload as a `String`, and returns a new `String`.
- `HelloGateway` is a `MessagingGateway`. Spring creates an implementation for us. It sends the input to the channel and returns whatever comes back.

### 4.2 How the Message Flows

1. You call `gateway.sayHello("World")`.
2. The gateway wraps `"World"` into a `Message` and puts it on `helloChannel`.
3. The `ServiceActivator` takes the message payload and runs `handle("World")`.
4. The return value `"Hello, World!"` is wrapped into a reply message.
5. The gateway returns the reply payload to the caller.

The code that uses it does not look different from a normal method call:

```java
HelloIntegrationConfig.HelloGateway gateway = ...;
String greeting = gateway.sayHello("World");
System.out.println(greeting); // Hello, World!
```

## 5. Testing the Example

A small test with JUnit 5 and Spring:

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

If the test dependencies are not available in your project, you can also load the context with `@SpringBootTest` or `AnnotationConfigApplicationContext`.

## 6. A Few Things to Notice

- `DirectChannel` is synchronous. The caller waits until the service activator finishes. This is fine for simple examples.
- If you want asynchronous handling, use a `QueueChannel` instead.
- The payload type in the service activator can be a `String`, a `Message<String>`, or your own object. Spring figures it out.
- `@MessagingGateway` removes the boilerplate of building and sending messages by hand.

## 7. Conclusion

That is the smallest useful Spring Integration example. You have a gateway, a channel, and a service activator. The gateway is the entry point, the channel is the pipe, and the service activator does the work. Once this is clear, it is easy to add more endpoints such as transformers, filters, routers, and adapters.

In the next post I will show how to add a transformer and a router to the same flow.
