---
tags: [hackaton-fiap, spec, design, userapi]
projeto: Conexão Solidária
servico: HackatonFiap.Users (UserAPI)
status: aprovado
atualizado: 2026-06-16
prds: [PRD-01, PRD-02, PRD-03]
---

# Design — UserAPI (HackatonFiap.Users)

> Spec de design do microsserviço de **Identidade & Acesso** do Conexão Solidária. Cobre [[PRD-01 - Autenticação & Autorização|PRD-01]], [[PRD-02 - Gerenciamento de Usuários|PRD-02]] e [[PRD-03 - Cadastro de Doador|PRD-03]]. Construído **estendendo o boilerplate `FGC.Users`** (origem "Fiap Cloud Games") e renomeando para `HackatonFiap.Users`.

## 1. Decisões fechadas (brainstorming)

| Tema | Decisão |
|------|---------|
| Escopo | UserAPI **completa**: PRD-01 + PRD-02 + PRD-03 de uma vez |
| Namespace | **`HackatonFiap.Users`** (`.Domain` / `.Application` / `.Infrastructure` / `.API` / `.UnitTests`) — vale também para `.Donations` / `.Payments` |
| Padrão CQRS | **Handlers manuais** (sem MediatR): Command/Query como records + Handler com `Result<T>`, injetados direto no controller |
| Stack | **.NET 8** (boilerplate já está em `net8.0`), EF Core 8, **xUnit + NSubstitute + FluentAssertions** |
| Idioma do código | **Inglês**, exceto os **valores das roles** que ficam em pt-BR (`Doador`, `GestorONG`) |
| Rotas HTTP | **Inglês com prefixo `/api`** (`/api/auth/login`, `/api/users/me`, ...) |
| ORM | **EF Core** (SQL Server, database-per-service) |
| Eventos cross-service | **Nenhum** no MVP (PRD-01/02/03 §9) — remove publicação `UserRegistered`/ServiceBus da UserAPI |
| Observabilidade | **OpenTelemetry** (traces + métricas) com exporter **Prometheus** em `/metrics`, sobre o Serilog existente |

## 2. Arquitetura & esqueleto

Clean Architecture, 4 projetos em `src/` + 1 em `tests/`, herdada do boilerplate:

- **`HackatonFiap.Users.Domain`** — entidades, VOs, enums, eventos de domínio, `Result`/`Result<T>`/`Error`. Sem dependências externas.
- **`HackatonFiap.Users.Application`** — Commands/Queries + Handlers + Validators (FluentValidation) + Interfaces + DTOs + Errors. Handlers retornam `Result<T>`.
- **`HackatonFiap.Users.Infrastructure`** — EF Core (`ApplicationDbContext`, configurations, repositories, migrations), `BcryptPasswordHasher`, `JwtTokenGenerator`, `AuditService`.
- **`HackatonFiap.Users.API`** — Controllers, middlewares (Correlation, RequestResponseLogging), wiring de DI, observabilidade, autenticação JWT.
- **`HackatonFiap.Users.UnitTests`** — xUnit + NSubstitute + FluentAssertions.

### Rename `FGC.Users` → `HackatonFiap.Users`
Mecânico, atinge: `.sln`, 5 `.csproj` + pastas, namespaces/usings, `docker-compose.yml`, `Dockerfile`, `k8s/`, `launchSettings.json`, `CLAUDE.md`. A pasta do repositório (`hackaton-fiap-users`) permanece.

## 3. Modelo de domínio

### Entidade `User` (aggregate root)
| Propriedade | Tipo | Notas |
|-------------|------|-------|
| `Id` | `Guid` | |
| `PersonType` | `PersonType` (enum) | `Individual` (PF) / `Company` (PJ) |
| `Document` | `Document` (VO) | CPF (PF) / CNPJ (PJ) com dígitos verificadores; único |
| `Name` | `string` | nome completo (PF) ou razão social (PJ) |
| `Email` | `string` | único; validado por FluentValidation |
| `Password` | `Password` (VO) | hash BCrypt (VO existente) |
| `Role` | `UserRole` (enum) | **`Doador` / `GestorONG`** (persistido como string) |
| `IsActive` | `bool` | soft-delete; `false` = Inativo |
| `IsOwner` | `bool` | exatamente um Owner no sistema |
| `CreatedAtUtc` / `UpdatedAtUtc` | `DateTime` | |

