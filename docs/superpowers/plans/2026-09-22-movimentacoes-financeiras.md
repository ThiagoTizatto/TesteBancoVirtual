# Sistema de Movimentações Financeiras — Plano de Implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implementar sistema bancário de movimentações financeiras com CQRS + Ledger append-only, concorrência otimista com retry automático (Polly), idempotência, logging estruturado (Serilog) e Swagger em português.

**Architecture:** Clean Architecture + DDD. Domínio puro sem dependências externas. Application com CQRS via MediatR + pipeline de validação (FluentValidation) + Polly retry. Infrastructure com EF Core 8 + SQLite. API com ASP.NET Core 8 + Serilog + Swagger. Toda a nomenclatura em português (Ubiquitous Language).

**Tech Stack:** .NET 8, ASP.NET Core 8, EF Core 8 + SQLite, MediatR 12, FluentValidation 11, Polly 8, Serilog.AspNetCore, Swashbuckle.AspNetCore, xUnit, FluentAssertions, Microsoft.AspNetCore.Mvc.Testing

---

## Mapa de Arquivos

```
desafio-banco/
├── MovimentacoesFinanceiras.sln
├── src/
│   ├── MovimentacoesFinanceiras.Dominio/
│   │   ├── MovimentacoesFinanceiras.Dominio.csproj
│   │   └── Contas/
│   │       ├── TipoLancamento.cs
│   │       ├── Dinheiro.cs
│   │       ├── Lancamento.cs
│   │       ├── Conta.cs
│   │       ├── IContaRepositorio.cs
│   │       └── Excecoes/
│   │           ├── SaldoInsuficienteException.cs
│   │           └── ContaNaoEncontradaException.cs
│   │
│   ├── MovimentacoesFinanceiras.Aplicacao/
│   │   ├── MovimentacoesFinanceiras.Aplicacao.csproj
│   │   └── Contas/
│   │       ├── Comportamentos/
│   │       │   └── ValidacaoBehavior.cs
│   │       ├── Comandos/
│   │       │   └── RegistrarMovimentacao/
│   │       │       ├── RegistrarMovimentacaoComando.cs
│   │       │       ├── RegistrarMovimentacaoManipulador.cs
│   │       │       ├── RegistrarMovimentacaoValidador.cs
│   │       │       └── LancamentoResposta.cs
│   │       └── Consultas/
│   │           ├── ConsultarSaldo/
│   │           │   ├── ConsultarSaldoConsulta.cs
│   │           │   ├── ConsultarSaldoManipulador.cs
│   │           │   └── SaldoResposta.cs
│   │           └── ConsultarSaldoEm/
│   │               ├── ConsultarSaldoEmConsulta.cs
│   │               ├── ConsultarSaldoEmManipulador.cs
│   │               └── SaldoHistoricoResposta.cs
│   │
│   ├── MovimentacoesFinanceiras.Infraestrutura/
│   │   ├── MovimentacoesFinanceiras.Infraestrutura.csproj
│   │   └── Persistencia/
│   │       ├── ContextoBancoDados.cs
│   │       ├── Repositorios/
│   │       │   └── ContaRepositorio.cs
│   │       └── Configuracoes/
│   │           ├── ContaConfiguracao.cs
│   │           └── LancamentoConfiguracao.cs
│   │
│   └── MovimentacoesFinanceiras.Api/
│       ├── MovimentacoesFinanceiras.Api.csproj
│       ├── Controladores/
│       │   └── ContasControlador.cs
│       ├── Middlewares/
│       │   ├── CorrelacaoIdMiddleware.cs
│       │   └── TratadorDeExcecoesMiddleware.cs
│       ├── Program.cs
│       └── appsettings.json
│
├── tests/
│   ├── Dominio.Testes/
│   │   ├── Dominio.Testes.csproj
│   │   └── Contas/
│   │       ├── ContaTestes.cs
│   │       └── DinheiroTestes.cs
│   └── Api.Testes/
│       ├── Api.Testes.csproj
│       ├── FabricaDeAplicacao.cs
│       └── Contas/
│           ├── ContasIntegracaoTestes.cs
│           └── ContasConcorrenciaTestes.cs
│
└── README.md
```

---

## Tarefa 1: Scaffold da solução

**Arquivos:**
- Criar: `MovimentacoesFinanceiras.sln` e todos os `.csproj`

- [ ] **Passo 1: Criar solução e projetos**

```bash
cd C:\Users\thiagots\Documents\desafio-banco
dotnet new sln -n MovimentacoesFinanceiras
dotnet new classlib -n MovimentacoesFinanceiras.Dominio -o src/MovimentacoesFinanceiras.Dominio --framework net8.0
dotnet new classlib -n MovimentacoesFinanceiras.Aplicacao -o src/MovimentacoesFinanceiras.Aplicacao --framework net8.0
dotnet new classlib -n MovimentacoesFinanceiras.Infraestrutura -o src/MovimentacoesFinanceiras.Infraestrutura --framework net8.0
dotnet new webapi -n MovimentacoesFinanceiras.Api -o src/MovimentacoesFinanceiras.Api --framework net8.0
dotnet new xunit -n Dominio.Testes -o tests/Dominio.Testes --framework net8.0
dotnet new xunit -n Api.Testes -o tests/Api.Testes --framework net8.0
```

- [ ] **Passo 2: Adicionar projetos à solução**

```bash
dotnet sln add src/MovimentacoesFinanceiras.Dominio/MovimentacoesFinanceiras.Dominio.csproj
dotnet sln add src/MovimentacoesFinanceiras.Aplicacao/MovimentacoesFinanceiras.Aplicacao.csproj
dotnet sln add src/MovimentacoesFinanceiras.Infraestrutura/MovimentacoesFinanceiras.Infraestrutura.csproj
dotnet sln add src/MovimentacoesFinanceiras.Api/MovimentacoesFinanceiras.Api.csproj
dotnet sln add tests/Dominio.Testes/Dominio.Testes.csproj
dotnet sln add tests/Api.Testes/Api.Testes.csproj
```

- [ ] **Passo 3: Adicionar referências entre projetos**

```bash
dotnet add src/MovimentacoesFinanceiras.Aplicacao reference src/MovimentacoesFinanceiras.Dominio
dotnet add src/MovimentacoesFinanceiras.Infraestrutura reference src/MovimentacoesFinanceiras.Dominio
dotnet add src/MovimentacoesFinanceiras.Api reference src/MovimentacoesFinanceiras.Aplicacao
dotnet add src/MovimentacoesFinanceiras.Api reference src/MovimentacoesFinanceiras.Infraestrutura
dotnet add tests/Dominio.Testes reference src/MovimentacoesFinanceiras.Dominio
dotnet add tests/Api.Testes reference src/MovimentacoesFinanceiras.Api
```

- [ ] **Passo 4: Instalar pacotes NuGet**

```bash
# Aplicacao
dotnet add src/MovimentacoesFinanceiras.Aplicacao package MediatR --version 12.*
dotnet add src/MovimentacoesFinanceiras.Aplicacao package FluentValidation --version 11.*
dotnet add src/MovimentacoesFinanceiras.Aplicacao package Polly --version 8.*

# Infraestrutura
dotnet add src/MovimentacoesFinanceiras.Infraestrutura package Microsoft.EntityFrameworkCore --version 8.*
dotnet add src/MovimentacoesFinanceiras.Infraestrutura package Microsoft.EntityFrameworkCore.Sqlite --version 8.*

# Api
dotnet add src/MovimentacoesFinanceiras.Api package Serilog.AspNetCore --version 8.*
dotnet add src/MovimentacoesFinanceiras.Api package Serilog.Sinks.Console --version 5.*
dotnet add src/MovimentacoesFinanceiras.Api package Swashbuckle.AspNetCore --version 6.*
dotnet add src/MovimentacoesFinanceiras.Api package Microsoft.EntityFrameworkCore.Design --version 8.*

# Testes
dotnet add tests/Dominio.Testes package FluentAssertions --version 6.*
dotnet add tests/Api.Testes package Microsoft.AspNetCore.Mvc.Testing --version 8.*
dotnet add tests/Api.Testes package FluentAssertions --version 6.*
dotnet add tests/Api.Testes package Microsoft.EntityFrameworkCore.InMemory --version 8.*
```

- [ ] **Passo 5: Remover arquivos gerados desnecessários**

```bash
rm src/MovimentacoesFinanceiras.Dominio/Class1.cs
rm src/MovimentacoesFinanceiras.Aplicacao/Class1.cs
rm src/MovimentacoesFinanceiras.Infraestrutura/Class1.cs
rm src/MovimentacoesFinanceiras.Api/WeatherForecast.cs
rm src/MovimentacoesFinanceiras.Api/Controllers/WeatherForecastController.cs
```

- [ ] **Passo 6: Verificar que a solução compila**

```bash
dotnet build
```
Esperado: `Build succeeded. 0 Warning(s). 0 Error(s).`

- [ ] **Passo 7: Commit**

```bash
git init
git add .
git commit -m "feat: scaffold da solucao com projetos e referencias"
```

---

## Tarefa 2: Domínio — TipoLancamento, Dinheiro e testes

**Arquivos:**
- Criar: `src/MovimentacoesFinanceiras.Dominio/Contas/TipoLancamento.cs`
- Criar: `src/MovimentacoesFinanceiras.Dominio/Contas/Dinheiro.cs`
- Criar: `tests/Dominio.Testes/Contas/DinheiroTestes.cs`

