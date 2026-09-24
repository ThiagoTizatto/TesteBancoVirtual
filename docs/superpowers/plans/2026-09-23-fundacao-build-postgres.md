# Fundação: Build Lock + PostgreSQL + Migrations + Docker Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. **Execute os subagentes de implementação com o modelo Sonnet.**

**Goal:** Estabelecer a fundação de produção: trava de warnings, PostgreSQL como banco único, migrations EF, Docker/docker-compose e testes de integração via Testcontainers.

**Architecture:** Substitui SQLite por PostgreSQL em toda a stack. O build passa a falhar em qualquer warning (`Directory.Build.props`). O schema deixa de ser criado por `EnsureCreated` e passa a Migrations aplicadas no startup. Testes de integração sobem um Postgres efêmero por instância via Testcontainers.

**Tech Stack:** .NET 10, EF Core 10, Npgsql, Testcontainers.PostgreSql, Docker, docker-compose.

**Referências:** design doc `docs/superpowers/specs/2026-09-23-hardening-producao-design.md` (seção 4); ADRs 008, 009, 010, 011, 012.

**Pré-requisito de ambiente:** Docker Desktop rodando (necessário para Testcontainers e docker-compose).

---

## Estrutura de Arquivos

- **Criar** `Directory.Build.props` (raiz) — propriedades MSBuild herdadas por todos os projetos (trava de warnings, TFM, nullable).
- **Criar** `src/MovimentacoesFinanceiras.Api/Dockerfile` — build multi-stage da API.
- **Criar** `docker-compose.yml` (raiz) — serviços `api` + `postgres`.
- **Criar** `.dockerignore` (raiz) — exclui `bin/`, `obj/`, `.git` do contexto de build.
- **Modificar** `src/MovimentacoesFinanceiras.Infraestrutura/MovimentacoesFinanceiras.Infraestrutura.csproj` — troca provider Sqlite→Npgsql, remove props duplicadas.
- **Modificar** os demais `.csproj` — remove `TargetFramework`/`Nullable`/`ImplicitUsings` (agora no `Directory.Build.props`).
- **Modificar** `src/MovimentacoesFinanceiras.Api/Program.cs` — `UseNpgsql`, `MigrateAsync` no lugar de `EnsureCreatedAsync`.
- **Modificar** `src/MovimentacoesFinanceiras.Api/appsettings.json` — connection string Postgres.
- **Modificar** `src/.../Configurations/ContaConfiguration.cs` — `versao_linha` como `uuid`.
- **Modificar** `src/.../Configurations/LancamentoConfiguration.cs` — índice único filtrado.
- **Criar** `src/MovimentacoesFinanceiras.Infraestrutura/Migrations/` — migration inicial (gerada por `dotnet ef`).
- **Modificar** `tests/Api.Testes/Api.Testes.csproj` — adiciona Testcontainers.PostgreSql, remove Sqlite/InMemory.
- **Modificar** `tests/Api.Testes/AplicacaoFactory.cs` — sobe container Postgres, aplica migrations.

---

## Task 1: Trava de warnings (Directory.Build.props)

**Files:**
- Create: `Directory.Build.props`
- Modify: `src/MovimentacoesFinanceiras.Api/MovimentacoesFinanceiras.Api.csproj`
- Modify: `src/MovimentacoesFinanceiras.Infraestrutura/MovimentacoesFinanceiras.Infraestrutura.csproj`
- Modify: `src/MovimentacoesFinanceiras.Aplicacao/MovimentacoesFinanceiras.Aplicacao.csproj`
- Modify: `src/MovimentacoesFinanceiras.Dominio/MovimentacoesFinanceiras.Dominio.csproj`
- Modify: `tests/Api.Testes/Api.Testes.csproj`
- Modify: `tests/Dominio.Testes/Dominio.Testes.csproj`

- [ ] **Step 1: Criar `Directory.Build.props` na raiz**

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

- [ ] **Step 2: Remover propriedades duplicadas dos `.csproj`**

Em cada `.csproj` listado, remova as linhas `<TargetFramework>`, `<Nullable>` e `<ImplicitUsings>` do `<PropertyGroup>` (agora vêm do `Directory.Build.props`). Mantenha propriedades específicas como `<IsPackable>false</IsPackable>` e `Sdk="...Web"`. Exemplo do resultado para `MovimentacoesFinanceiras.Dominio.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">

</Project>
```

Para o `.csproj` da Api e Infra, mantenha os `<ItemGroup>` de PackageReference/ProjectReference intactos e apenas esvazie/remova o `<PropertyGroup>` com as 3 props herdadas.

- [ ] **Step 3: Build para expor warnings-virando-erro**

