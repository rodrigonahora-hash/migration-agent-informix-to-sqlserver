---
name: regras-conversao-informix
description: Regras técnicas de conversão de código Informix para SQL Server (T-SQL) em aplicações C#/.NET legadas. Use ao analisar, converter ou revisar qualquer trecho de SQL Informix, classe DAO, ADO.NET, iBATIS/MyBatis ou Entity Framework que esteja sendo migrado de Informix para SQL Server — inclui mapeamento de funções, tipos, schemas, recuperação de identidade e leitura de DataReader.
---

# Regras Técnicas de Conversão Informix → SQL Server

Conhecimento de referência para converter código Informix (C#/.NET legado) para SQL Server. Aplique estas transformações ao identificar incompatibilidades. Prefira sempre a menor alteração possível que torne o código compatível, sem alterar regra de negócio.

## 1. Conexão com banco

- provider Informix / `OleDbConnection`, `IfxConnection`, `OnsConnection` → `SqlConnection`;
- strings de conexão e providers no `web.config` / `app.config` → ajustar para SQL Server;
- schema `informix` → schema real do SQL Server (`dbo` ou outro — validar);
- comandos de abertura de conexão incompatíveis → padrão SQL Server.

## 2. Objetos ADO.NET

- `OleDbCommand` → `SqlCommand`;
- `OleDbDataReader` / `OnsDataReader` → `SqlDataReader`;
- `OleDbDataAdapter` / `OnsDataAdapter` → `SqlDataAdapter`;
- parâmetros posicionais `?` → parâmetros nomeados `@param`;
- tipos `OleDbType` / Informix → `SqlDbType` compatível.

## 3. Funções e sintaxe SQL Informix incompatíveis

- `NVL(a, b)` → `ISNULL(a, b)` ou `COALESCE(a, b)`;
- `EXTEND(campo, ...)` → `CAST(campo AS DATE)` ou `CAST(campo AS DATETIME)`;
- `TODAY` → `CAST(GETDATE() AS DATE)`;
- `CURRENT` → `GETDATE()` ou `SYSDATETIME()`;
- `dbinfo('sqlca.sqlerrd1')` → `SCOPE_IDENTITY()` ou `OUTPUT INSERTED.Id`;
- concatenação `||` → `+` ou `CONCAT()`;
- `MATCHES(campo, 'A*')` → `campo LIKE 'A%'` (curinga `*` do Informix → `%`); se `MATCHES` retorna valor no SELECT, usar `CASE WHEN ... LIKE ... THEN 1 ELSE 0 END`;
- `SKIP n FIRST m` → `OFFSET n ROWS FETCH NEXT m ROWS ONLY` (exige `ORDER BY`); `FIRST m` sozinho → `TOP m`;
- `OUTER` (Informix) → `LEFT JOIN`;
- `TRIM()` → validar compatibilidade na versão do SQL Server;
- conversões de data específicas do Informix → equivalente T-SQL.

## 4. Tipos de dados

Validar diferenças: `DATE`, `DATETIME`, `SMALLDATETIME`, `DECIMAL`, `NUMERIC`, `CHAR`, `VARCHAR`, `TEXT`, `LVARCHAR`, e booleanos representados por `char`/`bit`/`smallint`. Atenção especial a comparações, inserts e updates com datas.

**Armadilha silenciosa de flags (`BIT` vs `CHAR`):** comparações de flag com literal de caractere (`= 'S'`, `= 'N'`) não geram erro de sintaxe, mas falham silenciosamente se a coluna virou `BIT` (armazena `0`/`1`) na migração. Registre em Pendências a confirmação do tipo real: se permaneceu `CHAR(1)`/`VARCHAR(1)`, `= 'S'` é válido; se virou `BIT`, adaptar para `= 1`/`= 0` conforme a semântica original.

## 5. Schemas

Referências a `informix.nome_tabela` → `dbo.nome_tabela` ou outro schema do SQL Server. Não assuma o schema: se não informado, indique a necessidade de validação contra o DDL real.

## 6. Identidade e chaves geradas

Recuperação do último ID inserido: `dbinfo('sqlca.sqlerrd1')` → `SELECT SCOPE_IDENTITY()` ou `OUTPUT INSERTED.Id`. Respeitar o padrão já usado no projeto.

**Atenção crítica ao tipo de retorno na leitura:** `SCOPE_IDENTITY()` retorna `numeric(38,0)`, não `int` — ler com `reader.GetInt32(0)` lança `InvalidCastException`. Leia com `Convert.ToInt32(reader.GetValue(0))`. Alternativa mais robusta: `OUTPUT INSERTED.Id` retorna o tipo real da coluna (geralmente `int`) e permite `GetInt32(0)` direto, mas exige reestruturar o comando.

```csharp
// insert ...; SELECT SCOPE_IDENTITY();
if (reader.Read())
    return Convert.ToInt32(reader.GetValue(0));
return -1; // preservar o contrato de erro já existente no método
```

## 7. Datas

Avalie: comparação entre `DATE` e `DATETIME`; formatação manual; strings `yyyy-MM-dd` e `yyyy-MM-dd HH:mm:ss`; `Convert.ToDateTime`; cultura/região; uso de parâmetros em vez de concatenação de datas em SQL. Não sugerir refatoração ampla se a correção pontual resolver a compatibilidade.

## 8. iBATIS / MyBatis (XMLs de mapeamento)

Verificar: SQL Informix embutido; resultMaps incompatíveis com tipos SQL Server; parâmetros; selects dinâmicos; joins antigos; funções Informix; schemas; includes; procedures chamadas pelo mapeamento.

## 9. Entity Framework legado (EDMX / DbContext)

Verificar: provider antigo; connection string; schema; tipos incompatíveis; migrations não aplicáveis; mapeamentos físicos; diferenças Informix vs SQL Server. Não recriar o modelo inteiro, exceto se indispensável para a migração.

## 10. Leitura de resultados no DataReader (padrão crítico do projeto)

O driver Informix frequentemente retornava valores numéricos como **texto**. No SQL Server os tipos vêm corretos e fortemente tipados. Padrão recorrente que gera `InvalidCastException` / `InvalidOperationException` em runtime:

- `reader.GetString(N)` em **coluna numérica** → `reader.GetInt32(N)` / `GetDecimal(N)` / `GetInt64(N)` conforme o tipo real;
- remover gambiarras de limpeza como `.ToString().Replace(".0", "")` (existiam só porque o Informix retornava números como texto decimal, ex.: `"42.0"`);
- `count(*)` retorna `Int32` — ler com `GetInt32(0)`, nunca `GetString(0)`;
- atenção ao tipo retornado por **funções**: `SCOPE_IDENTITY()` retorna `numeric(38,0)` (usar `Convert.ToInt32(reader.GetValue(0))`);
- proteger a leitura com `if (reader.Read())` antes de acessar colunas, tratando o caso de nenhum registro — o valor de retorno para "sem registro" deve respeitar o contrato existente do método (`0`, `-1` etc.), sem alterá-lo.

```csharp
// Antes (Informix): reader.Read();
//   return Convert.ToInt32(reader.GetString(0).ToString().Replace(".0", ""));
// Depois (SQL Server):
if (reader.Read())
    return reader.GetInt32(0);
return 0; // ajustar ao valor esperado pela regra de negócio do método
```
