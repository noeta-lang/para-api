# para/api

The HTTP-client composition layer over `std.http` — middleware, mocking, pagination, and the `@openapi` typed-client generator.

The split with the stdlib is deliberate and load-bearing: **`std.http` never invokes user code; `para.api` does.** std owns transport, client configuration, error classification, and the RFC 8288 `Link` primitive, and stops at `prepare`/`send`. Everything that has to *call back into a program* — middleware, mocking, pagination — lives here, composed from ordinary Noeta values under ordinary GC.

## What it provides

- **`para.api`** — `Api` (a client wrapper) plus the `Middleware` trait and `Next`. A middleware is an **onion**, not a before/after hook pair: each layer receives the request *and* the rest of the chain, so a layer can rewrite the request, inspect the response, answer without calling `next` (a mock, a cache hit), or call `next` more than once (a retry).
- **`para.api.middleware`** — the standard layers, each an ordinary Noeta value and none privileged by the framework: `Header`, `Logging`, `Mock`, `Cache`, `Record`, `Retry`.
- **`para.api.pagination`** — `pages(api, req, strategy)` returns a lazy `Iterator` of pages (fetched only as consumed, composes with `take`/`map`/`filter`). The strategy trait is `next_page(req, resp) -> ?Request`; `Link`, `Offset`, and `Page` ship built-in, and `Cursor` (the Slack/Notion body-cursor convention) is written as an ordinary user `impl` to demonstrate the trait is enough. There is deliberately **no default strategy** — a paginator that guesses reads page one and silently reports it as the whole result set.
- **The native half** (a Rust extension crate, the reason this package needs a `[trust]` grant): the **`@openapi("spec.json")` directive** — a compile-time expansion hook that reads an OpenAPI spec (something no Noeta code can do at compile time) and generates a typed client as members of the struct it decorates: one method per operation, plus `new` and `base_url()`. `noeta expand` shows exactly what was generated — that output is the real parsed source, so a spec change is a reviewable diff. Alongside it, the `para.url` module (`url.encode` / `url.decode`) — the percent-encoder the generated query strings are built with, and its inverse for taking a URL apart. Both are byte-wise over UTF-8, which is why they are native; `std.http.url` is the same pair in the standard library.

## Installation

```toml
[dependencies]
para = { version = "^0.3", package = "para/api" }

[directives]
openapi = "para/api"    # required to write `@openapi` — nothing is ambient

[trust]
native = ["para/api"]   # authorizes the package's native extension (the @openapi hook + para.url)
```

The package is keyed `para`, so its modules address as `para.api`, `para.api.middleware`, `para.api.pagination`, and `para.url`.

The `[directives]` entry is what makes `@openapi` resolve; a `use` neither substitutes for it nor is needed alongside it, and a program that writes `@openapi` without one gets an unknown-directive error. Name the package **identity** (`"para/api"`), not the dependency key: a key bound to a *scope* — `para = [{ … }, { … }]` — covers several packages at once and cannot say which one you meant. Only a consumer that writes `@openapi` needs the binding; the middleware and pagination surfaces are ordinary imports.

## The middleware onion — compose behavior around every request

An `Api` wraps a `std.http.Client` and a stack of layers. It is immutable, exactly like the client: `with` returns a **new** `Api`, so a derived one can never disturb the one it came from and sharing is safe.

```noeta
use para.api.Api
use para.api.middleware.{Header, Logging}
use std.http.client

api = Api.new(client.new("https://api.example.com"))
    .with(Logging.new())
    .with(Header.new("accept", "application/json"))

resp = api.get("/users/1")?
```

The verbs — `get`, `post`, `put`, `patch`, `delete`, and the general `request(method, path, body, headers)` — mirror `std.http.Client`'s exactly (same names, same optional trailing headers, same `Result<Response, HttpError>`), differing only in that they run the chain. `prepare(method, path)` builds a request without performing it, which is what pagination consumes below.

**The first layer registered is the outermost** — it sees the request first and the response last — the order every onion (PSR-15, Tower, Rack) uses, and the order that makes reading a chain top-to-bottom match the order things happen.

A layer implements one function, and receives the rest of the chain as a value:

```noeta
use std.http.{Request, Response, HttpError}
use para.api.{Middleware, Next}

pub struct Trace {
    fn new(): Trace { return Trace {} }
}

impl Middleware for Trace {
    fn handle(req: Request, next: Next): Result<Response, HttpError> {
        return next.run(req.with_header("x-trace", "abc"))
    }
}
```

Because `next` is callable rather than a before/after hook pair, a layer can do three things a hook pair cannot express:

| shape | meaning | standard layers of that shape |
| --- | --- | --- |
| pass through | rewrite the request, call `next.run`, inspect the response | `Header`, `Logging`, `Record` |
| short-circuit | answer without calling `next` at all — nothing inside runs, including the transport | `Mock`, `Cache` (on a hit) |
| call `next` more than once | re-run everything inside | `Retry` |

## Mocking and recording — hermetic tests without a socket

