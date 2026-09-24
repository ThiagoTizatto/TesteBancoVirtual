# Segurança: API Key + HTTPS/HSTS + Redaction + Secrets Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking. **Execute os subagentes de implementação com o modelo Sonnet.**

**Goal:** Proteger dados sensíveis: autenticação por API Key, HTTPS + HSTS, redaction formal de campos sensíveis nos logs e secrets fora do `appsettings.json`.

**Architecture:** Um `AuthenticationHandler` customizado valida o header `X-Api-Key` contra chaves em configuração. `[Authorize]` no controller; `/saude` e Swagger anônimos. HTTPS redirection + HSTS no pipeline. Serilog ganha uma destructuring policy que redige campos sensíveis. Connection string e API Keys migram para user-secrets/env.

**Tech Stack:** .NET 10, ASP.NET Core Authentication, Serilog.

**Pré-requisito:** Planos `fundacao-build-postgres` e `dominio-arquitetura` concluídos (inicializadores de camada, ProblemDetails ativos).

**Referências:** design doc seção 7; ADRs 018, 019, 020, 021, 022.

---

## Estrutura de Arquivos

- **Criar** `src/.../Api/Autenticacao/ApiKeyAuthenticationHandler.cs` — handler do esquema.
- **Criar** `src/.../Api/Autenticacao/ApiKeyOptions.cs` — opções (nome do header, chaves válidas).
- **Modificar** `src/.../Api/DependencyInjection.cs` — registra autenticação + Swagger com API Key.
- **Modificar** `src/.../Api/Program.cs` — pipeline: HTTPS, HSTS, Authentication, Authorization.
- **Modificar** `src/.../Api/Controllers/ContasController.cs` — `[Authorize]`.
- **Modificar** `src/.../Api/Program.cs` (health/swagger) — `[AllowAnonymous]` equivalente.
- **Criar** `src/.../Api/Logging/RedacaoDadosSensiveis.cs` — destructuring policy.
- **Modificar** `src/.../Api/Program.cs` (Serilog) — aplica a policy.
- **Modificar** `appsettings.json` — remove segredos, deixa estrutura.
- **Modificar** `tests/Api.Testes/AplicacaoFactory.cs` — injeta API Key de teste.
- **Modificar** `tests/Api.Testes/Contas/ContasIntegracaoTestes.cs` — envia header em todas as chamadas + testes 401.

---

## Task 1: Opções e handler de API Key

**Files:**
- Create: `src/MovimentacoesFinanceiras.Api/Autenticacao/ApiKeyOptions.cs`
- Create: `src/MovimentacoesFinanceiras.Api/Autenticacao/ApiKeyAuthenticationHandler.cs`

- [ ] **Step 1: Criar `ApiKeyOptions`**

`src/MovimentacoesFinanceiras.Api/Autenticacao/ApiKeyOptions.cs`:

```csharp
using Microsoft.AspNetCore.Authentication;

namespace MovimentacoesFinanceiras.Api.Autenticacao;

public class ApiKeyOptions : AuthenticationSchemeOptions
{
    public const string Esquema = "ApiKey";
    public const string NomeCabecalho = "X-Api-Key";

    public string[] ChavesValidas { get; set; } = [];
}
```

- [ ] **Step 2: Criar o handler**

`src/MovimentacoesFinanceiras.Api/Autenticacao/ApiKeyAuthenticationHandler.cs`:

