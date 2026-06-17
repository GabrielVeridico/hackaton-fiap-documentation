# PaymentAPI (HackatonFiap.Payments) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir a PaymentAPI do Conexão Solidária a partir do boilerplate FGC.Payments, alinhada ao padrão do `hackaton-fiap-users`: event-driven (consome `DonationRequestedEvent`, publica `PaymentApprovedEvent`/`PaymentDeclinedEvent`), com gateway mock determinístico, CQRS manual, EF Core, OpenTelemetry e idempotência.

**Architecture:** Clean Architecture em 4 projetos (`Domain`/`Application`/`Infrastructure`/`API`) + testes. Um `BackgroundService` consome uma subscription do Service Bus e invoca um command handler que cria/persiste o `Payment`, aplica o gateway mock e publica o resultado num tópico. Endpoint REST só de consulta. Idempotência por índice único em `DonationId`.

**Tech Stack:** .NET 8, EF Core 8 (SQL Server), Azure.Messaging.ServiceBus 7.20.x, OpenTelemetry 1.9.x (+ Prometheus exporter), Serilog, xUnit + NSubstitute + FluentAssertions.

**Spec:** `docs/superpowers/specs/2026-06-17-paymentapi-design.md`

---

## File Structure

```
hackaton-fiap-payments/
├── HackatonFiap.Payments.sln
├── global.json                                   # pin .NET 8 SDK
├── src/
│   ├── HackatonFiap.Payments.Domain/
│   │   ├── Common/Result.cs                      # Result / Result<T> / Error
│   │   ├── Enums/PaymentStatus.cs
│   │   ├── Enums/PaymentMethod.cs
│   │   └── Entities/Payment.cs
│   ├── HackatonFiap.Payments.Application/
│   │   ├── Abstractions/IPaymentRepository.cs
│   │   ├── Abstractions/IPaymentGateway.cs
│   │   ├── Abstractions/IEventPublisher.cs
│   │   ├── IntegrationEvents/DonationRequestedEvent.cs
│   │   ├── IntegrationEvents/PaymentApprovedEvent.cs
│   │   ├── IntegrationEvents/PaymentDeclinedEvent.cs
│   │   ├── Errors/PaymentErrors.cs
│   │   ├── Commands/ProcessDonationPayment/ProcessDonationPaymentCommand.cs
│   │   ├── Commands/ProcessDonationPayment/ProcessDonationPaymentCommandHandler.cs
│   │   └── Queries/GetPaymentByDonationId/GetPaymentByDonationIdQuery.cs (+ Handler + PaymentResponse)
│   ├── HackatonFiap.Payments.Infrastructure/
│   │   ├── Persistence/PaymentsDbContext.cs
│   │   ├── Persistence/Configurations/PaymentConfiguration.cs
│   │   ├── Persistence/Repositories/PaymentRepository.cs
│   │   ├── Gateway/MockPaymentGateway.cs
│   │   ├── Messaging/ServiceBusEventPublisher.cs
│   │   ├── Messaging/DonationRequestedConsumer.cs
│   │   └── Migrations/*                           # gerado pelo EF
│   └── HackatonFiap.Payments.API/
│       ├── Observability/Telemetry.cs
│       ├── Controllers/PaymentsController.cs
│       ├── appsettings.json / appsettings.Development.json
│       └── Program.cs
├── tests/
│   └── HackatonFiap.Payments.UnitTests/
│       ├── Domain/PaymentTests.cs
│       ├── Gateway/MockPaymentGatewayTests.cs
│       └── Application/ProcessDonationPaymentCommandHandlerTests.cs
│       └── Application/GetPaymentByDonationIdQueryHandlerTests.cs
├── .github/workflows/ci.yml
└── .github/k8s/deployment.yaml
```

**Convenções:** código em inglês; Allman braces; sempre usar chaves; early returns. `dotnet` commands rodam a partir da raiz do repo `hackaton-fiap-payments`.

---

## Task 1: Scaffolding do repositório a partir do FGC.Payments

**Files:**
- Create: toda a estrutura `src/` e `tests/`, `HackatonFiap.Payments.sln`, `global.json`

- [ ] **Step 1: Clonar o boilerplate para um diretório de trabalho temporário**

```bash
TMP="$TEMP/FGC.Payments-base"; rm -rf "$TMP"
git clone --depth 1 https://github.com/MatheusRoberto-Git/FGC.Payments.git "$TMP"
```

- [ ] **Step 2: Criar a estrutura final em `hackaton-fiap-payments` (src/ + tests/), copiando e renomeando os 4 projetos**

```bash
ROOT="C:/eSolution/Projetos/hackaton-fiap/hackaton-fiap-payments"
mkdir -p "$ROOT/src" "$ROOT/tests"
cp -r "$TMP/FGC.Payments.Domain"         "$ROOT/src/HackatonFiap.Payments.Domain"
cp -r "$TMP/FGC.Payments.Application"    "$ROOT/src/HackatonFiap.Payments.Application"
cp -r "$TMP/FGC.Payments.Infrastructure" "$ROOT/src/HackatonFiap.Payments.Infrastructure"
cp -r "$TMP/FGC.Payments.Presentation"   "$ROOT/src/HackatonFiap.Payments.API"
cp "$TMP/Dockerfile" "$TMP/.dockerignore" "$TMP/.gitignore" "$ROOT/" 2>/dev/null || true
# remover artefatos do boilerplate que serão recriados
find "$ROOT/src" -name "*.csproj" -newer /dev/null  # listagem de conferência
rm -rf "$ROOT"/src/*/bin "$ROOT"/src/*/obj
```

- [ ] **Step 3: Renomear os arquivos `.csproj` e o namespace raiz `FGC.Payments` → `HackatonFiap.Payments` e `Presentation` → `API`**

```bash
cd "$ROOT"
mv src/HackatonFiap.Payments.Domain/FGC.Payments.Domain.csproj                 src/HackatonFiap.Payments.Domain/HackatonFiap.Payments.Domain.csproj
mv src/HackatonFiap.Payments.Application/FGC.Payments.Application.csproj        src/HackatonFiap.Payments.Application/HackatonFiap.Payments.Application.csproj
mv src/HackatonFiap.Payments.Infrastructure/FGC.Payments.Infrastructure.csproj  src/HackatonFiap.Payments.Infrastructure/HackatonFiap.Payments.Infrastructure.csproj
mv src/HackatonFiap.Payments.API/FGC.Payments.Presentation.csproj               src/HackatonFiap.Payments.API/HackatonFiap.Payments.API.csproj
# substituição de namespace/refs em todo o código e csproj
grep -rl "FGC.Payments.Presentation" src | xargs sed -i 's/FGC\.Payments\.Presentation/HackatonFiap.Payments.API/g'
grep -rl "FGC.Payments"              src | xargs sed -i 's/FGC\.Payments/HackatonFiap.Payments/g'
```

- [ ] **Step 4: Criar `global.json` (pin .NET 8) e a solution `.sln` com os 5 projetos**

`global.json`:
```json
{
  "sdk": {
    "version": "8.0.0",
    "rollForward": "latestFeature"
  }
}
```

```bash
cd "$ROOT"
dotnet new sln -n HackatonFiap.Payments
dotnet sln add src/HackatonFiap.Payments.Domain/HackatonFiap.Payments.Domain.csproj \
               src/HackatonFiap.Payments.Application/HackatonFiap.Payments.Application.csproj \
               src/HackatonFiap.Payments.Infrastructure/HackatonFiap.Payments.Infrastructure.csproj \
               src/HackatonFiap.Payments.API/HackatonFiap.Payments.API.csproj
```

- [ ] **Step 5: Criar o projeto de testes e adicioná-lo à solution**

```bash
cd "$ROOT"
dotnet new xunit -n HackatonFiap.Payments.UnitTests -o tests/HackatonFiap.Payments.UnitTests
dotnet sln add tests/HackatonFiap.Payments.UnitTests/HackatonFiap.Payments.UnitTests.csproj
dotnet add tests/HackatonFiap.Payments.UnitTests reference \
  src/HackatonFiap.Payments.Domain src/HackatonFiap.Payments.Application
dotnet add tests/HackatonFiap.Payments.UnitTests package NSubstitute
dotnet add tests/HackatonFiap.Payments.UnitTests package FluentAssertions
```

- [ ] **Step 6: Remover o domínio "games" do boilerplate que será reescrito**

