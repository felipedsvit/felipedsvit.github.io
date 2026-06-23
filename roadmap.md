---
layout: default
title: Roadmap
description: O que está em desenvolvimento e o que vem por aí.
---

<div class="container" style="padding-top:3rem;padding-bottom:3rem">

<h1>Roadmap</h1>
<p style="color:var(--text-muted);margin-bottom:2rem">Status dos projetos ativos e próximos passos.</p>

<h2 class="section-title">Projetos ativos</h2>

<div class="roadmap-grid">
  <div class="roadmap-card">
    <div class="roadmap-card-header">
      <span class="badge badge-active">ativo</span>
      <span class="badge badge-public">público</span>
    </div>
    <h3><a href="/portfolio/erreia/">Erreia</a></h3>
    <p>Kanban leve em Go com SSE via Postgres LISTEN/NOTIFY.</p>
    <ul class="roadmap-list">
      <li class="done">SSE hub + Postgres LISTEN/NOTIFY</li>
      <li class="done">Auth por sessão com Argon2id</li>
      <li class="done">Avatares no MinIO</li>
      <li class="done">Binário único via embed.FS</li>
      <li class="todo">Compartilhamento de boards</li>
      <li class="todo">Labels e filtros em cards</li>
      <li class="todo">Release v1.0 com changelog automático</li>
    </ul>
  </div>

  <div class="roadmap-card">
    <div class="roadmap-card-header">
      <span class="badge badge-active">ativo</span>
      <span class="badge badge-private">privado</span>
    </div>
    <h3><a href="/portfolio/mez-go/">Mez</a></h3>
    <p>Plataforma de mensageria multicanal — WhatsApp, Telegram, Instagram, Messenger.</p>
    <ul class="roadmap-list">
      <li class="done">Adaptador WhatsApp (whatsmeow)</li>
      <li class="done">Adaptador Telegram</li>
      <li class="done">NATS JetStream como broker</li>
      <li class="done">Multi-tenant com Postgres RLS</li>
      <li class="todo">Adaptador Instagram Graph API</li>
      <li class="todo">Adaptador Messenger</li>
      <li class="todo">Dashboard de métricas por canal</li>
    </ul>
  </div>

  <div class="roadmap-card">
    <div class="roadmap-card-header">
      <span class="badge badge-active">ativo</span>
      <span class="badge badge-private">privado</span>
    </div>
    <h3><a href="/portfolio/comm/">Comm</a></h3>
    <p>Monolito modular para e-commerce com checkout, pagamentos e emissão fiscal.</p>
    <ul class="roadmap-list">
      <li class="done">Catálogo e carrinho</li>
      <li class="done">Checkout + Stripe + Mercado Pago</li>
      <li class="done">Event bus via Postgres LISTEN/NOTIFY</li>
      <li class="todo">Emissão de NF-e</li>
      <li class="todo">Integração Mercado Livre</li>
      <li class="todo">KYC para compra de réplicas</li>
    </ul>
  </div>

  <div class="roadmap-card">
    <div class="roadmap-card-header">
      <span class="badge badge-active">ativo</span>
      <span class="badge badge-public">público</span>
    </div>
    <h3><a href="/portfolio/htmx-4-specialist/">HTMX 4 Specialist</a></h3>
    <p>Skill para LLMs especializado em HTMX 4.0 e HDA.</p>
    <ul class="roadmap-list">
      <li class="done">Referência completa de atributos</li>
      <li class="done">Padrões HDA documentados</li>
      <li class="done">Evals de qualidade</li>
      <li class="todo">Agente comparador 2.x vs 4.0</li>
      <li class="todo">Exemplos com Templ</li>
    </ul>
  </div>
</div>

<h2 class="section-title" style="margin-top:3rem">Próximos projetos</h2>

<div class="roadmap-grid">
  <div class="roadmap-card roadmap-card-future">
    <div class="roadmap-card-header">
      <span class="badge" style="background:var(--bg-tertiary);color:var(--text-muted)">planejado</span>
    </div>
    <h3>go-hexkit</h3>
    <p>Biblioteca de utilitários para arquitetura hexagonal em Go — portas, adaptadores, transactional outbox.</p>
  </div>

  <div class="roadmap-card roadmap-card-future">
    <div class="roadmap-card-header">
      <span class="badge" style="background:var(--bg-tertiary);color:var(--text-muted)">planejado</span>
    </div>
    <h3>template-htmx-go</h3>
    <p>Starter kit para apps HTMX 4.0 + Templ + Go com SSE, dark mode e deploy em binário único.</p>
  </div>
</div>

<p style="margin-top:2rem;color:var(--text-muted);font-size:.875rem">
  Detalhes e issues em <a href="https://github.com/felipedsvit?tab=projects">GitHub Projects</a>.
</p>

</div>

<style>
.roadmap-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 1.25rem;
}
.roadmap-card {
  background: var(--bg-secondary);
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 1.4rem;
  display: flex;
  flex-direction: column;
  gap: .75rem;
}
.roadmap-card-future {
  border-style: dashed;
  opacity: .8;
}
.roadmap-card-header {
  display: flex;
  gap: .4rem;
}
.roadmap-card h3 {
  margin: 0;
  font-size: 1.05rem;
}
.roadmap-card h3 a {
  color: var(--text);
  text-decoration: none;
}
.roadmap-card h3 a:hover { color: var(--accent); }
.roadmap-card p {
  color: var(--text-muted);
  font-size: .875rem;
  margin: 0;
}
.roadmap-list {
  list-style: none;
  padding: 0;
  margin: 0;
  display: flex;
  flex-direction: column;
  gap: .3rem;
}
.roadmap-list li {
  font-size: .875rem;
  padding-left: 1.4rem;
  position: relative;
  color: var(--text-muted);
}
.roadmap-list li::before {
  position: absolute;
  left: 0;
}
.roadmap-list li.done { color: var(--text); }
.roadmap-list li.done::before { content: '✓'; color: var(--badge-pub-fg); }
.roadmap-list li.todo::before { content: '○'; }
</style>
