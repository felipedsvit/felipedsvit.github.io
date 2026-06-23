---
layout: default
title: Recursos
description: Links e referências técnicas curadas — Go, arquitetura, HTMX, PostgreSQL, mensageria.
---

<div class="container" style="padding-top:3rem;padding-bottom:3rem">

<h1>Recursos</h1>
<p style="color:var(--text-muted);margin-bottom:2.5rem">Links e referências que uso regularmente, organizados por área.</p>

{% for category in site.data.bookmarks %}
<section style="margin-bottom:2.5rem">
  <h2 class="section-title">{{ category.name }}</h2>
  <div class="resources-grid">
    {% for item in category.items %}
    <a class="resource-card" href="{{ item.url }}" target="_blank" rel="noopener">
      <span class="resource-title">{{ item.title }}</span>
      <span class="resource-desc">{{ item.description }}</span>
      <span class="resource-url">{{ item.url | remove: "https://" | truncate: 40 }}</span>
    </a>
    {% endfor %}
  </div>
</section>
{% endfor %}

</div>

<style>
.resources-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(240px, 1fr));
  gap: 1rem;
}
.resource-card {
  display: flex;
  flex-direction: column;
  gap: .4rem;
  background: var(--bg-secondary);
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 1rem 1.15rem;
  text-decoration: none;
  transition: border-color .15s;
}
.resource-card:hover {
  border-color: var(--accent);
  text-decoration: none;
}
.resource-title {
  font-weight: 600;
  font-size: .9rem;
  color: var(--text);
}
.resource-desc {
  font-size: .825rem;
  color: var(--text-muted);
  line-height: 1.4;
}
.resource-url {
  font-size: .75rem;
  color: var(--accent);
  margin-top: auto;
  font-family: 'SFMono-Regular', Consolas, monospace;
}
</style>
