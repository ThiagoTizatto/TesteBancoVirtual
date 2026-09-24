# Design: Sistema de Movimentações Financeiras

**Data:** 2026-09-22
**Autor:** Thiago
**Status:** Aprovado — revisado para 3 dias

---

## Índice

1. [Contexto e Problema de Negócio](#1-contexto-e-problema-de-negócio)
2. [Decisões Arquiteturais](#2-decisões-arquiteturais)
3. [Modelo de Domínio](#3-modelo-de-domínio)
4. [Estrutura de Projeto](#4-estrutura-de-projeto)
5. [Modelo de Dados](#5-modelo-de-dados)
6. [Contratos de API](#6-contratos-de-api)
7. [Consistência, Concorrência e Cenários de Falha](#7-consistência-concorrência-e-cenários-de-falha)
8. [Alta Demanda e Indisponibilidade Parcial](#8-alta-demanda-e-indisponibilidade-parcial)
9. [Proteção de Dados Sensíveis](#9-proteção-de-dados-sensíveis)
10. [Estratégia de Testes](#10-estratégia-de-testes)
11. [ADRs — Decisões de Arquitetura](#11-adrs--decisões-de-arquitetura)
12. [O que Faria Diferente com Mais Tempo](#12-o-que-faria-diferente-com-mais-tempo)

---

## 1. Contexto e Problema de Negócio

Um banco digital precisa de um sistema que:

- Registre movimentações financeiras de clientes (créditos e débitos) em suas contas
- Permita consultar a posição consolidada (saldo) de um cliente em um determinado momento no tempo
- Opere de forma confiável sob alta demanda, instabilidade e falhas parciais
- Garanta integridade total dos dados financeiros — qualquer inconsistência gera impacto direto ao cliente

### Invariantes do negócio

| Invariante | Descrição |
|---|---|
| Saldo nunca negativo | Débito só é permitido se `saldo_atual >= valor` |
| Movimentação é imutável | Lançamento registrado não pode ser alterado ou excluído |
| Rastreabilidade total | Todo crédito e débito deve ser auditável com timestamp |
| Idempotência | Retry de uma mesma requisição não pode gerar lançamento duplicado |

---

## 2. Decisões Arquiteturais

### 2.1 CQRS + Ledger Append-Only

A solução separa o **fluxo de escrita (Comando)** do **fluxo de leitura (Consulta)**:

**Escrita:** `POST /contas/{id}/movimentacoes`
- Registra um `Lancamento` imutável na tabela `lancamentos` (append-only)
- Atualiza o `saldo_atual` da `Conta` na mesma transação ACID
- Garante que nunca existe lançamento sem reflexo no saldo e vice-versa

**Leitura do saldo atual:** `GET /contas/{id}/saldo`
- Lê `saldo_atual` diretamente da tabela `contas` — O(1), sem agregação

**Leitura histórica (point-in-time):** `GET /contas/{id}/saldo?em=2025-01-15T12:00:00Z`
- Executa `SUM` sobre `lancamentos` filtrando por `criado_em <= :timestamp`
- Naturalmente correto porque o ledger é imutável — não há como o histórico mudar

### 2.2 Clean Architecture

O fluxo de dependências respeita estritamente:

```
Domínio ← Aplicação ← Infraestrutura ← Api
```

- **Domínio** não conhece banco, framework ou protocolo HTTP
- **Aplicação** orquestra casos de uso, não tem acesso direto ao banco
- **Infraestrutura** implementa interfaces definidas no Domínio
- **Api** é o ponto de entrada — converte HTTP em comandos/consultas

### 2.3 Resiliência com Polly

O `RegistrarMovimentacaoManipulador` usa `Polly` para absorver conflitos de concorrência internamente antes de desistir e retornar 409 ao cliente:

```csharp
var politicaRetentativa = Policy
    .Handle<DbUpdateConcurrencyException>()
    .WaitAndRetryAsync(3, tentativa =>
        TimeSpan.FromMilliseconds(50 * Math.Pow(2, tentativa)));

await politicaRetentativa.ExecuteAsync(() => repositorio.SalvarAsync(conta));
```

O cliente final raramente vê o conflito — o sistema o absorve. Apenas após 3 tentativas fracassadas o 409 é propagado.

### 2.4 Validação em Profundidade com FluentValidation

Três camadas independentes rejeitam dados inválidos antes de atingir o banco:

| Camada | Responsabilidade |
|---|---|
| **API** (FluentValidation) | Formato, tipos, tamanhos — borda do sistema |
| **Domínio** (`Dinheiro`, `Conta`) | Invariantes de negócio — saldo, valor > 0 |
| **Banco** (constraints EF Core) | Última linha de defesa — NOT NULL, CHECK |

### 2.5 Observabilidade com Serilog

Logging estruturado em JSON com enrichers de contexto. Cada request carrega um `correlacao_id` gerado no middleware e propagado por todas as camadas:

```json
{
  "nivel": "Information",
  "mensagem": "Movimentacao registrada",
  "conta_id": "3fa85f64...",
  "tipo": "Debito",
  "correlacao_id": "7c9e6679...",
  "duracao_ms": 12
}
```

Dados financeiros sensíveis (`valor`, `saldo_atual`, `descricao`) **nunca** aparecem nos logs.

### 2.6 DDD — Ubiquitous Language em Português

O código usa a língua do domínio bancário brasileiro. Nomes como `Conta`, `Lancamento`, `Dinheiro` e `TipoLancamento` eliminam a tradução mental entre o modelo de negócio e o código.

Ver ADR-004 para a justificativa e trade-offs dessa decisão.

### 2.7 Documentação Interativa com Swagger/OpenAPI

Swagger configurado em português com descrições, exemplos de request/response e agrupamento por contexto. O avaliador consegue testar todos os endpoints sem Postman ou curl.

---

## 3. Modelo de Domínio

### Agregado: `Conta`

```
Conta (Agregado Raiz)
├── Id: Guid
├── ClienteId: Guid
├── SaldoAtual: Dinheiro
├── VersaoLinha: byte[] (concurrency token)
├── CriadoEm: DateTime
└── Lancamentos: IReadOnlyList<Lancamento>
```

**Comportamentos:**
- `Creditar(Dinheiro valor, string descricao)` — adiciona ao saldo, registra lançamento
- `Debitar(Dinheiro valor, string descricao)` — valida saldo, subtrai, registra lançamento

**Invariante protegida:**
```csharp
public void Debitar(Dinheiro valor, string descricao)
{
    if (valor > SaldoAtual)
        throw new SaldoInsuficienteException(SaldoAtual, valor);

    SaldoAtual -= valor;
    _lancamentos.Add(Lancamento.CriarDebito(Id, valor, descricao));
}
```

### Entidade: `Lancamento`

```
Lancamento (Entidade)
├── Id: Guid
├── ContaId: Guid
├── Valor: Dinheiro        ← sempre positivo; sinal vem do Tipo
├── Tipo: TipoLancamento   ← Credito | Debito
├── Descricao: string
└── CriadoEm: DateTime
```

### Objeto de Valor: `Dinheiro`

```
Dinheiro (Objeto de Valor)
└── Quantia: decimal       ← sempre > 0; validado no construtor
```

Encapsula a validação de que valores monetários nunca são negativos ou zero. Comparação e operações aritméticas são sobrecarregadas para evitar manipulação direta de `decimal`.

### Enum: `TipoLancamento`

```csharp
public enum TipoLancamento
{
    Credito,
    Debito
}
```

---

## 4. Estrutura de Projeto

```
desafio-banco/
├── src/
│   ├── MovimentacoesFinanceiras.Dominio/
│   │   └── Contas/
│   │       ├── Conta.cs
│   │       ├── Lancamento.cs
│   │       ├── Dinheiro.cs
│   │       ├── TipoLancamento.cs
│   │       ├── IContaRepositorio.cs
│   │       └── Excecoes/
│   │           ├── SaldoInsuficienteException.cs
│   │           └── ContaNaoEncontradaException.cs
│   │
│   ├── MovimentacoesFinanceiras.Aplicacao/
│   │   └── Contas/
│   │       ├── Comandos/
│   │       │   └── RegistrarMovimentacao/
│   │       │       ├── RegistrarMovimentacaoComando.cs
│   │       │       ├── RegistrarMovimentacaoManipulador.cs
│   │       │       └── RegistrarMovimentacaoValidador.cs
│   │       └── Consultas/
│   │           ├── ConsultarSaldo/
│   │           │   ├── ConsultarSaldoConsulta.cs
│   │           │   └── ConsultarSaldoManipulador.cs
│   │           └── ConsultarSaldoEm/
│   │               ├── ConsultarSaldoEmConsulta.cs
│   │               └── ConsultarSaldoEmManipulador.cs
│   │
│   ├── MovimentacoesFinanceiras.Infraestrutura/
│   │   └── Persistencia/
│   │       ├── ContextoBancoDados.cs
│   │       ├── Repositorios/
│   │       │   └── ContaRepositorio.cs
│   │       └── Configuracoes/
│   │           ├── ContaConfiguracao.cs
│   │           └── LancamentoConfiguracao.cs
│   │
│   └── MovimentacoesFinanceiras.Api/
│       ├── Controladores/
│       │   └── ContasControlador.cs
│       ├── Middlewares/
│       │   └── TratadorDeExcecoesMiddleware.cs
│       ├── Program.cs
│       └── appsettings.json
│
├── tests/
│   ├── Dominio.Testes/
│   │   └── Contas/
│   │       ├── ContaTestes.cs
│   │       └── DinheiroTestes.cs
│   ├── Aplicacao.Testes/
│   │   └── Contas/
│   │       └── RegistrarMovimentacaoTestes.cs
│   └── Api.Testes/
│       └── Contas/
│           └── ContasIntegracaoTestes.cs
│
├── docs/
│   └── superpowers/
│       └── specs/
│           └── 2026-09-22-movimentacoes-financeiras-design.md
│
└── README.md
```

---

## 5. Modelo de Dados

### Tabela: `contas`

| Campo | Tipo | Restrição | Descrição |
|---|---|---|---|
| `id` | GUID | PK | Identificador opaco — nunca sequencial |
| `cliente_id` | GUID | NOT NULL, INDEX | Identificador do cliente dono da conta |
| `saldo_atual` | DECIMAL(18,2) | NOT NULL, DEFAULT 0 | Snapshot do saldo — atualizado atomicamente com o lançamento |
| `versao_linha` | BLOB | NOT NULL | Token de concorrência otimista (EF Core rowversion) |
| `criado_em` | DATETIME | NOT NULL | Timestamp de criação da conta |

### Tabela: `lancamentos` — append-only

| Campo | Tipo | Restrição | Descrição |
|---|---|---|---|
| `id` | GUID | PK | Identificador do lançamento |
| `conta_id` | GUID | FK → contas, NOT NULL | Conta à qual pertence |
| `valor` | DECIMAL(18,2) | NOT NULL, > 0 | Sempre positivo; o sinal vem do `tipo` |
| `tipo` | VARCHAR(10) | NOT NULL | `Credito` ou `Debito` |
| `descricao` | VARCHAR(255) | NULL | Descrição legível da movimentação |
| `chave_idempotencia` | VARCHAR(64) | UNIQUE, NULL | Previne lançamentos duplicados por retry |
| `criado_em` | DATETIME | NOT NULL | Timestamp do lançamento |

**Índices:**
- `lancamentos(conta_id, criado_em)` — otimiza queries de extrato e saldo histórico

**Query de saldo em ponto no tempo:**
```sql
SELECT
    SUM(CASE tipo WHEN 'Credito' THEN valor ELSE -valor END) AS saldo
FROM lancamentos
WHERE conta_id = :conta_id
  AND criado_em <= :data_referencia
```

---

## 6. Contratos de API

Todas as respostas de erro seguem **RFC 7807 — Problem Details** em português.

### `POST /contas`
Cria uma nova conta para um cliente.

**Corpo da requisição:**
```json
{ "clienteId": "3fa85f64-5717-4562-b3fc-2c963f66afa6" }
```

**Resposta 201 Created:**
```json
{
  "id": "...",
  "clienteId": "...",
  "saldoAtual": 0.00,
  "criadoEm": "2026-09-22T10:00:00Z"
}
```

---

### `POST /contas/{id}/movimentacoes`
Registra um crédito ou débito.

**Headers opcionais:**
- `Idempotency-Key: <uuid>` — garante que retries não gerem lançamentos duplicados

**Corpo da requisição:**
```json
{
  "tipo": "Credito",
  "valor": 100.00,
  "descricao": "Depósito inicial"
}
```

**Resposta 201 Created:**
```json
{
  "id": "...",
  "tipo": "Credito",
  "valor": 100.00,
  "descricao": "Depósito inicial",
  "criadoEm": "2026-09-22T10:05:00Z"
}
```

**Erros:**
- `422` — saldo insuficiente para débito
- `400` — valor zero ou negativo
- `404` — conta não encontrada
- `409` — conflito de concorrência (retry recomendado)

---

### `GET /contas/{id}/saldo`
Retorna o saldo atual da conta — leitura O(1) do snapshot.

**Resposta 200 OK:**
```json
{
  "saldo": 250.00,
  "consultadoEm": "2026-09-22T10:10:00Z"
}
```

---

### `GET /contas/{id}/saldo?em=2025-01-15T12:00:00Z`
Retorna o saldo consolidado num ponto específico no tempo.

**Resposta 200 OK:**
```json
{
  "saldo": 100.00,
  "referenciaEm": "2025-01-15T12:00:00Z"
}
```

---

### `GET /contas/{id}/movimentacoes`
Retorna o extrato paginado da conta.

**Query params:** `?pagina=1&tamanhoPagina=20`

**Resposta 200 OK:**
```json
{
  "itens": [
    {
      "id": "...",
      "tipo": "Credito",
      "valor": 100.00,
      "descricao": "Depósito inicial",
      "criadoEm": "2026-09-22T10:05:00Z"
    }
  ],
  "pagina": 1,
  "tamanhoPagina": 20,
  "total": 1
}
```

---

### Formato de erros (RFC 7807)

```json
{
  "tipo": "saldo-insuficiente",
  "titulo": "Saldo insuficiente para realizar o débito",
  "status": 422,
  "detalhe": "O saldo disponível é R$ 50,00 e o débito solicitado é R$ 100,00"
}
```

---

## 7. Consistência, Concorrência e Cenários de Falha

### Consistência transacional

A operação de registro de movimentação é **atomicamente consistente**:

1. A `Conta` valida o invariante de saldo (regra de domínio)
2. O `Lancamento` é inserido na tabela `lancamentos`
3. O `saldo_atual` da `Conta` é atualizado
4. A transação é commitada — as três operações ocorrem juntas ou nenhuma ocorre

Se o processo cair após o passo 2 mas antes do passo 4, o rollback automático desfaz tudo. Nunca há estado intermediário visível.

### Concorrência otimista com retry automático

A tabela `contas` possui o campo `versao_linha` gerenciado automaticamente pelo EF Core. O fluxo com Polly absorve conflitos internamente:

1. A aplicação lê `Conta` incluindo `versao_linha`
2. Ao fazer o UPDATE, o EF Core adiciona `WHERE versao_linha = :versao_lida`
3. Se outro processo já atualizou a linha → `DbUpdateConcurrencyException`
4. **Polly intercepta** e retenta até 3 vezes com backoff exponencial (50ms, 100ms, 200ms)
5. Em cada retentativa, a `Conta` é relida com o `versao_linha` atualizado
6. Apenas se todas as tentativas falharem o 409 Conflict é retornado ao cliente

### Idempotência

O campo `chave_idempotencia` (unique) na tabela `lancamentos` garante que:
- O cliente pode enviar o header `Idempotency-Key: <uuid>` com a requisição
- Se um lançamento com aquela chave já existe, a API retorna os dados do lançamento original com 200 (sem regravar)
- Elimina o risco de débito duplo por retry após timeout de rede

### Tabela de cenários de falha

| Cenário | Comportamento |
|---|---|
| Falha no meio da transação | Rollback automático — nenhum efeito visível |
| Timeout de rede (cliente retenta) | `Idempotency-Key` previne lançamento duplicado |
| Conflito de concorrência | 409 Conflict — cliente retenta com dados atualizados |
| Débito maior que saldo | 422 — invariante protegida no Agregado |
| Banco indisponível | 503 com header `Retry-After` |
| Conta inexistente | 404 — verificado antes de qualquer operação |

---

## 8. Alta Demanda e Indisponibilidade Parcial

### Capacidade atual (SQLite — desenvolvimento)

SQLite serializa escritas. Adequado para o desafio técnico e demonstração. Para produção, a troca é cirúrgica: apenas a string de conexão e o provider do EF Core mudam — o domínio e a aplicação são intocados.

### Escalabilidade de leitura

- **Saldo atual** — O(1). Candidato a cache Redis com TTL curto (ex: 5 segundos) em produção
- **Saldo histórico** — range scan com index `(conta_id, criado_em)` + eventual snapshot periódico para contas com histórico longo
- **Extrato** — paginado, index garante performance mesmo com muitos lançamentos

### Padrões de resiliência (evolução natural)

| Padrão | Propósito | Estado |
|---|---|---|
| Health check detalhado (`GET /saude`) | Reporta status do banco + latência em JSON | Implementado |
| Retry com backoff exponencial (Polly) | Absorve conflitos de concorrência internamente | Implementado |
| Correlation ID middleware | Rastreia requisições por todos os logs | Implementado |
| Graceful shutdown | Termina requisições em andamento antes de encerrar | Implementado via `IHostApplicationLifetime` |
| Circuit breaker | Para de tentar banco após N falhas consecutivas | Evolução futura |
| Réplicas de leitura | Separar tráfego de leitura/escrita em produção | Evolução futura (troca de connection string) |
| Fila de movimentações | Absorver picos via Kafka/RabbitMQ | Evolução futura |

---

## 9. Proteção de Dados Sensíveis

### Identificadores opacos

`id` e `cliente_id` são GUIDs gerados aleatoriamente. Nunca há IDs sequenciais expostos nas URLs — previne enumeração e scraping de dados.

### Autorização por ownership

> **Estado atual:** autenticação não está implementada nesta versão do desafio — ver Seção 12.

Em produção, antes de qualquer operação o controller validaria que o `ClienteId` do token JWT corresponde ao dono da conta solicitada. Um cliente autenticado não poderia acessar dados de outro.

### Mascaramento em logs

Logs registram `conta_id` e `tipo` da operação, mas **nunca `valor`, `saldo_atual` ou `descricao`**. Um log que expõe movimentações financeiras é um incidente de privacidade e pode violar a LGPD.

### Transporte seguro

HTTPS obrigatório. Em produção: TLS 1.2+ com renovação automática de certificado.

---

## 10. Estratégia de Testes

### Testes de Domínio (`Dominio.Testes`)

Testam as regras de negócio puras — zero dependência de banco, HTTP ou framework.

| Cenário | Resultado esperado |
|---|---|
| Creditar valor válido | Saldo aumenta, lançamento registrado |
| Debitar com saldo suficiente | Saldo diminui, lançamento registrado |
| Debitar mais do que o saldo | `SaldoInsuficienteException` |
| Criar `Dinheiro` com valor zero | `ArgumentException` |
| Criar `Dinheiro` com valor negativo | `ArgumentException` |
| `TipoLancamento` correto gravado | `Credito` vs `Debito` validados |

### Testes de Integração (`Api.Testes`)

Usam `WebApplicationFactory` com SQLite em memória. Nenhum Docker, nenhum serviço externo.

| Cenário | Status HTTP esperado |
|---|---|
| Registrar crédito válido | 201 Created |
| Registrar débito com saldo suficiente | 201 Created |
| Registrar débito sem saldo | 422 Unprocessable |
| Consultar saldo atual | 200 OK com valor correto |
| Consultar saldo em data passada | 200 OK com valor histórico correto |
| Consultar saldo em data futura | 200 OK com saldo atual (nenhum lançamento futuro) |
| Registrar movimentação com valor zero | 400 Bad Request |
| Consultar conta inexistente | 404 Not Found |
| Retry com mesma `Idempotency-Key` | 200 OK sem lançamento duplicado |

### Teste de Concorrência (`Api.Testes`)

Prova que a solução funciona corretamente sob pressão real:

```csharp
[Fact]
public async Task DebitosSimultaneos_NaoDevemGerarSaldoNegativo()
{
    // Conta com R$ 100; 10 débitos de R$ 20 disparados simultaneamente
    // Apenas 5 devem passar; saldo final deve ser exatamente R$ 0
    var tarefas = Enumerable.Range(0, 10)
        .Select(_ => cliente.PostAsync($"/contas/{contaId}/movimentacoes", debito20));

    var respostas = await Task.WhenAll(tarefas);

    var sucessos = respostas.Count(r => r.StatusCode == HttpStatusCode.Created);
    var saldoFinal = await ObterSaldo(contaId);

    Assert.Equal(5, sucessos);
    Assert.Equal(0m, saldoFinal);
}
```

### Teste de Integridade do Ledger (`Api.Testes`)

Garante que o snapshot `saldo_atual` nunca diverge do ledger após alta carga:

```csharp
[Fact]
public async Task AposMultiplasOperacoes_SaldoAtualDeveSerConsistenteComLedger()
{
    // 500 créditos de R$ 10 + 300 débitos de R$ 10 = saldo esperado R$ 2.000
    await RealizarOperacoesEmLote(creditos: 500, debitos: 300, valor: 10m);

    var saldoSnapshot = await ObterSaldoAtual(contaId);
    var saldoLedger = await RecalcularPeloLedger(contaId);

    Assert.Equal(saldoLedger, saldoSnapshot);
}
```

### Como rodar

```bash
dotnet test
```

---

## 11. ADRs — Decisões de Arquitetura

### ADR-001 — Ledger append-only em vez de atualizar saldo diretamente

**Contexto:** A abordagem mais simples seria ter apenas a tabela `contas` com um campo `saldo` atualizado a cada movimentação.

**Decisão:** Usar ledger append-only — cada movimentação gera um novo registro em `lancamentos`, nunca modificado.

**Consequências:**
- Rastreabilidade completa e auditável por design
- Saldo em ponto no tempo é trivial: `SUM` dos lançamentos até a data
- Risco zero de perda de histórico por bug de atualização
- Custo: queries históricas fazem agregação (mitigado com index e snapshot futuro)

**Alternativa rejeitada:** Atualizar apenas o campo `saldo` sem guardar o histórico tornaria impossível responder "qual era o saldo em 15/01/2025" sem um mecanismo adicional.

---

### ADR-002 — Concorrência otimista em vez de pessimistic lock

**Contexto:** Dois débitos simultâneos na mesma conta podem gerar saldo negativo se não houver controle de concorrência.

**Decisão:** Optimistic concurrency via `versao_linha` (EF Core rowversion).

**Consequências:**
- Sem bloqueio de leituras concorrentes
- Escala horizontalmente sem coordenação entre instâncias
- Conflitos são raros em contas individuais — custo do retry é baixo
- Em caso de conflito, retorna 409 e o cliente retenta

**Alternativa rejeitada:** `SELECT FOR UPDATE` (pessimistic) bloqueia a linha até o commit, reduz throughput e cria risco de deadlock em workloads complexos.

---

### ADR-003 — SQLite para desenvolvimento

**Contexto:** O desafio requer que a aplicação rode localmente com `dotnet run`, sem dependências externas.

**Decisão:** SQLite com EF Core. A string de conexão é configurada via `appsettings.json`.

**Consequências:**
- Zero fricção para o avaliador rodar
- SQLite serializa escritas — não adequado para produção com alto volume
- A troca para PostgreSQL em produção exige apenas mudar provider e connection string — o domínio é intocado

**Como trocar para produção:**
```csharp
// Program.cs — trocar:
options.UseSqlite(connectionString);
// por:
options.UseNpgsql(connectionString);
```

---

### ADR-005 — Polly para retry de concorrência

**Contexto:** Conflitos de concorrência otimista são transitórios — a causa desaparece após o primeiro commit vencer. Retornar 409 imediatamente transfere a responsabilidade de retry para o cliente.

**Decisão:** Usar Polly para retentar internamente até 3 vezes com backoff exponencial antes de propagar o 409.

**Consequências:**
- O cliente raramente vê o 409 — melhora a experiência mesmo sob carga
- Backoff exponencial evita thundering herd (todos retentando ao mesmo tempo)
- Custo: até 3 leituras adicionais do banco por conflito — aceitável dado que conflitos são raros

**Alternativa rejeitada:** Retornar 409 imediatamente e deixar o cliente retentar. Funciona, mas transfere complexidade para o chamador sem necessidade.

---

### ADR-006 — Serilog com logging estruturado

**Contexto:** Logs de texto (`$"Movimentação {id} registrada"`) não são pesquisáveis por campo em produção.

**Decisão:** Serilog com output JSON e enrichers de contexto (`correlacao_id`, `conta_id`, `tipo`).

**Consequências:**
- Logs são indexáveis por qualquer campo sem parsing de texto
- `correlacao_id` permite rastrear uma requisição específica de ponta a ponta
- Campos sensíveis (`valor`, `saldo_atual`) são explicitamente excluídos dos enrichers

---

### ADR-004 — Ubiquitous Language em português

**Contexto:** O sistema é para um banco digital brasileiro. O domínio de negócio é naturalmente expresso em português.

**Decisão:** Nomes de classes, métodos, variáveis, tabelas e campos em português.

**Consequências:**
- Elimina tradução mental entre modelo de negócio e código
- Nomes como `Conta`, `Lancamento`, `SaldoInsuficienteException` são autoexplicativos para qualquer analista de negócio brasileiro
- Trade-off: keywords do C# e .NET são em inglês — convivência inevitável

**Alternativa considerada:** Código em inglês, documentação em português. É a prática mais comum no mercado brasileiro, mas neste contexto o alinhamento com a Linguagem Ubíqua do domínio foi priorizado.

---

## 12. O que Faria Diferente com Mais Tempo

### Alta prioridade

- **Autenticação e autorização** — JWT com validação de ownership por `ClienteId`. Atualmente o sistema não tem autenticação implementada; os endpoints são abertos.
- **Snapshot periódico de saldo** — para contas com histórico de anos, o `SUM` pode se tornar lento. Um job agendado que grava snapshots periódicos resolve com O(1) para qualquer data.
- **Paginação por cursor** — o extrato usa offset pagination; cursor-based escala melhor para históricos longos.

### Baixa prioridade (produção real)

- **Circuit breaker** — Polly `CircuitBreakerPolicy` para parar de tentar o banco após N falhas consecutivas
- **Fila de movimentações** — Kafka ou RabbitMQ para absorver picos e desacoplar o registro do processamento
- **Réplicas de leitura** — separar queries de consulta das de escrita no PostgreSQL
- **Soft-delete de contas** — encerramento de conta sem perda de histórico
- **Relatórios de extrato** — exportação em PDF/CSV para fins regulatórios
