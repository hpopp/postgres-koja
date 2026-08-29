# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0] - 2026-08-29

Requires Koja 0.18.

### Added

- `execute` caches prepared statements per connection, keyed by SQL text and parameter types. The cache holds `Config.statement_cache_size` statements (default 256) with least-recently-used eviction. Set 0 to turn the cache off. Repeated statements skip the server parse and plan step.
- A cached statement that a schema change invalidated (SQLSTATE 0A000), or that the server dropped (SQLSTATE 26000), is prepared again and retried once instead of surfacing the error.
- `Value` gains `opt_bool`, `opt_int`, `opt_float`, and `opt_string` constructors that map `Option` values to SQL NULL, plus `as_*` and `*_or` accessors for reading result columns.
- The `params` argument of `Connection.execute` defaults to no parameters, so statements without placeholders need no empty list.

### Changed

- **Breaking change** Result rows are now typed. `QueryResult.rows` is `List<List<Postgres.Value>>` instead of `List<List<Option<String>>>`. Rows decode from the column type OIDs the server reports. Types without a first-class variant (numeric, timestamps, uuid, json) pass through as `Value.String`, and SQL NULL decodes to `Value.Null`.
- **Breaking change** `Connection.execute` takes `List<Postgres.Value>` parameters instead of `List<Option<String>>`. The driver declares parameter types in the Parse message (`Value.Int` as int8, `Value.Bool` as bool, `Value.Float` as float8), so most `$1::type` casts in SQL are no longer needed. Bind SQL NULL with `Value.Null`.

## [0.2.0] - 2026-08-03

Requires Koja 0.16.

### Changed

- **Breaking change** `Connection.query` and `Connection.execute` now return an anonymous tuple `(Connection, Result<QueryResult, Error>)` instead of `Pair`. Destructure with `(conn, outcome) = conn.query(sql)`.
- **Breaking change** The manifest name is now lowercase `postgres`, matching the Koja 0.16 package naming contract. Update the dependency key in `koja.toml`.

## [0.1.0] - 2026-07-15

Initial release.

### Added

- Connect to PostgreSQL over TCP with trust, cleartext password, or SCRAM-SHA-256 authentication.
- Run plain SQL with `Connection.query` via the simple query protocol.
- Run parameterized SQL with `Connection.execute` (`$1`-style text parameters, `Option.None` for NULL) via the extended query protocol.
- Server errors surface as structured values with severity, SQLSTATE code, and message.