Run: `dotnet build -warnaserror`
Expected: pode FALHAR se houver warnings pré-existentes (analyzers CA, nullable). Anote cada erro.

- [ ] **Step 4: Corrigir cada warning exposto**

Para cada erro reportado no Step 3, corrija na origem (não suprima com `#pragma`). Erros comuns esperados: `CAxxxx` de analyzers, nullability. Se um analyzer específico for ruído legítimo e amplo, é aceitável desativá-lo pontualmente via `.editorconfig` com justificativa em comentário — mas prefira corrigir. Reexecute `dotnet build -warnaserror` após cada correção.

- [ ] **Step 5: Verificar build limpo**

Run: `dotnet build -warnaserror`
Expected: `Build succeeded. 0 Warning(s). 0 Error(s)`

- [ ] **Step 6: Rodar testes (garantir que nada quebrou)**

Run: `dotnet test`
Expected: todos passam (a suíte ainda usa SQLite neste ponto).

- [ ] **Step 7: Commit**

```bash
git add Directory.Build.props src/ tests/
git commit -m "build: Directory.Build.props com TreatWarningsAsErrors e build limpo"
```

---

## Task 2: Trocar provider EF para PostgreSQL

**Files:**
- Modify: `src/MovimentacoesFinanceiras.Infraestrutura/MovimentacoesFinanceiras.Infraestrutura.csproj`
- Modify: `src/MovimentacoesFinanceiras.Api/appsettings.json`
- Modify: `src/MovimentacoesFinanceiras.Api/appsettings.Development.json`
- Modify: `src/MovimentacoesFinanceiras.Api/Program.cs:20-22`

- [ ] **Step 1: Trocar o pacote NuGet na Infraestrutura**

No `MovimentacoesFinanceiras.Infraestrutura.csproj`, substitua a linha do provider Sqlite por Npgsql:

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.EntityFrameworkCore" Version="10.*" />
  <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="10.*" />
</ItemGroup>
```

- [ ] **Step 2: Atualizar a connection string**

Em `appsettings.json`, troque o bloco `ConnectionStrings`:

```json
"ConnectionStrings": {
  "Postgres": "Host=localhost;Port=5432;Database=movimentacoes;Username=postgres;Password=postgres"
},
```

Faça o mesmo em `appsettings.Development.json` se ele definir a connection string (caso não defina, deixe como está).

- [ ] **Step 3: Trocar `UseSqlite` por `UseNpgsql` no `Program.cs`**

Substitua as linhas 20-22:

```csharp
builder.Services.AddDbContext<BancoDadosContext>(opcoes =>
    opcoes.UseNpgsql(builder.Configuration.GetConnectionString("Postgres")
        ?? "Host=localhost;Port=5432;Database=movimentacoes;Username=postgres;Password=postgres"));
```

- [ ] **Step 4: Ajustar `versao_linha` para `uuid` nativo**

Em `ContaConfiguration.cs`, substitua o mapeamento de `VersaoLinha` (linhas 17-22) por (Postgres tem tipo `uuid` nativo, dispensando a conversão para string):

```csharp
builder.Property(c => c.VersaoLinha)
       .HasColumnName("versao_linha")
       .HasColumnType("uuid")
       .IsConcurrencyToken()
       .IsRequired();
```

- [ ] **Step 5: Índice único filtrado para idempotência**

Em `LancamentoConfiguration.cs`, substitua o comentário sobre SQLite (linhas 24-27) e o índice único por (no Postgres, um índice único trata múltiplos NULLs como distintos por padrão, mas tornamos explícito com filtro):

```csharp
        // No PostgreSQL, valores NULL são distintos em índices únicos por padrão.
        // Tornamos explícito com um índice parcial: unicidade apenas para chaves não-nulas.
        builder.HasIndex(l => l.ChaveIdempotencia)
               .IsUnique()
               .HasFilter("chave_idempotencia IS NOT NULL");
