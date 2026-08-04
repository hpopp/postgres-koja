# Changelog

## [0.2.0] - 2026-08-03

Requires Koja 0.16.

- **Breaking change.** `Connection.query` and `Connection.execute` now return an anonymous tuple `(Connection, Result<QueryResult, Error>)` instead of `Pair`. Destructure with `(conn, outcome) = conn.query(sql)`.
- **Breaking change.** The manifest name is now lowercase `postgres`, matching the Koja 0.16 package naming contract. Update the dependency key in `koja.toml`.

## [0.1.0] - 2026-07-15

Initial release.

- Connect to PostgreSQL over TCP with trust, cleartext password, or SCRAM-SHA-256 authentication.
- Run plain SQL with `Connection.query` via the simple query protocol.
- Run parameterized SQL with `Connection.execute` (`$1`-style text parameters, `Option.None` for NULL) via the extended query protocol.
- Server errors surface as structured values with severity, SQLSTATE code, and message.
