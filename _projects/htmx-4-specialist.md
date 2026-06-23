---
title: HTMX 4 Specialist
description: Skill para LLMs especializado em HTMX 4.0, HDA e server-side rendering.
repo: https://github.com/felipedsvit/htmx-4-specialist
stack: [Markdown, YAML, HTMX, LLM]
status: active
visibility: public
featured: false
---

Skill reutilizável para agentes LLM com conhecimento especializado em HTMX 4.0 e Hypermedia-Driven Applications (HDA).

## O que é

Um conjunto de documentos estruturados que enrinam um LLM com conhecimento atualizado sobre HTMX 4.0 — que introduz mudanças significativas em relação ao HTMX 2.x e evita que modelos gerem código baseado em versões anteriores.

## Estrutura

```
SKILL.md      → definição da skill e instruções para o agente
agents/       → configurações de agentes
evals/        → casos de teste e avaliações
references/   → documentação e exemplos de referência
docs/         → guias de uso
```

## Casos de uso

- Gerar código HTMX 4.0 correto sem misturar com a API do 2.x
- Arquitetar HDA sem cair em padrões SPA
- Consultar padrões de SSE, SSR, lazy loading e partial rendering com HTMX 4

## Por que existe

HTMX 4.0 ainda é recente e os LLMs treinados em dados mais antigos tendem a gerar código 2.x. Esta skill serve de ground truth para sessões de desenvolvimento assistido por IA.