```

- [ ] **Step 6: Build**

Run: `dotnet build -warnaserror`
Expected: `Build succeeded. 0 Warning(s)`. (Testes ainda não rodam contra Postgres — isso é a Task 5.)

- [ ] **Step 7: Commit**

```bash
git add src/
git commit -m "feat: PostgreSQL como provider EF (substitui SQLite)"
```

---

## Task 3: Migrations EF (substitui EnsureCreated)

**Files:**
- Create: `src/MovimentacoesFinanceiras.Infraestrutura/Migrations/*` (gerado)
- Modify: `src/MovimentacoesFinanceiras.Api/Program.cs:52-57`

**Pré-condição:** Um Postgres acessível para o `dotnet ef` gerar a migration (design-time não conecta, mas o build sim). Se `dotnet ef` não estiver instalado: `dotnet tool install --global dotnet-ef`.

- [ ] **Step 1: Gerar a migration inicial**

Run (a partir da raiz):
```bash
dotnet ef migrations add InicialPostgres \
  --project src/MovimentacoesFinanceiras.Infraestrutura \
  --startup-project src/MovimentacoesFinanceiras.Api
```
Expected: cria `src/MovimentacoesFinanceiras.Infraestrutura/Migrations/<timestamp>_InicialPostgres.cs` + snapshot. Sem erro.

- [ ] **Step 2: Verificar a migration gerada**

Abra o arquivo `*_InicialPostgres.cs` e confirme: tabelas `contas` e `lancamentos`, coluna `versao_linha` do tipo `uuid`, índice único filtrado em `chave_idempotencia`, índice composto `(conta_id, criado_em)`.

- [ ] **Step 3: Trocar `EnsureCreatedAsync` por `MigrateAsync` no `Program.cs`**

Substitua o bloco das linhas 52-57:

```csharp
if (!app.Environment.IsEnvironment("Testing"))
{
    using var escopo = app.Services.CreateScope();
    var contexto = escopo.ServiceProvider.GetRequiredService<BancoDadosContext>();
    await contexto.Database.MigrateAsync();
}
```

- [ ] **Step 4: Build**

Run: `dotnet build -warnaserror`
Expected: `Build succeeded. 0 Warning(s)`.

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "feat: migrations EF aplicadas no startup (substitui EnsureCreated)"
```

---

## Task 4: Docker e docker-compose

**Files:**
- Create: `src/MovimentacoesFinanceiras.Api/Dockerfile`
- Create: `.dockerignore`
- Create: `docker-compose.yml`

- [ ] **Step 1: Criar `.dockerignore` na raiz**

```
**/bin/
**/obj/
.git/
.vs/
*.db
*.db-shm
*.db-wal
docs/
```

- [ ] **Step 2: Criar o `Dockerfile` multi-stage**

Em `src/MovimentacoesFinanceiras.Api/Dockerfile`:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY Directory.Build.props ./
COPY src/ ./src/
RUN dotnet restore src/MovimentacoesFinanceiras.Api/MovimentacoesFinanceiras.Api.csproj
RUN dotnet publish src/MovimentacoesFinanceiras.Api/MovimentacoesFinanceiras.Api.csproj \
    -c Release -o /app/publish --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080
ENTRYPOINT ["dotnet", "MovimentacoesFinanceiras.Api.dll"]
```

> Nota: o `Dockerfile` fica na Api mas o contexto de build é a raiz (para copiar `Directory.Build.props`). Isso é refletido no `docker-compose.yml` abaixo.

- [ ] **Step 3: Criar `docker-compose.yml` na raiz**

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: movimentacoes
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d movimentacoes"]
      interval: 5s
      timeout: 3s
      retries: 5

  api:
    build:
      context: .
      dockerfile: src/MovimentacoesFinanceiras.Api/Dockerfile
    environment:
      ConnectionStrings__Postgres: "Host=postgres;Port=5432;Database=movimentacoes;Username=postgres;Password=postgres"
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy

volumes:
  pgdata:
```

- [ ] **Step 4: Subir o compose e validar**

Run: `docker compose up --build -d`
Then: `docker compose ps`
Expected: `postgres` e `api` como `running`/`healthy`.

- [ ] **Step 5: Validar o health check da API**

Run: `curl http://localhost:8080/saude`
Expected: HTTP 200 (a API conectou no Postgres e aplicou migrations).

- [ ] **Step 6: Derrubar o compose**

Run: `docker compose down`

- [ ] **Step 7: Commit**

```bash
git add Dockerfile .dockerignore docker-compose.yml src/MovimentacoesFinanceiras.Api/Dockerfile
git commit -m "feat: Dockerfile multi-stage e docker-compose (api + postgres)"
```

---

## Task 5: Testes de integração via Testcontainers

**Files:**
- Modify: `tests/Api.Testes/Api.Testes.csproj`
- Modify: `tests/Api.Testes/AplicacaoFactory.cs`

- [ ] **Step 1: Trocar pacotes no `Api.Testes.csproj`**

Remova as linhas de `Microsoft.EntityFrameworkCore.InMemory` e `Microsoft.EntityFrameworkCore.Sqlite` e adicione Testcontainers:

```xml
<PackageReference Include="Testcontainers.PostgreSql" Version="4.*" />
<PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="10.*" />
```

Mantenha os demais pacotes (coverlet, FluentAssertions, Mvc.Testing, Test.Sdk, xunit).

- [ ] **Step 2: Reescrever `AplicacaoFactory.cs` para Testcontainers**

