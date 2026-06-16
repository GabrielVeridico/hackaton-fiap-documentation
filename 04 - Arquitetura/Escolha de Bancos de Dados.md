---
tags: [hackaton-fiap, arquitetura, banco-dados]
projeto: Conexão Solidária
---

# Escolha de Bancos de Dados

> Entregável obrigatório: **PDF justificando os 2 bancos (X e Y)**. Esta nota é o rascunho da justificativa. Modelo **CQRS**: banco de escrita transacional + banco de leitura escalável.

## Os dois bancos

| Banco | Uso (contexto) | Por quê | Trade-offs |
|-------|----------------|---------|------------|
| **X — SQL Server** | Escrita transacional: Usuários, Campanhas, Doações. **Um banco por serviço** (database-per-service). | Consistência forte, transações ACID e integridade referencial — essenciais para registrar doações e aplicar regras de campanha. Forte aderência ao .NET/EF Core. | Leitura pública sob alta concorrência escala mal num relacional → por isso a leitura sai para o Y. |
| **Y — Cosmos DB (API NoSQL)** | Leitura pública: *read model* do **Painel de Transparência** (RF05), atualizado por projeção. | Escala horizontal nativa, baixa latência e alta disponibilidade; documento denormalizado pronto para consulta; *free tier* cobre o MVP. Consistência eventual é aceitável e já assumida (RN05.4 / [[Requisitos Não Funcionais\|RNF15]]). | Custo por RU/s exige dimensionamento; consistência eventual (intencional); dados denormalizados/duplicados. |

## Fluxo de dados (CQRS)

```
POST /doacoes ─▶ DonationAPI ─▶ SQL Server (X)   [Doacao = Pendente]
                                   │ (saga de eventos — ver [[Domain Events]])
                                   ▼
                          PagamentoAprovadoEvent
                                   │
        consumer/BackgroundService ┤ (1) consolida ValorArrecadado no SQL Server (X)
                                   └ (2) projeta a campanha no Cosmos DB (Y)
                                   ▼
GET /transparencia/campanhas ◀─ Painel lê SOMENTE do Cosmos DB (Y)
```

## Modelo do read store (Cosmos DB)

- **Documento por campanha:** `{ id: campanhaId, titulo, meta, valorArrecadado, status }`.
- **Partition key:** `/id` (campanhaId) — dataset pequeno, *point-reads* rápidos; a listagem filtra por `status = Ativa`.
- **Consistência:** `Session` (padrão) — suficiente, pois o painel é eventualmente consistente por design.
- **Custo:** *free tier* do Cosmos (1000 RU/s + 25 GB) cobre a demo do hackathon.

**Relacionados:** [[Requisitos Técnicos]] · [[Domain Events]] · [[Requisitos Funcionais]] (RF05, RF06) · [[Visão Geral de Arquitetura]]
