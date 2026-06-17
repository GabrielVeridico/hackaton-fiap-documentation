---
tags: [hackaton-fiap, requisitos, nao-funcionais]
projeto: Conexão Solidária
status: em-definição
---

# Requisitos Não Funcionais

> Atributos de qualidade do sistema (o **como**, não o **quê**). Complementam os [[Requisitos Funcionais]] e se apoiam nos [[Requisitos Técnicos]] obrigatórios do enunciado.
>
> **Como preencher:** a coluna **Meta / Valor alvo** está como `_a definir_` — defina o número/limite de cada linha. Ajuste a **Prioridade** (MoSCoW) e remova/adicione linhas conforme necessário. Cada RNF deve ser **mensurável e verificável**.

## Convenções

- **ID:** `RNFxx` (numeração contínua, independente da categoria).
- **Prioridade (MoSCoW):** `Must` (obrigatório p/ entrega) · `Should` (importante) · `Could` (desejável) · `Won't` (fora de escopo neste MVP).
- **Verificação:** como o atendimento será comprovado (teste de carga, dashboard, revisão, etc.).
- Onde o enunciado já obriga algo (microsserviços, mensageria, K8s, observabilidade, CI/CD), o RNF apenas **quantifica** a meta.

---

## 1. Desempenho (Performance)

| ID    | Requisito                                                                                         | Como medir / verificar                     | Meta / Valor alvo               | Prioridade |
| ----- | ------------------------------------------------------------------------------------------------- | ------------------------------------------ | ------------------------------- | ---------- |
| RNF01 | Latência das APIs de leitura (ex.: `/transparencia/campanhas`) sob carga nominal                  | Teste de carga (k6/JMeter), p95            | _a definir_ (ex.: p95 < 300 ms) | Wont       |
| RNF02 | Latência das APIs de escrita (ex.: `POST /doacoes`) — resposta `202 Accepted`                     | Teste de carga, p95                        | _a definir_ (ex.: p95 < 500 ms) | Wont       |
| RNF03 | Throughput mínimo de doações aceitas por segundo                                                  | Teste de carga                             | _a definir_ (ex.: ≥ 100 req/s)  | Wont       |
| RNF04 | Tempo de consolidação do consumer (consumo do `PagamentoAprovadoEvent` → `ValorArrecadado` atualizado) | Medição fim-a-fim / métrica de lag da fila | _a definir_ (ex.: < 5 s em p95) | Wont       |

---

## 2. Escalabilidade

| ID    | Requisito                                                                   | Como medir / verificar                          | Meta / Valor alvo                       | Prioridade |
| ----- | --------------------------------------------------------------------------- | ----------------------------------------------- | --------------------------------------- | ---------- |
| RNF05 | Escala horizontal dos serviços (stateless) via réplicas no Kubernetes       | Manifests com `replicas` / HPA; teste de escala | 1 réplica a 2 réplicas. Ideal 1 réplica | Should     |
| RNF06 | Autoscaling baseado em CPU/memória ou tamanho da fila                       | HPA / KEDA configurado e testado                | 70% da CPU                              | Should     |
| RNF07 | Worker escala com o backlog da fila sem perda de mensagens                  | Teste com pico de mensagens                     | Nenhuma                                 | Wont       |
| RNF08 | Capacidade de dados suportada (campanhas, doadores, doações) sem degradação | Teste com volume sintético                      | Não terá                                | Wont       |

---

## 3. Disponibilidade & Confiabilidade

| ID    | Requisito                                                                            | Como medir / verificar                     | Meta / Valor alvo                     | Prioridade  |
| ----- | ------------------------------------------------------------------------------------ | ------------------------------------------ | ------------------------------------- | ----------- |
| RNF09 | Disponibilidade (uptime) dos serviços críticos                                       | Uptime via **Zabbix** (`/health`/`/ready`) + Grafana/Prometheus         | 99,5%                                 | Must        |
| RNF10 | Health checks (`/health` liveness + readiness) usados pelo K8s para auto-recuperação | Probes nos manifests; teste de kill de pod | Criar rotas nas APIs de health check. | Must        |
| RNF11 | Entrega garantida de eventos (ao menos uma vez) com fila de mensagens mortas (DLQ)   | DLQ configurada; teste de falha de consumo | Eventualmente processar               | Must        |
| RNF12 | Idempotência do consumer da DonationAPI (reprocessar `PagamentoAprovadoEvent` não soma em duplicidade) | Teste de reentrega — ver **RN06.10** | Dedup por `doacaoId` via tabela de eventos processados (inbox) | Must |
| RNF13 | Reentrega e tratamento de falha no consumo de eventos (ServiceBus) | Max delivery count + DLQ; teste de falha de consumo | 3 tentativas, depois DLQ | Must |
| RNF14 | RTO/RPO em caso de falha (recuperação e perda máxima de dados) | Plano + teste de restauração | Fora de escopo do MVP (hackathon) | Wont |