`Mock` answers from a canned table instead of the network. Routes are keyed `"METHOD /path"` — an exact match, not a pattern: the failure mode of an exact key is a loud unmatched route rather than a silently wrong reply. A route may also name a query string, and lookup prefers it over the bare path — which is what lets one mock answer a *paginated* endpoint.

```noeta
use para.api.middleware.Mock

mock = Mock.new()
    .route("get", "/repos", 200, "[\"a\", \"b\"]",
        {"link": "<https://api.example.com/repos?page=2>; rel=\"next\""})
    .route("get", "/repos?page=2", 200, "[\"c\"]")

api = Api.new(client.new("https://api.example.com")).with(mock)
```

An unmatched request does **not** fall through to the network — it answers the `fallback` status (`Mock.new(404)` by default), with a body naming the exact missing key so the fix is a copy-paste. A mock that silently reached the real API would make a test suite depend on the internet exactly when someone forgot a route. `route` returns a new `Mock`, so a base table can be shared and specialized; `reply(method, path, resp)` registers a `Response` value verbatim, headers and all.

`Record` is the recording half of the same loop: it captures every request/response pair that flows through, so a real session can be replayed forever without a network.

```noeta
rec = Record.new()
live = Api.new(client.new("https://api.example.com")).with(rec)
walk(live)                       // hits the network once

replay = Api.new(client.new("https://api.example.com")).with(rec.to_mock())
walk(replay)                     // hits nothing, forever
```

`rec.calls()` lists every request seen, in order, as `"METHOD /path?query"` — the same spelling `Mock.route` takes — so a test can assert on *what was called* and not only on what came back. `to_mock(fallback = 404)` replays responses verbatim, including the `Link` header, which is what makes a paginated walk replayable at all.

## Caching and retrying — the other two onion shapes

`Cache.new(ttl_ms)` answers a repeated request from memory. Three rules keep it from being a correctness hazard, and none is configurable: only safe methods (`GET`/`HEAD`) are cached, only 2xx responses are cached (a 503 is a statement about *now*), and entries expire — `ttl_ms` is required by `new` for that reason. `cache.hits()` counts the requests answered without touching `next`, which is the difference between "the cache is configured" and "the cache is working". Time comes from `std.time`'s deterministic monotonic clock, so a test can `time.sleep` past a TTL and observe the miss without waiting.

`Retry` re-runs the **whole chain** on a transient failure — unlike `std.http`'s own `retry(n)`, which lives inside `send`, beneath every layer, so a retried attempt never re-enters them (a per-attempt trace id is never re-minted, a `Record` sees one call where two happened).

```noeta
api = Api.new(client.new("https://api.example.com").retry(0))
    .with(Retry.new(2, 10))          // outermost: re-runs everything below it
    .with(Logging.new())
```

> [!WARNING]
> The two retries are not alternatives to stack: put `Retry` on a client configured `retry(0)`, or the attempt counts multiply — 3 outer × 3 inner = 9 requests for one call.

`Retry.new(max = 3, base_ms = 250)` mirrors std's defaults: exponential backoff doubling from 250ms (capped at 30s), and the four statuses that actually mean "later" — 429, 502, 503, 504. Note what is absent: **500** — a generic server error is usually deterministic, and hammering it helps nobody. `Retry.on(max, base_ms, statuses)` names the retryable statuses yourself (`[]` to retry transport failures only). What counts as a retryable *transport* failure is `HttpError.retryable()`, never message text; a server's own `Retry-After` (delta-seconds form) beats the computed curve, capped at 30s all the same. The last attempt returns whatever it got, success or not — the caller sees the real outcome rather than a synthesized "retries exhausted" error.

## Pagination — lazy pages, and a strategy you name

Paging is not one convention, it is at least four, and they disagree about everything: where the cursor lives, what ends the sequence, whether the next location is a URL or a number. So the extension point is a **strategy** — `next_page(req, resp) -> ?Request` — and `pages` demands one by parameter:

```noeta
use para.api.pagination.{pages, Link}

for page in pages(api, api.prepare("get", "/repos"), Link.new()) {
    resp = page?
    echo resp.status()
}
```

`pages` returns an `Iterator`, not a `List`: a generator suspends between requests, so pages are fetched only as they are consumed and the sequence composes with `take`/`map`/`filter` — `take(2)` performs exactly two requests.

| strategy | convention | stops when |
| --- | --- | --- |
| `Link.new()` / `Link.rel("prev")` | RFC 8288 `Link` header (GitHub, GitLab, Shopify) | the relation is absent |
| `Offset.new(limit, is_last)` / `Offset.named(offset_param, limit_param, limit, is_last)` | `offset` + `limit` query parameters | your `is_last` predicate, or a non-2xx |
| `Page.new(is_last)` / `Page.named(param, start, is_last)` | a `page` number query parameter | your `is_last` predicate, or a non-2xx |
| `Cursor.new(param = "cursor")` | an opaque body cursor (Slack, Notion) | `has_more: false` or an empty `next_cursor` |

An offset or page API sends no end-of-sequence signal at all — asking past the last page returns an empty page with a perfectly good 200 — so the caller supplies the stop rule as a predicate on the page that just arrived, usually "shorter than the limit" after a typed decode:

