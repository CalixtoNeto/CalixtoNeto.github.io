# calixtoneto.com

Site pessoal e blog (Jekyll + GitHub Pages).

## Novo artigo

Crie `_posts/AAAA-MM-DD-titulo.md`:

```markdown
---
layout: post
title: "Título"
description: "Resumo de uma linha"
---
Texto em markdown.
```

Para entrar na lista lateral da série, adicione `serie: "Nome"` e `ordem: N`.

## Rodar localmente (opcional)

Requer Ruby: `bundle install` e depois `bundle exec jekyll serve`. Abra http://localhost:4000.

## Publicar

`git push` na branch `main`. O GitHub Pages builda sozinho.
