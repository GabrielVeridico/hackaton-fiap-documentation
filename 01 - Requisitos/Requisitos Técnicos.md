---
tags: [hackaton-fiap, requisitos, tecnicos]
projeto: Conexão Solidária
status: definido
---

# Requisitos Técnicos

> Stack técnica decidida para o MVP — consolida os requisitos obrigatórios do enunciado e as escolhas do time. Justificativa dos bancos em [[Escolha de Bancos de Dados]]; decisões com trade-offs em [[Decisões de Arquitetura (ADRs)]].

## 1. Visão geral da stack

| Dimensão | Decisão | Detalhe / referência |
| --- | --- | --- |
| Cloud | **Azure** | todos os deploys |
| Runtime | **.NET 8** | todos os serviços |
| Microsserviços | **UserAPI · DonationAPI · PaymentAPI** | ver §2 |
| Mensageria | **Azure Service Bus** | saga de 2 eventos — §3 · [[Domain Events]] |
| Banco de escrita (X) | **SQL Server** | um banco por serviço — [[Escolha de Bancos de Dados]] |
| Banco de leitura (Y) | **Cosmos DB** (API NoSQL) | read model do Painel — [[Escolha de Bancos de Dados]] |
| Cluster | **AKS** (Azure Kubernetes Service) | deploy via Helm — §7 |
| Observabilidade | **OpenTelemetry + Prometheus + Grafana** | sem Zabbix — §5 |
| API Gateway | **Azure APIM** | rate limit + normalização — §6 |
| CI/CD | **GitHub Actions** | testes + build + imagem + deploy — §8 |
| Testes | **xUnit + NSubstitute** | em todos os serviços — §9 |
| Pagamento | **Gateway mock/simulado** | na PaymentAPI — §2 |
| Notificações | **Azure Functions** (gerenciado) | worker orientado a evento; consome o Service Bus; canal mock — §2 · [[PRD-07 - Notificações]] |

## 2. Microsserviços

| Serviço | Responsabilidade |
| --- | --- |
| **Hackaton-Fiap-UserAPI** | Autenticação (JWT) e dados de usuário. **No MVP não publica eventos cross-service** — auth/users são um bounded context isolado; nenhum consumer ou producer de Service Bus está presente neste serviço. |
| **Hackaton-Fiap-DonationAPI** | Doações, gestão de campanhas e Painel de Transparência. Hospeda o **consumer** que consolida o `ValorArrecadado`. |
| **Hackaton-Fiap-PaymentAPI** | Processamento de pagamentos. Gateway **simulado/mock** (aprovação/recusa determinística), sem provedor externo real. |

> Além dos 3 microsserviços, há a **NotificationFunction** — uma **Azure Function gerenciada** (fora do AKS), worker orientado a evento que **notifica o doador** do resultado do pagamento (canal **mock/log**). Ver [[PRD-07 - Notificações]].

## 3. Mensageria & fluxo de eventos

Broker: **Azure Service Bus**. O processamento de uma doação é uma **saga de 2 eventos**, respeitando o isolamento de dados (um banco por serviço — ver [[Domain Events]]):

1. **DonationAPI** recebe a doação (`POST /doacoes`), grava a `Doacao` como `Pendente` e publica `DoacaoSolicitadaEvent`.
2. **PaymentAPI** consome `DoacaoSolicitadaEvent`, processa o pagamento (gateway **mock**) e publica `PagamentoAprovadoEvent` ou `PagamentoRecusadoEvent`.
3. **DonationAPI** consome o resultado em um **consumer/BackgroundService próprio**: se aprovado, consolida o `ValorArrecadado` de forma **idempotente** (dedup por `doacaoId`) e projeta a campanha no read store (§4).
4. A **NotificationFunction** (Azure Function gerenciada) consome os **mesmos** eventos de resultado por uma **subscription própria** e **notifica o doador** (canal mock) — independente da consolidação.

Como os eventos de resultado têm dois consumidores independentes, são distribuídos via **tópico + subscriptions** (pub/sub). Falhas de consumo usam *max delivery count* + **DLQ** do Service Bus.

## 4. Persistência (CQRS)

Dois bancos com propósitos distintos (entregável obrigatório — ver [[Escolha de Bancos de Dados]]):

- **Escrita (X) — SQL Server:** dados transacionais com consistência forte (Usuários, Campanhas, Doações), **um banco por serviço** (database-per-service).
- **Leitura (Y) — Cosmos DB (API NoSQL):** *read model* público e escalável que alimenta o **Painel de Transparência** (RF05). Documento denormalizado por campanha (`{ id, titulo, meta, valorArrecadado, status }`), `partition key = /id`, consistência `Session`.

O **consumer da DonationAPI**, ao processar `PagamentoAprovadoEvent`, atualiza o SQL Server **e projeta** a campanha atualizada no Cosmos DB. O Painel de Transparência lê **somente** do Cosmos DB.

## 5. Observabilidade & logging

- **OpenTelemetry** em todos os serviços; `correlationId` e `traceId` propagados entre requisições.
- **Logging** estruturado em JSON de request e response.
- Cada serviço expõe as rotas **`/health`** e **`/metrics`**.
- **Prometheus** coleta as métricas; **Grafana** exibe os dashboards com métricas reais. **Não** será usado Zabbix.
- A **NotificationFunction** roda **fora do AKS**; sua observabilidade é via **Application Insights / Azure Monitor** (portanto fora do Prometheus/Grafana in-cluster).

## 6. API Gateway

**Azure APIM** expõe as rotas públicas, aplicando **rate limit** e **normalização** da rota chamada.

## 7. Kubernetes & deploy

- Cluster **AKS** (Azure Kubernetes Service); ConfigMaps para configuração.
- Deploy facilitado por **Helm** (charts dos manifests `.yaml`: Deployments, Services, ConfigMaps).

## 8. CI/CD

**GitHub** para versionamento; **GitHub Actions** executa o fluxo a cada push na branch principal: roda os testes de unidade, compila (.NET build), gera a imagem Docker e faz o deploy. A **NotificationFunction** tem um workflow próprio (`azure/functions-action`), separado do pipeline de imagem → AKS.

## 9. Testes

Testes de unidade com **xUnit + NSubstitute** em todos os microsserviços, executados na esteira de CI.

## 10. Conformidade com o enunciado

| Requisito (enunciado) | Status | Como atendemos |
| --- | :---: | --- |
| Microsserviços (≥ 2 serviços) | ✅ | 3 serviços (§2) |
| Mensageria assíncrona (broker + consumer) | ✅ | Service Bus + saga + consumer (§3) |
| Kubernetes (Deployments/Services/ConfigMaps) | ✅ | AKS + Helm (§7) |
| Observabilidade (`/health` ou `/metrics` + Grafana) | ✅ | OpenTelemetry + Prometheus + Grafana (§5) |
| CI/CD (build .NET + imagem a cada push) | ✅ | GitHub Actions (§8) |
| Testes de unidade *(bônus)* | ✅ | xUnit + NSubstitute (§9) |
| API Gateway *(bônus)* | ✅ | Azure APIM (§6) |
