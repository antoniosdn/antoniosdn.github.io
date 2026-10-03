# Blog técnico — Antonio

Site estático em [Astro](https://astro.build), publicado no GitHub Pages.

## Escrever um post

Crie `src/content/blog/slug-do-post.md`:

```md
---
title: 'Título'
description: 'Resumo de uma linha'
pubDate: '2026-10-03'
heroImage: '../../assets/imagem.jpg' # opcional
---

Texto em Markdown...
```

## Comandos

| Comando           | Ação                                  |
| ----------------- | ------------------------------------- |
| `npm run dev`     | Servidor local em `localhost:4321`    |
| `npm run build`   | Gera o site em `./dist/`              |
| `npm run preview` | Visualiza o build localmente          |

O push na branch `main` publica o site automaticamente (`.github/workflows/deploy.yml`).