Delete (serão recriados nas próximas tasks com o domínio de doação):
```bash
cd "$ROOT/src"
rm -f HackatonFiap.Payments.Domain/Entities/Payment.cs \
      HackatonFiap.Payments.Domain/Events/PaymentEvents.cs \
      HackatonFiap.Payments.Domain/Enums/PaymentEnums.cs \
      HackatonFiap.Payments.Application/UseCases/PaymentUseCases.cs \
      HackatonFiap.Payments.Application/DTOs/PaymentDTOs.cs \
      HackatonFiap.Payments.Application/Interfaces/IMessagePublisher.cs \
      HackatonFiap.Payments.Domain/Interfaces/IPaymentRepository.cs \
      HackatonFiap.Payments.Infrastructure/Messaging/ServiceBusPublisher.cs \
      HackatonFiap.Payments.Infrastructure/Repositories/PaymentRepository.cs \
      HackatonFiap.Payments.Infrastructure/Data/Configurations/PaymentConfiguration.cs \
      HackatonFiap.Payments.Presentation/Controllers/PaymentsController.cs \
      HackatonFiap.Payments.Presentation/Models/Requests/PaymentRequests.cs \
      HackatonFiap.Payments.Presentation/Models/Responses/PaymentResponses.cs
rm -rf HackatonFiap.Payments.Domain/Common   # AggregateRoot/Entity/IDomainEvent — não usaremos domain events internos
```
> O `PaymentsDbContext` em `Infrastructure/Data/Context/` será reescrito na Task 9; pode mantê-lo por ora (será sobrescrito).

- [ ] **Step 7: Commit do scaffolding**

```bash
cd "$ROOT"
git init -b main 2>/dev/null || true
git config user.name "GabrielVeridico"
git config user.email "60530060+GabrielVeridico@users.noreply.github.com"
git add -A
git commit -m "chore: scaffolding HackatonFiap.Payments a partir do boilerplate FGC.Payments (src/+tests/, .sln, global.json)"
```
> Não dar push ainda; o remote `origin` (GabrielVeridico/hackaton-fiap-payments) já existe e será configurado no final.

---

## Task 2: `Result<T>` e `Error` (Domain/Common)

**Files:**
- Create: `src/HackatonFiap.Payments.Domain/Common/Result.cs`

- [ ] **Step 1: Implementar `Result`, `Result<T>` e `Error`**

```csharp
namespace HackatonFiap.Payments.Domain.Common;

public sealed record Error(string Code, string Message)
{
    public static readonly Error None = new(string.Empty, string.Empty);
}

public class Result
{
    protected Result(bool isSuccess, Error error)
    {
        if (isSuccess && error != Error.None)
        {
            throw new InvalidOperationException("A successful result cannot carry an error.");
        }

        if (!isSuccess && error == Error.None)
        {
            throw new InvalidOperationException("A failed result requires an error.");
        }

        IsSuccess = isSuccess;
        Error = error;
    }

    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public Error Error { get; }

    public static Result Success() => new(true, Error.None);
    public static Result Failure(Error error) => new(false, error);
    public static Result<T> Success<T>(T value) => new(value, true, Error.None);
    public static Result<T> Failure<T>(Error error) => new(default!, false, error);
}

public sealed class Result<T> : Result
{
    private readonly T _value;

    internal Result(T value, bool isSuccess, Error error) : base(isSuccess, error)
    {
        _value = value;
    }

    public T Value => IsSuccess
        ? _value
        : throw new InvalidOperationException("Cannot access the value of a failed result.");
}
```

- [ ] **Step 2: Compilar o Domain**

Run: `dotnet build src/HackatonFiap.Payments.Domain`
Expected: build com êxito, 0 erros.

- [ ] **Step 3: Commit**

```bash
git add src/HackatonFiap.Payments.Domain/Common/Result.cs
git commit -m "feat(domain): Result<T> e Error"
```

---

## Task 3: Enums `PaymentStatus` e `PaymentMethod`

**Files:**
- Create: `src/HackatonFiap.Payments.Domain/Enums/PaymentStatus.cs`, `.../PaymentMethod.cs`

- [ ] **Step 1: Implementar os enums**

`PaymentStatus.cs`:
```csharp
namespace HackatonFiap.Payments.Domain.Enums;

public enum PaymentStatus
{
    Pending = 0,
    Approved = 1,
    Declined = 2
}
```

`PaymentMethod.cs`:
```csharp
namespace HackatonFiap.Payments.Domain.Enums;

public enum PaymentMethod
{
    Pix = 0,
    CreditCard = 1,
    BankTransfer = 2
}
```

- [ ] **Step 2: Build + Commit**

```bash
dotnet build src/HackatonFiap.Payments.Domain
git add src/HackatonFiap.Payments.Domain/Enums/
git commit -m "feat(domain): enums PaymentStatus e PaymentMethod"
```

---

## Task 4: Entidade `Payment` (TDD)

**Files:**
- Test: `tests/HackatonFiap.Payments.UnitTests/Domain/PaymentTests.cs`
- Create: `src/HackatonFiap.Payments.Domain/Entities/Payment.cs`

- [ ] **Step 1: Escrever os testes de comportamento da entidade**

```csharp
using FluentAssertions;
using HackatonFiap.Payments.Domain.Entities;
using HackatonFiap.Payments.Domain.Enums;
using Xunit;

namespace HackatonFiap.Payments.UnitTests.Domain;

public class PaymentTests
{
    private static Payment NewPending() => Payment.Create(
        donationId: Guid.NewGuid(),
        campaignId: Guid.NewGuid(),
        amount: 50.00m,
        method: PaymentMethod.Pix,
        donorId: Guid.NewGuid(),
        donorEmail: "donor@example.com",
        donorName: "Donor");

    [Fact]
    public void Create_starts_pending_with_transaction_id()
    {
        var payment = NewPending();

        payment.Status.Should().Be(PaymentStatus.Pending);
        payment.TransactionId.Should().StartWith("TXN-");
        payment.Id.Should().NotBe(Guid.Empty);
    }

    [Fact]
    public void Create_with_non_positive_amount_throws()
    {
        var act = () => Payment.Create(Guid.NewGuid(), Guid.NewGuid(), 0m,
            PaymentMethod.Pix, Guid.NewGuid(), "d@e.com", "D");

        act.Should().Throw<ArgumentException>();
    }

    [Fact]
    public void Approve_sets_status_and_processed_at()
    {
        var payment = NewPending();

        payment.Approve();

        payment.Status.Should().Be(PaymentStatus.Approved);
        payment.ProcessedAt.Should().NotBeNull();
    }

    [Fact]
    public void Decline_sets_status_and_reason()
    {
        var payment = NewPending();

        payment.Decline("centavos ,99");

        payment.Status.Should().Be(PaymentStatus.Declined);
        payment.DeclineReason.Should().Be("centavos ,99");
        payment.ProcessedAt.Should().NotBeNull();
    }

    [Fact]
    public void Approve_when_not_pending_throws()
    {
        var payment = NewPending();
        payment.Approve();

        var act = () => payment.Approve();

        act.Should().Throw<InvalidOperationException>();
    }
}
```

- [ ] **Step 2: Rodar os testes e ver falhar**

Run: `dotnet test --filter "FullyQualifiedName~PaymentTests"`
Expected: FALHA de compilação/execução (classe `Payment` não existe ainda).

- [ ] **Step 3: Implementar a entidade `Payment`**

