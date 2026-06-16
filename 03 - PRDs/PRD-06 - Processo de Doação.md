---
tags: [hackaton-fiap, prd]
prd-id: PRD-06
titulo: "Processo de Doação (saga + consumer)"
bounded-context: "Doações / Arrecadação"
owner: "Equipe Conexão Solidária"
status: rascunho
atualizado: 2026-06-12
---

# PRD-06 — Processo de Doação (saga + consumer)

## 1. Visão Geral
O doador autenticado envia uma intenção de doação; o pagamento é processado de forma assíncrona
por um gateway **simulado (mock)** e, **somente se aprovado**, o `ValorArrecadado` da campanha é
consolidado por um **consumer** idempotente. Cobre o endpoint de doação, a **saga de 2 eventos** e o
**consumer/BackgroundService** (o "Worker" de Arrecadação).

## 2. Atores / Personas
| Ator | Papel | Permissão (role) |
|------|-------|------------------|
| Doador | Faz a doação | `Doador` |
| PaymentAPI (mock) | Processa o pagamento | — (serviço) |
| Consumer (DonationAPI) | Consolida a arrecadação | — (interno) |

## 3. User Stories
- Como **Doador**, quero doar para uma campanha ativa, para apoiar a causa.
- Como **Doador**, quero consultar o status da minha doação, para saber se foi aprovada.
- Como **sistema**, quero consolidar a arrecadação de forma idempotente, para o total ficar correto mesmo com reentrega.

## 4. Requisitos Funcionais
| ID | Requisito | Prioridade |
|----|-----------|-----------|
| RF-1 | Registrar intenção de doação (assíncrona, 202) | Must |
| RF-2 | Processar pagamento via gateway mock | Must |
| RF-3 | Consolidar `ValorArrecadado` no consumer (idempotente) | Must |
| RF-4 | Consultar status da própria doação | Must |

## 5. Regras de Negócio

**Saga (2 eventos):**
```
Doador → POST /doacoes → DonationAPI grava Doacao=Pendente → publica DoacaoSolicitadaEvent
   → PaymentAPI consome → processa (mock) → publica PagamentoAprovadoEvent | PagamentoRecusadoEvent
   → consumer da DonationAPI consome o resultado:
        Aprovado → Doacao=Aprovada, consolida ValorArrecadado, projeta no Cosmos, conclui se meta batida
        Recusado → Doacao=Recusada (não contabiliza)
```

- **RN06.1** — Exige usuário autenticado com role `Doador`.
- **RN06.2** — A campanha informada deve existir.
- **RN06.3** — Doação só em campanha `Ativa` **e dentro do período** (`DataInicio ≤ agora ≤ DataFim`); senão **422**.
- **RN06.4** — `ValorDoacao` > 0.
- **RN06.5** — `FormaPagamento` obrigatória: `PIX` · `Cartão` · `Transferência`.
- **RN06.6** — Consolidação só **após** `PagamentoAprovadoEvent`.
- **RN06.7** — O endpoint não escreve `ValorArrecadado` de forma síncrona; só o consumer.
- **RN06.8** — A API responde **202 Accepted**: a intenção é aceita; pagamento e consolidação ocorrem depois.
- **RN06.9** — Pagamento recusado gera `PagamentoRecusadoEvent`; a `Doacao` fica `Recusada` e não é contabilizada.
- **RN06.10** — O consumer é **idempotente**: a mesma doação (`doacaoId`) não soma duas vezes (dedup via tabela de eventos processados).
- **RN06.11** — O gateway mock **aprova por padrão** e **recusa valores cujos centavos sejam `,99`** (marcador de teste determinístico); ajustável por configuração. 🆕
- **RN06.12** — A `Doacao` tem status `Pendente` (criação) → `Aprovada` ou `Recusada` (após o pagamento). 🆕
- **RN06.13** — O doador acompanha o resultado por **consulta** (`GET /doacoes/{id}`) **e** recebe uma **notificação assíncrona** do resultado, disparada por uma Azure Function (`NotificationFunction`) que consome os eventos de pagamento — canal **mock/log** no MVP (envio real de email/push fica fora de escopo). Ver [[PRD-07 - Notificações]]. 🆕
- **RN06.14** — Ao consolidar uma doação aprovada, o consumer também **projeta a campanha no Cosmos** (RN04.12) e, se `ValorArrecadado ≥ Meta`, **conclui a campanha** como `MetaAtingida` (RN04.10). 🆕

## 6. Requisitos Não-Funcionais
- **Assíncrono/Resiliência:** entrega ao-menos-uma-vez (RNF11); reentrega + DLQ no Service Bus (RNF13); idempotência (RNF12).
- **Consistência:** `ValorArrecadado` eventualmente consistente (RNF15).
- **Segurança:** RBAC `Doador` (RNF20); auditoria da doação (RNF24).

## 7. Modelo de Domínio (DDD)
- **Bounded Contexts:** Doações (intenção/status) e Arrecadação (consolidação) → ver [[Bounded Contexts]].
- **Agregados:** `Doacao` (DonationAPI); `Pagamento` (PaymentAPI).
- **VOs:** `ValorDoacao`, `FormaPagamento`, `StatusDoacao` (`Pendente`/`Aprovada`/`Recusada`).
- **Consumer/Worker:** BackgroundService na DonationAPI; mantém uma tabela de **eventos processados (inbox)** para idempotência.
- **Invariantes:** consolidação só de doação aprovada; uma doação conta no máximo uma vez; endpoint nunca escreve `ValorArrecadado`.