```csharp
using System.Security.Claims;
using System.Text.Encodings.Web;
using Microsoft.AspNetCore.Authentication;
using Microsoft.Extensions.Options;

namespace MovimentacoesFinanceiras.Api.Autenticacao;

public class ApiKeyAuthenticationHandler(
    IOptionsMonitor<ApiKeyOptions> options,
    ILoggerFactory logger,
    UrlEncoder encoder)
    : AuthenticationHandler<ApiKeyOptions>(options, logger, encoder)
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue(ApiKeyOptions.NomeCabecalho, out var valores))
            return Task.FromResult(AuthenticateResult.NoResult());

        var chaveRecebida = valores.FirstOrDefault();

        if (string.IsNullOrWhiteSpace(chaveRecebida) || !Options.ChavesValidas.Contains(chaveRecebida))
            return Task.FromResult(AuthenticateResult.Fail("API Key inválida."));

        var identidade = new ClaimsIdentity(
            [new Claim(ClaimTypes.Name, "servico-autenticado")],
            ApiKeyOptions.Esquema);
        var ticket = new AuthenticationTicket(new ClaimsPrincipal(identidade), ApiKeyOptions.Esquema);

        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

- [ ] **Step 3: Build**

Run: `dotnet build -warnaserror`
Expected: `Build succeeded` (ainda não registrado — sem efeito em runtime).

- [ ] **Step 4: Commit**

```bash
git add src/MovimentacoesFinanceiras.Api/Autenticacao/
git commit -m "feat(api): handler de autenticação por API Key"
```

---

## Task 2: Registrar autenticação e proteger endpoints

**Files:**
- Modify: `src/MovimentacoesFinanceiras.Api/DependencyInjection.cs`
- Modify: `src/MovimentacoesFinanceiras.Api/Program.cs`
- Modify: `src/MovimentacoesFinanceiras.Api/Controllers/ContasController.cs`
- Modify: `src/MovimentacoesFinanceiras.Api/appsettings.json`
- Modify: `src/MovimentacoesFinanceiras.Api/appsettings.Development.json`

- [ ] **Step 1: Registrar o esquema em `AddApi()`**

`AddApi` precisa acessar `IConfiguration` para ler as chaves. Altere a assinatura e adicione o registro. Em `DependencyInjection.cs`, troque a assinatura de `AddApi(this IServiceCollection services)` por `AddApi(this IServiceCollection services, IConfiguration configuration)` e adicione, antes do `return services;`:

```csharp
        var chaves = configuration.GetSection("ApiKeys").Get<string[]>() ?? [];

        services.AddAuthentication(ApiKeyOptions.Esquema)
            .AddScheme<ApiKeyOptions, ApiKeyAuthenticationHandler>(
                ApiKeyOptions.Esquema,
                opcoes => opcoes.ChavesValidas = chaves);

        services.AddAuthorization();
```

Adicione os `using`:
```csharp
using Microsoft.Extensions.Configuration;
using MovimentacoesFinanceiras.Api.Autenticacao;
```

- [ ] **Step 2: Ensinar o Swagger a enviar a API Key**

Dentro de `AddSwaggerGen(opcoes => { ... })`, após o `SwaggerDoc`, adicione:

```csharp
            opcoes.AddSecurityDefinition(ApiKeyOptions.Esquema, new()
            {
                Name = ApiKeyOptions.NomeCabecalho,
                In = Microsoft.OpenApi.Models.ParameterLocation.Header,
                Type = Microsoft.OpenApi.Models.SecuritySchemeType.ApiKey,
                Description = "Informe a API Key no header X-Api-Key."
            });
            opcoes.AddSecurityRequirement(new()
            {
                {
                    new() { Reference = new() { Type = Microsoft.OpenApi.Models.ReferenceType.SecurityScheme, Id = ApiKeyOptions.Esquema } },
                    Array.Empty<string>()
                }
            });
