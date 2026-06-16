---
tags: [hackaton-fiap, requisitos, funcionais]
projeto: Conexão Solidária
status: em-revisão
---

# Requisitos Funcionais

> Detalhamento das capacidades funcionais. Cada RF é candidato a um [[00 - Índice de PRDs|PRD]].
>
> Padrão de cada RF: **Ator → Descrição → Regras (RNxx.y) → Endpoints → Critérios de aceite → Dependências**.

## Convenções Gerais (fonte única)

- **Política de senha** (vale para todos os usuários): mínimo 8 caracteres, ≥ 1 letra maiúscula e ≥ 1 símbolo. Armazenada apenas como hash (BCrypt).
- **Soft-delete:** usuários nunca são removidos fisicamente — passam a `Inativo`.
- **Códigos HTTP padronizados:**

| Código                   | Uso                                         |
| ------------------------ | ------------------------------------------- |
| 200 / 201                | Sucesso / recurso criado                    |
| 202 Accepted             | Aceito para processamento assíncrono        |
| 400 Bad Request          | Erro de formato/campos obrigatórios         |
| 401 Unauthorized         | Sem token válido                            |
| 403 Forbidden            | Autenticado, mas role insuficiente          |
| 404 Not Found            | Recurso inexistente                         |
| 409 Conflict             | Conflito/duplicidade (ex.: email já existe) |
| 422 Unprocessable Entity | Violação de regra de negócio                |

---

## RF01 — Autenticação & Autorização (RBAC)

- **Ator:** Todos os usuários (Visitante, `Doador`, `GestorONG`)
- **Descrição:** O sistema autentica usuários via **JWT (Bearer token)** e autoriza o acesso a endpoints com base na **role** do usuário. O token é emitido no login e carrega as claims de identidade (`userId`, `email`, `role`). Endpoints de gestão são acessíveis **somente** por `GestorONG`.

**Regras de negócio:**
- **RN01.1** — Existem exatamente duas roles: `GestorONG` e `Doador`.
- **RN01.2** — O login exige email + senha válidos; a senha é comparada contra o hash (BCrypt) armazenado.
- **RN01.3** — O **access token** (JWT) deve conter as claims `userId`, `email` e `role`, com prazo de expiração de **4h**.
- **RN01.4** — Requisição sem token válido a endpoint protegido retorna **401**.
- **RN01.5** — Requisição com token válido mas role insuficiente retorna **403**.
- **RN01.6** — Endpoints de gestão (usuários e campanhas) exigem role `GestorONG`.
- **RN01.7** — O endpoint de doação exige usuário autenticado com role `Doador`.
- **RN01.8** — Usuário `Inativo` (ver RF02) não consegue autenticar.
- **RN01.9** — Além do access token, o login emite um **refresh token** com validade de **7 dias**, armazenado apenas como hash e **rotativo** (cada uso emite novo par e revoga o anterior). Detalhe em [[PRD-01 - Autenticação & Autorização]].
- **RN01.10** — Refresh token expirado ou revogado não renova a sessão (**401**).
- **RN01.11** — Inativar um usuário revoga todos os seus refresh tokens ativos.
- **RN01.12** — Reuso de um refresh token já rotacionado revoga toda a cadeia do usuário (proteção contra roubo).

**Endpoints (contrato resumido):**

| Método | Rota | Auth | Descrição |
|--------|------|------|-----------|
| POST | `/auth/login` | público | Valida credenciais e retorna access + refresh token |
| POST | `/auth/refresh` | refresh válido | Renova o par de tokens (refresh rotativo) |
| POST | `/auth/logout` | autenticado | Revoga o refresh token informado |

**Critérios de aceite (Gherkin):**

```gherkin
Cenário: Login com credenciais válidas
  Dado um usuário ativo com email e senha corretos
  Quando ele faz POST em /auth/login
  Então recebe 200 e um token JWT com a claim "role"

Cenário: Acesso negado por role insuficiente
  Dado um usuário autenticado com role "Doador"
  Quando ele faz POST em /campanhas
  Então recebe 403 Forbidden
```

