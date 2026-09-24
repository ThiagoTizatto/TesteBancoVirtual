# Resiliência, Observabilidade e Prova Externa Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. **Execute os subagentes de implementação com o modelo Sonnet.**

**Goal:** Fechar os eixos de "alta demanda" e "indisponibilidade parcial": rate limiting por API Key, circuit breaker separado do retry, métricas via OpenTelemetry/Prometheus, load test com NBomber, CI no GitHub Actions e documentação final.

**Architecture:** Rate limiter nativo do .NET particiona por API Key. Um `CircuitBreakerPolicy` (Polly) envolve o retry de concorrência via `PolicyWrap`, protegendo contra falha de conectividade do banco. Métricas instrumentadas com `Meter` nativo e exportadas via OpenTelemetry Prometheus em `/metrics`. Load test isolado (fora do CI). CI valida build sem warnings + testes.

**Tech Stack:** .NET 10 RateLimiter, Polly 8, OpenTelemetry + Prometheus exporter, NBomber, GitHub Actions.

**Pré-requisito:** Planos `fundacao`, `dominio-arquitetura` e `seguranca` concluídos.

**Referências:** design doc seção 8; ADRs 023, 024, 025, 026, 027, 028.

---

## Estrutura de Arquivos

- **Modificar** `src/.../Api/DependencyInjection.cs` — rate limiter, OpenTelemetry.
- **Modificar** `src/.../Api/Program.cs` — `UseRateLimiter`, `MapPrometheusScrapingEndpoint`.
- **Modificar** `src/.../Api/Controllers/ContasController.cs` — `[EnableRateLimiting]`.
- **Criar** `src/.../Aplicacao/Contas/Commands/RegistrarMovimentacao/PoliticasResiliencia.cs` — retry + breaker.
- **Modificar** `src/.../Aplicacao/.../RegistrarMovimentacaoHandler.cs` — usar PolicyWrap + métricas.
- **Criar** `src/.../Aplicacao/Metricas/MetricasMovimentacao.cs` — `Meter` e instrumentos.
- **Modificar** `src/.../Aplicacao/DependencyInjection.cs` — registrar métricas.
- **Criar** `tests/Carga.Testes/` — projeto NBomber (fora da solução de teste padrão do CI).
- **Criar** `.github/workflows/ci.yml` — pipeline.
- **Modificar** `README.md` — métricas, load test, resumo final.

---

## Task 1: Rate limiting particionado por API Key

**Files:**
- Modify: `src/MovimentacoesFinanceiras.Api/DependencyInjection.cs`
- Modify: `src/MovimentacoesFinanceiras.Api/Program.cs`
- Modify: `src/MovimentacoesFinanceiras.Api/Controllers/ContasController.cs`

- [ ] **Step 1: Registrar o rate limiter em `AddApi()`**

Em `DependencyInjection.cs`, adicione antes do `return services;`:

```csharp
        services.AddRateLimiter(opcoes =>
        {
            opcoes.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

            opcoes.AddPolicy("por-api-key", contexto =>
            {
                var chave = contexto.Request.Headers["X-Api-Key"].FirstOrDefault()
                    ?? contexto.Connection.RemoteIpAddress?.ToString()
                    ?? "anonimo";

                return RateLimitPartition.GetFixedWindowLimiter(chave, _ => new FixedWindowRateLimiterOptions
                {
                    PermitLimit = 100,
                    Window = TimeSpan.FromSeconds(10),
                    QueueLimit = 0
                });
            });
        });
```

Adicione os `using`:
```csharp
using System.Threading.RateLimiting;
using Microsoft.AspNetCore.RateLimiting;
```

- [ ] **Step 2: Ativar o middleware no pipeline**

Em `Program.cs`, adicione `app.UseRateLimiter();` após `app.UseAuthorization();` e antes de `app.MapControllers();`.

- [ ] **Step 3: Aplicar a policy ao controller**

Em `ContasController.cs`, adicione `[EnableRateLimiting("por-api-key")]` na classe (com `using Microsoft.AspNetCore.RateLimiting;`):

