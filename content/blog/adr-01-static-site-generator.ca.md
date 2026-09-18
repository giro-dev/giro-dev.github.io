---
title: "ADR-01: generador de llocs estàtics per al meu portfolio d’enginyeria"
date: 2026-07-21T00:00:00Z
description: "Per què he triat Hugo per a giro-dev.github.io en lloc de Jekyll, Next.js o HTML pla."
---
# ADR-01: generador de llocs estàtics per al meu portfolio d’enginyeria

## 1. Introducció

Necessitava un lloc personal que pogués mantenir mentre creix la meva carrera. Havia de ser fàcil d’actualitzar, estar guardat a Git, carregar ràpid i ser senzill de desplegar a GitHub Pages.

En aquest article mostraré les opcions que vaig considerar i per què vaig triar Hugo per a aquest lloc.

## 2. Context

El lloc ha de tenir pàgines de portfolio i articles. Vull mantenir separat el contingut del codi del lloc, per poder canviar el disseny sense reescriure totes les pàgines.

## 3. Opcions considerades

1. **HTML/CSS pla** — fàcil d’allotjar, però més difícil de mantenir quan creixen les seccions i els articles.
2. **Jekyll** — funciona de manera nativa amb GitHub Pages, però usa Ruby i prefereixo un sol binari estàtic.
3. **Next.js / React** — és potent, però és massa per a un portfolio principalment estàtic i afegeix complexitat de JavaScript.
4. **Hugo** — un binari, builds molt ràpids, contingut en Markdown, plantilles Go i CI/CD senzill.

## 4. Decisió

Faré servir **Hugo** amb **GitHub Actions** per desplegar a **GitHub Pages**.

## 5. Conseqüències

- **Contingut Markdown** — el contingut està escrit en Markdown i guardat a Git.
- **Layouts separats** — els layouts estan separats del contingut, així que un canvi de disseny no obliga a editar totes les pàgines.
- **Builds ràpids** — els builds triguen mil·lisegons i el desplegament es pot repetir.
- **Creixement senzill** — afegir un blog o una secció nova només necessita fitxers Markdown i petits canvis a les plantilles.

## 6. Conclusió

Hugo em dona la configuració de lloc estàtic senzilla que necessito. El contingut és fàcil d’editar i el desplegament és fàcil de repetir.
