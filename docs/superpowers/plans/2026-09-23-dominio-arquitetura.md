# Domínio (Dinheiro VO) + Arquitetura (CQRS/Clean) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. **Execute os subagentes de implementação com o modelo Sonnet.**

**Goal:** Enriquecer `Dinheiro` como Value Object completo (`SaldoAtual` vira `Dinheiro`), introduzir inicializadores por camada (`Add<Camada>()`), corrigir o vazamento de infra na Api, mover criação de conta para `CriarContaCommand`, migrar o saldo histórico para agregação SQL e o tratamento de erros para ProblemDetails nativo.

**Architecture:** O domínio ganha aritmética no VO e a invariante de saldo não-negativo permanece na `Conta`. Cada camada expõe um único `Add<Camada>()`; o `Program.cs` vira composition root magra. `BancoDadosContext`/`ContaRepository` viram `internal`. A Api depende só de `IMediator`.

**Tech Stack:** .NET 10, EF Core 10 (value converter), MediatR, FluentValidation, ASP.NET Core ProblemDetails.

**Pré-requisito:** Plano `2026-09-23-fundacao-build-postgres.md` concluído (Postgres, Testcontainers, build lock ativos).

**Referências:** design doc seção 3, 5, 6; ADRs 007, 013, 014, 015, 016, 017.

---

## Estrutura de Arquivos

- **Modificar** `src/.../Dominio/Contas/Dinheiro.cs` — `Zero`, `Reconstituir`, operadores.
- **Modificar** `src/.../Dominio/Contas/Conta.cs` — `SaldoAtual` vira `Dinheiro`.
- **Criar** `src/.../Dominio/Properties/AssemblyInfo.cs` — `InternalsVisibleTo` Infra.
- **Modificar** `src/.../Infraestrutura/.../ContaConfiguration.cs` — value converter do saldo.
- **Modificar** `src/.../Aplicacao/.../ConsultarSaldoHandler.cs` — `SaldoAtual.Quantia`.
- **Modificar** `src/.../Aplicacao/.../SaldoResponse.cs` (se tipar saldo) — manter `decimal`.
- **Criar** `src/.../Dominio/DependencyInjection.cs` — `AddDominio()`.
- **Criar** `src/.../Aplicacao/DependencyInjection.cs` — `AddAplicacao()`.
- **Criar** `src/.../Infraestrutura/DependencyInjection.cs` — `AddInfraestrutura(IConfiguration)`.
- **Criar** `src/.../Api/DependencyInjection.cs` — `AddApi()`.
- **Modificar** `src/.../Api/Program.cs` — composition root magra.
- **Criar** `src/.../Aplicacao/Contas/Commands/CriarConta/{CriarContaCommand,CriarContaHandler,CriarContaValidator,ContaResponse}.cs`.
- **Modificar** `src/.../Api/Controllers/ContasController.cs` — só `IMediator`.
- **Modificar** `src/.../Infraestrutura/.../BancoDadosContext.cs` e `ContaRepository.cs` — `internal`.
- **Criar** `src/.../Infraestrutura/Properties/AssemblyInfo.cs` — `InternalsVisibleTo` Api.Testes.
- **Modificar** `src/.../Infraestrutura/.../ContaRepository.cs` — `ConsultarSaldoEmAsync` via SQL.
- **Modificar** `src/.../Api/Middlewares/TratadorDeExcecoesMiddleware.cs` — ProblemDetails nativo.

---

## Task 1: Dinheiro como VO rico (TDD)

**Files:**
- Modify: `src/MovimentacoesFinanceiras.Dominio/Contas/Dinheiro.cs`
- Test: `tests/Dominio.Testes/Contas/DinheiroTestes.cs`

- [ ] **Step 1: Escrever os testes que falham (novos comportamentos)**

Adicione ao final de `DinheiroTestes.cs`, antes do último `}`:

