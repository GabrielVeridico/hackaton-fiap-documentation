# Design — Notificações via Azure Function (Service Bus)

- **Data:** 2026-06-16
- **Projeto:** Conexão Solidária (hackaton-fiap)
- **Escopo desta entrega:** **somente documentação** (atualizar as referências). Sem código nesta etapa.
- **Status:** aplicado às referências (2026-06-16)

## 1. Contexto

Hoje a notificação ao doador está **explicitamente fora de escopo** no MVP:

- `PRD-06` RN06.13 e `Requisitos Funcionais` RN06.13: *"o doador acompanha o resultado por consulta (`GET /doacoes/{id}`); sem notificação push/email no MVP"*.
- `PRD-06` seção 13 (Fora de Escopo): *"Notificação por email/push do resultado (só consulta)"*.

Há ainda uma **contradição interna**: `Requisitos Funcionais` RN06.9 afirma *"o doador é notificado do resultado"*. Esta mudança resolve a contradição **a favor de notificar**.

O fluxo de pagamento é uma saga coreografada: a **PaymentAPI** publica `PagamentoAprovadoEvent` / `PagamentoRecusadoEvent` no Azure Service Bus, hoje consumidos **apenas** pelo consumer da DonationAPI.

## 2. Decisões (fechadas com o usuário)

| # | Decisão | Escolha |
|---|---------|---------|
| 1 | Escopo desta tarefa | Somente documentação / referências |
| 2 | Modelo de deploy | **Azure Functions gerenciado** (plano Consumption), **fora do AKS** |
| 3 | Canal de notificação | **Mock/log** (coerente com o gateway de pagamento mock) |
| 4 | Como obter o destinatário | **Enriquecer os eventos** com dados do doador (sem chamada síncrona) |
| 5 | Documento da capacidade | **PRD-07 separado** (não folding no PRD-06) |

## 3. Componente novo — Notification Function

- **Nome lógico nos docs:** **Hackaton-Fiap-NotificationFunction** (descrito como componente arquitetural, igual a `UserAPI`/`DonationAPI`/`PaymentAPI`).
- **Responsabilidade única:** reagir ao resultado do pagamento e **notificar o doador** (aprovado ou recusado). Canal **mock/log** (ILogger + Application Insights). Não há provedor externo.
- **Subdomínio:** apoio/genérico. **Nunca** escreve `ValorArrecadado` nem toca em banco de outro serviço.
- **Restrição (decisão do usuário):** **não** linkar/referenciar nos docs o **projeto de código dentro da pasta** (`hackaton-fiap-users` nem o futuro `hackaton-fiap-notifications`) **ainda**. A documentação permanece conceitual; nada de paths de pasta ou links para o código. A convenção de nome (`FGC.Notifications.Function`, .NET 8 isolated worker) fica registrada **só aqui no spec** como intenção futura, sem entrar nas referências.

## 4. Topologia de mensageria (mudança necessária)

Para a Function ser um **segundo consumidor independente** dos eventos de resultado de pagamento, eles passam a ser distribuídos via **tópico (pub/sub)** com **subscription dedicada** para a Function — separada da subscription do consumer da DonationAPI. Cada subscription mantém seu próprio *max delivery count* + **DLQ**.

- Trigger nativo `ServiceBusTrigger` na Function.
- Entrega *at-least-once* (mantém RNF11). A notificação mock é idempotente por natureza (é log); reentrega pode gerar log duplicado — aceitável no MVP e registrado como ponto de atenção.

## 5. Enriquecimento dos eventos

Para a Function saber **para quem** notificar sem consultar a UserAPI. A DonationAPI obtém `doadorEmail`/`doadorNome` das **claims do JWT** (RN01.3 garante a claim `email`) no `POST /doacoes`.

| Evento | Campos hoje | Campos adicionados |
|--------|-------------|--------------------|
| `DoacaoSolicitadaEvent` | `doacaoId, idCampanha, valorDoacao, formaPagamento, doadorId` | **`doadorEmail`, `doadorNome`** |
| `PagamentoAprovadoEvent` | `doacaoId, idCampanha, valorDoacao, pagamentoId` | **`doadorId`, `doadorEmail`, `doadorNome`** |
| `PagamentoRecusadoEvent` | `doacaoId, idCampanha, motivo` | **`doadorId`, `doadorEmail`, `doadorNome`, `valorDoacao`** |

