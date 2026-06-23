---
layout: post
title: "HTMX 4.0: o que quebrou do 2.x e como migrar"
description: As mudanças mais impactantes do HTMX 4.0 — herança implícita, swap de 4xx/5xx, hx-sync e anti-padrões que você provavelmente já cometeu.
date: 2026-02-17
tags: [htmx, htmx4, frontend, hda]
---

O HTMX 4.0 não é drop-in replacement do 2.x. Há mudanças de comportamento que quebram silenciosamente — sem erro no console, só comportamento errado. Este artigo cobre as mais impactantes, baseado no [htmx-4-specialist](https://github.com/felipedsvit/htmx-4-specialist).

## 1. Herança implícita foi removida — use `:inherited`

No 2.x, atributos como `hx-confirm`, `hx-target` e `hx-boost` em um elemento pai eram herdados pelos filhos automaticamente. No 4.0, isso não existe mais.

**2.x (funcionava):**
```html
<tbody hx-confirm="Deletar?" hx-target="#result">
  <tr>
    <td><button hx-delete="/users/1">Deletar</button></td>
  </tr>
</tbody>
```

**4.0 (o botão ignora o `hx-confirm` do pai):**
```html
<!-- mesmo código, comportamento diferente -->
```

**4.0 correto — use `:inherited`:**
```html
<tbody hx-confirm:inherited="Deletar?" hx-target:inherited="#result">
  <tr>
    <td><button hx-delete="/users/1" hx-swap="delete">Deletar</button></td>
  </tr>
</tbody>
```

O sufixo `:inherited` é explícito por design. Herança implícita era fonte de bugs difíceis de rastrear — quem lê o HTML do botão não sabe de onde vem o `hx-confirm`.

**`hx-boost` também muda:**
```html
<!-- 2.x: todos os links viram AJAX -->
<nav hx-boost="true">...</nav>

<!-- 4.0: precisa ser explícito -->
<nav hx-boost:inherited="true">...</nav>
```

## 2. Respostas 4xx/5xx fazem swap por padrão

No 2.x, erros HTTP eram ignorados silenciosamente — o DOM não mudava. Você precisava da extensão `htmx-ext-response-targets` para interceptar e exibir erros de validação.

No 4.0, **4xx/5xx fazem swap no target padrão**. Isso muda como você estrutura validação de formulários:

**Padrão de validação no 4.0:**
```html
<form hx-post="/signup" hx-swap="outerHTML">
  <input name="email" type="email">
  <button>Criar conta</button>
</form>
```

```go
// servidor retorna 422 com o formulário re-renderizado com erros
func handleSignup(w http.ResponseWriter, r *http.Request) {
    if userExists(email) {
        w.WriteHeader(http.StatusUnprocessableEntity) // 422
        templates.SignupForm(FormData{
            Email:  email,
            Errors: map[string]string{"email": "Email já cadastrado"},
        }).Render(r.Context(), w)
        return
    }
    // ...
}
```

O 422 retorna o mesmo fragmento HTML do formulário, agora com as mensagens de erro. O HTMX substitui o `<form>` pelo fragmento retornado — sem JavaScript extra.

**Atenção**: retorne HTML, não JSON. O HTMX renderiza o `data` da resposta diretamente no DOM.

Para swap parcial (só o campo com erro, não o form inteiro):
```html
<form hx-post="/signup" hx-swap="outerHTML"
      hx-status:422="target:#email-error select:.error-msg">
  <input name="email">
  <div id="email-error">
    <!-- o servidor retorna .error-msg aqui -->
  </div>
</form>
```

## 3. `hx-sync` mudou de sintaxe

No 2.x, você controlava requests concorrentes com `hx-sync="this:replace"`. No 4.0, a sintaxe é a mesma mas o comportamento padrão mudou.

**Busca com debounce — 4.0:**
```html
<input type="search"
       name="q"
       hx-get="/search"
       hx-trigger="input changed delay:500ms, search"
       hx-target="#results">
```

No 4.0, requests em andamento são automaticamente cancelados quando um novo é disparado (via `AbortController`). Você não precisa mais de `hx-sync="this:abort"` para o caso mais comum.

**Quando ainda usar `hx-sync`:** coordenação entre elementos diferentes.
```html
<!-- validação não deve cancelar o submit -->
<form hx-post="/save" hx-sync="this:replace">
  <input name="title"
         hx-post="/validate"
         hx-trigger="change"
         hx-sync="closest form:abort">
</form>
```

## 4. Eventos do ciclo de vida mudaram de nome

Se você tem JavaScript ouvindo eventos do HTMX:

| 2.x | 4.0 |
|-----|-----|
| `htmx:xhr:loadend` | `htmx:after:request` |
| `htmx:beforeSwap` | `htmx:before:swap` |
| `htmx:afterSettle` | `htmx:after:settle` |
| `htmx:responseError` | `htmx:response:error` |

O formato mudou para `htmx:fase:evento` (com dois-pontos). Grep no seu código por `addEventListener('htmx:` antes de migrar.

**`hx-on` inline também muda:**
```html
<!-- 2.x -->
<button hx-on="htmx:afterRequest: console.log('done')">

<!-- 4.0 -->
<button hx-on:htmx:after:request="console.log('done')">
```

## 5. `hx-delete` não inclui o form pai

No 2.x, um `<button hx-delete>` dentro de um `<form>` incluía os inputs do formulário no request. No 4.0, não inclui mais.

```html
<!-- 4.0: o delete NÃO envia os inputs do form -->
<form>
  <input name="reason" value="spam">
  <button hx-delete="/users/1">Deletar</button>
</form>

<!-- 4.0: para incluir, seja explícito -->
<button hx-delete="/users/1" hx-include="closest form">Deletar</button>
```

## Anti-padrões que o HTMX 4.0 torna mais visíveis

**Retornar JSON:** o HTMX renderiza o `data` no DOM diretamente. Se você retorna JSON, o usuário vê `{"id":1,"name":"..."}` no lugar do HTML.

**Usar `hx-boost` sem `:inherited` e esperar que funcione:** vai quebrar silenciosamente — os links voltam a ser navegações completas.

**Usar `htmx-ext-response-targets` para 4xx:** a extensão não é necessária no 4.0 e pode conflitar com o comportamento padrão.

**`hx-trigger="every 1s"` sem condição de parada:** o servidor deve retornar o elemento sem o `hx-trigger` quando o job termina. Sem isso, o polling nunca para.

## Conclusão

O 4.0 é mais explícito, mais seguro e mais previsível que o 2.x. A remoção da herança implícita elimina uma categoria inteira de bugs. O swap de 4xx/5xx simplifica validação de formulários sem extensões. Vale a migração — mas faça com grep e testes, não na fé.

A referência completa dos atributos, eventos e padrões está em [htmx-4-specialist](https://github.com/felipedsvit/htmx-4-specialist).
