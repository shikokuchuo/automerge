# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Automerge is a CRDT (Conflict-free Replicated Data Type) library for building local-first collaborative applications. The core is implemented in Rust and exposed to multiple platforms through different bindings:

- **Rust core** (`rust/automerge/`) - Core CRDT implementation
- **JavaScript/WASM** (`javascript/`, `rust/automerge-wasm/`) - JavaScript wrapper around WASM-compiled Rust
- **C FFI** (`rust/automerge-c/`) - C bindings for other language integrations
- **Hexane** (`rust/hexane/`) - RLE (Run-Length Encoding) compression library used by Automerge

## Architecture Overview

### Rust Core → WASM → JavaScript Flow

1. **Core Rust Library** (`rust/automerge/`): Implements CRDTs, sync protocol, binary format
   - `automerge.rs` and `autocommit.rs`: Main document types
   - `change.rs` and `change_graph.rs`: Change tracking and merging
   - `columnar/`: Columnar storage format
   - `hydrate.rs`: Document hydration from stored format

2. **WASM Bindings** (`rust/automerge-wasm/`):
   - Uses `wasm-bindgen` to compile Rust to WebAssembly
   - Provides thin JavaScript wrapper around WASM binary
   - Outputs WebAssembly module + JS bindings

3. **JavaScript Wrapper** (`javascript/`):
   - Imports WASM bindings (copied to `src/wasm_bindgen_output/`)
   - Provides idiomatic JavaScript API using Proxies
   - Handles complex module loading via subpath exports in `package.json`
   - **Multiple entry points** for different environments:
     - `fullfat_node.js` - Node.js with direct WASM loading
     - `fullfat_bundler.js` - Webpack/bundler environments
     - `fullfat_workerd.js` - Cloudflare Workers
     - `fullfat_base64.js` - Base64-encoded WASM fallback
     - `slim.js` - JS-only, user loads WASM manually

### Workspace Structure

The Rust workspace (`rust/Cargo.toml`) contains these crates:
- `automerge` - Core library
- `automerge-wasm` - WebAssembly bindings
- `automerge-c` - C FFI bindings
- `automerge-cli` - Command-line tools
- `automerge-test` - Test utilities
- `hexane` - Compression library
- `edit-trace` - Performance tracing tools

## Common Development Commands

### Building and Testing Everything

```bash
# Run full CI test suite (formats, lints, builds, and tests everything)
./scripts/ci/run
```

### Rust Development

```bash
cd rust

# Format code
cargo fmt

# Check formatting
cargo fmt -- --check

# Lint (clippy)
cargo clippy --all-targets --all-features -- -D warnings

# Run tests for specific crates
RUST_LOG=error cargo test -p automerge
RUST_LOG=error cargo test -p automerge-test
RUST_LOG=error cargo test -p automerge-c
RUST_LOG=error cargo test -p automerge-wasm

# Build with optimizations
cargo build --release

# Run benchmarks
cargo bench
```

### JavaScript Development

```bash
cd javascript

# Install dependencies
yarn install

# Build (compiles WASM and bundles JS)
node scripts/build.mjs

# Run tests
yarn test

# Run packaging tests (tests all export paths)
yarn packaging-tests

# Lint
yarn lint

# Format check
yarn check-fmt

# Format code
yarn fmt
```

**Important**: When changing Rust code, you must rebuild the JavaScript package:
```bash
cd javascript
yarn run build
```

### Individual CI Scripts

```bash
# Format Rust code
./scripts/ci/fmt

# Format JavaScript code
./scripts/ci/fmt_js

# Lint Rust code (clippy)
./scripts/ci/lint

# Build and test Rust
./scripts/ci/build-test

# Build WASM and test
./scripts/ci/wasm_tests

# Build and test JavaScript
./scripts/ci/js_tests

# Test Node 18 packaging (for WebCrypto polyfill)
./scripts/ci/node_18_packaging_test

# Build C library
./scripts/ci/cmake-build Release static

# Generate Rust docs
./scripts/ci/rust-docs

# Check security advisories
./scripts/ci/advisory
```

### C Library Development

```bash
cd rust/automerge-c

# Build
cmake -E make_directory build
cmake -S . -B build
cmake --build build

# Install to system (e.g., /usr/local)
cmake --install build --prefix "/usr/local"

# Build with UTF-32 indexing (Unicode code points instead of UTF-8 bytes)
cmake -S . -B build -DUTF32_INDEXING=true

# Generate and view docs
cmake --build build --target automerge_docs
# Open build/docs/html/index.html in browser
```

## Testing Single Files

### Rust
```bash
cd rust
cargo test -p automerge --test test_file_name
```

### JavaScript
```bash
cd javascript
yarn test test/specific_test.ts
```

## Key Architectural Patterns

### Document Types
- **`Automerge`**: Low-level document with explicit transaction management
- **`AutoCommit`**: Higher-level document with automatic transaction commits (recommended for most use cases)

### Change and Sync Protocol
- Every edit creates a "change" with an actor ID
- Changes are immutable and form a DAG (Directed Acyclic Graph)
- Sync protocol efficiently transmits only missing changes
- Deterministic merge algorithm resolves conflicts

### JavaScript Module Loading Complexity
The JavaScript package has complex conditional exports to support:
- Bundlers with native WASM support (Webpack, esbuild)
- Node.js with direct WASM file access
- Browsers without bundlers (base64-encoded WASM)
- Cloudflare Workers and other edge runtimes
- Libraries that want to defer WASM loading (`/slim` export)

See `javascript/HACKING.md` for detailed explanation of packaging strategy.

### Node 18 WebCrypto Polyfill
The `getrandom` crate needs WebCrypto API for random number generation. Node 18 doesn't have it globally, so the build process (`javascript/scripts/build.mjs`) prepends a polyfill to `.cjs` files that imports `node:crypto`. Test this with `./scripts/ci/node_18_packaging_test`.

## Release Process

### JavaScript Packages (`@automerge/automerge`, `@automerge/automerge-wasm`)
1. Bump version in `javascript/package.json`
2. Create PR to `main`, wait for CI, merge
3. Create tag: `git tag js/automerge-<version>`
4. Create GitHub release referencing the tag
5. CI automatically publishes to NPM (requires `NPM_TOKEN` secret, refreshed every 30 days)

### Rust Package (`automerge` crate)
1. Bump version in `rust/automerge/Cargo.toml`
2. Create PR, merge to `main`
3. Create tag: `git tag rust/automerge@<version>`
4. Push tag: `git push origin rust/automerge@<version>`
5. Manually publish: `cargo publish`

## Important Notes

- **Minimum Rust version**: 1.89.0 (see `rust-toolchain.toml`)
- **Release profile**: Uses LTO and single codegen unit for optimal performance
- **Memory efficiency**: Automerge 3 achieved ~10x reduction in memory usage
- **Binary format**: Columnar storage format spec at https://automerge.org/automerge-binary-format-spec
- All commits should have descriptive messages explaining "why" not just "what"
