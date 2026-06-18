# DonationAPI (HackatonFiap.Donations) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir a DonationAPI do Conexão Solidária (campanhas, doações com saga + consumer idempotente, e painel de transparência via read model), compatível com o contrato de eventos da PaymentAPI já implementada.

**Architecture:** Clean Architecture multi-projeto (Domain/Application/Infrastructure/API + UnitTests), espelhando `hackaton-fiap-payments`. CQRS manual com `Result<T>`, EF Core 8 (SQL Server, db-per-service), Azure Service Bus (publisher + consumer BackgroundService), read model em Cosmos DB com fallback in-memory em Development, OpenTelemetry/Prometheus + Serilog.

**Tech Stack:** .NET 8, EF Core 8.0.28, Azure.Messaging.ServiceBus 7.20.1, Microsoft.Azure.Cosmos 3.43.0, OpenTelemetry 1.9.x, Serilog, xUnit + NSubstitute + FluentAssertions.

**Spec:** `docs/superpowers/specs/2026-06-18-donationapi-design.md`

**Repo:** `hackaton-fiap-donations` (branch `main`, commit direto). **Autoria dos commits: GabrielVeridico, SEM trailer `Co-Authored-By` / Claude.** Os comandos `git commit` deste plano NÃO incluem co-author.

**Convenções:** código e rotas em inglês; roles pt-BR (`Doador`/`GestorONG`). Allman/K&R conforme o código existente do payments (que usa K&R com chave na mesma linha de tipo? — não: payments usa chave em nova linha para tipos/métodos; siga o estilo do arquivo de referência de cada camada). Sempre chaves, early returns.

---

## File Structure (decomposição)

```
hackaton-fiap-donations/
├── global.json                         # SDK 8.0, rollForward latestFeature
├── HackatonFiap.Donations.sln
├── Dockerfile
├── .config/dotnet-tools.json           # dotnet-ef 8.0.28
├── src/
│   ├── HackatonFiap.Donations.Domain/
│   │   ├── Common/Result.cs            # Result/Error (copiado do payments)
│   │   ├── Enums/CampaignStatus.cs
│   │   ├── Enums/CompletionReason.cs
│   │   ├── Enums/DonationStatus.cs
│   │   ├── Enums/PaymentMethod.cs      # MESMO contrato do payments (Pix/CreditCard/BankTransfer)
│   │   ├── ValueObjects/Period.cs
│   │   └── Entities/{Campaign,Donation,ProcessedEvent}.cs
│   ├── HackatonFiap.Donations.Application/
│   │   ├── Abstractions/{ICampaignRepository,IDonationRepository,IProcessedEventStore,IUnitOfWork,IEventPublisher,ICampaignReadStore,IClock}.cs
│   │   ├── ReadModels/CampaignReadModel.cs
│   │   ├── Errors/{CampaignErrors,DonationErrors}.cs
│   │   ├── Observability/DonationMetrics.cs
│   │   ├── IntegrationEvents/{DonationRequestedEvent,PaymentApprovedEvent,PaymentDeclinedEvent}.cs
│   │   ├── Campaigns/ (commands/queries + handlers + responses)
│   │   ├── Donations/ (commands/queries + handlers + responses)
│   │   └── Transparency/ (ListActiveCampaignsQuery + handler + response)
│   ├── HackatonFiap.Donations.Infrastructure/
│   │   ├── Persistence/ (DbContext, Configurations, Factory, Repositories, UnitOfWork)
│   │   ├── Messaging/ (MessagingJson, ServiceBusEventPublisher, NoOpEventPublisher, PaymentResultConsumer)
│   │   ├── ReadStore/ (InMemoryCampaignReadStore, CosmosCampaignReadStore)
│   │   ├── Time/SystemClock.cs
│   │   └── BackgroundServices/CampaignExpirationWorker.cs
│   └── HackatonFiap.Donations.API/
│       ├── Common/{HttpErrorMapping,ClaimsPrincipalExtensions}.cs
│       ├── Controllers/{CampaignsController,DonationsController,TransparencyController}.cs
│       ├── Program.cs
│       ├── appsettings.json / appsettings.Development.json
│       └── Properties/launchSettings.json
├── tests/HackatonFiap.Donations.UnitTests/
└── .github/workflows/ (ci atualizado) + k8s
```

---

## Task 1: Scaffold da solução e projetos

**Files:**
- Delete: `CatalogAPI/` (pasta inteira), `CatalogAPI.sln`
- Create: `global.json`, `HackatonFiap.Donations.sln`, os 5 `.csproj`, `.config/dotnet-tools.json`, `Dockerfile`

- [ ] **Step 1: Remover o boilerplate CatalogAPI**

```bash
cd "C:/eSolution/Projetos/hackaton-fiap/hackaton-fiap-donations"
rm -rf CatalogAPI CatalogAPI.sln
```

- [ ] **Step 2: Criar `global.json`**

```json
{
  "sdk": {
    "version": "8.0.0",
    "rollForward": "latestFeature"
  }
}
```

- [ ] **Step 3: Criar `.config/dotnet-tools.json`**

```json
{
  "version": 1,
  "isRoot": true,
  "tools": {
    "dotnet-ef": {
      "version": "8.0.28",
      "commands": [ "dotnet-ef" ],
      "rollForward": false
    }
  }
}
```

- [ ] **Step 4: Criar os projetos e a solução via CLI**

```bash
cd "C:/eSolution/Projetos/hackaton-fiap/hackaton-fiap-donations"
dotnet new sln -n HackatonFiap.Donations
dotnet new classlib -o src/HackatonFiap.Donations.Domain -f net8.0
dotnet new classlib -o src/HackatonFiap.Donations.Application -f net8.0
dotnet new classlib -o src/HackatonFiap.Donations.Infrastructure -f net8.0
dotnet new webapi -o src/HackatonFiap.Donations.API -f net8.0 --no-openapi false
dotnet new xunit -o tests/HackatonFiap.Donations.UnitTests -f net8.0
rm -f src/HackatonFiap.Donations.Domain/Class1.cs src/HackatonFiap.Donations.Application/Class1.cs src/HackatonFiap.Donations.Infrastructure/Class1.cs tests/HackatonFiap.Donations.UnitTests/UnitTest1.cs
dotnet sln add src/HackatonFiap.Donations.Domain src/HackatonFiap.Donations.Application src/HackatonFiap.Donations.Infrastructure src/HackatonFiap.Donations.API tests/HackatonFiap.Donations.UnitTests
```

- [ ] **Step 5: Substituir os 5 `.csproj` pelo conteúdo exato abaixo**

`src/HackatonFiap.Donations.Domain/HackatonFiap.Donations.Domain.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
</Project>
```

`src/HackatonFiap.Donations.Application/HackatonFiap.Donations.Application.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
  <ItemGroup>
    <ProjectReference Include="..\HackatonFiap.Donations.Domain\HackatonFiap.Donations.Domain.csproj" />
  </ItemGroup>
</Project>
```

`src/HackatonFiap.Donations.Infrastructure/HackatonFiap.Donations.Infrastructure.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Azure.Messaging.ServiceBus" Version="7.20.1" />
    <PackageReference Include="Microsoft.Azure.Cosmos" Version="3.43.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.28" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="8.0.28">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.28" />
    <PackageReference Include="Microsoft.Extensions.Hosting.Abstractions" Version="8.0.*" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\HackatonFiap.Donations.Application\HackatonFiap.Donations.Application.csproj" />
    <ProjectReference Include="..\HackatonFiap.Donations.Domain\HackatonFiap.Donations.Domain.csproj" />
  </ItemGroup>
</Project>
```

`src/HackatonFiap.Donations.API/HackatonFiap.Donations.API.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="8.0.19" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="8.0.28">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="Microsoft.Extensions.Diagnostics.HealthChecks.EntityFrameworkCore" Version="8.0.28" />
    <PackageReference Include="OpenTelemetry.Exporter.Prometheus.AspNetCore" Version="1.9.0-beta.2" />
    <PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.9.0" />
    <PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.9.0" />
    <PackageReference Include="OpenTelemetry.Instrumentation.Runtime" Version="1.9.0" />
    <PackageReference Include="Serilog.AspNetCore" Version="8.0.*" />
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.6.2" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\HackatonFiap.Donations.Application\HackatonFiap.Donations.Application.csproj" />
    <ProjectReference Include="..\HackatonFiap.Donations.Infrastructure\HackatonFiap.Donations.Infrastructure.csproj" />
  </ItemGroup>
</Project>
```

`tests/HackatonFiap.Donations.UnitTests/HackatonFiap.Donations.UnitTests.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="coverlet.collector" Version="6.0.0" />
    <PackageReference Include="FluentAssertions" Version="8.10.0" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
    <PackageReference Include="NSubstitute" Version="5.3.0" />
    <PackageReference Include="xunit" Version="2.5.3" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.5.3" />
  </ItemGroup>
  <ItemGroup>
    <Using Include="Xunit" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\..\src\HackatonFiap.Donations.Domain\HackatonFiap.Donations.Domain.csproj" />
    <ProjectReference Include="..\..\src\HackatonFiap.Donations.Application\HackatonFiap.Donations.Application.csproj" />
    <ProjectReference Include="..\..\src\HackatonFiap.Donations.Infrastructure\HackatonFiap.Donations.Infrastructure.csproj" />
  </ItemGroup>
</Project>
```

- [ ] **Step 6: Criar `Dockerfile`**

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["global.json", "./"]
COPY ["HackatonFiap.Donations.sln", "./"]
COPY ["src/HackatonFiap.Donations.Domain/HackatonFiap.Donations.Domain.csproj", "src/HackatonFiap.Donations.Domain/"]
COPY ["src/HackatonFiap.Donations.Application/HackatonFiap.Donations.Application.csproj", "src/HackatonFiap.Donations.Application/"]
COPY ["src/HackatonFiap.Donations.Infrastructure/HackatonFiap.Donations.Infrastructure.csproj", "src/HackatonFiap.Donations.Infrastructure/"]
COPY ["src/HackatonFiap.Donations.API/HackatonFiap.Donations.API.csproj", "src/HackatonFiap.Donations.API/"]
RUN dotnet restore "src/HackatonFiap.Donations.API/HackatonFiap.Donations.API.csproj"
COPY . .
RUN dotnet publish "src/HackatonFiap.Donations.API/HackatonFiap.Donations.API.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "HackatonFiap.Donations.API.dll"]
```

- [ ] **Step 7: Restaurar tools e buildar a solução vazia**

Run:
```bash
dotnet tool restore
dotnet build HackatonFiap.Donations.sln
```
Expected: build PASS (5 projetos compilam; API ainda com template padrão).

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "chore: scaffold da solução HackatonFiap.Donations (remove CatalogAPI)"
```

---

## Task 2: Domain — Result/Error + enums

**Files:**
- Create: `src/HackatonFiap.Donations.Domain/Common/Result.cs`
- Create: `src/HackatonFiap.Donations.Domain/Enums/{CampaignStatus,CompletionReason,DonationStatus,PaymentMethod}.cs`

- [ ] **Step 1: Criar `Common/Result.cs`** (idêntico ao payments, namespace adaptado)

```csharp
namespace HackatonFiap.Donations.Domain.Common;

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

- [ ] **Step 2: Criar os enums**

`Enums/CampaignStatus.cs`:
```csharp
namespace HackatonFiap.Donations.Domain.Enums;

public enum CampaignStatus
{
    Active = 0,
    Completed = 1,
    Cancelled = 2
}
```

`Enums/CompletionReason.cs`:
```csharp
namespace HackatonFiap.Donations.Domain.Enums;

public enum CompletionReason
{
    GoalReached = 0,
    ManuallyClosed = 1,
    Expired = 2
}
```

`Enums/DonationStatus.cs`:
```csharp
namespace HackatonFiap.Donations.Domain.Enums;

public enum DonationStatus
{
    Pending = 0,
    Approved = 1,
    Declined = 2
}
```

`Enums/PaymentMethod.cs` (**mesmos nomes/ordem do payments — o contrato JSON depende disso**):
```csharp
namespace HackatonFiap.Donations.Domain.Enums;

public enum PaymentMethod
{
    Pix = 0,
    CreditCard = 1,
    BankTransfer = 2
}
```

- [ ] **Step 3: Build**

Run: `dotnet build src/HackatonFiap.Donations.Domain`
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
git add src/HackatonFiap.Donations.Domain
git commit -m "feat(domain): Result/Error e enums (CampaignStatus, CompletionReason, DonationStatus, PaymentMethod)"
```

---

## Task 3: Domain — Value Object `Period` (TDD)

**Files:**
- Create: `src/HackatonFiap.Donations.Domain/ValueObjects/Period.cs`
- Test: `tests/HackatonFiap.Donations.UnitTests/Domain/PeriodTests.cs`

`Period` é um VO imutável com factory `Create` que retorna `Result<Period>` (valida `EndDate >= StartDate`). Construtor privado sem parâmetros para o EF (owned type).

- [ ] **Step 1: Escrever o teste que falha**

`tests/HackatonFiap.Donations.UnitTests/Domain/PeriodTests.cs`:
```csharp
using FluentAssertions;
using HackatonFiap.Donations.Domain.ValueObjects;

namespace HackatonFiap.Donations.UnitTests.Domain;

public class PeriodTests
{
    [Fact]
    public void Create_with_end_after_start_succeeds()
    {
        var start = new DateTime(2026, 06, 01, 0, 0, 0, DateTimeKind.Utc);
        var end = new DateTime(2026, 07, 01, 0, 0, 0, DateTimeKind.Utc);

        var result = Period.Create(start, end);

        result.IsSuccess.Should().BeTrue();
        result.Value.StartDate.Should().Be(start);
        result.Value.EndDate.Should().Be(end);
    }

    [Fact]
    public void Create_with_end_before_start_fails()
    {
        var start = new DateTime(2026, 07, 01, 0, 0, 0, DateTimeKind.Utc);
        var end = new DateTime(2026, 06, 01, 0, 0, 0, DateTimeKind.Utc);

        var result = Period.Create(start, end);

        result.IsFailure.Should().BeTrue();
        result.Error.Code.Should().Be("Period.EndBeforeStart");
    }
}
```

- [ ] **Step 2: Rodar e confirmar falha de compilação/teste**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~PeriodTests"`
Expected: FAIL (Period não existe).

- [ ] **Step 3: Implementar `ValueObjects/Period.cs`**

```csharp
using HackatonFiap.Donations.Domain.Common;

namespace HackatonFiap.Donations.Domain.ValueObjects;

public sealed class Period
{
    private Period() { } // EF

    private Period(DateTime startDate, DateTime endDate)
    {
        StartDate = startDate;
        EndDate = endDate;
    }

    public DateTime StartDate { get; private set; }
    public DateTime EndDate { get; private set; }

    public static readonly Error EndBeforeStart =
        new("Period.EndBeforeStart", "A data fim não pode ser anterior à data início.");

    public static Result<Period> Create(DateTime startDate, DateTime endDate)
    {
        if (endDate < startDate)
        {
            return Result.Failure<Period>(EndBeforeStart);
        }

        return Result.Success(new Period(startDate, endDate));
    }

    public bool Contains(DateTime instant) => instant >= StartDate && instant <= EndDate;
}
```

- [ ] **Step 4: Rodar e confirmar PASS**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~PeriodTests"`
Expected: PASS (2 testes).

- [ ] **Step 5: Commit**

```bash
git add src/HackatonFiap.Donations.Domain/ValueObjects/Period.cs tests/HackatonFiap.Donations.UnitTests/Domain/PeriodTests.cs
git commit -m "feat(domain): VO Period com validação de intervalo"
```

---

## Task 4: Domain — entidade `Campaign` (TDD)

**Files:**
- Create: `src/HackatonFiap.Donations.Domain/Entities/Campaign.cs`
- Test: `tests/HackatonFiap.Donations.UnitTests/Domain/CampaignTests.cs`

`Campaign.Create` recebe `utcNow` (o handler passará `IClock.UtcNow`) para validar `EndDate` não-no-passado. Validações retornam `Result` (mapeadas a HTTP depois). Transições (`Complete`/`Cancel`) retornam `Result`; `AddRaised` é `void`.

- [ ] **Step 1: Escrever o teste que falha**

