---
title: "WebAssembly and Rust"
tags: programming wasm rust
---

WABT: tools for working with binaries

- wasm-strip (from wabt)
- wasm-opt (Binaryen)

## Rust

- wasm-pack does js junk. Do I want to use it for host-agnostic output?
- wasm32-unknown-unknown target
- wasm32-wasi target
- briefly summarise: wasm32-unknown-unknown, wasm32-wasi, wasm-pack, wee-alloc

```toml
[lib]
crate-type = ["cdylib"]
```

cdylib: for WebAssembly target it simply means "create a \*.wasm file without a start function"
create-type=rlib: to ensure that our library can be unit tested with wasm-pack test

Set default target:
.cargo/config

```toml
[build]
target = "wasm32-unknown-unknown"
```

```rust
#[no_mangle]
pub extern "C" fn func() {}
```

## Allocator

```rust
// Use `wee_alloc` as the global allocator.
#[global_allocator]
static ALLOC: wee_alloc::WeeAlloc = wee_alloc::WeeAlloc::INIT;
```

```toml
[dependencies]
wee_alloc = "0.4.5"

[profile.release]
# Tell `rustc` to optimize for small code size.
opt-level = "s"
```

## C

https://v8.dev/blog/emscripten-standalone-wasm
emcc test.c -s WASM=1 -o ctest.wasm -s ERROR_ON_UNDEFINED_SYMBOLS=0 -s STANDALONE_WASM
