# Design: Hardening para Produção — Movimentações Financeiras

**Data:** 2026-09-23
**Autor:** Thiago
**Status:** Aprovado
**Depende de:** [2026-09-22-movimentacoes-financeiras-design.md](2026-09-22-movimentacoes-financeiras-design.md)

---

## Índice

1. [Objetivo e Contexto](#1-objetivo-e-contexto)
2. [Princípios Transversais](#2-princípios-transversais)
3. [Composition Root e Inicializadores de Camada](#3-composition-root-e-inicializadores-de-camada)
4. [Fundação de Build, Postgres, Migrations e Docker](#4-fundação-de-build-postgres-migrations-e-docker)
5. [Domínio: Dinheiro rico e SaldoAtual como VO](#5-domínio-dinheiro-rico-e-saldoatual-como-vo)
6. [Arquitetura: fechar brechas de CQRS e Clean Architecture](#6-arquitetura-fechar-brechas-de-cqrs-e-clean-architecture)
7. [Segurança de Dados Sensíveis](#7-segurança-de-dados-sensíveis)
8. [Resiliência, Observabilidade e Prova Externa](#8-resiliência-observabilidade-e-prova-externa)
9. [Impacto nos Testes](#9-impacto-nos-testes)
10. [Ordem de Execução](#10-ordem-de-execução)
11. [Índice de ADRs](#11-índice-de-adrs)

---

## 1. Objetivo e Contexto

O sistema de movimentações financeiras já implementa o núcleo do desafio (ledger append-only + snapshot, concorrência otimista, idempotência, CQRS). Este documento descreve a evolução para **prontidão de produção**, cruzando cada mudança com os critérios explícitos do desafio técnico:

- *"Código fonte deve compilar sem erros e warnings"* (requisito obrigatório eliminatório).
- *"Como a solução se comportaria sob alta demanda ou indisponibilidade parcial."*
- *"Como proteger dados sensíveis dos clientes."*
- *"Estamos interessados em entender suas decisões tanto quanto no código."*

O escopo cobre os Tiers 0–3 levantados na avaliação: trava de warnings, PostgreSQL, migrations, Docker, segurança (API Key + HTTPS + redaction), correção de vazamentos arquiteturais, `Dinheiro` como VO rico, rate limiting, circuit breaker, métricas, CI, load test e documentação (ADRs por arquivo).

**Não-objetivos:** JWT com identidade de cliente, tabela de credenciais, snapshot periódico de saldo histórico, paginação por cursor. Todos documentados como evolução em ADRs.

---

## 2. Princípios Transversais

Dois princípios se aplicam a **todas** as seções seguintes:

- **Inicializadores por camada.** Cada projeto expõe um único método `Add<Camada>()` que encapsula todo o seu wiring de DI. O `Program.cs` é uma *composition root* magra que só orquestra os inicializadores. Nenhum registro de serviço vive espalhado no startup.
- **Toda decisão com alternativa vira ADR.** Nenhuma escolha arquitetural fica apenas no código. ADRs são arquivos separados em `docs/adr/`, numerados sequencialmente a partir de ADR-007 (continuando os ADR-001..006 já existentes no design doc anterior), no formato Contexto → Decisão → Alternativas → Consequências → Status.

---

## 3. Composition Root e Inicializadores de Camada

Cada projeto ganha um `DependencyInjection.cs` na raiz, com um método de extensão sobre `IServiceCollection`:

| Camada | Método | Responsabilidade de wiring |
|--------|--------|----------------------------|
| Dominio | `AddDominio()` | Domain services puros (placeholder honesto; domínio não depende de infra) |
| Aplicacao | `AddAplicacao()` | MediatR + handlers, validators FluentValidation, `ValidationBehavior` |
| Infraestrutura | `AddInfraestrutura(IConfiguration)` | `DbContext` Postgres, `IContaRepository`→`ContaRepository`, health checks, Polly policies |
| Api | `AddApi()` | Controllers, Swagger, API Key auth, rate limiter, ProblemDetails, métricas |

```csharp
// Program.cs — composition root magra
builder.Services
    .AddDominio()
    .AddAplicacao()
    .AddInfraestrutura(builder.Configuration)
    .AddApi();
```

O `Program.cs` não sabe *como* cada camada se monta, só *que* ela se monta. O pipeline de middlewares permanece no `Program.cs` (é responsabilidade da composition root ordenar o pipeline HTTP), na ordem: HTTPS → HSTS → correlação → autenticação → autorização → rate limiter → tratador de exceções → endpoints.

Ver **ADR-007**.

---

## 4. Fundação de Build, Postgres, Migrations e Docker

### 4.1 Trava de warnings (eliminatório)

`Directory.Build.props` na raiz, herdado por todos os `.csproj`:

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <AnalysisMode>All</AnalysisMode>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  </PropertyGroup>
</Project>
```

Todo warning vira erro e é corrigido neste passo. As propriedades redundantes (`TargetFramework`, `Nullable`, `ImplicitUsings`) são removidas dos `.csproj` individuais. Ver **ADR-008**.

### 4.2 SQLite → PostgreSQL

- Trocar `Microsoft.EntityFrameworkCore.Sqlite` por `Npgsql.EntityFrameworkCore.PostgreSQL` na Infraestrutura.
- `AddInfraestrutura` usa `UseNpgsql(cfg.GetConnectionString("Postgres"))`.
- `ContaConfiguration`: `versao_linha` de `Guid↔string TEXT` passa a `uuid` nativo; `saldo_atual` mapeado como `numeric(18,2)`.
- `VersaoLinha` **permanece um Guid manual** (não migra para `xmin` nativo do Postgres) para não acoplar o domínio a um detalhe do provider. Ver **ADR-010**.
- Índice único de idempotência vira *filtered index* (`WHERE chave_idempotencia IS NOT NULL`), preservando a semântica de múltiplos NULLs.

Ver **ADR-009** (Postgres como padrão único).

### 4.3 Migrations EF

- `dotnet ef migrations add InicialPostgres`.
- Startup aplica `db.Database.MigrateAsync()` em vez de `EnsureCreatedAsync()`.
- Remove a instrução "apague o `.db`" do README/CLAUDE.md. Ver **ADR-011**.

### 4.4 Docker

- `Dockerfile` multi-stage (SDK build → runtime aspnet).
- `docker-compose.yml`: serviços `api` + `postgres` (volume, healthcheck, `depends_on`). Connection string e API Key via env var.
- README: `docker-compose up` como caminho principal. Ver **ADR-012**.

---

## 5. Domínio: Dinheiro rico e SaldoAtual como VO

### 5.1 `Dinheiro` como VO completo

```csharp
public sealed class Dinheiro : IEquatable<Dinheiro>
{
    public decimal Quantia { get; }
    private Dinheiro(decimal quantia) => Quantia = quantia;

    public static Dinheiro Zero { get; } = new(0m);

    // Entrada de movimentação continua exigindo > 0
    public static Dinheiro De(decimal quantia) =>
        quantia <= 0
            ? throw new ArgumentException("Quantia deve ser positiva.", nameof(quantia))
            : new Dinheiro(quantia);

    // Rehidratação de persistência — aceita qualquer valor (inclusive zero)
    internal static Dinheiro Reconstituir(decimal quantia) => new(quantia);

    public Dinheiro Somar(Dinheiro outro) => new(Quantia + outro.Quantia);
    public Dinheiro Subtrair(Dinheiro outro) => new(Quantia - outro.Quantia);

    public static Dinheiro operator +(Dinheiro a, Dinheiro b) => a.Somar(b);
    public static Dinheiro operator -(Dinheiro a, Dinheiro b) => a.Subtrair(b);
    public static bool operator >(Dinheiro a, Dinheiro b) => a.Quantia > b.Quantia;
    public static bool operator <(Dinheiro a, Dinheiro b) => a.Quantia < b.Quantia;
    public static bool operator >=(Dinheiro a, Dinheiro b) => a.Quantia >= b.Quantia;
    public static bool operator <=(Dinheiro a, Dinheiro b) => a.Quantia <= b.Quantia;
    // Equals/GetHashCode/==/!=/ToString(pt-BR) mantidos
}
```

`De()` preserva a invariante de *entrada* (não se movimenta zero). O saldo zerado usa `Dinheiro.Zero`. `Reconstituir` é `internal` (com `InternalsVisibleTo` para a Infraestrutura) e serve só à rehidratação do EF — padrão comum para não violar invariantes de entrada ao materializar do banco. Ver **ADR-014**.

### 5.2 `Conta.SaldoAtual` passa a ser `Dinheiro`

```csharp
public Dinheiro SaldoAtual { get; private set; }  // inicia Dinheiro.Zero na factory Criar

public Lancamento Creditar(Dinheiro valor, string? descricao, string? chaveIdempotencia = null)
{
    SaldoAtual = SaldoAtual + valor;
    VersaoLinha = Guid.NewGuid();
    var lancamento = Lancamento.CriarCredito(Id, valor, descricao, chaveIdempotencia);
    _lancamentos.Add(lancamento);
    return lancamento;
}

public Lancamento Debitar(Dinheiro valor, string? descricao, string? chaveIdempotencia = null)
{
    if (valor > SaldoAtual)
        throw new SaldoInsuficienteException(SaldoAtual.Quantia, valor);
    SaldoAtual = SaldoAtual - valor;
    VersaoLinha = Guid.NewGuid();
    var lancamento = Lancamento.CriarDebito(Id, valor, descricao, chaveIdempotencia);
    _lancamentos.Add(lancamento);
    return lancamento;
}
```

A invariante de saldo não-negativo continua na `Conta` (guard clause), não no `Dinheiro`. A comparação `valor > SaldoAtual` fica expressiva via operador do VO.

### 5.3 Mapeamento EF

`SaldoAtual` mapeado via value converter: `HasConversion(d => d.Quantia, v => Dinheiro.Reconstituir(v))`, coluna `numeric(18,2)`.

### 5.4 `Lancamento.Valor` permanece `decimal`

O valor do lançamento (registro imutável do ledger, sempre positivo) segue `decimal` — mantém as queries de `SUM` simples e evita conversões EF extras. Ver **ADR-013**.

---

## 6. Arquitetura: fechar brechas de CQRS e Clean Architecture

### 6.1 `CriarContaCommand`

Hoje `ContasController.CriarConta` fala direto com o `BancoDadosContext`. Passa a:
- `CriarContaCommand(Guid ClienteId) : IRequest<ContaResponse>` em `Aplicacao/Contas/Commands/CriarConta/`.
- `CriarContaHandler` usa `IContaRepository.AdicionarAsync` + `SalvarAsync`.
- `CriarContaValidator` (`ClienteId NotEmpty`).

### 6.2 Remover vazamento de infra da Api

- `ContasController` depende **exclusivamente de `IMediator`** — deixa de receber `BancoDadosContext` e `IContaRepository`.
- `BancoDadosContext` e `ContaRepository` viram `internal`, registrados dentro de `AddInfraestrutura`. `InternalsVisibleTo` para `Api.Testes`.
- A Api não referencia mais tipos da Infraestrutura — só Aplicacao e Dominio. Fluxo de dependência de Clean Architecture verificável. Ver **ADR-015**.

### 6.3 Saldo histórico via `SUM` no banco

`ConsultarSaldoEmAsync` deixa de trazer lançamentos para memória. Passa a agregação SQL traduzida pelo Npgsql:

```
SUM(CASE WHEN tipo = 'Credito' THEN valor ELSE -valor END)
WHERE conta_id = @id AND criado_em <= @data
```

No LINQ o `CASE` é expresso via ternário dentro do `Sum` (respeitando a regra "sem `else`"). Fallback documentado se a tradução falhar. Ver **ADR-016**.

### 6.4 ProblemDetails nativo

`TratadorDeExcecoesMiddleware` migra da serialização JSON manual para `IProblemDetailsService` + `AddProblemDetails()`, preservando os mapeamentos (Validation→400, SaldoInsuficiente→422, ContaNaoEncontrada→404, catch-all→500) e adicionando `type`/`title`/`status`/`detail`/`instance` + extensions (`erros`, `correlacao_id`). Ver **ADR-017**.

---

## 7. Segurança de Dados Sensíveis

### 7.1 Autenticação por API Key

- `AutenticacaoApiKeyHandler : AuthenticationHandler<AuthenticationSchemeOptions>` lê o header `X-Api-Key` e valida contra chaves configuradas (secrets/env).
- Registrado em `AddApi()` via `AddAuthentication().AddScheme(...)`. `[Authorize]` no `ContasController`.
- `/saude` e Swagger ficam `[AllowAnonymous]`.
- Falha de auth retorna `401` em ProblemDetails.
- As chaves válidas ficam em **configuração** (env/secrets), sem tabela — auth serviço-a-serviço stateless. Ver **ADR-018** (API Key vs JWT) e **ADR-021** (store em configuração).

### 7.2 HTTPS + HSTS

`app.UseHttpsRedirection()` + `app.UseHsts()` (fora de Development). README ajustado para o endpoint HTTPS. Ver **ADR-019**.

### 7.3 Redaction formal no Serilog

`IDestructuringPolicy` custom que redige `valor`, `saldo`, `descricao` e demais PII. Revisão dos log statements. `UseSerilogRequestLogging` configurado para não capturar corpo de request/response. Passa de *afirmação* a *garantia no código*. Ver **ADR-020**.

### 7.4 Secrets management

- Dev: `dotnet user-secrets`. Docker/prod: variáveis de ambiente.
- `appsettings.json` mantém só placeholders. Vault (Azure Key Vault / AWS Secrets Manager) documentado como destino de produção. Ver **ADR-022**.

---

## 8. Resiliência, Observabilidade e Prova Externa

### 8.1 Rate limiting

`AddRateLimiter` nativo em `AddApi()`, particionado por API Key (fallback IP). Resposta `429` em ProblemDetails. Ver **ADR-023**.

### 8.2 Circuit breaker

`CircuitBreakerPolicy` (Polly) protege contra **indisponibilidade do banco** (`DbUpdateException`/`NpgsqlException`/timeout) — alvo diferente do retry de concorrência (`DbUpdateConcurrencyException`). Combinação via `PolicyWrap`: retry de concorrência (interno) envolto pelo circuit breaker de infra (externo). Ver **ADR-024**.

### 8.3 Métricas

`System.Diagnostics.Metrics.Meter` nativo instrumentando: contador de movimentações por tipo, histograma de latência de handler, contador de conflitos de concorrência, contador de rejeições de rate limit. Exposição via OpenTelemetry + exporter Prometheus (`/metrics`). Ver **ADR-025**.

### 8.4 Load test

Projeto/script `tests/Carga.Testes` com **NBomber**: movimentações concorrentes medindo throughput e p95/p99. Não roda no CI (caro/lento) — executável sob demanda, resultados-exemplo no README. Ver **ADR-026**.

### 8.5 CI — GitHub Actions

`.github/workflows/ci.yml`: `restore` → `build` (com `-warnaserror`, provando o Tier 0 publicamente) → `test` (Testcontainers usa o Docker do runner ubuntu). Roda em push/PR. Ver **ADR-027**.

### 8.6 Idempotência — evolução documentada

Sem TTL/storage dedicado agora. ADR documenta o estado atual (chave única em `lancamentos`) e a evolução (tabela dedicada com TTL ou Redis). Ver **ADR-028**.

---

## 9. Impacto nos Testes

- **Testcontainers.** `AplicacaoFactory` sobe um container Postgres efêmero por instância (`Testcontainers.PostgreSql`), substituindo o SQLite temporário. Fiel ao engine de produção — testa concorrência real do Postgres. O runner ubuntu da CI já tem Docker. Ver **ADR-009**.
- **Domínio.** `DinheiroTestes` ganha casos para `Zero`, `Somar`, `Subtrair`, operadores de comparação, e `De()` ainda rejeitando zero/negativo. `ContaTestes` ajustados para `SaldoAtual: Dinheiro`.
- **Integração.** Novos casos para `CriarContaCommand` via `IMediator`, autenticação (401 sem API Key, 200/201 com), rate limit (429), saldo histórico via SQL.
- Todos seguem AAA com blocos separados e sem `else`.

---

## 10. Ordem de Execução

Sequencial por risco/dependência; cada passo deixa `dotnet build && dotnet test` verdes antes do próximo:

1. Fundação de build (`Directory.Build.props` + corrigir warnings).
2. Postgres + migrations + docker-compose + Testcontainers nos testes.
3. Domínio (`Dinheiro` rico + `SaldoAtual: Dinheiro` + value converter).
4. Arquitetura (inicializadores de camada, `CriarContaCommand`, remover vazamento, saldo SQL, ProblemDetails).
5. Segurança (API Key + HTTPS/HSTS + redaction + secrets).
6. Resiliência/observabilidade (rate limiter, circuit breaker, métricas).
7. Prova externa (CI, Dockerfile, load test).
8. Docs (README + ADRs + design doc).

---

## 11. Índice de ADRs

Continuando a numeração dos ADR-001..006 do design doc anterior. Arquivos em `docs/adr/`.

| ADR | Decisão | Seção |
|-----|---------|-------|
| ADR-007 | Inicializadores de camada + composition root magra | 3 |
| ADR-008 | `TreatWarningsAsErrors` / build sob trava | 4.1 |
| ADR-009 | PostgreSQL como padrão único + Testcontainers nos testes | 4.2, 9 |
| ADR-010 | `VersaoLinha` Guid manual (vs `xmin` nativo) | 4.2 |
| ADR-011 | Migrations EF (vs EnsureCreated) | 4.3 |
| ADR-012 | Docker/docker-compose como caminho principal | 4.4 |
| ADR-013 | `Lancamento.Valor` permanece `decimal` | 5.4 |
| ADR-014 | `SaldoAtual: Dinheiro` + `Reconstituir` para rehidratação | 5.1, 5.2 |
| ADR-015 | Encapsular infra com `internal` + `InternalsVisibleTo` | 6.2 |
| ADR-016 | Saldo histórico via agregação SQL | 6.3 |
| ADR-017 | ProblemDetails nativo (vs serialização manual) | 6.4 |
| ADR-018 | API Key (vs JWT) | 7.1 |
| ADR-019 | HTTPS + HSTS | 7.2 |
| ADR-020 | Redaction como policy de código no Serilog | 7.3 |
| ADR-021 | API Keys em configuração (vs tabela) | 7.1 |
| ADR-022 | Estratégia de secrets em camadas | 7.4 |
| ADR-023 | Rate limiting particionado por API Key | 8.1 |
| ADR-024 | Circuit breaker separado do retry (PolicyWrap) | 8.2 |
| ADR-025 | OpenTelemetry + Prometheus para métricas | 8.3 |
| ADR-026 | NBomber para load test, fora do CI | 8.4 |
| ADR-027 | CI GitHub Actions com `-warnaserror` | 8.5 |
| ADR-028 | Idempotência sem TTL (com trilha de evolução) | 8.6 |
