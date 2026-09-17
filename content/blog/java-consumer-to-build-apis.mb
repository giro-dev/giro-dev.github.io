---
title: "Building a Fluent Java API with Consumer and BiConsumer"
date: 2026-09-17T00:00:00Z
description: "How to use Consumer and BiConsumer to build miningful and fluent Java APIs."
---


# Building a Fluent Java API with Consumer and BiConsumer

## 1. Overview

When I build small internal libraries I try to keep the public API easy to read. Lately I have been using `Consumer` and `BiConsumer` a lot to let callers configure objects without writing long anonymous classes. The result looks like a builder, but each step is just a lambda that sets something up.

In this post I will show a made-up example: a small `MenuBuilder` that lets you add menu items. We will see where `Consumer` fits, where `BiConsumer` fits, and how the two combine into one fluent chain.

## 2. The Idea: A Fluent Builder

A fluent API is one where method calls can be chained:

```java
List<MenuItem> menu = new MenuBuilder()
    .addItem("Home", item -> item.setIcon("home"))
    .addItem("Profile", item -> item.setIcon("user"))
    .build();
```

To make this work, each method returns `this`. The lambda is the interesting part. Instead of passing five parameters, the caller gets a `MenuItem` instance and configures it however they want.

## 3. Consumer: One-Argument Configuration

`Consumer<T>` is a functional interface that takes one argument and returns nothing. It is perfect when the caller only needs the object being built:

```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
}
```

In our builder, `addItem` receives a label and a `Consumer<MenuItem>`. The builder creates the item, calls the consumer, and stores the result.

```java
public class MenuItem {

    private String label;
    private String icon;
    private int position;
    private int badge;

    public void setLabel(String label) { this.label = label; }
    public void setIcon(String icon) { this.icon = icon; }
    public void setPosition(int position) { this.position = position; }
    public void setBadge(int badge) { this.badge = badge; }
}
```

```java
public class MenuBuilder {

    private final List<MenuItem> items = new ArrayList<>();

    public MenuBuilder addItem(String label, Consumer<MenuItem> config) {
        MenuItem item = new MenuItem();
        item.setLabel(label);
        config.accept(item);
        items.add(item);
        return this;
    }

    public List<MenuItem> build() {
        return items;
    }
}
```

Usage:

```java
List<MenuItem> menu = new MenuBuilder()
    .addItem("Home", item -> item.setIcon("home"))
    .addItem("Profile", item -> {
        item.setIcon("user");
        item.setBadge(3);
    })
    .build();
```

This is already quite readable. The caller does not need to know how `MenuItem` is created; they only configure it.

## 4. BiConsumer: Two-Argument Configuration

Sometimes the caller needs more than just the object. They might need an index, a key, or a context value. That is where `BiConsumer<T, U>` comes in.

```java
@FunctionalInterface
public interface BiConsumer<T, U> {
    void accept(T t, U u);
}
```

In our `MenuBuilder` we can add a method that also passes the position of the item:

```java
public MenuBuilder addItemAt(int position, String label, BiConsumer<Integer, MenuItem> config) {
    MenuItem item = new MenuItem();
    item.setLabel(label);
    item.setPosition(position);
    config.accept(position, item);
    items.add(item);
    return this;
}
```

Usage:

```java
List<MenuItem> menu = new MenuBuilder()
    .addItemAt(0, "Dashboard", (pos, item) -> {
        item.setIcon("dashboard");
        item.setPosition(pos);
    })
    .addItemAt(1, "Settings", (pos, item) -> item.setIcon("settings"))
    .build();
```

Here `pos` is the first argument and `item` is the second. The caller can use both if needed, or ignore the position and just configure the item.

## 5. Putting It Together

The full builder now has two methods: one for the common case and one for the case where an extra value is useful. Both return `this`, so they chain nicely.

```java
public class MenuBuilder {

    private final List<MenuItem> items = new ArrayList<>();

    public MenuBuilder addItem(String label, Consumer<MenuItem> config) {
        MenuItem item = new MenuItem();
        item.setLabel(label);
        config.accept(item);
        items.add(item);
        return this;
    }

    public MenuBuilder addItemAt(int position, String label, BiConsumer<Integer, MenuItem> config) {
        MenuItem item = new MenuItem();
        item.setLabel(label);
        item.setPosition(position);
        config.accept(position, item);
        items.add(item);
        return this;
    }

    public List<MenuItem> build() {
        return items;
    }
}
```

A complete example:

```java
List<MenuItem> menu = new MenuBuilder()
    .addItem("Home", item -> item.setIcon("home"))
    .addItem("Profile", item -> {
        item.setIcon("user");
        item.setBadge(3);
    })
    .addItemAt(0, "Dashboard", (pos, item) -> {
        item.setIcon("dashboard");
        item.setPosition(pos);
    })
    .build();
```

## 6. When to Use Consumer vs BiConsumer

I use this simple rule:

- Use `Consumer<T>` when the caller only needs the object being configured.
- Use `BiConsumer<T, U>` when the caller also needs a related value such as an index, a key, or a second object.

Examples:

- `Consumer<MenuItem>`: configure the item.
- `BiConsumer<Integer, MenuItem>`: configure the item while also knowing its position.
- `Consumer<Connection>`: configure a database connection.
- `BiConsumer<String, Connection>`: configure a connection that also knows its data source name.

## 7. A Few Tips

- Keep the lambdas short. If the configuration block grows, extract it into a private method.
- Do not force `BiConsumer` when `Consumer` is enough. Two arguments are only better if the second one carries real information.
- Return `this` from every fluent method. If the chain breaks, the API feels clunky.
- Method references (`MenuItem::setIcon`) only work when the lambda signature matches exactly. Usually that means the functional interface takes the same arguments as the method.

## 8. Conclusion

`Consumer` and `BiConsumer` are small but powerful tools for fluent APIs. They let the caller describe *what* to configure while the builder handles *how* the object is created. Start with `Consumer` for the simple cases and add `BiConsumer` when a second value makes the API easier to use. Once you get used to the pattern, it shows up everywhere: menu builders, report builders, test fixtures, and more.