```csharp
    [Fact]
    public void Zero_TemQuantiaZero()
    {
        // Act & Assert
        Dinheiro.Zero.Quantia.Should().Be(0m);
    }

    [Fact]
    public void Somar_RetornaSomaDasQuantias()
    {
        // Arrange
        var a = Dinheiro.De(100m);
        var b = Dinheiro.De(50m);

        // Act
        var resultado = a + b;

        // Assert
        resultado.Quantia.Should().Be(150m);
    }

    [Fact]
    public void Subtrair_RetornaDiferencaDasQuantias()
    {
        // Arrange
        var a = Dinheiro.De(100m);
        var b = Dinheiro.De(30m);

        // Act
        var resultado = a - b;

        // Assert
        resultado.Quantia.Should().Be(70m);
    }

    [Fact]
    public void Maior_QuandoQuantiaSuperior_RetornaTrue()
    {
        // Act & Assert
        (Dinheiro.De(100m) > Dinheiro.De(50m)).Should().BeTrue();
    }

    [Fact]
    public void MenorOuIgual_QuandoQuantiasIguais_RetornaTrue()
    {
        // Act & Assert
        (Dinheiro.De(50m) <= Dinheiro.De(50m)).Should().BeTrue();
    }
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `dotnet test tests/Dominio.Testes --filter "FullyQualifiedName~DinheiroTestes"`
Expected: FALHA de compilação (`Zero`, operadores `+`/`-`/`>`/`<=` não existem).

- [ ] **Step 3: Implementar em `Dinheiro.cs`**

Adicione dentro da classe, após o construtor privado e o método `De`:

```csharp
    public static Dinheiro Zero { get; } = new(0m);

    internal static Dinheiro Reconstituir(decimal quantia) => new(quantia);

    public Dinheiro Somar(Dinheiro outro) => new(Quantia + outro.Quantia);
    public Dinheiro Subtrair(Dinheiro outro) => new(Quantia - outro.Quantia);

    public static Dinheiro operator +(Dinheiro left, Dinheiro right) => left.Somar(right);
    public static Dinheiro operator -(Dinheiro left, Dinheiro right) => left.Subtrair(right);
    public static bool operator >(Dinheiro left, Dinheiro right) => left.Quantia > right.Quantia;
    public static bool operator <(Dinheiro left, Dinheiro right) => left.Quantia < right.Quantia;
    public static bool operator >=(Dinheiro left, Dinheiro right) => left.Quantia >= right.Quantia;
    public static bool operator <=(Dinheiro left, Dinheiro right) => left.Quantia <= right.Quantia;
```

- [ ] **Step 4: Rodar e ver passar**

Run: `dotnet test tests/Dominio.Testes --filter "FullyQualifiedName~DinheiroTestes"`
Expected: todos PASSAM.

- [ ] **Step 5: Commit**

```bash
git add src/MovimentacoesFinanceiras.Dominio/Contas/Dinheiro.cs tests/Dominio.Testes/Contas/DinheiroTestes.cs
git commit -m "feat(dominio): Dinheiro como VO rico (Zero, aritmética, comparação)"
```

---

## Task 2: SaldoAtual vira Dinheiro (TDD)

**Files:**
- Modify: `src/MovimentacoesFinanceiras.Dominio/Contas/Conta.cs`
- Test: `tests/Dominio.Testes/Contas/ContaTestes.cs`

- [ ] **Step 1: Ajustar os testes existentes para o tipo Dinheiro**

Em `ContaTestes.cs`, onde houver asserção sobre `conta.SaldoAtual` comparando com `decimal` (ex: `.SaldoAtual.Should().Be(100m)`), troque para `.SaldoAtual.Quantia.Should().Be(100m)`. Onde a conta recém-criada esperar saldo zero, use `.SaldoAtual.Quantia.Should().Be(0m)`. (Leia o arquivo e ajuste todas as ocorrências.)

- [ ] **Step 2: Rodar e ver falhar (compilação)**

Run: `dotnet test tests/Dominio.Testes --filter "FullyQualifiedName~ContaTestes"`
Expected: FALHA de compilação enquanto `SaldoAtual` ainda for `decimal` e os testes usarem `.Quantia` — OU passa se os testes antigos ainda casarem. O objetivo é que, após o Step 3, tudo compile e passe.

- [ ] **Step 3: Alterar `Conta.cs`**

Substitua a propriedade e os métodos:

```csharp
    public Dinheiro SaldoAtual { get; private set; } = Dinheiro.Zero;
```

Na factory `Criar`, troque `SaldoAtual = 0m,` por `SaldoAtual = Dinheiro.Zero,`.

Reescreva `Creditar`:

```csharp
    public Lancamento Creditar(Dinheiro valor, string? descricao, string? chaveIdempotencia = null)
    {
        SaldoAtual += valor;
        VersaoLinha = Guid.NewGuid();
        var lancamento = Lancamento.CriarCredito(Id, valor.Quantia, descricao, chaveIdempotencia);
        _lancamentos.Add(lancamento);
        return lancamento;
    }
```

Reescreva `Debitar`:

```csharp
    public Lancamento Debitar(Dinheiro valor, string? descricao, string? chaveIdempotencia = null)
    {
        if (valor > SaldoAtual)
            throw new Excecoes.SaldoInsuficienteException(SaldoAtual.Quantia, valor);
        SaldoAtual -= valor;
        VersaoLinha = Guid.NewGuid();
        var lancamento = Lancamento.CriarDebito(Id, valor.Quantia, descricao, chaveIdempotencia);
        _lancamentos.Add(lancamento);
        return lancamento;
    }