**Fábricas / métodos:** `RegisterDonor(personType, document, name, email, password)` (role `Doador`, ativo), `CreateByAdmin(...)` (role definida pelo chamador), `UpdateProfile(name)`, `ChangeRole(role)`, `Deactivate()`, `Reactivate()`.

**Invariantes:** email único; documento único e válido conforme o tipo; senha sempre em hash; exatamente um Owner sempre `Ativo` e `GestorONG`; usuário inativo não autentica.

### Entidade `RefreshToken`
| Propriedade | Tipo | Notas |
|-------------|------|-------|
| `Id` | `Guid` | |
| `UserId` | `Guid` | FK → `User` |
| `TokenHash` | `string` | SHA-256 do token; o token cru só é devolvido na resposta |
| `ExpiresAtUtc` | `DateTime` | +7 dias |
| `RevokedAtUtc` | `DateTime?` | |
| `ReplacedByTokenId` | `Guid?` | cadeia de rotação → detecção de reuso |
| `CreatedAtUtc` | `DateTime` | |

Helper `IsActive` = `RevokedAtUtc is null && ExpiresAtUtc > now`.

### Value Objects / enums
- **`Document`** (novo VO) — normaliza para dígitos, valida **dígitos verificadores** de CPF (11) / CNPJ (14) conforme `PersonType`. Não apenas máscara.
- **`PersonType`** (enum) — `Individual` / `Company`.
- **`UserRole`** (enum) — `Doador` / `GestorONG` (valores em pt-BR).
- **`Password`** (VO existente) — política: ≥8 caracteres, ≥1 letra, ≥1 dígito, ≥1 especial.

## 4. Camada de Application

### Commands
| Comando | PRD | Regras-chave |
|---------|-----|--------------|
| `RegisterDonorCommand` | 03 | role `Doador`; **201 sem token**; unicidade email+documento **incluindo inativos**: ativo → 409 "já cadastrado"; inativo → 409 "contate um administrador" |
| `AuthenticateUserCommand` (estende) | 01 | valida hash BCrypt + `IsActive`; emite **access (4h)** + **refresh (7d)** |
| `RefreshTokenCommand` | 01 | refresh válido → rota (revoga atual, emite novo par); **reuso de token rotacionado revoga toda a cadeia** (RN01.12) |
| `LogoutCommand` | 01 | revoga o refresh informado (204) |
| `CreateUserCommand` | 02 | GestorONG cria `Doador`; **Owner** cria `GestorONG` |
| `UpdateUserCommand` | 02 | edita dados; troca de role apenas via `ChangeUserRoleCommand` |
| `ChangeUserRoleCommand` | 02 | **somente Owner**: promove Doador→GestorONG / rebaixa GestorONG→Doador |
| `DeactivateUserCommand` | 02 | soft-delete + **revoga todos os refresh tokens** (RN01.11); Owner **nunca** |
| `ReactivateUserCommand` | 02 | Doador por qualquer GestorONG; GestorONG pelo Owner |
| `UpdateMyProfileCommand` | 02 | self: só `Name`; email/documento **imutáveis** |
| `ResetMyPasswordCommand` | 02 | self: **senha atual + nova** |

### Queries
| Query | PRD | Notas |
|-------|-----|-------|
| `GetProfileQuery` (existe) | 02 | `/api/users/me` |
| `GetUserByIdQuery` | 02 | detalhe (GestorONG) |
| `ListUsersQuery` | 02 | lista (GestorONG) |

