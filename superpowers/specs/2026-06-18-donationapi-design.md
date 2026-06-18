---
tags: [hackaton-fiap, spec, design, donationapi]
projeto: Conexão Solidária
servico: HackatonFiap.Donations
data: 2026-06-18
status: aprovado
---

# DonationAPI (HackatonFiap.Donations) — Design

## 1. Contexto e objetivo

A **DonationAPI** é o microsserviço central do Conexão Solidária. Concentra três frentes
(três bounded contexts no mesmo serviço — ver [[Bounded Contexts]]):

1. **Campanhas** ([[PRD-04 - Gestão de Campanhas]]) — CRUD e ciclo de vida das campanhas de arrecadação.
2. **Doações / Arrecadação** ([[PRD-06 - Processo de Doação]]) — endpoint de doação, saga coreografada
   com a PaymentAPI e o **consumer** idempotente que consolida o `AmountRaised`.
3. **Transparência** ([[PRD-05 - Painel de Transparência]]) — leitura pública servida **somente** do
   read model em Cosmos DB.

Segue o padrão já estabelecido em `hackaton-fiap-users` e `hackaton-fiap-payments`: Clean
Architecture, CQRS manual com `Result<T>`, EF Core 8, Azure Service Bus
(`Azure.Messaging.ServiceBus`), OpenTelemetry/Serilog, `/health`+`/ready`+`/metrics`,
xUnit + NSubstitute + FluentAssertions. Ver [[convencoes-microsservicos]].

> O repositório `hackaton-fiap-donations` trazia o boilerplate **`CatalogAPI`** (loja de games,
> single-project, **RabbitMQ/MassTransit**, Redis, Azure Search, MongoDB). Esse código é
> **descartado**; reaproveita-se apenas o que serve ao serviço (CI `.github`, `Dockerfile`,
> manifests k8s, `.gitignore`). A arquitetura passa a espelhar a PaymentAPI.

## 2. Decisões fechadas (brainstorming 2026-06-18)

- **Base/estrutura:** recriar em **Clean Architecture multi-projeto** no padrão PaymentAPI/UserAPI;
  descartar o código do `CatalogAPI` (reaproveitar só CI/Docker/k8s do repo).
- **Mensageria:** **Azure Service Bus** (não RabbitMQ), compatível com o contrato já implementado
  na PaymentAPI.
- **Read model (Cosmos):** abstração `ICampaignReadStore` com implementação **Cosmos** (quando
  configurado) e **fallback in-memory** em Development — espelha o padrão ServiceBus→NoOp da
  PaymentAPI. API e testes rodam sem Cosmos; produção usa Cosmos real.
- **Idioma:** **código e rotas em inglês** (`Campaign`/`Donation`, `/api/campaigns`,
  `/api/donations`, `/api/transparency/campaigns`); roles permanecem pt-BR (`Doador`/`GestorONG`).
  Os PRDs (em PT) seguem como fonte de regras; só o código/rotas ficam em inglês.
- **Escopo:** serviço completo (PRD-04 + PRD-05 + PRD-06) num único ciclo spec → plano → implementação.
- **Processo:** spec → revisão → plano (writing-plans) → implementação.

## 3. Estrutura de projetos (espelha payments/users)

```
hackaton-fiap-donations/
├── src/
│   ├── HackatonFiap.Donations.Domain          # entidades, VOs, enums, Result<T>/Error
│   ├── HackatonFiap.Donations.Application      # commands/queries + handlers, abstractions,
│   │                                           # integration events, errors, métricas
│   ├── HackatonFiap.Donations.Infrastructure   # EF Core, repositories, messaging (Service Bus),
│   │                                           # read store (Cosmos + InMemory), background services
│   └── HackatonFiap.Donations.API              # controllers, Program.cs, appsettings
├── tests/
│   └── HackatonFiap.Donations.UnitTests        # xUnit + NSubstitute + FluentAssertions
├── HackatonFiap.Donations.sln
├── global.json                                 # SDK 8.0, rollForward latestFeature
└── Dockerfile
```