```csharp
[Authorize]
[EnableRateLimiting("por-api-key")]
public class ContasController(IMediator mediador) : ControllerBase
```

- [ ] **Step 4: Build + testes**

Run: `dotnet build -warnaserror && dotnet test`
Expected: build limpo; testes passam (o limite de 100/10s não é atingido pelos testes funcionais).

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "feat(api): rate limiting particionado por API Key (429)"
```

---

## Task 2: Circuit breaker separado do retry (PolicyWrap)

**Files:**
- Create: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Commands/RegistrarMovimentacao/PoliticasResiliencia.cs`
- Modify: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Commands/RegistrarMovimentacao/RegistrarMovimentacaoHandler.cs`

- [ ] **Step 1: Extrair as políticas para uma classe dedicada**

`PoliticasResiliencia.cs`:

```csharp
using Microsoft.EntityFrameworkCore;
using Npgsql;
using Polly;
using Polly.CircuitBreaker;
using Polly.Retry;

namespace MovimentacoesFinanceiras.Aplicacao.Contas.Commands.RegistrarMovimentacao;

public static class PoliticasResiliencia
{
    // Retry para conflitos de concorrência — falha transitória, esperada sob carga.
    private static readonly AsyncRetryPolicy Retry = Policy
        .Handle<DbUpdateConcurrencyException>()
        .WaitAndRetryAsync(3, tentativa => TimeSpan.FromMilliseconds(50 * Math.Pow(2, tentativa)));

    // Circuit breaker para indisponibilidade do banco — abre após falhas de
    // conectividade consecutivas, falhando rápido para dar tempo de recuperação.
    private static readonly AsyncCircuitBreakerPolicy Breaker = Policy
        .Handle<NpgsqlException>()
        .Or<DbUpdateException>(ex => ex is not DbUpdateConcurrencyException)
        .CircuitBreakerAsync(
            exceptionsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(15));

    // Breaker (externo) envolve o Retry (interno): concorrência é retentada;
    // falha de conectividade abre o circuito sem retentar contra um banco morto.
    public static readonly IAsyncPolicy Combinada = Policy.WrapAsync(Breaker, Retry);
}
```

> `RegistrarMovimentacaoHandler` já referencia EF Core; garanta que a Aplicacao referencia Npgsql. Se não referenciar, adicione `<PackageReference Include="Npgsql" Version="10.*" />` ao `MovimentacoesFinanceiras.Aplicacao.csproj`.

- [ ] **Step 2: Usar a política combinada no handler**

Em `RegistrarMovimentacaoHandler.cs`, remova o campo `PoliticaRetentativa` (linhas 14-16) e troque a chamada `PoliticaRetentativa.ExecuteAsync(...)` (linha 32) por `PoliticasResiliencia.Combinada.ExecuteAsync(...)`. O corpo do lambda permanece idêntico.

- [ ] **Step 3: Build + testes (concorrência valida o retry interno)**

Run: `dotnet build -warnaserror && dotnet test`
Expected: build limpo; os testes de concorrência (10 débitos/créditos paralelos) continuam passando — o retry interno resolve os conflitos como antes; o breaker não dispara em operação normal.

- [ ] **Step 4: Commit**

```bash
git add src/
git commit -m "feat(resiliencia): circuit breaker de conectividade via PolicyWrap sobre o retry"
```

---

## Task 3: Métricas com Meter nativo

**Files:**
- Create: `src/MovimentacoesFinanceiras.Aplicacao/Metricas/MetricasMovimentacao.cs`
- Modify: `src/MovimentacoesFinanceiras.Aplicacao/DependencyInjection.cs`
- Modify: `src/MovimentacoesFinanceiras.Aplicacao/Contas/Commands/RegistrarMovimentacao/RegistrarMovimentacaoHandler.cs`

- [ ] **Step 1: Criar a classe de métricas**

`src/MovimentacoesFinanceiras.Aplicacao/Metricas/MetricasMovimentacao.cs`:

```csharp
using System.Diagnostics.Metrics;

namespace MovimentacoesFinanceiras.Aplicacao.Metricas;

public class MetricasMovimentacao
{
    public const string NomeMeter = "MovimentacoesFinanceiras";