`tests/HackatonFiap.Donations.UnitTests/Domain/CampaignTests.cs`:
```csharp
using FluentAssertions;
using HackatonFiap.Donations.Domain.Entities;
using HackatonFiap.Donations.Domain.Enums;
using Xunit;

namespace HackatonFiap.Donations.UnitTests.Domain;

public class CampaignTests
{
    private static readonly DateTime Now = new(2026, 06, 18, 12, 0, 0, DateTimeKind.Utc);

    private static Campaign NewActive(decimal goal = 1000m)
        => Campaign.Create("Inverno Solidário", "Agasalhos", Now.AddDays(-1), Now.AddDays(30), goal, Guid.NewGuid(), Now).Value;

    [Fact]
    public void Create_starts_active_with_zero_raised()
    {
        var campaign = NewActive();

        campaign.Status.Should().Be(CampaignStatus.Active);
        campaign.AmountRaised.Should().Be(0m);
        campaign.CompletionReason.Should().BeNull();
        campaign.Id.Should().NotBe(Guid.Empty);
    }

    [Fact]
    public void Create_with_non_positive_goal_fails()
    {
        var result = Campaign.Create("t", "d", Now, Now.AddDays(10), 0m, Guid.NewGuid(), Now);

        result.IsFailure.Should().BeTrue();
        result.Error.Code.Should().Be("Campaign.GoalMustBePositive");
    }

    [Fact]
    public void Create_with_end_date_in_past_fails()
    {
        var result = Campaign.Create("t", "d", Now.AddDays(-10), Now.AddDays(-1), 100m, Guid.NewGuid(), Now);

        result.IsFailure.Should().BeTrue();
        result.Error.Code.Should().Be("Campaign.EndDateInPast");
    }

    [Fact]
    public void Create_with_blank_title_fails()
    {
        var result = Campaign.Create("  ", "d", Now, Now.AddDays(10), 100m, Guid.NewGuid(), Now);

        result.IsFailure.Should().BeTrue();
        result.Error.Code.Should().Be("Campaign.TitleRequired");
    }

    [Fact]
    public void AddRaised_increments_amount()
    {
        var campaign = NewActive();

        campaign.AddRaised(250m);
        campaign.AddRaised(100m);

        campaign.AmountRaised.Should().Be(350m);
    }

    [Fact]
    public void Complete_from_active_sets_reason()
    {
        var campaign = NewActive();

        var result = campaign.Complete(CompletionReason.GoalReached);

        result.IsSuccess.Should().BeTrue();
        campaign.Status.Should().Be(CampaignStatus.Completed);
        campaign.CompletionReason.Should().Be(CompletionReason.GoalReached);
    }

    [Fact]
    public void Complete_when_terminal_fails()
    {
        var campaign = NewActive();
        campaign.Cancel();

        var result = campaign.Complete(CompletionReason.Expired);

        result.IsFailure.Should().BeTrue();
        result.Error.Code.Should().Be("Campaign.InvalidStatusTransition");
    }

    [Fact]
    public void Cancel_from_active_succeeds()
    {
        var campaign = NewActive();

        var result = campaign.Cancel();

        result.IsSuccess.Should().BeTrue();
        campaign.Status.Should().Be(CampaignStatus.Cancelled);
    }
}
```

- [ ] **Step 2: Rodar e confirmar falha**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~CampaignTests"`
Expected: FAIL (Campaign não existe).

- [ ] **Step 3: Implementar `Entities/Campaign.cs`**

```csharp
using HackatonFiap.Donations.Domain.Common;
using HackatonFiap.Donations.Domain.Enums;
using HackatonFiap.Donations.Domain.ValueObjects;

namespace HackatonFiap.Donations.Domain.Entities;

public class Campaign
{
    private Campaign() { } // EF

    private Campaign(string title, string description, Period period, decimal goal, Guid createdById, DateTime utcNow)
    {
        Id = Guid.NewGuid();
        Title = title;
        Description = description;
        Period = period;
        Goal = goal;
        CreatedById = createdById;
        AmountRaised = 0m;
        Status = CampaignStatus.Active;
        CompletionReason = null;
        CreatedAt = utcNow;
        UpdatedAt = utcNow;
    }

    public Guid Id { get; private set; }
    public string Title { get; private set; } = string.Empty;
    public string Description { get; private set; } = string.Empty;
    public Period Period { get; private set; } = null!;
    public decimal Goal { get; private set; }
    public decimal AmountRaised { get; private set; }
    public CampaignStatus Status { get; private set; }
    public CompletionReason? CompletionReason { get; private set; }
    public Guid CreatedById { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime UpdatedAt { get; private set; }

    public static readonly Error TitleRequired = new("Campaign.TitleRequired", "O título é obrigatório.");
    public static readonly Error GoalMustBePositive = new("Campaign.GoalMustBePositive", "A meta financeira deve ser maior que zero.");
    public static readonly Error EndDateInPast = new("Campaign.EndDateInPast", "A data fim não pode estar no passado.");
    public static readonly Error InvalidStatusTransition = new("Campaign.InvalidStatusTransition", "Transição de status inválida: a campanha não está ativa.");

    public static Result<Campaign> Create(string title, string description, DateTime startDate, DateTime endDate,
        decimal goal, Guid createdById, DateTime utcNow)
    {
        if (string.IsNullOrWhiteSpace(title))
        {
            return Result.Failure<Campaign>(TitleRequired);
        }

        if (goal <= 0m)
        {
            return Result.Failure<Campaign>(GoalMustBePositive);
        }

        if (endDate < utcNow)
        {
            return Result.Failure<Campaign>(EndDateInPast);
        }

        var period = Period.Create(startDate, endDate);
        if (period.IsFailure)
        {
            return Result.Failure<Campaign>(period.Error);
        }

        return Result.Success(new Campaign(title.Trim(), description ?? string.Empty, period.Value, goal, createdById, utcNow));
    }

    public Result Update(string title, string description, DateTime startDate, DateTime endDate, decimal goal, DateTime utcNow)
    {
        if (Status != CampaignStatus.Active)
        {
            return Result.Failure(InvalidStatusTransition);
        }

        if (string.IsNullOrWhiteSpace(title))
        {
            return Result.Failure(TitleRequired);
        }

        if (goal <= 0m)
        {
            return Result.Failure(GoalMustBePositive);
        }

        if (endDate < utcNow)
        {
            return Result.Failure(EndDateInPast);
        }

        var period = Period.Create(startDate, endDate);
        if (period.IsFailure)
        {
            return Result.Failure(period.Error);
        }

        Title = title.Trim();
        Description = description ?? string.Empty;
        Period = period.Value;
        Goal = goal;
        UpdatedAt = utcNow;
        return Result.Success();
    }

    public void AddRaised(decimal amount)
    {
        if (amount <= 0m)
        {
            return;
        }

        AmountRaised += amount;
    }

    public Result Complete(CompletionReason reason)
    {
        if (Status != CampaignStatus.Active)
        {
            return Result.Failure(InvalidStatusTransition);
        }

        Status = CampaignStatus.Completed;
        CompletionReason = reason;
        UpdatedAt = DateTime.UtcNow;
        return Result.Success();
    }

    public Result Cancel()
    {
        if (Status != CampaignStatus.Active)
        {
            return Result.Failure(InvalidStatusTransition);
        }

        Status = CampaignStatus.Cancelled;
        UpdatedAt = DateTime.UtcNow;
        return Result.Success();
    }
}
```

> Nota: `Complete`/`Cancel` usam `DateTime.UtcNow` para `UpdatedAt` (não é regra de negócio testável — apenas timestamp). A validação temporal de criação/edição usa o `utcNow` injetado.

- [ ] **Step 4: Rodar e confirmar PASS**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~CampaignTests"`
Expected: PASS (8 testes).

- [ ] **Step 5: Commit**

```bash
git add src/HackatonFiap.Donations.Domain/Entities/Campaign.cs tests/HackatonFiap.Donations.UnitTests/Domain/CampaignTests.cs
git commit -m "feat(domain): entidade Campaign com ciclo de vida e invariantes"
```

---

## Task 5: Domain — entidade `Donation` (TDD)

**Files:**
- Create: `src/HackatonFiap.Donations.Domain/Entities/Donation.cs`
- Test: `tests/HackatonFiap.Donations.UnitTests/Domain/DonationTests.cs`

- [ ] **Step 1: Escrever o teste que falha**

`tests/HackatonFiap.Donations.UnitTests/Domain/DonationTests.cs`:
```csharp
using FluentAssertions;
using HackatonFiap.Donations.Domain.Entities;
using HackatonFiap.Donations.Domain.Enums;
using Xunit;

namespace HackatonFiap.Donations.UnitTests.Domain;

public class DonationTests
{
    private static Donation NewPending(decimal amount = 50m)
        => Donation.Create(Guid.NewGuid(), amount, PaymentMethod.Pix, Guid.NewGuid(), "donor@example.com", "Donor").Value;

    [Fact]
    public void Create_starts_pending()
    {
        var donation = NewPending();

        donation.Status.Should().Be(DonationStatus.Pending);
        donation.Id.Should().NotBe(Guid.Empty);
        donation.ProcessedAt.Should().BeNull();
    }

    [Fact]
    public void Create_with_non_positive_amount_fails()
    {
        var result = Donation.Create(Guid.NewGuid(), 0m, PaymentMethod.Pix, Guid.NewGuid(), "d@e.com", "D");

        result.IsFailure.Should().BeTrue();
        result.Error.Code.Should().Be("Donation.AmountMustBePositive");
    }

    [Fact]
    public void Approve_sets_status_and_processed_at()
    {
        var donation = NewPending();

        donation.Approve();

        donation.Status.Should().Be(DonationStatus.Approved);
        donation.ProcessedAt.Should().NotBeNull();
    }

    [Fact]
    public void Decline_sets_status_and_reason()
    {
        var donation = NewPending();

        donation.Decline("centavos ,99");

        donation.Status.Should().Be(DonationStatus.Declined);
        donation.DeclineReason.Should().Be("centavos ,99");
        donation.ProcessedAt.Should().NotBeNull();
    }

    [Fact]
    public void Approve_when_not_pending_throws()
    {
        var donation = NewPending();
        donation.Approve();

        var act = () => donation.Approve();

        act.Should().Throw<InvalidOperationException>();
    }
}
```

- [ ] **Step 2: Rodar e confirmar falha**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~DonationTests"`
Expected: FAIL.

- [ ] **Step 3: Implementar `Entities/Donation.cs`**

```csharp
using HackatonFiap.Donations.Domain.Common;
using HackatonFiap.Donations.Domain.Enums;

namespace HackatonFiap.Donations.Domain.Entities;

public class Donation
{
    private Donation() { } // EF

    private Donation(Guid campaignId, decimal amount, PaymentMethod method, Guid donorId, string donorEmail, string donorName)
    {
        Id = Guid.NewGuid();
        CampaignId = campaignId;
        Amount = amount;
        Method = method;
        DonorId = donorId;
        DonorEmail = donorEmail;
        DonorName = donorName;
        Status = DonationStatus.Pending;
        CreatedAt = DateTime.UtcNow;
    }

    public Guid Id { get; private set; }
    public Guid CampaignId { get; private set; }
    public decimal Amount { get; private set; }
    public PaymentMethod Method { get; private set; }
    public DonationStatus Status { get; private set; }
    public Guid DonorId { get; private set; }
    public string DonorEmail { get; private set; } = string.Empty;
    public string DonorName { get; private set; } = string.Empty;
    public string? DeclineReason { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? ProcessedAt { get; private set; }

    public static readonly Error AmountMustBePositive = new("Donation.AmountMustBePositive", "O valor da doação deve ser maior que zero.");
    public static readonly Error CampaignRequired = new("Donation.CampaignRequired", "A campanha é obrigatória.");

    public static Result<Donation> Create(Guid campaignId, decimal amount, PaymentMethod method,
        Guid donorId, string donorEmail, string donorName)
    {
        if (campaignId == Guid.Empty)
        {
            return Result.Failure<Donation>(CampaignRequired);
        }

        if (amount <= 0m)
        {
            return Result.Failure<Donation>(AmountMustBePositive);
        }

        return Result.Success(new Donation(campaignId, amount, method, donorId, donorEmail ?? string.Empty, donorName ?? string.Empty));
    }

    public void Approve()
    {
        if (Status != DonationStatus.Pending)
        {
            throw new InvalidOperationException($"Não é possível aprovar doação com status {Status}.");
        }

        Status = DonationStatus.Approved;
        ProcessedAt = DateTime.UtcNow;
    }

    public void Decline(string reason)
    {
        if (Status != DonationStatus.Pending)
        {
            throw new InvalidOperationException($"Não é possível recusar doação com status {Status}.");
        }

        Status = DonationStatus.Declined;
        DeclineReason = string.IsNullOrWhiteSpace(reason) ? "Pagamento recusado." : reason;
        ProcessedAt = DateTime.UtcNow;
    }
}
```

- [ ] **Step 4: Rodar e confirmar PASS**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~DonationTests"`
Expected: PASS (5 testes).

- [ ] **Step 5: Commit**

```bash
git add src/HackatonFiap.Donations.Domain/Entities/Donation.cs tests/HackatonFiap.Donations.UnitTests/Domain/DonationTests.cs
git commit -m "feat(domain): entidade Donation (Pending -> Approved/Declined)"
```

---

## Task 6: Domain — `ProcessedEvent` (inbox)

**Files:**
- Create: `src/HackatonFiap.Donations.Domain/Entities/ProcessedEvent.cs`

- [ ] **Step 1: Implementar `Entities/ProcessedEvent.cs`**

```csharp
namespace HackatonFiap.Donations.Domain.Entities;

public class ProcessedEvent
{
    private ProcessedEvent() { } // EF

    public ProcessedEvent(Guid donationId, DateTime processedAt)
    {
        DonationId = donationId;
        ProcessedAt = processedAt;
    }

    public Guid DonationId { get; private set; }
    public DateTime ProcessedAt { get; private set; }
}
```

- [ ] **Step 2: Build**

Run: `dotnet build src/HackatonFiap.Donations.Domain`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
git add src/HackatonFiap.Donations.Domain/Entities/ProcessedEvent.cs
git commit -m "feat(domain): ProcessedEvent (inbox de idempotência)"
```

---

## Task 7: Application — abstractions, errors, métricas, read model, integration events

**Files (todos em `src/HackatonFiap.Donations.Application/`):**
- Create: `Abstractions/ICampaignRepository.cs`, `Abstractions/IDonationRepository.cs`, `Abstractions/IProcessedEventStore.cs`, `Abstractions/IUnitOfWork.cs`, `Abstractions/IEventPublisher.cs`, `Abstractions/ICampaignReadStore.cs`, `Abstractions/IClock.cs`
- Create: `ReadModels/CampaignReadModel.cs`
- Create: `Errors/DonationErrors.cs`
- Create: `Observability/DonationMetrics.cs`
- Create: `IntegrationEvents/DonationRequestedEvent.cs`, `IntegrationEvents/PaymentApprovedEvent.cs`, `IntegrationEvents/PaymentDeclinedEvent.cs`

Sem testes (interfaces/records/constantes). Os erros de `Campaign`/`Donation` que são invariantes ficam nas entidades (Task 4/5); aqui só os erros de **aplicação** específicos da doação (campanha não encontrada/inativa/fora do período/doação não encontrada).

- [ ] **Step 1: Criar as abstrações**

`Abstractions/IClock.cs`:
```csharp
namespace HackatonFiap.Donations.Application.Abstractions;

public interface IClock
{
    DateTime UtcNow { get; }
}
```

`Abstractions/IUnitOfWork.cs`:
```csharp
namespace HackatonFiap.Donations.Application.Abstractions;

public interface IUnitOfWork
{
    Task SaveChangesAsync(CancellationToken cancellationToken = default);
}
```

`Abstractions/ICampaignRepository.cs`:
```csharp
using HackatonFiap.Donations.Domain.Entities;

namespace HackatonFiap.Donations.Application.Abstractions;

