---
tags: [hackaton-fiap, api, userapi]
projeto: Conexão Solidária
servico: HackatonFiap.Users (UserAPI)
status: implementado
atualizado: 2026-06-16
prds: [PRD-01, PRD-02, PRD-03]
---

# API — UserAPI (HackatonFiap.Users)

> Serviço de **Identidade & Acesso** do Conexão Solidária. Cadastro de doador, autenticação JWT (access + refresh rotativo) e gestão de usuários com hierarquia Owner/GestorONG/Doador. Implementa [[PRD-01 - Autenticação & Autorização|PRD-01]], [[PRD-02 - Gerenciamento de Usuários|PRD-02]] e [[PRD-03 - Cadastro de Doador|PRD-03]]. Design em [[2026-06-16-userapi-design]].

## Stack
- **.NET 8**, Clean Architecture (`Domain`/`Application`/`Infrastructure`/`API`), **CQRS** (handlers manuais + `Result<T>`).
- **EF Core 8 + SQL Server** (escrita; database-per-service). Migrations aplicadas no startup.
- **JWT Bearer** (access 4h) + **refresh token** rotativo 7d (hash SHA-256, detecção de reuso).
- **BCrypt** (hash de senha), **Serilog**, **OpenTelemetry** (traces + métricas → `/metrics` Prometheus).
- Testes: **xUnit + NSubstitute + FluentAssertions** (57 testes).

## Autenticação & RBAC
- Access token (JWT) carrega `sub` (userId), `email`, `role` (`Doador`/`GestorONG`) e `isOwner`. Expira em **4h**.
- Refresh token: valor aleatório devolvido **uma vez**; só o **hash** é persistido. Expira em **7d**, é **rotativo** (cada uso revoga o anterior e emite novo par). Reuso de token rotacionado **revoga toda a cadeia** (RN01.12).
- `401` sem token válido; `403` role/owner insuficiente. Usuário **inativo não autentica** (RN01.8).

## Endpoints

### Auth (`/api/auth`)
| Método | Rota | Auth | Request | Sucesso | Erros |
|--------|------|------|---------|---------|-------|
| POST | `/api/auth/register` | público | `{ personType, document, name, email, password }` | `201` (UserResponse, sem token) | `400` (validação/documento), `409` (ativo: já cadastrado / inativo: contatar admin) |
| POST | `/api/auth/login` | público | `{ email, password }` | `200 { accessToken, refreshToken, expiresIn }` | `401` |
| POST | `/api/auth/refresh` | público | `{ refreshToken }` | `200 { accessToken, refreshToken, expiresIn }` | `401` (inválido/expirado/reuso) |
| POST | `/api/auth/logout` | autenticado | `{ refreshToken }` | `204` | — |

### Usuários (`/api/users`)
| Método | Rota | Auth | Request | Sucesso | Erros |
|--------|------|------|---------|---------|-------|
| POST | `/api/users` | `GestorONG` (Doador) · Owner (GestorONG) | `{ personType, document, name, email, password, role }` | `201` | `400`, `403`, `409` |
| PUT | `/api/users/{id}` | `GestorONG` (Doador) · Owner (GestorONG) | `{ name }` | `200` | `403`, `404` |
| PATCH | `/api/users/{id}/role` | **Owner** | `{ role }` | `204` | `403`, `404` |
| PATCH | `/api/users/{id}/deactivate` | matriz | — | `204` | `403`, `404` |
| PATCH | `/api/users/{id}/reactivate` | matriz | — | `204` | `403`, `404` |
| GET | `/api/users` | `GestorONG` | — | `200 [UserResponse]` | — |
| GET | `/api/users/{id}` | `GestorONG` | — | `200 UserResponse` | `404` |
| GET | `/api/users/me` | autenticado | — | `200 UserResponse` | — |
| PUT | `/api/users/me` | autenticado | `{ name }` | `200 UserResponse` | `404` |
| POST | `/api/users/me/reset-password` | autenticado | `{ currentPassword, newPassword }` | `204` | `400` (senha atual errada / nova fraca) |

### Observabilidade
| Método | Rota | Descrição |
|--------|------|-----------|
| GET | `/health` | liveness |
| GET | `/ready` | readiness |
| GET | `/metrics` | métricas Prometheus (runtime, ASP.NET Core + `users_logins_total`, `users_registrations_total`) |

> Os endpoints `/health`, `/ready` e `/metrics` são consumidos tanto pelo **Prometheus** (métricas de aplicação → Grafana) quanto pelo **Zabbix** (infra/host e disponibilidade: *scrape* do `/metrics`, *web scenarios* em `/health`/`/ready`). Ver [[Decisões de Arquitetura (ADRs)|ADR-002]].

## Tipos

**UserResponse**
```json
{ "Id": "guid", "PersonType": "Individual|Company", "Document": "digitos",
  "Name": "...", "Email": "...", "Role": "Doador|GestorONG",
  "IsActive": true, "IsOwner": false, "CreatedAtUtc": "...", "UpdatedAtUtc": "..." }
```
- `personType`: `0` = `Individual` (PF/CPF) · `1` = `Company` (PJ/CNPJ). `role`: `0` = `Doador` · `1` = `GestorONG` (enums por índice no JSON).
- `document`: CPF (11) ou CNPJ (14) com **dígitos verificadores** validados; único (considera inativos no cadastro).

## Matriz de permissões (PRD-02)
| Ação | Doador | GestorONG | Owner |
|------|:------:|:---------:|:-----:|
| Gerir próprios dados / senha | ✅ | ✅ | ✅ |
| Criar/editar/inativar/reativar **Doador** | — | ✅ | ✅ |
| Criar/inativar/reativar **GestorONG** | — | — | ✅ |
| Promover/rebaixar role | — | — | ✅ |
| Rebaixar/inativar o **Owner** | — | — | ❌ nunca |

- O **Owner** é semeado no startup (único, `isOwner=true`, `GestorONG`), credenciais via config/Key Vault.

## Regras de segurança (config)
- **Sem segredos no código/config commitado.** `Jwt:Key` e `Owner:Password`: em **Development** são gerados aleatoriamente no startup (logados); **fora de Development** são **obrigatórios** (fail-fast) via env/Key Vault.
- Caminho de autenticação respeita o soft-delete (não burla o query filter); unicidade de cadastro/admin usa métodos dedicados que enxergam inativos.

## Exemplos (curl)
```bash
# Cadastro de doador (PJ)
curl -X POST http://localhost:8080/api/auth/register -H "Content-Type: application/json" \
  -d '{"personType":1,"document":"11.444.777/0001-61","name":"Empresa LTDA","email":"e@x.com","password":"Senha@123"}'

# Login
curl -X POST http://localhost:8080/api/auth/login -H "Content-Type: application/json" \
  -d '{"email":"e@x.com","password":"Senha@123"}'

# Perfil (com token)
curl http://localhost:8080/api/users/me -H "Authorization: Bearer <accessToken>"
```

**Relacionados:** [[2026-06-16-userapi-design]] · [[PRD-01 - Autenticação & Autorização]] · [[PRD-02 - Gerenciamento de Usuários]] · [[PRD-03 - Cadastro de Doador]] · [[Visão Geral de Arquitetura]] · [[Requisitos Técnicos]]