**Reaproveitado do repo:** `.github` (CI/CD), `Dockerfile`, manifests k8s, `.gitignore`.
**Trazido de payments/users:** `Result<T>`/`Error`, CQRS manual, setup OpenTelemetry/Serilog,
`/health`+`/ready`+`/metrics`, padrão de Service Bus (publisher + consumer BackgroundService),
NSubstitute.

## 4. Domínio

### 4.1 `Campaign` (agregado — contexto Campanhas, [[PRD-04 - Gestão de Campanhas]])

| Campo | Tipo | Notas |
|-------|------|-------|
| `Id` | `Guid` | |
| `Title` | `string` | obrigatório |
| `Description` | `string` | |
| `Period` | VO `Period(StartDate, EndDate)` | `EndDate ≥ StartDate`; na criação `EndDate` não no passado |
| `Goal` | `decimal` | > 0 (RN04.3) |
| `AmountRaised` | `decimal` | **somente-leitura via API**; escrito só pelo consumer/job (RN04.7) |
| `Status` | enum `Active` → `Completed` \| `Cancelled` | terminal não volta a `Active` (RN04.8) |
| `CompletionReason` | enum? `GoalReached`/`ManuallyClosed`/`Expired` | preenchido **sse** `Completed` (RN04.9) |
| `CreatedById` | `Guid` | GestorONG criador (auditoria) |
| `CreatedAt` / `UpdatedAt` | `DateTime` | |

Métodos: `Create(...)` (status inicial `Active` — RN04.5), `Update(...)` (não toca `AmountRaised`),
`Complete(reason)`, `Cancel()`, `AddRaised(amount)`. Transições inválidas retornam `Result`
de falha (estado terminal — RN04.8).

**Invariantes:** `Goal > 0`; `EndDate ≥ StartDate`; `EndDate` não no passado na criação;
`AmountRaised` nunca via API; estados terminais; `CompletionReason` preenchido **se e somente se**
`Completed`.

Mapeamento de status PT (PRD) → EN (código): `Ativa→Active`, `Concluida→Completed`,
`Cancelada→Cancelled`; motivos `MetaAtingida→GoalReached`, `EncerradaManualmente→ManuallyClosed`,
`Expirada→Expired`.

### 4.2 `Donation` (agregado — contexto Doações, [[PRD-06 - Processo de Doação]])

| Campo | Tipo | Notas |
|-------|------|-------|
| `Id` | `Guid` | = doacaoId |
| `CampaignId` | `Guid` | |
| `Amount` | `decimal` | > 0 (RN06.4) |
| `PaymentMethod` | enum `Pix`/`CreditCard`/`BankTransfer` | **mesmo enum da PaymentAPI** (RN06.5) |
| `Status` | enum `Pending` → `Approved` \| `Declined` | RN06.12 |
| `DonorId` / `DonorEmail` / `DonorName` | | das claims do JWT (RN01.3); repassados no evento |
| `DeclineReason` | `string?` | preenchido quando recusada |
| `CreatedAt` / `ProcessedAt` | `DateTime` | |

Métodos: `Create(...)` (status `Pending`), `Approve()`, `Decline(reason)`. Transições validadas.

Mapeamento de status PT → EN: `Pendente→Pending`, `Aprovada→Approved`, `Recusada→Declined`.
Mapeamento de forma de pagamento PT (PRD) → EN (enum): `PIX→Pix`, `Cartão→CreditCard`,
`Transferência→BankTransfer`.

### 4.3 `ProcessedEvent` (inbox — idempotência do consumer, RN06.10)

| Campo | Tipo | Notas |
|-------|------|-------|
| `DonationId` | `Guid` | **PK / índice único** (dedup) |
| `ProcessedAt` | `DateTime` | |

## 5. Endpoints

