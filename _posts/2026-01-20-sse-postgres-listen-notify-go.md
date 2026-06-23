---
layout: post
title: "SSE em Go sem WebSocket: Postgres LISTEN/NOTIFY como message broker"
description: Como o Erreia entrega realtime para múltiplos clientes usando SSE + Postgres LISTEN/NOTIFY — sem Redis, sem NATS, sem infraestrutura extra.
date: 2026-01-20
tags: [go, postgres, sse, htmx, erreia]
---

No [Erreia](https://github.com/felipedsvit/erreia), cada aba aberta recebe atualizações em tempo real quando um card é movido, criado ou deletado. Sem polling. Sem WebSocket. Sem broker externo. Apenas Postgres e SSE.

## O problema

Precisava de realtime num Kanban simples. As alternativas óbvias têm custo:

- **WebSocket**: conexão bidirecional, protocolo de upgrade, precisa de suporte no proxy reverso
- **Redis Pub/Sub**: mais uma peça de infra pra operar
- **NATS**: ótimo, mas overkill pra um app que já tem Postgres

O Postgres tem `LISTEN/NOTIFY` desde os anos 90. É transacional, confiável e já está lá.

## A arquitetura

```
┌─────────────────────────────────────────────┐
│  Browser (tab 1)   Browser (tab 2)          │
│       │                   │                 │
│    SSE /events/boardID  SSE /events/boardID │
└───────┼───────────────────┼─────────────────┘
        │                   │
    ┌───▼───────────────────▼───┐
    │         Hub (in-memory)   │
    │   rooms[boardID] = {ch1, ch2} │
    └───────────────┬───────────┘
                    │ Broadcast(ev)
    ┌───────────────▼───────────┐
    │       Listener            │
    │  conn.WaitForNotification │
    └───────────────┬───────────┘
                    │
    ┌───────────────▼───────────┐
    │       PostgreSQL          │
    │  NOTIFY board_events '…'  │
    └───────────────────────────┘
```

Três peças: **Listener**, **Hub**, **SSE handler**. Cada uma com responsabilidade única.

## O Listener

O `Listener` mantém **uma única conexão** dedicada ao `LISTEN`. Não usa o pool — o pool é para queries; `LISTEN` precisa de uma conexão permanente própria.

```go
func (l *Listener) Run(ctx context.Context) {
    backoff := time.Second
    const maxBackoff = 30 * time.Second
    for {
        if ctx.Err() != nil {
            return
        }
        err := l.runOnce(ctx)
        if ctx.Err() != nil {
            return
        }
        l.logger.Warn("listener disconnected", "err", err, "retry_in", backoff)
        select {
        case <-ctx.Done():
            return
        case <-time.After(backoff):
        }
        backoff *= 2
        if backoff > maxBackoff {
            backoff = maxBackoff
        }
    }
}

func (l *Listener) runOnce(ctx context.Context) error {
    conn, err := pgx.Connect(ctx, l.dsn)
    if err != nil {
        return err
    }
    defer func() { _ = conn.Close(context.Background()) }()

    if _, err := conn.Exec(ctx, "LISTEN "+pgx.Identifier{l.channel}.Sanitize()); err != nil {
        return err
    }
    for {
        notif, err := conn.WaitForNotification(ctx)
        if err != nil {
            return err
        }
        ev, err := DecodeEvent(notif.Payload)
        if err != nil {
            l.logger.Warn("decode notify payload", "err", err, "raw", notif.Payload)
            continue
        }
        l.hub.Broadcast(ev)
    }
}
```

Notas importantes:
- `pgx.Identifier{l.channel}.Sanitize()` — evita SQL injection no nome do canal
- Backoff exponencial com cap de 30s — não derruba o banco em reconexões rápidas
- `context.Background()` no `defer Close` — o contexto pai já cancelou, mas o close precisa executar

## O Hub

O `Hub` é um fan-out: recebe um evento e distribui para todos os clientes subscritos no mesmo board.

```go
type Hub struct {
    mu      sync.RWMutex
    rooms   map[string]map[chan Event]struct{}
    logger  *slog.Logger
    dropped atomic.Uint64
}

func (h *Hub) Subscribe(boardID string) (<-chan Event, func()) {
    ch := make(chan Event, 4)
    h.mu.Lock()
    if _, ok := h.rooms[boardID]; !ok {
        h.rooms[boardID] = make(map[chan Event]struct{})
    }
    h.rooms[boardID][ch] = struct{}{}
    h.mu.Unlock()

    cancel := func() {
        h.mu.Lock()
        if room, ok := h.rooms[boardID]; ok {
            delete(room, ch)
            if len(room) == 0 {
                delete(h.rooms, boardID)
            }
        }
        h.mu.Unlock()
    }
    return ch, cancel
}

func (h *Hub) Broadcast(ev Event) {
    h.mu.RLock()
    room, ok := h.rooms[ev.BoardID]
    if !ok {
        h.mu.RUnlock()
        return
    }
    clients := make([]chan Event, 0, len(room))
    for c := range room {
        clients = append(clients, c)
    }
    h.mu.RUnlock()

    for _, c := range clients {
        select {
        case c <- ev:
        default:
            h.dropped.Add(1)
            h.logger.Warn("sse client too slow, dropping event", "board", ev.BoardID)
        }
    }
}
```

Duas decisões que importam:

**Buffer pequeno (4) e drop não-bloqueante**: se o cliente está lento (rede ruim, aba em background), o evento é descartado. O cliente vai recarregar no próximo foco. Isso garante que um cliente lento nunca bloqueia o broadcaster.

**Cópia dos clientes antes de enviar**: o `RLock` libera antes dos envios. Isso evita que um envio lento segure o lock e bloqueie novos subscribes ou o Broadcast de outros eventos.

**O canal não é fechado no cancel**: fechar um canal com sends em andamento causa panic. O canal é simplesmente removido do mapa e coletado pelo GC quando ninguém mais o referencia.

## O SSE handler

```go
func (s *Server) handleBoardEvents(w http.ResponseWriter, r *http.Request) {
    // ... auth check ...

    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "streaming unsupported", http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("X-Accel-Buffering", "no")
    w.WriteHeader(http.StatusOK)
    flusher.Flush()

    ch, cancel := s.deps.Hub.Subscribe(boardID)
    defer cancel()

    pingTicker := time.NewTicker(20 * time.Second)
    defer pingTicker.Stop()

    for {
        select {
        case <-r.Context().Done():
            return
        case <-pingTicker.C:
            fmt.Fprint(w, ": ping\n\n")
            flusher.Flush()
        case ev := <-ch:
            s.writeEvent(w, flusher, r, ev, csrfToken)
        }
    }
}
```

O `X-Accel-Buffering: no` é obrigatório com Nginx — sem ele o reverse proxy bufferiza a resposta e o cliente não recebe os eventos em tempo real.

O ping a cada 20s mantém a conexão viva através de proxies que fecham conexões idle.

## O NOTIFY no banco

Num trigger após cada INSERT/UPDATE em `cards` e `columns`:

```sql
CREATE OR REPLACE FUNCTION notify_board_event() RETURNS trigger AS $$
BEGIN
  PERFORM pg_notify(
    'board_events',
    json_build_object(
      'b',  NEW.board_id,
      'a',  TG_ARGV[0],
      'id', NEW.id,
      'v',  extract(epoch from now())::bigint
    )::text
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

O payload é JSON compacto (`b` = boardID, `a` = action). O Postgres tem limite de 8KB por NOTIFY — suficiente para IDs e metadados, mas não para conteúdo completo. O handler busca o dado completo via query quando recebe o evento.

## Conclusão

O padrão `LISTEN/NOTIFY + SSE` é ideal quando:
- Você já tem Postgres e não quer operar um broker extra
- A escala não exige múltiplas instâncias do servidor (ou você aceita eventos locais por instância)
- Os eventos são sinais de invalidação, não payloads grandes

O custo: uma conexão permanente dedicada ao `LISTEN`. O benefício: zero infra extra, zero latência de rede entre app e broker.

O código completo está em [github.com/felipedsvit/erreia](https://github.com/felipedsvit/erreia).
