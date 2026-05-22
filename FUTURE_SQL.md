# Potential Future: Validated `sql!` And `sql_file!`

This document captures a possible future direction for `pgQuill`.

It is intentionally outside the current implementation scope. The main project should focus first on the typed PostgreSQL DSL, schema generation, runtime integration, and record layer.

## Idea

A strong future version of `sql!` should:

- parse SQL using PostgreSQL's real parser
- validate referenced schemas, tables, columns, aliases, and functions against a generated snapshot
- infer parameter types
- infer output row shape
- fail at compile time when the schema and query disagree

Example direction:

```rust
let q = sql!(
    r#"
    select u.id, u.name
    from public.users as u
    where u.id = :user_id
    "#,
    user_id: i64
);

let row: (i64, String) = q.fetch_one(&db).await?;
```

Recommendation:

- keep it string-based first
- validate it against generated schema metadata or prepare-time artifacts
- rewrite named parameters to PostgreSQL positional binds internally

Why string-based first instead of token-tree SQL immediately:

- PostgreSQL has many operators (`@>`, `->>`, `#>>`, `?|`, casts, array syntax)
- Rust tokenization can make a token-tree SQL macro awkward quickly
- `$1` placeholders are not a great Rust macro UX

If a token-like macro is ever added later, this is a safer direction:

```rust
let q = psql!(
    SELECT u.id, u.name
    FROM public.users AS u
    WHERE u.id = ${user_id}
);
```

## Why This Is Realistic Here

This repository already has local precedent for the hardest part:

- `backend-rust/schemalane/schemalane-core` uses `pg_query`
- `backend-rust/schemalane/pg_query_fmt` formats PostgreSQL AST back to SQL

That means `pgQuill` can likely reuse the same parser family for:

- splitting and parsing SQL
- AST walking
- location-aware errors
- formatting validated queries for debugging

## Recommended Future Direction

The most important scope decision here is this:

- PostgreSQL parsing is realistic locally
- full PostgreSQL semantic analysis is a much larger project

So the recommended direction is a hybrid:

- use `pg_query`-style parsing locally for syntax, formatting, and diagnostics
- use a prepare/generate step against a real PostgreSQL database for authoritative type validation and result-shape capture
- check generated metadata or wrappers into the repo for offline CI builds

That is much closer to the SQLx / Cornucopia model, and it avoids turning `pgQuill` into a full PostgreSQL compiler front-end too early.

Potential implementation phases for this idea:

### Phase 1

- `sql!(r#"..."#, binds...)` or `sql_file!("queries/foo.sql")`
- validate queries during an explicit prepare/generate step against a dev database
- capture bind types and result-column metadata into generated artifacts
- emit lightweight wrappers that execute through `tokio-postgres`

### Phase 2

- use local PostgreSQL parsing for richer editor diagnostics and better error reporting
- support direct decoding into generated row structs or tuples
- add stronger support for CTE scopes, lateral scopes, subqueries, and set operations in generated metadata

### Phase 3

- optional token-tree sugar macro
- better diagnostics with highlighted query spans
- deeper local validation where it clearly adds value without duplicating PostgreSQL's whole type checker

Important caution:

Do not block the entire project on perfect local SQL inference. A useful `sql!` backed by authoritative database validation is already extremely valuable.
