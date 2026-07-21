---
title: "ADR-01: Static site generator for my engineering portfolio"
date: 2026-07-21T00:00:00Z
description: "Why I chose Hugo for giro-dev.github.io over Jekyll, Next.js or plain HTML."
---

## Context

I needed a maintainable personal site that could grow with my career. The site had to be easy to update, version-controlled, fast to load and simple to deploy to GitHub Pages.

## Options considered

1. **Plain HTML/CSS** — trivial to host, but hard to maintain as sections and posts grow.
2. **Jekyll** — native GitHub Pages support, but Ruby-based and I prefer a single static binary.
3. **Next.js / React** — powerful, but overkill for a mostly static portfolio and adds JavaScript complexity.
4. **Hugo** — single binary, extremely fast builds, Markdown content, Go templates and easy CI/CD.

## Decision

Use **Hugo** with **GitHub Actions** deploying to **GitHub Pages**.

## Consequences

- Content is written in Markdown and lives in Git.
- Layouts are decoupled from content, so changing the design does not require editing every page.
- Builds take milliseconds and the deployment is reproducible.
- Adding a blog or new sections only requires Markdown files and small template changes.