A PaymentAPI **repassa** os campos de doador recebidos no `DoacaoSolicitadaEvent` para os eventos de resultado.

## 6. Consequências (deploy / observabilidade / CI-CD)

- **Deploy:** Azure Functions gerenciado (Consumption), **fora do AKS** — não entra no pipeline de imagem → AKS.
- **Observabilidade:** por rodar fora do cluster, logs/métricas vão para **Application Insights / Azure Monitor**, **não** para o Prometheus/Grafana in-cluster. Trade-off registrado: a Function fica fora do dashboard Grafana padrão.
- **CI/CD:** workflow GitHub Actions próprio (`azure/functions-action`), separado do deploy AKS.

## 7. Referências a atualizar (plano de edição)

1. **NOVO `docs/03 - PRDs/PRD-07 - Notificações.md`** — a partir do `_Template - PRD`: capacidade de notificação, eventos consumidos, regras (canal mock, idempotência de log), Gherkin, fora de escopo (envio real de email).
2. **`docs/03 - PRDs/00 - Índice de PRDs.md`** — nova linha `RF07 | PRD-07 - Notificações | Notificações | rascunho`.
3. **`docs/01 - Requisitos/Requisitos Funcionais.md`** — nova **RF07 — Notificação ao Doador**; ajustar **RN06.13** (de "sem notificação" → "consulta **e** notificação assíncrona via Function").
4. **`docs/03 - PRDs/PRD-06 - Processo de Doação.md`** — ajustar RN06.13; seção 9 (2º consumidor dos eventos de resultado); seção 11 (dependência da Notification Function); seção 13 (sai "notificação fora de escopo"; entra "envio **real** de email fora de escopo — canal é mock"); payloads dos eventos enriquecidos.
5. **`docs/04 - Arquitetura/Visão Geral de Arquitetura.md`** — novo componente + aresta no diagrama Mermaid (`SB → NotificationFunction`), com nota "Azure Functions gerenciado, fora do AKS".
6. **`docs/02 - DDD/Domain Events.md`** — payloads enriquecidos; nota de topologia pub/sub + 2º consumidor.
7. **`docs/02 - DDD/Context Map.md`** — novo contexto **Notificações** consumindo "Pagamento Aprovado/Recusado".
8. **`docs/01 - Requisitos/Requisitos Técnicos.md`** — §1 (stack: Notificações = Azure Functions), §2 (serviço/Function), §3 (consumidor de notificação), §5 (observabilidade via App Insights), §8 (CI/CD próprio).
9. **`docs/Pontos-Chave do Projeto.md`** — seção A (capacidade de notificação) e seção B (pilar técnico: Azure Function serverless).
10. **`docs/04 - Arquitetura/Decisões de Arquitetura (ADRs).md`** — preencher **ADR-001 — "Notificações via Azure Function gerenciada (Service Bus pub/sub, canal mock)"** (contexto → decisão → consequências).

> **Não** serão tocados: `Bounded Contexts.md` e `Subdomínios.md` — ainda são placeholders "a-preencher"; preenchê-los pela metade quebraria a consistência. O contexto Notificações entra no `Context Map.md` (que está completo).

## 8. Fora de escopo (desta entrega)

- Código do projeto `FGC.Notifications.Function` (scaffold). *(O usuário pode pedir numa etapa seguinte.)*
- **Links/referências ao projeto de código dentro da pasta** (`hackaton-fiap-users` / futuro `hackaton-fiap-notifications`) — docs ficam conceituais por enquanto.
- Envio real de email/SMS (provedor externo). O canal é mock/log.
- Provisionamento real do tópico/subscriptions no Azure e dos workflows de CI/CD.

## 9. Riscos / pontos de atenção

- **Mudança de topologia** (queue → topic/subscription) precisa estar refletida de forma consistente em todos os docs de mensageria para não gerar nova contradição.
- **Reentrega** gera log duplicado na notificação mock (aceitável no MVP).
- **Observabilidade fragmentada**: Function fora do Grafana in-cluster (vai para App Insights) — deixar explícito na demo.