**Dependências:** depende de RF02 / RF03 (usuários precisam existir). É pré-requisito de RF02, RF04 e RF06.

---

## RF02 — Gerenciamento de Usuários

- **Ator:** `GestorONG` (administração); `Doador` (apenas os próprios dados)
- **Descrição:** Existe um `GestorONG` principal criado automaticamente (seed) no provisionamento do sistema. A partir dele, `GestorONG` pode criar e manter outros usuários — tanto `GestorONG` quanto `Doador`. Um `Doador` só visualiza/edita as próprias configurações.

**Regras de negócio:**
- **RN02.1** — Gestão de **`Doador`** (criar, editar, inativar) pode ser feita por **qualquer `GestorONG`**. Gestão de `GestorONG` e troca de role são exclusivas do **Owner** (ver RN02.8–RN02.10).
- **RN02.2** — Existe um `GestorONG` inicial criado por seed no provisionamento — ele é o **Owner**.
- **RN02.3** — A exclusão de qualquer usuário é **lógica (soft-delete)**: o usuário fica `Inativo`, nunca é removido fisicamente.
- **RN02.4** — A senha só pode ser **redefinida pelo próprio usuário**, independentemente da role (um `GestorONG` não redefine a senha de terceiros).
- **RN02.5** — Um `Doador` só acessa e edita os próprios dados/configurações; não enxerga dados de outros usuários.
- **RN02.6** — Senha segue a **política de senha** (ver Convenções Gerais).
- **RN02.7** — Sempre há **≥ 1 `GestorONG` ativo** — garantido pelo Owner, que nunca é inativado (ver RN02.8).
- **RN02.8** — O `GestorONG` do seed é o **Owner** (`isOwner = true`), é **único** e **nunca pode ser rebaixado nem inativado**. Credenciais do Owner vêm de secret/config (Key Vault). Detalhe em [[PRD-02 - Gerenciamento de Usuários]].
- **RN02.9** — **Somente o Owner** pode: criar `GestorONG`, promover `Doador`→`GestorONG`, rebaixar `GestorONG`→`Doador` e inativar `GestorONG`.
- **RN02.10** — Um `GestorONG` comum não altera roles nem cria/inativa outros `GestorONG`.
- **RN02.11** — A **reativação** de usuário inativo segue a mesma matriz da inativação (`Doador` por qualquer `GestorONG`; `GestorONG` pelo Owner). É o caminho de reativação citado em RF03 (RN03.9).

**Endpoints (contrato resumido):**

| Método | Rota | Auth | Descrição |
|--------|------|------|-----------|
| POST | `/usuarios` | `GestorONG` (Doador) · `Owner` (GestorONG) | Cria usuário |
| PUT | `/usuarios/{id}` | `GestorONG` (Doador) · `Owner` (GestorONG/role) | Edita usuário |
| PATCH | `/usuarios/{id}/inativar` | `GestorONG` (Doador) · `Owner` (GestorONG) | Inativa (soft-delete) |
| PATCH | `/usuarios/{id}/reativar` | `GestorONG` (Doador) · `Owner` (GestorONG) | Reativa usuário inativo |
| GET | `/usuarios` | `GestorONG` | Lista usuários |
| GET | `/usuarios/me` | autenticado | Dados/config do próprio usuário |
| PUT | `/usuarios/me` | autenticado | Edita o próprio perfil (nome/razão social) |
| POST | `/usuarios/me/redefinir-senha` | autenticado | Redefine a própria senha |

**Critérios de aceite (Gherkin):**

```gherkin
Cenário: Doador não gerencia usuários
  Dado um usuário autenticado com role "Doador"
  Quando ele faz POST em /usuarios
  Então recebe 403 Forbidden

Cenário: Exclusão é lógica
  Dado um GestorONG autenticado
  Quando ele inativa um Doador
  Então o usuário fica com status Inativo e não é removido do banco
  E esse usuário não consegue mais autenticar

Cenário: Reset de senha apenas pelo próprio usuário
  Dado um GestorONG tentando redefinir a senha de outro usuário
  Então a operação é negada

Cenário: Reativação de Doador inativo
  Dado um Doador Inativo
  Quando um GestorONG o reativa
  Então o Doador volta a Ativo e consegue autenticar novamente
```