| Método | Rota | Auth | Descrição | PRD |
|--------|------|------|-----------|-----|
| POST | `/api/campaigns` | `GestorONG` | cria campanha (status inicial `Active`) | PRD-04 |
| PUT | `/api/campaigns/{id}` | `GestorONG` | edita dados (não o `AmountRaised`) | PRD-04 |
| PATCH | `/api/campaigns/{id}/status` | `GestorONG` | encerra manualmente (`ManuallyClosed`) ou cancela | PRD-04 |
| GET | `/api/campaigns` | `GestorONG` | lista todas (gestão) | PRD-04 |
| GET | `/api/campaigns/{id}` | `GestorONG` | detalha uma campanha | PRD-04 |
| POST | `/api/donations` | `Doador` | grava `Pending`, publica evento → **202** | PRD-06 |
| GET | `/api/donations/{id}` | `Doador` | status da própria doação | PRD-06 |
| GET | `/api/transparency/campaigns` | **público** | lê só do Cosmos; apenas `Active` + percentual | PRD-05 |

- JWT Bearer com mesmo `Issuer`/`Audience` do ecossistema (`conexaosolidaria.local` /
  `conexaosolidaria.clients`). Policies: `ManagersOnly` (`GestorONG`/`Owner`) para campanhas;
  `DonorOnly` (`Doador`) para doações.
- `GET /api/donations/{id}` retorna só a doação **do próprio doador** (compara `DonorId` das claims).
- `/api/transparency/campaigns`, `/health`, `/ready`, `/metrics` são **anônimos** (RN05.3).

## 6. Saga e mensageria (compatível com a PaymentAPI já implementada)

### 6.1 Topologia (pub/sub, igual à já criada pela PaymentAPI)

- **Saída:** tópico `donation-requested` — a DonationAPI **publica** aqui; a PaymentAPI consome na
  subscription `payments`.
- **Entrada:** tópico `payment-result` — a DonationAPI **consome** na subscription **`donations`**;
  a NotificationFunction consome em `notifications`. `Subject` = `PaymentApproved`/`PaymentDeclined`.

### 6.2 Contratos de eventos (records em inglês, JSON com `JsonStringEnumConverter`)

Idênticos ao que a PaymentAPI já consome/publica — **não alterar nomes/campos**:

| Evento | Direção | Payload | Subject |
|--------|---------|---------|---------|
| `DonationRequestedEvent` | **publica** | `DonationId`, `CampaignId`, `Amount`, `PaymentMethod`, `DonorId`, `DonorEmail`, `DonorName` | `DonationRequested` |
| `PaymentApprovedEvent` | **consome** | `DonationId`, `CampaignId`, `Amount`, `PaymentId`, `DonorId`, `DonorEmail`, `DonorName` | `PaymentApproved` |
| `PaymentDeclinedEvent` | **consome** | `DonationId`, `CampaignId`, `Reason`, `Amount`, `DonorId`, `DonorEmail`, `DonorName` | `PaymentDeclined` |

### 6.3 Fluxo

```
POST /api/donations  (JWT: Doador)
  → valida: campanha existe (404 senão) · Status=Active E StartDate ≤ agora ≤ EndDate (422 senão) · Amount > 0
  → grava Donation = Pending  (SQL)
  → publica DonationRequestedEvent no tópico "donation-requested" (subject "DonationRequested"),
            DonorId/Email/Name vindos das claims do JWT
  → 202 Accepted { donationId, status: "Pending" }

tópico "payment-result" / subscription "donations"
  → PaymentResultConsumer (BackgroundService, ServiceBusProcessor, AutoComplete=false)
      roteia por Subject:
        "PaymentApproved" → ProcessPaymentApprovedCommandHandler
            idempotência: se ProcessedEvent(DonationId) existe → completa msg (no-op)
            senão, em UMA transação SQL:
               Donation.Approve()
               Campaign.AddRaised(Amount); se AmountRaised ≥ Goal → Campaign.Complete(GoalReached)
               grava ProcessedEvent(DonationId)
            após o commit → projeta a campanha no Cosmos (Upsert; re-projetável)
        "PaymentDeclined" → ProcessPaymentDeclinedCommandHandler
            idempotência idem; Donation.Decline(Reason) (não contabiliza)
      → completa a mensagem (sucesso) | abandona (falha → reentrega → DLQ após max delivery)

CampaignExpirationWorker (BackgroundService periódico)
  → campanhas Active com EndDate vencida → Complete(Expired) + projeta no Cosmos (RN04.11)

GET /api/donations/{id}  (JWT: Doador) → GetDonationByIdQueryHandler (filtra pelo DonorId das claims)
GET /api/transparency/campaigns (público) → ListActiveCampaignsQueryHandler → ICampaignReadStore (Cosmos)
```