    private readonly Counter<long> _movimentacoes;

    public MetricasMovimentacao(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create(NomeMeter);
        _movimentacoes = meter.CreateCounter<long>(
            "movimentacoes.total",
            description: "Total de movimentações registradas, por tipo.");
    }

    public void RegistrarMovimentacao(string tipo) =>
        _movimentacoes.Add(1, new KeyValuePair<string, object?>("tipo", tipo));
}
```

- [ ] **Step 2: Registrar como singleton em `AddAplicacao()`**

Em `DependencyInjection.cs` da Aplicacao, adicione antes do `return services;`:

```csharp
        services.AddSingleton<MovimentacoesFinanceiras.Aplicacao.Metricas.MetricasMovimentacao>();
```

- [ ] **Step 3: Instrumentar o handler**

Em `RegistrarMovimentacaoHandler.cs`, injete `MetricasMovimentacao` no construtor:

```csharp
public class RegistrarMovimentacaoHandler(
    IContaRepository repositorio,
    IServiceScopeFactory escopoFactory,
    MovimentacoesFinanceiras.Aplicacao.Metricas.MetricasMovimentacao metricas)
    : IRequestHandler<RegistrarMovimentacaoCommand, LancamentoResponse>
```

Dentro do lambda de `ExecuteAsync`, após `await repo.SalvarAsync(...)` e antes do `return ToResponse(lancamento);`, adicione:

```csharp
            metricas.RegistrarMovimentacao(request.Tipo.ToString());
```

- [ ] **Step 4: Build + testes**

Run: `dotnet build -warnaserror && dotnet test`
Expected: build limpo; testes passam.

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "feat(observabilidade): métrica de movimentações via Meter nativo"
```

---

## Task 4: Expor métricas via OpenTelemetry/Prometheus

**Files:**
- Modify: `src/MovimentacoesFinanceiras.Api/MovimentacoesFinanceiras.Api.csproj`
- Modify: `src/MovimentacoesFinanceiras.Api/DependencyInjection.cs`
- Modify: `src/MovimentacoesFinanceiras.Api/Program.cs`

- [ ] **Step 1: Adicionar pacotes OpenTelemetry**

No `MovimentacoesFinanceiras.Api.csproj`, adicione:

```xml
<PackageReference Include="OpenTelemetry.Extensions.Hosting" Version="1.*" />
<PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.*" />
<PackageReference Include="OpenTelemetry.Exporter.Prometheus.AspNetCore" Version="1.*" />
```

> O exporter Prometheus da AspNetCore pode estar em versão `-beta`. Se o restore falhar por versão estável inexistente, fixe a versão beta mais recente disponível (ex: `1.9.0-beta.2`) e registre isso como nota no ADR-025.

- [ ] **Step 2: Registrar OpenTelemetry em `AddApi()`**

Em `DependencyInjection.cs`, adicione antes do `return services;`:

```csharp
        services.AddOpenTelemetry()
            .WithMetrics(m => m
                .AddAspNetCoreInstrumentation()
                .AddMeter(MovimentacoesFinanceiras.Aplicacao.Metricas.MetricasMovimentacao.NomeMeter)
                .AddPrometheusExporter());
```

Adicione `using OpenTelemetry.Metrics;`.

- [ ] **Step 3: Mapear o endpoint `/metrics`**

Em `Program.cs`, após `app.MapHealthChecks("/saude");`, adicione:

```csharp
app.MapPrometheusScrapingEndpoint("/metrics");
```

- [ ] **Step 4: Build + testes**

Run: `dotnet build -warnaserror && dotnet test`
Expected: build limpo; testes passam.

- [ ] **Step 5: Validação manual do endpoint**

Suba a API (com Postgres) e rode `curl http://localhost:8080/metrics`.
Expected: saída no formato Prometheus, incluindo `http.server.request.duration` (instrumentação AspNetCore) e, após uma movimentação, `movimentacoes_total`.

- [ ] **Step 6: Commit**

```bash
git add src/
git commit -m "feat(observabilidade): exportação de métricas Prometheus em /metrics"
```

---

