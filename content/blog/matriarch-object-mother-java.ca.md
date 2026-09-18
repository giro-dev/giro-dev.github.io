---
title: "Matriarch: generació de dades de prova per a Java"
date: 2026-07-21T00:00:00Z
description: "Com construir una llibreria Object Mother fluent per accelerar els tests de Java."
---
# Matriarch: generació de dades de prova per a Java

## 1. Introducció

Els tests de Java sovint necessiten molts objectes abans de poder provar el comportament real. Crear aquests objectes a mà porta temps i fa que els tests siguin més difícils de llegir.

En aquest article mostraré la idea de **Matriarch**, una llibreria Java petita que crea dades de prova amb el patró Object Mother.

## 2. Problema

Escriure fixtures de prova per a POJOs i Records de Java és repetitiu i pot introduir errors. Quan els models creixen, els constructors i els builders també creixen, i els tests tenen més codi de preparació.

## 3. Enfocament

He implementat el patró **Object Mother** com una llibreria Java open-source petita amb una API fluent. Els usuaris defineixen una `Mother` per a una classe una sola vegada. Després poden crear instàncies vàlides i sobreescriure només els valors que necessita cada test.

## 4. Decisions principals

- **API fluent** — el chaining manté llegible la preparació dels tests.
- **Jackson + SnakeYAML** — els patrons YAML mantenen les fixtures externes i reutilitzables.
- **Llavors deterministes** — la mateixa llavor produeix les mateixes dades, i els tests es mantenen estables.
- **Integració amb JUnit 5** — `@MotherFactoryResource` i `@RandomArg` permeten tests basats en dades.
- **Maven Central + GitHub Actions** — la publicació i la CI s’executen automàticament.

## 5. Resultat

Matriarch és una llibreria reutilitzable que redueix la preparació de fixtures. Els tests es poden centrar en el comportament i no en construir objectes.

## 6. Conclusió

El patró Object Mother deixa les dades de prova comunes en un sol lloc. Matriarch hi afegeix una API fluent, patrons externs i dades deterministes perquè la preparació sigui petita i repetible.