- [ ] **Passo 1: Escrever os testes de Dinheiro**

Criar `tests/Dominio.Testes/Contas/DinheiroTestes.cs`:

```csharp
using FluentAssertions;
using MovimentacoesFinanceiras.Dominio.Contas;

namespace Dominio.Testes.Contas;

public class DinheiroTestes
{
    [Fact]
    public void De_QuandoValorPositivo_CriaDinheiro()
    {
        var dinheiro = Dinheiro.De(100m);
        dinheiro.Quantia.Should().Be(100m);
    }

    [Fact]
    public void De_QuandoValorZero_LancaArgumentException()
    {
        var acao = () => Dinheiro.De(0m);
        acao.Should().Throw<ArgumentException>()
            .WithMessage("*deve ser maior que zero*");
    }

    [Fact]
    public void De_QuandoValorNegativo_LancaArgumentException()
    {
        var acao = () => Dinheiro.De(-50m);
        acao.Should().Throw<ArgumentException>()
            .WithMessage("*deve ser maior que zero*");
    }

    [Fact]
    public void Igualdade_QuandoMesmaQuantia_SaoIguais()
    {
        Dinheiro.De(100m).Should().Be(Dinheiro.De(100m));
    }

    [Fact]
    public void ToString_RetornaFormatoMonetario()
    {
        Dinheiro.De(1500.50m).ToString().Should().Contain("1.500,50");
    }
}
```

- [ ] **Passo 2: Executar testes — confirmar falha**

```bash
dotnet test tests/Dominio.Testes
```
Esperado: FAIL — `Dinheiro` não existe.

- [ ] **Passo 3: Criar TipoLancamento**

Criar `src/MovimentacoesFinanceiras.Dominio/Contas/TipoLancamento.cs`:

```csharp
namespace MovimentacoesFinanceiras.Dominio.Contas;

public enum TipoLancamento
{
    Credito,
    Debito
}
```

- [ ] **Passo 4: Criar Dinheiro**

Criar `src/MovimentacoesFinanceiras.Dominio/Contas/Dinheiro.cs`:

```csharp
namespace MovimentacoesFinanceiras.Dominio.Contas;

public sealed class Dinheiro : IEquatable<Dinheiro>
{
    public decimal Quantia { get; }

    private Dinheiro(decimal quantia) => Quantia = quantia;

    public static Dinheiro De(decimal quantia)
    {
        if (quantia <= 0)
            throw new ArgumentException("O valor deve ser maior que zero.", nameof(quantia));
        return new Dinheiro(quantia);
    }

    public bool Equals(Dinheiro? other) => other is not null && Quantia == other.Quantia;
    public override bool Equals(object? obj) => obj is Dinheiro d && Equals(d);
    public override int GetHashCode() => Quantia.GetHashCode();
    public override string ToString() => $"R$ {Quantia:N2}";
}
```

- [ ] **Passo 5: Executar testes — confirmar aprovação**

```bash
dotnet test tests/Dominio.Testes
```
Esperado: PASS — 5 testes passando.

- [ ] **Passo 6: Commit**

```bash
git add src/MovimentacoesFinanceiras.Dominio/Contas/TipoLancamento.cs
git add src/MovimentacoesFinanceiras.Dominio/Contas/Dinheiro.cs
git add tests/Dominio.Testes/Contas/DinheiroTestes.cs
git commit -m "feat: TipoLancamento e Dinheiro (objeto de valor) com testes"
```

---

## Tarefa 3: Domínio — Exceções e Lancamento

**Arquivos:**
- Criar: `src/MovimentacoesFinanceiras.Dominio/Contas/Excecoes/SaldoInsuficienteException.cs`
- Criar: `src/MovimentacoesFinanceiras.Dominio/Contas/Excecoes/ContaNaoEncontradaException.cs`
- Criar: `src/MovimentacoesFinanceiras.Dominio/Contas/Lancamento.cs`

- [ ] **Passo 1: Criar SaldoInsuficienteException**

Criar `src/MovimentacoesFinanceiras.Dominio/Contas/Excecoes/SaldoInsuficienteException.cs`:

```csharp
namespace MovimentacoesFinanceiras.Dominio.Contas.Excecoes;

public sealed class SaldoInsuficienteException : Exception
{
    public SaldoInsuficienteException(decimal saldoAtual, Dinheiro valorSolicitado)
        : base($"Saldo insuficiente. Disponível: R$ {saldoAtual:N2}, solicitado: {valorSolicitado}.")
    {
        SaldoAtual = saldoAtual;
        ValorSolicitado = valorSolicitado;
    }

    public decimal SaldoAtual { get; }
    public Dinheiro ValorSolicitado { get; }
}
```

- [ ] **Passo 2: Criar ContaNaoEncontradaException**

Criar `src/MovimentacoesFinanceiras.Dominio/Contas/Excecoes/ContaNaoEncontradaException.cs`:

```csharp
namespace MovimentacoesFinanceiras.Dominio.Contas.Excecoes;

public sealed class ContaNaoEncontradaException : Exception
{
    public ContaNaoEncontradaException(Guid contaId)
        : base($"Conta {contaId} não encontrada.")
    {
        ContaId = contaId;
    }

    public Guid ContaId { get; }
}
```

- [ ] **Passo 3: Criar Lancamento**

Criar `src/MovimentacoesFinanceiras.Dominio/Contas/Lancamento.cs`:

```csharp
namespace MovimentacoesFinanceiras.Dominio.Contas;

public sealed class Lancamento
{
    private Lancamento() { }

    public Guid Id { get; private set; }
    public Guid ContaId { get; private set; }
    public decimal Valor { get; private set; }
    public TipoLancamento Tipo { get; private set; }
    public string? Descricao { get; private set; }
    public string? ChaveIdempotencia { get; private set; }
    public DateTime CriadoEm { get; private set; }

    public static Lancamento CriarCredito(Guid contaId, decimal valor, string? descricao, string? chaveIdempotencia = null) =>
        new()
        {
            Id = Guid.NewGuid(),
            ContaId = contaId,
            Valor = valor,
            Tipo = TipoLancamento.Credito,
            Descricao = descricao,
            ChaveIdempotencia = chaveIdempotencia,
            CriadoEm = DateTime.UtcNow
        };

    public static Lancamento CriarDebito(Guid contaId, decimal valor, string? descricao, string? chaveIdempotencia = null) =>
        new()
        {
            Id = Guid.NewGuid(),
            ContaId = contaId,
            Valor = valor,
            Tipo = TipoLancamento.Debito,
            Descricao = descricao,
            ChaveIdempotencia = chaveIdempotencia,
            CriadoEm = DateTime.UtcNow
        };
}
```

- [ ] **Passo 4: Compilar**

```bash
dotnet build src/MovimentacoesFinanceiras.Dominio
```
Esperado: `Build succeeded. 0 Warning(s). 0 Error(s).`

- [ ] **Passo 5: Commit**

```bash
git add src/MovimentacoesFinanceiras.Dominio/Contas/Excecoes/
git add src/MovimentacoesFinanceiras.Dominio/Contas/Lancamento.cs
git commit -m "feat: Lancamento, SaldoInsuficienteException e ContaNaoEncontradaException"
```

---

## Tarefa 4: Domínio — Conta (Agregado Raiz) e testes

**Arquivos:**
- Criar: `src/MovimentacoesFinanceiras.Dominio/Contas/Conta.cs`
- Criar: `tests/Dominio.Testes/Contas/ContaTestes.cs`

- [ ] **Passo 1: Escrever testes da Conta**

Criar `tests/Dominio.Testes/Contas/ContaTestes.cs`:

```csharp
using FluentAssertions;
using MovimentacoesFinanceiras.Dominio.Contas;
using MovimentacoesFinanceiras.Dominio.Contas.Excecoes;

namespace Dominio.Testes.Contas;

public class ContaTestes
{
    [Fact]
    public void Criar_DeveInicializarComSaldoZero()
    {
        var conta = Conta.Criar(Guid.NewGuid());
        conta.SaldoAtual.Should().Be(0m);
        conta.Lancamentos.Should().BeEmpty();
    }

    [Fact]
    public void Creditar_DeveAumentarSaldo()
    {
        var conta = Conta.Criar(Guid.NewGuid());
        conta.Creditar(Dinheiro.De(100m), "Depósito");
        conta.SaldoAtual.Should().Be(100m);
    }

    [Fact]
    public void Creditar_DeveRegistrarLancamentoDoTipoCorreto()
    {
        var conta = Conta.Criar(Guid.NewGuid());
        conta.Creditar(Dinheiro.De(100m), "Depósito");
        conta.Lancamentos.Should().HaveCount(1);
        conta.Lancamentos[0].Tipo.Should().Be(TipoLancamento.Credito);
        conta.Lancamentos[0].Valor.Should().Be(100m);
    }

    [Fact]
    public void Debitar_ComSaldoSuficiente_DeveReduzirSaldo()
    {
        var conta = Conta.Criar(Guid.NewGuid());
        conta.Creditar(Dinheiro.De(200m), "Depósito");
        conta.Debitar(Dinheiro.De(80m), "Saque");
        conta.SaldoAtual.Should().Be(120m);
    }

    [Fact]
    public void Debitar_DeveRegistrarLancamentoDoTipoCorreto()
    {
        var conta = Conta.Criar(Guid.NewGuid());
        conta.Creditar(Dinheiro.De(200m), "Depósito");
        conta.Debitar(Dinheiro.De(50m), "Saque");
        conta.Lancamentos[1].Tipo.Should().Be(TipoLancamento.Debito);
        conta.Lancamentos[1].Valor.Should().Be(50m);
    }

    [Fact]
    public void Debitar_ComSaldoInsuficiente_LancaSaldoInsuficienteException()
    {
        var conta = Conta.Criar(Guid.NewGuid());
        conta.Creditar(Dinheiro.De(50m), "Depósito");
        var acao = () => conta.Debitar(Dinheiro.De(100m), "Saque");
        acao.Should().Throw<SaldoInsuficienteException>()
            .Which.SaldoAtual.Should().Be(50m);
    }

    [Fact]
    public void Debitar_ComContaSemSaldo_LancaSaldoInsuficienteException()
    {
        var conta = Conta.Criar(Guid.NewGuid());
        var acao = () => conta.Debitar(Dinheiro.De(1m), "Saque");
        acao.Should().Throw<SaldoInsuficienteException>();
    }

    [Fact]
    public void Creditar_DeveAtualizarVersaoLinha()
    {
        var conta = Conta.Criar(Guid.NewGuid());
        var versaoAnterior = conta.VersaoLinha;
        conta.Creditar(Dinheiro.De(100m), "Depósito");
        conta.VersaoLinha.Should().NotBe(versaoAnterior);
    }

    [Fact]
    public void MultiplosLancamentos_SaldoDeveSerConsistente()
    {
        var conta = Conta.Criar(Guid.NewGuid());
        conta.Creditar(Dinheiro.De(500m), "Salário");
        conta.Debitar(Dinheiro.De(100m), "Aluguel");
        conta.Debitar(Dinheiro.De(50m), "Mercado");
        conta.Creditar(Dinheiro.De(200m), "Bônus");
        conta.SaldoAtual.Should().Be(550m);
        conta.Lancamentos.Should().HaveCount(4);
    }
}
```

- [ ] **Passo 2: Executar testes — confirmar falha**

```bash
dotnet test tests/Dominio.Testes
```
Esperado: FAIL — `Conta` não existe.

- [ ] **Passo 3: Criar Conta**

Criar `src/MovimentacoesFinanceiras.Dominio/Contas/Conta.cs`:

```csharp
using MovimentacoesFinanceiras.Dominio.Contas.Excecoes;

namespace MovimentacoesFinanceiras.Dominio.Contas;

public sealed class Conta
{
    private readonly List<Lancamento> _lancamentos = [];

    private Conta() { }

    public Guid Id { get; private set; }
    public Guid ClienteId { get; private set; }
    public decimal SaldoAtual { get; private set; }
    public Guid VersaoLinha { get; private set; }
    public DateTime CriadoEm { get; private set; }
    public IReadOnlyList<Lancamento> Lancamentos => _lancamentos.AsReadOnly();

    public static Conta Criar(Guid clienteId) =>
        new()
        {
            Id = Guid.NewGuid(),
            ClienteId = clienteId,
            SaldoAtual = 0m,
            VersaoLinha = Guid.NewGuid(),
            CriadoEm = DateTime.UtcNow
        };

    public Lancamento Creditar(Dinheiro valor, string? descricao, string? chaveIdempotencia = null)
    {
        SaldoAtual += valor.Quantia;
        VersaoLinha = Guid.NewGuid();
        var lancamento = Lancamento.CriarCredito(Id, valor.Quantia, descricao, chaveIdempotencia);
        _lancamentos.Add(lancamento);
        return lancamento;
    }

    public Lancamento Debitar(Dinheiro valor, string? descricao, string? chaveIdempotencia = null)
    {
        if (valor.Quantia > SaldoAtual)
            throw new SaldoInsuficienteException(SaldoAtual, valor);

        SaldoAtual -= valor.Quantia;
        VersaoLinha = Guid.NewGuid();
        var lancamento = Lancamento.CriarDebito(Id, valor.Quantia, descricao, chaveIdempotencia);
        _lancamentos.Add(lancamento);
        return lancamento;
    }
}
```

- [ ] **Passo 4: Executar testes — confirmar aprovação**

```bash
dotnet test tests/Dominio.Testes
```
Esperado: PASS — 13 testes passando (5 de Dinheiro + 8 de Conta).

- [ ] **Passo 5: Commit**

```bash
git add src/MovimentacoesFinanceiras.Dominio/Contas/Conta.cs
git add tests/Dominio.Testes/Contas/ContaTestes.cs
git commit -m "feat: Conta (agregado raiz) com Creditar, Debitar e VersaoLinha — testes passando"
```

---

## Tarefa 5: Domínio — IContaRepositorio

**Arquivos:**
- Criar: `src/MovimentacoesFinanceiras.Dominio/Contas/IContaRepositorio.cs`

- [ ] **Passo 1: Criar a interface**

Criar `src/MovimentacoesFinanceiras.Dominio/Contas/IContaRepositorio.cs`:

```csharp
namespace MovimentacoesFinanceiras.Dominio.Contas;

public interface IContaRepositorio
{
    Task<Conta?> ObterPorIdAsync(Guid id, CancellationToken cancellationToken = default);
    Task<Lancamento?> ObterLancamentoPorChaveIdempotenciaAsync(string chave, CancellationToken cancellationToken = default);
    Task AdicionarAsync(Conta conta, CancellationToken cancellationToken = default);
    Task SalvarAsync(CancellationToken cancellationToken = default);
    Task<decimal> ConsultarSaldoEmAsync(Guid contaId, DateTime dataReferencia, CancellationToken cancellationToken = default);
    Task<(IReadOnlyList<Lancamento> Itens, int Total)> ListarLancamentosAsync(
        Guid contaId, int pagina, int tamanhoPagina, CancellationToken cancellationToken = default);
}
```

- [ ] **Passo 2: Compilar**

```bash
dotnet build src/MovimentacoesFinanceiras.Dominio
```
Esperado: `Build succeeded. 0 Warning(s). 0 Error(s).`

- [ ] **Passo 3: Commit**

```bash
git add src/MovimentacoesFinanceiras.Dominio/Contas/IContaRepositorio.cs
git commit -m "feat: IContaRepositorio — porta de saída do domínio"
```

---

## Tarefa 6: Infraestrutura — EF Core, DbContext e Configurações

**Arquivos:**
- Criar: `src/MovimentacoesFinanceiras.Infraestrutura/Persistencia/ContextoBancoDados.cs`
- Criar: `src/MovimentacoesFinanceiras.Infraestrutura/Persistencia/Configuracoes/ContaConfiguracao.cs`
- Criar: `src/MovimentacoesFinanceiras.Infraestrutura/Persistencia/Configuracoes/LancamentoConfiguracao.cs`

- [ ] **Passo 1: Criar ContextoBancoDados**

Criar `src/MovimentacoesFinanceiras.Infraestrutura/Persistencia/ContextoBancoDados.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using MovimentacoesFinanceiras.Dominio.Contas;

namespace MovimentacoesFinanceiras.Infraestrutura.Persistencia;

public class ContextoBancoDados : DbContext
{
    public ContextoBancoDados(DbContextOptions<ContextoBancoDados> options) : base(options) { }

    public DbSet<Conta> Contas => Set<Conta>();
    public DbSet<Lancamento> Lancamentos => Set<Lancamento>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ContextoBancoDados).Assembly);
    }
}
```

- [ ] **Passo 2: Criar ContaConfiguracao**

Criar `src/MovimentacoesFinanceiras.Infraestrutura/Persistencia/Configuracoes/ContaConfiguracao.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using MovimentacoesFinanceiras.Dominio.Contas;

namespace MovimentacoesFinanceiras.Infraestrutura.Persistencia.Configuracoes;

public class ContaConfiguracao : IEntityTypeConfiguration<Conta>
{
    public void Configure(EntityTypeBuilder<Conta> builder)
    {
        builder.ToTable("contas");
        builder.HasKey(c => c.Id);

        builder.Property(c => c.Id).HasColumnName("id");
        builder.Property(c => c.ClienteId).HasColumnName("cliente_id").IsRequired();
        builder.Property(c => c.SaldoAtual).HasColumnName("saldo_atual").HasPrecision(18, 2).IsRequired();
        builder.Property(c => c.VersaoLinha).HasColumnName("versao_linha").IsConcurrencyToken().IsRequired();
        builder.Property(c => c.CriadoEm).HasColumnName("criado_em").IsRequired();

        builder.HasIndex(c => c.ClienteId);

        builder.HasMany(c => c.Lancamentos)
               .WithOne()
               .HasForeignKey(l => l.ContaId)
               .OnDelete(DeleteBehavior.Cascade);

        builder.Navigation(c => c.Lancamentos).HasField("_lancamentos");
    }
}
```

- [ ] **Passo 3: Criar LancamentoConfiguracao**