## Task 5: Load test com NBomber

**Files:**
- Create: `tests/Carga.Testes/Carga.Testes.csproj`
- Create: `tests/Carga.Testes/CenarioMovimentacoes.cs`
- Create: `tests/Carga.Testes/Program.cs`

> Este projeto é um executável de console (não roda no `dotnet test` do CI). Não o adicione às dependências de teste da CI.

- [ ] **Step 1: Criar o projeto**

`tests/Carga.Testes/Carga.Testes.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <IsPackable>false</IsPackable>
    <!-- Não incluído no CI; executado sob demanda -->
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="NBomber" Version="5.*" />
    <PackageReference Include="NBomber.Http" Version="5.*" />
  </ItemGroup>

</Project>
```

- [ ] **Step 2: Criar o cenário**

`tests/Carga.Testes/CenarioMovimentacoes.cs`:

```csharp
using System.Net.Http.Json;
using NBomber.CSharp;
using NBomber.Http.CSharp;

namespace Carga.Testes;

public static class CenarioMovimentacoes
{
    public static ScenarioProps Criar(string baseUrl, string apiKey, Guid contaId)
    {
        var http = new HttpClient();
        http.DefaultRequestHeaders.Add("X-Api-Key", apiKey);

        return Scenario.Create("creditos_concorrentes", async contexto =>
        {
            var requisicao = Http.CreateRequest("POST", $"{baseUrl}/contas/{contaId}/movimentacoes")
                .WithHeader("X-Api-Key", apiKey)
                .WithJsonBody(new { tipo = "Credito", valor = 10.0m, descricao = "carga" });

            var resposta = await Http.Send(http, requisicao);
            return resposta;
        })
        .WithLoadSimulations(
            Simulation.Inject(rate: 100, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromSeconds(30)));
    }
}
```

- [ ] **Step 3: Criar o `Program.cs`**

`tests/Carga.Testes/Program.cs`:

```csharp
using System.Net.Http.Json;
using Carga.Testes;
using NBomber.CSharp;

var baseUrl = Environment.GetEnvironmentVariable("CARGA_BASE_URL") ?? "http://localhost:8080";
var apiKey = Environment.GetEnvironmentVariable("CARGA_API_KEY") ?? "dev-key-local-somente";

// Cria uma conta para o teste de carga.
using var http = new HttpClient();
http.DefaultRequestHeaders.Add("X-Api-Key", apiKey);
var criacao = await http.PostAsJsonAsync($"{baseUrl}/contas", new { clienteId = Guid.NewGuid() });
criacao.EnsureSuccessStatusCode();
var conta = await criacao.Content.ReadFromJsonAsync<ContaCriada>();

NBomberRunner
    .RegisterScenarios(CenarioMovimentacoes.Criar(baseUrl, apiKey, conta!.Id))
    .Run();

record ContaCriada(Guid Id);
```

- [ ] **Step 4: Build isolado do projeto de carga**

Run: `dotnet build tests/Carga.Testes -warnaserror`
Expected: `Build succeeded`. (Não rode como parte de `dotnet test`.)

- [ ] **Step 5: Execução manual documentada (não automatizar)**

Com a API no ar (`docker compose up`), rode:
```bash
dotnet run --project tests/Carga.Testes -c Release
```
Expected: NBomber gera relatório com throughput e latências p95/p99. Copie os números-chave para o README.

- [ ] **Step 6: Garantir que o CI não roda o projeto de carga**

Se a solução (`.slnx`) incluir `Carga.Testes`, o `dotnet test` na raiz pode tentar rodá-lo. Como é `OutputType=Exe` sem framework de teste, `dotnet test` o ignora (sem test adapter). Confirme:
Run: `dotnet test`
Expected: roda apenas `Dominio.Testes` e `Api.Testes`; `Carga.Testes` não aparece como suíte.

- [ ] **Step 7: Commit**

```bash
git add tests/Carga.Testes/
git commit -m "test(carga): cenário de load test com NBomber (execução sob demanda)"
```

---

## Task 6: CI no GitHub Actions

**Files:**
- Create: `.github/workflows/ci.yml`

