# AGENTS.md

Guidance for coding agents working in this repo — the standalone repo of the **para/api** Noeta package (the HTTP-client composition layer + the `@openapi` directive), extracted from the noeta monorepo. Toolchain issues (the language, the `noeta` binary, `std.*`, the extension ABI) belong in the monorepo at github.com/noeta-lang/noeta, not here.

## Repo layout

- `noeta.toml` — the package manifest (`name = "para/api"`, `native = "native"` declares the Rust extension entry crate).
- `api.noe` / `middleware.noe` / `pagination.noe` — the pure-Noeta surface: the `Api`/`Middleware`/`Next` onion, the standard layers, and the pagination strategies. Deliberately NOT native: composition invokes user code, and a native client holding user closures was tried once and leaked.
- `crates/noeta-para-api/` — the impl crate: the `@openapi` compile-time expansion hook (spec parsing + codegen) and the `para.url` percent-encoder.
- `native/` — the thin entry crate the manifest's `native` key points at; re-exports `NOETA_EXTENSIONS`.
- `examples/*/` — each a standalone package depending on this repo via `para = { path = "../.." }`, with its committed `noeta.lock`.
- `.github/workflows/` — CI (`ci.yml`) and the tag-triggered registry publish (`release.yml`).

## Build & test

- `cargo test` inside `crates/noeta-para-api` works standalone — the toolchain crates are git dependencies (currently the pre-publish `file:///home/niklas/Code/lang` form; flips to `https://github.com/noeta-lang/noeta` at publish). `native/` builds the same way.
- Running the examples needs the `noeta` binary and **composes a toolchain** (the native crate is compiled in). Set:
  - `NOETA_TOOLCHAIN_REPO=file:///home/niklas/Code/lang` — MUST equal the URL the crates' Cargo.toml declares, or the composed build links two copies of the extension ABI and every impl fails with a two-`Extension`-traits E0308;
  - optionally `NOETA_TOOLCHAIN_SRC=<path to a noeta checkout>` to skip the git fetch.
- Then `noeta check` / `noeta test` each `examples/*` program.

## Conventions

- Rust code is `cargo fmt` and `cargo clippy --all-targets -- -D warnings` clean (toolchain pinned at 1.97.0 in CI).
- `noeta.lock` files under `examples/` **are committed** — leave resolved locks in place.
- Markdown never hard-wraps lines; American English throughout.
- Conventional commits. Never move a published `v*` tag — a release is a new tag.

## CI

`ci.yml` gates the Rust crates (fmt/clippy/test) and the examples (pinned released `noeta`); `release.yml` re-runs the crate gate then publishes the tag to the hosted registry (`noeta publish`, keyless Sigstore provenance via GitHub OIDC). Both go green only once the toolchain repo is published under github.com/noeta-lang/noeta and the `file:///` deps are flipped.