**Dependências:** RF01 (autorização). Relaciona-se com RF03 (auto-cadastro de `Doador`).

---

## RF03 — Cadastro de Doador (auto-cadastro público — PF ou PJ)

- **Ator:** Público (visitante anônimo)
- **Descrição:** Qualquer visitante pode se autocadastrar como doador, como **pessoa física (PF)** ou **pessoa jurídica (PJ)**, informando: **TipoPessoa** (`PF`/`PJ`), **Documento** (CPF para PF, CNPJ para PJ), **Nome Completo** (PF) ou **Razão Social** (PJ), **Email** (único) e **Senha**. O usuário criado recebe a role `Doador`. (A criação de `GestorONG` **não** ocorre por aqui — ver RF02.)

**Regras de negócio:**
- **RN03.1** — O `Email` deve ser único; cadastro com email já existente é rejeitado e a tela informa que já há um cadastro.
- **RN03.2** — O `TipoPessoa` define o documento exigido e o rótulo do nome: `PF` → `CPF` + `Nome Completo`; `PJ` → `CNPJ` + `Razão Social`.
- **RN03.3** — O `Documento` deve ser válido conforme o tipo: **CPF** (11 dígitos) ou **CNPJ** (14 dígitos), ambos com validação dos **dígitos verificadores** (não apenas máscara).
- **RN03.4** — O `Documento` deve ser **único** no banco (além do `Email`).
- **RN03.5** — A senha é armazenada **somente** como hash (BCrypt), nunca em texto puro.
- **RN03.6** — Todo usuário criado por este fluxo recebe a role `Doador`.
- **RN03.7** — Todos os campos são obrigatórios; senha segue a **política de senha** (ver Convenções Gerais).
- **RN03.8** — O cadastro **cria a conta (201) mas não autentica**; o doador deve fazer login em seguida (`POST /auth/login`). Detalhe em [[PRD-03 - Cadastro de Doador]].
- **RN03.9** — A unicidade considera **também usuários `Inativo`**: identificador de conta `Ativa` → **409** (já cadastrado); de conta `Inativa` → não cria e orienta a **contatar um administrador para reativar** (ver RF02/RN02.11). Não há auto-reativação pelo cadastro.

**Endpoints (contrato resumido):**

| Método | Rota | Auth | Descrição |
|--------|------|------|-----------|
| POST | `/auth/register` | público | Cria doador (201, sem token) |

**Critérios de aceite (Gherkin):**

```gherkin
Cenário: Cadastro PJ com CNPJ válido
  Dado um visitante com TipoPessoa "PJ"
  Quando ele se cadastra informando CNPJ válido e Razão Social
  Então o doador é criado com a role Doador

Cenário: Email ou Documento de conta ativa
  Dado que já existe usuário Ativo com o mesmo Email ou Documento
  Quando alguém tenta se cadastrar com ele
  Então recebe 409 informando que já há cadastro e nenhum usuário é criado

Cenário: Email ou Documento de conta inativa
  Dado que existe usuário Inativo com o mesmo Email ou Documento
  Quando alguém tenta se cadastrar com ele
  Então recebe 409 orientando a contatar um administrador para reativar e nenhum usuário é criado

Cenário: Documento inválido
  Quando alguém se cadastra com CPF/CNPJ de dígitos inválidos
  Então recebe 400 Bad Request
```

**Dependências:** nenhuma. Habilita RF01 (login) e RF06 (doação).

---

## RF04 — Gestão de Campanhas

- **Ator:** `GestorONG`
- **Descrição:** O gestor cadastra e mantém campanhas de arrecadação. Uma campanha contém obrigatoriamente: **Título** (string), **Descrição** (string), **DataInicio** (datetime), **DataFim** (datetime), **MetaFinanceira** (decimal) e **Status** (`Ativa`, `Concluida`, `Cancelada`). O **ValorArrecadado** é derivado das doações e **não** é editável pelo gestor (mantido pelo Worker — ver RF06).

