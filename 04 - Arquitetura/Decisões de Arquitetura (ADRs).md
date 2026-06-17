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

## ADR-002 — Zabbix para monitoramento de infraestrutura e disponibilidade (complementar ao Prometheus/Grafana)
- **Contexto:** O enunciado exige observabilidade com dashboards de métricas reais, hoje atendida por **OpenTelemetry → Prometheus → Grafana** (métricas de aplicação e negócio). Além disso, o time precisa de monitoramento de **infraestrutura/host** (CPU, memória, pods/nós do AKS) e de **disponibilidade/uptime** com **alertas** operacionais — capacidades em que o Zabbix é maduro e idiomático. Decisões anteriores haviam descartado o Zabbix; essa posição foi **revista**.
- **Decisão:** Adotar o **Zabbix** como camada **complementar** de observabilidade, responsável por **infra/host, disponibilidade e alertas**, **sem** substituir o Prometheus/Grafana (que seguem com as métricas de aplicação/negócio). Coleta por **três vias**: (1) *scrape* dos endpoints `/metrics` (formato Prometheus) via item HTTP + *preprocessing* Prometheus; (2) **Zabbix Agent** (DaemonSet) com templates de host/Kubernetes para CPU/memória/pods/nós; (3) *web scenarios* monitorando `/health` e `/ready` para uptime/disponibilidade. A **NotificationFunction** (Azure Function, fora do AKS) **permanece** em Application Insights / Azure Monitor — fora do escopo do Zabbix.
- **Consequências:**
  - (+) Monitoramento de infra e alertas de disponibilidade maduros, com escalonamento/notificação nativos do Zabbix.
  - (+) Reaproveita os endpoints `/metrics` e `/health` já expostos pelos serviços — **sem mudança na aplicação**.
  - (+) Divisão clara de responsabilidades: **Grafana/Prometheus** = aplicação/negócio; **Zabbix** = infra/host/disponibilidade.
  - (−) Duas plataformas de observabilidade para operar e manter (configuração do Zabbix Server + Agent no cluster).
  - (−) Possível sobreposição de métricas de infra entre Prometheus e Zabbix — definir a fonte de verdade por tipo de métrica para evitar alertas duplicados.
  - (−) A Function segue fora do painel unificado (continua em Application Insights).