```csharp
using HackatonFiap.Payments.Domain.Enums;

namespace HackatonFiap.Payments.Domain.Entities;

public class Payment
{
    private Payment() { }

    private Payment(Guid donationId, Guid campaignId, decimal amount, PaymentMethod method,
        Guid donorId, string donorEmail, string donorName)
    {
        Id = Guid.NewGuid();
        DonationId = donationId == Guid.Empty
            ? throw new ArgumentException("DonationId é obrigatório", nameof(donationId))
            : donationId;
        CampaignId = campaignId == Guid.Empty
            ? throw new ArgumentException("CampaignId é obrigatório", nameof(campaignId))
            : campaignId;
        Amount = amount > 0
            ? amount
            : throw new ArgumentException("Amount deve ser maior que zero", nameof(amount));
        Method = method;
        DonorId = donorId;
        DonorEmail = donorEmail;
        DonorName = donorName;
        Status = PaymentStatus.Pending;
        TransactionId = $"TXN-{DateTime.UtcNow:yyyyMMdd}-{Guid.NewGuid().ToString("N")[..8].ToUpperInvariant()}";
        CreatedAt = DateTime.UtcNow;
    }

    public Guid Id { get; private set; }
    public Guid DonationId { get; private set; }
    public Guid CampaignId { get; private set; }
    public decimal Amount { get; private set; }
    public PaymentMethod Method { get; private set; }
    public PaymentStatus Status { get; private set; }
    public string TransactionId { get; private set; } = string.Empty;
    public Guid DonorId { get; private set; }
    public string DonorEmail { get; private set; } = string.Empty;
    public string DonorName { get; private set; } = string.Empty;
    public DateTime CreatedAt { get; private set; }
    public DateTime? ProcessedAt { get; private set; }
    public string? DeclineReason { get; private set; }

    public static Payment Create(Guid donationId, Guid campaignId, decimal amount, PaymentMethod method,
        Guid donorId, string donorEmail, string donorName)
        => new(donationId, campaignId, amount, method, donorId, donorEmail, donorName);

    public void Approve()
    {
        if (Status != PaymentStatus.Pending)
        {
            throw new InvalidOperationException($"Não é possível aprovar pagamento com status {Status}.");
        }

        Status = PaymentStatus.Approved;
        ProcessedAt = DateTime.UtcNow;
    }

    public void Decline(string reason)
    {
        if (Status != PaymentStatus.Pending)
        {
            throw new InvalidOperationException($"Não é possível recusar pagamento com status {Status}.");
        }

        if (string.IsNullOrWhiteSpace(reason))
        {
            throw new ArgumentException("Motivo da recusa é obrigatório", nameof(reason));
        }

        Status = PaymentStatus.Declined;
        DeclineReason = reason;
        ProcessedAt = DateTime.UtcNow;
    }
}
```

- [ ] **Step 4: Rodar os testes e ver passar**

Run: `dotnet test --filter "FullyQualifiedName~PaymentTests"`
Expected: PASS (5 testes).

- [ ] **Step 5: Commit**

```bash
git add src/HackatonFiap.Payments.Domain/Entities/Payment.cs tests/HackatonFiap.Payments.UnitTests/Domain/PaymentTests.cs
git commit -m "feat(domain): entidade Payment (Create/Approve/Decline) + testes"
```

---

## Task 5: Eventos de integração e abstrações da Application

**Files:**
- Create: `src/HackatonFiap.Payments.Application/IntegrationEvents/{DonationRequestedEvent,PaymentApprovedEvent,PaymentDeclinedEvent}.cs`
- Create: `src/HackatonFiap.Payments.Application/Abstractions/{IPaymentRepository,IPaymentGateway,IEventPublisher}.cs`
- Create: `src/HackatonFiap.Payments.Application/Errors/PaymentErrors.cs`

- [ ] **Step 1: Adicionar referência do projeto Application → Domain (se ainda não houver)**

```bash
dotnet add src/HackatonFiap.Payments.Application reference src/HackatonFiap.Payments.Domain
```

- [ ] **Step 2: Criar os eventos de integração (records, inglês)**

`DonationRequestedEvent.cs`:
```csharp
using HackatonFiap.Payments.Domain.Enums;

namespace HackatonFiap.Payments.Application.IntegrationEvents;

public sealed record DonationRequestedEvent(
    Guid DonationId,
    Guid CampaignId,
    decimal Amount,
    PaymentMethod PaymentMethod,
    Guid DonorId,
    string DonorEmail,
    string DonorName);
```

`PaymentApprovedEvent.cs`:
```csharp
namespace HackatonFiap.Payments.Application.IntegrationEvents;

public sealed record PaymentApprovedEvent(
    Guid DonationId,
    Guid CampaignId,
    decimal Amount,
    Guid PaymentId,
    Guid DonorId,
    string DonorEmail,
    string DonorName);
```

`PaymentDeclinedEvent.cs`:
```csharp
namespace HackatonFiap.Payments.Application.IntegrationEvents;

public sealed record PaymentDeclinedEvent(
    Guid DonationId,
    Guid CampaignId,
    string Reason,
    decimal Amount,
    Guid DonorId,
    string DonorEmail,
    string DonorName);
```

- [ ] **Step 3: Criar as abstrações**

`Abstractions/IPaymentRepository.cs`:
```csharp
using HackatonFiap.Payments.Domain.Entities;

namespace HackatonFiap.Payments.Application.Abstractions;

public interface IPaymentRepository
{
    Task<Payment?> GetByDonationIdAsync(Guid donationId, CancellationToken cancellationToken = default);
    Task AddAsync(Payment payment, CancellationToken cancellationToken = default);
    Task SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

`Abstractions/IPaymentGateway.cs`:
```csharp
namespace HackatonFiap.Payments.Application.Abstractions;

public sealed record GatewayResult(bool Approved, string? DeclineReason);

public interface IPaymentGateway
{
    GatewayResult Process(decimal amount);
}
```

`Abstractions/IEventPublisher.cs`:
```csharp
namespace HackatonFiap.Payments.Application.Abstractions;

public interface IEventPublisher
{
    Task PublishAsync<TEvent>(TEvent integrationEvent, string subject, CancellationToken cancellationToken = default)
        where TEvent : notnull;
}
```

- [ ] **Step 4: Criar os erros da Application**

`Errors/PaymentErrors.cs`:
```csharp
using HackatonFiap.Payments.Domain.Common;

namespace HackatonFiap.Payments.Application.Errors;

public static class PaymentErrors
{
    public static readonly Error NotFound = new("Payment.NotFound", "Pagamento não encontrado para a doação informada.");
}
```

- [ ] **Step 5: Build + Commit**

```bash
dotnet build src/HackatonFiap.Payments.Application
git add src/HackatonFiap.Payments.Application/
git commit -m "feat(application): eventos de integração, abstrações (repo/gateway/publisher) e erros"
```

---

## Task 6: `MockPaymentGateway` determinístico (TDD)

**Files:**
- Test: `tests/HackatonFiap.Payments.UnitTests/Gateway/MockPaymentGatewayTests.cs`
- Create: `src/HackatonFiap.Payments.Infrastructure/Gateway/MockPaymentGateway.cs`

- [ ] **Step 1: Garantir referências do projeto de testes (Infrastructure) e da Infra → Application**

```bash
dotnet add src/HackatonFiap.Payments.Infrastructure reference src/HackatonFiap.Payments.Application
dotnet add tests/HackatonFiap.Payments.UnitTests reference src/HackatonFiap.Payments.Infrastructure
```

- [ ] **Step 2: Escrever os testes do gateway mock**

```csharp
using FluentAssertions;
using HackatonFiap.Payments.Infrastructure.Gateway;
using Microsoft.Extensions.Options;
using Xunit;

namespace HackatonFiap.Payments.UnitTests.Gateway;

public class MockPaymentGatewayTests
{
    private static MockPaymentGateway Gateway()
        => new(Options.Create(new MockPaymentGatewayOptions { DeclineCentavos = 99 }));

    [Theory]
    [InlineData(50.00)]
    [InlineData(10.50)]
    [InlineData(100.01)]
    public void Process_approves_by_default(decimal amount)
    {
        var result = Gateway().Process(amount);

        result.Approved.Should().BeTrue();
        result.DeclineReason.Should().BeNull();
    }

    [Theory]
    [InlineData(10.99)]
    [InlineData(0.99)]
    [InlineData(250.99)]
    public void Process_declines_when_centavos_match_marker(decimal amount)
    {
        var result = Gateway().Process(amount);

        result.Approved.Should().BeFalse();
        result.DeclineReason.Should().NotBeNullOrWhiteSpace();
    }
}
```

- [ ] **Step 3: Rodar e ver falhar**

Run: `dotnet test --filter "FullyQualifiedName~MockPaymentGatewayTests"`
Expected: FALHA de compilação (tipos não existem).

- [ ] **Step 4: Implementar `MockPaymentGatewayOptions` e `MockPaymentGateway`**

```csharp
using HackatonFiap.Payments.Application.Abstractions;
using Microsoft.Extensions.Options;

namespace HackatonFiap.Payments.Infrastructure.Gateway;

public sealed class MockPaymentGatewayOptions
{
    public const string SectionName = "PaymentGateway";

    // Recusa determinística quando os centavos do valor batem com este marcador (RN06.11).
    public int DeclineCentavos { get; set; } = 99;
}

