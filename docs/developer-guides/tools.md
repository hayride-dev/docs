---
sidebar_position: 5
title: Tools
---

## WebAssembly

These WebAssembly tools are useful when working with Hayride.

- [wasm-tools](https://github.com/bytecodealliance/wasm-tools)
  - CLI utilities for inspecting and manipulating WebAssembly modules and components.

- [wit-deps](https://github.com/bytecodealliance/wit-deps)
  - A WIT dependency manager for resolving and downloading WIT packages.

- [wit-bindgen-go](https://github.com/bytecodealliance/go-modules/)
  - A generator that produces Go bindings from WIT definitions.

- [wac](https://github.com/bytecodealliance/wac)
  - A CLI tool for composing and packaging WebAssembly components together.

:::tip
For a complete guide on using these tools with Rust and WebAssembly components, refer to the [Rust Component Model Guide](https://component-model.bytecodealliance.org/language-support/rust.html)
:::

## Go

Go can be used to build WebAssembly components through the following tools:

- [TinyGo](https://tinygo.org/)
  - A Go compiler with WebAssembly support. To build with the wasip2 target, use: `tinygo build -target=wasip2`

- [wit-bindgen-go](https://github.com/bytecodealliance/go-modules/)
  - Generates Go bindings from WIT definitions.

- [hayride bindings](https://github.com/hayride-dev/bindings)
  - Pre-generated Go bindings for Hayride, including helpful wrapper utilities.

:::tip
For a complete guide to Go tooling and WebAssembly component builds, see the [Go Component Model Guide](https://component-model.bytecodealliance.org/language-support/go.html)
:::

## Rust

Rust can be used to build WebAssembly components through the following tools:

- WASI-P2 Target
  - Add the appropriate Rust target:
  - `rustup target add wasm32-wasip2`

- [cargo component](https://github.com/bytecodealliance/cargo-component)
  - A Cargo subcommand for building and managing WebAssembly components in Rust.

:::tip
See the [Rust Component Model Guide](https://component-model.bytecodealliance.org/language-support/rust.html) for detailed instructions and examples.
:::
