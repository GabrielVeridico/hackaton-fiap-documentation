---
tags: [hackaton-fiap, fonte, enunciado]
projeto: Conexão Solidária
---

# Fonte — Enunciado do Hackathon (transcrição)

> Transcrição fiel do PDF `hackaton-fiap.pdf` (acentuação normalizada). Fonte da verdade para os requisitos.

## Contexto
A ONG **Esperança Solidária** atua há mais de 10 anos acolhendo crianças em situação de vulnerabilidade. Hoje a gestão de doadores e campanhas é manual, limitando a expansão. A diretoria decidiu criar a plataforma digital **"Conexão Solidária"**, focada em escalabilidade, observabilidade e automação. A missão é **arquitetar e desenvolver o MVP**.

## Requisitos Funcionais Detalhados
1. **Autenticação e Autorização (RBAC)** — JWT; roles `GestorONG` e `Doador`; endpoints de gestão bloqueados para não-`GestorONG`.
2. **Gestão de Campanhas (GestorONG)** — cadastro com: Título (string), Descrição (string), DataInicio (datetime), DataFim (datetime), MetaFinanceira (decimal), Status (Ativa/Concluida/Cancelada). Regra: data de término não pode estar no passado; meta > 0.
3. **Cadastro de Doador (público)** — Nome Completo, Email (único), CPF (validar formato), Senha (hash, ex.: BCrypt).
4. **Painel de Transparência (público)** — API que retorna só campanhas `Ativa`; exibir Título, Meta Financeira e Valor Total Arrecadado (calculado pelas doações processadas).
5. **Processo de Doação (Doador logado)** — enviar intenção com IdCampanha e ValorDoacao. Regra: não doar em campanhas encerradas/canceladas.

## Requisitos Técnicos Obrigatórios
- **Microsserviços:** ≥ 2 (ex.: API Campanhas/Usuários + Worker de Doações).
- **Mensageria assíncrona:** ao receber doação, a API NÃO atualiza o valor direto no banco; publica evento (ex.: `DoacaoRecebidaEvent`) em broker (RabbitMQ/Kafka); Worker consome e atualiza o Valor Total Arrecadado.
- **Kubernetes:** cluster (Minikube/Kind/Docker Desktop); entregar `.yaml` (Deployments, Services, ConfigMaps).
- **Observabilidade:** expor `/health` ou `/metrics`; Grafana com ≥1 dashboard de métricas reais (CPU/memória dos pods ou contagem de requisições HTTP).
- **CI/CD:** pipeline a cada push na branch principal; compila (.NET build) e gera imagem Docker; deploy no K8s opcional, gerar imagem obrigatório.

## Requisitos Técnicos Opcionais (bônus)
- Testes de unidade (xUnit/NUnit) no domínio, na CI.
- API Gateway (Ocelot/KrakenD).

## Entregáveis Mínimos e Critérios de Aceite
1. Repositório público com `README.md` (passo a passo).
2. Desenho de arquitetura + PDF justificando os bancos X e Y.
3. Vídeo (≤15 min): diagrama; pipeline CI gerando imagem; `kubectl get pods` + Grafana; auth/JWT; criar campanha; doação (payload → fila → valor atualizado).
4. Relatório de entrega (PDF/TXT): grupo, participantes + Discord, links (doc, repo, vídeo).
