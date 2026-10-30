```c#
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <AssemblyName>MyApp</AssemblyName>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Npgsql" Version="8.0.5" />
    <PackageReference Include="Dapper" Version="2.1.35" />
  </ItemGroup>
</Project>

CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    owner VARCHAR(100) NOT NULL,
    balance NUMERIC(12,2) NOT NULL DEFAULT 0 CHECK (balance >= 0)
);

INSERT INTO accounts (owner, balance)
VALUES ('Alice', 1000.00), ('Bob', 500.00);

using System.Data;
using Dapper;
using Npgsql;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

string cs = Environment.GetEnvironmentVariable("CONN_STRING")!;

app.MapPost("/transfer", async (Transfer t) =>
{
    if (t.FromId == t.ToId)
        return Results.BadRequest("FromId and ToId must differ.");
    if (t.Amount <= 0)
        return Results.BadRequest("Amount must be positive.");

    await using var conn = new NpgsqlConnection(cs);
    await conn.OpenAsync();

    await using var tx = await conn.BeginTransactionAsync(IsolationLevel.ReadCommitted);

    try
    {
        await using var debit = new NpgsqlCommand(
            @"UPDATE accounts
              SET balance = balance - @amt
              WHERE id = @from AND balance >= @amt", conn, tx);
        debit.Parameters.Add(new NpgsqlParameter("amt", t.Amount));
        debit.Parameters.Add(new NpgsqlParameter("from", t.FromId));

        int debited = await debit.ExecuteNonQueryAsync();
        if (debited == 0)
        {
            await tx.RollbackAsync();
            return Results.BadRequest("Insufficient funds or source account not found.");
        }

        await using var lockTo = new NpgsqlCommand(
            "SELECT id FROM accounts WHERE id = @to FOR UPDATE", conn, tx);
        lockTo.Parameters.Add(new NpgsqlParameter("to", t.ToId));

        var toObj = await lockTo.ExecuteScalarAsync();
        if (toObj is null)
        {
            await tx.RollbackAsync();
            return Results.NotFound($"Account {t.ToId} not found.");
        }

        await using var credit = new NpgsqlCommand(
            "UPDATE accounts SET balance = balance + @amt WHERE id = @to", conn, tx);
        credit.Parameters.Add(new NpgsqlParameter("amt", t.Amount));
        credit.Parameters.Add(new NpgsqlParameter("to", t.ToId));
        await credit.ExecuteNonQueryAsync();

        await tx.CommitAsync();
        return Results.Ok(new { from = t.FromId, to = t.ToId, amount = t.Amount });
    }
    catch
    {
        await tx.RollbackAsync();
        throw;
    }
});

app.MapGet("/accounts", async () =>
{
    await using var conn = new NpgsqlConnection(cs);
    await conn.OpenAsync();

    var rows = await conn.QueryAsync<Account>(
        "SELECT id, owner, balance FROM accounts ORDER BY id");

    return Results.Ok(rows);
});

app.MapGet("/accounts/report", () =>
{
    using var conn = new NpgsqlConnection(cs);
    conn.Open();

    using var adapter = new NpgsqlDataAdapter(
        "SELECT id, owner, balance FROM accounts ORDER BY id", conn);

    var table = new DataTable("accounts");
    adapter.Fill(table);

    var ds = new DataSet("report");
    ds.Tables.Add(table);

    var report = ds.Tables["accounts"]!.AsEnumerable()
        .Select(r => new
        {
            Id      = r.Field<int>("id"),
            Owner   = r.Field<string>("owner"),
            Balance = r.Field<decimal>("balance")
        })
        .ToList();

    return Results.Ok(new
    {
        total = report.Sum(x => x.Balance),
        count = report.Count,
        accounts = report
    });
});

app.Run();

record Transfer(int FromId, int ToId, decimal Amount);
record Account(int Id, string Owner, decimal Balance);

version: '3.8'
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: shopdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports: ["5432:5432"]
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d shopdb"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s
  app:
    build: .
    environment:
      CONN_STRING: "Host=db;Port=5432;Database=shopdb;Username=postgres;Password=postgres;Pooling=true;Minimum Pool Size=1;Maximum Pool Size=20"
    ports: ["8080:80"]
    depends_on:
      db:
        condition: service_healthy

Список счетов
curl -s localhost:8080/accounts

Перевод
curl -s -X POST localhost:8080/transfer \
  -H 'Content-Type: application/json' \
  -d '{"fromId":1,"toId":2,"amount":200}'

Недостаточно средств
curl -s -X POST localhost:8080/transfer \
  -H 'Content-Type: application/json' \
  -d '{"fromId":2,"toId":1,"amount":999999}'
Отчёт
curl -s localhost:8080/accounts/report

```