### 6.4 Ambiente sem Service Bus (Development)

Como na PaymentAPI: se `ServiceBus:ConnectionString` ausente em Development, registra
`NoOpEventPublisher` (loga e não publica) e **não** registra o `PaymentResultConsumer` — a API sobe
sem broker. Fora de Development, ausência de connection string é fail-fast.

## 7. CQRS (handlers manuais + `Result<T>`)

- Campanhas: `CreateCampaignCommand`, `UpdateCampaignCommand`, `ChangeCampaignStatusCommand`,
  `GetCampaignByIdQuery`, `ListCampaignsQuery`.
- Doações: `CreateDonationCommand` (publica o evento), `GetDonationByIdQuery`.
- Consumer: `ProcessPaymentApprovedCommand`, `ProcessPaymentDeclinedCommand`.
- Transparência: `ListActiveCampaignsQuery` (lê `ICampaignReadStore`).
- Sem MediatR; handlers injetados diretamente nos controllers / no consumer (= users/payments).

## 8. Read model — Cosmos DB (PRD-05)

- Interface `ICampaignReadStore`: `UpsertAsync(CampaignReadModel)`, `ListActiveAsync()`.
- Documento por campanha: `{ id: campaignId, title, goal, amountRaised, status }`;
  `partition key = /id`; consistência `Session` (ver [[Escolha de Bancos de Dados]]).
- Implementações:
  - `CosmosCampaignReadStore` — registrada quando `Cosmos:ConnectionString` configurado.
  - `InMemoryCampaignReadStore` — fallback em Development (singleton, thread-safe).
- **Escritores:** só o consumer (ao consolidar) e o `CampaignExpirationWorker` (ao expirar). O painel
  só lê. O `percentual` (`amountRaised / goal`) é derivado na query/projeção do painel.
- O painel `ListActiveAsync` retorna apenas `status = Active` (RN05.1) e nunca expõe dados de doador
  (RN05.5).

## 9. Tratamento de erros

`Result<T>`/`Error` mapeado para HTTP no controller:

| Situação | HTTP |
|----------|------|
| Validação de input (meta ≤ 0, DataFim no passado, amount ≤ 0, forma inválida) | `400` |
| Campanha/doação não encontrada | `404` |
| Regra de negócio: campanha inexistente para doação, encerrada ou fora do período | `422` |
| Transição inválida em estado terminal (campanha) | `409` |
| Doação solicitada aceita | `202` |

> **Janela do job de expiração (RN04.11/risco PRD-04 §14):** entre a `EndDate` e a próxima execução
> do `CampaignExpirationWorker`, a campanha ainda está `Active`. Por isso `POST /api/donations` valida
> **também o período** (não só o status) e devolve `422` se fora dele.

## 10. Persistência

EF Core 8 + SQL Server, **database-per-service** (`HackatonFiapDonationsDb`).
`DonationsDbContext` + configurations de `Campaign`, `Donation`, `ProcessedEvent` (índice único em
`DonationId`). Migration `InitialCreate`; migrations aplicadas no startup (fail-fast de connection
string fora de Development, como users/payments). `dotnet-ef` como tool local
(`.config/dotnet-tools.json`).

