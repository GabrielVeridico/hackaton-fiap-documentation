---
tags: [hackaton-fiap, spec, design, paymentapi]
projeto: Conexão Solidária
servico: HackatonFiap.Payments
data: 2026-06-17
status: aprovação-pendente
---

# PaymentAPI (HackatonFiap.Payments) — Design

## 1. Contexto e objetivo

A **PaymentAPI** é o microsserviço de pagamento do Conexão Solidária. No fluxo de doação
(saga coreografada — ver [[Domain Events]] e [[PRD-06 - Processo de Doação]]), ela:

1. **consome** o evento de uma doação solicitada,
2. **processa** o pagamento num **gateway simulado (mock)**,
3. **publica** o resultado (aprovado/recusado) de volta no Service Bus.

Parte do boilerplate **FGC.Payments** (`MatheusRoberto-Git/FGC.Payments`, mesmo autor do
FGC.Users que originou a UserAPI) e é **alinhada ao padrão já estabelecido no
`hackaton-fiap-users`**: Clean Architecture, CQRS manual com `Result<T>`, EF Core,
OpenTelemetry/Serilog, `/health`+`/ready`, xUnit + NSubstitute.

> O boilerplate FGC.Payments é orientado a **loja de games** (REST, `UserId`/`GameId`,
> gateway aleatório 90%, publica em fila) e usa **UseCases** em vez de CQRS. Este design
> adapta o domínio para **doação**, troca REST-processing por **consumer event-driven**,
> torna o mock **determinístico** e converte UseCases → **CQRS**.

## 2. Decisões fechadas (brainstorming 2026-06-17)

- **Base:** clonar FGC.Payments e alinhar ao padrão do `users`.
- **Operação:** **event-driven + REST de consulta** (consumer/publisher + `GET /api/payments/{donationId}`).
- **Service Bus:** modelo padrão **Tópicos + Subscriptions** (pub/sub nativo, SDK `Azure.Messaging.ServiceBus`).
- **Contrato de eventos:** **inglês** (`DonationRequestedEvent`, `PaymentApprovedEvent`, `PaymentDeclinedEvent`).
- **Auth do GET:** JWT Bearer restrito a **GestorONG/Owner**.
- **Processo:** spec → revisão → plano (writing-plans) → implementação.

## 3. Estrutura de projetos (espelha o `users`)

```
hackaton-fiap-payments/
├── src/
│   ├── HackatonFiap.Payments.Domain
│   ├── HackatonFiap.Payments.Application
│   ├── HackatonFiap.Payments.Infrastructure
│   └── HackatonFiap.Payments.API          # FGC chamava "Presentation" → renomeado p/ .API
├── tests/
│   └── HackatonFiap.Payments.UnitTests
└── HackatonFiap.Payments.sln               # .sln (como o users), não .slnx
```

**Reaproveitado do FGC.Payments:** `PaymentsDbContext`/EF configurations, base de
`ServiceBus`, `Dockerfile`, `.github` (CI/CD). **Trazido do `users`:** `Result<T>`/`Error`,
CQRS manual, setup OpenTelemetry/Serilog, `/health`+`/ready`, NSubstitute.

## 4. Domínio

Entidade-agregado **`Payment`** (código em inglês — convenção dos microsserviços):

| Campo | Tipo | Notas |
|-------|------|-------|
| `Id` | `Guid` | = pagamentoId, repassado no evento de aprovação |
| `DonationId` | `Guid` | **único** (idempotência) |
| `CampaignId` | `Guid` | repassado nos eventos de resultado |
| `Amount` | `decimal` | > 0 |
| `PaymentMethod` | enum `Pix`/`CreditCard`/`BankTransfer` | |
| `Status` | enum `Pending` → `Approved` \| `Declined` | |
| `TransactionId` | `string` | gerado (`TXN-...`) |
| `DonorId` / `DonorEmail` / `DonorName` | | repassados para enriquecer os eventos de resultado (a Function notifica sem chamar a UserAPI) |
| `CreatedAt` / `ProcessedAt` | `DateTime` | |
| `DeclineReason` | `string?` | preenchido quando recusado |

**Removido do boilerplate:** status `Processing`/`Refunded`/`Cancelled` e métodos
`Refund()`/`Cancel()` — estorno/refund estão **fora de escopo** (PRD-06 §13). `GameId` →
`CampaignId`/`DonationId`.

Métodos de negócio: `Create(...)` (status `Pending`), `Approve()` (→ `Approved`),
`Decline(reason)` (→ `Declined`). Transições validadas (lança/`Result` em transição inválida).

## 5. Contratos de eventos (Service Bus, inglês)

Mensagens de integração (distintas dos domain events internos):

| Evento | Direção | Payload |
|--------|---------|---------|
| `DonationRequestedEvent` | **consome** | `DonationId`, `CampaignId`, `Amount`, `PaymentMethod`, `DonorId`, `DonorEmail`, `DonorName` |
| `PaymentApprovedEvent` | **publica** | `DonationId`, `CampaignId`, `Amount`, `PaymentId`, `DonorId`, `DonorEmail`, `DonorName` |
| `PaymentDeclinedEvent` | **publica** | `DonationId`, `CampaignId`, `Reason`, `Amount`, `DonorId`, `DonorEmail`, `DonorName` |

