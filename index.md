---
layout: home
title: Felipe Dsvit
description: Go developer especializado em arquitetura hexagonal, mensageria multicanal e HTMX.
---

<div class="hero">
  <div class="container">
    <h1 class="hero-title">Felipe Dsvit</h1>
    <p class="hero-subtitle">Go developer — arquitetura hexagonal, mensageria multicanal, HTMX.</p>
    <div class="hero-cta">
      <a href="/portfolio/" class="btn btn-primary">Ver projetos</a>
      <a href="/about/" class="btn btn-secondary">Sobre mim</a>
    </div>
  </div>
</div>

<div class="container">
  <section class="section">
    <h2 class="section-title">Projetos em destaque</h2>
    <div class="projects-grid">
      {% assign featured = site.data.projects | where: "featured", true %}
      {% for project in featured %}
        {% include project-card.html project=project %}
      {% endfor %}
    </div>
  </section>

  <section class="section">
    <h2 class="section-title">Últimos artigos</h2>
    {% if site.posts.size > 0 %}
    <ul class="post-list">
      {% for post in site.posts limit:5 %}
      <li class="post-list-item">
        <a class="post-list-title" href="{{ post.url | relative_url }}">{{ post.title }}</a>
        <time class="post-list-date" datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%d %b %Y" }}</time>
      </li>
      {% endfor %}
    </ul>
    <a href="/blog/" class="btn btn-secondary" style="margin-top:1rem;display:inline-block">Todos os artigos →</a>
    {% else %}
    <p style="color:var(--text-muted)">Em breve — artigos sobre Go, arquitetura e HTMX.</p>
    {% endif %}
  </section>

  <section class="section">
    <h2 class="section-title">Stack</h2>
    <div class="skills-grid">
      {% for group in site.data.skills %}
      <div class="skill-category">
        <h4>{{ group.category }}</h4>
        <ul>
          {% for item in group.items %}
          <li>{{ item }}</li>
          {% endfor %}
        </ul>
      </div>
      {% endfor %}
    </div>
  </section>
</div>
