---
title: Comm
description: Monolito modular para e-commerce com checkout, pagamentos e emissão fiscal.
stack: [Go, PostgreSQL, Stripe, Mercado Pago, MinIO]
status: active
visibility: private
featured: true
---

Plataforma de e-commerce modular para loja de airsoft (PCE), construída como monolito hexagonal com módulos bem delimitados e Anti-Corruption Layer.

## Módulos

| Módulo       | Responsabilidade                              |
|--------------|-----------------------------------------------|
| catalog      | Produtos, variantes, estoque                  |
| cart         | Carrinho, promoções                           |
| order        | Pedidos, status, histórico                    |
| kyc          | Verificação de identidade (CRAF obrigatório)  |
| checkout     | Fluxo de compra                               |
| payment      | Stripe + Mercado Pago                         |
| shipping     | Cálculo de frete, rastreio                    |
| fiscal       | NF-e, DANFE                                  |
| saleschannel | Mercado Livre                                 |
| notification | Email, SMS, push                              |
| eventbus     | LISTEN/NOTIFY + SSE                           |
| audit        | Log imutável de ações                         |

## Destaques

- **Anti-Corruption Layer**: adaptadores para Mercado Livre, Stripe e Correios sem vazar libs externas ao domínio
- **Event bus interno**: Postgres `LISTEN/NOTIFY` como broker — zero infraestrutura adicional
- **KYC obrigatório**: fluxo de verificação de identidade para compra de réplicas de airsoft
- **Fiscal integrado**: emissão de NF-e diretamente no checkout

> Repositório privado. Detalhes disponíveis sob consulta.