- [ ] **Step 1: Criar o workflow**

`.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

jobs:
  build-e-testes:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build (sem warnings)
        run: dotnet build --no-restore -warnaserror -c Release

      - name: Test
        run: dotnet test --no-build -c Release --verbosity normal
```

> O runner `ubuntu-latest` tem Docker disponível, então os testes de integração via Testcontainers sobem o Postgres automaticamente. Não é necessário um serviço `postgres` no workflow — o Testcontainers gerencia o container.

- [ ] **Step 2: Validar sintaxe localmente (opcional)**

Se `act` ou similar estiver disponível, valide; caso contrário, confie no push. Confirme que `dotnet-version: '10.0.x'` corresponde ao SDK disponível no runner (se o .NET 10 ainda estiver em preview no runner, fixe a versão exata ou use `dotnet-quality: preview`).

- [ ] **Step 3: Commit e push para acionar o CI**

```bash
git add .github/workflows/ci.yml
git commit -m "ci: build sem warnings + testes no GitHub Actions"
git push
```
Expected: o workflow aparece na aba Actions do GitHub e conclui verde.

---

## Task 7: Documentação final

**Files:**
- Modify: `README.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Atualizar o README com os novos recursos**

Adicione/atualize seções: autenticação (`X-Api-Key` nos exemplos curl), rate limiting (limite e resposta 429), observabilidade (`/metrics` Prometheus, `/saude`), resiliência (retry + circuit breaker), load test (comando + números obtidos na Task 5), e um link para `docs/adr/` e para o design doc de hardening. Atualize a contagem de testes.

- [ ] **Step 2: Atualizar o CLAUDE.md**

Reflita no CLAUDE.md: Postgres (não SQLite), migrations, inicializadores por camada, autenticação por API Key, endpoints `/metrics`, e a nota de que a execução dos planos usa Sonnet. Remova referências obsoletas a SQLite/EnsureCreated.

- [ ] **Step 3: Commit**

```bash
git add README.md CLAUDE.md
git commit -m "docs: README e CLAUDE.md com auth, métricas, resiliência e load test"
```

---

## Self-Review (preenchido)

**Spec coverage:**
- ADR-023 rate limiting → Task 1 ✓
- ADR-024 circuit breaker PolicyWrap → Task 2 ✓
- ADR-025 métricas OpenTelemetry/Prometheus → Tasks 3, 4 ✓
- ADR-026 NBomber fora do CI → Task 5 ✓
- ADR-027 CI GitHub Actions → Task 6 ✓
- ADR-028 idempotência sem TTL → já documentada; sem código novo (mantida) ✓

**Placeholder scan:** sem TBD; todo passo com código/comando. Versões de pacote OpenTelemetry/NBomber usam wildcard com nota de fallback para beta.

**Type consistency:** `MetricasMovimentacao.NomeMeter` usado no registro do exporter (Task 4) e na classe (Task 3); `PoliticasResiliencia.Combinada` substitui `PoliticaRetentativa` no handler; `X-Api-Key` consistente entre rate limiter, NBomber e auth.

**Ponto de atenção 1:** a Task 2 e a Task 3 ambas modificam `RegistrarMovimentacaoHandler.cs` — se executadas por subagentes diferentes, execute-as em ordem (2 antes de 3) para evitar conflito no mesmo arquivo.

**Ponto de atenção 2:** o construtor do handler ganha um parâmetro novo (`MetricasMovimentacao`) na Task 3 — o MediatR resolve por DI automaticamente, mas confirme que o registro singleton (Task 3 Step 2) foi feito antes de rodar os testes.

**Ponto de atenção 3:** ADR-025 previa também histograma de latência de handler e contadores de retry/rate-limit. Este plano implementa o contador de movimentações + instrumentação AspNetCore (que já fornece latência HTTP por endpoint). Os contadores de retry/rate-limit são incrementos naturais: se quiser cobri-los, adicione instrumentos análogos em `MetricasMovimentacao` e chame-os no `onRetry` do Polly e numa rejeição do limiter. Documentado como refinamento para não inflar o escopo mínimo do eixo de observabilidade.
