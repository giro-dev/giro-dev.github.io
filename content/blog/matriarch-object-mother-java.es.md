---
title: "Matriarch: generación de datos de prueba para Java"
date: 2026-07-21T00:00:00Z
description: "Cómo construir una librería Object Mother fluent para acelerar los tests de Java."
---
# Matriarch: generación de datos de prueba para Java

## 1. Introducción

Los tests de Java suelen necesitar muchos objetos antes de poder probar el comportamiento real. Crear esos objetos a mano lleva tiempo y hace que los tests sean más difíciles de leer.

En este artículo mostraré la idea de **Matriarch**, una librería Java pequeña que crea datos de prueba con el patrón Object Mother.

## 2. Problema

Escribir fixtures de prueba para POJOs y Records de Java es repetitivo y puede introducir errores. Cuando los modelos crecen, los constructores y los builders también crecen, y los tests tienen más código de preparación.

## 3. Enfoque

He implementado el patrón **Object Mother** como una librería Java open-source pequeña con una API fluent. Los usuarios definen una `Mother` para una clase una sola vez. Después pueden crear instancias válidas y sobrescribir solo los valores que necesita cada test.

## 4. Decisiones principales

- **API fluent** — el chaining mantiene legible la preparación de los tests.
- **Jackson + SnakeYAML** — los patrones YAML mantienen las fixtures externas y reutilizables.
- **Semillas deterministas** — la misma semilla produce los mismos datos, y los tests se mantienen estables.
- **Integración con JUnit 5** — `@MotherFactoryResource` y `@RandomArg` permiten tests basados en datos.
- **Maven Central + GitHub Actions** — la publicación y la CI se ejecutan automáticamente.

## 5. Resultado

Matriarch es una librería reutilizable que reduce la preparación de fixtures. Los tests se pueden centrar en el comportamiento y no en construir objetos.

## 6. Conclusión

El patrón Object Mother deja los datos de prueba comunes en un solo lugar. Matriarch añade una API fluent, patrones externos y datos deterministas para que la preparación sea pequeña y repetible.