Criar `src/MovimentacoesFinanceiras.Infraestrutura/Persistencia/Configuracoes/LancamentoConfiguracao.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using MovimentacoesFinanceiras.Dominio.Contas;

namespace MovimentacoesFinanceiras.Infraestrutura.Persistencia.Configuracoes;

public class LancamentoConfiguracao : IEntityTypeConfiguration<Lancamento>
{
    public void Configure(EntityTypeBuilder<Lancamento> builder)
    {
        builder.ToTable("lancamentos");
        builder.HasKey(l => l.Id);

        builder.Property(l => l.Id).HasColumnName("id");
        builder.Property(l => l.ContaId).HasColumnName("conta_id").IsRequired();
        builder.Property(l => l.Valor).HasColumnName("valor").HasPrecision(18, 2).IsRequired();
        builder.Property(l => l.Tipo).HasColumnName("tipo").HasConversion<string>().IsRequired();
        builder.Property(l => l.Descricao).HasColumnName("descricao").HasMaxLength(255);
        builder.Property(l => l.ChaveIdempotencia).HasColumnName("chave_idempotencia").HasMaxLength(64);
        builder.Property(l => l.CriadoEm).HasColumnName("criado_em").IsRequired();

        builder.HasIndex(l => new { l.ContaId, l.CriadoEm });
        builder.HasIndex(l => l.ChaveIdempotencia).IsUnique().HasFilter("[chave_idempotencia] IS NOT NULL");
    }
}
```

- [ ] **Passo 4: Compilar infraestrutura**

```bash
dotnet build src/MovimentacoesFinanceiras.Infraestrutura
```
Esperado: `Build succeeded. 0 Warning(s). 0 Error(s).`

- [ ] **Passo 5: Commit**

```bash
git add src/MovimentacoesFinanceiras.Infraestrutura/
git commit -m "feat: ContextoBancoDados + configuracoes EF Core (contas e lancamentos)"
```

---

## Tarefa 7: Infraestrutura — ContaRepositorio

**Arquivos:**
- Criar: `src/MovimentacoesFinanceiras.Infraestrutura/Persistencia/Repositorios/ContaRepositorio.cs`

- [ ] **Passo 1: Criar ContaRepositorio**

Criar `src/MovimentacoesFinanceiras.Infraestrutura/Persistencia/Repositorios/ContaRepositorio.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using MovimentacoesFinanceiras.Dominio.Contas;

namespace MovimentacoesFinanceiras.Infraestrutura.Persistencia.Repositorios;

public class ContaRepositorio : IContaRepositorio
{
    private readonly ContextoBancoDados _contexto;

    public ContaRepositorio(ContextoBancoDados contexto) => _contexto = contexto;

    public async Task<Conta?> ObterPorIdAsync(Guid id, CancellationToken cancellationToken = default)
        => await _contexto.Contas.FirstOrDefaultAsync(c => c.Id == id, cancellationToken);

    public async Task<Lancamento?> ObterLancamentoPorChaveIdempotenciaAsync(string chave, CancellationToken cancellationToken = default)
        => await _contexto.Lancamentos.FirstOrDefaultAsync(l => l.ChaveIdempotencia == chave, cancellationToken);

    public async Task AdicionarAsync(Conta conta, CancellationToken cancellationToken = default)
        => await _contexto.Contas.AddAsync(conta, cancellationToken);

    public async Task SalvarAsync(CancellationToken cancellationToken = default)
        => await _contexto.SaveChangesAsync(cancellationToken);

    public async Task<decimal> ConsultarSaldoEmAsync(Guid contaId, DateTime dataReferencia, CancellationToken cancellationToken = default)
    {
        var lancamentos = await _contexto.Lancamentos
            .Where(l => l.ContaId == contaId && l.CriadoEm <= dataReferencia)
            .ToListAsync(cancellationToken);

        return lancamentos.Sum(l => l.Tipo == TipoLancamento.Credito ? l.Valor : -l.Valor);
    }

    public async Task<(IReadOnlyList<Lancamento> Itens, int Total)> ListarLancamentosAsync(
        Guid contaId, int pagina, int tamanhoPagina, CancellationToken cancellationToken = default)
    {
        var query = _contexto.Lancamentos
            .Where(l => l.ContaId == contaId)
            .OrderByDescending(l => l.CriadoEm);

        var total = await query.CountAsync(cancellationToken);
        var itens = await query
            .Skip((pagina - 1) * tamanhoPagina)
            .Take(tamanhoPagina)
            .ToListAsync(cancellationToken);

        return (itens, total);
    }
}
```

- [ ] **Passo 2: Compilar**

```bash
dotnet build src/MovimentacoesFinanceiras.Infraestrutura
```
Esperado: `Build succeeded. 0 Warning(s). 0 Error(s).`

- [ ] **Passo 3: Commit**

```bash
git add src/MovimentacoesFinanceiras.Infraestrutura/Persistencia/Repositorios/ContaRepositorio.cs
git commit -m "feat: ContaRepositorio — implementa IContaRepositorio com EF Core + SQLite"
```

---

## Tarefa 8: Aplicação — RegistrarMovimentacao

**Arquivos:**
- Criar: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Comportamentos/ValidacaoBehavior.cs`
- Criar: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Comandos/RegistrarMovimentacao/LancamentoResposta.cs`
- Criar: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Comandos/RegistrarMovimentacao/RegistrarMovimentacaoComando.cs`
- Criar: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Comandos/RegistrarMovimentacao/RegistrarMovimentacaoValidador.cs`
- Criar: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Comandos/RegistrarMovimentacao/RegistrarMovimentacaoManipulador.cs`

- [ ] **Passo 1: Criar ValidacaoBehavior**

Criar `src/MovimentacoesFinanceiras.Aplicacao/Contas/Comportamentos/ValidacaoBehavior.cs`:

```csharp
using FluentValidation;
using MediatR;

namespace MovimentacoesFinanceiras.Aplicacao.Contas.Comportamentos;

public class ValidacaoBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validadores)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken cancellationToken)
    {
        if (!validadores.Any())
            return await next();

        var contexto = new ValidationContext<TRequest>(request);
        var erros = validadores
            .Select(v => v.Validate(contexto))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (erros.Count > 0)
            throw new ValidationException(erros);

        return await next();
    }
}
```

- [ ] **Passo 2: Criar LancamentoResposta**

Criar `src/MovimentacoesFinanceiras.Aplicacao/Contas/Comandos/RegistrarMovimentacao/LancamentoResposta.cs`:

```csharp
using MovimentacoesFinanceiras.Dominio.Contas;

namespace MovimentacoesFinanceiras.Aplicacao.Contas.Comandos.RegistrarMovimentacao;

public record LancamentoResposta(
    Guid Id,
    TipoLancamento Tipo,
    decimal Valor,
    string? Descricao,
    DateTime CriadoEm
);
```

- [ ] **Passo 3: Criar RegistrarMovimentacaoComando**

Criar `src/MovimentacoesFinanceiras.Aplicacao/Contas/Comandos/RegistrarMovimentacao/RegistrarMovimentacaoComando.cs`:

```csharp
using MediatR;
using MovimentacoesFinanceiras.Dominio.Contas;

namespace MovimentacoesFinanceiras.Aplicacao.Contas.Comandos.RegistrarMovimentacao;

public record RegistrarMovimentacaoComando(
    Guid ContaId,
    TipoLancamento Tipo,
    decimal Valor,
    string? Descricao,
    string? ChaveIdempotencia
) : IRequest<LancamentoResposta>;
```

- [ ] **Passo 4: Criar RegistrarMovimentacaoValidador**

Criar `src/MovimentacoesFinanceiras.Aplicacao/Contas/Comandos/RegistrarMovimentacao/RegistrarMovimentacaoValidador.cs`:

```csharp
using FluentValidation;
using MovimentacoesFinanceiras.Dominio.Contas;

namespace MovimentacoesFinanceiras.Aplicacao.Contas.Comandos.RegistrarMovimentacao;

public class RegistrarMovimentacaoValidador : AbstractValidator<RegistrarMovimentacaoComando>
{
    public RegistrarMovimentacaoValidador()
    {
        RuleFor(x => x.ContaId)
            .NotEmpty().WithMessage("O identificador da conta é obrigatório.");

        RuleFor(x => x.Valor)
            .GreaterThan(0).WithMessage("O valor deve ser maior que zero.");

        RuleFor(x => x.Tipo)
            .IsInEnum().WithMessage("O tipo deve ser Credito ou Debito.");

        RuleFor(x => x.Descricao)
            .MaximumLength(255).WithMessage("A descrição deve ter no máximo 255 caracteres.")
            .When(x => x.Descricao is not null);

        RuleFor(x => x.ChaveIdempotencia)
            .MaximumLength(64).WithMessage("A chave de idempotência deve ter no máximo 64 caracteres.")
            .When(x => x.ChaveIdempotencia is not null);
    }
}
```

- [ ] **Passo 5: Criar RegistrarMovimentacaoManipulador com Polly**

Criar `src/MovimentacoesFinanceiras.Aplicacao/Contas/Comandos/RegistrarMovimentacao/RegistrarMovimentacaoManipulador.cs`:

```csharp
using MediatR;
using Microsoft.EntityFrameworkCore;
using MovimentacoesFinanceiras.Dominio.Contas;
using MovimentacoesFinanceiras.Dominio.Contas.Excecoes;
using Polly;
using Polly.Retry;

namespace MovimentacoesFinanceiras.Aplicacao.Contas.Comandos.RegistrarMovimentacao;

