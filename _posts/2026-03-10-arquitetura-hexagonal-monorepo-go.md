---
layout: post
title: "Monorepo Go com múltiplos binários e arquitetura hexagonal"
description: Como estruturar um monorepo Go com 6 cmd/* independentes, portas e adaptadores por domínio, e CI/CD por binário sem acoplamento entre módulos.
date: 2026-03-10
tags: [go, hexagonal, monorepo, arquitetura, mez]
---

Quando uma plataforma de mensageria precisa suportar WhatsApp, Telegram, Instagram e Messenger simultaneamente, a primeira tentação é um binário monolítico com switches por canal. É o caminho mais rápido para um `main.go` de mil linhas impossível de testar.

A alternativa que usei no Mez: monorepo com múltiplos binários e arquitetura hexagonal. Um canal, um adaptador. Um adaptador, uma porta.

## A estrutura

```
mez-go/
├── cmd/
│   ├── gateway/     → recebe mensagens de todos os canais
│   ├── worker/      → processa e roteia mensagens
│   ├── api/         → HTTP API para o frontend
│   ├── scheduler/   → jobs periódicos (retry, cleanup)
│   ├── migrator/    → migrations de banco (roda e sai)
│   └── monitor/     → métricas e healthcheck
│
├── internal/
│   ├── domain/      → entidades, erros de domínio, value objects
│   ├── ports/       → interfaces (o que o domínio precisa)
│   └── adapters/
│       ├── whatsapp/    → whatsmeow
│       ├── telegram/    → tgbot-api
│       ├── instagram/   → Graph API
│       ├── messenger/   → Graph API
│       ├── waba/        → WhatsApp Business API
│       ├── nats/        → JetStream publisher/consumer
│       ├── postgres/    → repositórios
│       └── redis/       → cache, rate limiting
```

## Por que múltiplos binários?

Cada `cmd/*` é compilado e deployado independentemente. Isso tem consequências concretas:

**Escala independente**: o `gateway` recebe alto volume de webhooks dos canais; o `worker` processa mensagens com latência diferente. Escalar ambos juntos seria desperdício.

**Falha isolada**: se o `scheduler` trava, o gateway continua recebendo e o worker continua processando. Num monolito, uma goroutine em pânico pode derrubar tudo.

**CI/CD por binário**: o pipeline só reconstrói e deploya os binários cujos pacotes mudaram.

```yaml
# GitHub Actions — detecta quais cmd/* mudaram
- name: Detect changed binaries
  id: changes
  run: |
    CHANGED=$(git diff --name-only HEAD~1 HEAD)
    for cmd in gateway worker api scheduler monitor; do
      if echo "$CHANGED" | grep -q "cmd/$cmd\|internal/"; then
        echo "$cmd=true" >> $GITHUB_OUTPUT
      fi
    done
```

## A porta define o contrato

A porta é uma interface Go. O domínio fala com o mundo exterior apenas através de portas — nunca importa um adaptador diretamente.

```go
// internal/ports/messaging.go

// MessageSender é o que o domínio precisa para enviar mensagens.
// Não sabe se é WhatsApp, Telegram ou mock de teste.
type MessageSender interface {
    Send(ctx context.Context, msg OutboundMessage) (string, error)
    SendTyping(ctx context.Context, to ChannelID) error
}

// ChannelListener é o que o gateway precisa para receber mensagens.
type ChannelListener interface {
    Listen(ctx context.Context, events chan<- InboundEvent) error
    Channel() ChannelType
}

// ConversationRepository é o que o worker precisa de persistência.
type ConversationRepository interface {
    Get(ctx context.Context, id ConversationID) (*Conversation, error)
    Save(ctx context.Context, c *Conversation) error
    ListByTenant(ctx context.Context, tenantID TenantID) ([]*Conversation, error)
}
```

## O adaptador implementa a porta

```go
// internal/adapters/whatsapp/sender.go

type Sender struct {
    client *whatsmeow.Client
    logger *slog.Logger
}

// Send implementa ports.MessageSender
func (s *Sender) Send(ctx context.Context, msg ports.OutboundMessage) (string, error) {
    // traduz ports.OutboundMessage → whatsmeow.Message
    // retorna o ID da mensagem enviada
}

// Garantia em tempo de compilação que *Sender satisfaz a interface
var _ ports.MessageSender = (*Sender)(nil)
```

O `var _ ports.MessageSender = (*Sender)(nil)` é uma verificação em tempo de compilação. Se `Sender` não implementar todos os métodos de `MessageSender`, o build falha com mensagem clara. Nenhum teste necessário para isso.

## Composição no cmd/

O `cmd/gateway` é o ponto de composição. Ele instancia os adaptadores e os injeta no domínio:

```go
// cmd/gateway/main.go

func main() {
    cfg := config.Load()

    pool := database.Open(ctx, cfg.DatabaseURL)
    nats := natsadapter.New(cfg.NATSUrl)
    logger := slog.New(...)

    // Adaptadores concretos
    waListener := whatsapp.NewListener(cfg.WhatsApp, logger)
    tgListener := telegram.NewListener(cfg.Telegram, logger)

    // Domínio recebe as portas (interfaces), não os adaptadores
    gw := gateway.New(
        []ports.ChannelListener{waListener, tgListener},
        nats,      // implementa ports.EventPublisher
        logger,
    )

    gw.Run(ctx)
}
```

O `gateway.New` aceita `[]ports.ChannelListener` — um slice de interface. Adicionar o Instagram não muda a assinatura do construtor nem o domínio. Só cria um novo adaptador e o adiciona ao slice.

## Anti-Corruption Layer nos adaptadores externos

Cada canal tem seu próprio modelo de dados. O `whatsmeow` tem `types.JID`, `waProto.Message`, `events.Message`. O Telegram tem `tgbotapi.Update`. Nenhum desses tipos deve vazar para além do adaptador.

```go
// internal/adapters/whatsapp/listener.go

// translateEvent converte whatsmeow event → domain event
// Nenhum tipo de whatsmeow existe fora deste arquivo.
func translateEvent(ev *events.Message) (ports.InboundEvent, error) {
    return ports.InboundEvent{
        Channel:   ports.ChannelWhatsApp,
        ExternalID: ev.Info.ID,
        From:      ports.ChannelID(ev.Info.Sender.String()),
        Body:      extractBody(ev.Message),
        Timestamp: ev.Info.Timestamp,
    }, nil
}
```

Quando a lib `whatsmeow` muda (e muda com frequência, pois segue o protocolo do WhatsApp Web), só `internal/adapters/whatsapp/` precisa ser atualizado. O domínio não sabe que `whatsmeow` existe.

## O que aprendi

**Comece com uma porta, um adaptador, um teste.** A tentação é modelar tudo antes de escrever código. Resisti: comecei pelo WhatsApp, estabilizei a interface `ChannelListener`, depois adicionei Telegram. A interface emergiu do uso real.

**Não crie adaptadores para o que você controla.** O banco de dados é seu — o repositório Postgres pode usar `pgx` diretamente sem passar por uma interface genérica de "database". A porta existe para o que é externo e fora do seu controle.

**Múltiplos binários com um único módulo Go**: tudo em um `go.mod`, um `go.sum`. Os binários compartilham código mas são compilados independentemente. Não é o mesmo que múltiplos módulos Go — evitei essa complexidade.