```

- [ ] **Step 4: Rodar e ver passar**

Run: `dotnet test tests/Dominio.Testes`
Expected: todos os testes de domínio PASSAM.

- [ ] **Step 5: Commit**

```bash
git add src/MovimentacoesFinanceiras.Dominio/Contas/Conta.cs tests/Dominio.Testes/Contas/ContaTestes.cs
git commit -m "feat(dominio): SaldoAtual como Dinheiro"
```

---

## Task 3: Mapeamento EF do saldo (value converter) + InternalsVisibleTo

**Files:**
- Create: `src/MovimentacoesFinanceiras.Dominio/Properties/AssemblyInfo.cs`
- Modify: `src/MovimentacoesFinanceiras.Infraestrutura/Persistencia/Configurations/ContaConfiguration.cs`
- Modify: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Queries/ConsultarSaldo/ConsultarSaldoHandler.cs`

- [ ] **Step 1: Expor `Reconstituir` (internal) à Infraestrutura**

Crie `src/MovimentacoesFinanceiras.Dominio/Properties/AssemblyInfo.cs`:

```csharp
using System.Runtime.CompilerServices;

[assembly: InternalsVisibleTo("MovimentacoesFinanceiras.Infraestrutura")]
```

- [ ] **Step 2: Value converter no `ContaConfiguration`**

Substitua a linha do `SaldoAtual` (hoje `builder.Property(c => c.SaldoAtual).HasColumnName("saldo_atual").HasPrecision(18, 2).IsRequired();`) por:

```csharp
        builder.Property(c => c.SaldoAtual)
               .HasColumnName("saldo_atual")
               .HasConversion(d => d.Quantia, v => Dinheiro.Reconstituir(v))
               .HasPrecision(18, 2)
               .IsRequired();
```

- [ ] **Step 3: Ajustar `ConsultarSaldoHandler`**

`SaldoResponse` recebe um `decimal`. Como `SaldoAtual` agora é `Dinheiro`, troque a linha 15:

```csharp
        return new SaldoResponse(conta.SaldoAtual.Quantia, DateTime.UtcNow);
```

- [ ] **Step 4: Regenerar a migration (schema do saldo não muda de tipo, mas confirme)**

O saldo continua `numeric(18,2)`; o value converter não altera o schema. Rode o build; se o EF exigir nova migration por mudança de modelo, gere-a:

Run: `dotnet build -warnaserror`
Expected: `Build succeeded`. Se `dotnet test` (Task 5) acusar model-diff, rode:
```bash
dotnet ef migrations add SaldoComoDinheiro --project src/MovimentacoesFinanceiras.Infraestrutura --startup-project src/MovimentacoesFinanceiras.Api
```
(Provavelmente não será necessário — o tipo de coluna é idêntico.)

- [ ] **Step 5: Rodar toda a suíte**

Run: `dotnet test`
Expected: PASSAM (domínio + integração; o saldo persiste e materializa como `Dinheiro`).

- [ ] **Step 6: Commit**

```bash
git add src/
git commit -m "feat(infra): value converter de Dinheiro para SaldoAtual"
```

---

## Task 4: Inicializadores por camada + composition root magra

**Files:**
- Create: `src/MovimentacoesFinanceiras.Dominio/DependencyInjection.cs`
- Create: `src/MovimentacoesFinanceiras.Aplicacao/DependencyInjection.cs`
- Create: `src/MovimentacoesFinanceiras.Infraestrutura/DependencyInjection.cs`
- Create: `src/MovimentacoesFinanceiras.Api/DependencyInjection.cs`
- Modify: `src/MovimentacoesFinanceiras.Api/Program.cs`
- Modify: `src/MovimentacoesFinanceiras.Aplicacao/MovimentacoesFinanceiras.Aplicacao.csproj` (add MS.Extensions.DI.Abstractions se faltar)

- [ ] **Step 1: `AddDominio()`**

Crie `src/MovimentacoesFinanceiras.Dominio/DependencyInjection.cs`:

```csharp
using Microsoft.Extensions.DependencyInjection;

namespace MovimentacoesFinanceiras.Dominio;

public static class DependencyInjection
{
    public static IServiceCollection AddDominio(this IServiceCollection services)
    {
        // Sem domain services com dependência de DI no momento.
        // Placeholder honesto: mantém a simetria de composition root por camada.
        return services;
    }
}
```

Se o projeto Dominio não referenciar `Microsoft.Extensions.DependencyInjection.Abstractions`, adicione ao `.csproj`:
```xml
<ItemGroup>
  <PackageReference Include="Microsoft.Extensions.DependencyInjection.Abstractions" Version="10.*" />
</ItemGroup>
```

- [ ] **Step 2: `AddAplicacao()`**

Crie `src/MovimentacoesFinanceiras.Aplicacao/DependencyInjection.cs`:

