---
tags: [hackaton-fiap, prd, indice]
projeto: Conexão Solidária
---

# Índice de PRDs

> Um PRD por feature/capacidade. Crie cada um a partir do [[_Template - PRD]].

| RF | PRD | Bounded Context | Status |
|----|-----|-----------------|--------|
| RF01 | [[PRD-01 - Autenticação & Autorização]] | Identidade & Acesso | 🟦 rascunho |
| RF02 | [[PRD-02 - Gerenciamento de Usuários]] | Identidade & Acesso | 🟦 rascunho |
| RF03 | [[PRD-03 - Cadastro de Doador]] | Identidade & Acesso | 🟦 rascunho |
| RF04 | [[PRD-04 - Gestão de Campanhas]] | Campanhas | 🟦 rascunho |
| RF05 | [[PRD-05 - Painel de Transparência]] | Transparência | 🟦 rascunho |
| RF06 | [[PRD-06 - Processo de Doação]] | Doações | 🟦 rascunho |
| RF07 | [[PRD-07 - Notificações]] | Notificações | 🟦 rascunho |
| — | Worker/consumer (Arrecadação) | Arrecadação | ✅ coberto no [[PRD-06 - Processo de Doação]] |

> Decidido: o Worker (Arrecadação) **não** vira PRD à parte — é o **consumer/BackgroundService** descrito como seção do [[PRD-06 - Processo de Doação]] (saga + consolidação idempotente).