Substitua o conteúdo inteiro de `AplicacaoFactory.cs`:

```csharp
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.AspNetCore.TestHost;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using MovimentacoesFinanceiras.Infraestrutura.Persistencia;
using Testcontainers.PostgreSql;

namespace Api.Testes;

public class AplicacaoFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16")
        .WithDatabase("movimentacoes_teste")
        .WithUsername("postgres")
        .WithPassword("postgres")
        .Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.UseEnvironment("Testing");

        builder.ConfigureTestServices(servicos =>
        {
            var descritores = servicos
                .Where(d =>
                    d.ServiceType == typeof(DbContextOptions<BancoDadosContext>) ||
                    d.ServiceType == typeof(BancoDadosContext))
                .ToList();

            foreach (var descritor in descritores)
                servicos.Remove(descritor);

            servicos.AddDbContext<BancoDadosContext>(opcoes =>
                opcoes
                    .UseNpgsql(_postgres.GetConnectionString())
                    .EnableSensitiveDataLogging());
        });
    }

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();

        using var escopo = Services.CreateScope();
        var ctx = escopo.ServiceProvider.GetRequiredService<BancoDadosContext>();
        await ctx.Database.MigrateAsync();
    }

    public new async Task DisposeAsync()
    {
        await base.DisposeAsync();
        await _postgres.DisposeAsync();
    }
}
```

- [ ] **Step 3: Build**

Run: `dotnet build -warnaserror`
Expected: `Build succeeded. 0 Warning(s)`.

- [ ] **Step 4: Rodar a suíte completa (Docker precisa estar rodando)**

Run: `dotnet test`
Expected: todos os testes passam. Os testes de integração e concorrência agora rodam contra Postgres real em container. Primeira execução é mais lenta (pull da imagem `postgres:16`).

> Se algum teste de concorrência falhar por diferença de comportamento SQLite→Postgres (ex: contagem exata de débitos que passam), avalie se a asserção era acoplada ao SQLite. O comportamento correto é: nunca gerar saldo negativo. Ajuste a asserção para verificar a invariante (saldo final ≥ 0 e snapshot = soma do ledger), não um número mágico de sucessos.

- [ ] **Step 5: Commit**

```bash
git add tests/
git commit -m "test: integração via Testcontainers PostgreSQL (substitui SQLite)"
```

---

## Task 6: Atualizar documentação de execução

**Files:**
- Modify: `README.md`
- Modify: `CLAUDE.md`

- [ ] **Step 1: Atualizar o README**

Na seção "como rodar", substitua as instruções de SQLite/`dotnet run` por:

```markdown
## Como rodar

### Via Docker (recomendado)
```bash
docker compose up --build
```
API em `http://localhost:8080` (Swagger na raiz). Postgres sobe automaticamente.

### Local (requer Postgres)
Suba um Postgres (ex: `docker compose up postgres -d`), então:
```bash
dotnet run --project src/MovimentacoesFinanceiras.Api
```
As migrations são aplicadas automaticamente no startup.
```

Remova qualquer menção a "apague o `.db`". Ajuste a contagem de testes se necessário.

- [ ] **Step 2: Atualizar o CLAUDE.md**

Na seção de comandos/arquitetura, substitua a nota sobre `EnsureCreatedAsync` e "apagar o `.db`" por uma nota sobre migrations EF e Postgres via docker-compose. Atualize o "Target framework" se necessário e a menção a SQLite.

- [ ] **Step 3: Commit**

```bash
git add README.md CLAUDE.md
git commit -m "docs: instruções de execução com Docker/Postgres e migrations"
```

---

## Self-Review (preenchido)

**Spec coverage (seção 4 + ADRs 008-012):**
- ADR-008 TreatWarningsAsErrors → Task 1 ✓
- ADR-009 Postgres + Testcontainers → Tasks 2, 5 ✓
- ADR-010 VersaoLinha uuid → Task 2 Step 4 ✓
- ADR-011 Migrations → Task 3 ✓
- ADR-012 Docker → Task 4 ✓

**Placeholder scan:** nenhum TBD; todo passo tem código/comando concreto.

**Type consistency:** `BancoDadosContext`, `IContaRepository`, connection string key `"Postgres"` consistentes entre Program.cs, appsettings, AplicacaoFactory e docker-compose (`ConnectionStrings__Postgres`).

**Ponto de atenção herdado:** o arquivo `ContaRepository.cs` já estava modificado (não-commitado) antes deste plano. Antes da Task 2, verifique com `git diff src/.../ContaRepository.cs` se essa modificação deve ser mantida, commitada à parte ou revertida — não deixe pendência não relacionada entrar nos commits deste plano.