```csharp
using FluentValidation;
using MediatR;
using Microsoft.Extensions.DependencyInjection;
using MovimentacoesFinanceiras.Aplicacao.Contas.Behaviors;
using MovimentacoesFinanceiras.Aplicacao.Contas.Commands.RegistrarMovimentacao;

namespace MovimentacoesFinanceiras.Aplicacao;

public static class DependencyInjection
{
    public static IServiceCollection AddAplicacao(this IServiceCollection services)
    {
        var assembly = typeof(RegistrarMovimentacaoCommand).Assembly;

        services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(assembly));
        services.AddValidatorsFromAssembly(assembly);
        services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));

        return services;
    }
}
```

- [ ] **Step 3: `AddInfraestrutura()`**

Crie `src/MovimentacoesFinanceiras.Infraestrutura/DependencyInjection.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using MovimentacoesFinanceiras.Dominio.Contas;
using MovimentacoesFinanceiras.Infraestrutura.Persistencia;
using MovimentacoesFinanceiras.Infraestrutura.Persistencia.Repositories;

namespace MovimentacoesFinanceiras.Infraestrutura;

public static class DependencyInjection
{
    public static IServiceCollection AddInfraestrutura(this IServiceCollection services, IConfiguration configuration)
    {
        services.AddDbContext<BancoDadosContext>(opcoes =>
            opcoes.UseNpgsql(configuration.GetConnectionString("Postgres")
                ?? "Host=localhost;Port=5432;Database=movimentacoes;Username=postgres;Password=postgres"));

        services.AddScoped<IContaRepository, ContaRepository>();

        services.AddHealthChecks()
            .AddDbContextCheck<BancoDadosContext>("banco-de-dados");

        return services;
    }
}
```

- [ ] **Step 4: `AddApi()`**

Crie `src/MovimentacoesFinanceiras.Api/DependencyInjection.cs`:

```csharp
using System.Text.Json.Serialization;

namespace MovimentacoesFinanceiras.Api;

public static class DependencyInjection
{
    public static IServiceCollection AddApi(this IServiceCollection services)
    {
        services.AddControllers()
            .AddJsonOptions(opcoes =>
                opcoes.JsonSerializerOptions.Converters.Add(new JsonStringEnumConverter()));

        services.AddEndpointsApiExplorer();
        services.AddSwaggerGen(opcoes =>
        {
            opcoes.SwaggerDoc("v1", new()
            {
                Title = "API de Movimentações Financeiras",
                Description = "Registra movimentações financeiras e consulta saldos de contas bancárias. " +
                              "Utiliza CQRS com ledger append-only para rastreabilidade completa.",
                Version = "v1"
            });
        });

        return services;
    }
}
```

- [ ] **Step 5: Emagrecer o `Program.cs`**

Substitua o `Program.cs` inteiro por:

```csharp
using Microsoft.EntityFrameworkCore;
using MovimentacoesFinanceiras.Aplicacao;
using MovimentacoesFinanceiras.Api;
using MovimentacoesFinanceiras.Dominio;
using MovimentacoesFinanceiras.Infraestrutura;
using MovimentacoesFinanceiras.Infraestrutura.Persistencia;
using Serilog;

var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((ctx, config) =>
    config
        .ReadFrom.Configuration(ctx.Configuration)
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .WriteTo.Console(new Serilog.Formatting.Json.JsonFormatter()));

builder.Services
    .AddDominio()
    .AddAplicacao()
    .AddInfraestrutura(builder.Configuration)
    .AddApi();

var app = builder.Build();

if (!app.Environment.IsEnvironment("Testing"))
{
    using var escopo = app.Services.CreateScope();
    var contexto = escopo.ServiceProvider.GetRequiredService<BancoDadosContext>();
    await contexto.Database.MigrateAsync();
}

app.UseSwagger();
app.UseSwaggerUI(opcoes =>
{
    opcoes.SwaggerEndpoint("/swagger/v1/swagger.json", "API de Movimentações Financeiras v1");
    opcoes.RoutePrefix = string.Empty;
});

app.UseSerilogRequestLogging();
app.UseMiddleware<MovimentacoesFinanceiras.Api.Middlewares.CorrelacaoIdMiddleware>();
app.UseMiddleware<MovimentacoesFinanceiras.Api.Middlewares.TratadorDeExcecoesMiddleware>();
app.MapControllers();
app.MapHealthChecks("/saude");

app.Run();

public partial class Program { }
```

- [ ] **Step 6: Build + testes**

Run: `dotnet build -warnaserror && dotnet test`
Expected: build limpo; todos os testes passam (o wiring foi movido, comportamento idêntico).

- [ ] **Step 7: Commit**

```bash
git add src/
git commit -m "refactor: inicializadores por camada e composition root magra"
```

---

## Task 5: CriarContaCommand (TDD) + remover DbContext do controller