---

## 4. Consistência de Dados

| ID    | Requisito                                                                          | Como medir / verificar                          | Meta / Valor alvo      | Prioridade |
| ----- | ---------------------------------------------------------------------------------- | ----------------------------------------------- | ---------------------- | ---------- |
| RNF15 | `ValorArrecadado` reflete consistência eventual; janela máxima de atraso aceitável | Medição do lag de consolidação — ver **RN05.4** | < 10 s (p95) entre pagamento aprovado e consolidação | Must |
| RNF16 | Nenhuma doação aprovada é perdida (evento sempre consolidado)                      | Reconciliação eventos publicados × consolidados | _a definir_ (0 perdas) | Must       |

---

## 5. Segurança

| ID    | Requisito                                                                             | Como medir / verificar                  | Meta / Valor alvo                                          | Prioridade |
| ----- | ------------------------------------------------------------------------------------- | --------------------------------------- | ---------------------------------------------------------- | ---------- |
| RNF17 | Tráfego cifrado em trânsito (TLS/HTTPS)                                               | Configuração de ingress/gateway         | _a definir_ (ex.: TLS 1.2+)                                | Must       |
| RNF18 | Senhas armazenadas apenas como hash (BCrypt)                                          | Revisão de código — ver **RN03.5**      | Senhas criptografadas no banco de dados                    | Must       |
| RNF19 | Autenticação JWT (access 4h) + refresh token rotativo e revogável (7d)                | Revisão — ver **RN01.3/RN01.9** e [[PRD-01 - Autenticação & Autorização]] | Access 4h + refresh 7d (hash, rotativo) | Must       |
| RNF20 | Autorização por role (RBAC) em endpoints protegidos                                   | Testes de 401/403 — ver **RF01**        | Membros com as autenticações devidas                       | Must       |
| RNF21 | Segredos (connection strings, chaves) fora do código, via K8s Secrets/ConfigMaps      | Revisão de manifests                    | Utilizar key vaults                                        | Must       |
| RNF22 | Proteção contra abuso (rate limiting / throttling) em endpoints públicos              | Configuração no API Gateway / teste     | Utilização do APIM                                         | Must       |
| RNF23 | Validação de entrada contra injeção e dados malformados (OWASP Top 10)                | Revisão + testes                        | Testes de unidade                                          | Must       |
| RNF24 | Trilha de auditoria de ações sensíveis (login, criação/inativação de usuário, doação) | Logs estruturados / auditoria           | Logs com trace e correlationId para acompanhar os caminhos | Must       |

---

## 6. Observabilidade & Monitoramento

> **Divisão de responsabilidades:** **Prometheus + Grafana** cobrem métricas de **aplicação e negócio** (via OpenTelemetry); **Zabbix** cobre **infraestrutura/host, disponibilidade e alertas** — coleta por *scrape* do `/metrics`, **Zabbix Agent** (host/K8s) e *web scenarios* em `/health`/`/ready`. Ver [[Decisões de Arquitetura (ADRs)|ADR-002]].

| ID    | Requisito                                                                     | Como medir / verificar                           | Meta / Valor alvo                                                              | Prioridade |
| ----- | ----------------------------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------ | ---------- |
| RNF25 | Endpoints `/health` e/ou `/metrics` expostos por cada serviço                 | Verificação direta — ver [[Requisitos Técnicos]] | Endpoint com health e metrics usando OpenTelemetry (consumidos por Prometheus e Zabbix)                             | Must       |
| RNF26 | Dashboards reais: **Grafana** (aplicação/negócio) + **Zabbix** (infra/host)            | Dashboard funcional                              | Dados com métricas reais                                                       | Must       |
| RNF27 | Monitoramento de infra/host (CPU/memória/pods/nós) via **Zabbix**                      | Zabbix Agent (templates host/K8s) + scrape do /metrics       | Infra monitorada no **Zabbix** (dashboards + alertas)                                      | Must       |
| RNF28 | Logs estruturados (JSON) com correlação de requisições (correlation/trace id) | Inspeção de logs                                 | Logs rastreáveis                                                               | Must       |
| RNF29 | Métricas de fila (profundidade, lag, mensagens em DLQ) visíveis               | Dashboard / exporter                             | Métricas no grafana                                                            | Must       |
| RNF30 | Alertas configurados para indisponibilidade e backlog anormal                 | Triggers/ações no **Zabbix** (indisponibilidade, CPU/memória) + Grafana/Prometheus (backlog de fila)         | Alertas para indisponibilidade, consumo excessivo de serviço e uso abusivo de CPU | Must       |

