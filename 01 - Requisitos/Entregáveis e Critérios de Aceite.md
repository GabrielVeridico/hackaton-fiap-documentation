---
tags: [hackaton-fiap, entregaveis, criterios-aceite]
projeto: Conexão Solidária
---

# Entregáveis e Critérios de Aceite

## Entregáveis
- [ ] **Repositório público** com `README.md` (passo a passo para subir infra + app localmente).
- [ ] **Diagrama de arquitetura** (microsserviços, bancos, broker, observabilidade).
- [ ] **PDF de justificativa** da escolha dos 2 bancos de dados (SQL Server + Cosmos DB — ver [[Escolha de Bancos de Dados]]).
- [ ] **Vídeo demo** (≤ 15 min) — roteiro abaixo.
- [ ] **Relatório de entrega** (PDF/TXT) — grupo, participantes + Discord, links.

## Roteiro obrigatório do vídeo
1. Explicação do diagrama de arquitetura.
2. Pipeline de CI executando e gerando a imagem Docker.
3. `kubectl get pods` (pods rodando) + dashboard Grafana em tempo real.
4. Funcionamento:
   1. Autenticação via Postman/Swagger → token JWT.
   2. Criação de uma campanha.
   3. Doação: payload enviado → mensagem no broker (Azure ServiceBus) → pagamento aprovado (mock) → API pública (lendo do Cosmos DB) provando o valor atualizado pelo consumer.