**Topologia (pub/sub):**
- Entrada: tópico `donation-requested`, subscription `payments` (consumida pela PaymentAPI).
- Saída: tópico `payment-result`, subscriptions `donations` (consumer da DonationAPI) e
  `notifications` (NotificationFunction). `Subject` = `PaymentApproved`/`PaymentDeclined`
  para filtragem/roteamento.

> **Follow-up de doc:** [[Domain Events]] e [[PRD-06 - Processo de Doação]] descrevem os
> eventos em PT-BR (`DoacaoSolicitadaEvent` etc.). Como o contrato passa a ser **inglês**,
> essas docs serão atualizadas para casar (e a futura DonationAPI seguirá os nomes em inglês).

## 6. Fluxo

```
tópico donation-requested / subscription "payments"
  → DonationRequestedConsumer (BackgroundService, ServiceBusProcessor)
      desserializa DonationRequestedEvent
      → cria escopo → ProcessDonationPaymentCommandHandler
            idempotência: se já existe Payment p/ DonationId → republica resultado e completa
            senão: cria Payment (Pending) → IPaymentGateway.Process(amount)
                   aprovado → Payment.Approve()  → PaymentApprovedEvent
                   recusado → Payment.Decline(r) → PaymentDeclinedEvent
            persiste (EF) e publica no tópico payment-result
      → completa a mensagem (sucesso) | abandona (falha → reentrega → DLQ após max delivery)

GET /api/payments/{donationId} → GetPaymentByDonationIdQueryHandler  (JWT: GestorONG/Owner)
```

## 7. CQRS (handlers manuais + `Result<T>`)

- `ProcessDonationPaymentCommand` / `...Handler` — acionado pelo consumer.
- `GetPaymentByDonationIdQuery` / `...Handler` — consulta REST.
- Sem MediatR; handlers injetados diretamente (= users).

## 8. Gateway mock determinístico (RN06.11)

`IPaymentGateway` → `MockPaymentGateway`: **aprova por padrão**; **recusa quando os centavos
do `Amount` forem `,99`** (marcador de teste determinístico). Comportamento/marcador
configurável via `appsettings` (ex.: `PaymentGateway:DeclineCentavos = 99`).

## 9. Idempotência (at-least-once)

Índice **único em `DonationId`**. Ao reprocessar o mesmo `DonationRequestedEvent`
(reentrega), o handler encontra o `Payment` existente, **não cria duplicado** e **republica
o resultado correspondente** (aprovado/recusado), garantindo convergência sem efeito duplo.

## 10. Persistência

EF Core 8 + SQL Server, **database-per-service** (`HackatonFiapPaymentsDb`).
`PaymentsDbContext` + configuration de `Payment` (índice único em `DonationId`).
Migration `InitialCreate`; migrations aplicadas no startup (fail-fast de connection string
em ambientes não-Development, como no users).

## 11. Observabilidade (= users)

- **OpenTelemetry** traces + metrics → `/metrics` (Prometheus); **Serilog** estruturado com
  correlation/trace id; consumo do Service Bus instrumentado.
- **`/health`** (liveness, sem dependência) e **`/ready`** (readiness com
  `AddDbContextCheck<PaymentsDbContext>`) — mesmo padrão da melhoria recente do users.
- Métricas custom: `payments_processed_total`, `payments_approved_total`,
  `payments_declined_total`.

## 12. Autenticação

JWT Bearer (mesmo emissor/audience do ecossistema). O **consumer** não usa JWT. O endpoint
`GET /api/payments/{donationId}` exige autenticação **restrita a GestorONG/Owner** (pagamento
é dado interno/auditoria). `/health`, `/ready`, `/metrics` anônimos.

## 13. Testes (xUnit + NSubstitute + FluentAssertions)

- **Domain:** `Payment` (criação, `Approve`/`Decline`, transições inválidas, validações).
- **Gateway:** `MockPaymentGateway` (aprova; recusa `,99`).
- **Application:** `ProcessDonationPaymentCommandHandler` (aprovado, recusado por `,99`,
  **idempotência** — reprocessar não duplica), `GetPaymentByDonationIdQueryHandler`.
- Dependências (repo, gateway, publisher) mockadas via NSubstitute. Sem testes de integração
  (mesmo padrão do users).

## 14. CI/CD + Kubernetes

- CI (`.github/workflows`): build .NET + testes + imagem Docker a cada push na principal.
- `deployment.yaml`: probes `/health`/`/ready`, annotations Prometheus (`scrape`, `path=/metrics`,
  `port=8080`), porta 8080, secrets via Key Vault/Secrets (= users).

## 15. Fora de escopo

- Gateway de pagamento real / credenciais; estorno (refund), cancelamento, parcelamento, multimoeda.
- A consolidação do `ValorArrecadado` e a notificação são de **outros serviços** (DonationAPI
  consumer / NotificationFunction).

## 16. Follow-ups

- [ ] Atualizar [[Domain Events]] e [[PRD-06 - Processo de Doação]] para os nomes de evento em **inglês**.
- [ ] Atualizar [[Visão Geral de Arquitetura]] e [[Requisitos Técnicos]] quando a PaymentAPI estiver implementada.
- [ ] Doc de API da PaymentAPI em `docs/06 - APIs/` (nos moldes da UserAPI).
