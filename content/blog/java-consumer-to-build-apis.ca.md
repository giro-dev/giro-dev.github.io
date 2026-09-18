---
title: "Construir una API Java fluent amb Consumer i BiConsumer"
date: 2026-09-17T00:00:00Z
description: "Com fer servir Consumer i BiConsumer per construir APIs Java clares i fluides."
---
# Construir una API Java fluent amb Consumer i BiConsumer

## 1. Introducció

Quan construeixo llibreries internes petites, intento que l’API pública sigui fàcil de llegir. Darrerament faig servir sovint `Consumer` i `BiConsumer` perquè els callers puguin configurar objectes sense escriure classes anònimes llargues. El resultat s’assembla a un builder, però cada pas és només un lambda que prepara alguna cosa.

En aquest article mostraré un exemple inventat: un `MenuBuilder` petit que permet afegir elements a un menú. Veurem on encaixa `Consumer`, on encaixa `BiConsumer` i com es combinen en una sola cadena fluent.

## 2. La idea: un builder fluent

Una API fluent és una API on les crides als mètodes es poden encadenar:

```java
List<MenuItem> menu = new MenuBuilder()
    .addItem("Home", item -> item.setIcon("home"))
    .addItem("Profile", item -> item.setIcon("user"))
    .build();
```

Perquè això funcioni, cada mètode retorna `this`. El lambda és la part interessant. En lloc de passar cinc paràmetres, el caller rep una instància de `MenuItem` i la configura com vol.

## 3. Consumer: configuració amb un argument

`Consumer<T>` és una interfície funcional que rep un argument i no retorna res. És útil quan el caller només necessita l’objecte que s’està construint:

```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
}
```

Al nostre builder, `addItem` rep un label i un `Consumer<MenuItem>`. El builder crea l’element, crida el consumer i guarda el resultat.

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

Ús:

```java
List<MenuItem> menu = new MenuBuilder()
    .addItem("Home", item -> item.setIcon("home"))
    .addItem("Profile", item -> {
        item.setIcon("user");
        item.setBadge(3);
    })
    .build();
```

Això ja és força llegible. El caller no ha de saber com es crea `MenuItem`; només el configura.

## 4. BiConsumer: configuració amb dos arguments

De vegades el caller necessita alguna cosa més que l’objecte. Pot necessitar un índex, una clau o un valor de context. Aquí és on entra `BiConsumer<T, U>`.

```java
@FunctionalInterface
public interface BiConsumer<T, U> {
    void accept(T t, U u);
}
```

Al nostre `MenuBuilder` podem afegir un mètode que també passi la posició de l’element:

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

Ús:

```java
List<MenuItem> menu = new MenuBuilder()
    .addItemAt(0, "Dashboard", (pos, item) -> {
        item.setIcon("dashboard");
        item.setPosition(pos);
    })
    .addItemAt(1, "Settings", (pos, item) -> item.setIcon("settings"))
    .build();
```

Aquí `pos` és el primer argument i `item` és el segon. El caller pot fer servir tots dos o ignorar la posició i configurar només l’element.

## 5. Ajuntar-ho

El builder complet té dos mètodes: un per al cas habitual i un altre per al cas on és útil un valor extra. Tots dos retornen `this`, així que es poden encadenar bé.

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

Un exemple complet:

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

## 6. Quan fer servir Consumer o BiConsumer

Jo faig servir aquesta regla senzilla:

- Fes servir `Consumer<T>` quan el caller només necessita l’objecte que configura.
- Fes servir `BiConsumer<T, U>` quan el caller també necessita un valor relacionat, com un índex, una clau o un segon objecte.

Exemples:

- `Consumer<MenuItem>`: configura l’element.
- `BiConsumer<Integer, MenuItem>`: configura l’element i coneix la seva posició.
- `Consumer<Connection>`: configura una connexió de base de dades.
- `BiConsumer<String, Connection>`: configura una connexió que també coneix el nom de la seva font de dades.

## 7. Alguns consells

- Mantén els lambdas curts. Si el bloc de configuració creix, extreu-lo a un mètode privat.
- No obliguis a fer servir `BiConsumer` quan `Consumer` és suficient. Dos arguments només són millors si el segon porta informació real.
- Retorna `this` des de cada mètode fluent. Si la cadena es trenca, l’API sembla incòmoda.
- Les referències a mètodes (`MenuItem::setIcon`) només funcionen quan la signatura del lambda encaixa exactament. Normalment això vol dir que la interfície funcional rep els mateixos arguments que el mètode.

## 8. Conclusió

`Consumer` i `BiConsumer` són eines petites però útils per a APIs fluent. Permeten que el caller descrigui *què* vol configurar mentre el builder gestiona *com* es crea l’objecte. Comença amb `Consumer` per als casos senzills i afegeix `BiConsumer` quan un segon valor fa l’API més fàcil de fer servir. Quan t’hi acostumes, el patró apareix a tot arreu: builders de menús, builders d’informes, fixtures de test i més.