**Files:**
- Create: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Commands/CriarConta/CriarContaCommand.cs`
- Create: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Commands/CriarConta/ContaResponse.cs`
- Create: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Commands/CriarConta/CriarContaHandler.cs`
- Create: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Commands/CriarConta/CriarContaValidator.cs`
- Modify: `src/MovimentacoesFinanceiras.Api/Controllers/ContasController.cs`
- Test: `tests/Api.Testes/Contas/ContasIntegracaoTestes.cs` (já existe o teste POST 201)

- [ ] **Step 1: Criar o Command e o Response**

`CriarContaCommand.cs`:
```csharp
using MediatR;

namespace MovimentacoesFinanceiras.Aplicacao.Contas.Commands.CriarConta;

public record CriarContaCommand(Guid ClienteId) : IRequest<ContaResponse>;
```

`ContaResponse.cs`:
```csharp
namespace MovimentacoesFinanceiras.Aplicacao.Contas.Commands.CriarConta;

public record ContaResponse(Guid Id, Guid ClienteId, decimal SaldoAtual, DateTime CriadoEm);
```

- [ ] **Step 2: Criar o Validator**

`CriarContaValidator.cs`:
```csharp
using FluentValidation;

namespace MovimentacoesFinanceiras.Aplicacao.Contas.Commands.CriarConta;

public class CriarContaValidator : AbstractValidator<CriarContaCommand>
{
    public CriarContaValidator()
    {
        RuleFor(c => c.ClienteId).NotEmpty();
    }
}
```

- [ ] **Step 3: Criar o Handler**

`CriarContaHandler.cs`:
```csharp
using MediatR;
using MovimentacoesFinanceiras.Dominio.Contas;

namespace MovimentacoesFinanceiras.Aplicacao.Contas.Commands.CriarConta;

public class CriarContaHandler(IContaRepository repositorio)
    : IRequestHandler<CriarContaCommand, ContaResponse>
{
    public async Task<ContaResponse> Handle(CriarContaCommand request, CancellationToken cancellationToken)
    {
        var conta = Conta.Criar(request.ClienteId);
        await repositorio.AdicionarAsync(conta, cancellationToken);
        await repositorio.SalvarAsync(cancellationToken);
        return new ContaResponse(conta.Id, conta.ClienteId, conta.SaldoAtual.Quantia, conta.CriadoEm);
    }
}
```

- [ ] **Step 4: Reescrever o `ContasController` para depender só de `IMediator`**

Substitua o arquivo inteiro:

```csharp
using MediatR;
using Microsoft.AspNetCore.Mvc;
using MovimentacoesFinanceiras.Aplicacao.Contas.Commands.CriarConta;
using MovimentacoesFinanceiras.Aplicacao.Contas.Commands.RegistrarMovimentacao;
using MovimentacoesFinanceiras.Aplicacao.Contas.Queries.ConsultarSaldo;
using MovimentacoesFinanceiras.Aplicacao.Contas.Queries.ConsultarSaldoEm;
using MovimentacoesFinanceiras.Aplicacao.Contas.Queries.ListarMovimentacoes;
using MovimentacoesFinanceiras.Dominio.Contas;

namespace MovimentacoesFinanceiras.Api.Controllers;

[ApiController]
[Route("contas")]
[Produces("application/json")]
public class ContasController(IMediator mediador) : ControllerBase
{
    /// <summary>Cria uma nova conta para um cliente.</summary>
    [HttpPost]
    [ProducesResponseType(typeof(ContaResponse), StatusCodes.Status201Created)]
    public async Task<IActionResult> CriarConta(
        [FromBody] CriarContaRequest request,
        CancellationToken cancellationToken)
    {
        var response = await mediador.Send(new CriarContaCommand(request.ClienteId), cancellationToken);
        return CreatedAtAction(nameof(ConsultarSaldo), new { id = response.Id }, response);
    }

    /// <summary>Registra uma movimentação financeira (crédito ou débito).</summary>
    [HttpPost("{id:guid}/movimentacoes")]
    [ProducesResponseType(typeof(LancamentoResponse), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(StatusCodes.Status422UnprocessableEntity)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> RegistrarMovimentacao(
        Guid id,
        [FromBody] RegistrarMovimentacaoRequest request,
        [FromHeader(Name = "Idempotency-Key")] string? chaveIdempotencia,
        CancellationToken cancellationToken)
    {
        var command = new RegistrarMovimentacaoCommand(id, request.Tipo, request.Valor, request.Descricao, chaveIdempotencia);
        var response = await mediador.Send(command, cancellationToken);
        return StatusCode(StatusCodes.Status201Created, response);
    }

    /// <summary>Consulta o saldo da conta (atual ou point-in-time com ?em=).</summary>
    [HttpGet("{id:guid}/saldo")]
    [ProducesResponseType(typeof(SaldoResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(SaldoHistoricoResponse), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> ConsultarSaldo(
        Guid id,
        [FromQuery] DateTime? em,
        CancellationToken cancellationToken)
    {
        if (em.HasValue)
        {
            var queryHistorico = new ConsultarSaldoEmQuery(id, em.Value.ToUniversalTime());
            return Ok(await mediador.Send(queryHistorico, cancellationToken));
        }

        return Ok(await mediador.Send(new ConsultarSaldoQuery(id), cancellationToken));
    }

    /// <summary>Lista o extrato paginado da conta.</summary>
    [HttpGet("{id:guid}/movimentacoes")]
    [ProducesResponseType(StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> ListarMovimentacoes(
        Guid id,
        [FromQuery] int pagina = 1,
        [FromQuery] int tamanhoPagina = 20,
        CancellationToken cancellationToken = default)
    {
        var query = new ListarMovimentacoesQuery(id, pagina, tamanhoPagina);
        return Ok(await mediador.Send(query, cancellationToken));
    }
}

public record CriarContaRequest(Guid ClienteId);
public record RegistrarMovimentacaoRequest(TipoLancamento Tipo, decimal Valor, string? Descricao);
```