```

- [ ] **Step 3: Atualizar a chamada em `Program.cs`**

Troque `.AddApi()` por `.AddApi(builder.Configuration)` no encadeamento. Adicione ao pipeline, **após** `UseSwaggerUI` e **antes** de `MapControllers`:

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

- [ ] **Step 4: Proteger o controller**

Em `ContasController.cs`, adicione `[Authorize]` na classe (e o `using Microsoft.AspNetCore.Authorization;`):

```csharp
[ApiController]
[Route("contas")]
[Produces("application/json")]
[Authorize]
public class ContasController(IMediator mediador) : ControllerBase
```

`/saude` é mapeado por `MapHealthChecks` (não passa por `[Authorize]` de controller) e o Swagger não exige auth — ambos permanecem acessíveis sem chave.

- [ ] **Step 5: Definir chave de dev (não em appsettings versionado)**

Em `appsettings.json`, adicione a estrutura vazia (sem valor real):

```json
"ApiKeys": [],
```

Em `appsettings.Development.json`, adicione uma chave de desenvolvimento (este arquivo é para conveniência local; a chave de produção vem de env/secrets):

```json
"ApiKeys": [ "dev-key-local-somente" ],
```

- [ ] **Step 6: Build**

Run: `dotnet build -warnaserror`
Expected: `Build succeeded`. (Os testes de integração vão quebrar sem header — corrigido na Task 3.)

- [ ] **Step 7: Commit**

```bash
git add src/
git commit -m "feat(api): autenticação por API Key com [Authorize] e Swagger security"
```

---

## Task 3: Ajustar testes de integração para API Key (TDD do 401)

**Files:**
- Modify: `tests/Api.Testes/AplicacaoFactory.cs`
- Modify: `tests/Api.Testes/Contas/ContasIntegracaoTestes.cs`

- [ ] **Step 1: Configurar a API Key de teste na factory**

Em `AplicacaoFactory.cs`, dentro de `ConfigureWebHost`, antes de `ConfigureTestServices`, force uma chave conhecida via configuração em memória:

```csharp
        builder.ConfigureAppConfiguration((_, config) =>
        {
            config.AddInMemoryCollection(new Dictionary<string, string?>
            {
                ["ApiKeys:0"] = "chave-de-teste"
            });
        });
```

Adicione o `using Microsoft.Extensions.Configuration;` no topo.

Exponha a chave como constante para os testes:

```csharp
    public const string ApiKeyTeste = "chave-de-teste";
```

- [ ] **Step 2: Enviar o header nas requisições existentes**

Em `ContasIntegracaoTestes.cs`, onde o `HttpClient` é criado (`factory.CreateClient()`), adicione o header padrão. Crie um helper no início da classe de teste:

```csharp
    private static HttpClient CriarClienteAutenticado(AplicacaoFactory factory)
    {
        var cliente = factory.CreateClient();
        cliente.DefaultRequestHeaders.Add(AplicacaoFactory.ApiKeyTeste is null ? "" : "X-Api-Key", AplicacaoFactory.ApiKeyTeste);
        return cliente;
    }
```

E substitua as chamadas `factory.CreateClient()` por `CriarClienteAutenticado(factory)` em todos os testes que hoje esperam 2xx/4xx de domínio. (Simplifique: se a criação do cliente estiver centralizada, ajuste só lá.)

- [ ] **Step 3: Escrever o teste que falha — 401 sem chave**

Adicione um teste novo:

```csharp
    [Fact]
    public async Task RequisicaoSemApiKey_Retorna401()
    {
        // Arrange
        await using var factory = new AplicacaoFactory();
        var cliente = factory.CreateClient(); // sem header X-Api-Key

        // Act
        var resposta = await cliente.PostAsJsonAsync("/contas", new { clienteId = Guid.NewGuid() });

        // Assert
        resposta.StatusCode.Should().Be(System.Net.HttpStatusCode.Unauthorized);
    }
