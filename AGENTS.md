# AGENTS.md

Guidance for coding agents working in this repo — the standalone repo of the **para/api** Noeta package (the HTTP-client composition layer + the `@openapi` directive), extracted from the noeta monorepo. Toolchain issues (the language, the `noeta` binary, `std.*`, the extension ABI) belong in the monorepo at github.com/noeta-lang/noeta, not here.

## Repo layout

- `noeta.toml` — the package manifest (`name = "para/api"`, `native = "native"` declares the Rust extension entry crate).
- `api.noe` / `middleware.noe` / `pagination.noe` — the pure-Noeta surface: the `Api`/`Middleware`/`Next` onion, the standard layers, and the pagination strategies. Deliberately NOT native: composition invokes user code, and a native client holding user closures was tried once and leaked.
- `crates/noeta-para-api/` — the impl crate: the `@openapi` compile-time expansion hook (spec parsing + codegen) and the `para.url` percent-encoder.
- `native/` — the thin entry crate the manifest's `native` key points at; re-exports `NOETA_EXTENSIONS`.
- `examples/*/` — each a standalone package depending on this repo via `para = [{ path = "../.." }]`. A `@openapi` example also spells `package = "para/api"` on that entry and binds `[directives] openapi = "para/api"`: a directive resolves only through a binding, and the binding must name a package identity rather than a scope key. Their `noeta.lock` files are **not** committed (`.gitignore`) — they regenerate on every run.
- Put at least one call to whatever an example proves **outside** any `@test` block. A dev-tier body is white-box (it may reach a type's private methods), so a public surface exercised only from `@test` type-checks however private it is — which is exactly how a `pub` that should have been written gets missed.
- `.github/workflows/` — CI (`ci.yml`) and the tag-triggered registry publish (`release.yml`).

## Build & test

- `cargo test` inside `crates/noeta-para-api` works standalone. Its real dependency is the **published** contract crate `noeta-ext-abi = "0.5"` (a range — a patch toolchain release must not cost this manifest an edit). Its `[dev-dependencies]` are compiler **internals** (`noeta-loader`/`noeta-lexer`/`noeta-ast`/`noeta-check`/`noeta-stdlib`), unpublished and carrying no stability promise, so they stay git pins on the exact release tag and are bumped deliberately. A `[patch."<toolchain url>"]` folds their copy of `noeta-ext-abi` onto the published one — without it there are **two** copies of the ABI and `dyn Extension` stops matching. `native/` builds the same way.
- Running the examples needs the `noeta` binary and **composes a toolchain** (the native crate is compiled in). Set:
  - nothing, in the common case: the compose `[patch]` key defaults to the binary's baked repository URL (`https://github.com/noeta-lang/noeta`), which now equals the URL the crates' Cargo.toml declares. When overriding to a fork or local clone, `NOETA_TOOLCHAIN_REPO` MUST equal the declared URL, or the composed build links two copies of the extension ABI and every impl fails with a two-`Extension`-traits E0308;
  - optionally `NOETA_TOOLCHAIN_SRC=<path to a noeta checkout>` to skip the git fetch.
- Then `noeta check` / `noeta test` each `examples/*` program.

## Conventions

- Rust: default `rustfmt` style (no `rustfmt.toml`), `cargo clippy --all-targets -- -D warnings` clean, zero compiler warnings; the CI toolchain is pinned at 1.97.0 — lint against it locally (a floating `@stable` surfaces lints CI doesn't have yet, and vice versa).
- Rust naming: `snake_case` files/functions, `PascalCase` types, `SCREAMING_SNAKE_CASE` constants; prefer enums and constants over magic strings.
- Markdown never hard-wraps lines.
- **American English** throughout — code, comments, and docs (`behavior`, not `behaviour`).
- **Conventional commits** for all commit titles. Commit each green slice as it completes, but **never `git push` without explicit authorization**. Never move a published `v*` tag — a release is a new tag.
- Implement in full — no stubs or TODOs; new functionality lands with tests.
- Keep `README.md` and this file up to date when layout or behavior changes.

## CI

`ci.yml` gates the Rust crates (fmt/clippy/test) and the examples (pinned released `noeta`); `release.yml` re-runs the crate gate then publishes the tag to the hosted registry (`noeta publish`, keyless Sigstore provenance via GitHub OIDC). Both go green only once the toolchain repo is published under github.com/noeta-lang/noeta and the `file:///` deps are flipped.