**Regras de negócio:**
- **RN04.1** — Apenas `GestorONG` pode criar, editar ou alterar o status de campanhas.
- **RN04.2** — A `DataFim` não pode estar no passado no momento da criação.
- **RN04.3** — A `MetaFinanceira` deve ser maior que zero.
- **RN04.4** — `Status` só admite `Ativa`, `Concluida`, `Cancelada`.
- **RN04.5** — Na criação, o status inicial é `Ativa`.
- **RN04.6** — `DataFim` deve ser maior ou igual à `DataInicio`.
- **RN04.7** — O `ValorArrecadado` é somente-leitura nesta capacidade (escrito apenas pelo consumer da DonationAPI — RF06).
- **RN04.8** — Uma campanha `Concluida` ou `Cancelada` não pode voltar para `Ativa`.
- **RN04.9** — Uma campanha `Ativa` é concluída por três vias, registradas em `MotivoConclusao`: **MetaAtingida** (auto, `ValorArrecadado ≥ Meta`, no consumer), **EncerradaManualmente** (GestorONG encerra fora da meta) e **Expirada** (auto, `DataFim` vencida sem atingir a meta). Detalhe em [[PRD-04 - Gestão de Campanhas]].
- **RN04.10** — A conclusão por **data** é feita por um BackgroundService periódico na DonationAPI; a conclusão por **meta** ocorre no consumer (idempotente).
- **RN04.11** — Toda transição de status atualiza também o read model (Cosmos) para o Painel (RF05) parar de listar a campanha encerrada.

**Endpoints (contrato resumido):**

| Método | Rota | Auth | Descrição |
|--------|------|------|-----------|
| POST | `/campanhas` | `GestorONG` | Cria campanha |
| PUT | `/campanhas/{id}` | `GestorONG` | Edita dados da campanha |
| PATCH | `/campanhas/{id}/status` | `GestorONG` | Altera o status |
| GET | `/campanhas` | `GestorONG` | Lista todas (gestão) |
| GET | `/campanhas/{id}` | `GestorONG` | Detalha uma campanha |

**Critérios de aceite (Gherkin):**

```gherkin
Cenário: Bloquear meta inválida
  Dado um GestorONG autenticado
  Quando ele cria uma campanha com MetaFinanceira igual a 0
  Então recebe 400 e a campanha não é criada

Cenário: Bloquear data de término no passado
  Dado um GestorONG autenticado
  Quando ele cria uma campanha com DataFim anterior a hoje
  Então recebe 400 e a campanha não é criada
```

**Dependências:** RF01 (autorização por role). Alimenta RF05 e RF06.

---

## RF05 — Painel de Transparência

- **Ator:** Público
- **Descrição:** API pública que lista **apenas** campanhas com status `Ativa`, exibindo **Título**, **Meta Financeira** e **Valor Total Arrecadado** (calculado com base nas doações já processadas pelo Worker). Não exige autenticação.

**Regras de negócio:**
- **RN05.1** — Somente campanhas com status `Ativa` aparecem na listagem.
- **RN05.2** — O `Valor Total Arrecadado` é a soma das doações **processadas** (consolidadas pelo Worker), não das intenções recebidas.
- **RN05.3** — Endpoint público — não requer token.
- **RN05.4** — O valor reflete **consistência eventual**: representa o último processamento concluído pelo Worker, podendo estar atrás de uma doação recém-enviada.
- **RN05.5** — Campos sensíveis (dados de doadores, descrição interna) não são expostos.
- **RN05.6** — A leitura é servida **somente do read model (Cosmos DB)**, não do SQL Server. Detalhe em [[PRD-05 - Painel de Transparência]].

**Endpoints (contrato resumido):**

| Método | Rota | Auth | Descrição |
|--------|------|------|-----------|
| GET | `/transparencia/campanhas` | público | Lista campanhas ativas + arrecadado |

**Critérios de aceite (Gherkin):**

