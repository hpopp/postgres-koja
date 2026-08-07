# Postgres

[![CI](https://github.com/hpopp/postgres-koja/actions/workflows/ci.yml/badge.svg)](https://github.com/hpopp/postgres-koja/actions/workflows/ci.yml)
[![Last Updated](https://img.shields.io/github/last-commit/hpopp/postgres-koja.svg)](https://github.com/hpopp/postgres-koja/commits/main)

A PostgreSQL driver for [Koja](https://github.com/koja-lang/koja). It speaks
the v3 wire protocol over TCP, with no C dependencies beyond the Koja stdlib.

## Features

- Trust, cleartext password, and SCRAM-SHA-256 authentication
- Simple query protocol (`query`) for plain SQL
- Extended query protocol (`execute`) with `$1`-style text parameters
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
    # result.rows:   List<List<Option<String>>> (None = SQL NULL)
    # result.tag:    command tag, e.g. "SELECT 2"
    result.rows.print()

  Result.Err(e) ->
    IO.puts(e.message())
end

# Parameterized query via the extended protocol. Parameters are text,
# so cast them in SQL. Option.None binds SQL NULL.
params: List<Option<String>> = [Option.Some("42")]
(conn, _) = conn.execute("SELECT name FROM users WHERE id = $1::int", params)

_ = conn.close()
```

Every result value is the text-format string the server sent
(`Option.None` for NULL). Interpret them with `to_int()` / `to_float()`
as needed.

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

- Typed row decoding (values are text)
- TLS (`sslmode`). Connect over trusted networks or a local socket proxy.
- MD5 password authentication
- Connection pooling
- Binary parameter/result formats
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