public interface ICampaignRepository
{
    Task<Campaign?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task AddAsync(Campaign campaign, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<Campaign>> ListAsync(CancellationToken cancellationToken = default);
    Task<IReadOnlyList<Campaign>> ListActiveExpiredAsync(DateTime utcNow, CancellationToken cancellationToken = default);
}
```

`Abstractions/IDonationRepository.cs`:
```csharp
using HackatonFiap.Donations.Domain.Entities;

namespace HackatonFiap.Donations.Application.Abstractions;

public interface IDonationRepository
{
    Task<Donation?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task AddAsync(Donation donation, CancellationToken cancellationToken = default);
}
```

`Abstractions/IProcessedEventStore.cs`:
```csharp
using HackatonFiap.Donations.Domain.Entities;

namespace HackatonFiap.Donations.Application.Abstractions;

public interface IProcessedEventStore
{
    Task<bool> ExistsAsync(Guid donationId, CancellationToken cancellationToken = default);
    Task AddAsync(ProcessedEvent processedEvent, CancellationToken cancellationToken = default);
}
```

`Abstractions/IEventPublisher.cs`:
```csharp
namespace HackatonFiap.Donations.Application.Abstractions;

public interface IEventPublisher
{
    Task PublishAsync<TEvent>(TEvent integrationEvent, string subject, CancellationToken cancellationToken = default)
        where TEvent : notnull;
}
```

`Abstractions/ICampaignReadStore.cs`:
```csharp
using HackatonFiap.Donations.Application.ReadModels;

namespace HackatonFiap.Donations.Application.Abstractions;

public interface ICampaignReadStore
{
    Task UpsertAsync(CampaignReadModel campaign, CancellationToken cancellationToken = default);
    Task<IReadOnlyList<CampaignReadModel>> ListActiveAsync(CancellationToken cancellationToken = default);
}
```

- [ ] **Step 2: Criar o read model**

`ReadModels/CampaignReadModel.cs`:
```csharp
namespace HackatonFiap.Donations.Application.ReadModels;

public sealed record CampaignReadModel(
    Guid Id,
    string Title,
    decimal Goal,
    decimal AmountRaised,
    string Status);
```

- [ ] **Step 3: Criar os erros de aplicação da doação**

`Errors/DonationErrors.cs`:
```csharp
using HackatonFiap.Donations.Domain.Common;

namespace HackatonFiap.Donations.Application.Errors;

public static class DonationErrors
{
    public static readonly Error CampaignNotFound =
        new("Donation.CampaignNotFound", "Campanha informada não existe.");
    public static readonly Error CampaignNotActive =
        new("Donation.CampaignNotActive", "A campanha não está ativa.");
    public static readonly Error OutsidePeriod =
        new("Donation.OutsidePeriod", "A doação está fora do período da campanha.");
    public static readonly Error NotFound =
        new("Donation.NotFound", "Doação não encontrada.");
}
```

> Os códigos `Donation.CampaignNotFound`/`CampaignNotActive`/`OutsidePeriod` mapeiam para **422** (PRD-06 §8). `Donation.NotFound` → 404. Códigos de `Campaign.*` (invariantes) → 400, exceto `Campaign.InvalidStatusTransition` → 409 e `Campaign.NotFound` → 404 (ver Task 17).

- [ ] **Step 4: Criar as métricas**

`Observability/DonationMetrics.cs`:
```csharp
using System.Diagnostics.Metrics;

namespace HackatonFiap.Donations.Application.Observability;

public static class DonationMetrics
{
    public const string ServiceName = "HackatonFiap.Donations";

    private static readonly Meter Meter = new(ServiceName);