> Nota: o extrato foi movido para uma Query (`ListarMovimentacoesQuery`) para eliminar o último acesso direto a `BancoDadosContext`/`IContaRepository` no controller. Isso é feito na Task 6.

- [ ] **Step 5: Build (vai falhar — falta ListarMovimentacoesQuery)**

Run: `dotnet build -warnaserror`
Expected: FALHA (namespace `ListarMovimentacoes` ainda não existe). Prosseguir para Task 6 antes de testar. Não commite ainda.

---

## Task 6: ListarMovimentacoesQuery (elimina último acesso a infra no controller)

**Files:**
- Create: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Queries/ListarMovimentacoes/ListarMovimentacoesQuery.cs`
- Create: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Queries/ListarMovimentacoes/ExtratoResponse.cs`
- Create: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Queries/ListarMovimentacoes/ListarMovimentacoesHandler.cs`

- [ ] **Step 1: Query + Response**

`ListarMovimentacoesQuery.cs`:
```csharp
using MediatR;

namespace MovimentacoesFinanceiras.Aplicacao.Contas.Queries.ListarMovimentacoes;

public record ListarMovimentacoesQuery(Guid ContaId, int Pagina, int TamanhoPagina)
    : IRequest<ExtratoResponse>;
```

`ExtratoResponse.cs`:
```csharp
using MovimentacoesFinanceiras.Dominio.Contas;

namespace MovimentacoesFinanceiras.Aplicacao.Contas.Queries.ListarMovimentacoes;

public record ItemExtrato(Guid Id, string Tipo, decimal Valor, string? Descricao, DateTime CriadoEm);

public record ExtratoResponse(IReadOnlyList<ItemExtrato> Itens, int Pagina, int TamanhoPagina, int Total);
```

- [ ] **Step 2: Handler (valida existência da conta e pagina)**

`ListarMovimentacoesHandler.cs`:
```csharp
using MediatR;
using MovimentacoesFinanceiras.Dominio.Contas;
using MovimentacoesFinanceiras.Dominio.Contas.Excecoes;

namespace MovimentacoesFinanceiras.Aplicacao.Contas.Queries.ListarMovimentacoes;

public class ListarMovimentacoesHandler(IContaRepository repositorio)
    : IRequestHandler<ListarMovimentacoesQuery, ExtratoResponse>
{
    public async Task<ExtratoResponse> Handle(ListarMovimentacoesQuery request, CancellationToken cancellationToken)
    {
        var conta = await repositorio.ObterPorIdAsync(request.ContaId, cancellationToken)
            ?? throw new ContaNaoEncontradaException(request.ContaId);

        var (itens, total) = await repositorio.ListarLancamentosAsync(
            request.ContaId, request.Pagina, request.TamanhoPagina, cancellationToken);

        var mapeados = itens
            .Select(l => new ItemExtrato(l.Id, l.Tipo.ToString(), l.Valor, l.Descricao, l.CriadoEm))
            .ToList();

        return new ExtratoResponse(mapeados, request.Pagina, request.TamanhoPagina, total);
    }
}
```

- [ ] **Step 3: Build + testes**

Run: `dotnet build -warnaserror && dotnet test`
Expected: build limpo; testes de integração passam (o teste POST /contas → 201 e o extrato continuam válidos; o corpo do extrato agora vem do `ExtratoResponse` — se algum teste assertava chaves específicas do JSON anônimo antigo, ajuste-o para as chaves do record: `itens`, `pagina`, `tamanhoPagina`, `total`).

- [ ] **Step 4: Commit (Tasks 5+6 juntas)**

```bash
git add src/
git commit -m "feat: CriarContaCommand e ListarMovimentacoesQuery — controller só depende de IMediator"
```

---

## Task 7: Encapsular infra (internal + InternalsVisibleTo)

**Files:**
- Modify: `src/MovimentacoesFinanceiras.Infraestrutura/Persistencia/BancoDadosContext.cs`
- Modify: `src/MovimentacoesFinanceiras.Infraestrutura/Persistencia/Repositories/ContaRepository.cs`
- Create: `src/MovimentacoesFinanceiras.Infraestrutura/Properties/AssemblyInfo.cs`

- [ ] **Step 1: `InternalsVisibleTo` para os testes**

Crie `src/MovimentacoesFinanceiras.Infraestrutura/Properties/AssemblyInfo.cs`:

```csharp
using System.Runtime.CompilerServices;

