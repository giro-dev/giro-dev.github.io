---
title: "ADR-01: generador de sitios estáticos para mi portfolio de ingeniería"
date: 2026-07-21T00:00:00Z
description: "Por qué he elegido Hugo para giro-dev.github.io en lugar de Jekyll, Next.js o HTML plano."
---
# ADR-01: generador de sitios estáticos para mi portfolio de ingeniería

## 1. Introducción

Necesitaba un sitio personal que pudiera mantener mientras crece mi carrera. Tenía que ser fácil de actualizar, estar guardado en Git, cargar rápido y ser sencillo de desplegar en GitHub Pages.

En este artículo mostraré las opciones que consideré y por qué elegí Hugo para este sitio.

## 2. Contexto

El sitio necesita páginas de portfolio y artículos. Quiero mantener separado el contenido del código del sitio para poder cambiar el diseño sin reescribir cada página.

## 3. Opciones consideradas

1. **HTML/CSS plano** — fácil de alojar, pero más difícil de mantener cuando crecen las secciones y los artículos.
2. **Jekyll** — funciona de forma nativa con GitHub Pages, pero usa Ruby y prefiero un solo binario estático.
3. **Next.js / React** — es potente, pero es demasiado para un portfolio principalmente estático y añade complejidad de JavaScript.
4. **Hugo** — un binario, builds muy rápidos, contenido en Markdown, plantillas Go y CI/CD sencillo.

## 4. Decisión

Usaré **Hugo** con **GitHub Actions** para desplegar en **GitHub Pages**.

## 5. Consecuencias

- **Contenido Markdown** — el contenido está escrito en Markdown y guardado en Git.
- **Layouts separados** — los layouts están separados del contenido, así que un cambio de diseño no obliga a editar todas las páginas.
- **Builds rápidos** — los builds tardan milisegundos y el despliegue se puede repetir.
- **Crecimiento sencillo** — añadir un blog o una sección nueva solo necesita archivos Markdown y pequeños cambios en las plantillas.

## 6. Conclusión

Hugo me da la configuración sencilla de sitio estático que necesito. El contenido es fácil de editar y el despliegue es fácil de repetir.