    public static readonly Counter<long> DonationsReceived =
        Meter.CreateCounter<long>("donations_received_total", description: "Total de intenções de doação recebidas.");
    public static readonly Counter<long> DonationsApproved =
        Meter.CreateCounter<long>("donations_approved_total", description: "Total de doações aprovadas/consolidadas.");
    public static readonly Counter<long> DonationsDeclined =
        Meter.CreateCounter<long>("donations_declined_total", description: "Total de doações recusadas.");
    public static readonly Counter<long> CampaignsCompleted =
        Meter.CreateCounter<long>("campaigns_completed_total", description: "Total de campanhas concluídas.");
    public static readonly Counter<double> AmountRaised =
        Meter.CreateCounter<double>("amount_raised_total", description: "Valor total consolidado nas campanhas.");
}
```

- [ ] **Step 5: Criar os integration events** (contrato idêntico ao consumido/publicado pela PaymentAPI)

`IntegrationEvents/DonationRequestedEvent.cs`:
```csharp
using HackatonFiap.Donations.Domain.Enums;

namespace HackatonFiap.Donations.Application.IntegrationEvents;

public sealed record DonationRequestedEvent(
    Guid DonationId,
    Guid CampaignId,
    decimal Amount,
    PaymentMethod PaymentMethod,
    Guid DonorId,
    string DonorEmail,
    string DonorName);
```

`IntegrationEvents/PaymentApprovedEvent.cs`:
```csharp
namespace HackatonFiap.Donations.Application.IntegrationEvents;

public sealed record PaymentApprovedEvent(
    Guid DonationId,
    Guid CampaignId,
    decimal Amount,
    Guid PaymentId,
    Guid DonorId,
    string DonorEmail,
    string DonorName);
```

`IntegrationEvents/PaymentDeclinedEvent.cs`:
```csharp
namespace HackatonFiap.Donations.Application.IntegrationEvents;

public sealed record PaymentDeclinedEvent(
    Guid DonationId,
    Guid CampaignId,
    string Reason,
    decimal Amount,
    Guid DonorId,
    string DonorEmail,
    string DonorName);
```

- [ ] **Step 6: Build**

Run: `dotnet build src/HackatonFiap.Donations.Application`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
git add src/HackatonFiap.Donations.Application
git commit -m "feat(application): abstrações, erros, métricas, read model e integration events"
```

---

## Task 8: Application — handlers de Campanha (TDD)

**Files (em `src/HackatonFiap.Donations.Application/Campaigns/`):**
- Create: `CampaignResponse.cs`, `CampaignStatusAction.cs`
- Create: `CreateCampaign/CreateCampaignCommand.cs` (+ `...Handler.cs`)
- Create: `UpdateCampaign/UpdateCampaignCommand.cs` (+ Handler)
- Create: `ChangeCampaignStatus/ChangeCampaignStatusCommand.cs` (+ Handler)
- Create: `GetCampaignById/GetCampaignByIdQuery.cs` (+ Handler)
- Create: `ListCampaigns/ListCampaignsQuery.cs` (+ Handler)
- Test: `tests/.../Application/CampaignHandlersTests.cs`

Todos os handlers de escrita projetam no read store após o `SaveChanges` (RN04.12).

- [ ] **Step 1: Criar os tipos compartilhados**

`Campaigns/CampaignStatusAction.cs`:
```csharp
namespace HackatonFiap.Donations.Application.Campaigns;

public enum CampaignStatusAction
{
    Close = 0,   // encerramento manual -> Completed / ManuallyClosed
    Cancel = 1   // cancelamento        -> Cancelled
}
```

`Campaigns/CampaignResponse.cs`:
```csharp
namespace HackatonFiap.Donations.Application.Campaigns;

public sealed record CampaignResponse(
    Guid Id,
    string Title,
    string Description,
    DateTime StartDate,
    DateTime EndDate,
    decimal Goal,
    decimal AmountRaised,
    string Status,
    string? CompletionReason,
    DateTime CreatedAt,
    DateTime UpdatedAt);
```

- [ ] **Step 2: Escrever os testes que falham**

`tests/HackatonFiap.Donations.UnitTests/Application/CampaignHandlersTests.cs`:
```csharp
using FluentAssertions;
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.Campaigns;
using HackatonFiap.Donations.Application.Campaigns.ChangeCampaignStatus;
using HackatonFiap.Donations.Application.Campaigns.CreateCampaign;
using HackatonFiap.Donations.Application.ReadModels;
using HackatonFiap.Donations.Domain.Entities;
using HackatonFiap.Donations.Domain.Enums;
using NSubstitute;
using Xunit;

namespace HackatonFiap.Donations.UnitTests.Application;

public class CampaignHandlersTests
{
    private static readonly DateTime Now = new(2026, 06, 18, 12, 0, 0, DateTimeKind.Utc);

    private readonly ICampaignRepository _repository = Substitute.For<ICampaignRepository>();
    private readonly IUnitOfWork _uow = Substitute.For<IUnitOfWork>();
    private readonly ICampaignReadStore _readStore = Substitute.For<ICampaignReadStore>();
    private readonly IClock _clock = Substitute.For<IClock>();

    public CampaignHandlersTests() => _clock.UtcNow.Returns(Now);

    [Fact]
    public async Task Create_persists_and_projects_to_read_store()
    {
        var handler = new CreateCampaignCommandHandler(_repository, _uow, _readStore, _clock);
        var command = new CreateCampaignCommand("Inverno", "Agasalhos", Now, Now.AddDays(30), 1000m, Guid.NewGuid());

        var result = await handler.Handle(command, CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        await _repository.Received(1).AddAsync(Arg.Any<Campaign>(), Arg.Any<CancellationToken>());
        await _uow.Received(1).SaveChangesAsync(Arg.Any<CancellationToken>());
        await _readStore.Received(1).UpsertAsync(
            Arg.Is<CampaignReadModel>(c => c.Status == "Active" && c.Goal == 1000m),
            Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task Create_with_invalid_goal_fails_without_persisting()
    {
        var handler = new CreateCampaignCommandHandler(_repository, _uow, _readStore, _clock);
        var command = new CreateCampaignCommand("Inverno", "Agasalhos", Now, Now.AddDays(30), 0m, Guid.NewGuid());

        var result = await handler.Handle(command, CancellationToken.None);

        result.IsFailure.Should().BeTrue();
        result.Error.Code.Should().Be("Campaign.GoalMustBePositive");
        await _repository.DidNotReceive().AddAsync(Arg.Any<Campaign>(), Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task ChangeStatus_close_completes_with_manually_closed()
    {
        var campaign = Campaign.Create("t", "d", Now.AddDays(-1), Now.AddDays(10), 100m, Guid.NewGuid(), Now).Value;
        _repository.GetByIdAsync(campaign.Id).Returns(campaign);
        var handler = new ChangeCampaignStatusCommandHandler(_repository, _uow, _readStore);

        var result = await handler.Handle(new ChangeCampaignStatusCommand(campaign.Id, CampaignStatusAction.Close), CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        campaign.Status.Should().Be(CampaignStatus.Completed);
        campaign.CompletionReason.Should().Be(CompletionReason.ManuallyClosed);
        await _readStore.Received(1).UpsertAsync(Arg.Any<CampaignReadModel>(), Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task ChangeStatus_when_not_found_returns_not_found()
    {
        _repository.GetByIdAsync(Arg.Any<Guid>()).Returns((Campaign?)null);
        var handler = new ChangeCampaignStatusCommandHandler(_repository, _uow, _readStore);

        var result = await handler.Handle(new ChangeCampaignStatusCommand(Guid.NewGuid(), CampaignStatusAction.Cancel), CancellationToken.None);

        result.IsFailure.Should().BeTrue();
        result.Error.Code.Should().Be("Campaign.NotFound");
    }
}
```

- [ ] **Step 3: Rodar e confirmar falha**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~CampaignHandlersTests"`
Expected: FAIL (handlers não existem).

- [ ] **Step 4: Criar o erro `Campaign.NotFound` na Application**

Adicione ao arquivo `Errors/CampaignErrors.cs` (criar):
```csharp
using HackatonFiap.Donations.Domain.Common;

namespace HackatonFiap.Donations.Application.Errors;

public static class CampaignErrors
{
    public static readonly Error NotFound = new("Campaign.NotFound", "Campanha não encontrada.");
}
```

- [ ] **Step 5: Implementar os handlers e mapper**

`Campaigns/CreateCampaign/CreateCampaignCommand.cs`:
```csharp
namespace HackatonFiap.Donations.Application.Campaigns.CreateCampaign;

public sealed record CreateCampaignCommand(
    string Title,
    string Description,
    DateTime StartDate,
    DateTime EndDate,
    decimal Goal,
    Guid CreatedById);
```

`Campaigns/CreateCampaign/CreateCampaignCommandHandler.cs`:
```csharp
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.ReadModels;
using HackatonFiap.Donations.Domain.Common;
using HackatonFiap.Donations.Domain.Entities;

namespace HackatonFiap.Donations.Application.Campaigns.CreateCampaign;

public sealed class CreateCampaignCommandHandler
{
    private readonly ICampaignRepository _repository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly ICampaignReadStore _readStore;
    private readonly IClock _clock;

    public CreateCampaignCommandHandler(ICampaignRepository repository, IUnitOfWork unitOfWork,
        ICampaignReadStore readStore, IClock clock)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
        _readStore = readStore;
        _clock = clock;
    }

    public async Task<Result<Guid>> Handle(CreateCampaignCommand command, CancellationToken cancellationToken)
    {
        var creation = Campaign.Create(command.Title, command.Description, command.StartDate, command.EndDate,
            command.Goal, command.CreatedById, _clock.UtcNow);
        if (creation.IsFailure)
        {
            return Result.Failure<Guid>(creation.Error);
        }

        var campaign = creation.Value;
        await _repository.AddAsync(campaign, cancellationToken);
        await _unitOfWork.SaveChangesAsync(cancellationToken);
        await _readStore.UpsertAsync(campaign.ToReadModel(), cancellationToken);

        return Result.Success(campaign.Id);
    }
}
```

`Campaigns/CampaignMappings.cs` (mapeamentos reutilizados por handlers):
```csharp
using HackatonFiap.Donations.Application.ReadModels;
using HackatonFiap.Donations.Domain.Entities;

namespace HackatonFiap.Donations.Application.Campaigns;

public static class CampaignMappings
{
    public static CampaignReadModel ToReadModel(this Campaign campaign)
        => new(campaign.Id, campaign.Title, campaign.Goal, campaign.AmountRaised, campaign.Status.ToString());

    public static CampaignResponse ToResponse(this Campaign campaign)
        => new(campaign.Id, campaign.Title, campaign.Description, campaign.Period.StartDate, campaign.Period.EndDate,
            campaign.Goal, campaign.AmountRaised, campaign.Status.ToString(),
            campaign.CompletionReason?.ToString(), campaign.CreatedAt, campaign.UpdatedAt);
}
```

`Campaigns/UpdateCampaign/UpdateCampaignCommand.cs`:
```csharp
namespace HackatonFiap.Donations.Application.Campaigns.UpdateCampaign;

public sealed record UpdateCampaignCommand(
    Guid Id,
    string Title,
    string Description,
    DateTime StartDate,
    DateTime EndDate,
    decimal Goal);
```

`Campaigns/UpdateCampaign/UpdateCampaignCommandHandler.cs`:
```csharp
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.Campaigns;
using HackatonFiap.Donations.Application.Errors;
using HackatonFiap.Donations.Domain.Common;

namespace HackatonFiap.Donations.Application.Campaigns.UpdateCampaign;

public sealed class UpdateCampaignCommandHandler
{
    private readonly ICampaignRepository _repository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly ICampaignReadStore _readStore;
    private readonly IClock _clock;

    public UpdateCampaignCommandHandler(ICampaignRepository repository, IUnitOfWork unitOfWork,
        ICampaignReadStore readStore, IClock clock)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
        _readStore = readStore;
        _clock = clock;
    }

    public async Task<Result> Handle(UpdateCampaignCommand command, CancellationToken cancellationToken)
    {
        var campaign = await _repository.GetByIdAsync(command.Id, cancellationToken);
        if (campaign is null)
        {
            return Result.Failure(CampaignErrors.NotFound);
        }

        var update = campaign.Update(command.Title, command.Description, command.StartDate, command.EndDate,
            command.Goal, _clock.UtcNow);
        if (update.IsFailure)
        {
            return update;
        }

        await _unitOfWork.SaveChangesAsync(cancellationToken);
        await _readStore.UpsertAsync(campaign.ToReadModel(), cancellationToken);
        return Result.Success();
    }
}
```

`Campaigns/ChangeCampaignStatus/ChangeCampaignStatusCommand.cs`:
```csharp
namespace HackatonFiap.Donations.Application.Campaigns.ChangeCampaignStatus;

public sealed record ChangeCampaignStatusCommand(Guid Id, CampaignStatusAction Action);
```

`Campaigns/ChangeCampaignStatus/ChangeCampaignStatusCommandHandler.cs`:
```csharp
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.Campaigns;
using HackatonFiap.Donations.Application.Errors;
using HackatonFiap.Donations.Domain.Common;
using HackatonFiap.Donations.Domain.Enums;

namespace HackatonFiap.Donations.Application.Campaigns.ChangeCampaignStatus;

public sealed class ChangeCampaignStatusCommandHandler
{
    private readonly ICampaignRepository _repository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly ICampaignReadStore _readStore;

    public ChangeCampaignStatusCommandHandler(ICampaignRepository repository, IUnitOfWork unitOfWork, ICampaignReadStore readStore)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
        _readStore = readStore;
    }

    public async Task<Result> Handle(ChangeCampaignStatusCommand command, CancellationToken cancellationToken)
    {
        var campaign = await _repository.GetByIdAsync(command.Id, cancellationToken);
        if (campaign is null)
        {
            return Result.Failure(CampaignErrors.NotFound);
        }

        var transition = command.Action == CampaignStatusAction.Close
            ? campaign.Complete(CompletionReason.ManuallyClosed)
            : campaign.Cancel();

        if (transition.IsFailure)
        {
            return transition;
        }

        await _unitOfWork.SaveChangesAsync(cancellationToken);
        await _readStore.UpsertAsync(campaign.ToReadModel(), cancellationToken);
        return Result.Success();
    }
}
```

`Campaigns/GetCampaignById/GetCampaignByIdQuery.cs`:
```csharp
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.Campaigns;
using HackatonFiap.Donations.Application.Errors;
using HackatonFiap.Donations.Domain.Common;

namespace HackatonFiap.Donations.Application.Campaigns.GetCampaignById;

public sealed record GetCampaignByIdQuery(Guid Id);

public sealed class GetCampaignByIdQueryHandler
{
    private readonly ICampaignRepository _repository;

    public GetCampaignByIdQueryHandler(ICampaignRepository repository) => _repository = repository;

    public async Task<Result<CampaignResponse>> Handle(GetCampaignByIdQuery query, CancellationToken cancellationToken)
    {
        var campaign = await _repository.GetByIdAsync(query.Id, cancellationToken);
        if (campaign is null)
        {
            return Result.Failure<CampaignResponse>(CampaignErrors.NotFound);
        }

        return Result.Success(campaign.ToResponse());
    }
}
```

`Campaigns/ListCampaigns/ListCampaignsQuery.cs`:
```csharp
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.Campaigns;
using HackatonFiap.Donations.Domain.Common;

namespace HackatonFiap.Donations.Application.Campaigns.ListCampaigns;

public sealed record ListCampaignsQuery;

public sealed class ListCampaignsQueryHandler
{
    private readonly ICampaignRepository _repository;

    public ListCampaignsQueryHandler(ICampaignRepository repository) => _repository = repository;

    public async Task<Result<IReadOnlyList<CampaignResponse>>> Handle(ListCampaignsQuery query, CancellationToken cancellationToken)
    {
        var campaigns = await _repository.ListAsync(cancellationToken);
        IReadOnlyList<CampaignResponse> responses = campaigns.Select(c => c.ToResponse()).ToList();
        return Result.Success(responses);
    }
}
```

- [ ] **Step 6: Rodar e confirmar PASS**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~CampaignHandlersTests"`
Expected: PASS (4 testes).

- [ ] **Step 7: Commit**

```bash
git add src/HackatonFiap.Donations.Application tests/HackatonFiap.Donations.UnitTests/Application/CampaignHandlersTests.cs
git commit -m "feat(application): handlers de campanha (create/update/status/get/list) com projeção no read store"
```

---

## Task 9: Application — handlers de Doação (create + get) (TDD)

**Files (em `src/HackatonFiap.Donations.Application/Donations/`):**
- Create: `DonationResponse.cs`
- Create: `CreateDonation/CreateDonationCommand.cs` (+ Handler)
- Create: `GetDonationById/GetDonationByIdQuery.cs` (+ Handler)
- Test: `tests/.../Application/DonationHandlersTests.cs`

`CreateDonationCommandHandler` valida campanha (existe/ativa/dentro do período → 422), grava `Pending`, publica `DonationRequestedEvent` com subject `"DonationRequested"`, incrementa métrica `DonationsReceived`.

- [ ] **Step 1: Escrever os testes que falham**

`tests/HackatonFiap.Donations.UnitTests/Application/DonationHandlersTests.cs`:
```csharp
using FluentAssertions;
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.Donations.CreateDonation;
using HackatonFiap.Donations.Application.Donations.GetDonationById;
using HackatonFiap.Donations.Application.IntegrationEvents;
using HackatonFiap.Donations.Domain.Entities;
using HackatonFiap.Donations.Domain.Enums;
using NSubstitute;
using Xunit;

namespace HackatonFiap.Donations.UnitTests.Application;

public class DonationHandlersTests
{
    private static readonly DateTime Now = new(2026, 06, 18, 12, 0, 0, DateTimeKind.Utc);

    private readonly ICampaignRepository _campaigns = Substitute.For<ICampaignRepository>();
    private readonly IDonationRepository _donations = Substitute.For<IDonationRepository>();
    private readonly IUnitOfWork _uow = Substitute.For<IUnitOfWork>();
    private readonly IEventPublisher _publisher = Substitute.For<IEventPublisher>();
    private readonly IClock _clock = Substitute.For<IClock>();

    public DonationHandlersTests() => _clock.UtcNow.Returns(Now);

    private CreateDonationCommandHandler CreateHandler()
        => new(_campaigns, _donations, _uow, _publisher, _clock);

    private static Campaign ActiveCampaign()
        => Campaign.Create("t", "d", Now.AddDays(-1), Now.AddDays(10), 1000m, Guid.NewGuid(), Now).Value;

    private static CreateDonationCommand Command(Guid campaignId, decimal amount = 50m)
        => new(campaignId, amount, PaymentMethod.Pix, Guid.NewGuid(), "donor@example.com", "Donor");

    [Fact]
    public async Task Create_in_active_campaign_persists_and_publishes_event()
    {
        var campaign = ActiveCampaign();
        _campaigns.GetByIdAsync(campaign.Id).Returns(campaign);

        var result = await CreateHandler().Handle(Command(campaign.Id), CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        await _donations.Received(1).AddAsync(Arg.Is<Donation>(d => d.Status == DonationStatus.Pending), Arg.Any<CancellationToken>());
        await _uow.Received(1).SaveChangesAsync(Arg.Any<CancellationToken>());
        await _publisher.Received(1).PublishAsync(
            Arg.Is<DonationRequestedEvent>(e => e.CampaignId == campaign.Id && e.PaymentMethod == PaymentMethod.Pix),
            "DonationRequested",
            Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task Create_for_missing_campaign_returns_422_and_publishes_nothing()
    {
        _campaigns.GetByIdAsync(Arg.Any<Guid>()).Returns((Campaign?)null);

        var result = await CreateHandler().Handle(Command(Guid.NewGuid()), CancellationToken.None);

        result.IsFailure.Should().BeTrue();
        result.Error.Code.Should().Be("Donation.CampaignNotFound");
        await _publisher.DidNotReceive().PublishAsync(Arg.Any<DonationRequestedEvent>(), Arg.Any<string>(), Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task Create_in_cancelled_campaign_returns_not_active()
    {
        var campaign = ActiveCampaign();
        campaign.Cancel();
        _campaigns.GetByIdAsync(campaign.Id).Returns(campaign);

        var result = await CreateHandler().Handle(Command(campaign.Id), CancellationToken.None);

        result.IsFailure.Should().BeTrue();
        result.Error.Code.Should().Be("Donation.CampaignNotActive");
    }

    [Fact]
    public async Task Create_outside_period_returns_outside_period()
    {
        // Campanha ativa, mas o período já terminou (janela do job de expiração — RN04.11 / RN06.3).
        var campaign = Campaign.Create("t", "d", Now.AddDays(-10), Now.AddDays(-1), 1000m, Guid.NewGuid(), Now.AddDays(-11)).Value;
        _campaigns.GetByIdAsync(campaign.Id).Returns(campaign);

        var result = await CreateHandler().Handle(Command(campaign.Id), CancellationToken.None);

        result.IsFailure.Should().BeTrue();
        result.Error.Code.Should().Be("Donation.OutsidePeriod");
    }

    [Fact]
    public async Task GetById_returns_donation_of_owner()
    {
        var donation = Donation.Create(Guid.NewGuid(), 50m, PaymentMethod.Pix, Guid.NewGuid(), "d@e.com", "D").Value;
        _donations.GetByIdAsync(donation.Id).Returns(donation);
        var handler = new GetDonationByIdQueryHandler(_donations);

        var result = await handler.Handle(new GetDonationByIdQuery(donation.Id, donation.DonorId), CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        result.Value.Id.Should().Be(donation.Id);
    }

    [Fact]
    public async Task GetById_for_other_donor_returns_not_found()
    {
        var donation = Donation.Create(Guid.NewGuid(), 50m, PaymentMethod.Pix, Guid.NewGuid(), "d@e.com", "D").Value;
        _donations.GetByIdAsync(donation.Id).Returns(donation);
        var handler = new GetDonationByIdQueryHandler(_donations);

        var result = await handler.Handle(new GetDonationByIdQuery(donation.Id, Guid.NewGuid()), CancellationToken.None);

        result.IsFailure.Should().BeTrue();
        result.Error.Code.Should().Be("Donation.NotFound");
    }
}
```

- [ ] **Step 2: Rodar e confirmar falha**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~DonationHandlersTests"`
Expected: FAIL.

- [ ] **Step 3: Implementar os tipos e handlers**

`Donations/DonationResponse.cs`:
```csharp
namespace HackatonFiap.Donations.Application.Donations;

public sealed record DonationResponse(
    Guid Id,
    Guid CampaignId,
    decimal Amount,
    string Method,
    string Status,
    string? DeclineReason,
    DateTime CreatedAt,
    DateTime? ProcessedAt);
```

`Donations/CreateDonation/CreateDonationCommand.cs`:
```csharp
using HackatonFiap.Donations.Domain.Enums;

namespace HackatonFiap.Donations.Application.Donations.CreateDonation;

public sealed record CreateDonationCommand(
    Guid CampaignId,
    decimal Amount,
    PaymentMethod PaymentMethod,
    Guid DonorId,
    string DonorEmail,
    string DonorName);
```

`Donations/CreateDonation/CreateDonationCommandHandler.cs`:
```csharp
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.Errors;
using HackatonFiap.Donations.Application.IntegrationEvents;
using HackatonFiap.Donations.Application.Observability;
using HackatonFiap.Donations.Domain.Common;
using HackatonFiap.Donations.Domain.Entities;
using HackatonFiap.Donations.Domain.Enums;

namespace HackatonFiap.Donations.Application.Donations.CreateDonation;

public sealed class CreateDonationCommandHandler
{
    public const string EventSubject = "DonationRequested";

    private readonly ICampaignRepository _campaigns;
    private readonly IDonationRepository _donations;
    private readonly IUnitOfWork _unitOfWork;
    private readonly IEventPublisher _publisher;
    private readonly IClock _clock;

    public CreateDonationCommandHandler(ICampaignRepository campaigns, IDonationRepository donations,
        IUnitOfWork unitOfWork, IEventPublisher publisher, IClock clock)
    {
        _campaigns = campaigns;
        _donations = donations;
        _unitOfWork = unitOfWork;
        _publisher = publisher;
        _clock = clock;
    }

    public async Task<Result<Guid>> Handle(CreateDonationCommand command, CancellationToken cancellationToken)
    {
        var campaign = await _campaigns.GetByIdAsync(command.CampaignId, cancellationToken);
        if (campaign is null)
        {
            return Result.Failure<Guid>(DonationErrors.CampaignNotFound);
        }

        if (campaign.Status != CampaignStatus.Active)
        {
            return Result.Failure<Guid>(DonationErrors.CampaignNotActive);
        }

        if (!campaign.Period.Contains(_clock.UtcNow))
        {
            return Result.Failure<Guid>(DonationErrors.OutsidePeriod);
        }

        var creation = Donation.Create(command.CampaignId, command.Amount, command.PaymentMethod,
            command.DonorId, command.DonorEmail, command.DonorName);
        if (creation.IsFailure)
        {
            return Result.Failure<Guid>(creation.Error);
        }

        var donation = creation.Value;
        await _donations.AddAsync(donation, cancellationToken);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        var integrationEvent = new DonationRequestedEvent(
            donation.Id, donation.CampaignId, donation.Amount, donation.Method,
            donation.DonorId, donation.DonorEmail, donation.DonorName);
        await _publisher.PublishAsync(integrationEvent, EventSubject, cancellationToken);

        DonationMetrics.DonationsReceived.Add(1);
        return Result.Success(donation.Id);
    }
}
```

`Donations/GetDonationById/GetDonationByIdQuery.cs`:
```csharp
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.Errors;
using HackatonFiap.Donations.Domain.Common;

namespace HackatonFiap.Donations.Application.Donations.GetDonationById;

public sealed record GetDonationByIdQuery(Guid Id, Guid RequestingDonorId);

public sealed class GetDonationByIdQueryHandler
{
    private readonly IDonationRepository _repository;

    public GetDonationByIdQueryHandler(IDonationRepository repository) => _repository = repository;

    public async Task<Result<DonationResponse>> Handle(GetDonationByIdQuery query, CancellationToken cancellationToken)
    {
        var donation = await _repository.GetByIdAsync(query.Id, cancellationToken);
        if (donation is null || donation.DonorId != query.RequestingDonorId)
        {
            return Result.Failure<DonationResponse>(DonationErrors.NotFound);
        }

        return Result.Success(new DonationResponse(
            donation.Id, donation.CampaignId, donation.Amount, donation.Method.ToString(),
            donation.Status.ToString(), donation.DeclineReason, donation.CreatedAt, donation.ProcessedAt));
    }
}
```

- [ ] **Step 4: Rodar e confirmar PASS**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~DonationHandlersTests"`
Expected: PASS (6 testes).

- [ ] **Step 5: Commit**

```bash
git add src/HackatonFiap.Donations.Application tests/HackatonFiap.Donations.UnitTests/Application/DonationHandlersTests.cs
git commit -m "feat(application): handlers de doação (create publica evento; get restrito ao doador)"
```

---

## Task 10: Application — handlers do consumer (aprovado/recusado) com idempotência (TDD)

**Files (em `src/HackatonFiap.Donations.Application/Donations/`):**
- Create: `ProcessPaymentApproved/ProcessPaymentApprovedCommand.cs` (+ Handler)
- Create: `ProcessPaymentDeclined/ProcessPaymentDeclinedCommand.cs` (+ Handler)
- Test: `tests/.../Application/PaymentResultHandlersTests.cs`

Aprovado: dedup via inbox → `Donation.Approve()` + `Campaign.AddRaised()` + conclui por meta se `AmountRaised ≥ Goal` → grava `ProcessedEvent` → `SaveChanges` → projeta no read store. Recusado: dedup → `Donation.Decline(reason)` → grava `ProcessedEvent` → `SaveChanges`.

- [ ] **Step 1: Escrever os testes que falham**

`tests/HackatonFiap.Donations.UnitTests/Application/PaymentResultHandlersTests.cs`:
```csharp
using FluentAssertions;
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.Donations.ProcessPaymentApproved;
using HackatonFiap.Donations.Application.Donations.ProcessPaymentDeclined;
using HackatonFiap.Donations.Application.ReadModels;
using HackatonFiap.Donations.Domain.Entities;
using HackatonFiap.Donations.Domain.Enums;
using NSubstitute;
using Xunit;

namespace HackatonFiap.Donations.UnitTests.Application;

public class PaymentResultHandlersTests
{
    private static readonly DateTime Now = new(2026, 06, 18, 12, 0, 0, DateTimeKind.Utc);

    private readonly IDonationRepository _donations = Substitute.For<IDonationRepository>();
    private readonly ICampaignRepository _campaigns = Substitute.For<ICampaignRepository>();
    private readonly IProcessedEventStore _processed = Substitute.For<IProcessedEventStore>();
    private readonly IUnitOfWork _uow = Substitute.For<IUnitOfWork>();
    private readonly ICampaignReadStore _readStore = Substitute.For<ICampaignReadStore>();
    private readonly IClock _clock = Substitute.For<IClock>();

    public PaymentResultHandlersTests() => _clock.UtcNow.Returns(Now);

    private ProcessPaymentApprovedCommandHandler ApprovedHandler()
        => new(_donations, _campaigns, _processed, _uow, _readStore, _clock);

    private ProcessPaymentDeclinedCommandHandler DeclinedHandler()
        => new(_donations, _processed, _uow, _clock);

    private static Campaign CampaignWithGoal(decimal goal)
        => Campaign.Create("t", "d", Now.AddDays(-1), Now.AddDays(10), goal, Guid.NewGuid(), Now).Value;

    private static Donation PendingDonation(Guid campaignId, decimal amount)
        => Donation.Create(campaignId, amount, PaymentMethod.Pix, Guid.NewGuid(), "d@e.com", "D").Value;

    [Fact]
    public async Task Approved_consolidates_amount_and_projects()
    {
        var campaign = CampaignWithGoal(1000m);
        var donation = PendingDonation(campaign.Id, 200m);
        _processed.ExistsAsync(donation.Id).Returns(false);
        _donations.GetByIdAsync(donation.Id).Returns(donation);
        _campaigns.GetByIdAsync(campaign.Id).Returns(campaign);

        var command = new ProcessPaymentApprovedCommand(donation.Id, campaign.Id, 200m, Guid.NewGuid(), donation.DonorId, "d@e.com", "D");
        var result = await ApprovedHandler().Handle(command, CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        donation.Status.Should().Be(DonationStatus.Approved);
        campaign.AmountRaised.Should().Be(200m);
        campaign.Status.Should().Be(CampaignStatus.Active);
        await _processed.Received(1).AddAsync(Arg.Any<ProcessedEvent>(), Arg.Any<CancellationToken>());
        await _uow.Received(1).SaveChangesAsync(Arg.Any<CancellationToken>());
        await _readStore.Received(1).UpsertAsync(Arg.Is<CampaignReadModel>(c => c.AmountRaised == 200m), Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task Approved_reaching_goal_completes_campaign()
    {
        var campaign = CampaignWithGoal(150m);
        var donation = PendingDonation(campaign.Id, 200m);
        _processed.ExistsAsync(donation.Id).Returns(false);
        _donations.GetByIdAsync(donation.Id).Returns(donation);
        _campaigns.GetByIdAsync(campaign.Id).Returns(campaign);

        var command = new ProcessPaymentApprovedCommand(donation.Id, campaign.Id, 200m, Guid.NewGuid(), donation.DonorId, "d@e.com", "D");
        var result = await ApprovedHandler().Handle(command, CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        campaign.Status.Should().Be(CampaignStatus.Completed);
        campaign.CompletionReason.Should().Be(CompletionReason.GoalReached);
    }

    [Fact]
    public async Task Approved_is_idempotent_when_already_processed()
    {
        var donationId = Guid.NewGuid();
        _processed.ExistsAsync(donationId).Returns(true);

        var command = new ProcessPaymentApprovedCommand(donationId, Guid.NewGuid(), 200m, Guid.NewGuid(), Guid.NewGuid(), "d@e.com", "D");
        var result = await ApprovedHandler().Handle(command, CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        await _donations.DidNotReceive().GetByIdAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>());
        await _uow.DidNotReceive().SaveChangesAsync(Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task Declined_marks_donation_declined_without_consolidating()
    {
        var donation = PendingDonation(Guid.NewGuid(), 99.99m);
        _processed.ExistsAsync(donation.Id).Returns(false);
        _donations.GetByIdAsync(donation.Id).Returns(donation);

        var command = new ProcessPaymentDeclinedCommand(donation.Id, donation.CampaignId, "centavos ,99", 99.99m, donation.DonorId, "d@e.com", "D");
        var result = await DeclinedHandler().Handle(command, CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        donation.Status.Should().Be(DonationStatus.Declined);
        await _processed.Received(1).AddAsync(Arg.Any<ProcessedEvent>(), Arg.Any<CancellationToken>());
        await _uow.Received(1).SaveChangesAsync(Arg.Any<CancellationToken>());
    }
}
```

- [ ] **Step 2: Rodar e confirmar falha**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~PaymentResultHandlersTests"`
Expected: FAIL.

- [ ] **Step 3: Implementar os handlers**

`Donations/ProcessPaymentApproved/ProcessPaymentApprovedCommand.cs`:
```csharp
namespace HackatonFiap.Donations.Application.Donations.ProcessPaymentApproved;

public sealed record ProcessPaymentApprovedCommand(
    Guid DonationId,
    Guid CampaignId,
    decimal Amount,
    Guid PaymentId,
    Guid DonorId,
    string DonorEmail,
    string DonorName);
```

`Donations/ProcessPaymentApproved/ProcessPaymentApprovedCommandHandler.cs`:
```csharp
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.Campaigns;
using HackatonFiap.Donations.Application.Errors;
using HackatonFiap.Donations.Application.Observability;
using HackatonFiap.Donations.Domain.Common;
using HackatonFiap.Donations.Domain.Entities;
using HackatonFiap.Donations.Domain.Enums;

namespace HackatonFiap.Donations.Application.Donations.ProcessPaymentApproved;

public sealed class ProcessPaymentApprovedCommandHandler
{
    private readonly IDonationRepository _donations;
    private readonly ICampaignRepository _campaigns;
    private readonly IProcessedEventStore _processed;
    private readonly IUnitOfWork _unitOfWork;
    private readonly ICampaignReadStore _readStore;
    private readonly IClock _clock;

    public ProcessPaymentApprovedCommandHandler(IDonationRepository donations, ICampaignRepository campaigns,
        IProcessedEventStore processed, IUnitOfWork unitOfWork, ICampaignReadStore readStore, IClock clock)
    {
        _donations = donations;
        _campaigns = campaigns;
        _processed = processed;
        _unitOfWork = unitOfWork;
        _readStore = readStore;
        _clock = clock;
    }

    public async Task<Result> Handle(ProcessPaymentApprovedCommand command, CancellationToken cancellationToken)
    {
        if (await _processed.ExistsAsync(command.DonationId, cancellationToken))
        {
            return Result.Success(); // idempotência (RN06.10)
        }

        var donation = await _donations.GetByIdAsync(command.DonationId, cancellationToken);
        if (donation is null)
        {
            return Result.Failure(DonationErrors.NotFound); // doação ainda não persistida -> abandona p/ reentrega
        }

        donation.Approve();

        Campaign? campaign = await _campaigns.GetByIdAsync(command.CampaignId, cancellationToken);
        if (campaign is not null)
        {
            campaign.AddRaised(command.Amount);
            if (campaign.Status == CampaignStatus.Active && campaign.AmountRaised >= campaign.Goal)
            {
                campaign.Complete(CompletionReason.GoalReached);
                DonationMetrics.CampaignsCompleted.Add(1);
            }
        }

        await _processed.AddAsync(new ProcessedEvent(command.DonationId, _clock.UtcNow), cancellationToken);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        if (campaign is not null)
        {
            await _readStore.UpsertAsync(campaign.ToReadModel(), cancellationToken);
        }

        DonationMetrics.DonationsApproved.Add(1);
        DonationMetrics.AmountRaised.Add((double)command.Amount);
        return Result.Success();
    }
}
```

`Donations/ProcessPaymentDeclined/ProcessPaymentDeclinedCommand.cs`:
```csharp
namespace HackatonFiap.Donations.Application.Donations.ProcessPaymentDeclined;

public sealed record ProcessPaymentDeclinedCommand(
    Guid DonationId,
    Guid CampaignId,
    string Reason,
    decimal Amount,
    Guid DonorId,
    string DonorEmail,
    string DonorName);
```

`Donations/ProcessPaymentDeclined/ProcessPaymentDeclinedCommandHandler.cs`:
```csharp
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.Errors;
using HackatonFiap.Donations.Application.Observability;
using HackatonFiap.Donations.Domain.Common;
using HackatonFiap.Donations.Domain.Entities;

namespace HackatonFiap.Donations.Application.Donations.ProcessPaymentDeclined;

public sealed class ProcessPaymentDeclinedCommandHandler
{
    private readonly IDonationRepository _donations;
    private readonly IProcessedEventStore _processed;
    private readonly IUnitOfWork _unitOfWork;
    private readonly IClock _clock;

    public ProcessPaymentDeclinedCommandHandler(IDonationRepository donations, IProcessedEventStore processed,
        IUnitOfWork unitOfWork, IClock clock)
    {
        _donations = donations;
        _processed = processed;
        _unitOfWork = unitOfWork;
        _clock = clock;
    }

    public async Task<Result> Handle(ProcessPaymentDeclinedCommand command, CancellationToken cancellationToken)
    {
        if (await _processed.ExistsAsync(command.DonationId, cancellationToken))
        {
            return Result.Success();
        }

        var donation = await _donations.GetByIdAsync(command.DonationId, cancellationToken);
        if (donation is null)
        {
            return Result.Failure(DonationErrors.NotFound);
        }

        donation.Decline(command.Reason);
        await _processed.AddAsync(new ProcessedEvent(command.DonationId, _clock.UtcNow), cancellationToken);
        await _unitOfWork.SaveChangesAsync(cancellationToken);

        DonationMetrics.DonationsDeclined.Add(1);
        return Result.Success();
    }
}
```

- [ ] **Step 4: Rodar e confirmar PASS**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~PaymentResultHandlersTests"`
Expected: PASS (4 testes).

- [ ] **Step 5: Commit**

```bash
git add src/HackatonFiap.Donations.Application tests/HackatonFiap.Donations.UnitTests/Application/PaymentResultHandlersTests.cs
git commit -m "feat(application): consumer handlers (aprovado consolida+conclui por meta; recusado; idempotentes)"
```

---

## Task 11: Application — `ExpireDueCampaignsCommandHandler` (TDD)

**Files:**
- Create: `src/HackatonFiap.Donations.Application/Campaigns/ExpireDueCampaigns/ExpireDueCampaignsCommandHandler.cs`
- Test: adicionar em `CampaignHandlersTests.cs` ou novo `ExpireDueCampaignsHandlerTests.cs`

- [ ] **Step 1: Escrever o teste que falha**

`tests/HackatonFiap.Donations.UnitTests/Application/ExpireDueCampaignsHandlerTests.cs`:
```csharp
using FluentAssertions;
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.Campaigns.ExpireDueCampaigns;
using HackatonFiap.Donations.Application.ReadModels;
using HackatonFiap.Donations.Domain.Entities;
using HackatonFiap.Donations.Domain.Enums;
using NSubstitute;
using Xunit;

namespace HackatonFiap.Donations.UnitTests.Application;

public class ExpireDueCampaignsHandlerTests
{
    private static readonly DateTime Now = new(2026, 06, 18, 12, 0, 0, DateTimeKind.Utc);

    [Fact]
    public async Task Expires_active_campaigns_past_end_date_and_projects()
    {
        var repository = Substitute.For<ICampaignRepository>();
        var uow = Substitute.For<IUnitOfWork>();
        var readStore = Substitute.For<ICampaignReadStore>();
        var clock = Substitute.For<IClock>();
        clock.UtcNow.Returns(Now);

        var expired = Campaign.Create("t", "d", Now.AddDays(-30), Now.AddDays(-1), 1000m, Guid.NewGuid(), Now.AddDays(-31)).Value;
        repository.ListActiveExpiredAsync(Now).Returns(new[] { expired });

        var handler = new ExpireDueCampaignsCommandHandler(repository, uow, readStore, clock);
        await handler.Handle(CancellationToken.None);

        expired.Status.Should().Be(CampaignStatus.Completed);
        expired.CompletionReason.Should().Be(CompletionReason.Expired);
        await uow.Received(1).SaveChangesAsync(Arg.Any<CancellationToken>());
        await readStore.Received(1).UpsertAsync(Arg.Any<CampaignReadModel>(), Arg.Any<CancellationToken>());
    }
}
```

- [ ] **Step 2: Rodar e confirmar falha**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~ExpireDueCampaignsHandlerTests"`
Expected: FAIL.

- [ ] **Step 3: Implementar o handler**

`Campaigns/ExpireDueCampaigns/ExpireDueCampaignsCommandHandler.cs`:
```csharp
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.Campaigns;
using HackatonFiap.Donations.Application.Observability;
using HackatonFiap.Donations.Domain.Enums;

namespace HackatonFiap.Donations.Application.Campaigns.ExpireDueCampaigns;

public sealed class ExpireDueCampaignsCommandHandler
{
    private readonly ICampaignRepository _repository;
    private readonly IUnitOfWork _unitOfWork;
    private readonly ICampaignReadStore _readStore;
    private readonly IClock _clock;

    public ExpireDueCampaignsCommandHandler(ICampaignRepository repository, IUnitOfWork unitOfWork,
        ICampaignReadStore readStore, IClock clock)
    {
        _repository = repository;
        _unitOfWork = unitOfWork;
        _readStore = readStore;
        _clock = clock;
    }

    public async Task Handle(CancellationToken cancellationToken)
    {
        var due = await _repository.ListActiveExpiredAsync(_clock.UtcNow, cancellationToken);
        if (due.Count == 0)
        {
            return;
        }

        foreach (var campaign in due)
        {
            var transition = campaign.Complete(CompletionReason.Expired);
            if (transition.IsSuccess)
            {
                DonationMetrics.CampaignsCompleted.Add(1);
            }
        }

        await _unitOfWork.SaveChangesAsync(cancellationToken);

        foreach (var campaign in due)
        {
            await _readStore.UpsertAsync(campaign.ToReadModel(), cancellationToken);
        }
    }
}
```

- [ ] **Step 4: Rodar e confirmar PASS**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~ExpireDueCampaignsHandlerTests"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/HackatonFiap.Donations.Application tests/HackatonFiap.Donations.UnitTests/Application/ExpireDueCampaignsHandlerTests.cs
git commit -m "feat(application): ExpireDueCampaigns (conclui campanhas vencidas como Expired)"
```

---

## Task 12: Application — Transparência (`ListActiveCampaignsQuery`) (TDD)

**Files:**
- Create: `src/HackatonFiap.Donations.Application/Transparency/TransparencyCampaignResponse.cs`
- Create: `src/HackatonFiap.Donations.Application/Transparency/ListActiveCampaignsQuery.cs` (+ Handler)
- Test: `tests/.../Application/ListActiveCampaignsHandlerTests.cs`

- [ ] **Step 1: Escrever o teste que falha**

`tests/HackatonFiap.Donations.UnitTests/Application/ListActiveCampaignsHandlerTests.cs`:
```csharp
using FluentAssertions;
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.ReadModels;
using HackatonFiap.Donations.Application.Transparency;
using NSubstitute;
using Xunit;

namespace HackatonFiap.Donations.UnitTests.Application;

public class ListActiveCampaignsHandlerTests
{
    [Fact]
    public async Task Maps_read_models_and_computes_percentual()
    {
        var readStore = Substitute.For<ICampaignReadStore>();
        readStore.ListActiveAsync().Returns(new[]
        {
            new CampaignReadModel(Guid.NewGuid(), "Inverno", 1000m, 250m, "Active")
        });
        var handler = new ListActiveCampaignsQueryHandler(readStore);

        var result = await handler.Handle(new ListActiveCampaignsQuery(), CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        result.Value.Should().HaveCount(1);
        result.Value[0].Title.Should().Be("Inverno");
        result.Value[0].Percentual.Should().Be(25m);
    }
}
```

- [ ] **Step 2: Rodar e confirmar falha**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~ListActiveCampaignsHandlerTests"`
Expected: FAIL.

- [ ] **Step 3: Implementar**

`Transparency/TransparencyCampaignResponse.cs`:
```csharp
namespace HackatonFiap.Donations.Application.Transparency;

public sealed record TransparencyCampaignResponse(
    string Title,
    decimal Goal,
    decimal AmountRaised,
    decimal Percentual);
```

`Transparency/ListActiveCampaignsQuery.cs`:
```csharp
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Domain.Common;

namespace HackatonFiap.Donations.Application.Transparency;

public sealed record ListActiveCampaignsQuery;

public sealed class ListActiveCampaignsQueryHandler
{
    private readonly ICampaignReadStore _readStore;

    public ListActiveCampaignsQueryHandler(ICampaignReadStore readStore) => _readStore = readStore;

    public async Task<Result<IReadOnlyList<TransparencyCampaignResponse>>> Handle(
        ListActiveCampaignsQuery query, CancellationToken cancellationToken)
    {
        var campaigns = await _readStore.ListActiveAsync(cancellationToken);
        IReadOnlyList<TransparencyCampaignResponse> responses = campaigns
            .Select(c => new TransparencyCampaignResponse(
                c.Title, c.Goal, c.AmountRaised,
                c.Goal > 0m ? Math.Round(c.AmountRaised / c.Goal * 100m, 2) : 0m))
            .ToList();

        return Result.Success(responses);
    }
}
```

- [ ] **Step 4: Rodar e confirmar PASS**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~ListActiveCampaignsHandlerTests"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add src/HackatonFiap.Donations.Application tests/HackatonFiap.Donations.UnitTests/Application/ListActiveCampaignsHandlerTests.cs
git commit -m "feat(application): query do painel de transparência (lista ativas + percentual)"
```

---

## Task 13: Infrastructure — persistência (DbContext, configurations, repositories, UoW) + migration

**Files (em `src/HackatonFiap.Donations.Infrastructure/`):**
- Create: `Persistence/DonationsDbContext.cs`, `Persistence/DonationsDbContextFactory.cs`
- Create: `Persistence/Configurations/{CampaignConfiguration,DonationConfiguration,ProcessedEventConfiguration}.cs`
- Create: `Persistence/Repositories/{CampaignRepository,DonationRepository,ProcessedEventStore}.cs`, `Persistence/UnitOfWork.cs`
- Create: migration `InitialCreate`

- [ ] **Step 1: Criar `Persistence/DonationsDbContext.cs`**

```csharp
using HackatonFiap.Donations.Domain.Entities;
using Microsoft.EntityFrameworkCore;

namespace HackatonFiap.Donations.Infrastructure.Persistence;

public class DonationsDbContext : DbContext
{
    public DonationsDbContext(DbContextOptions<DonationsDbContext> options) : base(options) { }

