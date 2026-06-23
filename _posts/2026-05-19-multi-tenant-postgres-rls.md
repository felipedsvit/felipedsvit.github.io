---
layout: post
title: "Multi-tenancy com Postgres RLS — isolamento real no nível do banco"
description: Como implementar Row Level Security no Postgres para isolamento multi-tenant em Go, sem WHERE tenantID em toda query, sem middleware de aplicação.
date: 2026-05-19
tags: [go, postgres, multi-tenant, rls, arquitetura, mez]
---

No Mez, cada empresa cliente é um tenant. Um tenant não pode ver as conversas, contatos ou mensagens de outro tenant. Em sistemas de mensageria multicanal isso não é opcional — é exigência legal e de segurança.

A abordagem comum é adicionar `WHERE tenant_id = $1` em toda query. Funciona, mas é frágil: uma query nova sem o filtro expõe dados de todos os tenants. O desenvolvedor precisa lembrar de adicionar o filtro em 100% das queries, 100% das vezes.

O Postgres tem uma solução melhor: Row Level Security.

## O que é RLS

RLS permite definir políticas no nível da tabela que filtram automaticamente as linhas visíveis para cada conexão. A política é executada pelo banco, não pela aplicação. Uma query sem `WHERE tenant_id` simplesmente retorna zero linhas para um tenant incorreto — não vaza dados.

```sql
-- Habilita RLS na tabela de conversas
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;

-- Política: só vê conversas do seu tenant
CREATE POLICY tenant_isolation ON conversations
    USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

`current_setting('app.tenant_id')` lê uma variável de sessão que definimos antes de cada query. O banco aplica o filtro em todo SELECT, UPDATE, DELETE e INSERT.

## Propagando o tenant em Go

O tenant vem do JWT ou da sessão HTTP. Precisamos propagá-lo para o Postgres antes de executar qualquer query no request.

```go
// adapters/postgres/tenant.go

// WithTenant retorna uma conexão do pool com o tenant_id configurado.
// A variável é local à transação/sessão e não vaza entre requests.
func WithTenant(ctx context.Context, pool *pgxpool.Pool, tenantID uuid.UUID) (pgx.Conn, func(), error) {
    conn, err := pool.Acquire(ctx)
    if err != nil {
        return nil, nil, err
    }

    _, err = conn.Exec(ctx,
        "SELECT set_config('app.tenant_id', $1, true)", // true = local à transação
        tenantID.String(),
    )
    if err != nil {
        conn.Release()
        return nil, nil, err
    }

    return conn.Conn(), conn.Release, nil
}
```

O terceiro parâmetro `true` em `set_config` define que a variável é local à transação — ela é revertida no final da transação. Isso garante que a conexão, quando devolvida ao pool, não carrega o tenant_id do request anterior.

## Middleware HTTP

```go
// internal/web/middleware.go

func TenantMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        tenantID, err := extractTenantFromJWT(r)
        if err != nil {
            http.Error(w, "unauthorized", http.StatusUnauthorized)
            return
        }
        ctx := context.WithValue(r.Context(), tenantKey, tenantID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// No repositório: pega o tenant do contexto
func (r *ConversationRepo) List(ctx context.Context) ([]*Conversation, error) {
    tenantID := ctx.Value(tenantKey).(uuid.UUID)

    conn, release, err := postgres.WithTenant(ctx, r.pool, tenantID)
    defer release()

    // Essa query não tem WHERE tenant_id — o RLS aplica automaticamente
    rows, err := conn.Query(ctx, "SELECT * FROM conversations ORDER BY created_at DESC")
    // ...
}
```

O repositório não sabe que RLS existe. Ele só executa a query — o banco aplica o filtro.

## Políticas para operações diferentes

Leitura e escrita podem ter políticas diferentes:

```sql
-- Apenas lê conversas do próprio tenant
CREATE POLICY conversations_select ON conversations
    FOR SELECT
    USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- Só pode inserir conversas no próprio tenant
CREATE POLICY conversations_insert ON conversations
    FOR INSERT
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);

-- Não pode deletar (só soft delete via UPDATE)
CREATE POLICY conversations_delete ON conversations
    FOR DELETE
    USING (false);
```

O `WITH CHECK` na política de INSERT garante que mesmo um código bugado que tenta inserir com um `tenant_id` diferente vai falhar — o banco rejeita a linha.

## Superuser e operações administrativas

O RLS é bypassado por superusers e por roles com `BYPASSRLS`. Para migrations e operações administrativas, use um role dedicado:

```sql
-- Role de aplicação — sujeito ao RLS
CREATE ROLE app_user;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO app_user;

-- Role administrativo — bypassa RLS
CREATE ROLE admin_user BYPASSRLS;
```

A aplicação conecta como `app_user`. As migrations rodam como `admin_user` (ou superuser). Nunca use superuser na connection string da aplicação.

## Testando o isolamento

```go
func TestTenantIsolation(t *testing.T) {
    // Cria dois tenants e uma conversa para cada
    tenant1 := createTenant(t, pool)
    tenant2 := createTenant(t, pool)

    createConversation(t, pool, tenant1, "conversa do tenant 1")
    createConversation(t, pool, tenant2, "conversa do tenant 2")

    // Conecta como tenant1 e lista conversas
    conn, release, _ := postgres.WithTenant(ctx, pool, tenant1)
    defer release()

    rows, _ := conn.Query(ctx, "SELECT title FROM conversations")
    var titles []string
    // ...

    // tenant1 só vê sua própria conversa
    assert.Equal(t, []string{"conversa do tenant 1"}, titles)
}
```

Esse teste falha se o RLS não estiver configurado corretamente — não é possível "esquecer" de testar porque o isolamento é a propriedade fundamental do sistema.

## Vantagens e custos

**Vantagens:**
- Isolamento garantido pelo banco — não depende de código de aplicação
- Uma query sem filtro retorna zero linhas, não vaza dados
- Auditoria trivial: qualquer log de query SQL já inclui o tenant implicitamente

**Custos:**
- `set_config` antes de cada operação — overhead mínimo (~0.1ms), mas existe
- Ferramentas de admin (psql, DBeaver) precisam do `set_config` para ver dados como tenant
- Diagnóstico de "por que não vejo esse registro" pode confundir quem não conhece RLS

Para sistemas multi-tenant onde a separação de dados é requisito de segurança, esse custo é trivial comparado ao risco de um `WHERE tenant_id` esquecido.