### Interfaces
- `IUserRepository` — estende: `FindByEmailAsync` / `FindByDocumentAsync` **bypassando o query filter** (enxergam inativos), `GetByIdAsync`, `ListAsync`, `UpdateAsync`, `SaveNewAsync`.
- `IRefreshTokenRepository` (novo) — `AddAsync`, `FindByHashAsync`, `RevokeAsync`, `RevokeAllForUserAsync`, rotação.
- `IJwtTokenGenerator` (existe) — claims `sub`/userId, `email`, `role`, `isOwner`; access 4h.
- `IPasswordHasher`, `IAuditService` (existentes).

## 5. Superfície da API (inglês + `/api`)

| Método | Rota | Auth | Request | Response |
|--------|------|------|---------|----------|
| POST | `/api/auth/register` | público | `{ personType, document, name, email, password }` | `201` / `400` / `409` |
| POST | `/api/auth/login` | público | `{ email, password }` | `200 { accessToken, refreshToken, expiresIn }` / `401` |
| POST | `/api/auth/refresh` | público | `{ refreshToken }` | `200 { accessToken, refreshToken, expiresIn }` / `401` |
| POST | `/api/auth/logout` | autenticado | `{ refreshToken }` | `204` |
| POST | `/api/users` | `GestorONG`/Owner | `{ personType, document, name, email, password, role }` | `201` / `400` / `403` / `409` |
| PUT | `/api/users/{id}` | `GestorONG`/Owner | `{ name }` (+ `role` só Owner) | `200` / `403` / `404` |
| PATCH | `/api/users/{id}/deactivate` | matriz | — | `204` / `403` / `404` |
| PATCH | `/api/users/{id}/reactivate` | matriz | — | `204` / `403` / `404` |
| GET | `/api/users` | `GestorONG` | — | `200 [ ... ]` |
| GET | `/api/users/me` | autenticado | — | `200 { ... }` |
| PUT | `/api/users/me` | autenticado | `{ name }` | `200` |
| POST | `/api/users/me/reset-password` | autenticado | `{ currentPassword, newPassword }` | `204` / `400` |
| GET | `/health` · `/ready` · `/metrics` | público | — | probes / métricas Prometheus |

Erros via `ProblemDetails`; `401` sem token, `403` role/owner insuficiente.

## 6. Autorização & matriz de permissões (PRD-02)

- `[Authorize(Roles="GestorONG")]` nos endpoints de gestão; `[Authorize]` nos `me`.
- Checagens **data-driven** nos handlers (não dá para expressar só por role): **Owner-only** (criar/promover/rebaixar/inativar GestorONG), **Owner intocável**, **GestorONG comum não mexe em GestorONG**, **Doador só os próprios dados**. Violação → `Result.Failure` mapeado para **403**.
- JWT carrega `isOwner` (evita hit no banco); o handler **revalida** contra o estado atual.
- **Seed do Owner**: lido de config/Key Vault (`Owner:Email` / `Owner:Password`), `isOwner=true`, role `GestorONG`, ativo; **nunca** rebaixado/inativado. Em dev, via `appsettings`/env.

## 7. Fluxo de Auth & Refresh Token

```
login → valida hash + IsActive → access(4h) + refresh(7d, hash persistido)
refresh → acha por hash → se revogado/rotacionado: REVOGA CADEIA + 401 (reuso)
        → se válido: revoga atual, encadeia ReplacedByTokenId, emite novo par
logout → revoga o refresh
deactivate(user) → RevokeAllForUserAsync (RN01.11)
```

## 8. Persistência & migrations (EF Core / SQL Server)

- Estende `UserConfiguration`: `Document` como VO owned com **índice único**, `PersonType`/`Role` como string, `IsOwner`, `Name`/`Email` (índice único). Mantém `Password` owned (`PasswordHash`) e o **global query filter** `IsActive` — repos de unicidade/reativação usam `IgnoreQueryFilters()`.
- Nova `RefreshTokenConfiguration` + `DbSet<RefreshToken>`.
- **Migration `InitialCreate` reescrita** (hackathon, sem dados legados). `EnableRetryOnFailure` + `Database.Migrate()` no startup mantidos.