## 8. Contratos / API
| Método | Rota | Auth | Request | Response |
|--------|------|------|---------|----------|
| POST | `/doacoes` | `Doador` | `{ idCampanha, valorDoacao, formaPagamento }` | `202 { doacaoId, status: "Pendente" }` / `422` (campanha inválida/encerrada/fora do período) |
| GET | `/doacoes/{id}` | `Doador` | — | `200 { doacaoId, status, valorDoacao, idCampanha }` |

## 9. Eventos de Domínio
| Evento | Publica | Consome | Quando |
|--------|---------|---------|--------|
| `DoacaoSolicitadaEvent` | DonationAPI | PaymentAPI | Doação registrada (`Pendente`) |
| `PagamentoAprovadoEvent` | PaymentAPI | DonationAPI (consumer) · NotificationFunction | Mock aprovou |
| `PagamentoRecusadoEvent` | PaymentAPI | DonationAPI (consumer) · NotificationFunction | Mock recusou |
> Detalhe e payloads em [[Domain Events]].

## 10. Critérios de Aceite (Gherkin)
```gherkin
Cenário: Doação aprovada em campanha ativa
  Dado um Doador autenticado e uma campanha Ativa dentro do período
  Quando ele faz POST em /doacoes com valor válido (centavos ≠ ,99)
  Então a API responde 202 com status Pendente
  E a DonationAPI publica DoacaoSolicitadaEvent
  E após o mock aprovar, o consumer consolida o ValorArrecadado

Cenário: Pagamento recusado pelo marcador de teste
  Dado um Doador autenticado e uma campanha Ativa
  Quando ele doa um valor com centavos ,99
  Então o mock recusa e publica PagamentoRecusadoEvent
  E a doação fica Recusada e não é contabilizada

Cenário: Bloquear doação fora do período/encerrada
  Dado uma campanha Concluida ou com DataFim vencida
  Quando um Doador tenta doar
  Então recebe 422 e nenhum evento é publicado

Cenário: Consolidação idempotente
  Dado um PagamentoAprovadoEvent na fila
  Quando o consumer o processa
  Então o ValorArrecadado é incrementado uma vez
  E reprocessar o mesmo evento não soma novamente

Cenário: Conclusão por meta
  Dado que a doação faz o ValorArrecadado atingir a Meta
  Quando o consumer consolida
  Então a campanha vira Concluida com MotivoConclusao = MetaAtingida
```

## 11. Dependências e Integrações
- Depende de **PRD-01** (auth) e **PRD-04** (campanha/status/período). Alimenta **PRD-05** (Painel).
- Integra com **PaymentAPI** (mock), **Azure Service Bus** e o **consumer** da DonationAPI. Os eventos de resultado também alimentam a **NotificationFunction** (notificação ao doador) — ver [[PRD-07 - Notificações]].
- Serviços: **DonationAPI** + **PaymentAPI**. Persistência: SQL Server (Doacao, Pagamento, inbox) + projeção Cosmos.

## 12. Diagramas

```mermaid
sequenceDiagram
    actor D as Doador
    participant DON as DonationAPI
    participant SB as Service Bus
    participant PAY as PaymentAPI mock
    participant CON as Consumer
    D->>DON: POST /doacoes (idCampanha, valor, forma)
    DON->>DON: valida campanha Ativa + período
    DON->>DON: grava Doacao = Pendente
    DON-->>D: 202 Accepted (doacaoId)
    DON->>SB: DoacaoSolicitadaEvent
    SB->>PAY: DoacaoSolicitadaEvent
    PAY->>PAY: processa (mock: recusa se centavos = ,99)
    alt aprovado
        PAY->>SB: PagamentoAprovadoEvent
        SB->>CON: PagamentoAprovadoEvent
        CON->>CON: Doacao=Aprovada; +ValorArrecadado (idempotente)
        CON->>CON: projeta Cosmos; conclui se meta batida
    else recusado
        PAY->>SB: PagamentoRecusadoEvent
        SB->>CON: PagamentoRecusadoEvent
        CON->>CON: Doacao=Recusada (não contabiliza)
    end
    D->>DON: GET /doacoes/{id} (consulta status)
```

## 13. Fora de Escopo
- Gateway de pagamento real / credenciais.
- Estorno/refund, parcelamento, múltiplas moedas.
- Envio **real** de email/push do resultado (provedor externo). A notificação existe no MVP, porém com canal **mock/log** — ver [[PRD-07 - Notificações]].
- Doação anônima (exige login).

## 14. Riscos / Pontos de Atenção
- **Mock não é pagamento real** — deixar claro na demo; trocar por provedor real é trabalho futuro.
- **Ordenação/reentrega de eventos:** consumer precisa tolerar fora de ordem e duplicado (idempotência — RN06.10).
- **Concorrência na conclusão por meta** (RN06.14): transição única mesmo com doações simultâneas.
- **Falha de pagamento/processamento:** após N tentativas, mensagem vai para a **DLQ** (RNF13) — precisa de tratamento/observabilidade (RNF29).
```