```gherkin
Cenário: Listar apenas campanhas ativas
  Dado que existem campanhas Ativa, Concluida e Cancelada
  Quando o público consulta GET /transparencia/campanhas
  Então apenas as campanhas Ativa são retornadas
  E cada item exibe Título, Meta Financeira e Valor Arrecadado
```

**Dependências:** RF04 (campanhas) e RF06 (valor consolidado pelo Worker).

---

## RF06 — Processo de Doação

- **Ator:** `Doador` logado
- **Descrição:** O doador autenticado envia uma **intenção de doação** com `IdCampanha`, `ValorDoacao` e `FormaPagamento` (PIX, cartão ou transferência) à **DonationAPI**. O fluxo é uma **saga de 2 eventos** (ver [[Domain Events]]): a DonationAPI grava a `Doacao` como `Pendente` e publica `DoacaoSolicitadaEvent`; a **PaymentAPI** consome esse evento, processa o pagamento em um **gateway simulado (mock)** e publica `PagamentoAprovadoEvent` ou `PagamentoRecusadoEvent`; a DonationAPI consome o resultado em um **consumer/BackgroundService** e, **somente se aprovado**, consolida o `ValorArrecadado` da campanha. O endpoint de doação **nunca** escreve o `ValorArrecadado` de forma síncrona.

**Regras de negócio:**
- **RN06.1** — Exige usuário autenticado com role `Doador`.
- **RN06.2** — A campanha informada deve existir.
- **RN06.3** — A doação só é permitida em campanha `Ativa` **e dentro do período** (`DataInicio ≤ agora ≤ DataFim`); campanhas `Concluida`/`Cancelada` ou fora do período são rejeitadas (422).
- **RN06.4** — O `ValorDoacao` deve ser maior que zero.
- **RN06.5** — A `FormaPagamento` é obrigatória e deve ser uma das: `PIX`, `Cartão`, `Transferência`.
- **RN06.6** — A consolidação do `ValorArrecadado` só ocorre **após** a PaymentAPI publicar `PagamentoAprovadoEvent`.
- **RN06.7** — O endpoint `POST /doacoes` **não** escreve o `ValorArrecadado` de forma síncrona; a consolidação é feita apenas pelo consumer/BackgroundService da DonationAPI (requisito arquitetural do enunciado).
- **RN06.8** — A API responde de forma assíncrona (`202 Accepted`): a intenção é aceita; o resultado do pagamento e a consolidação ocorrem depois.
- **RN06.9** — Pagamento **recusado** gera `PagamentoRecusadoEvent`; a doação fica como `Recusada` e não é contabilizada no `ValorArrecadado`; o doador é notificado do resultado.
- **RN06.10** — O consumer da DonationAPI é **idempotente**: a mesma doação (`doacaoId`) não pode ser somada duas vezes em reentrega da mensagem (dedup via tabela de eventos processados).
- **RN06.11** — O gateway é **simulado (mock)**: aprova por padrão e **recusa valores cujos centavos sejam `,99`** (marcador de teste determinístico, ajustável). Detalhe em [[PRD-06 - Processo de Doação]].
- **RN06.12** — A `Doacao` tem status `Pendente` (criação) → `Aprovada` ou `Recusada` (após o pagamento).
- **RN06.13** — O doador acompanha o resultado por consulta (`GET /doacoes/{id}`) **e** recebe **notificação assíncrona** do resultado, disparada por uma Azure Function (`NotificationFunction`) que consome os eventos de pagamento — canal **mock/log** no MVP (envio real de email/push fora de escopo). Ver **RF07** / [[PRD-07 - Notificações]].
- **RN06.14** — Ao consolidar uma doação aprovada, o consumer também projeta a campanha no Cosmos (RN04.11) e, se `ValorArrecadado ≥ Meta`, conclui a campanha como `MetaAtingida` (RN04.09).

**Endpoints (contrato resumido):**

| Método | Rota            | Auth     | Descrição                                      |
| ------ | --------------- | -------- | ---------------------------------------------- |
| POST   | `/doacoes`      | `Doador` | Registra intenção de doação (inicia pagamento) |
| GET    | `/doacoes/{id}` | `Doador` | Consulta status da própria doação              |

