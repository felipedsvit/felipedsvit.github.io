---
title: "SQL: schema Kanban com posicionamento e soft-delete"
tags: [sql, postgres, schema]
date: 2026-01-20
---

Schema de boards, colunas e cards com UUID, posicionamento por inteiro e índices adequados. Extraído das [migrations do Erreia](https://github.com/felipedsvit/erreia).

```sql
-- Users
CREATE TABLE IF NOT EXISTS users (
    id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    email           text UNIQUE NOT NULL,
    password_hash   text NOT NULL,
    display_name    text NOT NULL,
    avatar_key      text NOT NULL DEFAULT '',
    created_at      timestamptz NOT NULL DEFAULT now(),
    updated_at      timestamptz NOT NULL DEFAULT now()
);

-- Sessions (compatível com alexedwards/scs)
CREATE TABLE IF NOT EXISTS sessions (
    token   text PRIMARY KEY,
    data    bytea NOT NULL,
    expiry  timestamptz NOT NULL
);
CREATE INDEX IF NOT EXISTS sessions_expiry_idx ON sessions (expiry);

-- Boards
CREATE TABLE IF NOT EXISTS boards (
    id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_id   uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title      text NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS boards_owner_idx ON boards (owner_id);

-- Columns
CREATE TABLE IF NOT EXISTS columns (
    id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    board_id   uuid NOT NULL REFERENCES boards(id) ON DELETE CASCADE,
    title      text NOT NULL,
    position   int  NOT NULL,
    created_at timestamptz NOT NULL DEFAULT now(),
    updated_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS columns_board_position_idx ON columns (board_id, position);

-- Cards
CREATE TABLE IF NOT EXISTS cards (
    id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    column_id   uuid NOT NULL REFERENCES columns(id) ON DELETE CASCADE,
    title       text NOT NULL,
    description text NOT NULL DEFAULT '',
    position    int  NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now(),
    updated_at  timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX IF NOT EXISTS cards_column_position_idx ON cards (column_id, position);
```

Trigger de NOTIFY para realtime (dispara após INSERT/UPDATE em cards):

```sql
CREATE OR REPLACE FUNCTION notify_board_event() RETURNS trigger AS $$
DECLARE
    board_id uuid;
BEGIN
    SELECT c.board_id INTO board_id
    FROM columns c WHERE c.id = NEW.column_id;

    PERFORM pg_notify('board_events', json_build_object(
        'b',  board_id,
        'a',  TG_ARGV[0],
        'id', NEW.id,
        'c',  NEW.column_id,
        'p',  NEW.position,
        'v',  extract(epoch from now())::bigint
    )::text);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER cards_notify_insert
    AFTER INSERT ON cards
    FOR EACH ROW EXECUTE FUNCTION notify_board_event('card-created');

CREATE TRIGGER cards_notify_update
    AFTER UPDATE ON cards
    FOR EACH ROW EXECUTE FUNCTION notify_board_event('card-updated');
```
