---
layout: post
title: "WebAssembly: Interfacing with the host environment"
tags: programming WebAssembly Rust
---

## Purpose
many ways to handle the interface between wasm bin and host. This is not for quickly getting started being productive, but for how to do things manually, strategies for doing so, and what the available or future options are. Not specific to any host environment. Main focus is js or wasmer as host, and rust as source language.

## Communicating simple data
- declaring ffi + exposing functions in Rust and C. Using C ABI. FFI unsafe.
- exposing stuff from the host (js + wasmer)

By default, the resulting binary will expect the declared functions to exist with root import path "env". To customise this, add a decorator
`#[link(wasm_import_module = "example")]`
(? what's this called in rust, what is "link" and what is "wasm_import_module"? I guess "link" is a decorator which accepts linker options to be applied to the decorated symbol)

## General intro to the issue
- types in wasm are numbers + others (anyref or whatever) (link to doc). Representation of more complex types (arrays, strings, structs) is built on top of this and is specific to the source language. Therefore for an external interface, data has to be communicated in both directions using numbers. data needs to be de/serialised

## Memories
- Wasm module currently can have up to 1 memory, which can be either imported or not.
For Rust, in order to have the memory imported instead of exported, add this to `.cargo/config`. The host will then be expected to provide a memory with import path "env" "memory".
- example in js + wasmer

```toml
[build]
rustflags = ["-C", "link-args=--import-memory"]
```

## From wasm to host
- The host is able to access the linear memory of the wasm module. So to send data from the wasm to the host, a pointer can be passed through the interface and the host can interpret it as an offset value within the module's memory.
- As an example - say we want to expose functionality to the wasm module to allow it to send lines of text for the host to display to the user.
  A fixed-size string can be represented as a pointer to some utf-8 encoded text, and a length in bytes. A function can be exposed to the wasm module that accepts these two parameters.

```rust
extern "C" {
    fn send_line_unsafe(ptr: *const u8, len: usize);
}

fn send_line(text: &str) {
    unsafe { send_line_unsafe(text.as_ptr(), text.len()) }
}
```
In order to implement the `sendLine` function, the host will need to interpret the referenced data as utf-8. It usually would be wrapped in the host language's native string type and may involve a copy.
TODO js and/or wasmer implementation

- In this simple case, the lifetime of the string can be tied to the lifetime of the `send_line_unsafe` function since the host doesn't need the data after it has returned. Therefore the wasm module can retain ownership of the data.

If the lifetime of the data is longer, then it may be necessary to make the host responsible for cleaning up the associated memory when it is done.

### From host to wasm
- A wasm module only has access to its own isolated linear section of memory. So the host can't give the module a reference to data from the host process's memory. 
For the host to send heap data to the wasm, the host must directly write or copy the data into the wasm memory.
- To do this, there must be some memory allocated for the host to write to. Either some space in the wasm module's memory needs to be reserved for host's use, or the wasm module needs to be responsible for allocating memory from it's own heap for this purpose.
- A generic solution is to expose dynamic allocation functions to the host, which can be implemented using Rust's allocator.
TODO  Example of allocation functions + writing to it from host.

### Opaque references
- In some cases data that needs to be referenced in the host-wasm interface only needs to be interpreted by its owner.
- E.g. from the host language, some data in the wasm memory could be operated on by passing a pointer (or other unique reference) around to different functions of the wasm module, rather than reading or writing to it directly.

TODO: validate the example

```rust
struct Vector {
  x: u32,
  y: u32
}

#[no_mangle]
pub extern "C" fn dot_product(a: &Vector, B: &Vector) -> u32 {
  ...
}
```

```ts
class Vector {
  private readonly ref: number;

  constructor(ref: number) {
    this.ref = ref;
  }

  static dotProduct(a: Vector, b: Vector): number {
    return instance.dot_product(a, b);
  }
}
```

Todo:
- about wasm bindgen, wasmpack. Standard lib. Wasi
- about interface types
- code examples wherever possible
- if there is a lot, put more generic stuff about using rust with wasm in another post

Things still to find out:

- wasm import paths
- i32 type used for everything: how this affects API with unsigned values or values with different width
- host owning memory
- wasm bindgen: what it actually does