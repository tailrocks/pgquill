# pgQuill

> pgQuill — write PostgreSQL with types, not strings

## Thesis

`pgQuill` is inspired by jOOQ from the Java world.

There is no close Rust equivalent today, so this specification will sometimes refer to jOOQ as a base reference for ideas, capabilities, and tradeoffs.

That reference is intentionally partial:

- `pgQuill` is not a line-by-line port of jOOQ
- some jOOQ features will be ignored
- some APIs and implementation details will differ
- when Rust ergonomics, PostgreSQL-only scope, or project goals point in a different direction, `pgQuill` should make its own design decisions

`pgQuill` is a very promising idea if it stays aggressively focused:

- PostgreSQL only
- `tokio-postgres` only at runtime
- SQL-first, not ORM-first
- real AST, not a string builder
- generated schema metadata and Rust code from `pg_catalog`
- typed SQL that still looks like SQL

There is a real gap in Rust today:

- `Diesel` is very type-safe, but its DSL often stops feeling like SQL
- `SeaQuery` is flexible, but less schema-first and less compiler-driven
- `sqlx` keeps SQL visible, but its compile-time checking is query-string centric rather than a jOOQ-style generated DSL

`pgQuill` can sit in that gap by combining two things in one system:

1. A jOOQ-like typed DSL that reads close to PostgreSQL
2. Schema-aware code generation from PostgreSQL catalogs

## What Success Looks Like

The product should feel like this:

```rust
use pgquill::prelude::*;
use crate::schema::public::{posts, users};

let q = select((users::id, users::name))
    .from(users)
    .filter(users::id.equal(1_i64))
    .order_by(users::name.asc())
    .limit(10);

let rows: Vec<(i64, String)> = q.fetch_all(&db).await?;
```

And this should fail at compile time:

```rust
users::id.equal("abc"); // wrong Rust type for bigint/int column
```

The SQL should remain recognizable:

```rust
let q = with(active_users)
    .select((users::id, users::name))
    .from(users)
    .left_join(posts)
    .on(posts::author_id.equal(users::id))
    .filter(users::deleted_at.is_null());
```

The biggest differentiator is not only the DSL. It is the combination of:

- generated schema modules
- typed runtime execution
- a real AST built for PostgreSQL rather than a generic string builder

## Native Driver Principle

`pgQuill` should use `tokio-postgres` and `postgres-types` as natively as possible.

That means:

- use `tokio_postgres::types::ToSql`, `FromSql`, `FromSqlOwned`, and `Type` directly
- rely on the fact that `tokio_postgres::types` is the natural place to access the `postgres-types` API in application code
- prefer existing driver-supported Rust mappings over inventing a parallel runtime type layer
- add `pgQuill` abstractions only where the driver layer does not already solve the problem

So `pgQuill` should add value in:

- SQL AST and DSL construction
- schema code generation
- typed query/result composition

It should avoid adding:

- a duplicate Postgres codec layer
- a duplicate runtime type registry
- copy-pasted replacements for `postgres-types`

## Product Pillars

### 1. SQL-first DSL

The DSL should model SQL clauses directly:

- `select`
- `from`
- `join`
- `where`
- `group_by`
- `having`
- `order_by`
- `limit`
- `offset`
- `insert_into`
- `update`
- `delete_from`
- `returning`
- `with`

The ideal mental model is:

- if it exists in PostgreSQL, it should exist in the AST
- the DSL is only a typed way to construct that AST
- rendering is a separate step

In the fluent Rust API, `filter(...)` is preferred over `where(...)` because `where` is a Rust keyword. In SQL itself, `WHERE` should stay spelled as real SQL.

### 2. Schema-driven generation

Generated code should give users:

- schema modules
- table values
- typed columns
- row structs
- relationship metadata
- generated aliasable table values
- a schema snapshot for future validation, debugging, and tooling

### 3. Type-checked SQL without hiding SQL

The core implementation scope should focus on the typed DSL.

A validated raw SQL surface is a separate potential future idea and is documented in [FUTURE_SQL.md](./FUTURE_SQL.md).
That file is not part of this README's implementation specification, should not be used for current planning, and should not be considered part of the implementation for now.

## jOOQ DSL Design Principles For pgQuill

`pgQuill` should explicitly follow the spirit of jOOQ's DSL API design, especially as described in the jOOQ manual:

