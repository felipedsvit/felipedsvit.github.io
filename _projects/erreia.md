---
title: Erreia
description: Kanban leve em Go com realtime via SSE e Postgres LISTEN/NOTIFY.
repo: https://github.com/felipedsvit/erreia
stack: [Go, HTMX, Templ, PostgreSQL, MinIO, Argon2id]
status: active
visibility: public
featured: true
---

Kanban minimalista construído como experimento em arquitetura sem JavaScript frameworks.

## Destaques técnicos

- **Realtime sem WebSocket**: SSE via `Postgres LISTEN/NOTIFY` — o banco é o message broker
- **Auth por sessão**: Argon2id com salt aleatório, sem JWT
- **Avatares no MinIO**: storage self-hosted, zero dependência de S3 pago
- **CSS manual**: zero Tailwind, zero Bootstrap — variáveis CSS e reset simples
- **Binário único**: ~15 MB incluindo assets embarcados via `embed.FS`
- **Templ**: templates Go compilados em tempo de build, type-safe

## Arquitetura

```
cmd/server/
internal/
  auth/       → sessões, Argon2id
  avatar/     → upload, resize, MinIO
  board/      → domínio Kanban (cards, colunas, boards)
  config/     → variáveis de ambiente
  database/   → pool Postgres, migrations
  realtime/   → SSE hub, LISTEN/NOTIFY
  session/    → cookie store
  storage/    → adapter MinIO
  user/       → usuários
  web/        → handlers HTTP, rotas
```

## Stack

| Camada     | Tecnologia      |
|------------|-----------------|
| Linguagem  | Go 1.23+        |
| Templates  | Templ           |
| Frontend   | HTMX 2.x        |
| Banco      | PostgreSQL      |
| Storage    | MinIO           |
| Auth       | Argon2id        |
| Deploy     | Docker (single binary) |

## CI/CD

Build e push de imagem Docker via GitHub Actions em push para `main`.
