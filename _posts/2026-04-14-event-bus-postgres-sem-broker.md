---
layout: post
title: "Event bus interno com Postgres — sem Kafka, sem Redis, sem infra extra"
description: Como usar Postgres LISTEN/NOTIFY como event bus entre módulos de um monolito, garantindo entrega transacional sem operador de broker.
date: 2026-04-14
tags: [go, postgres, event-driven, arquitetura, comm]
---

Quando comecei o [Comm](portfolio/comm/), precisava de comunicação assíncrona entre módulos: ao confirmar um pedido, o módulo `order` precisa avisar `notification`, `fiscal` e `inventory`. Três opções:

1. **Chamada direta**: `order` importa `notification` — acoplamento direto, sem isolamento
2. **Kafka/RabbitMQ**: funciona, mas agora tenho mais uma peça pra operar, monitorar, fazer backup
3. **Postgres LISTEN/NOTIFY**: já tenho Postgres rodando, é transacional por natureza

Escolhi a terceira. Aqui está o padrão.

## O problema de chamar direto

```go
// ❌ acoplamento que cresce sem controle
func (s *OrderService) Confirm(ctx context.Context, id OrderID) error {
    order, _ := s.repo.Get(ctx, id)
    order.Confirm()
    s.repo.Save(ctx, order)

    // agora order conhece notification, fiscal e inventory
    s.notifications.Send(ctx, ConfirmationEmail{Order: order})
    s.fiscal.EmitInvoice(ctx, order)
    s.inventory.Reserve(ctx, order.Items)
    return nil
}
```

`order` virou coordenador de tudo. Adicionar `loyalty_points` exige mudar `OrderService`.

## O padrão com LISTEN/NOTIFY

A ideia central: **o evento é publicado dentro da mesma transação que muda o estado**. Não é possível confirmar um pedido sem publicar o evento, nem publicar um evento sem confirmar o pedido. A consistência é garantida pelo banco.

```go
// ports/eventbus.go
type EventBus interface {
    Publish(ctx context.Context, tx pgx.Tx, event DomainEvent) error
}

// adapters/postgres/eventbus.go
type PgEventBus struct{}

func (b *PgEventBus) Publish(ctx context.Context, tx pgx.Tx, event DomainEvent) error {
    payload, err := json.Marshal(event)
    if err != nil {
        return err
    }
    // pg_notify dentro da transação — só dispara se o commit acontecer
    _, err = tx.Exec(ctx,
        "SELECT pg_notify($1, $2)",
        event.Channel(), string(payload),
    )
    return err
}
```

```go
// internal/order/service.go
func (s *OrderService) Confirm(ctx context.Context, id OrderID) error {
    return s.db.BeginTxFunc(ctx, func(tx pgx.Tx) error {
        order, _ := s.repo.GetTx(ctx, tx, id)
        order.Confirm()
        s.repo.SaveTx(ctx, tx, order)

        // publicado na mesma transação — ou ambos persistem, ou nenhum
        return s.bus.Publish(ctx, tx, OrderConfirmedEvent{Order: order})
    })
}
```

`order` não sabe que `notification` existe. Só publica um evento.

## Os subscribers

Cada módulo que precisa reagir a `order_confirmed` roda um goroutine com `LISTEN`:

```go
// internal/notification/subscriber.go

func (s *Subscriber) Run(ctx context.Context) {
    conn, _ := pgx.Connect(ctx, s.dsn)
    defer conn.Close(ctx)
    conn.Exec(ctx, "LISTEN order_confirmed")

    for {
        notif, err := conn.WaitForNotification(ctx)
        if err != nil {
            return
        }
        var ev order.ConfirmedEvent
        json.Unmarshal([]byte(notif.Payload), &ev)
        s.handler.Handle(ctx, ev)
    }
}
```

`notification`, `fiscal` e `inventory` têm cada um seu subscriber independente. Adicionar `loyalty_points` é criar um novo subscriber — sem tocar em `order`.

## Garantias e limitações

**O que o LISTEN/NOTIFY garante:**
- O NOTIFY só ocorre se a transação commitar (sem fantasmas)
- Entrega para todos os subscribers conectados no momento do NOTIFY
- Payload de até 8KB por notificação

**O que não garante:**
- **Entrega se o subscriber não estiver conectado**: se `notification` reiniciou durante o NOTIFY, o evento é perdido
- **Ordem entre diferentes canais**: dois NOTIFYs em canais diferentes podem chegar na ordem inversa
- **Exactly-once**: se o handler falha após receber mas antes de persistir o efeito, o evento não é reprocessado automaticamente

## Quando isso não é suficiente

Para casos onde você precisa de **replay** e **guaranteed delivery** (o subscriber pode estar offline por horas), o padrão transactional outbox é mais adequado:

```sql
CREATE TABLE outbox (
    id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    channel     text NOT NULL,
    payload     jsonb NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now(),
    sent_at     timestamptz
);
```

Um job separado lê a `outbox`, publica os eventos pendentes e marca `sent_at`. Garante que nenhum evento seja perdido mesmo com subscribers offline. O custo: polling periódico e complexidade extra.

Para o Comm, os eventos são de baixo volume e os subscribers são sempre locais (mesmo processo). A perda ocasional de um evento de notificação é aceitável — o cliente pode rever o pedido no portal. LISTEN/NOTIFY foi suficiente.

## Conclusão

O Postgres como event bus funciona bem para monolitos modulares onde:
- Os subscribers vivem no mesmo processo ou na mesma máquina
- O volume de eventos é moderado (milhares/hora, não milhões/segundo)
- A perda eventual de eventos em reinicializações é tolerável
- Você não quer operar um broker separado

Para escala horizontal e garantia de entrega, NATS JetStream ou Kafka são mais adequados — e é exatamente o que uso no [Mez](portfolio/mez-go/) para mensageria multicanal.