## 9. Observabilidade — OpenTelemetry (obrigatório)

> Requisito do enunciado (RNF) e reforçado pelo time. Sobre o Serilog já existente.

- **Pacotes:** `OpenTelemetry.Extensions.Hosting`, `OpenTelemetry.Instrumentation.AspNetCore`, `.Http`, `.EntityFrameworkCore`, `.Runtime`, `OpenTelemetry.Exporter.Prometheus.AspNetCore` (+ `OpenTelemetry.Exporter.OpenTelemetryProtocol` opcional para OTLP).
- **Tracing:** instrumentação AspNetCore + HttpClient + EF Core; `ResourceBuilder` com `service.name = HackatonFiap.Users`; propagação de `traceId`/`correlationId`.
- **Métricas:** instrumentação AspNetCore + Runtime + métricas custom (ex.: contadores de login/refresh/registro); **exporter Prometheus** exposto em **`/metrics`** (`MapPrometheusScrapingEndpoint`).
- **Correlação de logs:** enriquecer Serilog com `traceId`/`spanId` para casar logs ↔ traces; `correlationId` já propagado pelo `CorrelationMiddleware`.
- **Health:** `/health` (liveness) e `/ready` (readiness) já existentes.

## 10. Testes (xUnit + NSubstitute + FluentAssertions)

- Trocar **Moq → NSubstitute** no `.csproj` e reescrever a sintaxe dos testes existentes.
- Novos testes: `Document` (CPF/CNPJ válidos e inválidos), `RegisterDonor` (unicidade ativo→409 / inativo→409 orienta admin), `Authenticate` (inativo, senha errada, sucesso emite par), `RefreshToken` (rotação + reuso revoga cadeia), `Deactivate` (revoga refresh tokens), matriz de permissões (Owner-only, Owner intocável, GestorONG×GestorONG, Doador self-only), `ResetMyPassword` (senha atual obrigatória, só self), seed do Owner.

## 11. Defaults assumidos
1. `Email` como `string` + FluentValidation + índice único (não VO).
2. `Status` = `IsActive` (bool), reaproveitando soft-delete/query filter.
3. `reset-password` exige senha atual + nova.
4. `PersonType` = `Individual`/`Company` (CPF/CNPJ permanecem como nomes próprios no `Document`).

## 12. Fora de escopo (MVP)
Recuperação de senha por email, MFA/login social, verificação de email no cadastro, múltiplos Owners/transferência de Owner, edição de email/documento, importação em massa, consentimento LGPD.

## 13. Fases de implementação (visão macro)
1. Rename `FGC.Users` → `HackatonFiap.Users` (build verde).
2. Domínio: `PersonType`, `Document` (VO + validação), `UserRole` (Doador/GestorONG), `User` (campos novos + métodos), `RefreshToken`.
3. PRD-03 — RegisterDonor (handler + validator + rota + unicidade).
4. PRD-01 — Authenticate (access+refresh) + Refresh (rotação/reuso) + Logout.
5. PRD-02 — gestão de usuários + matriz de permissões + Owner seed.
6. Observabilidade OpenTelemetry + `/metrics`.
7. Persistência: configurations + migration `InitialCreate`.
8. Testes (NSubstitute) + ajustes de CI.
9. Atualizar `CLAUDE.md`, `README.md`, docker/k8s e o doc de [[Requisitos Técnicos]] (test lib NSubstitute).

**Relacionados:** [[PRD-01 - Autenticação & Autorização]] · [[PRD-02 - Gerenciamento de Usuários]] · [[PRD-03 - Cadastro de Doador]] · [[Requisitos Técnicos]] · [[Visão Geral de Arquitetura]] · [[Bounded Contexts]]