**Critérios de aceite (Gherkin):**

```gherkin
Cenário: Doação aprovada em campanha ativa
  Dado um Doador autenticado e uma campanha Ativa
  Quando ele faz POST em /doacoes com IdCampanha, ValorDoacao e FormaPagamento válidos
  Então a API responde 202 Accepted
  E a DonationAPI publica DoacaoSolicitadaEvent no broker
  E a PaymentAPI processa o pagamento (mock) e publica PagamentoAprovadoEvent

Cenário: Pagamento recusado
  Dado um Doador autenticado e uma campanha Ativa
  Quando ele faz POST em /doacoes e o pagamento (mock) é recusado
  Então a PaymentAPI publica PagamentoRecusadoEvent
  E a doação fica como Recusada e não é contabilizada

Cenário: Bloquear doação em campanha encerrada
  Dado uma campanha com status Concluida
  Quando um Doador tenta doar para ela
  Então recebe 422 e nenhum evento é publicado

Cenário: Consumer consolida o valor (idempotente)
  Dado um PagamentoAprovadoEvent na fila
  Quando o consumer da DonationAPI o consome
  Então o ValorArrecadado da campanha é incrementado pelo ValorDoacao
  E reprocessar o mesmo evento não soma novamente
```

**Dependências:** RF01 (auth), RF04 (campanha/status). Integra com a **PaymentAPI** (gateway de pagamento **simulado/mock**), mensageria (Azure ServiceBus) e o **consumer da DonationAPI**. Alimenta RF05. Os eventos de resultado também disparam a notificação (RF07).

---

## RF07 — Notificação ao Doador

- **Ator:** `Doador` (destinatário); `NotificationFunction` (serviço)
- **Descrição:** Ao final do processamento do pagamento, o doador recebe uma **notificação assíncrona** informando se a doação foi **aprovada** ou **recusada**. A notificação é disparada por uma **Azure Function** dedicada (`NotificationFunction`, gerenciada, **fora do AKS**), acionada pelos eventos `PagamentoAprovadoEvent`/`PagamentoRecusadoEvent` no **Azure ServiceBus** e consumindo-os por uma **subscription própria** — independente do consumer de arrecadação da DonationAPI. No MVP o canal é **mock/log** (registro estruturado + Application Insights), **sem** provedor externo de email/SMS. Detalhe em [[PRD-07 - Notificações]].

**Regras de negócio:**
- **RN07.1** — A notificação é disparada **somente** após `PagamentoAprovadoEvent` ou `PagamentoRecusadoEvent`.
- **RN07.2** — Canal **mock/log** no MVP; sem envio real de email/SMS.
- **RN07.3** — Consumidor **independente** (subscription própria); falha na notificação **não** afeta a consolidação do `ValorArrecadado`.
- **RN07.4** — A Function **não** escreve em banco de outro serviço nem altera status/`ValorArrecadado`.
- **RN07.5** — O destinatário (`doadorEmail`/`doadorNome`) vem **enriquecido no próprio evento**; sem chamada síncrona à UserAPI.
- **RN07.6** — Entrega *at-least-once*: reprocessamento pode gerar log de notificação duplicado; após o limite a mensagem vai para a **DLQ**.

**Endpoints (contrato resumido):** nenhum — worker orientado a evento (trigger `ServiceBusTrigger`), sem API HTTP.

**Critérios de aceite (Gherkin):**

```gherkin
Cenário: Notificação de pagamento aprovado
  Dado um PagamentoAprovadoEvent na subscription da NotificationFunction
  Quando a Function o consome
  Então registra uma notificação de "doação aprovada" para o doadorEmail do evento

Cenário: Notificação de pagamento recusado
  Dado um PagamentoRecusadoEvent na subscription da NotificationFunction
  Quando a Function o consome
  Então registra uma notificação de "pagamento recusado" para o doadorEmail do evento
```

**Dependências:** RF06 (eventos de resultado do pagamento) e RF01 (claims de identidade que alimentam o enriquecimento dos eventos). Não alimenta outras capacidades.