- [The DSL API](https://www.jooq.org/doc/latest/manual/sql-building/dsl-api/)
- [Mutability (historic)](https://www.jooq.org/doc/latest/manual/sql-building/dsl-api/mutability-historic/)
- [Don't do this with jOOQ: referencing the Step types](https://www.jooq.org/doc/latest/manual/reference/dont-do-this/dont-do-this-jooq-step-types/)

The most important principles to carry over are:

### 1. The DSL is the primary user-facing API

The default way to work with `pgQuill` should be the fluent SQL DSL, not a lower-level query object model.

That means:

- docs should teach the DSL first
- generated schema code should be designed for the DSL first
- helper APIs should feel like extensions of the DSL, not an unrelated layer

The internal AST still matters, but it should mostly power the DSL rather than replace it in normal application code.

### 2. The API should read like SQL

The main ergonomic goal is not merely "fluent Rust." It is "Rust that visually tracks SQL."

That means:

- clause order should follow SQL clause order
- names should stay close to SQL terms where Rust syntax allows it
- projection, join, filtering, grouping, ordering, and returning should be easy to recognize at a glance

If an API choice is more Rust-idiomatic but makes the resulting query look much less like SQL, `pgQuill` should be cautious about taking that trade.

### 3. The builder should encode SQL grammar, not just chain methods

One of jOOQ's strongest ideas is that the DSL is not merely a stringy fluent API. Its type structure reflects legal SQL shape.

`pgQuill` should aim for the same effect:

- illegal clause ordering should be hard or impossible to express
- incomplete statements should be typed as incomplete
- invalid combinations should fail as early as possible

In Rust, this does not have to mean exposing a huge public hierarchy of named step types. But the internal builder states should still model SQL grammar in a BNF-like way.

### 4. Do not leak step types into user code

jOOQ explicitly warns users not to depend on its `Step` interfaces directly. That is a very important lesson for `pgQuill`.

The Rust equivalent is:

- do not make users name intermediate clause-state types in normal code
- do not encourage APIs that pass builder-state types around the application
- keep clause-state generics/internal types mostly opaque

Users should mostly write:

```rust
let q = select(users::all())
    .from(users)
    .filter(users::is_active.is_true())
    .order_by(users::name.asc());
```

They should not need to care what the exact intermediate builder type is after `.from(...)` or `.filter(...)`.

### 5. Prefer immutable statement building

jOOQ's own manual documents that parts of its DSL are historically mutable. That is not a design to copy.

For `pgQuill`, immutable query construction is the better default:

- each call conceptually returns a new query value
- expression nodes are immutable
- statement builders should behave immutably from the user's perspective

That will make the library easier to reason about in Rust and safer for dynamic query composition.

### 6. The AST and DSL should stay close together

jOOQ distinguishes between DSL API and model API, but the two are tightly related.

`pgQuill` should do the same:

- the DSL constructs the AST
- the AST can still be inspected, rendered, transformed, or reused
- advanced users may eventually work with a lower-level model API, but the DSL remains the main path

This is one of the strongest reasons to keep a real AST at the core.

### 7. IDE discoverability matters

jOOQ's manual emphasizes that syntax-oriented types plus autocomplete are powerful. `pgQuill` should optimize for the same effect in Rust analyzers and IDEs.

That means:

- generated schema items should be easy to discover through completion
- operator names should be predictable and consistent
- common PostgreSQL functions should be available through predictable free functions
- aliases, joins, and projections should remain obvious from the API surface

### 8. Dynamic SQL should be a first-class use case

jOOQ's DSL is powerful because it handles both static and dynamic query construction.

`pgQuill` should preserve that:

- expressions and conditions should be composable values
- optional clauses should be easy to add conditionally
- users should be able to build query fragments and combine them safely

This is another reason to avoid exposing public step-type machinery as the main abstraction.

## Concrete Implications For pgQuill

If `pgQuill` is built around those principles, a few design consequences become clear:

- keep the DSL as the primary API surface
- make the underlying AST real and explicit
- use grammar-aware builder states internally, but do not leak them as public API commitments
- prefer immutable query construction
- keep method names SQL-like, but adapted to Rust where keywords force renaming
- design everything for autocomplete and generated-schema discoverability
- support dynamic query composition without forcing users into raw SQL too early

## Specification Language And Validation Boundaries

To keep this README actionable as an implementation specification, these keywords have strict meaning:

- `MUST`: required behavior for a conforming implementation
- `SHOULD`: recommended default behavior, with deviation allowed only for strong reasons
- `MAY`: optional behavior

Validation boundaries are also part of the contract:

- compile time (`rustc`) MUST reject statically knowable DSL type errors for static query shapes
- codegen time MUST reject invalid catalog/config combinations before runtime
- runtime MUST handle execution failures and dynamic projection/DTO mismatch checks

For static typed selections, runtime validation MUST NOT be used as the primary correctness mechanism when compile-time typing can enforce the rule.

## Conformance Profiles

To remove scope ambiguity, this specification defines three conformance profiles:

- `P1` (Core): mandatory baseline for initial production use
- `P2` (Extended PostgreSQL): advanced query forms and PostgreSQL type surface
- `P3` (Advanced SQL): late-surface features with higher implementation complexity

Conformance rules:

- a build that claims `P1` conformance MUST satisfy every `P1` requirement in this document
- a build that claims `P2` conformance MUST satisfy `P1` + `P2`
- a build that claims `P3` conformance MUST satisfy `P1` + `P2` + `P3`
- a build that claims full/final `pgQuill` implementation conformance MUST satisfy all three profiles

Profile activation policy:

- when a section and the profile mapping disagree, the profile mapping in `Conformance Scope` is authoritative
- requirements attached to a later profile are intentionally non-blocking for earlier profiles

## Final DSL Conventions

To keep the DSL readable and predictable, `pgQuill` should standardize on these conventions:

- use generated table values in the value namespace: `.from(users)`, `.join(posts)`, and top-level statements like `update(users)`
- keep columns in the module namespace: `users::id`, `users::email`, `posts::author_id`
- keep one primary query style: `select(...).from(...).join(...).filter(...)`
- keep the project strongly type-oriented: query construction should make the selected schema and decoded Rust shape obvious from the call site
- do not add convenience query sugar in the initial phases; prefer one explicit approach over multiple equivalent entry points
- allow PostgreSQL `SELECT` without `FROM` when the query consists only of scalar expressions, functions, and lifted literal values
- use `filter(...)` consistently for `WHERE`
- use top-level `SELECT` constructors consistently:
  - `select(...)`
  - `select_distinct(...)`
  - `select_distinct_on(on_exprs, selection)`
- do not expose a generic `SELECT *` constructor in the fluent DSL
- use `select(users::all())` when you specifically mean typed `users.*`
- prefer explicit typed selections over bare `*`, because generic `SELECT *` works poorly with static row typing, joins, and schema evolution
- discourage generic wildcard query styles in general, because they make it harder to predict potential result columns and decoded Rust output by reading the DSL
- use free functions for SQL functions, except where SQL is more naturally expression-oriented:
  - `count()` for `COUNT(*)`
  - `expr.count()` for `COUNT(expr)`
  - `concat(...)`
  - `trim(...)`
  - `now()`
- use `val(...)` to lift Rust values into SQL expressions when a literal must participate in the AST directly, especially for aliasing
- do not add a separate public `COUNT(1)` spelling; `count()` and `expr.count()` are the canonical `COUNT` forms
- use the same clause methods for static and dynamic queries:
  - `order_by(...)`, not a separate dynamic-only method
  - `limit(...)`, with optional values handled by input traits rather than a second method name
- prefer `all_of([...])` and `any_of([...])` for optional predicate composition
- keep terminal methods consistent across result shapes:
  - `fetch_all(...)`
  - `fetch_one(...)`
  - `fetch_optional(...)`
  - `fetch_into::<T>(...)`
  - `fetch_one_into::<T>(...)`
  - `fetch_optional_into::<T>(...)`

This gives the project one canonical style instead of a family of competing styles.

## Grammar And Clause-State Contract

To satisfy compile-time grammar guarantees, builder state transitions MUST follow this contract.

### `SELECT` grammar contract (P1)

- start state MUST be one of: `select(...)`, `select_distinct(...)`, `select_distinct_on(...)`
- `from(...)` MAY be omitted only for PostgreSQL `SELECT` without `FROM`
- after `from(...)`, zero or more joins MAY be appended
- `filter(...)` MAY appear at most once and only after `from(...)` / joins
- `group_by(...)` MAY appear at most once
- `having(...)` MUST only be legal when `group_by(...)` is already present
- `order_by(...)`, `limit(...)`, and `offset(...)` MAY each appear at most once
- terminal methods MUST be available only on grammatically complete query states

### `INSERT` grammar contract (P1)

- `insert_into(table)` MUST be followed by `columns(...)` then `values(...)` in the initial implementation
- `on_conflict_on_constraint(...)` MAY appear only after a complete insert source
- `do_nothing()` and `do_update_set()` MUST only be legal after `on_conflict_on_constraint(...)`
- `returning(...)` MAY appear at most once and only after write-shape completion
- `execute(...)` MUST only be legal when no `returning(...)` is present
- `fetch_*` terminals MUST only be legal when `returning(...)` is present

### `UPDATE` grammar contract (P1)

- `update(table)` MUST require at least one `set(...)` call before terminal methods
- `filter(...)` MAY appear at most once
- `returning(...)` MAY appear at most once
- `execute(...)` and `fetch_*` terminal availability MUST follow the same `returning(...)` rule as `INSERT`

### `DELETE` grammar contract (P1)

- `delete_from(table)` MAY optionally include `filter(...)`
- `returning(...)` MAY appear at most once
- `execute(...)` and `fetch_*` terminal availability MUST follow the same `returning(...)` rule as `INSERT`

### `MERGE` grammar contract (P3)

- `merge_into(target).using(source).on(condition)` is the required structural prefix
- one or more `when_*` branches MUST be present
- branch order MUST be preserved exactly in the AST and renderer
- `returning(...)` MAY be exposed only if the active PostgreSQL runtime target supports it for the rendered branch form

Common compile-time behavior:

- illegal clause transitions MUST be rejected at compile time for statically known query shapes
- dynamic clause omission (for example empty optional inputs) MUST remove the clause rather than permit invalid intermediate states

## Conformance Scope

### Profile `P1` (Core Required Baseline)

- PostgreSQL 18+ only (minimum supported version is PostgreSQL 18)
- `SELECT`, `INSERT`, `UPDATE`, `DELETE`
- PostgreSQL `ON CONFLICT`
- select modifiers: `DISTINCT` and PostgreSQL `DISTINCT ON`
- joins, aliases, subqueries
- `RETURNING`
- `GROUP BY` / `HAVING`
- built-in SQL functions and basic aggregates
- generated tables, columns, primary keys, foreign keys, unique keys
- typed decoding through `tokio-postgres`

### Profile `P2` (Extended PostgreSQL Features)

- `WITH` / CTE, including `WITH RECURSIVE`
- `LATERAL`
- PostgreSQL table functions via explicit declarations
- arrays
- `JSONB`
- `hstore`
- `citext`
- PostgreSQL domains
- PostgreSQL enums
- composite types

### Profile `P3` (Advanced SQL)

- `MERGE`
- window functions

### Full Conformance Target

- a final implementation claim MUST satisfy `P1` + `P2` + `P3`

### Permanent Non-Goals

- multiple SQL dialects
- ORM-style unit of work / identity map
- hidden lazy loading
- automatic migration system inside `pgQuill`
- runtime abstraction over multiple drivers
- first-class view and materialized view support
- database function code generation
- record layer / entity CRUD (delegated to `toasty`; see *Runtime Integration And Toasty Pairing*)

### PostgreSQL Version Policy

`pgQuill` targets PostgreSQL 18 and newer only.

Implementation contract:

- code generation MUST fail fast for target servers below PostgreSQL 18 with `CodegenError::UnsupportedPostgresVersion`
- runtime SHOULD verify server version compatibility once at startup or first connection and return `ExecutionError::UnsupportedServerVersion` when violated
- no compatibility shims for PostgreSQL 17 or earlier should be added to the main DSL surface

This is important: jOOQ is powerful partly because it is close to SQL, not because it behaves like a full ORM.

## Strong Recommendation: Typed Wrappers Over An Untyped Core AST

This is the most important architectural decision.

Do not encode the full AST shape in Rust generics. That leads toward Diesel-style type complexity, giant compiler errors, and long compile times.

Instead, use:

- a real runtime AST with enums/structs
- typed wrapper structs with `PhantomData`
- semantic validation while building the AST

Example direction:

```rust
pub struct Expr<T> {
    node: ExprNode,
    _ty: std::marker::PhantomData<T>,
}

pub enum ExprNode {
    Column(ColumnRef),
    Param(ParamRef),
    Literal(Literal),
    Binary {
        left: Box<ExprNode>,
        op: BinaryOp,
        right: Box<ExprNode>,
    },
    Call(FunctionCall),
    Cast {
        expr: Box<ExprNode>,
        ty: PgTypeId,
    },
    FieldAccess {
        expr: Box<ExprNode>,
        field: String,
    },
    Index {
        expr: Box<ExprNode>,
        index: Box<ExprNode>,
    },
    Subquery(Box<SelectAst>),
}

pub struct Select<S> {
    ast: SelectAst,
    _selection: std::marker::PhantomData<S>,
}
```

That gives you:

- a real AST
- good runtime inspection and rendering
- compiler-checked expression types
- much saner trait bounds than a fully generic AST

This is the best path if you want jOOQ ergonomics in Rust without reproducing Diesel's complexity.

## Type System Sketch

The SQL type system should be explicit and PostgreSQL-native, but it should not duplicate the runtime type machinery already provided by `postgres-types`.

The better model is:

- Rust types describe values
- `tokio_postgres::types::Type` describes the concrete PostgreSQL type
- generated schema metadata preserves PostgreSQL-specific identity such as domain, enum, composite, array, `citext`, and `hstore`

Conceptually:

```rust
pub struct Expr<T> {
    node: ExprNode,
    pg_type: tokio_postgres::types::Type,
    _rust: std::marker::PhantomData<T>,
}
```

Nullability should use native Rust:

```rust
Option<T>
```

Core expression rules:

- `Expr<i64>` can compare with `Expr<i64>`
- `Expr<Option<i64>>` is not the same as `Expr<i64>`
- `Expr<bool>` is required for `WHERE`, `ON`, `HAVING`
- function calls define output types
- selected fields determine the row shape

Strict typing contract:

- `pgQuill` MUST use strict type matching for typed DSL operations
- implicit numeric widening/coercion (for example `i32` to `i64`) MUST NOT be applied automatically in typed comparisons or assignments
- domain and base types MUST be treated as distinct typed expressions unless the user performs an explicit cast
- when coercion is needed, users MUST opt in with explicit casts in the DSL
- this strictness is intentional and should be treated as a core Rust ergonomics and correctness feature rather than a temporary limitation

Ergonomic literal lifting policy (Option B):

- strict SQL typing remains the default contract
- `pgQuill` MAY provide ergonomic lifting for obvious borrowed literal forms where SQL type identity is unambiguous (for example `&str` into text-like expression inputs)
- this ergonomic lifting MUST NOT introduce numeric widening/coercion
- this ergonomic lifting MUST NOT bypass explicit domain/base distinction rules
- if type identity is ambiguous, the user must use explicit `val(...)` or cast expressions

Rust-facing type mapping should be generated whenever possible:

- `bool` -> `bool`
- `int2` -> `i16`
- `int4` -> `i32`
- `int8` -> `i64`
- `text` / `varchar` / `citext` -> `String`
- `bytea` -> `Vec<u8>`
- nullable columns -> `Option<T>`
- arrays -> `Vec<T>`
- `uuid` -> `uuid::Uuid`
- `jsonb` -> `serde_json::Value` by default, or a user type built on the native JSON support
- `hstore` -> `std::collections::HashMap<String, Option<String>>`
- domains -> generated domain newtype by default, or a user-provided Rust type through codegen config
- enums -> generated Rust enum
- composites -> generated Rust struct

For PostgreSQL user-defined types, `postgres-types` derive support is a strong fit:

- enums can derive `ToSql` / `FromSql`
- composites can derive `ToSql` / `FromSql`
- domain-mapped Rust newtypes can derive `ToSql` / `FromSql` as tuple structs
- transparent Rust-only wrappers can use `#[postgres(transparent)]` when appropriate

## Nullability Model

Nullability is where SQL DSLs become either credible or frustrating.

Recommended rule:

- generate column nullability from catalog metadata
- propagate nullability through joins and expressions
- err on the side of `Option<T>` when inference is uncertain

Examples:

- left-joined table columns become nullable
- `count()` and `expr.count()` are non-null
- ordinary scalar subquery result is nullable unless proven otherwise
- `coalesce` returns non-null only if the output can be proven non-null

Conservative default for unresolved inference:

- scalar subquery expressions SHOULD be treated as nullable (`Option<T>`) by default
- expression operators/functions with uncertain nullability SHOULD produce nullable outputs
- this is preferable to unsound non-null inference and matches the specification goal of correctness-first typing

Deterministic nullability rules required in `P1`:

- arithmetic operators (`+`, `-`, `*`, `/`) MUST return nullable output when any input is nullable, otherwise non-null
- `count()` and `expr.count()` MUST be non-null
- `sum`, `avg`, `min`, and `max` MUST be nullable
- `is_null`, `is_not_null`, `is_true`, `is_not_true`, `is_false`, `is_not_false`, `is_unknown`, and `is_not_unknown` MUST be non-null boolean
- `is_distinct_from` and `is_not_distinct_from` MUST be non-null boolean
- scalar subquery expressions MUST default to nullable unless the API requires explicit null-elimination (for example `coalesce`)

Worked nullability examples (implementation reference):

```rust
// Assume catalog nullability:
// users::id: Expr<i64>
// users::middle_name: Expr<Option<String>>
// users::age: Expr<Option<i32>>
// users::score: Expr<i32>
```

- `users::id` -> `Expr<i64>`
- `users::middle_name` -> `Expr<Option<String>>`
- `users::score.plus(1_i32)` -> `Expr<i32>`
- `users::age.plus(1_i32)` -> `Expr<Option<i32>>`
- `users::score.plus(users::age)` -> `Expr<Option<i32>>`
- `coalesce(users::age, val(0_i32))` -> `Expr<i32>`
- `coalesce(users::middle_name, val("n/a"))` -> `Expr<String>`

Join-driven examples:

```rust
let q = select((users::id, posts::title))
    .from(users)
    .left_join(posts)
    .on(posts::author_id.equal(users::id));
```

- `users::id` in projection remains non-null: `i64`
- `posts::title` in projection becomes nullable under left join: `Option<String>`

Scalar subquery example:

```rust
let latest_title = select(posts::title)
    .from(posts)
    .filter(posts::author_id.equal(users::id))
    .order_by(posts::created_at.desc())
    .limit(1)
    .scalar_subquery();
```

- `latest_title` SHOULD be treated as `Expr<Option<String>>` by default
- callers can make non-null intent explicit with `coalesce(latest_title, val(""))` or another explicit null-handling expression

If nullability inference is incomplete at first, it is better to be conservative than incorrect.

## DSL API Ideas

## A. Core jOOQ-style fluent DSL

This should be the main API.

```rust
let q = select((users::id, users::name))
    .from(users)
    .filter(users::id.equal(1_i64));
```

With aliases:

```rust
let u = users.alias("u");
let p = posts.alias("p");

let q = select((u.col(users::name), p.col(posts::title)))
    .from(u)
    .join(p)
    .on(p.col(posts::author_id).equal(u.col(users::id)));
```

This is very close to SQL and readable in reviews.

Do not make a table-rooted API like `users.select(...)` or `select_from(users)` part of the primary design. It moves the visual shape away from SQL and creates a second way to express the same query.

## B. JOIN design

The jOOQ join-clause documentation has an important design lesson for `pgQuill`:

- joins should work directly after `.from(...)` for readability
- joins should also work on table expressions themselves for nested join trees

That gives two complementary styles.

### 1. Query-chain joins

This should be the default style for most application code:

```rust
let q = select((author::id, book::title))
    .from(author)
    .join(book)
    .on(book::author_id.equal(author::id));
```

This is the clearest equivalent of ordinary SQL:

```sql
SELECT author.id, book.title
FROM author
JOIN book ON book.author_id = author.id
```

This should be the style shown most often in docs.

### 2. Table-expression joins

For more advanced join trees, `pgQuill` should also allow joins directly on table expressions:

```rust
let joined_books = book
    .join(book_to_book_store)
    .on(book_to_book_store::book_id.equal(book::id));

let q = select((author::id, book::title))
    .from(
        author
            .left_join(joined_books)
            .on(book::author_id.equal(author::id)),
    );
```

This is valuable because it allows:

- nested join trees
- explicit parenthesized join structure
- reusable joined table expressions
- better support for complex join graphs

That follows the same basic idea jOOQ calls out in its join-clause page.

### 3. Join kinds

At minimum, `pgQuill` should support these join constructors:

- `join(...)`
- `left_join(...)`
- `right_join(...)`
- `full_join(...)`
- `cross_join(...)`

Likely follow-up support:

- `natural_join(...)`
- `natural_left_join(...)`

For PostgreSQL-only features or higher-level semantics, `pgQuill` can later consider:

- `semi_join(...)`
- `anti_join(...)`

The last two do not need to be literal PostgreSQL syntax in the final renderer. They can be AST-level concepts lowered to `EXISTS` / `NOT EXISTS` when appropriate.

### 4. Join predicates

Initially, the DSL should make `ON` explicit and typed. `JOIN ... USING (...)` should stay out of the current implementation plan, because it changes result-shape semantics and needs a more precise API.

```rust
let q = select(book::all())
    .from(book)
    .join(author)
    .on(book::author_id.equal(author::id));
```

Recommended rules:

- `join(...)` is the canonical spelling for `INNER JOIN`
- `join(...)` should require a follow-up `on(...)` unless the join type does not need one
- `cross_join(...)` should not allow `on(...)`
- `LATERAL` should be modeled as a table-expression wrapper, for example `lateral(subquery)` or `lateral(table_function(...))`
- a lateral target should then be joined through the normal `join(...)`, `left_join(...)`, or `cross_join(...)` APIs
- `on_true()` should remain available for cases like left joining a lateral subquery without a predicate
- `JOIN ... USING (...)` should stay outside the current implementation plan, because it changes result-shape semantics and needs a more precise API than plain column references

### 5. Nullability propagation across joins

This is very important for type safety:

- `INNER JOIN` keeps joined columns non-null according to their base nullability
- `LEFT JOIN` makes right-side projected columns nullable
- `RIGHT JOIN` makes left-side projected columns nullable
- `FULL JOIN` makes both sides nullable

This interacts directly with selection typing and generated row decoding, so the join model and the nullability model should be designed together.

### 6. Relationship-aware joins can be a later layer

Because `pgQuill` generates foreign-key metadata, it may eventually support one relationship-aware join sugar:

```rust
book.join(author).on_fk()
```

This is the right shape if the project wants relationship sugar at all, because it still reads like `JOIN ... ON ...`.

Recommended rule:

- keep `join(...).on_fk()` as the only relationship-aware join shorthand
- use it only when there is exactly one obvious foreign-key relationship between the two sides
- if there are multiple possible relationships, require either `on_fk(relation)` or a fully explicit `.on(...)`

Do not add a second spelling like `join_related(...)`. It is less SQL-shaped and creates two ways to write the same join.

This should remain a later ergonomic layer. The primary API should still allow fully explicit joins, because that is what keeps the library honest and SQL-shaped.

### 7. Dynamic joins should stay composable

One useful thing in jOOQ is that complex joins can still be built dynamically.

`pgQuill` should preserve that by making joined table expressions first-class values:

```rust
let from_item = if include_author {
    book
        .join(author)
        .on(book::author_id.equal(author::id))
        .into_from_item()
} else {
    book.into_from_item()
};

let q = select(book::title).from(from_item);
```

That is better than exposing public step types or forcing raw SQL for optional joins.

## C. Basic operator surface

`pgQuill` should support the core PostgreSQL comparison and matching operators with consistent Rust-friendly long names.

Recommended baseline:

- `equal(value)` -> `=`
- `not_equal(value)` -> `<>`
- `greater_than(value)` -> `>`
- `greater_or_equal(value)` -> `>=`
- `less_than(value)` -> `<`
- `less_or_equal(value)` -> `<=`
- `is_null()` -> `IS NULL`
- `is_not_null()` -> `IS NOT NULL`
- `is_true()` -> `IS TRUE`
- `is_not_true()` -> `IS NOT TRUE`
- `is_false()` -> `IS FALSE`
- `is_not_false()` -> `IS NOT FALSE`
- `is_unknown()` -> `IS UNKNOWN`
- `is_not_unknown()` -> `IS NOT UNKNOWN`
- `is_distinct_from(value)` -> `IS DISTINCT FROM`
- `is_not_distinct_from(value)` -> `IS NOT DISTINCT FROM`
- `is_in(values)` -> `IN (...)`
- `like(pattern)` -> `LIKE`
- `ilike(pattern)` -> `ILIKE`

Examples:

```rust
users::id.equal(1_i64)
users::created_at.greater_or_equal(cutoff)
users::name.like("Al%")
users::email.ilike("%@example.com")
users::deleted_at.is_not_null()
users::id.is_in(vec![1_i64, 2, 3])
users::is_active.is_true()
users::is_deleted.is_not_false()
users::parent_id.is_distinct_from(users::owner_id)
```

This keeps the API:

- close to SQL
- compatible with Rust naming rules
- consistent with the jOOQ concepts you already use today
- explicit and self-documenting in larger queries

Use `is_in(...)` instead of `in(...)`, because `in` is a Rust keyword.

For projection aliases, use `alias(...)` instead of `as(...)`, because `as` is a Rust keyword:

```rust
bitcoin_address::address_compressed.alias("address")
```

## D. Condition composition

`pgQuill` should make boolean condition composition feel natural, because this is one of the most common jOOQ patterns in real code.

The basic rule should be:

- comparison operators produce a boolean expression
- boolean expressions can be combined with `.and(...)`, `.or(...)`, and `.not()`
- nesting should be represented by the AST directly, not reconstructed later from strings

Straight translation of your jOOQ example:

```rust
let rows = select(book::all())
    .from(book)
    .filter(
        book::author_id
            .equal(1_i64)
            .and(book::title.equal("1984")),
    )
    .fetch_all(&db)
    .await?;
```

Nested `OR` / `AND` should work the same way:

```rust
let rows = select(book::all())
    .from(book)
    .filter(
        book::author_id.equal(1_i64).and(
            book::title.equal("1984").or(
                book::title.equal("Animal Farm"),
            ),
        ),
    )
    .fetch_all(&db)
    .await?;
```

When mixing `AND` and `OR`, prefer explicit grouping:

```rust
let condition = all_of([
    Some(any_of([
        Some(book::author_id.equal(1_i64)),
        Some(book::author_id.equal(2_i64)),
    ])),
    Some(book::published_in.greater_or_equal(1940_i32)),
]);

let rows = select(book::all())
    .from(book)
    .filter(condition)
    .fetch_all(&db)
    .await?;
```

Important design note:

- method chaining should preserve the explicit tree structure the user writes
- the SQL renderer should insert parentheses whenever needed to preserve meaning
- users should not have to think about SQL precedence rules when building nested expressions

For more dynamic query construction, the canonical public combinators are:

- `all_of([cond1, cond2, cond3])`
- `any_of([cond1, cond2, cond3])`

Example:

```rust
let condition = all_of([
    Some(book::author_id.equal(1_i64)),
    title_filter.map(|t| book::title.ilike(t)),
    published_after.map(|y| book::published_in.greater_or_equal(y)),
]);

let rows = select(book::all())
    .from(book)
    .filter(condition)
    .fetch_all(&db)
    .await?;
```

That is especially useful in Rust for optional filters and dynamic search screens.

## E. Dynamic SQL and optional query parts

The jOOQ dynamic SQL pages reinforce a very important idea for `pgQuill`:

- select items are values
- conditions are values
- table expressions and join trees are values
- order-by items, group-by items, and CTEs are values

That means query fragments should be constructible outside the final statement and then embedded later.

This fits `pgQuill` very well, because the whole design already assumes a real AST rather than string concatenation.

`into_from_item()` should be a real documented advanced helper for dynamic AST assembly. Similar low-level conversions like `into_select_item()` should also be available where needed, but they should not be treated as the primary public DSL style.

Example:

```rust
let display_name = concat(
    trim(author::first_name),
    trim(author::last_name),
);

let select_items = vec![
    display_name.alias("display_name"),
    count(),
];

let from_item = author
    .join(book)
    .on(author::id.equal(book::author_id))
    .into_from_item();

let rows = select(select_items)
    .from(from_item)
    .group_by((author::id, author::first_name, author::last_name))
    .order_by(count().desc())
    .fetch_all(&db)
    .await?;
```

That is the core jOOQ dynamic SQL idea translated into Rust: query parts are reusable AST values.

### Recommended Rust-native interpretation

`pgQuill` should copy the capability, but not necessarily the exact Java API shape.

In jOOQ, helpers like `noCondition()`, `noField()`, `noTable()`, and `noPath()` act like optional query parts that disappear when rendered.

In Rust, the most natural building blocks are usually:

- `Option<T>`
- `Vec<T>`
- arrays/slices of optional parts
- `then_some(...)`
- iterator-driven composition

So the primary `pgQuill` dynamic SQL story should be:

- use `all_of([...])` / `any_of([...])` for optional predicates in the public DSL
- use `Vec<SelectItem>` / `Vec<OrderByItem>` / `Vec<Cte>` for dynamic clause assembly
- use the normal clause methods like `limit(...)` and `offset(...)`, with optional values handled by input traits rather than separate dynamic-only APIs
- use `FromItem` / `JoinTree` values that can be selected conditionally before the final query is built

`Option<Condition>` can still be a useful low-level building block internally, but the canonical public style should stay `all_of([...])` / `any_of([...])`.

This is more idiomatic in Rust than copying Java sentinel objects everywhere.

### 1. Optional conditions

This is the easiest and most important case, and it is already a natural fit for `pgQuill`.

Recommended direction:

- `all_of([...])` should ignore missing conditions
- `any_of([...])` should ignore missing conditions
- if the final condition set is empty, the clause should disappear entirely

Example:

```rust
let condition = all_of([
    title.map(|t| book::title.ilike(t)),
    author_id.map(|id| book::author_id.equal(id)),
    only_published.then_some(book::published_at.is_not_null()),
]);

let rows = select(book::all())
    .from(book)
    .filter(condition)
    .fetch_all(&db)
    .await?;
```

This is effectively the `noCondition()` idea, but with a Rust-first surface.

The canonical public style should be collection-based composition with `all_of([...])` and `any_of([...])`.

Do not make `no_condition()` part of the main documented DSL. If it exists at all, it should stay internal or advanced-only.

### Clause Input Trait Contract (normative)

Dynamic and static builders MUST share clause method names, and clause input adapters MUST follow this behavior:

- `filter(...)` MUST accept `Condition` and `Option<Condition>` inputs; `None` MUST omit `WHERE`
- `having(...)` MUST accept `Condition` and `Option<Condition>` inputs; `None` MUST omit `HAVING`
- `order_by(...)` MUST accept one or many `OrderByItem` values, including optional items; missing items MUST be ignored and an empty final set MUST omit `ORDER BY`
- `group_by(...)` MUST accept one or many group expressions; an empty final set MUST omit `GROUP BY`
- `limit(...)` and `offset(...)` MUST accept optional scalar inputs; missing values MUST omit the clause
- negative `limit` / `offset` values MUST fail query building with explicit `QueryBuildError` variants
- `all_of([...])` and `any_of([...])` with no present conditions MUST omit the clause that consumes the result

### 2. Optional fields and scalar clause parts

The jOOQ `noField()` idea is useful, but `pgQuill` should apply it carefully.

For clauses like `ORDER BY`, `LIMIT`, or `OFFSET`, Rust already has a very natural alternative:

```rust
let order_by = [
    sort_by_title.then_some(book::title.asc().into_order_by_item()),
    sort_by_year.then_some(book::published_in.desc().into_order_by_item()),
];

let limit = paginated.then_some(50_i64);

let rows = select(book::id)
    .from(book)
    .order_by(order_by)
    .limit(limit)
    .fetch_all(&db)
    .await?;
```

This is cleaner than forcing a universal `no_field()` placeholder into every clause.

Important design recommendation:

- keep `no_field()` internal or advanced-only
- it is reasonable only in non-projecting clauses like `ORDER BY`, `LIMIT`, `OFFSET`
- do not use it to build dynamic `SELECT` lists, because projecting a fake placeholder field would complicate typed row shapes

For dynamic projections, it is better to build the final `Vec<SelectItem>` directly.

That should stay within the same `select(...)` constructor, but it should be treated as a different result-typing path:

- statically known selections should decode to compile-time Rust types
- runtime-built `Vec<SelectItem>` selections should decode to `DynamicRow` by default

Example:

```rust
let mut select_items = vec![users::id.into_select_item()];

if include_name {
    select_items.push(users::name.alias("name").into_select_item());
}

let rows: Vec<DynamicRow> = select(select_items)
    .from(users)
    .fetch_all(&db)
    .await?;

let id: i64 = rows[0].get_index(0)?;
let name: String = rows[0].get_name("name")?;
```

This is the honest trade-off: the query is still SQL-shaped and schema-aware, but once the projected shape becomes runtime-defined, the decoded row shape must also become runtime-defined.

### 3. Optional tables, joins, and paths

The jOOQ `noTable()` and `noPath()` ideas are useful, but Rust can model this more cleanly by constructing the join tree conditionally before attaching it to the query.

Example:

```rust
let from_item = if include_author {
    book
        .join(author)
        .on(book::author_id.equal(author::id))
        .into_from_item()
} else {
    book.into_from_item()
};

let rows = select(book::id)
    .from(from_item)
    .fetch_all(&db)
    .await?;
```

This gives the same result as `noTable()` without introducing a special placeholder whose following `on(...)` clause must later be ignored.

That same pattern should work for generated relationship paths as well. If `pgQuill` later generates explicit path objects, they should still compose through conditional `FromItem` / `JoinTree` assembly rather than introducing a second family of path-specific helper spellings.

That is a better fit for the rest of the DSL: one compositional model, not multiple equivalent convenience APIs.

### 4. What `pgQuill` should copy from jOOQ, and what it should not

`pgQuill` should absolutely copy these ideas:

- every clause fragment is a first-class AST value
- query fragments can be built outside the final statement
- empty dynamic fragments should omit the clause rather than render broken SQL
- dynamic SQL should work across `SELECT`, `FROM`, `JOIN`, `WHERE`, `ORDER BY`, and `WITH`

`pgQuill` should not copy these details literally:

- Java-style sentinel objects as the default dynamic composition mechanism
- APIs where `.on(...)` is written after an absent join target and then silently ignored
- `no_field()` as a normal way to shape projected result rows

The finalized direction should be:

- Rust-native dynamic composition first
- keep `no_field()` internal or advanced-only for non-projecting clauses
- prefer `Option`, `Vec`, and conditional AST assembly over `no_table()` / `no_path()`
- use `all_of([...])` and `any_of([...])` as the canonical public way to build optional predicates
- do not make `no_table()` or `no_path()` part of the main documented DSL

## F. Potential Future Ideas

Validated `sql!` / `sql_file!` surfaces are intentionally outside the current implementation scope.

That idea is documented separately in [FUTURE_SQL.md](./FUTURE_SQL.md).
It is only a future idea. It should not be implemented now, and it should not be treated as part of the current README scope at all.

## Recommended DSL Examples

### Basic select

```rust
let q = select((users::id, users::name))
    .from(users)
    .filter(users::id.equal(1_i64));
```

### SELECT without `FROM`

Because PostgreSQL allows `SELECT` without a `FROM` clause, `pgQuill` should support it directly:

```rust
let row: (time::OffsetDateTime, String, String) = select((
        now(),
        val("abc"),
        val("xyz").alias("one"),
    ))
    .fetch_one(&db)
    .await?;
```

This should render to:

```sql
SELECT now(), 'abc', 'xyz' AS one
```

### DISTINCT

`pgQuill` should support ordinary `DISTINCT` directly in the DSL.

```rust
let titles = select_distinct(book::title)
    .from(book)
    .fetch_all(&db)
    .await?;
```

### DISTINCT ON

Because `pgQuill` is PostgreSQL-only, `DISTINCT ON` should be a first-class feature rather than an escape hatch.

Recommended shape:

```rust
let rows = select_distinct_on(
        (book::language_id,),
        (book::language_id, book::title),
    )
    .from(book)
    .order_by((book::language_id.asc(), book::title.asc()))
    .fetch_all(&db)
    .await?;
```

This should render to:

```sql
SELECT DISTINCT ON (book.language_id)
  book.language_id,
  book.title
FROM book
ORDER BY book.language_id, book.title
```

Implementation note:

- the AST should model `DISTINCT` as part of `SELECT`, not as a post-processing flag
- `DISTINCT ON` should carry a typed expression list
- the first argument to `select_distinct_on(...)` should always be list-shaped, even for one expression
- the DSL should expose `select(...)`, `select_distinct(...)`, and `select_distinct_on(...)` as the canonical `SELECT` constructors
- `pgQuill` should render the query faithfully and leave PostgreSQL to enforce the rule that `ORDER BY` should begin with the `DISTINCT ON` expressions

### Insert with `RETURNING`

```rust
let inserted = insert_into(users)
    .columns((users::email, users::name))
    .values(("a@example.com", "Alice"))
    .returning((users::id, users::created_at))
    .fetch_one(&db)
    .await?;
```

`returning(selection)` should turn a DML statement into a typed row-producing statement. That means it should use the same generic fetch terminals as `SELECT`:

- `fetch_all(...)`
- `fetch_one(...)`
- `fetch_optional(...)`
- `fetch_into::<T>(...)`
- `fetch_one_into::<T>(...)`
- `fetch_optional_into::<T>(...)`

By contrast, DML without `returning(...)` should use `execute(&db)` and return only the affected-row count.

### Update

```rust
let affected = update(users)
    .set(users::name, "Updated Alice")
    .filter(users::id.equal(1_i64))
    .execute(&db)
    .await?;
```

### ON CONFLICT

`ON CONFLICT` is a required Phase 1 feature.

Scope for conflict targets in the initial implementation:

- only typed generated `ON CONSTRAINT` targets are supported
- `UNIQUE` and primary-key constraints (`pg_constraint.contype IN ('u', 'p')`) are emitted as typed conflict targets
- column-list conflict targets are out of scope for now
- expression index conflict targets are out of scope for now
- partial-index predicate conflict targets are out of scope for now

The code generator MUST emit typed constraint constants for eligible unique and primary-key constraints so callers avoid stringly-typed conflict targets.

Example generated shape:

```rust
pub mod users_constraints {
    pub const users_email_key: UniqueConstraint = ...;
}
```

Required actions:

- `DO NOTHING`
- `DO UPDATE`

`DO NOTHING` example:

```rust
let affected = insert_into(users)
    .columns((users::email, users::name))
    .values((email, name))
    .on_conflict_on_constraint(users_constraints::users_email_key)
    .do_nothing()
    .execute(&db)
    .await?;
```

`DO UPDATE` example:

```rust
let affected = insert_into(users)
    .columns((users::email, users::name))
    .values((email, name))
    .on_conflict_on_constraint(users_constraints::users_email_key)
    .do_update_set()
        .set(users::name, excluded(users::name))
        .set(users::updated_at, now())
        .filter(excluded(users::name).is_distinct_from(users::name))
    .execute(&db)
    .await?;
```

Design rules:

- upsert semantics belong to the SQL DSL (`insert ... on conflict ...` and `merge`); entity-level `save()` semantics are outside `pgQuill`'s scope (delegated to `toasty`; see *Runtime Integration And Toasty Pairing*)
- `MERGE` and `ON CONFLICT` remain distinct statement families in the AST and renderer
- `DO UPDATE ... WHERE ...` MUST be supported in Phase 1
- `excluded(...)` MUST only be valid inside `ON CONFLICT ... DO UPDATE` assignments and predicates
- when a table has no eligible generated unique/primary-key constraint constants, typed `on_conflict_on_constraint(...)` is unavailable for that table; raw-string conflict targets are out of scope for the initial implementation

### CTE

```rust
let active_users = select((users::id, users::name))
    .from(users)
    .filter(users::deleted_at.is_null())
    .cte("active_users");

let au = active_users.reference();

let q = with(active_users)
    .select((au.id, au.name))
    .from(au);
```

Typing rule for non-recursive CTE references:

- `cte(...).reference()` should expose a typed field set derived from the CTE output schema
- simple selected columns keep their output names, so `users::id` becomes `au.id`
- aliased expressions expose the alias name, so `expr.alias("display_name")` becomes `au.display_name`
- output field types should be the same as the selected expression types
- if a selected expression has no stable output name, it should have to be aliased before the CTE can be referenced by named fields
- if two selected expressions would expose the same output name, CTE creation should fail until the query uses distinct aliases

Example:

```rust
let active_users = select((
        users::id,
        concat(users::first_name, users::last_name).alias("display_name"),
    ))
    .from(users)
    .filter(users::deleted_at.is_null())
    .cte("active_users");

let au = active_users.reference();

let q = with(active_users)
    .select((au.id, au.display_name))
    .from(au);
```

### WITH RECURSIVE

`pgQuill` should support recursive CTEs as real first-class AST nodes, not as raw SQL escape hatches.

The Rust shape should be:

```rust
let t = recursive_cte("t").columns((
    cte_column::<i64>("id"),
    cte_column::<String>("name"),
    cte_column::<String>("path"),
));

let t_ref = t.reference();

let t = t.define(
    select((
        directory::id,
        directory::label,
        directory::label,
    ))
    .from(directory)
    .filter(directory::parent_id.is_null())
    .union_all(
        select((
            directory::id,
            directory::label,
            concat(t_ref.path, "\\", directory::label),
        ))
        .from(t_ref)
        .join(directory)
        .on(t_ref.id.equal(directory::parent_id)),
    ),
);

let t_rows = t.reference();

let rows = with_recursive(t)
    .select((t_rows.id, t_rows.name, t_rows.path))
    .from(t_rows)
    .fetch_all(&db)
    .await?;
```

This design has a few important properties:

- the recursive CTE columns are declared explicitly and typed
- the non-recursive term and recursive term are both normal query AST nodes
- `union_all(...)` is part of the AST, not a string splice
- the self-reference is explicit through `t.reference()`

Typing rule for recursive CTE references:

- `.columns(...)` is the sole source of exposed recursive CTE field names and types
- `t.reference()` should expose fields from that declared column list, so `cte_column::<i64>("id")` becomes `t_ref.id`
- the non-recursive term and recursive term must each produce the same arity as the declared column list
- each selected expression in both terms must be type-compatible with the corresponding declared CTE column
- top-level aliases inside the non-recursive or recursive term do not define the recursive CTE output schema
- if such aliases are present, they should be ignored for recursive CTE schema definition
- the preferred style is to declare recursive CTE column names in `.columns(...)` and avoid redundant term aliases

That keeps the API close to both PostgreSQL and jOOQ, while still staying Rust-friendly.

Longer-term, the recursive CTE API should leave room for PostgreSQL-specific extensions documented on the `WITH` page, especially:

- search ordering columns
- cycle detection columns
- materialization controls where PostgreSQL allows them

Those do not all need to be implemented in the early phases, but the AST should not block them.

### LATERAL

`LATERAL` should cover the full PostgreSQL pattern:

- correlated subqueries in `FROM`
- `JOIN LATERAL (...) ON true`
- `LEFT JOIN LATERAL (...) ON true`
- `CROSS JOIN LATERAL (...)`
- set-returning functions used as per-row table expressions

```rust
let u = users.alias("u");

let recent_post = lateral(
    select((posts::id, posts::title))
        .from(posts)
        .filter(posts::author_id.equal(u.col(users::id)))
        .order_by(posts::created_at.desc())
        .limit(1),
)
.alias("recent_post");

let q = select((u.col(users::name), recent_post.col(posts::title)))
    .from(u)
    .left_join(recent_post)
    .on_true();
```

`lateral(subquery)` should be the canonical public model. `pgQuill` should not introduce a second family of dedicated methods like `join_lateral(...)` or `left_join_lateral(...)`.

That means the core cases should map like this:

- `JOIN LATERAL (...) ON true` -> `.join(lateral_target).on_true()`
- `LEFT JOIN LATERAL (...) ON true` -> `.left_join(lateral_target).on_true()`
- `CROSS JOIN LATERAL (...)` -> `.cross_join(lateral_target)`

Top-N-per-group queries should therefore be natural:

```rust
let u = users.alias("u");

let top_orders = lateral(
    select(orders::all())
        .from(orders)
        .filter(orders::user_id.equal(u.col(users::id)))
        .order_by(orders::amount.desc())
        .limit(3),
)
.alias("top_orders");

let q = select((u.col(users::id), top_orders.col(orders::id)))
    .from(u)
    .join(top_orders)
    .on_true();
```

For set-returning functions, `pgQuill` should treat them as table expressions too. When the function result needs named output columns, the derived table should declare its output schema explicitly, just like recursive CTEs and `values_table(...)` do.

The intended public model should be:

- `table_function(expr)` converts a set-returning function call into a table expression
- `.columns(...)` declares the output schema of that table expression when named columns are needed
- `derived_column::<T>(name)` declares one typed output column in that schema

These should be treated as real public DSL building blocks, not just example pseudocode.

Typed table-function declaration contract:

- code generation MUST NOT generate database function wrappers automatically
- `pgQuill` MUST expose an explicit typed declaration API for table functions (macro or builder-based)
- declared function signatures MUST drive argument typing and table-function output typing
- declared table functions MUST integrate with `table_function(...)` and `lateral(...)` without requiring raw SQL strings

Recommended direction:

```rust
let u = users.alias("u");
let item = derived_column::<serde_json::Value>("item");

let tags = lateral(
    table_function(jsonb_array_elements(u.col(users::tags)))
        .columns((item,)),
)
.alias("tags");

let q = select((u.col(users::id), tags.col(item)))
    .from(u)
    .cross_join(tags);
```

The same pattern should cover row-dependent functions like `generate_series(...)`:

- wrap the set-returning function in `table_function(...)`
- declare output columns with typed derived-column descriptors
- use `cross_join(...)`, `join(...).on_true()`, or `left_join(...).on_true()` depending on SQL semantics

In other words, `lateral(...)` should not be limited to subqueries. It should work for any table expression whose evaluation may depend on rows from items that appear earlier in `FROM`.

### JSONB

```rust
let q = select(users::id)
    .from(users)
    .filter(
        users::profile
            .contains_json(json!({"role": "admin"}))
            .and(users::profile.path_text(["address", "city"]).equal("Bangkok")),
    );
```

### HSTORE

`pgQuill` should also support PostgreSQL `hstore` directly, because it is already part of the current Java codebase through jOOQ forced type mappings.

Recommended default Rust mapping:

```rust
std::collections::HashMap<String, Option<String>>
```

That preserves:

- native compatibility with `postgres-types`
- nullable values, which `hstore` allows
- straightforward encode/decode behavior

Recommended operator/function surface:

- `contains_hstore(...)` -> `@>`
- `contained_in_hstore(...)` -> `<@`
- `concat_hstore(...)` -> `||`
- `key("foo")` -> `->`
- `contains_key("foo")` -> `?`
- `delete_key("foo")`

Example:

```rust
let q = select(account::row_id)
    .from(account)
    .filter(
        account::attributes
            .contains_key("kyc_status")
            .and(account::attributes.key("kyc_status").equal("approved")),
    );
```

And for containment:

```rust
let q = select(account::row_id)
    .from(account)
    .filter(
        account::attributes.contains_hstore(hstore([
            ("tier", Some("vip")),
            ("region", Some("apac")),
        ])),
    );
```

Implementation note:

- `hstore` should be treated as a first-class PostgreSQL extended type, similar in spirit to `jsonb`
- the generator should preserve `hstore` column metadata explicitly
- `tokio-postgres` / `postgres-types` native support should be used directly where available
- if a project does not enable `hstore` support, generated code should fail clearly rather than silently mapping it to plain text

### CITEXT

`pgQuill` should also support PostgreSQL `citext` directly, because the current Java codebase already uses `citext` bindings in generated jOOQ code.

Recommended default Rust mapping:

```rust
String
```

Important PostgreSQL behavior:

- `citext` stores the original text value
- equality comparison is case-insensitive
- `LIKE` and `ILIKE` are both case-insensitive on `citext`
- if the application needs case-sensitive behavior, it should cast to `text`

That means code like this should be natural:

```rust
let row = select(users::row_id)
    .from(users)
    .filter(users::email.equal("Alice@Example.com"))
    .fetch_optional(&db)
    .await?;
```

and it should behave case-insensitively because the underlying column type is `citext`.

Likewise:

```rust
let rows = select(users::row_id)
    .from(users)
    .filter(users::email.like("%@example.com"))
    .fetch_all(&db)
    .await?;
```

should also be case-insensitive for a `citext` column, because that is how PostgreSQL defines the type's operator behavior.

Implementation note:

- `citext` should be treated as a first-class PostgreSQL extended type, not silently collapsed into plain `text`
- generated schema metadata should preserve that a column is `citext`, even if the default Rust runtime type is `String`
- the DSL should allow explicit casts to `text` when the user needs case-sensitive comparison semantics

This is also one of the places where PostgreSQL-only scope is beneficial: `pgQuill` can model `citext` honestly instead of pretending it is just a generic string.

### ARRAY

```rust
let q = select(posts::id)
    .from(posts)
    .filter(posts::tags.contains(array(["rust", "postgres"])));
```

### DOMAIN

`pgQuill` should support PostgreSQL `DOMAIN` types explicitly.

This matters because domains are not just aliases. They can carry:

- `NOT NULL`
- default values
- `CHECK` constraints
- semantic meaning above the base type

Recommended `pgQuill` behavior:

- preserve domain identity in generated schema metadata
- know the underlying base type for expression typing and encoding
- allow a domain to map to a user-provided Rust type when the domain is semantically important

Example:

```sql
CREATE DOMAIN email_address AS citext
CHECK (VALUE LIKE '%@%');
```

Recommended codegen override:

```toml
[pgquill.codegen.schemas.public.domains.email_address]
rust_type = "crate::types::EmailAddress"
```

The application can then provide its own Rust type:

```rust
use postgres_types::{FromSql, ToSql};

#[derive(Debug, Clone, ToSql, FromSql)]
#[postgres(transparent)]
pub struct EmailAddress(pub String);
```

If the application wants early validation or normalization, it can keep that logic in its own type:

```rust
impl EmailAddress {
    pub fn new(value: String) -> Result<Self, EmailError> {
        let value = value.trim().to_lowercase();

        if !value.contains('@') {
            return Err(EmailError::InvalidFormat);
        }

        Ok(Self(value))
    }
}
```

A column using that domain should remain distinguishable from plain `citext` in the generated schema:

```rust
pub const email: Column<crate::types::EmailAddress> = ...;
```

This gives `pgQuill` room to support two layers:

- SQL typing: this is a domain over a base PostgreSQL type
- Rust typing: this maps to a generated domain newtype by default, or a user-provided Rust type when overridden

Recommended default behavior:

- if no explicit `rust_type` override is configured, code generation MUST emit and use a domain-specific Rust newtype
- generated table columns that use a domain MUST expose that domain newtype in their typed column descriptors
- preserve the domain identity in metadata and AST typing for both generated and user-overridden Rust types
- allow per-domain `rust_type = "..."` overrides for stronger domain modeling
- the user-provided Rust type should implement `ToSql` / `FromSql`, and for simple transparent wrappers `#[postgres(transparent)]` is usually the right fit

Important design note:

- domain constraints are enforced by PostgreSQL, not reimplemented by `pgQuill`
- `pgQuill` should model domains faithfully, but it does not need to duplicate all server-side validation logic
- if the application wants early validation or normalization, that should live in the user-provided Rust type rather than in generated code
- casts between domain and base type should remain expressible in the DSL

This is another place where PostgreSQL-first scope pays off. A generic SQL builder would be tempted to erase domains completely, but `pgQuill` should keep them visible.

### Composite types

```rust
let q = select(orders::shipping_address.field(order_address::city))
    .from(orders)
    .filter(orders::shipping_address.field(order_address::country).equal("TH"));
```

### MERGE

Because `pgQuill` is PostgreSQL-only, `MERGE` should be a first-class statement type rather than an afterthought.

Recommended shape:

```rust
let source_rows: Vec<(String, String)> = vec![
    ("acct_001".to_string(), "Alice".to_string()),
    ("acct_002".to_string(), "Bob".to_string()),
];

let src = values_table(
    (account::external_id, account::name),
    source_rows,
)
.alias("src");

let affected = merge_into(account)
    .using(src)
    .on(account::external_id.equal(src.col(account::external_id)))
    .when_matched_then_update()
        .set(account::name, src.col(account::name))
        .set(account::updated_at, now())
    .when_not_matched_then_insert((
        account::external_id,
        account::name,
        account::created_at,
    ))
        .values((
            src.col(account::external_id),
            src.col(account::name),
            now(),
        ))
    .execute(&db)
    .await?;
```

This mirrors PostgreSQL's structure well:

- `MERGE INTO target`
- `USING source`
- `ON join_condition`
- one or more `WHEN ... THEN ...` clauses

For `VALUES` sources, `pgQuill` should make the source shape explicit:

- `values_table(columns, rows)` should be the initial public API
- `columns` defines the source column names and types
- when real table columns are used there, only their names and types are borrowed, not table identity
- `src.col(descriptor)` should mean "look up the corresponding field in the synthetic source schema declared by `values_table(...)`"
- `src.col(descriptor)` should only be valid for descriptors that were part of that declared source schema
- `src.col(account::external_id)` must not imply that `src` is linked to the real `account` table; `account::external_id` is only acting as a borrowed schema descriptor there
- `rows` must match the declared column tuple in arity and Rust types

So for:

```rust
values_table((account::external_id, account::name), source_rows)
```

the shape of `source_rows` should be an iterator of 2-tuples:

```rust
Vec<(ExternalIdRustType, String)>
```

In the example above, `String` is only illustrative. The real Rust type of the first tuple element should be whatever Rust type `account::external_id` maps to.

`pgQuill` should support at least these PostgreSQL branches:

- `when_matched_then_update()`
- `when_matched_then_delete()`
- `when_not_matched_then_insert(...)`
- `when_matched_and(condition)...`
- `when_not_matched_and(condition)...`

Nice follow-up support:

- `when_not_matched_by_source_then_delete()` if PostgreSQL syntax and target version support it
- `RETURNING` if PostgreSQL allows it for the chosen merge form and runtime target

Important design points:

- `MERGE` should be its own AST node, not lowered early into `INSERT ... ON CONFLICT`
- `MERGE` and `INSERT ... ON CONFLICT` should both exist; they solve overlapping but not identical problems
- the branch ordering should be preserved exactly, because PostgreSQL evaluates `WHEN` clauses in order
- the source should be able to be a table, subquery, values table, or CTE reference

This is one of the places where PostgreSQL-only scope is a major advantage. `pgQuill` can expose `MERGE` honestly without pretending it is portable SQL.

These examples are intentionally SQL-shaped. That should remain the design center.

## Codegen Configuration Schema And Precedence

`pgQuill` should define one canonical configuration schema for code generation, with strict validation and deterministic precedence.

Recommended top-level shape:

```toml
[pgquill.codegen]
default_schema = "public"

[pgquill.codegen.schemas.public]

[pgquill.codegen.schemas.public.domains.email_address]
rust_type = "crate::types::EmailAddress"
```

Required precedence order (lowest to highest):

1. built-in defaults
2. schema-level defaults
3. table/domain-specific overrides

Required initial key set:

- `pgquill.codegen.default_schema: String`
- `pgquill.codegen.schemas.<schema>.domains.<domain>.rust_type: String`

Configuration merge contract:

- scalar values use highest-precedence override (`table` > `schema defaults` > built-in)
- array values use highest-precedence replacement (not concatenation)
- absent keys inherit from the next lower precedence level
- schema defaults apply to every table in that schema unless a table-level override is present

Validation and fail-fast rules:

- unknown config keys MUST fail code generation
- `default_schema` MUST exist in the introspected catalog
- configured schemas/tables/domains that do not exist in the introspected catalog MUST fail code generation
- configured columns that do not exist in the target table MUST fail code generation with `CodegenError::UnknownConfiguredColumn`
- symbol collisions in generated output MUST fail with `CodegenError::NamingCollision`
- automatic renaming is out of scope for the initial implementation

Determinism rules:

- effective configuration after merges MUST be deterministic
- generated output for the same catalog + same config MUST be stable in ordering and naming
- configuration merge behavior MUST be documented and tested with integration fixtures

## Identifier Normalization And Symbol Mapping

Code generation should define one deterministic mapping from SQL identifiers to Rust symbols.

Required contract:

- SQL rendering MUST continue to use catalog identifiers and SQL aliases with PostgreSQL quoting rules
- Rust symbols MUST be derived deterministically from catalog identifiers
- non-identifier characters in Rust symbol candidates MUST be normalized to `_`
- leading numeric characters MUST be prefixed to produce valid Rust identifiers
- Rust keywords MUST be emitted using raw identifiers (for example `r#type`)
- when two generated symbols normalize to the same Rust item in the same namespace, code generation MUST fail with `CodegenError::NamingCollision`
- auto-suffix renaming strategies are out of scope for the initial implementation
- schema snapshot output SHOULD include both SQL identifier and generated Rust symbol for traceability

## Generated Code Model

A build script should introspect the live database and generate Rust code and a machine-readable schema snapshot.

Build-script integration:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    println!("cargo:rerun-if-changed=pgquill.toml");
    println!("cargo:rerun-if-env-changed=DATABASE_URL");

    let out_dir = std::path::PathBuf::from(std::env::var("OUT_DIR")?);

    pgquill_codegen::Generator::from_env()?
        .write_to_dir(&out_dir)?;

    Ok(())
}
```

The generator writes multiple files instead of one monolithic output:

```text
$OUT_DIR/
  pgquill_schema/
    mod.rs
    public/
      mod.rs
      users.rs
      posts.rs
      order_address.rs
    __pgquill_schema.json
```

Schema snapshot contract (`__pgquill_schema.json`):

- snapshot format MUST include a top-level `format_version`
- snapshot format MUST include `default_schema` and a stable list of schema models
- each schema item MUST include SQL identifiers and normalized Rust symbol mappings
- table entries MUST include columns, nullability, type identity, keys, and relationship metadata required for deterministic regeneration
- output ordering in all arrays MUST be stable and deterministic
- snapshot content used for determinism checks MUST exclude environment-volatile identifiers (for example runtime-assigned OIDs)
- breaking snapshot format changes MUST increment `format_version`

Each schema gets its own subdirectory, and each table gets its own file. This keeps generated code readable, produces better compiler diagnostics (errors point to specific table files), and avoids IDE slowdowns from a single massive generated file.

The application crate includes the generated root module:

```rust
pub mod schema {
    include!(concat!(env!("OUT_DIR"), "/pgquill_schema/mod.rs"));
}
```

This build integration model should mirror proven workspace patterns used for protobuf and GraphQL generation: write deterministic artifacts to `OUT_DIR`, declare explicit `cargo:rerun-if-*` triggers, and keep generated code as build products.

Generated output could look like:

```text
src/schema/
  mod.rs
  public/
    mod.rs
    users.rs
    posts.rs
    order_address.rs
  __pgquill_schema.json
```

Generated source and snapshots should be deterministic in structure.

At minimum, generation should use stable ordering for:

- schema modules
- tables within a schema
- fields within a generated row
- metadata arrays in the schema snapshot

The goal is stable review diffs and reproducible codegen output from the same catalog state.

Recommended generated items per table:

- a generated table value like `users`
- column constants
- a `Row` struct
- per-table unique/primary-key constraint constants for typed `ON CONFLICT ... ON CONSTRAINT` targets
- primary-key metadata
- stable field ordering and field indexes for decoding
- relationship metadata
- stable type identity keys (schema-qualified type names and category tags), with OIDs only in optional debug metadata that is excluded from determinism checks

Rust has separate namespaces for modules and values, so generated code should expose both:

- a module `users` containing `users::id`, `users::name`, and other column items
- a value `users` representing the table reference used in `.from(users)`

Built-in PostgreSQL functions like `now()` or `concat(...)` should stay as ordinary free functions.

Database functions discovered in the schema should not be generated at all. Code generation should ignore them entirely, and function usage should come from explicit typed declarations in application or support crates.

Example generated shape:

```rust
pub mod public {
    pub mod users {
        pub struct Table;

        pub const id: Column<i64> = ...;
        pub const email: Column<String> = ...;
        pub const profile: Column<serde_json::Value> = ...;
        pub const created_at: Column<time::OffsetDateTime> = ...;

        pub struct Row {
            pub id: i64,
            pub email: String,
            pub profile: serde_json::Value,
            pub created_at: time::OffsetDateTime,
        }
    }

    #[allow(non_upper_case_globals)]
    pub const users: users::Table = users::Table;
}
```

Relationships should also be generated as metadata, even if the DSL does not immediately auto-join on them:

- foreign key source and target
- cardinality hints
- named relationship constants

That gives room for later ergonomic APIs without forcing ORM semantics today.

## Catalog Introspection Strategy

Use `pg_catalog` as the primary source of truth.

`information_schema` is helpful, but not rich enough for a PostgreSQL-native tool that cares about arrays, enums, composites, OIDs, generated columns, and many database-specific details.

Recommended catalog sources:

- `pg_namespace` for schemas
- `pg_class` for tables, views, materialized views, composite backing relations
- `pg_attribute` for columns and composite fields
- `pg_type` for scalar, array, domain, enum, composite, pseudo-type metadata
- `pg_enum` for enum labels
- `pg_constraint` for primary keys, foreign keys, unique, checks
- `pg_attrdef` for column defaults
- `pg_index` for index metadata
- `pg_description` later for documentation comments

`pg_class` still has to be queried because PostgreSQL stores several relation kinds there, but code generation should emit first-class schema items only for supported relation kinds.

In the current scope:

- ordinary tables should be generated
- composite backing relations should be used as metadata for composite types
- views and materialized views should be ignored by code generation

Recommended generator output model:

- internal `SchemaModel`
- Rust code emission
- optional JSON schema snapshot for tests, debugging, and future tooling

This separation will pay off. It lets the DSL, code generator, and any future tooling share one metadata model.

## tokio-postgres Runtime Design

Runtime should stay thin.

Suggested responsibilities:

- render AST to SQL text
- collect bind values in order
- execute through `tokio-postgres`
- decode rows into tuples, generated structs, or user structs
- carry enough parameter type information to use `prepare_typed` or `query_typed` when PostgreSQL cannot infer bind types reliably

Suggested runtime API:

```rust
pub trait Executor {
    async fn execute(&self, query: &RenderedQuery) -> Result<u64, PgQuillError>;
    async fn query_one<T>(&self, query: TypedQuery<T>) -> Result<T, PgQuillError>;
    async fn query_optional<T>(&self, query: TypedQuery<T>) -> Result<Option<T>, PgQuillError>;
    async fn query_all<T>(&self, query: TypedQuery<T>) -> Result<Vec<T>, PgQuillError>;
}
```

Executor coverage requirement:

- `Executor` MUST be implemented for `tokio_postgres::Client`
- `Executor` MUST be implemented for `tokio_postgres::Transaction<'_>`

Transaction usage examples:

```rust
let tx = db.transaction().await?;

let user_id = insert_into(users)
    .columns((users::email, users::name))
    .values((email, name))
    .returning(users::id)
    .fetch_one(&tx)
    .await?;

update(accounts)
    .set(accounts::owner_user_id, user_id)
    .filter(accounts::row_id.equal(account_id))
    .execute(&tx)
    .await?;

tx.commit().await?;
```

```rust
let tx = db.transaction().await?;

let maybe_user = select(users::all())
    .from(users)
    .filter(users::id.equal(user_id))
    .fetch_optional(&tx)
    .await?;

if maybe_user.is_none() {
    tx.rollback().await?;
    return Err(AppError::NotFound);
}

tx.commit().await?;
```

Practical implementation detail:

- render to `String`
- hold bind values in a stable arena like `Vec<Box<dyn ToSql + Sync>>`
- pass `&[&(dyn ToSql + Sync)]` to `tokio-postgres`

Recommended execution behavior:

- use normal prepared statements when PostgreSQL type inference is obvious
- use `prepare_typed` by default once the AST can produce precise parameter types
- keep `query_typed` available as a fallback path for awkward statements or generated raw-SQL wrappers
- runtime should reject unsupported PostgreSQL server versions with `ExecutionError::UnsupportedServerVersion`

SQL rendering contract:

- rendered SQL MUST use PostgreSQL positional bind placeholders (`$1`, `$2`, ...)
- rendered SQL MUST always quote identifiers (schemas, tables, columns, aliases) using PostgreSQL identifier quoting rules
- rendered SQL MUST always schema-qualify table references
- these rules are part of the deterministic rendering contract for correctness and predictable SQL text output

Identifier qualification matrix:

- base table references in `FROM`, `JOIN`, `UPDATE`, and `DELETE` targets MUST be schema-qualified
- aliased base tables MUST render as schema-qualified base relation plus alias, and subsequent column references MUST use the alias
- unaliased base-table columns MUST use the table identifier selected by the renderer for that relation instance
- CTE references MUST NOT be schema-qualified
- derived tables, `values_table(...)`, and declared table-function relations MUST NOT be schema-qualified
- column aliases and projection aliases MUST never be schema-qualified

Why positional placeholders matter:

- they match `tokio-postgres` and PostgreSQL native prepared statement conventions directly
- they preserve stable parameter ordering from AST bind collection to execution
- they avoid introducing a second placeholder dialect in generated SQL

Nice extras later:

- prepared statement cache keyed by rendered SQL
- query logging / debug formatting
- explain hooks
- optional tracing integration

## Result Typing

The selected shape should define the SQL output schema, and the terminal method should choose how that schema is decoded.

For statically known selections, the decoded Rust shape should also be statically known.

For runtime-built projections like `Vec<SelectItem>`, the query should still use the same public `select(...)` constructor, but it should decode to `DynamicRow` by default.

Examples:

- `select((now(), val("abc"), val("xyz").alias("one"))).fetch_one(&db)` -> `(OffsetDateTime, String, String)`
- `select(users::id).from(users).fetch_all(&db)` -> `Vec<i64>`
- `select((users::id, users::name)).from(users).fetch_all(&db)` -> `Vec<(i64, String)>`
- `select(users::all()).from(users).fetch_all(&db)` -> `Vec<users::Row>`
- `select(dynamic_items).from(users).fetch_all(&db)` -> `Vec<DynamicRow>`
- `insert_into(users).columns(...).values(...).returning((users::id, users::created_at)).fetch_one(&db)` -> `(i64, OffsetDateTime)`

That means:

- one statically known selection can support more than one decode target, as long as the terminal is compatible with the selected schema
- runtime-built projections should use runtime row decoding unless the caller explicitly chooses a DTO mapping path like `fetch_into::<T>()`

Validation boundary for result typing:

- static selections MUST enforce selection type compatibility at compile time
- dynamic selections MUST validate mapping at runtime
- DTO decode failures from runtime validation should return `DecodeError::DtoMappingFailed` (or a more specific decode variant)

Terminal row-count semantics:

- `fetch_one(...)` MUST fail when zero rows are returned
- `fetch_one(...)` MUST fail when more than one row is returned
- `fetch_optional(...)` MUST return `Ok(None)` when zero rows are returned
- `fetch_optional(...)` MUST fail when more than one row is returned
- the same semantics apply to `fetch_one_into(...)` and `fetch_optional_into(...)`
- the same semantics apply to row-producing DML terminals (`insert/update/delete/merge ... returning`)

There should be no bare `select_all()` constructor in the fluent DSL. Typed generated row decoding should use the explicit one-table shape `select(table::all()).from(table)`.

More generally, `pgQuill` should avoid generic query forms that weaken type visibility. A reader should usually be able to look at the DSL and make a good guess about both:

- the SQL projection that will be rendered
- the Rust shape that will be decoded

That type-oriented readability is one of the main goals of the project.

For statically known selections, you will want a `Selection` trait:

```rust
pub trait Selection {
    type Rust;
    fn fields(&self) -> Vec<FieldMeta>;
}
```

Then `Select<S>` can expose:

```rust
impl<S: Selection> Select<S> {
    pub async fn fetch_all(self, db: &impl Executor) -> Result<Vec<S::Rust>, PgQuillError>;
    pub async fn fetch_one(self, db: &impl Executor) -> Result<S::Rust, PgQuillError>;
    pub async fn fetch_optional(self, db: &impl Executor) -> Result<Option<S::Rust>, PgQuillError>;
}
```

This keeps call sites clean.

For runtime-built projections, the implementation should use a separate dynamic query wrapper internally while keeping the same public `select(...)` constructor.

Conceptually, the implementation should look like this:

```rust
pub struct Select<S> {
    ast: SelectAst,
    _selection: PhantomData<S>,
}

pub struct DynamicSelect {
    ast: SelectAst,
    plan: Arc<DynamicSelectionPlan>,
}

pub struct ReturningDml<S> {
    ast: DmlAst,
    _selection: PhantomData<S>,
}

pub struct DynamicRow {
    row: tokio_postgres::Row,
    plan: Arc<DynamicSelectionPlan>,
}
```

The public API should still look uniform:

```rust
select(users::all())
    .from(users)
    .filter(users::id.equal(id))
    .fetch_optional(&db)
```

And also:

```rust
select(dynamic_items)
    .from(users)
    .fetch_all(&db)
```

But internally:

- `select(static_selection)` should produce a typed `Select<S>`
- `select(dynamic_items)` should produce a `DynamicSelect`

Likewise, `insert(...).returning(selection)`, `update(...).returning(selection)`, `delete(...).returning(selection)`, and future row-producing `MERGE` forms should become a typed `ReturningDml<S>`-style wrapper that exposes the generic fetch terminals.

That gives `pgQuill` the right balance:

- compile-time gating for obvious query-shape eligibility
- no need for users to name intermediate wrapper types
- no need to expose a large public hierarchy of step or state types
- only minimal runtime sanity checks as a backstop

`DynamicRow` should be a thin wrapper over native `tokio-postgres` row decoding plus a selection plan. It should not introduce a duplicate codec layer.

Recommended `DynamicRow` access patterns:

- `get_index(idx)` for positional access
- `get_name(name)` for unique aliased or named output fields
- `get_column(column)` for simple generated columns that were selected directly and appear unambiguously

Recommended rules:

- `get_index(...)` is the universal fallback
- `get_name(...)` should fail on duplicate output names
- `get_column(...)` should work only for direct generated columns, not arbitrary expressions
- if a computed expression is meant to be read by name or mapped into a DTO, it should be explicitly aliased
- `fetch_into::<T>()` on a dynamic selection may exist, but it should be runtime-validated rather than compile-time-guaranteed

Recommended terminal split:

- `Select<S>`:
  - `fetch_all`
  - `fetch_one`
  - `fetch_optional`
  - `fetch_into`
  - `fetch_one_into`
  - `fetch_optional_into`
- `DynamicSelect`:
  - `fetch_all` -> `Vec<DynamicRow>`
  - `fetch_one` -> `DynamicRow`
  - `fetch_optional` -> `Option<DynamicRow>`
  - `fetch_into`
  - `fetch_one_into`
  - `fetch_optional_into`
- `ReturningDml<S>`:
  - `fetch_all`
  - `fetch_one`
  - `fetch_optional`
  - `fetch_into`
  - `fetch_one_into`
  - `fetch_optional_into`

## Loading Shapes: Rows, Tuples, DTOs

`pgQuill` supports three explicit result shapes. Entity-level loading (a loaded entity you can mutate and save) is NOT a `pgQuill` result shape — that is `toasty`'s job (see *Runtime Integration And Toasty Pairing*).

### 1. Plain generated row loading

For generated table rows:

```rust
let items: Vec<bitcoin_address::Row> = select(bitcoin_address::all())
    .from(bitcoin_address)
    .filter(bitcoin_address::public_key_hash160.is_in(public_key_hash160_list))
    .fetch_all(&db)
    .await?;
```

`select(table::all()).from(table)` decodes into the generated `table::Row` struct. This is the analogue of jOOQ's `selectFrom(TABLE).fetch()` pattern, minus any entity-lifecycle semantics.

### 2. Tuple loading

Rust tuples replace jOOQ's `RecordN` cleanly:

```rust
let items: Vec<(String, String)> = select((
        bitcoin_address::address_compressed,
        bitcoin_address::address_uncompressed,
    ))
    .from(bitcoin_address)
    .filter(bitcoin_address::public_key_hash160.is_in(public_key_hash160_list))
    .fetch_all(&db)
    .await?;
```

### 3. DTO loading

For ad-hoc result models, `pgQuill` maps rows into custom structs.

Note on native driver capabilities:

- `tokio-postgres` natively provides row/column access (`Row::get` / `Row::try_get`) but does not provide built-in full-row struct derive mapping
- `postgres-types` `FromSql` / `ToSql` are value-codec traits for individual PostgreSQL values/types, not full result-row mappers
- so `pgQuill` should keep a lightweight row-mapping layer for DTO decoding

Recommended shape:

```rust
#[derive(pgquill::FromRow)]
struct AddressAndFormat {
    address: String,
    format: BitcoinAddressFormat,
}

let items: Vec<AddressAndFormat> = select((
        bitcoin_address::address_compressed.alias("address"),
        bitcoin_address::format,
    ))
    .from(bitcoin_address)
    .filter(bitcoin_address::public_key_hash160.is_in(public_key_hash160_list))
    .fetch_into(&db)
    .await?;
```

That is the closest analogue to jOOQ's `.fetchInto(AddressAndFormat.class)`.

Recommended method family:

- generic terminals on statically known selections:
  - `fetch_all(&db)` -> `Vec<T>`
  - `fetch_one(&db)` -> `T`
  - `fetch_optional(&db)` -> `Option<T>`
- generic terminals on dynamic projections:
  - `fetch_all(&db)` -> `Vec<DynamicRow>`
  - `fetch_one(&db)` -> `DynamicRow`
  - `fetch_optional(&db)` -> `Option<DynamicRow>`
- `fetch_into::<T>(&db)` -> `Vec<T>`
- `fetch_one_into::<T>(&db)` -> `T`
- `fetch_optional_into::<T>(&db)` -> `Option<T>`

For dynamic projections, the `fetch_into::<T>()` family should be treated as runtime-validated mapping on top of `DynamicRow`, not as compile-time selection typing.

Recommended DTO mapping strictness:

- tuple mapping is positional
- struct mapping is name-based
- extra selected columns MUST fail mapping
- non-optional struct fields MUST map to exactly one output column
- optional struct fields (`Option<T>`) MAY map to `None` when the output column is absent or SQL `NULL`
- duplicate output names for a struct field MUST fail
- incompatible type decoding MUST fail

The struct mapping should support:

- positional mapping for tuples
- name-based mapping for structs following serde naming conventions for `rename` / `rename_all`
- explicit aliases for projected expressions

`pgquill::FromRow` derive scope:

- keep the derive minimal and focused on row mapping
- support `#[serde(rename = "...")]` and container-level `#[serde(rename_all = "...")]`
- do not attempt to replicate the entire serde feature surface in initial phases

### 4. No convenience shortcuts in the initial phases

In the initial phases, `pgQuill` should intentionally keep one canonical construction path:

- `select(...).from(...).filter(...)`

Do not add executor shortcuts (like jOOQ's `fetchOptional(TABLE, condition)`) in the initial phases. They can be reconsidered later, but the starting point should be one predictable and explicit query construction path.

## Runtime Integration And Toasty Pairing

This section defines how `pgQuill` integrates with application code at runtime, and how it pairs with [`tokio-rs/toasty`](https://github.com/tokio-rs/toasty) for entity CRUD. The pairing is the canonical ChainArgos pattern.

### Scope split

- `pgQuill` owns **complex queries**: joins, CTEs, `LATERAL`, window functions, analytical `UPDATE`/`DELETE`, dynamic `WHERE`, `RETURNING`, and any query where SQL shape is the primary concern.
- `toasty` owns **entity CRUD**: creating, loading, saving, and deleting single entities. Hand-written `#[derive(toasty::Model)]` structs, annotated to match schemalane's schema.
- `schemalane` owns **schema and migrations**, authoritatively. Both `pgQuill` (via `pg_catalog` introspection) and `toasty` (via hand-written models) are consumers. `toasty::Db::builder().push_schema()` MUST NOT be called in ChainArgos applications.

Neither library replaces the other. Application code uses each for what it is best at; they share a transaction via the executor trait below.

### `PgExecutor` trait

`pgQuill` runtime terminals MUST take `&mut impl PgExecutor`, where:

```rust
#[async_trait]
pub trait PgExecutor: Send {
    async fn execute_raw(
        &mut self,
        sql: &str,
        params: &[&(dyn ToSql + Sync)],
    ) -> Result<u64, ExecutionError>;

    async fn query_raw(
        &mut self,
        sql: &str,
        params: &[&(dyn ToSql + Sync)],
    ) -> Result<Vec<tokio_postgres::Row>, ExecutionError>;
}
```

Implementations `pgQuill` MUST provide directly:

- `impl PgExecutor for tokio_postgres::Client`
- `impl<'a> PgExecutor for tokio_postgres::Transaction<'a>`

Implementations provided via an opt-in `toasty-bridge` feature in the `pgquill-toasty` crate:

- `impl<'a> PgExecutor for toasty::Transaction<'a>` — requires the upstream patch or our fork (see *Fork strategy and upstream plan* below)
- `impl PgExecutor for toasty::Connection`

### Transaction sharing architecture

A single PostgreSQL transaction, shared between `toasty` and `pgQuill`, is the primary supported atomicity story:

```rust
let mut tx = toasty_db.transaction().await?;
user.save(&mut tx).await?;                       // toasty owns entity lifecycle
pgquill_update(...).execute(&mut tx).await?;     // pgQuill, same transaction
tx.commit().await?;
```

Invariants:

- `toasty::Transaction<'a>` owns a physical `tokio_postgres::Client` connection and a channel to a per-connection task. `PgExecutor::execute_raw` on that transaction MUST dispatch through the same channel so that `pgQuill` operations hit the same connection, inside the same transaction.
- `pgQuill` MUST NOT begin, commit, or roll back transactions itself. Transaction lifecycle is the caller's responsibility (via `toasty` or via raw `tokio_postgres::Transaction`).
- Failure modes follow PostgreSQL semantics: once any statement fails inside a transaction, further statements (from either library) return `current transaction is aborted, commands ignored until end of transaction block`. `pgQuill` and `toasty` MUST propagate this error as-is rather than swallowing it.
- Rollback-on-drop follows `toasty::Transaction`'s existing `Drop` behavior — no new logic required in `pgQuill`.
- There MUST be no unified error type spanning `pgquill::ExecutionError` and `toasty::Error`. Both implement `std::error::Error` and compose through `anyhow::Error` / `thiserror` at the application boundary.

### Fork strategy and upstream plan

As of 2026-04-23, `toasty::Transaction<'_>` does NOT expose raw parameterized SQL execution. The patch this integration depends on spans three crates in the toasty workspace:

- **`toasty-core`** — add `Connection::execute_raw_sql(sql, params)` as a new trait method with a default-`Err(NotSupported)` implementation so non-PostgreSQL drivers remain unaffected.
- **`toasty`** — add `ConnectionOperation::ExecRaw { sql, params, reply }`, route it in the per-connection task, expose `Transaction::execute_raw` and `query_raw` publicly.
- **`toasty-driver-postgresql`** — implement the new `toasty-core` trait method by forwarding to the owned `tokio_postgres::Client`.

Until this patch lands upstream, ChainArgos maintains a fork of `tokio-rs/toasty` with the patch applied. Consuming crates' `Cargo.toml` points at the fork via `git` dependency. When the upstream PR merges, the dependency flips to crates.io with no code changes on the ChainArgos side.

The same patchset IS the PR submitted upstream. The fork is the PR rehearsal, not a permanent divergence.

A custom `toasty` driver is NOT part of this design. `toasty-driver-postgresql` continues to be used as-is (patched for the `execute_raw_sql` trait method). Writing a custom driver does not remove the need for the patches in `toasty` and `toasty-core`, because the per-connection task and `ConnectionOperation` dispatch live above the driver layer.

### Optimistic locking pattern

Neither `pgQuill` nor `toasty` exposes a first-class optimistic-locking feature for PostgreSQL today. The canonical pattern is a `pgQuill` UPDATE with a version predicate:

```rust
// Assumes users has: id BIGINT PRIMARY KEY, version BIGINT NOT NULL, ...
let rows_affected = update(users)
    .set(users::name, val(new_name))
    .set(users::version, users::version + val(1))
    .filter(users::id.equal(user.id).and(users::version.equal(expected_version)))
    .execute(&mut tx)
    .await?;
if rows_affected == 0 {
    return Err(AppError::OptimisticLockMismatch);
}
```

Naming convention: the version column SHOULD be named `version` with Rust type `i64` (matching SQL `BIGINT`). This aligns with the attribute name introduced by `toasty` PR #694 (see *Upstream tracking* below), so future migration to a native `#[version]` derive becomes a pure syntactic change for application code.

### Model freshness across pgQuill writes

When `pgQuill` updates a row that `toasty` holds in memory, `toasty`'s in-memory model does NOT refresh automatically. This is by design — `pgQuill` does not know about `toasty` models (which preserves the clean boundary), and auto-refresh cannot work for bulk updates that touch rows `toasty` never loaded.

The three patterns, in order of preference:

1. **Re-fetch after the pgQuill write.** One extra query. Simple, correct, inexpensive outside hot loops.
2. **Drop the stale binding.** `drop(user)` after the `pgQuill` write turns staleness into a compile error.
3. **Prefer `toasty` for entity updates.** If the update is expressible as `entity.field = new; entity.save(&mut tx).await?`, use `toasty`. Reserve `pgQuill` writes for operations `toasty` cannot express cleanly — bulk `UPDATE ... FROM`, windowed writes, complex `WHERE` on joins. **This is the most important guideline**; it minimizes the situations where freshness matters.

### Upstream tracking

The integration depends on — and contributes to — upstream `tokio-rs/toasty`. These items MUST be tracked as part of the `pgQuill` implementation plan:

- **TODO (PR to open):** Contribute the `execute_raw` / `query_raw` patch described in *Fork strategy and upstream plan* to `tokio-rs/toasty`. Reference upstream issue `#424` (*Missing Use Case: SELECT ... FOR UPDATE / SKIP LOCKED*, which discussion `#713` notes can be alleviated by "ability to send own SQL") and issue `#456` (no `RETURNING` on UPDATE) as motivation beyond ChainArgos's specific need.
- **TODO (PR to track):** `toasty` PR `#694` (*feat: add `#[version]` optimistic concurrency control for DynamoDB*, opened 2026-04-17, stacked on `#674`). When that PR merges, open a follow-up issue or PR to extend `#[version]` support to the SQL drivers (`toasty-driver-postgresql`, `toasty-driver-mysql`, `toasty-driver-sqlite`). The internal machinery (`UpdateByKey.condition`, `ReadModifyWrite`, `Error::condition_failed`) already works across drivers; the follow-up is exposure-of-existing-functionality, not new engine work.
- **TODO (spec update, future):** Once SQL-side `#[version]` lands, update *Optimistic locking pattern* above to prefer the native `#[version]` attribute on `toasty` models. The manual `pgQuill` UPDATE pattern remains supported as an escape hatch for cases the derive does not cover.

### Testing strategy (integration tests)

Per `backend-rust/AGENTS.md`, all integration tests MUST use `testcontainers` with the `postgres` feature. No dependency on the repository's Docker Compose containers. Shared setup belongs in `tests/common/mod.rs`.

The integration-critical scenarios for the `pgquill-toasty` bridge are:

- **Atomic commit of mixed writes.** Load via `toasty`, modify and save, `pgQuill` UPDATE on a different table, commit. Reconnect on a separate connection; assert both writes visible.
- **Rollback on toasty failure.** `toasty` save that violates a constraint, `pgQuill` UPDATE attempted after, commit attempted. Assert the first error propagates, the second returns PG's `current transaction is aborted`, and commit is rejected. Reconnect; assert neither write applied.
- **Rollback on pgQuill failure.** Symmetric: `toasty` save succeeds, `pgQuill` UPDATE fails, commit rejected. Reconnect; neither visible.
- **Drop without commit rolls back.** Start tx, do both kinds of write, `drop(tx)` without commit. Reconnect; neither visible. This exercises `toasty`'s existing `Drop` queuing a rollback through the per-connection task even with the new `ConnectionOperation::ExecRaw` variant in the mix.
- **Single-connection guarantee.** Instrument the fork's `ExecRaw` dispatch to record the per-connection task id. Assert every operation within one `toasty::Transaction` hits the same task. This is the integration-level proof that `PgExecutor::execute_raw` on `toasty::Transaction` does not route to a different connection.
- **Savepoint inside a mixed transaction.** `toasty` save, open a savepoint via `tx.transaction()`, `pgQuill` UPDATE inside the savepoint, rollback to savepoint. Assert the `toasty` save is visible after the outer commit; the `pgQuill` UPDATE is not.

Test setup MUST apply `schemalane` migrations before any `toasty` model use, matching production's authority order. This also acts as a smoke test that `toasty` models stay in sync with `schemalane`: `Db::builder().build()` fails if a model field does not match the table.

## Canonical Error Taxonomy

`pgQuill` should expose one stable top-level error model and avoid ad-hoc per-feature naming.

Recommended baseline:

```rust
pub enum PgQuillError {
    Build(QueryBuildError),
    Execute(ExecutionError),
    Decode(DecodeError),
    Codegen(CodegenError),
}
```

Public API error contract:

- public async query terminals (`fetch_*`, `execute`, `query_*`) MUST return `Result<_, PgQuillError>`
- narrower domain failures SHOULD be exposed through the corresponding top-level variant (`PgQuillError::Decode`, `PgQuillError::Execute`, and so on)

```rust
pub enum QueryBuildError {
    InvalidClauseOrder,
    MissingJoinPredicate,
    InvalidLimitValue { value: i64 },
    InvalidOffsetValue { value: i64 },
    DuplicateOutputName { name: String },
    UnnamedExpressionInNamedContext,
}
```

```rust
pub enum ExecutionError {
    UnsupportedServerVersion { minimum: u16, actual: u16 },
    Database(tokio_postgres::Error),
    Timeout,
    Canceled,
}
```

```rust
pub enum DecodeError {
    ColumnNotFound { name: String },
    DuplicateColumnName { name: String },
    TypeMismatch {
        column: String,
        expected: &'static str,
        actual: String,
    },
    NullForNonNullable { column: String },
    DtoMappingFailed { target: &'static str, reason: String },
    UnexpectedRowCount {
        expected: RowCountExpectation,
        actual: usize,
    },
}
```

```rust
pub enum RowCountExpectation {
    ExactlyOne,
    ZeroOrOne,
}
```

```rust
pub enum CodegenError {
    UnsupportedPostgresVersion { minimum: u16, actual: u16 },
    UnknownConfiguredColumn { table: String, column: String },
    NamingCollision { symbol: String, context: String },
}
```

This taxonomy should be treated as stable API contract guidance for the phased implementation plan, even if concrete enum names evolve slightly.

Config and naming policy clarifications:

- code generation config schema and precedence are defined in `Codegen Configuration Schema And Precedence`
- naming collisions MUST fail fast via `CodegenError::NamingCollision`; automatic renaming is out of scope for the initial implementation

## Implementation Phases

These phases are implementation waves, not release-version labels.

`pgQuill` intentionally uses a phase-based development model because the full product requires multiple capability layers, iterative validation, and real usage feedback. In this model:

- each phase defines a coherent engineering slice that can be built and verified independently
- phases can be validated incrementally in real workloads before expanding scope
- the eventual product release `v1.0` is reached only after all planned phases are completed, integrated, and validated end-to-end

So this roadmap should be read as progressive capability delivery, not as "Phase 1 = v1" or "Phase 2 = v2".

### Phase 1

- catalog introspection
- generated tables and columns
- typed scalar expressions
- `SELECT ... FROM ... WHERE ...`
- `GROUP BY` / `HAVING`
- select modifiers: `DISTINCT` and `DISTINCT ON`
- built-in scalar functions and basic aggregates
- joins and aliases
- subqueries
- insert/update/delete
- `RETURNING`
- PostgreSQL `ON CONFLICT`
- tuple, generated row, and DTO decoding
- SQL rendering
- `tokio-postgres` execution
- `PgExecutor` trait with impls for `tokio_postgres::Client` and `tokio_postgres::Transaction` (plus the opt-in `toasty::Transaction` bridge via `pgquill-toasty`; see *Runtime Integration And Toasty Pairing*)

### Phase 2

- CTEs
- `WITH RECURSIVE`
- table functions / set-returning functions
- lateral joins
- arrays
- JSONB
- `hstore`
- `citext`
- domains
- enums
- composites

### Phase 3

- `MERGE`
- window functions

## Recommended Crate Layout

Do not split too early, but a clean eventual layout could be:

- `pgquill-core`
  - AST
  - SQL types
  - expression traits
  - renderer interfaces
- `pgquill-postgres`
  - PostgreSQL renderer
  - PostgreSQL operators/functions
  - type mappings
- `pgquill-tokio-postgres`
  - execution and decoding
- `pgquill-codegen`
  - catalog introspection
  - Rust emitter
  - optional schema snapshot emitter
- `pgquill-macros`
  - derives for row decoding if needed

For an actual first implementation, starting with only these may be better:

- `pgquill-core`
- `pgquill-codegen`
- `pgquill-tokio-postgres`

Then add macros after the first DSL path works.

## Testing Strategy

The implementation should rely on both unit tests and integration tests.

Unit tests should cover:

- AST construction and clause-shape transitions
- SQL rendering for representative PostgreSQL queries
- expression typing and nullability behavior
- operator and function rendering

Integration tests should use a real PostgreSQL instance. `testcontainers` is a good default choice for this project.

Runtime integration tests should cover:

- executing rendered SQL through `tokio-postgres`
- decoding into tuples, generated rows, and DTOs
- PostgreSQL-specific features such as arrays, `JSONB`, `hstore`, `citext`, domains, enums, composites, `LATERAL`, `WITH RECURSIVE`, `ON CONFLICT`, and `MERGE` as those phases land
- shared-transaction behavior through the `PgExecutor` trait, including the `toasty::Transaction` bridge (see *Runtime Integration And Toasty Pairing* for the full integration test matrix)

Codegen should also have integration tests. A good approach is:

1. start PostgreSQL with `testcontainers`
2. apply fixture DDL or real migrations
3. run `pgquill-codegen` against that live database
4. assert the generated Rust output and schema snapshot against approved fixtures
5. compile the generated code in a test crate to ensure it actually type-checks

So codegen testing should not rely on string inspection alone. It should combine:

- live introspection tests against PostgreSQL
- snapshot or golden-file tests for generated output
- compile tests for the emitted Rust API

For codegen specifically, it is reasonable to keep a small suite of focused schema fixtures:

- simple tables and foreign keys
- nullable vs non-nullable columns
- identity columns
- arrays, `JSONB`, `hstore`, and `citext`
- domains, enums, and composites

Conformance acceptance matrix (MUST):

- compile-fail tests MUST verify type mismatch rejection, illegal clause transitions, and illegal terminal availability for static query shapes
- runtime integration tests MUST assert row-count terminal semantics for `fetch_one`/`fetch_optional` and row-producing DML terminals
- deterministic codegen tests MUST assert stable generated Rust and snapshot output for identical catalog + config input
- error-contract tests MUST assert required error variant mapping for build, decode, execution, and codegen failures

## Risks

### 1. Rust type-level complexity

Risk:

- if every AST node becomes part of the type, ergonomics collapse

Mitigation:

- typed wrappers over erased AST

### 2. Nullability inference

Risk:

- incorrect nullability makes the system feel unsafe or annoying

Mitigation:

- conservative inference first
- explicit `nullable()` / `assume_not_null()` escape hatches only where truly needed

### 3. PostgreSQL feature surface area

Risk:

- PostgreSQL is large, especially around JSONB, arrays, composites, ranges, and functions

Mitigation:

- build extensible AST nodes and operator registries
- start with the features most common in application SQL

### 4. Row decoding for composites and nested types

Risk:

- `tokio-postgres` decoding gets trickier with composite and custom types

Mitigation:

- support default decoding for generated composite structs
- allow user-provided codecs

## Why PostgreSQL-only Is A Huge Advantage

This is not a limitation. It is a strength.

By supporting only PostgreSQL, `pgQuill` can:

- model PostgreSQL syntax honestly
- expose PostgreSQL-specific operators directly
- introspect rich PostgreSQL catalog metadata
- avoid lowest-common-denominator design
- make any future validated raw SQL surface much more accurate

This is how you avoid becoming a weaker clone of generic Rust query builders.

## How It Could Fit This Repository

There is a nice local alignment already:

- `schemalane` is PostgreSQL-first migration tooling
- `pg_query` is already present in the workspace
- `pg_query_fmt` already formats PostgreSQL AST

So a believable long-term story in this repo is:

1. use `schemalane` for migrations
2. build the application crate — the build script introspects the migrated database and generates schema code and snapshot
3. build typed queries first, with any future validated SQL layer sitting on top of that same snapshot

That is a very coherent toolchain.

## My Recommendation

`pgQuill` is worth building, but only if it stays disciplined.

The best strategy is:

1. Build a real PostgreSQL AST with typed wrappers, not a string builder
2. Generate schema modules from `pg_catalog`
3. Ship a clean jOOQ-like DSL first
4. Keep validated raw SQL as a separate future idea, not part of the core implementation scope
5. Treat token-tree SQL as optional sugar, not as the foundation

If you execute that well, `pgQuill` can become a genuinely differentiated Rust library:

- closer to jOOQ than existing Rust options
- more SQL-native than Diesel
- more typed and schema-driven than SeaQuery
- more composable than raw `sqlx` query strings

## Suggested First Milestone

If I were implementing this, I would make the first milestone very narrow:

- introspect tables + columns + primary/foreign keys
- generate `schema::public::*`
- support typed `select/from/where/join/order_by/limit`
- render SQL with bind parameters
- run through `tokio-postgres`
- decode into tuples and generated row structs

If that feels good, the project is real.

Everything else can grow on top of that foundation.
