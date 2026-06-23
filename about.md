---
layout: default
title: Sobre
description: Go developer especializado em arquitetura hexagonal, mensageria multicanal e HTMX.
---

<div class="container" style="padding-top:3rem;padding-bottom:3rem">

# Sobre

Desenvolvedor Go com foco em arquiteturas hexagonais, sistemas de mensageria multicanal e interfaces hipermídia com HTMX.

## O que faço

- **Backend em Go** — binários únicos, stdlib-first, arquitetura hexagonal com portas e adaptadores bem definidos
- **Mensageria multicanal** — WhatsApp, Telegram, Instagram, Messenger via NATS JetStream
- **Interfaces com HTMX + Templ** — SSR, SSE, HDA sem overhead de SPA
- **Sistemas multi-tenant** — PostgreSQL RLS, isolamento por organização, audit trails
- **E-commerce modular** — checkout, pagamentos (Stripe, Mercado Pago), fiscal (NF-e)

## Stack principal

| Área          | Tecnologias                                   |
|---------------|-----------------------------------------------|
| Backend       | Go, PostgreSQL, NATS JetStream, Redis, MinIO  |
| Frontend      | HTMX, Templ, CSS                              |
| Arquitetura   | Hexagonal, DDD, Event-Driven, Multi-tenant    |
| DevOps        | Docker, GitHub Actions, Linux                 |

## Projetos

{% for project in site.data.projects %}
- **[{{ project.title }}]({{ project.url | relative_url }})** — {{ project.description }}
{% endfor %}

## Contato

- GitHub: [github.com/felipedsvit](https://github.com/felipedsvit)
- Email: [felipedsvit@gmail.com](mailto:felipedsvit@gmail.com)

</div>
