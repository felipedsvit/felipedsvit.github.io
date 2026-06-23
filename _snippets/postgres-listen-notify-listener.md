---
title: "Go: Listener Postgres LISTEN/NOTIFY com reconnect automático"
tags: [go, postgres, event-driven]
date: 2026-01-20
---

Listener dedicado com backoff exponencial e reconexão automática. Extraído do [Erreia](https://github.com/felipedsvit/erreia/blob/main/internal/realtime/listener.go).

```go
import (
    "github.com/jackc/pgx/v5"
)

type Listener struct {
    dsn     string
    channel string
    handler func(payload string)
}

// Run bloqueia até ctx cancelar, reconectando com backoff após desconexões.
func (l *Listener) Run(ctx context.Context) {
    backoff := time.Second
    const maxBackoff = 30 * time.Second

    for {
        if ctx.Err() != nil {
            return
        }
        if err := l.runOnce(ctx); ctx.Err() != nil {
            return
        } else if err != nil {
            log.Printf("listener desconectado: %v, retry em %s", err, backoff)
            select {
            case <-ctx.Done():
                return
            case <-time.After(backoff):
            }
            backoff = min(backoff*2, maxBackoff)
        }
    }
}

func (l *Listener) runOnce(ctx context.Context) error {
    conn, err := pgx.Connect(ctx, l.dsn)
    if err != nil {
        return err
    }
    // context.Background() no defer: ctx pode estar cancelado, mas o Close precisa rodar
    defer func() { _ = conn.Close(context.Background()) }()

    // Sanitize evita SQL injection no nome do canal
    _, err = conn.Exec(ctx, "LISTEN "+pgx.Identifier{l.channel}.Sanitize())
    if err != nil {
        return err
    }

    for {
        notif, err := conn.WaitForNotification(ctx)
        if err != nil {
            return err // causa reconexão no loop externo
        }
        l.handler(notif.Payload)
    }
}
```

O NOTIFY correspondente no Postgres (dentro de uma transação):

```sql
-- Dispara junto com o INSERT/UPDATE — atomicidade garantida
SELECT pg_notify('meu_canal', json_build_object(
    'tipo', 'pedido_criado',
    'id',   NEW.id::text,
    'ts',   extract(epoch from now())::bigint
)::text);
```

> O Listener usa uma conexão **separada do pool** — o `LISTEN` precisa de uma conexão permanente dedicada. Não misture com conexões de query normal.