    public DbSet<Campaign> Campaigns => Set<Campaign>();
    public DbSet<Donation> Donations => Set<Donation>();
    public DbSet<ProcessedEvent> ProcessedEvents => Set<ProcessedEvent>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(DonationsDbContext).Assembly);
        base.OnModelCreating(modelBuilder);
    }
}
```

- [ ] **Step 2: Criar as configurations**

`Persistence/Configurations/CampaignConfiguration.cs`:
```csharp
using HackatonFiap.Donations.Domain.Entities;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace HackatonFiap.Donations.Infrastructure.Persistence.Configurations;

public sealed class CampaignConfiguration : IEntityTypeConfiguration<Campaign>
{
    public void Configure(EntityTypeBuilder<Campaign> builder)
    {
        builder.ToTable("Campaigns");
        builder.HasKey(c => c.Id);

        builder.Property(c => c.Title).HasMaxLength(200).IsRequired();
        builder.Property(c => c.Description).HasMaxLength(2000);
        builder.Property(c => c.Goal).HasColumnType("decimal(18,2)").IsRequired();
        builder.Property(c => c.AmountRaised).HasColumnType("decimal(18,2)").IsRequired();
        builder.Property(c => c.Status).HasConversion<int>().IsRequired();
        builder.Property(c => c.CompletionReason).HasConversion<int?>();
        builder.Property(c => c.CreatedById).IsRequired();
        builder.Property(c => c.CreatedAt).IsRequired();
        builder.Property(c => c.UpdatedAt).IsRequired();

        builder.OwnsOne(c => c.Period, p =>
        {
            p.Property(x => x.StartDate).HasColumnName("StartDate").IsRequired();
            p.Property(x => x.EndDate).HasColumnName("EndDate").IsRequired();
        });
        builder.Navigation(c => c.Period).IsRequired();

        builder.HasIndex(c => c.Status);
    }
}
```

`Persistence/Configurations/DonationConfiguration.cs`:
```csharp
using HackatonFiap.Donations.Domain.Entities;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace HackatonFiap.Donations.Infrastructure.Persistence.Configurations;

