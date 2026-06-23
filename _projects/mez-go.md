---
title: Mez
description: Plataforma de mensageria multicanal — WhatsApp, Telegram, Instagram, Messenger.
stack: [Go, NATS JetStream, PostgreSQL, Redis, MinIO, Docker]
status: active
visibility: private
featured: true
---

Plataforma de atendimento e automação multicanal com arquitetura hexagonal, multi-tenant e event-driven.

**[Acessar página do projeto →](https://mez-go-page.felipedsvit.workers.dev/)**

## O que faz

Centraliza mensagens de WhatsApp, Telegram, Instagram Direct e Messenger em uma única interface, com roteamento inteligente, filas por canal e histórico persistido por tenant.

## Arquitetura (visão pública)

- **Monorepo multi-binário**: 6 `cmd/*` independentes — gateway, worker, api, etc.
- **Hexagonal**: portas e adaptadores por canal — cada rede social é um adaptador isolado
- **NATS JetStream**: filas por canal, replay de mensagens, dead-letter queue
- **Multi-tenant**: Postgres RLS com isolamento por organização
- **Redis**: cache de sessão WhatsApp, rate limiting por tenant

## Canais suportados

| Canal     | Adapter    |
|-----------|------------|
| WhatsApp  | whatsmeow  |
| Telegram  | tgbot-api  |
| Instagram | Graph API  |
| Messenger | Graph API  |
| WhatsApp Business | WABA API |

## CI/CD

GitHub Actions com pipelines de CI, release semântico e geração de SDK.

> Repositório privado. Arquitetura disponível sob consulta.
