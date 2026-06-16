---
tags: [hackaton-fiap, arquitetura, adr]
projeto: Conexão Solidária
---

# Decisões de Arquitetura (ADRs)

> Registre cada decisão relevante (contexto → decisão → consequências).

## ADR-001 — Notificações via Azure Function gerenciada (Service Bus pub/sub, canal mock)
- **Contexto:** O doador precisa ser avisado do resultado do pagamento (aprovado/recusado). O resultado já trafega como evento no Azure Service Bus (`PagamentoAprovadoEvent`/`PagamentoRecusadoEvent`), hoje consumido apenas pelo consumer de arrecadação da DonationAPI. Queremos notificar **sem** acoplar essa responsabilidade à saga de arrecadação nem introduzir provedor externo de email no MVP.
- **Decisão:** Criar a **NotificationFunction** — uma **Azure Function gerenciada** (plano Consumption, fora do AKS) acionada por `ServiceBusTrigger`, consumindo os eventos de resultado por uma **subscription própria** (tópico/pub-sub). Canal de notificação **mock/log** (log estruturado + Application Insights). Os eventos são **enriquecidos** com `doadorId`/`doadorEmail`/`doadorNome` (origem: claims do JWT na DonationAPI, repassados pela PaymentAPI), evitando chamada síncrona à UserAPI. Capacidade detalhada em [[PRD-07 - Notificações]].
- **Consequências:**
  - (+) Desacoplamento total da arrecadação: falha na notificação não afeta a consolidação do `ValorArrecadado`.
  - (+) Uso idiomático de serverless; trigger nativo de Service Bus; escala automática.
  - (−) A topologia dos eventos de resultado passa de consumidor único para **tópico + múltiplas subscriptions**.
  - (−) Observabilidade fragmentada: a Function fica **fora** do Prometheus/Grafana in-cluster (métricas em Application Insights).
  - (−) Deploy/CI-CD próprios (`azure/functions-action`), separados do pipeline de imagem → AKS.
  - (−) *At-least-once*: reentrega pode gerar log de notificação duplicado (aceitável no MVP).
