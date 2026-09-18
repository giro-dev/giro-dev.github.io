---
title: "Construir una API Java fluent con Consumer y BiConsumer"
date: 2026-09-17T00:00:00Z
description: "Cómo usar Consumer y BiConsumer para construir APIs Java claras y fluidas."
---
# Construir una API Java fluent con Consumer y BiConsumer

## 1. Introducción

Cuando construyo librerías internas pequeñas, intento que la API pública sea fácil de leer. Últimamente uso `Consumer` y `BiConsumer` a menudo para que los callers puedan configurar objetos sin escribir clases anónimas largas. El resultado se parece a un builder, pero cada paso es solo un lambda que prepara algo.

En este artículo mostraré un ejemplo inventado: un `MenuBuilder` pequeño que permite añadir elementos a un menú. Veremos dónde encaja `Consumer`, dónde encaja `BiConsumer` y cómo se combinan en una sola cadena fluent.

## 2. La idea: un builder fluent

Una API fluent es una API donde las llamadas a los métodos se pueden encadenar:

```java
List<MenuItem> menu = new MenuBuilder()
    .addItem("Home", item -> item.setIcon("home"))
    .addItem("Profile", item -> item.setIcon("user"))
    .build();
```

Para que esto funcione, cada método devuelve `this`. El lambda es la parte interesante. En lugar de pasar cinco parámetros, el caller recibe una instancia de `MenuItem` y la configura como quiere.

## 3. Consumer: configuración con un argumento

`Consumer<T>` es una interfaz funcional que recibe un argumento y no devuelve nada. Es útil cuando el caller solo necesita el objeto que se está construyendo:

```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
}
```

En nuestro builder, `addItem` recibe un label y un `Consumer<MenuItem>`. El builder crea el elemento, llama al consumer y guarda el resultado.

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

Uso:

```java
List<MenuItem> menu = new MenuBuilder()
    .addItem("Home", item -> item.setIcon("home"))
    .addItem("Profile", item -> {
        item.setIcon("user");
        item.setBadge(3);
    })
    .build();
```

Esto ya es bastante legible. El caller no necesita saber cómo se crea `MenuItem`; solo lo configura.

## 4. BiConsumer: configuración con dos argumentos

A veces el caller necesita algo más que el objeto. Puede necesitar un índice, una clave o un valor de contexto. Ahí es donde entra `BiConsumer<T, U>`.

```java
@FunctionalInterface
public interface BiConsumer<T, U> {
    void accept(T t, U u);
}
```

En nuestro `MenuBuilder` podemos añadir un método que también pase la posición del elemento:

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

Uso:

```java
List<MenuItem> menu = new MenuBuilder()
    .addItemAt(0, "Dashboard", (pos, item) -> {
        item.setIcon("dashboard");
        item.setPosition(pos);
    })
    .addItemAt(1, "Settings", (pos, item) -> item.setIcon("settings"))
    .build();
```

Aquí `pos` es el primer argumento y `item` es el segundo. El caller puede usar ambos o ignorar la posición y configurar solo el elemento.

## 5. Juntarlo todo

El builder completo tiene dos métodos: uno para el caso normal y otro para cuando es útil un valor extra. Los dos devuelven `this`, así que se pueden encadenar bien.

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

Un ejemplo completo:

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

## 6. Cuándo usar Consumer o BiConsumer

Yo uso esta regla sencilla:

- Usa `Consumer<T>` cuando el caller solo necesita el objeto que configura.
- Usa `BiConsumer<T, U>` cuando el caller también necesita un valor relacionado, como un índice, una clave o un segundo objeto.

Ejemplos:

- `Consumer<MenuItem>`: configura el elemento.
- `BiConsumer<Integer, MenuItem>`: configura el elemento y conoce su posición.
- `Consumer<Connection>`: configura una conexión de base de datos.
- `BiConsumer<String, Connection>`: configura una conexión que también conoce el nombre de su fuente de datos.

## 7. Algunos consejos

- Mantén los lambdas cortos. Si el bloque de configuración crece, extráelo a un método privado.
- No fuerces `BiConsumer` cuando `Consumer` es suficiente. Dos argumentos solo son mejores si el segundo lleva información real.
- Devuelve `this` desde cada método fluent. Si la cadena se rompe, la API resulta incómoda.
- Las referencias a métodos (`MenuItem::setIcon`) solo funcionan cuando la firma del lambda encaja exactamente. Normalmente significa que la interfaz funcional recibe los mismos argumentos que el método.

## 8. Conclusión

`Consumer` y `BiConsumer` son herramientas pequeñas pero útiles para APIs fluent. Permiten que el caller describa *qué* configurar mientras el builder gestiona *cómo* se crea el objeto. Empieza con `Consumer` para los casos sencillos y añade `BiConsumer` cuando un segundo valor hace la API más fácil de usar. Cuando te acostumbras al patrón, aparece en todas partes: builders de menús, builders de informes, fixtures de test y más.
