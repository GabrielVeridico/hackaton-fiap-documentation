---
tags: [hackaton-fiap, ddd]
projeto: Conexão Solidária
status: definido
---

# Context Map

> Relação entre os Bounded Contexts e como se integram. Detalhe de cada contexto em [[Bounded Contexts]]; eventos em [[Domain Events]].

## Mapa

```mermaid
flowchart LR
    subgraph IA["Identidade & Acesso · UserAPI"]
        Auth["Auth/JWT + Usuários"]
    end
    subgraph Pag["Pagamentos · PaymentAPI"]
        Pagamento["Pagamento (mock)"]
    end
    subgraph Don["DonationAPI"]
        Campanha["Campanhas"]
        Doacao["Doações"]
        Consol["Arrecadação (consumer)"]
        Painel["Transparência (read model)"]
    end
    subgraph Notif["Notificações · NotificationFunction (Azure Function)"]
        Notificacao["Notificação ao doador (mock)"]
    end

    Auth -- "identidade/role (upstream)" --> Doacao
    Auth -- "identidade/role (upstream)" --> Campanha
    Doacao -- "valida campanha" --> Campanha
    Doacao -- "DoacaoSolicitadaEvent" --> Pagamento
    Pagamento -- "Pagamento Aprovado/Recusado" --> Consol
    Pagamento -- "Pagamento Aprovado/Recusado" --> Notificacao
    Consol -- "atualiza ValorArrecadado" --> Campanha
    Consol -- "projeta" --> Painel
    Campanha -- "projeta status" --> Painel
```

## Relações (resumo)
- **Identidade & Acesso** é *upstream* de todos (fornece identidade e role).
- **Doações** depende de **Campanhas** (campanha precisa existir, estar `Ativa` e dentro do período).
- **Pagamentos** integra com Doações via eventos (saga) — sem acesso a banco alheio.
- **Arrecadação** (consumer) consolida o `ValorArrecadado` em **Campanhas** e projeta o read model de **Transparência**.
- **Transparência** é *downstream* puro: só lê a projeção (Cosmos).
- **Notificações** consome o resultado do pagamento (subscription própria, pub/sub) e **notifica o doador** (canal mock); é *downstream* da Pagamentos e **independente** da Arrecadação. Ver [[PRD-07 - Notificações]].

**Relacionados:** [[Bounded Contexts]] · [[Domain Events]] · [[Visão Geral de Arquitetura]]