A consolidação (Donation + Campaign + ProcessedEvent) ocorre numa **única transação** no SQL; a
projeção no Cosmos é feita **após o commit** e é re-projetável (idempotente por `id`), de modo que
uma falha de projeção não corrompe o estado transacional.

## 11. Observabilidade (= users/payments)

- **OpenTelemetry** metrics → `/metrics` (Prometheus); **Serilog** estruturado com correlation/trace id;
  consumo do Service Bus instrumentado.
- **`/health`** (liveness, sem dependência) e **`/ready`** (readiness com
  `AddDbContextCheck<DonationsDbContext>`).
- Métricas custom de negócio: `donations_received_total`, `donations_approved_total`,
  `donations_declined_total`, `campaigns_completed_total`, `amount_raised_total`.

## 12. Autenticação

JWT Bearer (mesmo emissor/audience do ecossistema). O **consumer** e o **worker** não usam JWT.
Campanhas exigem `GestorONG`/`Owner`; doações exigem `Doador`; `GET /api/donations/{id}` é restrito
ao próprio doador; transparência, `/health`, `/ready`, `/metrics` são anônimos.

## 13. Testes (xUnit + NSubstitute + FluentAssertions)

- **Domain:** `Campaign` (criação, invariantes Goal/Period, `Complete`/`Cancel`/`AddRaised`,
  conclusão por meta, transições inválidas em estado terminal) e `Donation` (criação, `Approve`/
  `Decline`, transições).
- **Application — campanhas:** create (bloqueia meta ≤ 0 e DataFim no passado), update (não altera
  `AmountRaised`), change status (encerrar/cancelar; bloqueia em terminal).
- **Application — doações:** create (bloqueia campanha inexistente/encerrada/fora do período →
  422; publica `DonationRequestedEvent` no caminho feliz), get by id (filtra por doador).
- **Application — consumer:** approved (consolida `AmountRaised`, conclui por meta, **idempotente** —
  reprocessar não soma de novo), declined (marca `Declined`, não contabiliza).
- **Read store:** `InMemoryCampaignReadStore` (upsert/list active, filtra não-ativas).
- **Worker:** lógica de expiração (campanha `Active` vencida → `Expired`).
- Dependências (repos, publisher, read store, relógio) mockadas via NSubstitute. Sem testes de
  integração (mesmo padrão de users/payments).

## 14. CI/CD + Kubernetes

- CI (`.github/workflows`): build .NET + testes + imagem Docker a cada push na principal.
- `deployment.yaml`: probes `/health`/`/ready`, annotations Prometheus (`scrape`, `path=/metrics`,
  `port=8080`), porta 8080, secrets via Key Vault/CSI (= users/payments). ConfigMap inclui
  `Cosmos:*` e `ServiceBus:*` além de `ConnectionStrings:Default` e `Jwt:*`.

## 15. Fora de escopo

- Reabertura de campanhas terminais; edição manual de `AmountRaised`; categorias/tags/imagens; metas
  múltiplas/marcos (PRD-04 §13).
- Detalhe por campanha no painel; filtros/busca/paginação; histórico de encerradas (PRD-05 §13).
- Gateway de pagamento real (é da PaymentAPI, mock); estorno/refund, parcelamento, multimoeda;
  doação anônima (PRD-06 §13). Envio **real** de notificação (é da NotificationFunction, canal mock).

## 16. Follow-ups

- [ ] Doc de API da DonationAPI em `docs/06 - APIs/` (nos moldes da UserAPI).
- [ ] Atualizar [[Visão Geral de Arquitetura]] / [[Requisitos Técnicos]] quando implementada.
- [ ] (Já pendente desde a PaymentAPI) alinhar [[Domain Events]]/[[PRD-06 - Processo de Doação]] aos
      nomes de evento em inglês — a DonationAPI já os adota.
