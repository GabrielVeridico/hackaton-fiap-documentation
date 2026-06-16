---
tags: [hackaton-fiap, visao-geral]
projeto: Conexão Solidária
---

# 🔑 Pontos-Chave do Projeto

> Síntese do que o projeto **obrigatoriamente** precisa ter, extraída do enunciado e já refletindo as **decisões de stack** do time. Use como checklist macro. Detalhamento em [[Requisitos Funcionais]] e [[Requisitos Técnicos]].

## A. Capacidades Funcionais (o quê)
1. **Autenticação & Autorização (RBAC)** — JWT; roles `GestorONG` e `Doador`; endpoints de gestão restritos a `GestorONG`.
2. **Gestão de Campanhas** (GestorONG) — CRUD com regras (data fim não no passado, meta > 0, status `Ativa`/`Concluída`/`Cancelada`).
3. **Cadastro de Doador** (público) — PF ou PJ: TipoPessoa, Documento (CPF/CNPJ, único + validação de dígitos), Nome/Razão Social, Email (único), Senha (hash/BCrypt).
4. **Painel de Transparência** (público) — lista só campanhas `Ativa`; exibe Título, Meta e Valor Arrecadado.
5. **Processo de Doação** (Doador logado) — intenção (IdCampanha + Valor); proibido doar em campanha encerrada/cancelada.
6. **Notificação ao Doador** — ao processar o pagamento, uma **Azure Function** notifica o doador do resultado (aprovado/recusado); canal **mock/log** no MVP. Ver [[PRD-07 - Notificações]].

## B. Pilares Técnicos Obrigatórios (como)
1. **Microsserviços** — **UserAPI**, **DonationAPI** e **PaymentAPI** (≥ 2 exigidos). Ver [[Requisitos Técnicos]].
2. **Mensageria assíncrona** — broker **Azure ServiceBus**. Saga de 2 eventos: DonationAPI publica `DoacaoSolicitadaEvent` → PaymentAPI processa (mock) e publica `PagamentoAprovadoEvent`/`PagamentoRecusadoEvent` → consumer da DonationAPI consolida o Valor Arrecadado. O endpoint de doação **não** atualiza o valor direto (ver [[Domain Events]]). Os eventos de resultado também são consumidos por uma **Azure Function** (`NotificationFunction`, gerenciada, fora do AKS) que **notifica o doador** — segundo consumidor via tópico/subscription (ver [[PRD-07 - Notificações]]).
3. **Kubernetes** — cluster **AKS (Azure)**, deploy via **Helm**. Entregar `.yaml`/charts (Deployments, Services, ConfigMaps).
4. **Observabilidade** — expor `/health` e `/metrics` (OpenTelemetry); **Grafana + Prometheus** com ≥1 dashboard de métricas reais (CPU/memória dos pods, contagem de requisições). **Sem Zabbix.**
5. **CI/CD** — **GitHub Actions** a cada push na branch principal: roda testes, compila (.NET build) e gera imagem Docker; deploy no AKS. (Mínimo exigido pelo enunciado: gerar a imagem.)

## C. Bônus (sem impacto na nota)
- Testes de unidade (**xUnit + NSubstitute**) no domínio, rodando na esteira de CI.
- API Gateway — **Azure APIM** roteando as rotas públicas (rate limit + normalização).

## D. Entregáveis
1. **Repositório público** + `README.md` com passo a passo para subir infra e app localmente.
2. **Desenho de arquitetura** (diagrama: microsserviços, bancos, broker, observabilidade) + **PDF justificando a escolha dos 2 bancos** (SQL Server + Cosmos DB — ver [[Escolha de Bancos de Dados]]).
3. **Vídeo demo (≤15 min)**: diagrama → pipeline CI gerando imagem → `kubectl get pods` + dashboard Grafana → auth/JWT → criar campanha → doação (payload → ServiceBus → valor atualizado pelo consumer, lido do Cosmos).
4. **Relatório de entrega** (PDF/TXT): nome do grupo, participantes + usernames Discord, links (doc, repo, vídeo).

## E. Decisões-chave (fechadas)
- **Cloud** ✅ **Azure** (AKS, ServiceBus, Cosmos DB, APIM).
- **Dois bancos diferentes** ✅ **X = SQL Server** (escrita transacional, um banco por serviço) + **Y = Cosmos DB** (leitura do Painel). Ver [[Escolha de Bancos de Dados]].
- **Consistência eventual** entre a doação enviada e o Valor Arrecadado exibido (janela alvo < 10 s — RNF15).
- **Idempotência** no consumer da DonationAPI (reentrega de mensagem não soma a doação 2x — RN06.10 / RNF12).
- **Stack .NET** ✅ **.NET 8**; broker ✅ **Azure ServiceBus**.
- **Observabilidade** ✅ **Grafana + Prometheus + OpenTelemetry** (sem Zabbix).
