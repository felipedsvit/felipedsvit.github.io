Usa este prompt. Está desenhado para fazer outra IA agir como **arquiteto de ecossistema GitHub**, não só "fazer um site".

---

## Prompt

```text
Quero que atues como arquiteto especialista em GitHub Pages, Jekyll e organização estratégica de repositórios GitHub.

Objetivo:
Planejar um ecossistema completo usando apenas recursos gratuitos e nativos do GitHub, maximizando visibilidade, documentação, automação e reaproveitamento.

Contexto:
Sou desenvolvedor (GitHub: felipedsvit) e quero transformar meu GitHub em um hub profissional, técnico e bem estruturado.

Meus repositórios principais atuais, obtidos via `gh repo view`:

### Publicos

1. **erreia** (`felipedsvit/erreia`, Go, public)
   - Kanban leve em Go (Templ + HTMX + SSE via Postgres LISTEN/NOTIFY)
   - Auth por sessão (Argon2id), avatar no MinIO, CSS manual
   - Binário único de ~15 MB
   - Estrutura: `cmd/server`, `internal/{auth,avatar,board,config,database,realtime,session,storage,user,web}`
   - CI: GitHub Actions (build-and-push)
   - Branch: `main`

2. **htmx-4-specialist** (`felipedsvit/htmx-4-specialist`, public)
   - Skill/agent para LLMs especializado em HTMX 4.0, HDA, server-side rendering
   - Estrutura: `SKILL.md`, `agents/`, `evals/`, `references/`, `docs/`
   - Branch: `main`

### Privados

3. **mez-go** (`felipedsvit/mez-go`, Go, private)
   - Plataforma de mensageria multicanal (WhatsApp, Telegram, Instagram, Messenger)
   - Arquitetura hexagonal/clean, multi-binary monorepo (6 cmd/*)
   - Stack: Go + NATS JetStream + Postgres (RLS multi-tenant) + Redis + MinIO
   - Adaptadores: whatsmeow, tgbot, waba, instagram, messenger, tdlib (placeholder)
   - Deploy CI/CD: GitHub Actions (ci.yml, release.yml, sdk.yml)
   - Branch: `develop` (principal)

4. **comm** (`felipedsvit/comm`, Go, private)
   - Monolito modular Go para loja de airsoft (PCE)
   - Hexagonal + Anti-Corruption Layer
   - Módulos: catalog, cart, order, kyc, checkout, payment (Stripe/MP), shipping, fiscal, saleschannel (ML), secrets, audit, notification, storage, eventbus (LISTEN/NOTIFY + SSE)
   - Branch: `main`

Meu perfil: desenvolvedor Go especializado em arquiteturas hexagonais, sistemas de mensageria, HTMX, sistemas multi-tenant e e-commerce. Código em português nos comentários.

Quero usar:

- GitHub Pages
- Jekyll (nativo do GitHub)
- GitHub Actions (free tier)
- GitHub Wiki
- Issues
- Projects
- Discussions
- Releases
- Templates
- GitHub Profile README
- Badges
- Custom collections do Jekyll
- GitHub API (quando útil)
- SEO básico
- Open Graph
- RSS
- Sitemap
- Tags/Categorias
- Search local sem backend
- Dark mode
- Versionamento sem custos

Preciso que planejes:

1. Estrutura do meu GitHub Pages:
- homepage
- about
- blog técnico
- portfolio
- changelog
- roadmap público
- laboratório/experimentos
- documentação
- snippets reutilizáveis
- bookmarks/resources

2. Arquitetura dos repositórios:
Quero separar inteligentemente usando naming convention baseada nos meus repositórios reais:
- `erreia` — projeto principal (Go + HTMX + Postgres)
- `mez-go` — projeto principal privado (mensageria multicanal)
- `comm` — projeto principal privado (e-commerce)
- `htmx-4-specialist` — template/skill reutilizável
- futuros: templates, libs/utilitários, estudos, experiments, snippets, docs, playgrounds

Define naming conventions que funcionem com os nomes já existentes.

3. Estrutura Jekyll:
Define:
- collections ideais
- layouts
- includes
- data files
- taxonomias
- organização escalável

4. Estratégia de conteúdo:
O que publicar para maximizar:
- autoridade técnica
- discoverability
- SEO
- networking
- recrutadores
- contribuição open source
Foco nos temas: Go, arquitetura hexagonal, mensageria multicanal, HTMX, sistemas multi-tenant, e-commerce.

5. Automação com GitHub Actions:
Quero ideias para:
- deploy do Pages
- lint dos repos Go (golangci-lint)
- gerar changelog automaticamente
- gerar docs automáticas (godoc, etc.)
- gerar índice de snippets
- validar links
- backup/export dos repos privados

6. Aproveitamento máximo do ecossistema GitHub:
Como usar:
- Discussions como comunidade
- Issues como backlog público
- Projects como roadmap
- Releases como histórico
- Wiki como documentação viva

7. Estratégia de interligação:
Como conectar tudo:
Pages → repos (erreia, mez-go, comm, htmx-4-specialist) → docs → blog → profile → projects

8. Escalabilidade futura:
Como crescer sem quebrar a estrutura, adicionando novos repos Go.

Responde em formato:

- mapa da arquitetura
- árvore de diretórios
- fluxo de navegação
- convenções
- automações
- melhores práticas
- erros a evitar

Prioridades:
máximo reaproveitamento
mínimo custo
máxima profissionalização
máxima manutenção simples
zero dependência paga
zero backend obrigatório
```