```

- [ ] **Step 4: Rodar e ver o novo teste passar e os antigos também**

Run: `dotnet test tests/Api.Testes`
Expected: o teste 401 PASSA; os demais PASSAM porque agora enviam a chave. Se algum ainda falhar com 401, faltou trocar o cliente por `CriarClienteAutenticado`.

- [ ] **Step 5: Commit**

```bash
git add tests/
git commit -m "test(api): API Key nos testes de integração + caso 401"
```

---

## Task 4: HTTPS + HSTS

**Files:**
- Modify: `src/MovimentacoesFinanceiras.Api/Program.cs`

- [ ] **Step 1: Adicionar redirection e HSTS ao pipeline**

Em `Program.cs`, logo após `var app = builder.Build();` e o bloco de migrations, adicione (antes de `UseSwagger`):

```csharp
if (!app.Environment.IsDevelopment() && !app.Environment.IsEnvironment("Testing"))
{
    app.UseHsts();
}

app.UseHttpsRedirection();
```

> `UseHsts` fora de Development/Testing (HSTS em dev atrapalha; em Testing o `WebApplicationFactory` usa HTTP). `UseHttpsRedirection` sempre ativo — em Testing o cliente de teste ignora o redirect por usar o handler in-memory.

- [ ] **Step 2: Build + testes**

Run: `dotnet build -warnaserror && dotnet test`
Expected: build limpo; testes passam (o `WebApplicationFactory` opera sobre HTTP in-memory, sem quebrar com o redirect).

> Se os testes de integração passarem a falhar com 307/redirect, adicione `AllowAutoRedirect = false` não resolve — a causa seria `UseHttpsRedirection` ativo em Testing. Nesse caso, envolva `app.UseHttpsRedirection();` também na condição `!IsEnvironment("Testing")`.

- [ ] **Step 3: Commit**

```bash
git add src/
git commit -m "feat(api): HTTPS redirection e HSTS"
```

---

## Task 5: Redaction de dados sensíveis no Serilog

**Files:**
- Create: `src/MovimentacoesFinanceiras.Api/Logging/RedacaoDadosSensiveis.cs`
- Modify: `src/MovimentacoesFinanceiras.Api/Program.cs`

- [ ] **Step 1: Criar a destructuring policy**

`src/MovimentacoesFinanceiras.Api/Logging/RedacaoDadosSensiveis.cs`:

```csharp
using Serilog.Core;
using Serilog.Events;

namespace MovimentacoesFinanceiras.Api.Logging;

/// <summary>
/// Redige campos sensíveis ao serializar objetos nos logs, garantindo por
/// construção que valores monetários e PII nunca sejam persistidos em log.
/// </summary>
public class RedacaoDadosSensiveis : IDestructuringPolicy
{
    private static readonly HashSet<string> CamposSensiveis = new(StringComparer.OrdinalIgnoreCase)
    {
        "valor", "saldo", "saldoAtual", "quantia", "descricao"
    };

    public bool TryDestructure(object value, ILogEventPropertyValueFactory factory, out LogEventPropertyValue? result)
    {
        var propriedades = value.GetType().GetProperties();

        var estruturado = propriedades.Select(p =>
        {
            var conteudo = CamposSensiveis.Contains(p.Name)
                ? "[REDIGIDO]"
                : p.GetValue(value);
            return new LogEventProperty(p.Name, factory.CreatePropertyValue(conteudo, true));
        });

        result = new StructureValue(estruturado);
        return true;
    }
}
```

- [ ] **Step 2: Aplicar a policy no Serilog**

Em `Program.cs`, no `UseSerilog`, adicione `.Destructure.With<RedacaoDadosSensiveis>()`:

```csharp
builder.Host.UseSerilog((ctx, config) =>
    config
        .ReadFrom.Configuration(ctx.Configuration)
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .Destructure.With<MovimentacoesFinanceiras.Api.Logging.RedacaoDadosSensiveis>()
        .WriteTo.Console(new Serilog.Formatting.Json.JsonFormatter()));