public sealed class MockPaymentGateway : IPaymentGateway
{
    private readonly MockPaymentGatewayOptions _options;

    public MockPaymentGateway(IOptions<MockPaymentGatewayOptions> options)
    {
        _options = options.Value;
    }

    public GatewayResult Process(decimal amount)
    {
        var centavos = (int)(Math.Round(amount, 2) * 100) % 100;

        if (centavos == _options.DeclineCentavos)
        {
            return new GatewayResult(false, $"Pagamento recusado pelo gateway (marcador de teste: centavos {_options.DeclineCentavos:D2}).");
        }

        return new GatewayResult(true, null);
    }
}
```

- [ ] **Step 5: Rodar e ver passar**

Run: `dotnet test --filter "FullyQualifiedName~MockPaymentGatewayTests"`
Expected: PASS (6 casos).

- [ ] **Step 6: Commit**

```bash
git add src/HackatonFiap.Payments.Infrastructure/Gateway/ tests/HackatonFiap.Payments.UnitTests/Gateway/
git commit -m "feat(infra): MockPaymentGateway determinístico (recusa centavos ,99) + testes"
```

---

## Task 7: `ProcessDonationPaymentCommand` + Handler (TDD)

**Files:**
- Test: `tests/HackatonFiap.Payments.UnitTests/Application/ProcessDonationPaymentCommandHandlerTests.cs`
- Create: `src/HackatonFiap.Payments.Application/Commands/ProcessDonationPayment/ProcessDonationPaymentCommand.cs`
- Create: `.../ProcessDonationPaymentCommandHandler.cs`

- [ ] **Step 1: Escrever os testes do handler (aprovado, recusado, idempotência)**

```csharp
using FluentAssertions;
using HackatonFiap.Payments.Application.Abstractions;
using HackatonFiap.Payments.Application.Commands.ProcessDonationPayment;
using HackatonFiap.Payments.Application.IntegrationEvents;
using HackatonFiap.Payments.Domain.Entities;
using HackatonFiap.Payments.Domain.Enums;
using NSubstitute;
using Xunit;

namespace HackatonFiap.Payments.UnitTests.Application;

public class ProcessDonationPaymentCommandHandlerTests
{
    private readonly IPaymentRepository _repository = Substitute.For<IPaymentRepository>();
    private readonly IPaymentGateway _gateway = Substitute.For<IPaymentGateway>();
    private readonly IEventPublisher _publisher = Substitute.For<IEventPublisher>();

    private ProcessDonationPaymentCommandHandler Handler() => new(_repository, _gateway, _publisher);

    private static ProcessDonationPaymentCommand NewCommand(decimal amount = 50.00m) => new(
        DonationId: Guid.NewGuid(),
        CampaignId: Guid.NewGuid(),
        Amount: amount,
        PaymentMethod: PaymentMethod.Pix,
        DonorId: Guid.NewGuid(),
        DonorEmail: "donor@example.com",
        DonorName: "Donor");