public class RegistrarMovimentacaoManipulador(IContaRepositorio repositorio)
    : IRequestHandler<RegistrarMovimentacaoComando, LancamentoResposta>
{
    private static readonly AsyncRetryPolicy PoliticaRetentativa = Policy
        .Handle<DbUpdateConcurrencyException>()
        .WaitAndRetryAsync(3, tentativa => TimeSpan.FromMilliseconds(50 * Math.Pow(2, tentativa)));

    public async Task<LancamentoResposta> Handle(RegistrarMovimentacaoComando request, CancellationToken cancellationToken)
    {
        if (request.ChaveIdempotencia is not null)
        {
            var lancamentoExistente = await repositorio.ObterLancamentoPorChaveIdempotenciaAsync(
                request.ChaveIdempotencia, cancellationToken);

            if (lancamentoExistente is not null)
                return ToResposta(lancamentoExistente);
        }

        return await PoliticaRetentativa.ExecuteAsync(async () =>
        {
            var conta = await repositorio.ObterPorIdAsync(request.ContaId, cancellationToken)
                ?? throw new ContaNaoEncontradaException(request.ContaId);

            var dinheiro = Dinheiro.De(request.Valor);

            var lancamento = request.Tipo == TipoLancamento.Credito
                ? conta.Creditar(dinheiro, request.Descricao, request.ChaveIdempotencia)
                : conta.Debitar(dinheiro, request.Descricao, request.ChaveIdempotencia);

            await repositorio.SalvarAsync(cancellationToken);
            return ToResposta(lancamento);
        });
    }

    private static LancamentoResposta ToResposta(Lancamento lancamento) =>
        new(lancamento.Id, lancamento.Tipo, lancamento.Valor, lancamento.Descricao, lancamento.CriadoEm);
}
```

- [ ] **Passo 6: Compilar aplicação**

```bash
dotnet build src/MovimentacoesFinanceiras.Aplicacao
```
Esperado: `Build succeeded. 0 Warning(s). 0 Error(s).`

- [ ] **Passo 7: Commit**

```bash
git add src/MovimentacoesFinanceiras.Aplicacao/
git commit -m "feat: RegistrarMovimentacao — comando, validador e manipulador com Polly retry"
```

---

## Tarefa 9: Aplicação — ConsultarSaldo e ConsultarSaldoEm

**Arquivos:**
- Criar: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Consultas/ConsultarSaldo/SaldoResposta.cs`
- Criar: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Consultas/ConsultarSaldo/ConsultarSaldoConsulta.cs`
- Criar: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Consultas/ConsultarSaldo/ConsultarSaldoManipulador.cs`
- Criar: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Consultas/ConsultarSaldoEm/SaldoHistoricoResposta.cs`
- Criar: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Consultas/ConsultarSaldoEm/ConsultarSaldoEmConsulta.cs`
- Criar: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Consultas/ConsultarSaldoEm/ConsultarSaldoEmManipulador.cs`

- [ ] **Passo 1: Criar SaldoResposta e ConsultarSaldoConsulta**

Criar `src/MovimentacoesFinanceiras.Aplicacao/Contas/Consultas/ConsultarSaldo/SaldoResposta.cs`:

```csharp
namespace MovimentacoesFinanceiras.Aplicacao.Contas.Consultas.ConsultarSaldo;

public record SaldoResposta(decimal Saldo, DateTime ConsultadoEm);
```

Criar `src/MovimentacoesFinanceiras.Aplicacao/Contas/Consultas/ConsultarSaldo/ConsultarSaldoConsulta.cs`:

```csharp
using MediatR;

namespace MovimentacoesFinanceiras.Aplicacao.Contas.Consultas.ConsultarSaldo;

public record ConsultarSaldoConsulta(Guid ContaId) : IRequest<SaldoResposta>;
```

- [ ] **Passo 2: Criar ConsultarSaldoManipulador**

Criar `src/MovimentacoesFinanceiras.Aplicacao/Contas/Consultas/ConsultarSaldo/ConsultarSaldoManipulador.cs`:

```csharp
using MediatR;
using MovimentacoesFinanceiras.Dominio.Contas;
using MovimentacoesFinanceiras.Dominio.Contas.Excecoes;

namespace MovimentacoesFinanceiras.Aplicacao.Contas.Consultas.ConsultarSaldo;

public class ConsultarSaldoManipulador(IContaRepositorio repositorio)
    : IRequestHandler<ConsultarSaldoConsulta, SaldoResposta>
{
    public async Task<SaldoResposta> Handle(ConsultarSaldoConsulta request, CancellationToken cancellationToken)
    {
        var conta = await repositorio.ObterPorIdAsync(request.ContaId, cancellationToken)
            ?? throw new ContaNaoEncontradaException(request.ContaId);

        return new SaldoResposta(conta.SaldoAtual, DateTime.UtcNow);
    }
}
```

- [ ] **Passo 3: Criar SaldoHistoricoResposta e ConsultarSaldoEmConsulta**

Criar `src/MovimentacoesFinanceiras.Aplicacao/Contas/Consultas/ConsultarSaldoEm/SaldoHistoricoResposta.cs`:

```csharp
namespace MovimentacoesFinanceiras.Aplicacao.Contas.Consultas.ConsultarSaldoEm;

public record SaldoHistoricoResposta(decimal Saldo, DateTime ReferenciaEm);
```

Criar `src/MovimentacoesFinanceiras.Aplicacao/Contas/Consultas/ConsultarSaldoEm/ConsultarSaldoEmConsulta.cs`:

```csharp
using MediatR;

namespace MovimentacoesFinanceiras.Aplicacao.Contas.Consultas.ConsultarSaldoEm;

public record ConsultarSaldoEmConsulta(Guid ContaId, DateTime DataReferencia) : IRequest<SaldoHistoricoResposta>;
```

- [ ] **Passo 4: Criar ConsultarSaldoEmManipulador**

Criar `src/MovimentacoesFinanceiras.Aplicacao/Contas/Consultas/ConsultarSaldoEm/ConsultarSaldoEmManipulador.cs`:

```csharp
using MediatR;
using MovimentacoesFinanceiras.Dominio.Contas;
using MovimentacoesFinanceiras.Dominio.Contas.Excecoes;

namespace MovimentacoesFinanceiras.Aplicacao.Contas.Consultas.ConsultarSaldoEm;

public class ConsultarSaldoEmManipulador(IContaRepositorio repositorio)
    : IRequestHandler<ConsultarSaldoEmConsulta, SaldoHistoricoResposta>
{
    public async Task<SaldoHistoricoResposta> Handle(ConsultarSaldoEmConsulta request, CancellationToken cancellationToken)
    {
        var conta = await repositorio.ObterPorIdAsync(request.ContaId, cancellationToken)
            ?? throw new ContaNaoEncontradaException(request.ContaId);

        var saldo = await repositorio.ConsultarSaldoEmAsync(request.ContaId, request.DataReferencia, cancellationToken);

        return new SaldoHistoricoResposta(saldo, request.DataReferencia);
    }
}
```

- [ ] **Passo 5: Compilar**

```bash
dotnet build src/MovimentacoesFinanceiras.Aplicacao
```
Esperado: `Build succeeded. 0 Warning(s). 0 Error(s).`

- [ ] **Passo 6: Commit**

```bash
git add src/MovimentacoesFinanceiras.Aplicacao/Contas/Consultas/
git commit -m "feat: ConsultarSaldo e ConsultarSaldoEm — consultas CQRS"
```

---

## Tarefa 10: API — Program.cs, DI, Serilog, Swagger e HealthCheck

**Arquivos:**
- Modificar: `src/MovimentacoesFinanceiras.Api/Program.cs`
- Criar: `src/MovimentacoesFinanceiras.Api/appsettings.json`

- [ ] **Passo 1: Criar Program.cs**

Substituir o conteúdo de `src/MovimentacoesFinanceiras.Api/Program.cs`:

```csharp
using FluentValidation;
using MediatR;
using Microsoft.EntityFrameworkCore;
using MovimentacoesFinanceiras.Aplicacao.Contas.Comportamentos;
using MovimentacoesFinanceiras.Aplicacao.Contas.Comandos.RegistrarMovimentacao;
using MovimentacoesFinanceiras.Dominio.Contas;
using MovimentacoesFinanceiras.Infraestrutura.Persistencia;
using MovimentacoesFinanceiras.Infraestrutura.Persistencia.Repositorios;
using Serilog;

var builder = WebApplication.CreateBuilder(args);

builder.Host.UseSerilog((ctx, config) =>
    config
        .ReadFrom.Configuration(ctx.Configuration)
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .WriteTo.Console(new Serilog.Formatting.Json.JsonFormatter()));

builder.Services.AddDbContext<ContextoBancoDados>(opcoes =>
    opcoes.UseSqlite(builder.Configuration.GetConnectionString("Padrao")
        ?? "Data Source=movimentacoes.db"));

builder.Services.AddScoped<IContaRepositorio, ContaRepositorio>();

builder.Services.AddMediatR(cfg =>
    cfg.RegisterServicesFromAssembly(typeof(RegistrarMovimentacaoComando).Assembly));

builder.Services.AddValidatorsFromAssembly(typeof(RegistrarMovimentacaoComando).Assembly);
builder.Services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidacaoBehavior<,>));

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(opcoes =>
{
    opcoes.SwaggerDoc("v1", new()
    {
        Title = "API de Movimentações Financeiras",
        Description = "Registra movimentações financeiras e consulta saldos de contas bancárias. " +
                      "Utiliza CQRS com ledger append-only para rastreabilidade completa.",
        Version = "v1"
    });
});

builder.Services.AddHealthChecks()
    .AddDbContextCheck<ContextoBancoDados>("banco-de-dados");

var app = builder.Build();