```

- [ ] **Step 3: Build**

Run: `dotnet build -warnaserror`
Expected: `Build succeeded`.

- [ ] **Step 4: Verificação manual (opcional, mas recomendada)**

Suba a API (`docker compose up postgres -d` + `dotnet run --project src/MovimentacoesFinanceiras.Api`), faça uma movimentação e confirme no log JSON do console que campos como `valor`/`saldo` não aparecem em claro caso um objeto seja logado. (A garantia principal é a policy; logs de request do Serilog já não incluem o corpo.)

- [ ] **Step 5: Commit**

```bash
git add src/
git commit -m "feat(api): redaction de campos sensíveis nos logs (Serilog policy)"
```

---

## Task 6: Secrets fora do appsettings

**Files:**
- Modify: `src/MovimentacoesFinanceiras.Api/appsettings.json`
- Modify: `README.md`
- (Ação de ambiente) `dotnet user-secrets`

- [ ] **Step 1: Remover valores sensíveis do appsettings versionado**

Em `appsettings.json`, deixe a connection string como placeholder vazio e `ApiKeys` vazio:

```json
{
  "ConnectionStrings": {
    "Postgres": ""
  },
  "ApiKeys": [],
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

> A connection string real vem de: user-secrets (dev), env var `ConnectionStrings__Postgres` (docker-compose/prod). O fallback hardcoded em `AddInfraestrutura` cobre a execução local sem secrets. As API Keys vêm de env/secrets; `appsettings.Development.json` mantém a chave de dev para conveniência.

- [ ] **Step 2: Habilitar user-secrets no projeto Api**

Run:
```bash
dotnet user-secrets init --project src/MovimentacoesFinanceiras.Api
dotnet user-secrets set "ConnectionStrings:Postgres" "Host=localhost;Port=5432;Database=movimentacoes;Username=postgres;Password=postgres" --project src/MovimentacoesFinanceiras.Api
dotnet user-secrets set "ApiKeys:0" "dev-key-local-somente" --project src/MovimentacoesFinanceiras.Api
```
Expected: `Successfully saved ...`.

- [ ] **Step 3: Documentar no README**

Adicione uma seção "Configuração e segredos" ao README explicando: user-secrets em dev, env vars (`ConnectionStrings__Postgres`, `ApiKeys__0`) em Docker/prod, e o header `X-Api-Key` obrigatório nas chamadas. Atualize os exemplos `curl` para incluir `-H "X-Api-Key: <sua-chave>"`.

- [ ] **Step 4: Build + testes**

Run: `dotnet build -warnaserror && dotnet test`
Expected: build limpo; testes passam (a factory injeta config em memória, independente de user-secrets).

- [ ] **Step 5: Commit**

```bash
git add src/ README.md
git commit -m "chore(seguranca): segredos via user-secrets/env; appsettings sem valores sensíveis"
```

---

## Self-Review (preenchido)

**Spec coverage:**
- ADR-018 API Key vs JWT → Tasks 1, 2 ✓
- ADR-021 keys em configuração → Task 2 Step 1, Task 6 ✓
- ADR-019 HTTPS+HSTS → Task 4 ✓
- ADR-020 redaction → Task 5 ✓
- ADR-022 secrets em camadas → Task 6 ✓

**Placeholder scan:** sem TBD; todo passo com código/comando.

**Type consistency:** `ApiKeyOptions.Esquema`/`NomeCabecalho`/`ChavesValidas` consistentes entre options, handler e registro; `AplicacaoFactory.ApiKeyTeste` usada nos testes.

**Ponto de atenção 1:** `AddApi()` muda de assinatura (recebe `IConfiguration`) — a chamada em `Program.cs` (Task 2 Step 3) e qualquer outra referência devem ser atualizadas juntas.

**Ponto de atenção 2:** ordem do pipeline — `UseAuthentication`/`UseAuthorization` devem vir depois de `UseHttpsRedirection` e antes de `MapControllers`. O `TratadorDeExcecoesMiddleware` deve continuar cedo o suficiente para capturar exceções dos handlers, mas 401 é gerado pelo middleware de auth (não é exceção de domínio) — não requer alteração no tratador.