    [Fact]
    public async Task Approved_payment_persists_and_publishes_PaymentApprovedEvent()
    {
        _repository.GetByDonationIdAsync(Arg.Any<Guid>()).Returns((Payment?)null);
        _gateway.Process(Arg.Any<decimal>()).Returns(new GatewayResult(true, null));
        var command = NewCommand();

        var result = await Handler().Handle(command, CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        await _repository.Received(1).AddAsync(Arg.Is<Payment>(p => p.Status == PaymentStatus.Approved));
        await _publisher.Received(1).PublishAsync(
            Arg.Is<PaymentApprovedEvent>(e => e.DonationId == command.DonationId),
            "PaymentApproved",
            Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task Declined_payment_publishes_PaymentDeclinedEvent()
    {
        _repository.GetByDonationIdAsync(Arg.Any<Guid>()).Returns((Payment?)null);
        _gateway.Process(Arg.Any<decimal>()).Returns(new GatewayResult(false, "recusado"));
        var command = NewCommand(10.99m);

        var result = await Handler().Handle(command, CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        await _repository.Received(1).AddAsync(Arg.Is<Payment>(p => p.Status == PaymentStatus.Declined));
        await _publisher.Received(1).PublishAsync(
            Arg.Any<PaymentDeclinedEvent>(), "PaymentDeclined", Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task Duplicate_donation_is_idempotent_does_not_create_again_but_republishes()
    {
        var command = NewCommand();
        var existing = Payment.Create(command.DonationId, command.CampaignId, command.Amount,
            command.PaymentMethod, command.DonorId, command.DonorEmail, command.DonorName);
        existing.Approve();
        _repository.GetByDonationIdAsync(command.DonationId).Returns(existing);

        var result = await Handler().Handle(command, CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        await _repository.DidNotReceive().AddAsync(Arg.Any<Payment>());
        _gateway.DidNotReceive().Process(Arg.Any<decimal>());
        await _publisher.Received(1).PublishAsync(
            Arg.Any<PaymentApprovedEvent>(), "PaymentApproved", Arg.Any<CancellationToken>());
    }
}
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `dotnet test --filter "FullyQualifiedName~ProcessDonationPaymentCommandHandlerTests"`
Expected: FALHA de compilação (command/handler não existem).

- [ ] **Step 3: Implementar o command (record)**

`ProcessDonationPaymentCommand.cs`:
```csharp
using HackatonFiap.Payments.Domain.Enums;

namespace HackatonFiap.Payments.Application.Commands.ProcessDonationPayment;

public sealed record ProcessDonationPaymentCommand(
    Guid DonationId,
    Guid CampaignId,
    decimal Amount,
    PaymentMethod PaymentMethod,
    Guid DonorId,
    string DonorEmail,
    string DonorName);
```

- [ ] **Step 4: Implementar o handler**

`ProcessDonationPaymentCommandHandler.cs`:
```csharp
using HackatonFiap.Payments.Application.Abstractions;
using HackatonFiap.Payments.Application.IntegrationEvents;
using HackatonFiap.Payments.Domain.Common;
using HackatonFiap.Payments.Domain.Entities;
using HackatonFiap.Payments.Domain.Enums;

namespace HackatonFiap.Payments.Application.Commands.ProcessDonationPayment;

public sealed class ProcessDonationPaymentCommandHandler
{
    private readonly IPaymentRepository _repository;
    private readonly IPaymentGateway _gateway;
    private readonly IEventPublisher _publisher;

    public ProcessDonationPaymentCommandHandler(
        IPaymentRepository repository, IPaymentGateway gateway, IEventPublisher publisher)
    {
        _repository = repository;
        _gateway = gateway;
        _publisher = publisher;
    }

    public async Task<Result> Handle(ProcessDonationPaymentCommand command, CancellationToken cancellationToken)
    {
        // Idempotência (at-least-once): se já processamos esta doação, apenas republica o resultado.
        var existing = await _repository.GetByDonationIdAsync(command.DonationId, cancellationToken);
        if (existing is not null)
        {
            await PublishResultAsync(existing, cancellationToken);
            return Result.Success();
        }

        var payment = Payment.Create(
            command.DonationId, command.CampaignId, command.Amount, command.PaymentMethod,
            command.DonorId, command.DonorEmail, command.DonorName);

        var gatewayResult = _gateway.Process(command.Amount);
        if (gatewayResult.Approved)
        {
            payment.Approve();
        }
        else
        {
            payment.Decline(gatewayResult.DeclineReason ?? "Pagamento recusado.");
        }

        await _repository.AddAsync(payment, cancellationToken);
        await _repository.SaveChangesAsync(cancellationToken);
        await PublishResultAsync(payment, cancellationToken);

        return Result.Success();
    }

    private Task PublishResultAsync(Payment payment, CancellationToken cancellationToken)
    {
        if (payment.Status == PaymentStatus.Approved)
        {
            var approved = new PaymentApprovedEvent(
                payment.DonationId, payment.CampaignId, payment.Amount, payment.Id,
                payment.DonorId, payment.DonorEmail, payment.DonorName);
            return _publisher.PublishAsync(approved, "PaymentApproved", cancellationToken);
        }

        var declined = new PaymentDeclinedEvent(
            payment.DonationId, payment.CampaignId, payment.DeclineReason ?? "Pagamento recusado.",
            payment.Amount, payment.DonorId, payment.DonorEmail, payment.DonorName);
        return _publisher.PublishAsync(declined, "PaymentDeclined", cancellationToken);
    }
}
```

- [ ] **Step 5: Rodar e ver passar**

Run: `dotnet test --filter "FullyQualifiedName~ProcessDonationPaymentCommandHandlerTests"`
Expected: PASS (3 testes).

- [ ] **Step 6: Commit**

```bash
git add src/HackatonFiap.Payments.Application/Commands/ tests/HackatonFiap.Payments.UnitTests/Application/ProcessDonationPaymentCommandHandlerTests.cs
git commit -m "feat(application): ProcessDonationPaymentCommandHandler (aprovação/recusa/idempotência) + testes"
```

---

## Task 8: `GetPaymentByDonationIdQuery` + Handler (TDD)

**Files:**
- Test: `tests/HackatonFiap.Payments.UnitTests/Application/GetPaymentByDonationIdQueryHandlerTests.cs`
- Create: `.../Queries/GetPaymentByDonationId/GetPaymentByDonationIdQuery.cs` (Query + PaymentResponse + Handler)

- [ ] **Step 1: Escrever os testes da query**

```csharp
using FluentAssertions;
using HackatonFiap.Payments.Application.Abstractions;
using HackatonFiap.Payments.Application.Errors;
using HackatonFiap.Payments.Application.Queries.GetPaymentByDonationId;
using HackatonFiap.Payments.Domain.Entities;
using HackatonFiap.Payments.Domain.Enums;
using NSubstitute;
using Xunit;

namespace HackatonFiap.Payments.UnitTests.Application;

public class GetPaymentByDonationIdQueryHandlerTests
{
    private readonly IPaymentRepository _repository = Substitute.For<IPaymentRepository>();

    [Fact]
    public async Task Returns_payment_when_found()
    {
        var donationId = Guid.NewGuid();
        var payment = Payment.Create(donationId, Guid.NewGuid(), 50m, PaymentMethod.Pix,
            Guid.NewGuid(), "d@e.com", "D");
        _repository.GetByDonationIdAsync(donationId).Returns(payment);

        var result = await new GetPaymentByDonationIdQueryHandler(_repository)
            .Handle(new GetPaymentByDonationIdQuery(donationId), CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        result.Value.DonationId.Should().Be(donationId);
        result.Value.Status.Should().Be(nameof(PaymentStatus.Pending));
    }

    [Fact]
    public async Task Returns_failure_when_not_found()
    {
        _repository.GetByDonationIdAsync(Arg.Any<Guid>()).Returns((Payment?)null);

        var result = await new GetPaymentByDonationIdQueryHandler(_repository)
            .Handle(new GetPaymentByDonationIdQuery(Guid.NewGuid()), CancellationToken.None);

        result.IsFailure.Should().BeTrue();
        result.Error.Should().Be(PaymentErrors.NotFound);
    }
}
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `dotnet test --filter "FullyQualifiedName~GetPaymentByDonationIdQueryHandlerTests"`
Expected: FALHA de compilação.

- [ ] **Step 3: Implementar query, response e handler**

`GetPaymentByDonationIdQuery.cs`:
```csharp
using HackatonFiap.Payments.Application.Abstractions;
using HackatonFiap.Payments.Application.Errors;
using HackatonFiap.Payments.Domain.Common;

namespace HackatonFiap.Payments.Application.Queries.GetPaymentByDonationId;

public sealed record GetPaymentByDonationIdQuery(Guid DonationId);

public sealed record PaymentResponse(
    Guid Id,
    Guid DonationId,
    Guid CampaignId,
    decimal Amount,
    string Method,
    string Status,
    string TransactionId,
    DateTime CreatedAt,
    DateTime? ProcessedAt,
    string? DeclineReason);

public sealed class GetPaymentByDonationIdQueryHandler
{
    private readonly IPaymentRepository _repository;

    public GetPaymentByDonationIdQueryHandler(IPaymentRepository repository)
    {
        _repository = repository;
    }

    public async Task<Result<PaymentResponse>> Handle(
        GetPaymentByDonationIdQuery query, CancellationToken cancellationToken)
    {
        var payment = await _repository.GetByDonationIdAsync(query.DonationId, cancellationToken);
        if (payment is null)
        {
            return Result.Failure<PaymentResponse>(PaymentErrors.NotFound);
        }

        return Result.Success(new PaymentResponse(
            payment.Id, payment.DonationId, payment.CampaignId, payment.Amount,
            payment.Method.ToString(), payment.Status.ToString(), payment.TransactionId,
            payment.CreatedAt, payment.ProcessedAt, payment.DeclineReason));
    }
}
```

- [ ] **Step 4: Rodar e ver passar**

Run: `dotnet test --filter "FullyQualifiedName~GetPaymentByDonationIdQueryHandlerTests"`
Expected: PASS (2 testes).

- [ ] **Step 5: Commit**

```bash
git add src/HackatonFiap.Payments.Application/Queries/ tests/HackatonFiap.Payments.UnitTests/Application/GetPaymentByDonationIdQueryHandlerTests.cs
git commit -m "feat(application): GetPaymentByDonationIdQueryHandler + testes"
```

---

## Task 9: Persistência — `PaymentsDbContext`, configuration e repository

**Files:**
- Create/Replace: `src/HackatonFiap.Payments.Infrastructure/Persistence/PaymentsDbContext.cs`
- Create: `.../Persistence/Configurations/PaymentConfiguration.cs`
- Create: `.../Persistence/Repositories/PaymentRepository.cs`

- [ ] **Step 1: Garantir pacotes EF na Infrastructure**

```bash
dotnet add src/HackatonFiap.Payments.Infrastructure package Microsoft.EntityFrameworkCore --version 8.0.*
dotnet add src/HackatonFiap.Payments.Infrastructure package Microsoft.EntityFrameworkCore.SqlServer --version 8.0.*
```

- [ ] **Step 2: Implementar o DbContext**

`Persistence/PaymentsDbContext.cs`:
```csharp
using HackatonFiap.Payments.Domain.Entities;
using Microsoft.EntityFrameworkCore;

namespace HackatonFiap.Payments.Infrastructure.Persistence;

public class PaymentsDbContext : DbContext
{
    public PaymentsDbContext(DbContextOptions<PaymentsDbContext> options) : base(options) { }

    public DbSet<Payment> Payments => Set<Payment>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(PaymentsDbContext).Assembly);
        base.OnModelCreating(modelBuilder);
    }
}
```

- [ ] **Step 3: Implementar a configuration (índice único em DonationId)**

`Persistence/Configurations/PaymentConfiguration.cs`:
```csharp
using HackatonFiap.Payments.Domain.Entities;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace HackatonFiap.Payments.Infrastructure.Persistence.Configurations;

public sealed class PaymentConfiguration : IEntityTypeConfiguration<Payment>
{
    public void Configure(EntityTypeBuilder<Payment> builder)
    {
        builder.ToTable("Payments");
        builder.HasKey(p => p.Id);

        builder.Property(p => p.DonationId).IsRequired();
        builder.HasIndex(p => p.DonationId).IsUnique();   // idempotência

        builder.Property(p => p.CampaignId).IsRequired();
        builder.Property(p => p.Amount).HasColumnType("decimal(18,2)").IsRequired();
        builder.Property(p => p.Method).HasConversion<int>().IsRequired();
        builder.Property(p => p.Status).HasConversion<int>().IsRequired();
        builder.Property(p => p.TransactionId).HasMaxLength(40).IsRequired();
        builder.Property(p => p.DonorEmail).HasMaxLength(256).IsRequired();
        builder.Property(p => p.DonorName).HasMaxLength(256).IsRequired();
        builder.Property(p => p.DeclineReason).HasMaxLength(512);
    }
}
```

- [ ] **Step 4: Implementar o repository**

`Persistence/Repositories/PaymentRepository.cs`:
```csharp
using HackatonFiap.Payments.Application.Abstractions;
using HackatonFiap.Payments.Domain.Entities;
using Microsoft.EntityFrameworkCore;

namespace HackatonFiap.Payments.Infrastructure.Persistence.Repositories;

public sealed class PaymentRepository : IPaymentRepository
{
    private readonly PaymentsDbContext _context;

    public PaymentRepository(PaymentsDbContext context)
    {
        _context = context;
    }

    public Task<Payment?> GetByDonationIdAsync(Guid donationId, CancellationToken cancellationToken = default)
        => _context.Payments.AsNoTracking().FirstOrDefaultAsync(p => p.DonationId == donationId, cancellationToken);

    public async Task AddAsync(Payment payment, CancellationToken cancellationToken = default)
        => await _context.Payments.AddAsync(payment, cancellationToken);

    public Task SaveChangesAsync(CancellationToken cancellationToken = default)
        => _context.SaveChangesAsync(cancellationToken);
}
```

- [ ] **Step 5: Build + Commit**

```bash
dotnet build src/HackatonFiap.Payments.Infrastructure
git add src/HackatonFiap.Payments.Infrastructure/Persistence/
git commit -m "feat(infra): PaymentsDbContext, PaymentConfiguration (índice único DonationId) e repository"
```

---

## Task 10: `ServiceBusEventPublisher` (tópico de resultado)

**Files:**
- Create: `src/HackatonFiap.Payments.Infrastructure/Messaging/ServiceBusEventPublisher.cs`

- [ ] **Step 1: Garantir pacote do Service Bus**

```bash
dotnet add src/HackatonFiap.Payments.Infrastructure package Azure.Messaging.ServiceBus --version 7.20.*
```

- [ ] **Step 2: Implementar o publisher (publica no tópico `payment-result` com `Subject`)**

`Messaging/ServiceBusEventPublisher.cs`:
```csharp
using System.Text.Json;
using Azure.Messaging.ServiceBus;
using HackatonFiap.Payments.Application.Abstractions;
using Microsoft.Extensions.Logging;

namespace HackatonFiap.Payments.Infrastructure.Messaging;

public sealed class ServiceBusEventPublisher : IEventPublisher, IAsyncDisposable
{
    private readonly ServiceBusSender _sender;
    private readonly ILogger<ServiceBusEventPublisher> _logger;

    public ServiceBusEventPublisher(ServiceBusClient client, string resultTopicName, ILogger<ServiceBusEventPublisher> logger)
    {
        _sender = client.CreateSender(resultTopicName);
        _logger = logger;
    }

    public async Task PublishAsync<TEvent>(TEvent integrationEvent, string subject, CancellationToken cancellationToken = default)
        where TEvent : notnull
    {
        var body = JsonSerializer.Serialize(integrationEvent);
        var message = new ServiceBusMessage(body)
        {
            ContentType = "application/json",
            Subject = subject
        };

        await _sender.SendMessageAsync(message, cancellationToken);
        _logger.LogInformation("Evento publicado no tópico de resultado. Subject={Subject}", subject);
    }

    public ValueTask DisposeAsync() => _sender.DisposeAsync();
}
```
> O `ServiceBusClient` e o nome do tópico são injetados pelo `Program.cs` (Task 13).

- [ ] **Step 3: Build + Commit**

```bash
dotnet build src/HackatonFiap.Payments.Infrastructure
git add src/HackatonFiap.Payments.Infrastructure/Messaging/ServiceBusEventPublisher.cs
git commit -m "feat(infra): ServiceBusEventPublisher (tópico payment-result + Subject)"
```

---

## Task 11: `DonationRequestedConsumer` (BackgroundService)

**Files:**
- Create: `src/HackatonFiap.Payments.Infrastructure/Messaging/DonationRequestedConsumer.cs`

- [ ] **Step 1: Garantir pacotes (Hosting Abstractions e DI) na Infrastructure**

```bash
dotnet add src/HackatonFiap.Payments.Infrastructure package Microsoft.Extensions.Hosting.Abstractions --version 8.0.*
dotnet add src/HackatonFiap.Payments.Infrastructure package Microsoft.Extensions.DependencyInjection.Abstractions --version 8.0.*
```

- [ ] **Step 2: Implementar o consumer (processa a subscription `payments` do tópico `donation-requested`)**

`Messaging/DonationRequestedConsumer.cs`:
```csharp
using System.Text.Json;
using Azure.Messaging.ServiceBus;
using HackatonFiap.Payments.Application.Commands.ProcessDonationPayment;
using HackatonFiap.Payments.Application.IntegrationEvents;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

namespace HackatonFiap.Payments.Infrastructure.Messaging;

public sealed class DonationRequestedConsumer : BackgroundService
{
    private readonly ServiceBusProcessor _processor;
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<DonationRequestedConsumer> _logger;

    public DonationRequestedConsumer(
        ServiceBusClient client,
        string requestTopicName,
        string subscriptionName,
        IServiceScopeFactory scopeFactory,
        ILogger<DonationRequestedConsumer> logger)
    {
        _processor = client.CreateProcessor(requestTopicName, subscriptionName, new ServiceBusProcessorOptions
        {
            AutoCompleteMessages = false,
            MaxConcurrentCalls = 1
        });
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _processor.ProcessMessageAsync += OnMessageAsync;
        _processor.ProcessErrorAsync += OnErrorAsync;
        await _processor.StartProcessingAsync(stoppingToken);
    }

    private async Task OnMessageAsync(ProcessMessageEventArgs args)
    {
        try
        {
            var evt = JsonSerializer.Deserialize<DonationRequestedEvent>(args.Message.Body.ToString());
            if (evt is null)
            {
                _logger.LogWarning("DonationRequestedEvent inválido; enviando para dead-letter.");
                await args.DeadLetterMessageAsync(args.Message, "InvalidPayload", "Body nulo ou inválido.");
                return;
            }

            using var scope = _scopeFactory.CreateScope();
            var handler = scope.ServiceProvider.GetRequiredService<ProcessDonationPaymentCommandHandler>();

            var command = new ProcessDonationPaymentCommand(
                evt.DonationId, evt.CampaignId, evt.Amount, evt.PaymentMethod,
                evt.DonorId, evt.DonorEmail, evt.DonorName);

            var result = await handler.Handle(command, args.CancellationToken);
            if (result.IsFailure)
            {
                _logger.LogError("Falha ao processar pagamento da doação {DonationId}: {Error}",
                    evt.DonationId, result.Error.Message);
                await args.AbandonMessageAsync(args.Message);
                return;
            }

            await args.CompleteMessageAsync(args.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Erro ao processar mensagem; abandonando para reentrega.");
            await args.AbandonMessageAsync(args.Message);
        }
    }

    private Task OnErrorAsync(ProcessErrorEventArgs args)
    {
        _logger.LogError(args.Exception, "Erro no ServiceBusProcessor. Source={Source}", args.ErrorSource);
        return Task.CompletedTask;
    }

    public override async Task StopAsync(CancellationToken cancellationToken)
    {
        await _processor.StopProcessingAsync(cancellationToken);
        await _processor.DisposeAsync();
        await base.StopAsync(cancellationToken);
    }
}
```

- [ ] **Step 3: Build + Commit**

```bash
dotnet build src/HackatonFiap.Payments.Infrastructure
git add src/HackatonFiap.Payments.Infrastructure/Messaging/DonationRequestedConsumer.cs
git commit -m "feat(infra): DonationRequestedConsumer (BackgroundService, subscription payments)"
```

---

## Task 12: Observabilidade — `Telemetry` (Meter)

**Files:**
- Create: `src/HackatonFiap.Payments.API/Observability/Telemetry.cs`

- [ ] **Step 1: Implementar `Telemetry` com os contadores de negócio**

`Observability/Telemetry.cs`:
```csharp
using System.Diagnostics.Metrics;

namespace HackatonFiap.Payments.API.Observability;

public static class Telemetry
{
    public const string ServiceName = "HackatonFiap.Payments";

    private static readonly Meter Meter = new(ServiceName);

    public static readonly Counter<long> PaymentsProcessed =
        Meter.CreateCounter<long>("payments_processed_total", description: "Total de pagamentos processados.");
    public static readonly Counter<long> PaymentsApproved =
        Meter.CreateCounter<long>("payments_approved_total", description: "Total de pagamentos aprovados.");
    public static readonly Counter<long> PaymentsDeclined =
        Meter.CreateCounter<long>("payments_declined_total", description: "Total de pagamentos recusados.");
}
```

- [ ] **Step 2: Incrementar os contadores no handler**

Modify: `src/HackatonFiap.Payments.Application/Commands/ProcessDonationPayment/ProcessDonationPaymentCommandHandler.cs`

> A Application não referencia a API. Em vez de acoplar, exponha os contadores via uma abstração simples na Application e implemente na API — OU (mais simples para o MVP) mantenha os contadores na Application. Para este plano, mova `Telemetry` para a Application:

- Create `src/HackatonFiap.Payments.Application/Observability/PaymentMetrics.cs` com o mesmo conteúdo do Step 1 (namespace `HackatonFiap.Payments.Application.Observability`), e **não** crie o arquivo em `API/Observability`. No `ProcessDonationPaymentCommandHandler.Handle`, após decidir aprovação/recusa, adicione:

```csharp
using HackatonFiap.Payments.Application.Observability;
// ...
PaymentMetrics.PaymentsProcessed.Add(1);
if (payment.Status == PaymentStatus.Approved)
{
    PaymentMetrics.PaymentsApproved.Add(1);
}
else
{
    PaymentMetrics.PaymentsDeclined.Add(1);
}
```
> Coloque os incrementos **apenas no caminho de criação** (não no caminho idempotente de republicação). O `Meter` name continua `HackatonFiap.Payments` para o exporter capturar.

- [ ] **Step 3: Build + testes (garantir que os testes do handler seguem verdes) + Commit**

```bash
dotnet build
dotnet test --filter "FullyQualifiedName~ProcessDonationPaymentCommandHandlerTests"
git add src/HackatonFiap.Payments.Application/Observability/ src/HackatonFiap.Payments.Application/Commands/
git commit -m "feat(obs): métricas de negócio (processed/approved/declined) no handler"
```

---

## Task 13: API — pacotes, `Program.cs`, `PaymentsController`, appsettings

**Files:**
- Modify: `src/HackatonFiap.Payments.API/HackatonFiap.Payments.API.csproj`
- Replace: `src/HackatonFiap.Payments.API/Program.cs`
- Create: `src/HackatonFiap.Payments.API/Controllers/PaymentsController.cs`
- Modify: `src/HackatonFiap.Payments.API/appsettings.json`

- [ ] **Step 1: Adicionar pacotes na API (alinhado ao users)**

```bash
cd "C:/eSolution/Projetos/hackaton-fiap/hackaton-fiap-payments"
dotnet add src/HackatonFiap.Payments.API package Microsoft.AspNetCore.Authentication.JwtBearer --version 8.0.*
dotnet add src/HackatonFiap.Payments.API package Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore --version 8.0.*
dotnet add src/HackatonFiap.Payments.API package Serilog.AspNetCore --version 8.0.*
dotnet add src/HackatonFiap.Payments.API package OpenTelemetry.Extensions.Hosting --version 1.9.0
dotnet add src/HackatonFiap.Payments.API package OpenTelemetry.Instrumentation.AspNetCore --version 1.9.0
dotnet add src/HackatonFiap.Payments.API package OpenTelemetry.Instrumentation.Runtime --version 1.9.0
dotnet add src/HackatonFiap.Payments.API package OpenTelemetry.Exporter.Prometheus.AspNetCore --version 1.9.0-beta.2
dotnet add src/HackatonFiap.Payments.API package Swashbuckle.AspNetCore --version 6.6.2
```

- [ ] **Step 2: Substituir o `Program.cs`**

`Program.cs`:
```csharp
using System.Text;
using Azure.Messaging.ServiceBus;
using HackatonFiap.Payments.API.Observability;
using HackatonFiap.Payments.Application.Abstractions;
using HackatonFiap.Payments.Application.Commands.ProcessDonationPayment;
using HackatonFiap.Payments.Application.Queries.GetPaymentByDonationId;
using HackatonFiap.Payments.Infrastructure.Gateway;
using HackatonFiap.Payments.Infrastructure.Messaging;
using HackatonFiap.Payments.Infrastructure.Persistence;
using HackatonFiap.Payments.Infrastructure.Persistence.Repositories;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.EntityFrameworkCore;
using Microsoft.IdentityModel.Tokens;
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;
using Serilog;

Log.Logger = new LoggerConfiguration().MinimumLevel.Information().WriteTo.Console().CreateBootstrapLogger();

try
{
    var builder = WebApplication.CreateBuilder(args);
    var configuration = builder.Configuration;

    builder.Host.UseSerilog((ctx, sp, cfg) => cfg
        .ReadFrom.Configuration(ctx.Configuration)
        .Enrich.FromLogContext()
        .Enrich.WithProperty("ServiceName", Telemetry.ServiceName)
        .WriteTo.Console());

    builder.Services.AddControllers();
    builder.Services.AddEndpointsApiExplorer();
    builder.Services.AddSwaggerGen();

    // Database
    var connectionString = configuration.GetValue<string>("ConnectionStrings:Default");
    if (string.IsNullOrWhiteSpace(connectionString))
    {
        throw new InvalidOperationException("ConnectionStrings:Default must be configured.");
    }
    builder.Services.AddDbContext<PaymentsDbContext>(opt =>
        opt.UseSqlServer(connectionString, sql => sql.EnableRetryOnFailure(6, TimeSpan.FromSeconds(30), null)));

    // CQRS handlers + repository
    builder.Services.AddScoped<IPaymentRepository, PaymentRepository>();
    builder.Services.AddScoped<ProcessDonationPaymentCommandHandler>();
    builder.Services.AddScoped<GetPaymentByDonationIdQueryHandler>();

    // Gateway mock
    builder.Services.Configure<MockPaymentGatewayOptions>(
        configuration.GetSection(MockPaymentGatewayOptions.SectionName));
    builder.Services.AddSingleton<IPaymentGateway, MockPaymentGateway>();

    // Service Bus: client + publisher + consumer
    var sbConnection = configuration.GetValue<string>("ServiceBus:ConnectionString")
        ?? throw new InvalidOperationException("ServiceBus:ConnectionString must be configured.");
    var requestTopic = configuration.GetValue<string>("ServiceBus:RequestTopic") ?? "donation-requested";
    var subscription = configuration.GetValue<string>("ServiceBus:Subscription") ?? "payments";
    var resultTopic = configuration.GetValue<string>("ServiceBus:ResultTopic") ?? "payment-result";

    builder.Services.AddSingleton(_ => new ServiceBusClient(sbConnection));
    builder.Services.AddSingleton<IEventPublisher>(sp => new ServiceBusEventPublisher(
        sp.GetRequiredService<ServiceBusClient>(), resultTopic,
        sp.GetRequiredService<ILogger<ServiceBusEventPublisher>>()));
    builder.Services.AddHostedService(sp => new DonationRequestedConsumer(
        sp.GetRequiredService<ServiceBusClient>(), requestTopic, subscription,
        sp.GetRequiredService<IServiceScopeFactory>(),
        sp.GetRequiredService<ILogger<DonationRequestedConsumer>>()));

    // Auth (JWT) — endpoint de consulta restrito a GestorONG/Owner
    var jwtKey = configuration.GetValue<string>("Jwt:Key");
    var jwtIssuer = configuration.GetValue<string>("Jwt:Issuer") ?? "conexaosolidaria.local";
    var jwtAudience = configuration.GetValue<string>("Jwt:Audience") ?? "conexaosolidaria.clients";
    if (string.IsNullOrEmpty(jwtKey) && !builder.Environment.IsDevelopment())
    {
        throw new InvalidOperationException("Jwt:Key must be configured outside Development.");
    }
    jwtKey ??= Convert.ToBase64String(System.Security.Cryptography.RandomNumberGenerator.GetBytes(48));

    builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(options =>
        {
            options.RequireHttpsMetadata = false;
            options.TokenValidationParameters = new TokenValidationParameters
            {
                ValidateIssuer = true,
                ValidIssuer = jwtIssuer,
                ValidateAudience = true,
                ValidAudience = jwtAudience,
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtKey)),
                ValidateLifetime = true
            };
        });
    builder.Services.AddAuthorization(options =>
        options.AddPolicy("ManagersOnly", p => p.RequireRole("GestorONG", "Owner")));

    // Observabilidade
    builder.Services.AddOpenTelemetry().WithMetrics(m => m
        .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService(Telemetry.ServiceName))
        .AddMeter(Telemetry.ServiceName)
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddPrometheusExporter());

    builder.Services.AddHealthChecks()
        .AddDbContextCheck<PaymentsDbContext>(name: "sqlserver", tags: new[] { "ready" });

    var app = builder.Build();

    using (var scope = app.Services.CreateScope())
    {
        scope.ServiceProvider.GetRequiredService<PaymentsDbContext>().Database.Migrate();
    }

    app.UseSerilogRequestLogging();
    app.UseSwagger();
    app.UseSwaggerUI();
    app.UseAuthentication();
    app.UseAuthorization();
    app.MapControllers();

    app.MapHealthChecks("/health", new HealthCheckOptions { Predicate = _ => false });
    app.MapHealthChecks("/ready", new HealthCheckOptions { Predicate = c => c.Tags.Contains("ready") });
    app.MapPrometheusScrapingEndpoint("/metrics");

    app.Run();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
}
finally
{
    Log.CloseAndFlush();
}
```

> Mova `Telemetry`/`PaymentMetrics` para `Application.Observability` (Task 12) e ajuste o `using` no `Program.cs` para `HackatonFiap.Payments.Application.Observability` se necessário; o `AddMeter(Telemetry.ServiceName)` deve referenciar a mesma constante de nome do `Meter`.

- [ ] **Step 3: Criar o `PaymentsController` (GET de consulta)**

`Controllers/PaymentsController.cs`:
```csharp
using HackatonFiap.Payments.Application.Queries.GetPaymentByDonationId;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace HackatonFiap.Payments.API.Controllers;

[ApiController]
[Route("api/payments")]
[Authorize(Policy = "ManagersOnly")]
public sealed class PaymentsController : ControllerBase
{
    private readonly GetPaymentByDonationIdQueryHandler _handler;

    public PaymentsController(GetPaymentByDonationIdQueryHandler handler)
    {
        _handler = handler;
    }

    [HttpGet("{donationId:guid}")]
    public async Task<IActionResult> GetByDonationId(Guid donationId, CancellationToken cancellationToken)
    {
        var result = await _handler.Handle(new GetPaymentByDonationIdQuery(donationId), cancellationToken);
        if (result.IsFailure)
        {
            return NotFound(new { error = result.Error.Code, message = result.Error.Message });
        }

        return Ok(result.Value);
    }
}
```

- [ ] **Step 4: Configurar `appsettings.json`**

`appsettings.json` (substituir conteúdo de demo do boilerplate):
```json
{
  "Serilog": { "MinimumLevel": { "Default": "Information" } },
  "ConnectionStrings": {
    "Default": ""
  },
  "ServiceBus": {
    "ConnectionString": "",
    "RequestTopic": "donation-requested",
    "Subscription": "payments",
    "ResultTopic": "payment-result"
  },
  "Jwt": {
    "Issuer": "conexaosolidaria.local",
    "Audience": "conexaosolidaria.clients",
    "Key": ""
  },
  "PaymentGateway": { "DeclineCentavos": 99 },
  "AllowedHosts": "*"
}
```

- [ ] **Step 5: Build da solução inteira + rodar todos os testes**

Run:
```bash
dotnet build HackatonFiap.Payments.sln
dotnet test --nologo
```
Expected: build com êxito; todos os testes (Domain + Gateway + Application) passam.

- [ ] **Step 6: Commit**

```bash
git add src/HackatonFiap.Payments.API/
git commit -m "feat(api): Program.cs (DI, JWT, OTel, Serilog, health), PaymentsController e appsettings"
```

---

## Task 14: Migration `InitialCreate`

**Files:**
- Create: `src/HackatonFiap.Payments.Infrastructure/Migrations/*` (gerado)

- [ ] **Step 1: Restaurar a tool do EF e os pacotes de design**

```bash
cd "C:/eSolution/Projetos/hackaton-fiap/hackaton-fiap-payments"
dotnet new tool-manifest 2>/dev/null || true
dotnet tool install dotnet-ef --version 8.0.* 2>/dev/null || dotnet tool restore
dotnet add src/HackatonFiap.Payments.Infrastructure package Microsoft.EntityFrameworkCore.Design --version 8.0.*
```

- [ ] **Step 2: Gerar a migration**

```bash
dotnet ef migrations add InitialCreate \
  --project src/HackatonFiap.Payments.Infrastructure \
  --startup-project src/HackatonFiap.Payments.API \
  --output-dir Migrations
```
Expected: cria `Migrations/<timestamp>_InitialCreate.cs` com a tabela `Payments` e o índice único em `DonationId`.

- [ ] **Step 3: Build + Commit**

```bash
dotnet build HackatonFiap.Payments.sln
git add src/HackatonFiap.Payments.Infrastructure/Migrations/ .config/dotnet-tools.json
git commit -m "feat(infra): migration InitialCreate (Payments + índice único DonationId)"
```

---

## Task 15: CI/CD e Kubernetes

**Files:**
- Modify/Create: `.github/workflows/ci.yml`
- Create: `.github/k8s/deployment.yaml`

- [ ] **Step 1: CI — build + testes + imagem (adaptar do boilerplate)**

`.github/workflows/ci.yml`:
```yaml
name: CI
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      - run: dotnet restore HackatonFiap.Payments.sln
      - run: dotnet build HackatonFiap.Payments.sln --no-restore -c Release
      - run: dotnet test HackatonFiap.Payments.sln --no-build -c Release --verbosity minimal
      - name: Build Docker image
        run: docker build -t hackatonfiap-payments:${{ github.sha }} .
```

- [ ] **Step 2: K8s — deployment com probes e annotations Prometheus (porta 8080)**

`.github/k8s/deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hackatonfiap-payments
  namespace: hackatonfiap
  labels:
    app: hackatonfiap-payments
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hackatonfiap-payments
  template:
    metadata:
      labels:
        app: hackatonfiap-payments
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/metrics"
        prometheus.io/port: "8080"
    spec:
      containers:
        - name: hackatonfiap-payments
          image: fcgregistry.azurecr.io/hackatonfiap-payments:latest
          ports:
            - containerPort: 8080
          envFrom:
            - secretRef:
                name: hackatonfiap-payments-secret
          resources:
            requests: { memory: "64Mi", cpu: "50m" }
            limits: { memory: "256Mi", cpu: "500m" }
          livenessProbe:
            httpGet: { path: /health, port: 8080 }
            initialDelaySeconds: 60
            periodSeconds: 15
          readinessProbe:
            httpGet: { path: /ready, port: 8080 }
            initialDelaySeconds: 40
            periodSeconds: 10
```

- [ ] **Step 3: Commit**

```bash
git add .github/
git commit -m "ci: build/test/imagem + manifest K8s (probes /health,/ready + Prometheus)"
```

---

## Task 16: Push e fechamento

- [ ] **Step 1: Configurar remote e fazer o push (forçando o GCM — ver memória de credenciais)**

```bash
cd "C:/eSolution/Projetos/hackaton-fiap/hackaton-fiap-payments"
git remote get-url origin 2>/dev/null || git remote add origin https://github.com/GabrielVeridico/hackaton-fiap-payments.git
git -c credential.helper= -c credential.helper=manager push -u origin main
```
Expected: push aceito (pode mostrar "Bypassed rule violations").

- [ ] **Step 2: Verificação final**

Run: `dotnet test --nologo`
Expected: todos os testes verdes. Conferir no GitHub que o conteúdo subiu.

---

## Self-Review (preenchido pelo autor do plano)

**Spec coverage:**
- Estrutura/renome (spec §3) → Task 1. ✅
- Domínio `Payment` (spec §4) → Tasks 3–4. ✅
- Eventos em inglês (spec §5) → Task 5. ✅
- Fluxo consumer→handler→publisher (spec §6) → Tasks 7, 10, 11. ✅
- CQRS (spec §7) → Tasks 7, 8. ✅
- Gateway mock determinístico (spec §8) → Task 6. ✅
- Idempotência por índice único (spec §9) → Tasks 7 (lógica) + 9 (índice). ✅
- Persistência/db-per-service (spec §10) → Tasks 9, 14. ✅
- Observabilidade (spec §11) → Tasks 12, 13. ✅
- Auth GestorONG (spec §12) → Task 13. ✅
- Testes (spec §13) → Tasks 4, 6, 7, 8. ✅
- CI/CD + K8s (spec §14) → Task 15. ✅

**Notas de execução:**
- `Telemetry`/`PaymentMetrics` vive na **Application** (`HackatonFiap.Payments.Application.Observability`) para evitar a Application depender da API; o `Program.cs` referencia o mesmo nome de `Meter`.
- Service Bus assume que tópicos/subscriptions (`donation-requested`/`payments`, `payment-result`) **já existem** no namespace (provisionamento é infra, fora do código).
- Follow-up (spec §16): atualizar Domain Events/PRD-06 para nomes em inglês e criar a doc de API em `docs/06 - APIs/`.
