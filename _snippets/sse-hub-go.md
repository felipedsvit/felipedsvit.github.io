---
title: "Go: SSE Hub com fan-out e drop não-bloqueante"
tags: [go, sse, concorrência]
date: 2026-01-20
---

Hub de Server-Sent Events com `sync.RWMutex`, buffer pequeno e drop de clientes lentos. Extraído do [Erreia](https://github.com/felipedsvit/erreia/blob/main/internal/realtime/hub.go).

```go
type Hub struct {
    mu      sync.RWMutex
    rooms   map[string]map[chan Event]struct{}
    dropped atomic.Uint64
}

func NewHub() *Hub {
    return &Hub{rooms: make(map[string]map[chan Event]struct{})}
}

// Subscribe registra um cliente para roomID.
// Retorna canal read-only e função de cancelamento.
// O canal NÃO é fechado no cancel — fechar com send pendente causa panic.
func (h *Hub) Subscribe(roomID string) (<-chan Event, func()) {
    ch := make(chan Event, 4) // buffer pequeno: cliente lento é dropado rápido
    h.mu.Lock()
    if _, ok := h.rooms[roomID]; !ok {
        h.rooms[roomID] = make(map[chan Event]struct{})
    }
    h.rooms[roomID][ch] = struct{}{}
    h.mu.Unlock()

    return ch, func() {
        h.mu.Lock()
        if room, ok := h.rooms[roomID]; ok {
            delete(room, ch)
            if len(room) == 0 {
                delete(h.rooms, roomID)
            }
        }
        h.mu.Unlock()
    }
}

// Broadcast distribui ev para todos os subscribers do ev.RoomID.
// Libera o RLock antes dos envios — cliente lento não trava outras rooms.
func (h *Hub) Broadcast(ev Event) {
    h.mu.RLock()
    room, ok := h.rooms[ev.RoomID]
    if !ok {
        h.mu.RUnlock()
        return
    }
    clients := make([]chan Event, 0, len(room))
    for c := range room {
        clients = append(clients, c)
    }
    h.mu.RUnlock() // libera antes dos sends

    for _, c := range clients {
        select {
        case c <- ev:
        default:
            h.dropped.Add(1) // cliente lento: descarta o evento
        }
    }
}
```

Handler SSE que usa o Hub:

```go
func handleSSE(hub *Hub, roomID string) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        flusher := w.(http.Flusher)
        w.Header().Set("Content-Type", "text/event-stream")
        w.Header().Set("Cache-Control", "no-cache")
        w.Header().Set("X-Accel-Buffering", "no") // desabilita buffer do Nginx
        flusher.Flush()

        ch, cancel := hub.Subscribe(roomID)
        defer cancel()

        ping := time.NewTicker(20 * time.Second)
        defer ping.Stop()

        for {
            select {
            case <-r.Context().Done():
                return
            case <-ping.C:
                fmt.Fprint(w, ": ping\n\n")
                flusher.Flush()
            case ev := <-ch:
                fmt.Fprintf(w, "event: %s\ndata: %s\n\n", ev.Type, ev.Data)
                flusher.Flush()
            }
        }
    }
}
```
