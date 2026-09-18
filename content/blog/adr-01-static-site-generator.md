---
title: "ADR-01: Static site generator for my engineering portfolio"
date: 2026-07-21T00:00:00Z
description: "Why I chose Hugo for giro-dev.github.io over Jekyll, Next.js or plain HTML."
---
# ADR-01: Static site generator for my engineering portfolio

## 1. Overview

I needed a personal site that I can maintain while my career grows. It must be easy to update, stored in Git, fast to load, and simple to deploy to GitHub Pages.

In this post I will show the options I considered and why I chose Hugo for this site.

## 2. Context

The site needs to support portfolio pages and blog posts. I want the content and the site code to stay separate, so I can change the design without rewriting every page.

## 3. Options considered

1. **Plain HTML/CSS** — easy to host, but harder to maintain as sections and posts grow.
2. **Jekyll** — works natively with GitHub Pages, but it uses Ruby and I prefer one static binary.
3. **Next.js / React** — powerful, but too much for a mostly static portfolio and it adds JavaScript complexity.
4. **Hugo** — one binary, very fast builds, Markdown content, Go templates, and simple CI/CD.

## 4. Decision

I will use **Hugo** with **GitHub Actions** to deploy to **GitHub Pages**.

## 5. Consequences

- **Markdown content** — content is written in Markdown and stored in Git.
- **Separate layouts** — layouts are separate from content, so a design change does not require editing every page.
- **Fast builds** — builds take milliseconds and the deployment can be reproduced.
- **Simple growth** — adding a blog or a new section needs Markdown files and small template changes.

## 6. Conclusion

Hugo gives me the simple static site setup I need. It keeps content easy to edit and deployment easy to repeat.
