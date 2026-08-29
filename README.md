# Postgres

[![CI](https://github.com/hpopp/postgres-koja/actions/workflows/ci.yml/badge.svg)](https://github.com/hpopp/postgres-koja/actions/workflows/ci.yml)
[![Last Updated](https://img.shields.io/github/last-commit/hpopp/postgres-koja.svg)](https://github.com/hpopp/postgres-koja/commits/main)

A PostgreSQL driver for [Koja](https://github.com/koja-lang/koja). It speaks
the v3 wire protocol over TCP, with no C dependencies beyond the Koja stdlib.

## Features

- Trust, cleartext password, and SCRAM-SHA-256 authentication
- Simple query protocol (`query`) for plain SQL
- Extended query protocol (`execute`) with typed `$1`-style parameters
- Typed results. Rows decode to `Postgres.Value` from the column types the server reports
- Per-connection prepared statement cache with LRU eviction
- Structured server errors carrying severity, SQLSTATE code, and message

## Installation

Add the package as a path dependency in your `koja.toml`:

```toml
[dependencies]
Postgres = { path = "../postgres" }
```

## Usage

```koja
alias Postgres.Config
alias Postgres.Connection
alias Postgres.Value

config = Config.new("127.0.0.1", 5432, "app_user", "app_db")
  .with_password("secret")

conn =
  match Connection.connect(config)
    Result.Ok(c) -> c
    Result.Err(e) -> return Result.Err(e.message())
  end

# Simple query. The connection is a value, so rebind the returned copy.
(conn, outcome) = conn.query("SELECT id, name FROM users")

match outcome
  Result.Ok(result) ->
    # result.fields: List<String>
    # result.rows:   List<List<Value>> (Value.Null = SQL NULL)
    # result.tag:    command tag, e.g. "SELECT 2"
    result.rows.print()
  Result.Err(e) -> IO.puts(e.message())
end

# Parameterized query via the extended protocol. Parameters are typed
# values. The driver declares their types, so the SQL needs no casts.
# Value.Null binds SQL NULL, and `params` defaults to none.
(conn, _) = conn.execute("SELECT name FROM users WHERE id = $1", [Value.int(42)])

_ = conn.close()
```

### Values

Result columns decode to `Postgres.Value` from the type OID the server
reports for each column. The driver maps `bool` to `Value.Bool`, the
integer family to `Value.Int`, and the float family to `Value.Float`.
Every other type (`numeric`, timestamps, `uuid`, `json`) passes
through as `Value.String` in the server's text format.

Parameters bind the same way. `Value.Int` declares `int8`,
`Value.Bool` declares `bool`, and `Value.Float` declares `float8`, so
most queries need no `$1::type` casts. `Value.String` leaves the type
for the server to infer from context.

Constructors cover both required and nullable values. `Value.int(n)`
wraps an `Int`, and `Value.opt_int(maybe)` wraps an `Option<Int>` with
`None` becoming SQL NULL (same for `bool`, `float`, and `string`).
Read result columns with the matching accessors. `value.as_int()`
returns `Option<Int>`, and `value.int_or(fallback)` unwraps with a
fallback.

### Statement cache

`execute` prepares each distinct statement once per connection and
reuses it, keyed by the SQL text and parameter types. The cache holds
`Config.statement_cache_size` statements (default 256) and evicts the
least recently used. Set the size to 0 to turn the cache off.

The cache recovers from stale statements on its own. When a schema
change invalidates a cached plan (SQLSTATE 0A000), or the server
dropped the statement (SQLSTATE 26000, for example a pooler ran
`DISCARD ALL`), the driver evicts the entry, prepares the statement
again, and retries once. Both errors fire at Bind, before the
statement runs, so the retry cannot execute a statement twice.

### Errors

Every failure is a `Postgres.Error`:

| Variant                        | Meaning                                                  |
| ------------------------------ | -------------------------------------------------------- |
| `ConnectFailed(String)`        | The TCP connection failed                                |
| `AuthenticationFailed(String)` | Missing password, or a SCRAM proof or verification error |
| `UnsupportedAuthentication`    | The server demands a method the driver lacks (MD5)       |
| `Server(ServerError)`          | Server-reported error with SQLSTATE + severity           |
| `Protocol(String)`             | Malformed or unexpected wire data                        |
| `IO(String)`                   | Socket read/write failure                                |
| `ConnectionClosed`             | Server closed the connection                             |

`error.message()` renders any variant as a human-readable string.

## Not yet supported

- A row-to-struct decode protocol on top of `Postgres.Value`
- TLS (`sslmode`). Connect over trusted networks or a local socket proxy.
- MD5 password authentication
- Connection pooling
- Binary parameter/result formats
- Decimal and date-time value types (they pass through as text)
- SASLprep normalization of exotic Unicode passwords

## Development

Unit tests are pure. Integration tests expect the bundled Postgres:

```sh
docker compose up -d
koja test
```

The container (Postgres 16 on host port 5434, database `koja_test`)
provisions one user per authentication method: `koja_trust` (trust),
`koja_password` (cleartext), and `koja_scram` (SCRAM-SHA-256).

## License

Copyright (c) 2026 Henry Popp

This project is MIT licensed. See the [LICENSE](LICENSE) for details.