---

## 7. Manutenibilidade & Qualidade

| ID    | Requisito                                                                      | Como medir / verificar                             | Meta / Valor alvo                                         | Prioridade |
| ----- | ------------------------------------------------------------------------------ | -------------------------------------------------- | --------------------------------------------------------- | ---------- |
| RNF31 | Cobertura de testes de unidade no domínio (xUnit/NUnit), rodando na CI         | Relatório de cobertura                             | >=80%                                                     | Must       |
| RNF32 | Padrões de código aplicados (Clean Architecture, DDD, convenções do workspace) | Revisão / linters                                  | Clean Architecture em todos os serviços                   | Must       |
| RNF33 | Documentação de API (Swagger/OpenAPI) disponível por serviço                   | Endpoint Swagger acessível                         | Apenas em ambiente de desenvolvimento                     | Must       |
| RNF34 | Build reprodutível e pipeline CI/CD verde como gate de merge                   | Pipeline configurado — ver [[Requisitos Técnicos]] | CI/CD configurados para deploy e testes no Github Actions | Must       |

---

## 8. Implantação & Portabilidade

| ID    | Requisito                                                                  | Como medir / verificar                 | Meta / Valor alvo                       | Prioridade |
| ----- | -------------------------------------------------------------------------- | -------------------------------------- | --------------------------------------- | ---------- |
| RNF35 | Serviços containerizados (imagem Docker por serviço)                       | Dockerfile + imagem publicada          | Containers definidos                    | Must       |
| RNF36 | Orquestração via manifests Kubernetes (Deployments, Services, ConfigMaps)  | Manifests aplicáveis no cluster        | Utilizar o Helm para facilitar o deploy | Must       |
| RNF37 | CI/CD gera build .NET + imagem Docker a cada push na branch principal      | Pipeline — ver [[Requisitos Técnicos]] | A cada push na main rodar o ciclo       | Must       |
| RNF38 | Limites de recursos (requests/limits) definidos por pod                    | Revisão de manifests                   | Será de acordo com cada serviço         | Should     |
| RNF39 | Configuração externalizada por ambiente (sem rebuild para trocar ambiente) | ConfigMaps/Secrets por ambiente        | Secrets por serviço                     | Must       |

---

## 9. Usabilidade & Acessibilidade _(frontend a definir)_

> Frontend ainda **não definido** — foco atual no backend. RNF40 vale para a API (contrato de erro); RNF41 fica adiado até a stack de frontend ser decidida.

| ID    | Requisito                                                                    | Como medir / verificar                   | Meta / Valor alvo                                      | Prioridade |
| ----- | ---------------------------------------------------------------------------- | ---------------------------------------- | ------------------------------------------------------ | ---------- |
| RNF40 | Mensagens de erro claras e padronizadas (ex.: email/documento já cadastrado) | Revisão de UX/contratos — ver **RN03.1** | Mensagens sempre claras                                | Must       |
| RNF41 | Responsividade / acessibilidade básica do Painel de Transparência            | Teste manual / Lighthouse                | Adiado — frontend ainda não definido (foco no backend) | Wont       |

---

## 10. Conformidade & Privacidade (LGPD)

| ID    | Requisito                                                           | Como medir / verificar   | Meta / Valor alvo                                   | Prioridade |
| ----- | ------------------------------------------------------------------- | ------------------------ | --------------------------------------------------- | ---------- |
| RNF42 | Tratamento de dados pessoais (CPF/CNPJ, email) conforme LGPD        | Revisão de aderência     | Não será tratado                                    | Wont       |
| RNF43 | Não exposição de dados sensíveis em endpoints públicos              | Revisão — ver **RN05.5** | Não deve expor dados sensíveis nunca                | Must       |
| RNF44 | Exclusão lógica (soft-delete) preserva histórico sem remoção física | Revisão — ver **RN02.3** | Todos os registros com softdelete e filtros globais | Should     |
| RNF45 | Política de retenção/anonimização de dados                          | Definição documentada    | Não terá                                            | Wont       |

---

## Resumo / pendências de definição

- [ ] Definir as **metas numéricas** (`_a definir_`) de cada RNF, priorizando as linhas `Must`.
- [ ] Confirmar a **prioridade (MoSCoW)** de cada linha.
- [ ] Revisar quais categorias estão **fora de escopo** do MVP (ex.: seção 9 se não houver frontend próprio).
- [ ] Vincular cada RNF crítico ao [[00 - Índice de PRDs|PRD]] e às [[Decisões de Arquitetura (ADRs)]] correspondentes.

**Relacionados:** [[Requisitos Funcionais]] · [[Requisitos Técnicos]] · [[Entregáveis e Critérios de Aceite]] · [[Visão Geral de Arquitetura]]