using (var escopo = app.Services.CreateScope())
{
    var contexto = escopo.ServiceProvider.GetRequiredService<ContextoBancoDados>();
    await contexto.Database.EnsureCreatedAsync();
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

- [ ] **Passo 2: Criar appsettings.json**

Substituir `src/MovimentacoesFinanceiras.Api/appsettings.json`:

```json
{
  "ConnectionStrings": {
    "Padrao": "Data Source=movimentacoes.db"
  },
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.EntityFrameworkCore": "Warning"
      }
    }
  },
  "AllowedHosts": "*"
}
```

- [ ] **Passo 3: Compilar**

```bash
dotnet build src/MovimentacoesFinanceiras.Api
```
Esperado: `Build succeeded. 0 Warning(s). 0 Error(s).`

- [ ] **Passo 4: Commit**

```bash
git add src/MovimentacoesFinanceiras.Api/Program.cs
git add src/MovimentacoesFinanceiras.Api/appsettings.json
git commit -m "feat: Program.cs com DI, Serilog, Swagger em portugues e HealthCheck"
```

---

## Tarefa 11: API — Middlewares

**Arquivos:**
- Criar: `src/MovimentacoesFinanceiras.Api/Middlewares/CorrelacaoIdMiddleware.cs`
- Criar: `src/MovimentacoesFinanceiras.Api/Middlewares/TratadorDeExcecoesMiddleware.cs`

- [ ] **Passo 1: Criar CorrelacaoIdMiddleware**

Criar `src/MovimentacoesFinanceiras.Api/Middlewares/CorrelacaoIdMiddleware.cs`:

```csharp
using Serilog.Context;

namespace MovimentacoesFinanceiras.Api.Middlewares;

public class CorrelacaoIdMiddleware(RequestDelegate proximo)
{
    private const string CabecalhoCorrelacaoId = "X-Correlation-Id";

    public async Task InvokeAsync(HttpContext contexto)
    {
        var correlacaoId = contexto.Request.Headers[CabecalhoCorrelacaoId].FirstOrDefault()
            ?? Guid.NewGuid().ToString();

        contexto.Response.Headers[CabecalhoCorrelacaoId] = correlacaoId;

        using (LogContext.PushProperty("correlacao_id", correlacaoId))
        {
            await proximo(contexto);
        }
    }
}
```

- [ ] **Passo 2: Criar TratadorDeExcecoesMiddleware**

Criar `src/MovimentacoesFinanceiras.Api/Middlewares/TratadorDeExcecoesMiddleware.cs`:

```csharp
using System.Text.Json;
using FluentValidation;
using MovimentacoesFinanceiras.Dominio.Contas.Excecoes;

namespace MovimentacoesFinanceiras.Api.Middlewares;

public class TratadorDeExcecoesMiddleware(RequestDelegate proximo, ILogger<TratadorDeExcecoesMiddleware> logger)
{
    private static readonly JsonSerializerOptions OpcoesJson = new() { PropertyNamingPolicy = JsonNamingPolicy.CamelCase };

    public async Task InvokeAsync(HttpContext contexto)
    {
        try
        {
            await proximo(contexto);
        }
        catch (ValidationException ex)
        {
            contexto.Response.StatusCode = StatusCodes.Status400BadRequest;
            contexto.Response.ContentType = "application/problem+json";
            var problema = new
            {
                tipo = "requisicao-invalida",
                titulo = "A requisição contém dados inválidos",
                status = 400,
                erros = ex.Errors.Select(e => e.ErrorMessage)
            };
            await contexto.Response.WriteAsync(JsonSerializer.Serialize(problema, OpcoesJson));
        }
        catch (SaldoInsuficienteException ex)
        {
            contexto.Response.StatusCode = StatusCodes.Status422UnprocessableEntity;
            contexto.Response.ContentType = "application/problem+json";
            var problema = new
            {
                tipo = "saldo-insuficiente",
                titulo = "Saldo insuficiente para realizar o débito",
                status = 422,
                detalhe = ex.Message
            };
            await contexto.Response.WriteAsync(JsonSerializer.Serialize(problema, OpcoesJson));
        }
        catch (ContaNaoEncontradaException ex)
        {
            contexto.Response.StatusCode = StatusCodes.Status404NotFound;
            contexto.Response.ContentType = "application/problem+json";
            var problema = new
            {
                tipo = "conta-nao-encontrada",
                titulo = "Conta não encontrada",
                status = 404,
                detalhe = ex.Message
            };
            await contexto.Response.WriteAsync(JsonSerializer.Serialize(problema, OpcoesJson));
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Erro nao tratado");
            contexto.Response.StatusCode = StatusCodes.Status500InternalServerError;
            contexto.Response.ContentType = "application/problem+json";
            var problema = new
            {
                tipo = "erro-interno",
                titulo = "Ocorreu um erro interno. Tente novamente.",
                status = 500
            };
            await contexto.Response.WriteAsync(JsonSerializer.Serialize(problema, OpcoesJson));
        }
    }
}
```

- [ ] **Passo 3: Compilar**

```bash
dotnet build src/MovimentacoesFinanceiras.Api
```
Esperado: `Build succeeded. 0 Warning(s). 0 Error(s).`

- [ ] **Passo 4: Commit**

```bash
git add src/MovimentacoesFinanceiras.Api/Middlewares/
git commit -m "feat: CorrelacaoIdMiddleware e TratadorDeExcecoesMiddleware com Problem Details em portugues"
```

---

## Tarefa 12: API — ContasControlador

**Arquivos:**
- Criar: `src/MovimentacoesFinanceiras.Api/Controladores/ContasControlador.cs`

- [ ] **Passo 1: Criar ContasControlador**

Criar `src/MovimentacoesFinanceiras.Api/Controladores/ContasControlador.cs`:

```csharp
using MediatR;
using Microsoft.AspNetCore.Mvc;
using MovimentacoesFinanceiras.Aplicacao.Contas.Comandos.RegistrarMovimentacao;
using MovimentacoesFinanceiras.Aplicacao.Contas.Consultas.ConsultarSaldo;
using MovimentacoesFinanceiras.Aplicacao.Contas.Consultas.ConsultarSaldoEm;
using MovimentacoesFinanceiras.Dominio.Contas;
using MovimentacoesFinanceiras.Infraestrutura.Persistencia;

namespace MovimentacoesFinanceiras.Api.Controladores;

[ApiController]
[Route("contas")]
[Produces("application/json")]
public class ContasControlador(IMediator mediador, ContextoBancoDados contexto, IContaRepositorio repositorio) : ControllerBase
{
    /// <summary>Cria uma nova conta para um cliente.</summary>
    [HttpPost]
    [ProducesResponseType(typeof(object), StatusCodes.Status201Created)]
    public async Task<IActionResult> CriarConta(
        [FromBody] CriarContaRequisicao requisicao,
        CancellationToken cancellationToken)
    {
        var conta = Conta.Criar(requisicao.ClienteId);
        await contexto.Contas.AddAsync(conta, cancellationToken);
        await contexto.SaveChangesAsync(cancellationToken);

        var resposta = new
        {
            id = conta.Id,
            clienteId = conta.ClienteId,
            saldoAtual = conta.SaldoAtual,
            criadoEm = conta.CriadoEm
        };

        return CreatedAtAction(nameof(ConsultarSaldo), new { id = conta.Id }, resposta);
    }

    /// <summary>Registra uma movimentação financeira (crédito ou débito).</summary>
    [HttpPost("{id:guid}/movimentacoes")]
    [ProducesResponseType(typeof(LancamentoResposta), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    [ProducesResponseType(StatusCodes.Status422UnprocessableEntity)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> RegistrarMovimentacao(
        Guid id,
        [FromBody] RegistrarMovimentacaoRequisicao requisicao,
        [FromHeader(Name = "Idempotency-Key")] string? chaveIdempotencia,
        CancellationToken cancellationToken)
    {
        var comando = new RegistrarMovimentacaoComando(id, requisicao.Tipo, requisicao.Valor, requisicao.Descricao, chaveIdempotencia);
        var resposta = await mediador.Send(comando, cancellationToken);
        return StatusCode(StatusCodes.Status201Created, resposta);
    }

    /// <summary>
    /// Consulta o saldo da conta.
    /// Sem o parâmetro 'em': retorna o saldo atual (O(1)).
    /// Com o parâmetro 'em': retorna o saldo no ponto no tempo informado.
    /// </summary>
    [HttpGet("{id:guid}/saldo")]
    [ProducesResponseType(typeof(SaldoResposta), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(SaldoHistoricoResposta), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> ConsultarSaldo(
        Guid id,
        [FromQuery] DateTime? em,
        CancellationToken cancellationToken)
    {
        if (em.HasValue)
        {
            var consulta = new ConsultarSaldoEmConsulta(id, em.Value.ToUniversalTime());
            var resposta = await mediador.Send(consulta, cancellationToken);
            return Ok(resposta);
        }
        else
        {
            var consulta = new ConsultarSaldoConsulta(id);
            var resposta = await mediador.Send(consulta, cancellationToken);
            return Ok(resposta);
        }
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
        var conta = await contexto.Contas.FindAsync([id], cancellationToken);
        if (conta is null) return NotFound();

        var (itens, total) = await repositorio.ListarLancamentosAsync(id, pagina, tamanhoPagina, cancellationToken);

        return Ok(new
        {
            itens = itens.Select(l => new
            {
                id = l.Id,
                tipo = l.Tipo.ToString(),
                valor = l.Valor,
                descricao = l.Descricao,
                criadoEm = l.CriadoEm
            }),
            pagina,
            tamanhoPagina,
            total
        });
    }
}

public record CriarContaRequisicao(Guid ClienteId);
public record RegistrarMovimentacaoRequisicao(TipoLancamento Tipo, decimal Valor, string? Descricao);
```

- [ ] **Passo 2: Compilar e executar**

```bash
dotnet build
dotnet run --project src/MovimentacoesFinanceiras.Api
```
Esperado: Aplicação sobe em `http://localhost:5000`. Swagger acessível em `http://localhost:5000`.

- [ ] **Passo 3: Smoke test manual via Swagger**

Abrir `http://localhost:5000` no browser e:
1. `POST /contas` com `{ "clienteId": "3fa85f64-5717-4562-b3fc-2c963f66afa6" }` → 201
2. `POST /contas/{id}/movimentacoes` com `{ "tipo": "Credito", "valor": 100.00 }` → 201
3. `GET /contas/{id}/saldo` → 200 com saldo 100.00
4. `GET /saude` → 200 com status do banco

- [ ] **Passo 4: Commit**

```bash
git add src/MovimentacoesFinanceiras.Api/Controladores/ContasControlador.cs
git commit -m "feat: ContasControlador com todos os endpoints — POST /contas, POST /movimentacoes, GET /saldo, GET /movimentacoes"
```

---

## Tarefa 13: Testes de Integração

**Arquivos:**
- Criar: `tests/Api.Testes/FabricaDeAplicacao.cs`
- Criar: `tests/Api.Testes/Contas/ContasIntegracaoTestes.cs`

- [ ] **Passo 1: Criar FabricaDeAplicacao**

Criar `tests/Api.Testes/FabricaDeAplicacao.cs`:

```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using MovimentacoesFinanceiras.Infraestrutura.Persistencia;

namespace Api.Testes;

public class FabricaDeAplicacao : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(servicos =>
        {
            var descritores = servicos
                .Where(d => d.ServiceType == typeof(DbContextOptions<ContextoBancoDados>) ||
                            d.ServiceType == typeof(ContextoBancoDados))
                .ToList();

            foreach (var descritor in descritores)
                servicos.Remove(descritor);

            servicos.AddDbContext<ContextoBancoDados>(opcoes =>
                opcoes.UseInMemoryDatabase($"TestDb-{Guid.NewGuid()}"));
        });
    }
}
```

- [ ] **Passo 2: Criar testes de integração**

Criar `tests/Api.Testes/Contas/ContasIntegracaoTestes.cs`:

```csharp
using System.Net;
using System.Net.Http.Json;
using System.Text.Json;
using FluentAssertions;
using MovimentacoesFinanceiras.Dominio.Contas;

namespace Api.Testes.Contas;

public class ContasIntegracaoTestes(FabricaDeAplicacao fabrica) : IClassFixture<FabricaDeAplicacao>
{
    private readonly HttpClient _cliente = fabrica.CreateClient();
    private static readonly JsonSerializerOptions OpcoesJson = new() { PropertyNameCaseInsensitive = true };

    private async Task<Guid> CriarContaAsync()
    {
        var resposta = await _cliente.PostAsJsonAsync("/contas", new { clienteId = Guid.NewGuid() });
        resposta.StatusCode.Should().Be(HttpStatusCode.Created);
        var json = await resposta.Content.ReadFromJsonAsync<JsonElement>();
        return json.GetProperty("id").GetGuid();
    }

    [Fact]
    public async Task PostContas_DeveRetornar201ComIdGerado()
    {
        var resposta = await _cliente.PostAsJsonAsync("/contas", new { clienteId = Guid.NewGuid() });
        resposta.StatusCode.Should().Be(HttpStatusCode.Created);
        var corpo = await resposta.Content.ReadFromJsonAsync<JsonElement>();
        corpo.GetProperty("id").GetGuid().Should().NotBeEmpty();
        corpo.GetProperty("saldoAtual").GetDecimal().Should().Be(0m);
    }

    [Fact]
    public async Task PostMovimentacoes_Credito_DeveRetornar201()
    {
        var contaId = await CriarContaAsync();
        var resposta = await _cliente.PostAsJsonAsync(
            $"/contas/{contaId}/movimentacoes",
            new { tipo = "Credito", valor = 100.00m, descricao = "Depósito" });
        resposta.StatusCode.Should().Be(HttpStatusCode.Created);
        var corpo = await resposta.Content.ReadFromJsonAsync<JsonElement>();
        corpo.GetProperty("valor").GetDecimal().Should().Be(100m);
    }

    [Fact]
    public async Task PostMovimentacoes_DebitoComSaldoSuficiente_DeveRetornar201()
    {
        var contaId = await CriarContaAsync();
        await _cliente.PostAsJsonAsync($"/contas/{contaId}/movimentacoes",
            new { tipo = "Credito", valor = 200m });
        var resposta = await _cliente.PostAsJsonAsync($"/contas/{contaId}/movimentacoes",
            new { tipo = "Debito", valor = 80m });
        resposta.StatusCode.Should().Be(HttpStatusCode.Created);
    }

    [Fact]
    public async Task PostMovimentacoes_DebitoSemSaldo_DeveRetornar422()
    {
        var contaId = await CriarContaAsync();
        var resposta = await _cliente.PostAsJsonAsync($"/contas/{contaId}/movimentacoes",
            new { tipo = "Debito", valor = 100m });
        resposta.StatusCode.Should().Be(HttpStatusCode.UnprocessableEntity);
        var corpo = await resposta.Content.ReadFromJsonAsync<JsonElement>();
        corpo.GetProperty("tipo").GetString().Should().Be("saldo-insuficiente");
    }

    [Fact]
    public async Task GetSaldo_DeveRetornarSaldoAtualCorreto()
    {
        var contaId = await CriarContaAsync();
        await _cliente.PostAsJsonAsync($"/contas/{contaId}/movimentacoes", new { tipo = "Credito", valor = 300m });
        await _cliente.PostAsJsonAsync($"/contas/{contaId}/movimentacoes", new { tipo = "Debito", valor = 50m });

        var resposta = await _cliente.GetAsync($"/contas/{contaId}/saldo");
        resposta.StatusCode.Should().Be(HttpStatusCode.OK);
        var corpo = await resposta.Content.ReadFromJsonAsync<JsonElement>();
        corpo.GetProperty("saldo").GetDecimal().Should().Be(250m);
    }

    [Fact]
    public async Task GetSaldoEm_DeveRetornarSaldoHistoricoCorreto()
    {
        var contaId = await CriarContaAsync();
        await _cliente.PostAsJsonAsync($"/contas/{contaId}/movimentacoes", new { tipo = "Credito", valor = 100m });
        var marcaTemporal = DateTime.UtcNow;
        await _cliente.PostAsJsonAsync($"/contas/{contaId}/movimentacoes", new { tipo = "Credito", valor = 200m });

        var dataFormatada = Uri.EscapeDataString(marcaTemporal.ToString("o"));
        var resposta = await _cliente.GetAsync($"/contas/{contaId}/saldo?em={dataFormatada}");
        resposta.StatusCode.Should().Be(HttpStatusCode.OK);
        var corpo = await resposta.Content.ReadFromJsonAsync<JsonElement>();
        corpo.GetProperty("saldo").GetDecimal().Should().Be(100m);
    }

    [Fact]
    public async Task PostMovimentacoes_ComValorZero_DeveRetornar400()
    {
        var contaId = await CriarContaAsync();
        var resposta = await _cliente.PostAsJsonAsync($"/contas/{contaId}/movimentacoes",
            new { tipo = "Credito", valor = 0m });
        resposta.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }

    [Fact]
    public async Task GetSaldo_ContaInexistente_DeveRetornar404()
    {
        var resposta = await _cliente.GetAsync($"/contas/{Guid.NewGuid()}/saldo");
        resposta.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }

    [Fact]
    public async Task PostMovimentacoes_ComMesmaChaveIdempotencia_NaoDeveDuplicarLancamento()
    {
        var clienteLocal = fabrica.CreateClient();
        var contaId = await CriarContaAsync();
        var chave = Guid.NewGuid().ToString();
        clienteLocal.DefaultRequestHeaders.Add("Idempotency-Key", chave);

        await clienteLocal.PostAsJsonAsync($"/contas/{contaId}/movimentacoes", new { tipo = "Credito", valor = 100m });
        await clienteLocal.PostAsJsonAsync($"/contas/{contaId}/movimentacoes", new { tipo = "Credito", valor = 100m });

        var resposta = await _cliente.GetAsync($"/contas/{contaId}/saldo");
        var corpo = await resposta.Content.ReadFromJsonAsync<JsonElement>();
        corpo.GetProperty("saldo").GetDecimal().Should().Be(100m);
    }

    [Fact]
    public async Task GetSaude_DeveRetornar200()
    {
        var resposta = await _cliente.GetAsync("/saude");
        resposta.StatusCode.Should().Be(HttpStatusCode.OK);
    }
}
```

- [ ] **Passo 3: Executar testes de integração**

```bash
dotnet test tests/Api.Testes --logger "console;verbosity=normal"
```
Esperado: PASS — todos os testes passando.

- [ ] **Passo 4: Executar suite completa**

```bash
dotnet test
```
Esperado: PASS — todos os testes de domínio + integração passando.

- [ ] **Passo 5: Commit**

```bash
git add tests/Api.Testes/
git commit -m "test: testes de integracao cobrindo todos os endpoints e cenarios de erro"
```

---

## Tarefa 14: Testes de Concorrência e Integridade do Ledger

**Arquivos:**
- Criar: `tests/Api.Testes/Contas/ContasConcorrenciaTestes.cs`

- [ ] **Passo 1: Criar testes de concorrência**

Criar `tests/Api.Testes/Contas/ContasConcorrenciaTestes.cs`:

```csharp
using System.Net;
using System.Net.Http.Json;
using System.Text.Json;
using FluentAssertions;
using Microsoft.Extensions.DependencyInjection;
using MovimentacoesFinanceiras.Dominio.Contas;
using MovimentacoesFinanceiras.Infraestrutura.Persistencia;

namespace Api.Testes.Contas;

public class ContasConcorrenciaTestes(FabricaDeAplicacao fabrica) : IClassFixture<FabricaDeAplicacao>
{
    private async Task<Guid> CriarContaComSaldo(HttpClient cliente, decimal saldoInicial)
    {
        var resposta = await cliente.PostAsJsonAsync("/contas", new { clienteId = Guid.NewGuid() });
        var json = await resposta.Content.ReadFromJsonAsync<JsonElement>();
        var contaId = json.GetProperty("id").GetGuid();

        await cliente.PostAsJsonAsync($"/contas/{contaId}/movimentacoes",
            new { tipo = "Credito", valor = saldoInicial });

        return contaId;
    }

    [Fact]
    public async Task DebitosSimultaneos_NaoDevemGerarSaldoNegativo()
    {
        var cliente = fabrica.CreateClient();
        var contaId = await CriarContaComSaldo(cliente, 100m);

        var tarefas = Enumerable.Range(0, 10)
            .Select(_ => cliente.PostAsJsonAsync(
                $"/contas/{contaId}/movimentacoes",
                new { tipo = "Debito", valor = 20m }))
            .ToList();

        var respostas = await Task.WhenAll(tarefas);

        var sucessos = respostas.Count(r => r.StatusCode == HttpStatusCode.Created);
        var falhas422 = respostas.Count(r => r.StatusCode == HttpStatusCode.UnprocessableEntity);

        sucessos.Should().Be(5, "apenas 5 débitos de R$20 cabem num saldo de R$100");
        falhas422.Should().BeGreaterThan(0, "os demais devem falhar por saldo insuficiente");

        var respostaSaldo = await cliente.GetAsync($"/contas/{contaId}/saldo");
        var json = await respostaSaldo.Content.ReadFromJsonAsync<JsonElement>();
        json.GetProperty("saldo").GetDecimal().Should().Be(0m, "o saldo deve ser exatamente zero");
    }

    [Fact]
    public async Task CreditosSimultaneos_DevemSerTodosPersistidos()
    {
        var cliente = fabrica.CreateClient();
        var contaId = await CriarContaComSaldo(cliente, 0.01m);

        var tarefas = Enumerable.Range(0, 10)
            .Select(_ => cliente.PostAsJsonAsync(
                $"/contas/{contaId}/movimentacoes",
                new { tipo = "Credito", valor = 50m }))
            .ToList();

        var respostas = await Task.WhenAll(tarefas);
        respostas.All(r => r.StatusCode == HttpStatusCode.Created)
            .Should().BeTrue("todos os créditos devem ser aceitos");

        var respostaSaldo = await cliente.GetAsync($"/contas/{contaId}/saldo");
        var json = await respostaSaldo.Content.ReadFromJsonAsync<JsonElement>();
        json.GetProperty("saldo").GetDecimal().Should().Be(500.01m);
    }

    [Fact]
    public async Task AposMultiplasOperacoes_SaldoSnapshotDeveSerIgualAoSomaDosLancamentos()
    {
        using var escopo = fabrica.Services.CreateScope();
        var contexto = escopo.ServiceProvider.GetRequiredService<ContextoBancoDados>();
        var cliente = fabrica.CreateClient();

        var contaId = await CriarContaComSaldo(cliente, 0.01m);

        var creditosTarefas = Enumerable.Range(0, 20)
            .Select(_ => cliente.PostAsJsonAsync($"/contas/{contaId}/movimentacoes",
                new { tipo = "Credito", valor = 50m }));
        await Task.WhenAll(creditosTarefas);

        var debitosTarefas = Enumerable.Range(0, 5)
            .Select(_ => cliente.PostAsJsonAsync($"/contas/{contaId}/movimentacoes",
                new { tipo = "Debito", valor = 30m }));
        await Task.WhenAll(debitosTarefas);

        var conta = await contexto.Contas.FindAsync(contaId);
        var saldoSnapshot = conta!.SaldoAtual;

        var lancamentos = contexto.Lancamentos.Where(l => l.ContaId == contaId).ToList();
        var saldoLedger = lancamentos.Sum(l => l.Tipo == TipoLancamento.Credito ? l.Valor : -l.Valor);

        saldoSnapshot.Should().Be(saldoLedger,
            "o snapshot saldo_atual nunca deve divergir do somatório dos lançamentos");
    }
}
```

- [ ] **Passo 2: Executar testes de concorrência**

```bash
dotnet test tests/Api.Testes --filter "ContasConcorrencia" --logger "console;verbosity=normal"
```
Esperado: PASS — 3 testes passando.

- [ ] **Passo 3: Executar suite completa**

```bash
dotnet test
```
Esperado: PASS — todos os testes passando (domínio + integração + concorrência).

- [ ] **Passo 4: Commit**

```bash
git add tests/Api.Testes/Contas/ContasConcorrenciaTestes.cs
git commit -m "test: testes de concorrencia e integridade do ledger — prova que saldo nunca diverge"
```

---

## Tarefa 15: README

**Arquivos:**
- Criar: `README.md`

- [ ] **Passo 1: Criar README**

Criar `README.md` na raiz do projeto:

```markdown
# Sistema de Movimentações Financeiras

Sistema bancário para registro de movimentações financeiras (créditos e débitos) e consulta de saldo histórico, construído como solução para o desafio técnico de arquiteto de software.

## Como rodar

### Pré-requisitos

- [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)

### Executar a aplicação

```bash
git clone <url-do-repositorio>
cd desafio-banco
dotnet run --project src/MovimentacoesFinanceiras.Api
```

A API sobe em `http://localhost:5000`. O Swagger está disponível na raiz: `http://localhost:5000`.

### Executar os testes

```bash
dotnet test
```

---

## Exemplos de uso

### Criar uma conta

```bash
curl -X POST http://localhost:5000/contas \
  -H "Content-Type: application/json" \
  -d '{"clienteId": "3fa85f64-5717-4562-b3fc-2c963f66afa6"}'
```

### Registrar um crédito

```bash
curl -X POST http://localhost:5000/contas/{id}/movimentacoes \
  -H "Content-Type: application/json" \
  -d '{"tipo": "Credito", "valor": 100.00, "descricao": "Depósito inicial"}'
```

### Consultar saldo atual

```bash
curl http://localhost:5000/contas/{id}/saldo
```

### Consultar saldo em ponto no tempo

```bash
curl "http://localhost:5000/contas/{id}/saldo?em=2025-01-15T12:00:00Z"
```

---

## Arquitetura

A solução usa **CQRS + Ledger append-only** dentro de uma **Clean Architecture** com **DDD**. A documentação completa das decisões arquiteturais está em:

- [`docs/superpowers/specs/2026-09-22-movimentacoes-financeiras-design.md`](docs/superpowers/specs/2026-09-22-movimentacoes-financeiras-design.md) — Design doc completo com ADRs

### Decisões principais

| Decisão | Justificativa |
|---|---|
| Ledger append-only | Rastreabilidade imutável; saldo histórico via SUM sem mecanismo extra |
| CQRS | Leitura e escrita com modelos e caminhos independentes |
| Concorrência otimista + Polly retry | Sem bloqueio de leituras; conflitos absorvidos internamente |
| Idempotência via `Idempotency-Key` | Retries do cliente nunca geram lançamentos duplicados |
| SQLite | Zero dependências externas para rodar localmente |

### O que ficou de fora (e por quê)

- **Autenticação JWT** — fora do escopo do desafio; ponto de extensão natural documentado
- **Snapshot periódico de saldo** — necessário para histórico de anos; ledger resolve o desafio
- **Circuit breaker** — relevante em produção; Polly já está no projeto como dependência
```

- [ ] **Passo 2: Verificar build e testes finais**

```bash
dotnet build
dotnet test
```
Esperado: `Build succeeded. 0 Warning(s). 0 Error(s).` e todos os testes passando.

- [ ] **Passo 3: Commit final**

```bash
git add README.md
git commit -m "docs: README com instrucoes de execucao, exemplos de uso e resumo arquitetural"
```

---

## Resumo das Tarefas

| # | Tarefa | Dia |
|---|---|---|
| 1 | Scaffold da solução | 1 |
| 2 | TipoLancamento + Dinheiro + testes | 1 |
| 3 | Exceções + Lancamento | 1 |
| 4 | Conta (Agregado) + testes | 1 |
| 5 | IContaRepositorio | 1 |
| 6 | EF Core + Configurações | 1 |
| 7 | ContaRepositorio | 1 |
| 8 | RegistrarMovimentacao (Command + Polly) | 2 |
| 9 | ConsultarSaldo + ConsultarSaldoEm | 2 |
| 10 | Program.cs (Serilog + Swagger + HealthCheck) | 2 |
| 11 | Middlewares (CorrelacaoId + Excecoes) | 2 |
| 12 | ContasControlador | 2 |
| 13 | Testes de integração | 2 |
| 14 | Testes de concorrência e integridade | 3 |
| 15 | README | 3 |