```noeta
strategy = Offset.new(100, fn(r) => match r.json::<List<Item>>() {
    Ok(items) => items.len() < 100,
    // A body that will not decode cannot be counted, so stop rather than page forever.
    Err(_) => true,
})
```

`Offset.named` and `Page.named` cover the same conventions under other spellings (`skip`/`take`, a 0-based `page`); `Cursor` ships as a worked example of a user-written strategy — it is `pub` and perfectly usable, but its point is proving the trait is enough. Writing your own is one `impl PageStrategy`, and the two URL helpers the built-ins are written in terms of are `pub` for exactly that: `pagination.resolve_url(base, target)` resolves a possibly-relative link against the URL it was found at, and `pagination.set_query(url, name, value)` sets or replaces one query parameter while preserving the rest. A body-URL convention (Stripe's fully-qualified `next`) is `Cursor`'s code with `req.with_url(next)` in place of `set_query`.

Each element of the sequence is the whole `Result<Response, HttpError>`, not a bare `Response`: a transport blip on page 9 arrives as an element you can `?`, log, or ignore — the 8 pages already in hand survive it — and the sequence ends there, because there is no next request to derive from a response that never came.

## `@openapi` — a typed client generated from a spec

The package's native half is a compile-time expansion hook: `@openapi("spec.json")` reads an OpenAPI document and generates a client as members of the struct it decorates.

```noeta
use para.api.Api
use std.http.client

@openapi("petstore.json")
struct PetStore {}

store = PetStore.new(Api.new(client.new(PetStore.base_url())))
resp = store.show_pet_by_id("7")?
```

The struct is written empty; the directive generates the `api` field, a `new(api)` constructor, `base_url()` from the spec's first `servers[].url`, and one method per operation — named from its `operationId` (snake-cased), or derived from the method and path when the document omits one. The spec path is resolved against the file the directive is written in, so a spec sits next to the code that uses it. Run **`noeta expand`** to see exactly what was generated — that output is the real parsed source, so a spec change is a reviewable diff.

Generated signatures follow one shape: required parameters (path parameters, required query parameters) are **typed and positional**; a `body: string = ""` appears when the operation declares a request body; optional query parameters arrive in a trailing `query: Map<string, string> = {}`, listed by key and type in the generated doc comment. Every call routes through `self.api.request(...)` — never the raw client — so `Mock`, `Cache`, and `Logging` apply to generated clients for free, and path parameters are percent-encoded through `Api.encode` (query strings through `Api.query_string`), both also available to hand-written code.

> [!NOTE]
> An expansion contributes members of the declaration it decorates, and nothing else — so the generated methods answer `Response` and the caller decodes with `resp.json::<T>()`; the directive cannot generate the `Pet` struct itself. Two further edges, stated plainly: only **JSON** specs are read (convert a YAML spec first — both spellings are the same document), and `header`/`cookie` parameters are ignored, because middleware is the right place to set those.

## Error handling — `?` means the network broke

Everything answers `Result<Response, HttpError>`, and the line is sharp: an `Err` means precisely "the network broke" — an HTTP error *status* is an answer, and arrives as an ordinary `Ok(Response)` for you to inspect with `resp.status()` / `resp.ok()`. Match on `HttpError.kind()` (and `retryable()`), never on message text — `retryable()` is true for exactly the transient kinds (`timeout`/`dns`/`connect`) and false for the deterministic ones. Response bodies are remote input, so `resp.json::<T>()` is recoverable by construction: a body that will not decode is an `Err` to handle, not an abort.

## Examples

- [`examples/paginated-client/`](examples/paginated-client) — middleware, mocking, and all four pagination strategies, with a hermetic `@test` suite driven entirely against a `Mock` layer.
- [`examples/typed-client/`](examples/typed-client) — `@openapi` end to end: the `[directives]` binding, a spec on disk, and the generated client called from ordinary top-level code (deliberately not only from `@test`, which is white-box and would type-check against a private surface).

## Requirements

Noeta **0.5 or later**. The floor is not a preference: from 0.5 a method is private by default in every type kind, so this package's `pub` surface — and the `pub` on every member `@openapi` generates — is grammar an earlier toolchain reads differently.

Consumers compile this package's native crate locally: `cargo` and a Rust toolchain (1.95+) must be on `PATH`. The Noeta toolchain composes and builds it automatically on first use.

## Development

- `cargo test` in `crates/noeta-para-api` runs the expansion tests (they drive a real link, exercising the same path the compiler takes).
- `noeta check` / `noeta test` the programs under `examples/` (each is its own package depending on this repo by path).

A note for anyone adding to `examples/`: put at least one call to whatever you are proving **outside** any `@test` block. A dev-tier body is white-box — it may reach a type's private methods — so a public surface exercised only from `@test` type-checks no matter how private it is.

See [AGENTS.md](AGENTS.md) for the repo layout and the toolchain environment the examples need.

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or <http://www.apache.org/licenses/LICENSE-2.0>)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or <http://opensource.org/licenses/MIT>)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any additional terms or conditions.
