---
title: "HTMX 4.0: padrões essenciais — search, SSE, inline edit, polling"
tags: [htmx, htmx4, frontend, hda]
date: 2026-02-17
---

Padrões do dia-a-dia com HTMX 4.0. Extraídos do [htmx-4-specialist](https://github.com/felipedsvit/htmx-4-specialist).

## Busca com debounce

```html
<input type="search"
       name="q"
       hx-get="/search"
       hx-trigger="input changed delay:500ms, search"
       hx-target="#results"
       hx-indicator="#spinner">

<img id="spinner" class="htmx-indicator" src="/spinner.gif" alt="Buscando...">
<div id="results" aria-live="polite"></div>
```

- `changed` — só dispara se o valor mudou
- `delay:500ms` — debounce; reseta a cada keystroke
- `, search` — também dispara no evento `search` (limpar campo com ×)
- No 4.0, requests anteriores são cancelados automaticamente via `AbortController`

## Herança explícita com `:inherited`

```html
<!-- No 4.0, herança implícita foi removida. Use :inherited -->
<tbody hx-confirm:inherited="Deletar registro?"
       hx-target:inherited="#resultado">
  <tr>
    <td>Item 1</td>
    <td><button hx-delete="/items/1" hx-swap="delete">Deletar</button></td>
  </tr>
</tbody>
```

## Validação inline (4xx faz swap por padrão)

```html
<form hx-post="/signup" hx-swap="outerHTML">
  <input name="email" type="email">
  <button>Criar conta</button>
</form>
```

Servidor retorna `422` com o formulário re-renderizado + erros — o HTMX faz o swap automaticamente. Sem extensão, sem JavaScript extra.

## SSE com HTMX

{% raw %}
```html
<div hx-ext="sse" sse-connect="/events/{{ .BoardID }}">
  <div sse-swap="card-updated"
       hx-target="#cards"
       hx-swap="outerHTML settle:0ms">
  </div>
</div>
<div id="cards">{{ range .Cards }}...{{ end }}</div>
```
{% endraw %}

O servidor envia:
```
event: card-updated
data: <div id="cards">...HTML atualizado...</div>

```

## Inline edit (click-to-edit)

{% raw %}
```html
<!-- Modo visualização: clique troca para o form -->
<div hx-get="/users/1/edit" hx-trigger="click" hx-swap="outerHTML">
  <p>{{ .User.Name }} <em>clique para editar</em></p>
</div>
```

```html
<!-- Servidor retorna o form -->
<form hx-put="/users/1" hx-swap="outerHTML">
  <input name="name" value="{{ .User.Name }}">
  <button>Salvar</button>
  <button type="button" hx-get="/users/1" hx-swap="outerHTML">Cancelar</button>
</form>
```
{% endraw %}

## Polling com parada

{% raw %}
```html
<!-- Poll enquanto o job estiver em andamento -->
<div hx-get="/jobs/{{ .ID }}/status"
     hx-trigger="load delay:2s"
     hx-swap="outerHTML">
  <progress value="{{ .Progress }}" max="100"></progress>
  <span>{{ .Progress }}%...</span>
</div>
```
{% endraw %}

Quando o job termina, o servidor retorna um div **sem** `hx-trigger` — o polling para automaticamente.

## Lazy loading

```html
<div hx-get="/widgets/recent-orders"
     hx-trigger="load"
     hx-swap="outerHTML">
  <p class="loading">Carregando...</p>
</div>
```

## Anti-padrões do 4.0

```html
<!-- ❌ herança implícita não funciona mais -->
<div hx-confirm="Tem certeza?">
  <button hx-delete="/item/1">Deletar</button>
</div>

<!-- ✅ correto -->
<div hx-confirm:inherited="Tem certeza?">
  <button hx-delete="/item/1">Deletar</button>
</div>

<!-- ❌ não retorne JSON — retorne HTML -->
<!-- ✅ retorne fragmentos HTML direto no body da resposta -->
```