[assembly: InternalsVisibleTo("Api.Testes")]
```

- [ ] **Step 2: Tornar `BancoDadosContext` internal**

Em `BancoDadosContext.cs`, troque `public class BancoDadosContext` por `internal class BancoDadosContext`. O construtor pode permanecer `public` (necessário para o EF); a classe internal já restringe o acesso externo.

> Atenção: `AddDbContextCheck<BancoDadosContext>` em `AddInfraestrutura` está no mesmo assembly — OK. `AplicacaoFactory` (Api.Testes) acessa via `InternalsVisibleTo` — OK.

- [ ] **Step 3: Tornar `ContaRepository` internal**

Em `ContaRepository.cs`, troque `public class ContaRepository` por `internal sealed class ContaRepository`. O registro `AddScoped<IContaRepository, ContaRepository>()` está em `AddInfraestrutura` (mesmo assembly) — OK.

- [ ] **Step 4: Build + testes**

Run: `dotnet build -warnaserror && dotnet test`
Expected: build limpo (a Api não referencia mais `BancoDadosContext`/`ContaRepository` diretamente — isso foi removido nas Tasks 5-6). Testes passam.

> Se o build falhar por a Api ainda referenciar tipos internal da Infra, é sinal de vazamento residual — localize e mova para uma Query/Command (não reabra o acesso).

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "refactor(infra): BancoDadosContext e ContaRepository internos"
```

---

## Task 8: Saldo histórico via agregação SQL

**Files:**
- Modify: `src/MovimentacoesFinanceiras.Infraestrutura/Persistencia/Repositories/ContaRepository.cs:25-32`
- Test: `tests/Api.Testes/Contas/ContasIntegracaoTestes.cs` (teste de saldo histórico já existe)

- [ ] **Step 1: Reescrever `ConsultarSaldoEmAsync` para somar no banco**

Substitua o método (linhas 25-32):

```csharp
    public async Task<decimal> ConsultarSaldoEmAsync(Guid contaId, DateTime dataReferencia, CancellationToken cancellationToken = default)
    {
        return await _contexto.Lancamentos
            .Where(l => l.ContaId == contaId && l.CriadoEm <= dataReferencia)
            .SumAsync(l => l.Tipo == TipoLancamento.Credito ? l.Valor : -l.Valor, cancellationToken);
    }
```

> `SumAsync` com o ternário é traduzido pelo Npgsql para `SUM(CASE WHEN ... THEN valor ELSE -valor END)` — a soma ocorre no banco, não em memória. Respeita a regra "sem `else`" (ternário, não `if/else`).

- [ ] **Step 2: Build + rodar o teste de saldo histórico**

Run: `dotnet test tests/Api.Testes --filter "FullyQualifiedName~Saldo"`
Expected: PASSA. O resultado numérico é idêntico ao da soma em memória.

> Se o Npgsql lançar erro de tradução LINQ→SQL (improvável para este ternário), o fallback é materializar só as colunas necessárias: `.Select(l => new { l.Tipo, l.Valor })` antes do `Sum` em memória — mas tente a tradução SQL primeiro.

- [ ] **Step 3: Rodar suíte completa**

Run: `dotnet test`
Expected: todos passam.

- [ ] **Step 4: Commit**

```bash
git add src/
git commit -m "perf(infra): saldo histórico via agregação SQL (SumAsync)"
```

---

## Task 9: ProblemDetails nativo

**Files:**
- Modify: `src/MovimentacoesFinanceiras.Api/DependencyInjection.cs`
- Modify: `src/MovimentacoesFinanceiras.Api/Middlewares/TratadorDeExcecoesMiddleware.cs`
- Test: `tests/Api.Testes/Contas/ContasIntegracaoTestes.cs` (400/404/422)

- [ ] **Step 1: Registrar ProblemDetails em `AddApi()`**

Em `src/MovimentacoesFinanceiras.Api/DependencyInjection.cs`, adicione no início do método `AddApi`, antes de `AddControllers`:

```csharp
        services.AddProblemDetails();
```

- [ ] **Step 2: Reescrever o middleware usando `IProblemDetailsService`**

Substitua o `TratadorDeExcecoesMiddleware.cs` inteiro:

```csharp
using FluentValidation;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using MovimentacoesFinanceiras.Dominio.Contas.Excecoes;

namespace MovimentacoesFinanceiras.Api.Middlewares;

public class TratadorDeExcecoesMiddleware(
    RequestDelegate proximo,
    IProblemDetailsService problemDetailsService,
    ILogger<TratadorDeExcecoesMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext contexto)
    {
        try
        {
            await proximo(contexto);
        }
        catch (ValidationException ex)
        {
            await EscreverProblema(contexto, StatusCodes.Status400BadRequest,
                "A requisição contém dados inválidos", null,
                new Dictionary<string, object?> { ["erros"] = ex.Errors.Select(e => e.ErrorMessage) });
        }
        catch (SaldoInsuficienteException ex)
        {
            await EscreverProblema(contexto, StatusCodes.Status422UnprocessableEntity,
                "Saldo insuficiente para realizar o débito", ex.Message, null);
        }
        catch (ContaNaoEncontradaException ex)
        {
            await EscreverProblema(contexto, StatusCodes.Status404NotFound,
                "Conta não encontrada", ex.Message, null);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Erro nao tratado");
            await EscreverProblema(contexto, StatusCodes.Status500InternalServerError,
                "Ocorreu um erro interno. Tente novamente.", null, null);
        }
    }

    private async Task EscreverProblema(
        HttpContext contexto, int status, string titulo, string? detalhe,
        IDictionary<string, object?>? extensoes)
    {
        contexto.Response.StatusCode = status;

        var problema = new ProblemDetails
        {
            Status = status,
            Title = titulo,
            Detail = detalhe,
            Instance = contexto.Request.Path
        };

        if (extensoes is not null)
        {
            foreach (var (chave, valor) in extensoes)
                problema.Extensions[chave] = valor;
        }

        if (contexto.Items.TryGetValue("correlacao_id", out var correlacao) && correlacao is not null)
            problema.Extensions["correlacao_id"] = correlacao;

        await problemDetailsService.WriteAsync(new ProblemDetailsContext
        {
            HttpContext = contexto,
            ProblemDetails = problema
        });
    }
}
```

> Observação: se `CorrelacaoIdMiddleware` não armazena `correlacao_id` em `HttpContext.Items`, a linha do `TryGetValue` simplesmente não adiciona a extension (sem erro). Opcionalmente ajuste o `CorrelacaoIdMiddleware` para also fazer `contexto.Items["correlacao_id"] = id;` — verifique o arquivo antes.

- [ ] **Step 3: Build**

Run: `dotnet build -warnaserror`
Expected: `Build succeeded`.

- [ ] **Step 4: Rodar os testes de erro**

Run: `dotnet test tests/Api.Testes`
Expected: PASSAM. Se algum teste assertava as chaves antigas do corpo (`tipo`, `titulo`, `detalhe`), ajuste para o schema RFC-7807 nativo (`title`, `detail`, `status`, `type`) e a extension `erros`. O `Content-Type` continua `application/problem+json` (padrão do ProblemDetails).

- [ ] **Step 5: Commit**

```bash
git add src/ tests/
git commit -m "refactor(api): tratamento de erros via ProblemDetails nativo (RFC-7807)"
```

---

## Self-Review (preenchido)

**Spec coverage:**
- ADR-014 SaldoAtual Dinheiro → Tasks 1, 2, 3 ✓
- ADR-013 Lancamento.Valor decimal → mantido (factories usam `valor.Quantia`) ✓
- ADR-007 inicializadores → Task 4 ✓
- ADR-015 encapsular infra → Tasks 5, 6, 7 ✓
- ADR-016 saldo SQL → Task 8 ✓
- ADR-017 ProblemDetails → Task 9 ✓

**Placeholder scan:** sem TBD; todo passo tem código/comando.

**Type consistency:** `Dinheiro.Reconstituir` (Task 1) usado no converter (Task 3); `SaldoAtual.Quantia` consistente em Conta, ConsultarSaldoHandler, CriarContaHandler; `ContaResponse`/`ExtratoResponse`/`ListarMovimentacoesQuery` definidos antes de serem usados no controller; `ItemExtrato` definido junto do `ExtratoResponse`.

**Dependência entre tasks:** Task 5 deixa o build quebrado de propósito até a Task 6 (documentado). Executar 5 e 6 na sequência sem testar entre elas.

**Ponto de atenção:** o `RegistrarMovimentacaoHandler` chama `Dinheiro.De(request.Valor)` — continua válido (entrada > 0). Nenhuma mudança necessária lá.
