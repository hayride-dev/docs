---
sidebar_position: 1
title: WebAssembly
---

WebAssembly (Wasm) is a low-level binary format designed as a fast, portable compilation target for modern languages. Originally built for the browser, Wasm now powers secure, sandboxed execution across platforms—from edge devices to servers.

:::info
This page introduces key WebAssembly concepts and how they relate to Hayride. For foundational knowledge, see the [WebAssembly documentation](https://webassembly.org).
:::

## Design Goals

WebAssembly emphasizes:

- **Simplicity and small size** – Fast to load and easy to implement.
- **Language and tool neutrality** – Supports Rust, C/C++, Go, and others.
- **Security and isolation** – Strong sandboxing with strict control over memory and execution.
- **Modular evolution** – New features are introduced via standalone proposals (e.g., threads, GC, exceptions).

## Key Properties

Wasm strikes a balance between low-level performance and high-level safety, making it ideal for embedding untrusted or portable logic in distributed systems.

- **Portability** – Runs consistently across OSes and CPU architectures.
- **Safety** – Executes in a sandbox; no host access unless explicitly granted.
- **Performance** – Approaches native speed via AOT and JIT compilation.
- **Compactness** – Binary format is small and efficient to transmit.
- **Determinism** – Predictable behavior enables reproducibility and distributed consistency.

## Component Model

As WebAssembly evolves from single binaries to composable systems, the **Component Model** enables structured composition:

- **Interfaces via WIT** – Defines shared data and functions between modules.
- **Dynamic linking** – Components declare dependencies with well-defined contracts.
- **Resource isolation** – Securely combines untrusted logic into cohesive systems.

Hayride builds on this model to run AI agents, tools, and morphs together in tightly controlled, interoperable environments.

### WebAssembly Component 

Per the [component model documentation](https://component-model.bytecodealliance.org/design/components.html) compoents can be by the following three perspectives:
- **Logical** – components act as containers for modules—or even other components—that define their interfaces and dependencies using WIT.
- **Conceptually** – components are self-contained units of code that communicate solely through defined interfaces, avoiding shared memory
- **Physically** – a component is a WebAssembly file with a specific format. Inside, it may contain multiple core WebAssembly modules and nested components, linked together through their imports and exports.

The external interface of a component—its imports and exports—represents a world, but the component itself determines how that world is realized internally.

Interfaces are defined using [WIT](https://component-model.bytecodealliance.org/design/wit.html), which allows for a clear and consistent way to describe the functions and data that a component exposes through interface types. This enables components to be easily composed and reused, as they can be linked together based on their shared interfaces.

## WASI

**WASI (WebAssembly System Interface)** defines standard APIs for accessing host resources safely and portably. Developed by the WebAssembly community, WASI brings system capabilities like I/O, clocks, and networking into sandboxed runtimes.

### Evolution

- **Preview 1** – Early MVP APIs based on POSIX, using the `witx` IDL. Widely adopted.
- **Preview 2 (Stable)** – A reimagined, modular system built on the Component Model using `wit`, with better type support and broader language coverage.

### Goals

- Provide portable, modular APIs independent of runtime or OS
- Maintain security through capability-based access
- Support progressive evolution and tooling compatibility
- Enable safe and composable general-purpose applications

### Design Principles

- **Capability-Based Security** – No ambient authority; access must be granted explicitly.
- **No Global State** – Each module only sees what it’s given.
- **Interposition-Friendly** – Interfaces can be wrapped or overridden in WebAssembly itself.
- **Compatibility via Tooling** – Backwards support through layers like `wasi-libc`.
- **Portability & Modularity** – APIs are optional and independently maintained. Hosts implement only what they support.

### Community & Proposals

WASI APIs are tracked on the [WebAssembly/WASI GitHub](https://github.com/WebAssembly/WASI). New APIs are proposed using a standard format and defined using WIT, following the design rules of the Component Model.

## WebAssembly Interface Types (WIT)

**WIT** (WebAssembly Interface Types) defines how components describe the functions and data they import and export—enabling language-independent and runtime-agnostic composition.

Key benefits of WIT:

- **Clear Interface Contracts** – Each component explicitly declares its dependencies and capabilities.
- **Language Interoperability** – Enables cross-language function calls (e.g., Rust → Go) without custom glue code.
- **Portable ABI** – Interfaces are encoded in a standard way, decoupling modules from platform-specific bindings.
- **Composable Components** – WIT enables components to be linked together automatically based on matching imports and exports.

A WIT interface might define something like:

```wit
interface logger {
  log: func(msg: string)
}
```
This allows any component implementing or depending on logger to interact via a shared contract.

WIT is the foundation for the Component Model, making large-scale Wasm systems like Hayride possible.

---
## Wasm + Hayride 

Hayride adopts WebAssembly with a focus on WASI Preview 2 and the Component Model to power secure, interoperable execution of Morphs—Hayride’s modular AI agents and tools. 

WASI's modular and capability-based approach aligns directly with Hayride’s priorities:
- Security-first execution
- Defined interfaces between components
- Broad host portability

We are committed to contributing to the WebAssembly ecosystem and aligning Hayride with the evolving standards around WASI and WIT.