public sealed class DonationConfiguration : IEntityTypeConfiguration<Donation>
{
    public void Configure(EntityTypeBuilder<Donation> builder)
    {
        builder.ToTable("Donations");
        builder.HasKey(d => d.Id);

        builder.Property(d => d.CampaignId).IsRequired();
        builder.Property(d => d.Amount).HasColumnType("decimal(18,2)").IsRequired();
        builder.Property(d => d.Method).HasConversion<int>().IsRequired();
        builder.Property(d => d.Status).HasConversion<int>().IsRequired();
        builder.Property(d => d.DonorId).IsRequired();
        builder.Property(d => d.DonorEmail).HasMaxLength(256).IsRequired();
        builder.Property(d => d.DonorName).HasMaxLength(256).IsRequired();
        builder.Property(d => d.DeclineReason).HasMaxLength(512);
        builder.Property(d => d.CreatedAt).IsRequired();

        builder.HasIndex(d => d.CampaignId);
        builder.HasIndex(d => d.DonorId);
    }
}
```

`Persistence/Configurations/ProcessedEventConfiguration.cs`:
```csharp
using HackatonFiap.Donations.Domain.Entities;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace HackatonFiap.Donations.Infrastructure.Persistence.Configurations;

public sealed class ProcessedEventConfiguration : IEntityTypeConfiguration<ProcessedEvent>
{
    public void Configure(EntityTypeBuilder<ProcessedEvent> builder)
    {
        builder.ToTable("ProcessedEvents");
        builder.HasKey(p => p.DonationId);   // PK = idempotência por doação
        builder.Property(p => p.ProcessedAt).IsRequired();
    }
}
```

- [ ] **Step 3: Criar `UnitOfWork` e repositórios**

`Persistence/UnitOfWork.cs`:
```csharp
using HackatonFiap.Donations.Application.Abstractions;

namespace HackatonFiap.Donations.Infrastructure.Persistence;

public sealed class UnitOfWork : IUnitOfWork
{
    private readonly DonationsDbContext _context;

    public UnitOfWork(DonationsDbContext context) => _context = context;

    public Task SaveChangesAsync(CancellationToken cancellationToken = default)
        => _context.SaveChangesAsync(cancellationToken);
}
```

`Persistence/Repositories/CampaignRepository.cs`:
```csharp
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Domain.Entities;
using HackatonFiap.Donations.Domain.Enums;
using Microsoft.EntityFrameworkCore;

namespace HackatonFiap.Donations.Infrastructure.Persistence.Repositories;

public sealed class CampaignRepository : ICampaignRepository
{
    private readonly DonationsDbContext _context;

    public CampaignRepository(DonationsDbContext context) => _context = context;

    public Task<Campaign?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
        => _context.Campaigns.FirstOrDefaultAsync(c => c.Id == id, cancellationToken);

    public async Task AddAsync(Campaign campaign, CancellationToken cancellationToken = default)
        => await _context.Campaigns.AddAsync(campaign, cancellationToken);

    public async Task<IReadOnlyList<Campaign>> ListAsync(CancellationToken cancellationToken = default)
        => await _context.Campaigns.AsNoTracking().OrderByDescending(c => c.CreatedAt).ToListAsync(cancellationToken);

    public async Task<IReadOnlyList<Campaign>> ListActiveExpiredAsync(DateTime utcNow, CancellationToken cancellationToken = default)
        => await _context.Campaigns
            .Where(c => c.Status == CampaignStatus.Active && c.Period.EndDate < utcNow)
            .ToListAsync(cancellationToken);
}
```

`Persistence/Repositories/DonationRepository.cs`:
```csharp
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Domain.Entities;
using Microsoft.EntityFrameworkCore;

namespace HackatonFiap.Donations.Infrastructure.Persistence.Repositories;

public sealed class DonationRepository : IDonationRepository
{
    private readonly DonationsDbContext _context;

    public DonationRepository(DonationsDbContext context) => _context = context;

    public Task<Donation?> GetByIdAsync(Guid id, CancellationToken cancellationToken = default)
        => _context.Donations.FirstOrDefaultAsync(d => d.Id == id, cancellationToken);

    public async Task AddAsync(Donation donation, CancellationToken cancellationToken = default)
        => await _context.Donations.AddAsync(donation, cancellationToken);
}
```

`Persistence/Repositories/ProcessedEventStore.cs`:
```csharp
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Domain.Entities;
using Microsoft.EntityFrameworkCore;

namespace HackatonFiap.Donations.Infrastructure.Persistence.Repositories;

public sealed class ProcessedEventStore : IProcessedEventStore
{
    private readonly DonationsDbContext _context;

    public ProcessedEventStore(DonationsDbContext context) => _context = context;

    public Task<bool> ExistsAsync(Guid donationId, CancellationToken cancellationToken = default)
        => _context.ProcessedEvents.AsNoTracking().AnyAsync(p => p.DonationId == donationId, cancellationToken);

    public async Task AddAsync(ProcessedEvent processedEvent, CancellationToken cancellationToken = default)
        => await _context.ProcessedEvents.AddAsync(processedEvent, cancellationToken);
}
```

- [ ] **Step 4: Criar a factory de design-time**

`Persistence/DonationsDbContextFactory.cs`:
```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Design;

namespace HackatonFiap.Donations.Infrastructure.Persistence;

public sealed class DonationsDbContextFactory : IDesignTimeDbContextFactory<DonationsDbContext>
{
    public DonationsDbContext CreateDbContext(string[] args)
    {
        var options = new DbContextOptionsBuilder<DonationsDbContext>()
            .UseSqlServer("Server=localhost;Database=HackatonFiapDonationsDb;Trusted_Connection=True;TrustServerCertificate=true;")
            .Options;

        return new DonationsDbContext(options);
    }
}
```

- [ ] **Step 5: Build e gerar a migration**

Run:
```bash
dotnet build src/HackatonFiap.Donations.Infrastructure
dotnet tool restore
dotnet ef migrations add InitialCreate \
  --project src/HackatonFiap.Donations.Infrastructure \
  --startup-project src/HackatonFiap.Donations.Infrastructure \
  --output-dir Persistence/Migrations
