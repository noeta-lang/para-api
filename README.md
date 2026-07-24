# para/api

The HTTP-client composition layer over `std.http` — middleware, mocking, pagination, and the `@openapi` typed-client generator.

The split with the stdlib is deliberate and load-bearing: **`std.http` never invokes user code; `para.api` does.** std owns transport, client configuration, error classification, and the RFC 8288 `Link` primitive, and stops at `prepare`/`send`. Everything that has to *call back into a program* — middleware, mocking, pagination — lives here, composed from ordinary Noeta values under ordinary GC.

## What it provides

- **`para.api`** — `Api` (a client wrapper) plus the `Middleware` trait and `Next`. A middleware is an **onion**, not a before/after hook pair: each layer receives the request *and* the rest of the chain, so a layer can rewrite the request, inspect the response, answer without calling `next` (a mock, a cache hit), or call `next` more than once (a retry).
- **`para.api.middleware`** — the standard layers, each an ordinary Noeta value and none privileged by the framework: `Header`, `Logging`, `Mock`, `Cache`, `Record`, `Retry`.
- **`para.api.pagination`** — `pages(api, req, strategy)` returns a lazy `Iterator` of pages (fetched only as consumed, composes with `take`/`map`/`filter`). The strategy trait is `next_page(req, resp) -> ?Request`; `Link`, `Offset`, and `Page` ship built-in, and `Cursor` (the Slack/Notion body-cursor convention) is written as an ordinary user `impl` to demonstrate the trait is enough. There is deliberately **no default strategy** — a paginator that guesses reads page one and silently reports it as the whole result set.
- **The native half** (a Rust extension crate, the reason this package needs a `[trust]` grant): the **`@openapi("spec.json")` directive** — a compile-time expansion hook that reads an OpenAPI spec (something no Noeta code can do at compile time) and generates a typed client as members of the struct it decorates: one method per operation, plus `new` and `base_url()`. `noeta expand` shows exactly what was generated — that output is the real parsed source, so a spec change is a reviewable diff. Alongside it, the `para.url` module (`url.encode`) — the percent-encoder the generated query strings are built with.

## Installation

```toml
[dependencies]
para = { version = "^0.1", package = "para/api" }

[trust]
native = ["para/api"]   # authorizes the package's native extension (the @openapi hook + para.url)
```

The package is keyed `para`, so its modules address as `para.api`, `para.api.middleware`, `para.api.pagination`, and `para.url`.

## Usage

```noeta
use para.api.Api
use para.api.middleware.Logging
use para.api.pagination.{pages, Link}
use std.http.client

api = Api.new(client.new("https://api.example.com")).with(Logging.new())

resp = api.get("/users/1")?

for page in pages(api, api.prepare("get", "/repos"), Link.new()) {
    echo page?.status()
}
```

And the generated client:

```noeta
use para.api.Api
use std.http.client

@openapi("petstore.json")
struct PetStore {}

store = PetStore.new(Api.new(client.new(PetStore.base_url())))
resp = store.show_pet_by_id("7")?
```

The generated methods answer `Response` and the caller decodes with `resp.json::<T>()` — an expansion contributes members of the declaration it decorates, and nothing else, so the directive cannot generate the `Pet` struct itself.

## Examples

- [`examples/paginated-client/`](examples/paginated-client) — middleware, mocking, and all four pagination strategies, with a hermetic `@test` suite driven entirely against a `Mock` layer.

## Requirements

Consumers compile this package's native crate locally: `cargo` and a Rust toolchain (1.95+) must be on `PATH`. The Noeta toolchain composes and builds it automatically on first use.

## Development

- `cargo test` in `crates/noeta-para-api` runs the expansion tests (they drive a real link, exercising the same path the compiler takes).
- `noeta check` / `noeta test` the programs under `examples/` (each is its own package depending on this repo by path).

See [AGENTS.md](AGENTS.md) for the repo layout and the toolchain environment the examples need.

## License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or <http://www.apache.org/licenses/LICENSE-2.0>)
- MIT license ([LICENSE-MIT](LICENSE-MIT) or <http://opensource.org/licenses/MIT>)

at your option.

### Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in the work by you, as defined in the Apache-2.0 license, shall be dual licensed as above, without any additional terms or conditions.