```
Expected: migration criada em `Persistence/Migrations/*_InitialCreate.cs` (tabelas Campaigns com StartDate/EndDate, Donations, ProcessedEvents). Se o EF reclamar de startup-project sem host, use a própria Infrastructure (a factory de design-time supre o host).

- [ ] **Step 6: Commit**

```bash
git add src/HackatonFiap.Donations.Infrastructure
git commit -m "feat(infra): DonationsDbContext, configurations, repositórios, UoW e migration InitialCreate"
```

---

## Task 14: Infrastructure — read store (InMemory TDD + Cosmos)

**Files (em `src/HackatonFiap.Donations.Infrastructure/ReadStore/`):**
- Create: `InMemoryCampaignReadStore.cs`
- Create: `CosmosCampaignReadStore.cs` + `CosmosOptions.cs`
- Test: `tests/.../Infrastructure/InMemoryCampaignReadStoreTests.cs`

- [ ] **Step 1: Escrever o teste que falha (InMemory)**

`tests/HackatonFiap.Donations.UnitTests/Infrastructure/InMemoryCampaignReadStoreTests.cs`:
```csharp
using FluentAssertions;
using HackatonFiap.Donations.Application.ReadModels;
using HackatonFiap.Donations.Infrastructure.ReadStore;
using Xunit;

namespace HackatonFiap.Donations.UnitTests.Infrastructure;

public class InMemoryCampaignReadStoreTests
{
    [Fact]
    public async Task Upsert_replaces_by_id_and_list_active_filters_non_active()
    {
        var store = new InMemoryCampaignReadStore();
        var id = Guid.NewGuid();

        await store.UpsertAsync(new CampaignReadModel(id, "A", 100m, 10m, "Active"));
        await store.UpsertAsync(new CampaignReadModel(id, "A", 100m, 50m, "Active")); // upsert mesmo id
        await store.UpsertAsync(new CampaignReadModel(Guid.NewGuid(), "B", 100m, 0m, "Completed"));

        var active = await store.ListActiveAsync();

        active.Should().HaveCount(1);
        active[0].AmountRaised.Should().Be(50m);
    }
}
```

- [ ] **Step 2: Rodar e confirmar falha**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~InMemoryCampaignReadStoreTests"`
Expected: FAIL.

- [ ] **Step 3: Implementar `InMemoryCampaignReadStore`**

```csharp
using System.Collections.Concurrent;
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.ReadModels;

namespace HackatonFiap.Donations.Infrastructure.ReadStore;

public sealed class InMemoryCampaignReadStore : ICampaignReadStore
{
    private readonly ConcurrentDictionary<Guid, CampaignReadModel> _store = new();

    public Task UpsertAsync(CampaignReadModel campaign, CancellationToken cancellationToken = default)
    {
        _store[campaign.Id] = campaign;
        return Task.CompletedTask;
    }

    public Task<IReadOnlyList<CampaignReadModel>> ListActiveAsync(CancellationToken cancellationToken = default)
    {
        IReadOnlyList<CampaignReadModel> active = _store.Values
            .Where(c => c.Status == "Active")
            .ToList();
        return Task.FromResult(active);
    }
}
```

- [ ] **Step 4: Rodar e confirmar PASS**

Run: `dotnet test tests/HackatonFiap.Donations.UnitTests --filter "FullyQualifiedName~InMemoryCampaignReadStoreTests"`
Expected: PASS.

- [ ] **Step 5: Implementar o Cosmos read store** (sem teste unitário — depende de recurso externo; validado em runtime)

`ReadStore/CosmosOptions.cs`:
```csharp
namespace HackatonFiap.Donations.Infrastructure.ReadStore;

public sealed class CosmosOptions
{
    public const string SectionName = "Cosmos";

    public string ConnectionString { get; set; } = string.Empty;
    public string Database { get; set; } = "HackatonFiapDonations";
    public string Container { get; set; } = "campaigns";
}
```

`ReadStore/CosmosCampaignReadStore.cs`:
```csharp
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.ReadModels;
using Microsoft.Azure.Cosmos;

namespace HackatonFiap.Donations.Infrastructure.ReadStore;

public sealed class CosmosCampaignReadStore : ICampaignReadStore
{
    private readonly Container _container;

    private sealed record CampaignDocument(string id, string title, decimal goal, decimal amountRaised, string status);

    public CosmosCampaignReadStore(CosmosClient client, CosmosOptions options)
    {
        _container = client.GetContainer(options.Database, options.Container);
    }

    public async Task UpsertAsync(CampaignReadModel campaign, CancellationToken cancellationToken = default)
    {
        var doc = new CampaignDocument(campaign.Id.ToString(), campaign.Title, campaign.Goal, campaign.AmountRaised, campaign.Status);
        await _container.UpsertItemAsync(doc, new PartitionKey(doc.id), cancellationToken: cancellationToken);
    }

    public async Task<IReadOnlyList<CampaignReadModel>> ListActiveAsync(CancellationToken cancellationToken = default)
    {
        var query = new QueryDefinition("SELECT * FROM c WHERE c.status = @status").WithParameter("@status", "Active");
        var iterator = _container.GetItemQueryIterator<CampaignDocument>(query);

        var results = new List<CampaignReadModel>();
        while (iterator.HasMoreResults)
        {
            var page = await iterator.ReadNextAsync(cancellationToken);
            results.AddRange(page.Select(d => new CampaignReadModel(Guid.Parse(d.id), d.title, d.goal, d.amountRaised, d.status)));
        }

        return results;
    }
}
```

- [ ] **Step 6: Build + Commit**

Run: `dotnet build src/HackatonFiap.Donations.Infrastructure`
Expected: PASS.
```bash
git add src/HackatonFiap.Donations.Infrastructure tests/HackatonFiap.Donations.UnitTests/Infrastructure/InMemoryCampaignReadStoreTests.cs
git commit -m "feat(infra): read store do painel (InMemory + Cosmos)"
```

---

## Task 15: Infrastructure — messaging (publisher, consumer, JSON, NoOp) + clock

**Files (em `src/HackatonFiap.Donations.Infrastructure/`):**
- Create: `Messaging/MessagingJson.cs`, `Messaging/ServiceBusEventPublisher.cs`, `Messaging/NoOpEventPublisher.cs`, `Messaging/PaymentResultConsumer.cs`
- Create: `Time/SystemClock.cs`

O `ServiceBusEventPublisher` publica no **tópico de requisição** (`donation-requested`). O `PaymentResultConsumer` consome o tópico `payment-result` na subscription `donations`, roteando por `Subject`.

- [ ] **Step 1: Criar `Time/SystemClock.cs`**

```csharp
using HackatonFiap.Donations.Application.Abstractions;

namespace HackatonFiap.Donations.Infrastructure.Time;

public sealed class SystemClock : IClock
{
    public DateTime UtcNow => DateTime.UtcNow;
}
```

- [ ] **Step 2: Criar `Messaging/MessagingJson.cs`** (idêntico ao payments — enum como string)

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

namespace HackatonFiap.Donations.Infrastructure.Messaging;

internal static class MessagingJson
{
    public static readonly JsonSerializerOptions Options = new()
    {
        Converters = { new JsonStringEnumConverter() }
    };
}
```

- [ ] **Step 3: Criar `Messaging/ServiceBusEventPublisher.cs`**

```csharp
using System.Text.Json;
using Azure.Messaging.ServiceBus;
using HackatonFiap.Donations.Application.Abstractions;
using Microsoft.Extensions.Logging;

namespace HackatonFiap.Donations.Infrastructure.Messaging;

public sealed class ServiceBusEventPublisher : IEventPublisher, IAsyncDisposable
{
    private readonly ServiceBusSender _sender;
    private readonly ILogger<ServiceBusEventPublisher> _logger;

    public ServiceBusEventPublisher(ServiceBusClient client, string topicName, ILogger<ServiceBusEventPublisher> logger)
    {
        _sender = client.CreateSender(topicName);
        _logger = logger;
    }

    public async Task PublishAsync<TEvent>(TEvent integrationEvent, string subject, CancellationToken cancellationToken = default)
        where TEvent : notnull
    {
        var body = JsonSerializer.Serialize(integrationEvent, MessagingJson.Options);
        var message = new ServiceBusMessage(body)
        {
            ContentType = "application/json",
            Subject = subject
        };

        await _sender.SendMessageAsync(message, cancellationToken);
        _logger.LogInformation("Evento publicado. Subject={Subject}", subject);
    }

    public ValueTask DisposeAsync() => _sender.DisposeAsync();
}
```

- [ ] **Step 4: Criar `Messaging/NoOpEventPublisher.cs`**

```csharp
using HackatonFiap.Donations.Application.Abstractions;
using Microsoft.Extensions.Logging;

namespace HackatonFiap.Donations.Infrastructure.Messaging;

public sealed class NoOpEventPublisher : IEventPublisher
{
    private readonly ILogger<NoOpEventPublisher> _logger;

    public NoOpEventPublisher(ILogger<NoOpEventPublisher> logger) => _logger = logger;

    public Task PublishAsync<TEvent>(TEvent integrationEvent, string subject, CancellationToken cancellationToken = default)
        where TEvent : notnull
    {
        _logger.LogWarning("ServiceBus não configurado — evento {Subject} NÃO publicado (NoOp).", subject);
        return Task.CompletedTask;
    }
}
```

- [ ] **Step 5: Criar `Messaging/PaymentResultConsumer.cs`**

```csharp
using System.Text.Json;
using Azure.Messaging.ServiceBus;
using HackatonFiap.Donations.Application.Donations.ProcessPaymentApproved;
using HackatonFiap.Donations.Application.Donations.ProcessPaymentDeclined;
using HackatonFiap.Donations.Application.IntegrationEvents;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

namespace HackatonFiap.Donations.Infrastructure.Messaging;

public sealed class PaymentResultConsumer : BackgroundService
{
    public const string ApprovedSubject = "PaymentApproved";
    public const string DeclinedSubject = "PaymentDeclined";

    private readonly ServiceBusProcessor _processor;
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<PaymentResultConsumer> _logger;

    public PaymentResultConsumer(ServiceBusClient client, string resultTopicName, string subscriptionName,
        IServiceScopeFactory scopeFactory, ILogger<PaymentResultConsumer> logger)
    {
        _processor = client.CreateProcessor(resultTopicName, subscriptionName, new ServiceBusProcessorOptions
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
        var subject = args.Message.Subject;
        var body = args.Message.Body.ToString();

        try
        {
            using var scope = _scopeFactory.CreateScope();

            if (subject == ApprovedSubject)
            {
                var evt = JsonSerializer.Deserialize<PaymentApprovedEvent>(body, MessagingJson.Options);
                if (evt is null)
                {
                    await args.DeadLetterMessageAsync(args.Message, "InvalidPayload", "PaymentApprovedEvent nulo.");
                    return;
                }

                var handler = scope.ServiceProvider.GetRequiredService<ProcessPaymentApprovedCommandHandler>();
                var result = await handler.Handle(new ProcessPaymentApprovedCommand(
                    evt.DonationId, evt.CampaignId, evt.Amount, evt.PaymentId, evt.DonorId, evt.DonorEmail, evt.DonorName),
                    args.CancellationToken);

                if (result.IsFailure)
                {
                    _logger.LogWarning("Falha ao consolidar doação {DonationId}: {Error}. Abandonando p/ reentrega.",
                        evt.DonationId, result.Error.Message);
                    await args.AbandonMessageAsync(args.Message);
                    return;
                }
            }
            else if (subject == DeclinedSubject)
            {
                var evt = JsonSerializer.Deserialize<PaymentDeclinedEvent>(body, MessagingJson.Options);
                if (evt is null)
                {
                    await args.DeadLetterMessageAsync(args.Message, "InvalidPayload", "PaymentDeclinedEvent nulo.");
                    return;
                }

                var handler = scope.ServiceProvider.GetRequiredService<ProcessPaymentDeclinedCommandHandler>();
                var result = await handler.Handle(new ProcessPaymentDeclinedCommand(
                    evt.DonationId, evt.CampaignId, evt.Reason, evt.Amount, evt.DonorId, evt.DonorEmail, evt.DonorName),
                    args.CancellationToken);

                if (result.IsFailure)
                {
                    _logger.LogWarning("Falha ao recusar doação {DonationId}: {Error}. Abandonando p/ reentrega.",
                        evt.DonationId, result.Error.Message);
                    await args.AbandonMessageAsync(args.Message);
                    return;
                }
            }
            else
            {
                _logger.LogWarning("Subject desconhecido '{Subject}' — dead-letter.", subject);
                await args.DeadLetterMessageAsync(args.Message, "UnknownSubject", subject);
                return;
            }

            await args.CompleteMessageAsync(args.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Erro ao processar resultado de pagamento; abandonando para reentrega.");
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

- [ ] **Step 6: Build + Commit**

Run: `dotnet build src/HackatonFiap.Donations.Infrastructure`
Expected: PASS.
```bash
git add src/HackatonFiap.Donations.Infrastructure
git commit -m "feat(infra): Service Bus publisher/consumer (roteia por Subject) + NoOp + SystemClock"
```

---

## Task 16: Infrastructure — `CampaignExpirationWorker` (BackgroundService periódico)

**Files:**
- Create: `src/HackatonFiap.Donations.Infrastructure/BackgroundServices/CampaignExpirationWorker.cs`

- [ ] **Step 1: Implementar o worker**

```csharp
using HackatonFiap.Donations.Application.Campaigns.ExpireDueCampaigns;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

namespace HackatonFiap.Donations.Infrastructure.BackgroundServices;

public sealed class CampaignExpirationWorker : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<CampaignExpirationWorker> _logger;
    private readonly TimeSpan _interval;

    public CampaignExpirationWorker(IServiceScopeFactory scopeFactory, ILogger<CampaignExpirationWorker> logger, TimeSpan interval)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
        _interval = interval;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(_interval);
        do
        {
            try
            {
                using var scope = _scopeFactory.CreateScope();
                var handler = scope.ServiceProvider.GetRequiredService<ExpireDueCampaignsCommandHandler>();
                await handler.Handle(stoppingToken);
            }
            catch (OperationCanceledException)
            {
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Erro ao expirar campanhas vencidas.");
            }
        }
        while (await timer.WaitForNextTickAsync(stoppingToken));
    }
}
```

- [ ] **Step 2: Build + Commit**

Run: `dotnet build src/HackatonFiap.Donations.Infrastructure`
Expected: PASS.
```bash
git add src/HackatonFiap.Donations.Infrastructure
git commit -m "feat(infra): CampaignExpirationWorker (conclusão por data periódica)"
```

---

## Task 17: API — mapeamento de erros, claims e controllers

**Files (em `src/HackatonFiap.Donations.API/`):**
- Create: `Common/HttpErrorMapping.cs`, `Common/ClaimsPrincipalExtensions.cs`
- Create: `Controllers/CampaignsController.cs`, `Controllers/DonationsController.cs`, `Controllers/TransparencyController.cs`

- [ ] **Step 1: Criar `Common/HttpErrorMapping.cs`** (mapeia `Error.Code` → status HTTP)

```csharp
using HackatonFiap.Donations.Domain.Common;
using Microsoft.AspNetCore.Mvc;

namespace HackatonFiap.Donations.API.Common;

public static class HttpErrorMapping
{
    public static IActionResult ToProblem(this Error error)
    {
        var status = error.Code switch
        {
            "Campaign.NotFound" => StatusCodes.Status404NotFound,
            "Donation.NotFound" => StatusCodes.Status404NotFound,
            "Campaign.InvalidStatusTransition" => StatusCodes.Status409Conflict,
            "Donation.CampaignNotFound" => StatusCodes.Status422UnprocessableEntity,
            "Donation.CampaignNotActive" => StatusCodes.Status422UnprocessableEntity,
            "Donation.OutsidePeriod" => StatusCodes.Status422UnprocessableEntity,
            _ => StatusCodes.Status400BadRequest // validações (goal, datas, título, amount, período VO)
        };

        return new ObjectResult(new { error = error.Code, message = error.Message }) { StatusCode = status };
    }
}
```

- [ ] **Step 2: Criar `Common/ClaimsPrincipalExtensions.cs`**

> As claims devem casar com o token emitido pela UserAPI. A extração é tolerante (tenta `sub`/`nameidentifier` para id, `email`, `name`/`unique_name`). **Verificar contra a UserAPI** ao integrar ponta-a-ponta.

```csharp
using System.Security.Claims;

namespace HackatonFiap.Donations.API.Common;

public static class ClaimsPrincipalExtensions
{
    public static Guid GetUserId(this ClaimsPrincipal user)
    {
        var raw = user.FindFirstValue(ClaimTypes.NameIdentifier)
                  ?? user.FindFirstValue("sub")
                  ?? user.FindFirstValue("nameid");
        return Guid.TryParse(raw, out var id) ? id : Guid.Empty;
    }

    public static string GetEmail(this ClaimsPrincipal user)
        => user.FindFirstValue(ClaimTypes.Email) ?? user.FindFirstValue("email") ?? string.Empty;

    public static string GetName(this ClaimsPrincipal user)
        => user.FindFirstValue("name")
           ?? user.FindFirstValue(ClaimTypes.Name)
           ?? user.FindFirstValue("unique_name")
           ?? string.Empty;
}
```

- [ ] **Step 3: Criar `Controllers/CampaignsController.cs`**

```csharp
using HackatonFiap.Donations.API.Common;
using HackatonFiap.Donations.Application.Campaigns;
using HackatonFiap.Donations.Application.Campaigns.ChangeCampaignStatus;
using HackatonFiap.Donations.Application.Campaigns.CreateCampaign;
using HackatonFiap.Donations.Application.Campaigns.GetCampaignById;
using HackatonFiap.Donations.Application.Campaigns.ListCampaigns;
using HackatonFiap.Donations.Application.Campaigns.UpdateCampaign;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace HackatonFiap.Donations.API.Controllers;

[ApiController]
[Route("api/campaigns")]
[Authorize(Policy = "ManagersOnly")]
public sealed class CampaignsController : ControllerBase
{
    public sealed record CreateCampaignRequest(string Title, string Description, DateTime StartDate, DateTime EndDate, decimal Goal);
    public sealed record UpdateCampaignRequest(string Title, string Description, DateTime StartDate, DateTime EndDate, decimal Goal);
    public sealed record ChangeStatusRequest(CampaignStatusAction Action);

    private readonly CreateCampaignCommandHandler _create;
    private readonly UpdateCampaignCommandHandler _update;
    private readonly ChangeCampaignStatusCommandHandler _changeStatus;
    private readonly GetCampaignByIdQueryHandler _getById;
    private readonly ListCampaignsQueryHandler _list;

    public CampaignsController(CreateCampaignCommandHandler create, UpdateCampaignCommandHandler update,
        ChangeCampaignStatusCommandHandler changeStatus, GetCampaignByIdQueryHandler getById, ListCampaignsQueryHandler list)
    {
        _create = create;
        _update = update;
        _changeStatus = changeStatus;
        _getById = getById;
        _list = list;
    }

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateCampaignRequest request, CancellationToken cancellationToken)
    {
        var command = new CreateCampaignCommand(request.Title, request.Description, request.StartDate, request.EndDate, request.Goal, User.GetUserId());
        var result = await _create.Handle(command, cancellationToken);
        if (result.IsFailure)
        {
            return result.Error.ToProblem();
        }

        return CreatedAtAction(nameof(GetById), new { id = result.Value }, new { id = result.Value });
    }

    [HttpPut("{id:guid}")]
    public async Task<IActionResult> Update(Guid id, [FromBody] UpdateCampaignRequest request, CancellationToken cancellationToken)
    {
        var result = await _update.Handle(new UpdateCampaignCommand(id, request.Title, request.Description, request.StartDate, request.EndDate, request.Goal), cancellationToken);
        return result.IsFailure ? result.Error.ToProblem() : NoContent();
    }

    [HttpPatch("{id:guid}/status")]
    public async Task<IActionResult> ChangeStatus(Guid id, [FromBody] ChangeStatusRequest request, CancellationToken cancellationToken)
    {
        var result = await _changeStatus.Handle(new ChangeCampaignStatusCommand(id, request.Action), cancellationToken);
        return result.IsFailure ? result.Error.ToProblem() : NoContent();
    }

    [HttpGet]
    public async Task<IActionResult> List(CancellationToken cancellationToken)
    {
        var result = await _list.Handle(new ListCampaignsQuery(), cancellationToken);
        return Ok(result.Value);
    }

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetById(Guid id, CancellationToken cancellationToken)
    {
        var result = await _getById.Handle(new GetCampaignByIdQuery(id), cancellationToken);
        return result.IsFailure ? result.Error.ToProblem() : Ok(result.Value);
    }
}
```

- [ ] **Step 4: Criar `Controllers/DonationsController.cs`**

```csharp
using HackatonFiap.Donations.API.Common;
using HackatonFiap.Donations.Application.Donations.CreateDonation;
using HackatonFiap.Donations.Application.Donations.GetDonationById;
using HackatonFiap.Donations.Domain.Enums;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace HackatonFiap.Donations.API.Controllers;

[ApiController]
[Route("api/donations")]
[Authorize(Policy = "DonorOnly")]
public sealed class DonationsController : ControllerBase
{
    public sealed record CreateDonationRequest(Guid CampaignId, decimal Amount, PaymentMethod PaymentMethod);

    private readonly CreateDonationCommandHandler _create;
    private readonly GetDonationByIdQueryHandler _getById;

    public DonationsController(CreateDonationCommandHandler create, GetDonationByIdQueryHandler getById)
    {
        _create = create;
        _getById = getById;
    }

    [HttpPost]
    public async Task<IActionResult> Create([FromBody] CreateDonationRequest request, CancellationToken cancellationToken)
    {
        var command = new CreateDonationCommand(request.CampaignId, request.Amount, request.PaymentMethod,
            User.GetUserId(), User.GetEmail(), User.GetName());
        var result = await _create.Handle(command, cancellationToken);
        if (result.IsFailure)
        {
            return result.Error.ToProblem();
        }

        return Accepted(new { donationId = result.Value, status = "Pending" });
    }

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetById(Guid id, CancellationToken cancellationToken)
    {
        var result = await _getById.Handle(new GetDonationByIdQuery(id, User.GetUserId()), cancellationToken);
        return result.IsFailure ? result.Error.ToProblem() : Ok(result.Value);
    }
}
```

- [ ] **Step 5: Criar `Controllers/TransparencyController.cs`**

```csharp
using HackatonFiap.Donations.Application.Transparency;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace HackatonFiap.Donations.API.Controllers;

[ApiController]
[Route("api/transparency")]
[AllowAnonymous]
public sealed class TransparencyController : ControllerBase
{
    private readonly ListActiveCampaignsQueryHandler _handler;

    public TransparencyController(ListActiveCampaignsQueryHandler handler) => _handler = handler;

    [HttpGet("campaigns")]
    public async Task<IActionResult> ListCampaigns(CancellationToken cancellationToken)
    {
        var result = await _handler.Handle(new ListActiveCampaignsQuery(), cancellationToken);
        return Ok(result.Value);
    }
}
```

- [ ] **Step 6: Commit** (a API ainda não compila sem o `Program.cs` da Task 18; commitar junto na Task 18)

Pular commit isolado; seguir para a Task 18.

---

## Task 18: API — `Program.cs`, appsettings, launchSettings + bootstrap

**Files (em `src/HackatonFiap.Donations.API/`):**
- Replace: `Program.cs`
- Create/Replace: `appsettings.json`, `appsettings.Development.json`, `Properties/launchSettings.json`
- Delete: `WeatherForecast.cs` e `Controllers/WeatherForecastController.cs` se o template os tiver criado.

- [ ] **Step 1: Remover artefatos do template**

```bash
cd "C:/eSolution/Projetos/hackaton-fiap/hackaton-fiap-donations"
rm -f src/HackatonFiap.Donations.API/WeatherForecast.cs src/HackatonFiap.Donations.API/Controllers/WeatherForecastController.cs
```

- [ ] **Step 2: Substituir `Program.cs`**

```csharp
using System.Text;
using Azure.Messaging.ServiceBus;
using HackatonFiap.Donations.Application.Abstractions;
using HackatonFiap.Donations.Application.Campaigns.ChangeCampaignStatus;
using HackatonFiap.Donations.Application.Campaigns.CreateCampaign;
using HackatonFiap.Donations.Application.Campaigns.ExpireDueCampaigns;
using HackatonFiap.Donations.Application.Campaigns.GetCampaignById;
using HackatonFiap.Donations.Application.Campaigns.ListCampaigns;
using HackatonFiap.Donations.Application.Campaigns.UpdateCampaign;
using HackatonFiap.Donations.Application.Donations.CreateDonation;
using HackatonFiap.Donations.Application.Donations.GetDonationById;
using HackatonFiap.Donations.Application.Donations.ProcessPaymentApproved;
using HackatonFiap.Donations.Application.Donations.ProcessPaymentDeclined;
using HackatonFiap.Donations.Application.Observability;
using HackatonFiap.Donations.Application.Transparency;
using HackatonFiap.Donations.Infrastructure.BackgroundServices;
using HackatonFiap.Donations.Infrastructure.Messaging;
using HackatonFiap.Donations.Infrastructure.Persistence;
using HackatonFiap.Donations.Infrastructure.Persistence.Repositories;
using HackatonFiap.Donations.Infrastructure.ReadStore;
using HackatonFiap.Donations.Infrastructure.Time;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Diagnostics.HealthChecks;
using Microsoft.Azure.Cosmos;
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
        .Enrich.WithProperty("ServiceName", DonationMetrics.ServiceName)
        .WriteTo.Console());

    builder.Services.AddControllers();
    builder.Services.AddEndpointsApiExplorer();
    builder.Services.AddSwaggerGen();

    // Database (SQL Server)
    var connectionString = configuration.GetValue<string>("ConnectionStrings:Default");
    if (string.IsNullOrWhiteSpace(connectionString))
    {
        throw new InvalidOperationException("ConnectionStrings:Default must be configured.");
    }
    builder.Services.AddDbContext<DonationsDbContext>(opt =>
        opt.UseSqlServer(connectionString, sql => sql.EnableRetryOnFailure(6, TimeSpan.FromSeconds(30), null)));

    // Repositories, UoW, clock
    builder.Services.AddScoped<ICampaignRepository, CampaignRepository>();
    builder.Services.AddScoped<IDonationRepository, DonationRepository>();
    builder.Services.AddScoped<IProcessedEventStore, ProcessedEventStore>();
    builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
    builder.Services.AddSingleton<IClock, SystemClock>();

    // CQRS handlers
    builder.Services.AddScoped<CreateCampaignCommandHandler>();
    builder.Services.AddScoped<UpdateCampaignCommandHandler>();
    builder.Services.AddScoped<ChangeCampaignStatusCommandHandler>();
    builder.Services.AddScoped<GetCampaignByIdQueryHandler>();
    builder.Services.AddScoped<ListCampaignsQueryHandler>();
    builder.Services.AddScoped<ExpireDueCampaignsCommandHandler>();
    builder.Services.AddScoped<CreateDonationCommandHandler>();
    builder.Services.AddScoped<GetDonationByIdQueryHandler>();
    builder.Services.AddScoped<ProcessPaymentApprovedCommandHandler>();
    builder.Services.AddScoped<ProcessPaymentDeclinedCommandHandler>();
    builder.Services.AddScoped<ListActiveCampaignsQueryHandler>();

    // Read store: Cosmos quando configurado; senão fallback in-memory (Development)
    var cosmosConnection = configuration.GetValue<string>("Cosmos:ConnectionString");
    if (!string.IsNullOrWhiteSpace(cosmosConnection))
    {
        var cosmosOptions = new CosmosOptions
        {
            ConnectionString = cosmosConnection,
            Database = configuration.GetValue<string>("Cosmos:Database") ?? "HackatonFiapDonations",
            Container = configuration.GetValue<string>("Cosmos:Container") ?? "campaigns"
        };
        builder.Services.AddSingleton(cosmosOptions);
        builder.Services.AddSingleton(_ => new CosmosClient(cosmosConnection));
        builder.Services.AddSingleton<ICampaignReadStore, CosmosCampaignReadStore>();
    }
    else if (builder.Environment.IsDevelopment())
    {
        Log.Warning("Cosmos:ConnectionString não configurado — usando InMemoryCampaignReadStore (Development).");
        builder.Services.AddSingleton<ICampaignReadStore, InMemoryCampaignReadStore>();
    }
    else
    {
        throw new InvalidOperationException("Cosmos:ConnectionString must be configured outside Development.");
    }

    // Service Bus: publisher (tópico de requisição) + consumer (tópico de resultado)
    var sbConnection = configuration.GetValue<string>("ServiceBus:ConnectionString");
    if (!string.IsNullOrWhiteSpace(sbConnection))
    {
        var requestTopic = configuration.GetValue<string>("ServiceBus:RequestTopic") ?? "donation-requested";
        var resultTopic = configuration.GetValue<string>("ServiceBus:ResultTopic") ?? "payment-result";
        var resultSubscription = configuration.GetValue<string>("ServiceBus:ResultSubscription") ?? "donations";

        builder.Services.AddSingleton(_ => new ServiceBusClient(sbConnection));
        builder.Services.AddSingleton<IEventPublisher>(sp => new ServiceBusEventPublisher(
            sp.GetRequiredService<ServiceBusClient>(), requestTopic,
            sp.GetRequiredService<ILogger<ServiceBusEventPublisher>>()));
        builder.Services.AddHostedService(sp => new PaymentResultConsumer(
            sp.GetRequiredService<ServiceBusClient>(), resultTopic, resultSubscription,
            sp.GetRequiredService<IServiceScopeFactory>(),
            sp.GetRequiredService<ILogger<PaymentResultConsumer>>()));
    }
    else if (builder.Environment.IsDevelopment())
    {
        Log.Warning("ServiceBus:ConnectionString não configurado — NoOpEventPublisher e consumer desabilitado (Development).");
        builder.Services.AddSingleton<IEventPublisher, NoOpEventPublisher>();
    }
    else
    {
        throw new InvalidOperationException("ServiceBus:ConnectionString must be configured outside Development.");
    }

    // Worker de expiração (sempre ativo)
    var expirationInterval = TimeSpan.FromSeconds(configuration.GetValue<int?>("Campaigns:ExpirationScanIntervalSeconds") ?? 60);
    builder.Services.AddHostedService(sp => new CampaignExpirationWorker(
        sp.GetRequiredService<IServiceScopeFactory>(),
        sp.GetRequiredService<ILogger<CampaignExpirationWorker>>(),
        expirationInterval));

    // Auth (JWT) — mesmas issuer/audience do ecossistema
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
    {
        options.AddPolicy("ManagersOnly", p => p.RequireRole("GestorONG", "Owner"));
        options.AddPolicy("DonorOnly", p => p.RequireRole("Doador"));
    });

    // Observability
    builder.Services.AddOpenTelemetry().WithMetrics(m => m
        .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService(DonationMetrics.ServiceName))
        .AddMeter(DonationMetrics.ServiceName)
        .AddAspNetCoreInstrumentation()
        .AddRuntimeInstrumentation()
        .AddPrometheusExporter());

    builder.Services.AddHealthChecks()
        .AddDbContextCheck<DonationsDbContext>(name: "sqlserver", tags: new[] { "ready" });

    var app = builder.Build();

    using (var scope = app.Services.CreateScope())
    {
        scope.ServiceProvider.GetRequiredService<DonationsDbContext>().Database.Migrate();
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

public partial class Program { }
```

- [ ] **Step 3: Substituir `appsettings.json`**

```json
{
  "Serilog": { "MinimumLevel": { "Default": "Information" } },
  "ConnectionStrings": {
    "Default": ""
  },
  "ServiceBus": {
    "ConnectionString": "",
    "RequestTopic": "donation-requested",
    "ResultTopic": "payment-result",
    "ResultSubscription": "donations"
  },
  "Cosmos": {
    "ConnectionString": "",
    "Database": "HackatonFiapDonations",
    "Container": "campaigns"
  },
  "Jwt": {
    "Issuer": "conexaosolidaria.local",
    "Audience": "conexaosolidaria.clients",
    "Key": ""
  },
  "Campaigns": { "ExpirationScanIntervalSeconds": 60 },
  "AllowedHosts": "*"
}
```

- [ ] **Step 4: Criar `appsettings.Development.json`**

```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft.AspNetCore": "Information",
        "Microsoft.EntityFrameworkCore": "Information"
      }
    }
  },
  "ConnectionStrings": {
    "Default": "Server=localhost,1433;Database=HackatonFiapDonationsDb;User Id=sa;Password=Your_password123;TrustServerCertificate=true;"
  },
  "ServiceBus": { "ConnectionString": "" },
  "Cosmos": { "ConnectionString": "" }
}
```

- [ ] **Step 5: Substituir `Properties/launchSettings.json`**

```json
{
  "$schema": "http://json.schemastore.org/launchsettings.json",
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "http://localhost:5120",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

- [ ] **Step 6: Build da solução inteira**

Run: `dotnet build HackatonFiap.Donations.sln`
Expected: PASS (todos os projetos, incluindo a API).

- [ ] **Step 7: Commit**

```bash
git add src/HackatonFiap.Donations.API
git commit -m "feat(api): controllers (campaigns/donations/transparency), Program.cs e configuração"
```

---

## Task 19: CI/CD e Kubernetes

**Files:**
- Replace: `.github/workflows/ci-cd.yaml` (renomear job/imagem para donations e usar a nova `.sln`)
- Modify: `.github/workflows/k8s/deployment.yaml` (imagem/labels/env de donations)
- Keep/adjust: `.github/workflows/k8s/secret-provider-class.yaml`

- [ ] **Step 1: Substituir `.github/workflows/ci-cd.yaml`** (CI mínima alinhada ao payments; manter o CD se os secrets existirem)

```yaml
name: CI/CD - DonationAPI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  IMAGE_NAME: hackatonfiap-donations

# Princípio do menor privilégio: o token só precisa ler o repo para checkout/build.
permissions:
  contents: read

jobs:
  ci:
    name: Build & Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'
      - run: dotnet restore HackatonFiap.Donations.sln
      - run: dotnet build HackatonFiap.Donations.sln --no-restore -c Release
      - run: dotnet test HackatonFiap.Donations.sln --no-build -c Release --verbosity minimal
      - name: Build Docker image
        run: docker build -t ${{ env.IMAGE_NAME }}:${{ github.sha }} .
```

> **Segurança (resposta ao security review do scaffold):** o arquivo `ci-cd.yaml` herdado do CatalogAPI é **substituído** por este; isso remove o bloco Trivy com `exit-code: '0'` (gate de segurança que não falhava) e o `trivy-action@master` (ref mutável). O `permissions: contents: read` no topo elimina as permissões default excessivas do `GITHUB_TOKEN`.
> O bloco de CD (deploy AKS) do CatalogAPI pode ser readicionado depois, trocando `fcg-catalog`→`hackatonfiap-donations`, `-f CatalogAPI/Dockerfile`→`-f Dockerfile`, e os manifests `k8s/*`. **Ao readicioná-lo, obrigatoriamente:** (a) usar `exit-code: '1'` no Trivy (com `ignore-unfixed: true` se quiser tolerar advisories sem patch) para que HIGH/CRITICAL **falhem** o job antes do push/deploy; (b) **pinar** `aquasecurity/trivy-action` (e demais actions de terceiros) por tag ou commit SHA (ex.: `@0.24.0`), nunca `@master`; (c) habilitar Dependabot para actions. Mantido fora do MVP de CI para não depender de secrets ausentes; registrar como follow-up.

- [ ] **Step 2: Atualizar `.github/workflows/k8s/deployment.yaml`**

Substituir nome/labels/imagem e as `env` do CatalogAPI (Redis/Mongo/AzureSearch) pelas da DonationAPI. Conteúdo:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hackatonfiap-donations
  namespace: fcg
  labels:
    app: hackatonfiap-donations
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hackatonfiap-donations
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  template:
    metadata:
      labels:
        app: hackatonfiap-donations
        azure.workload.identity/use: "true"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/metrics"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: sa-fcg-catalog
      imagePullSecrets:
        - name: acr-secret
      containers:
        - name: hackatonfiap-donations
          image: fcgregistry.azurecr.io/hackatonfiap-donations:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Production
            - name: ConnectionStrings__Default
              valueFrom:
                secretKeyRef:
                  name: hackatonfiap-donations-secret
                  key: ConnectionStrings__Default
            - name: ServiceBus__ConnectionString
              valueFrom:
                secretKeyRef:
                  name: hackatonfiap-donations-secret
                  key: ServiceBus__ConnectionString
            - name: Cosmos__ConnectionString
              valueFrom:
                secretKeyRef:
                  name: hackatonfiap-donations-secret
                  key: Cosmos__ConnectionString
            - name: Jwt__Key
              valueFrom:
                secretKeyRef:
                  name: hackatonfiap-donations-secret
                  key: Jwt__Key
          resources:
            requests:
              cpu: 50m
              memory: 96Mi
            limits:
              cpu: 500m
              memory: 384Mi
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 6
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 15
            timeoutSeconds: 5
            failureThreshold: 6
```

> Ajustar `secret-provider-class.yaml` para expor as chaves `ConnectionStrings__Default`, `ServiceBus__ConnectionString`, `Cosmos__ConnectionString`, `Jwt__Key` (nos moldes do que o cluster já usa). Probes alinhadas ao serviço: `/ready` (readiness) e `/health` (liveness).

- [ ] **Step 3: Commit**

```bash
git add .github
git commit -m "ci: pipeline e manifests k8s da DonationAPI (remove resíduos do CatalogAPI)"
```

---

## Task 20: Verificação final

- [ ] **Step 1: Build + testes da solução completa**

Run:
```bash
cd "C:/eSolution/Projetos/hackaton-fiap/hackaton-fiap-donations"
dotnet build HackatonFiap.Donations.sln -c Release
dotnet test HackatonFiap.Donations.sln -c Release --verbosity minimal
```
Expected: build PASS; **todos os testes verdes** (Period, Campaign, Donation, Campaign handlers, Donation handlers, PaymentResult handlers, ExpireDueCampaigns, ListActiveCampaigns, InMemoryCampaignReadStore).

- [ ] **Step 2: Smoke local (opcional, requer SQL Server)**

Run (com SQL em Docker e `ASPNETCORE_ENVIRONMENT=Development`):
```bash
dotnet run --project src/HackatonFiap.Donations.API
```
Validar: sobe sem Service Bus/Cosmos (NoOp + InMemory), aplica migration, `/health` 200, `/ready` 200 (com SQL), `/metrics` expõe Prometheus, Swagger lista `campaigns`/`donations`/`transparency`.

- [ ] **Step 3: Commit final / push** (se o usuário autorizar push)

```bash
git push origin main
```

> Lembrete: autoria **GabrielVeridico**, **sem** `Co-Authored-By`. Se o push retornar 403, usar o Git Credential Manager (não o helper do gh) — ver memória `git-push-hackaton-repos`.

---

## Notas finais para o executor

- **Contrato de eventos é imutável** (a PaymentAPI já está em produção): nomes de evento, campos, tópicos `donation-requested`/`payment-result`, subscription `donations`, subjects `PaymentApproved`/`PaymentDeclined`, enum `PaymentMethod` (`Pix`/`CreditCard`/`BankTransfer`), JSON com enum-as-string.
- **Idempotência**: o `ProcessedEvent` (PK = `DonationId`) é gravado na **mesma transação** da consolidação; a projeção no Cosmos ocorre após o commit e é re-projetável.
- **Janela do job de expiração**: `POST /api/donations` valida o período (`Period.Contains`) além do status — por isso o teste `Create_outside_period_returns_outside_period`.
- **Claims**: validar os tipos reais emitidos pela UserAPI ao integrar ponta-a-ponta (a extração é tolerante a `sub`/`nameidentifier`, `email`, `name`/`unique_name`).
- **Commits**: nunca incluir trailer `Co-Authored-By`/Claude (instrução do usuário).
