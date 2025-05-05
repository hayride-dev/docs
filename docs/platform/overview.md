---
sidebar_position: 1
title: Overview
---

**Hayride** is a universal application platform for building and running portable WebAssembly components across environments.

With built-in support for language-agnostic interfaces, secure execution, agent orchestration, and lifecycle management, Hayride enables developers to create scalable, AI-driven, and distributed systems with modular architecture.


<div style={{ display: 'flex', justifyContent: 'center' }}>
  <div style={{ position: 'relative', display: 'inline-block' }}>
    <img src={require('/img/hayride-platform.png').default}  width="90%" style={{ display: 'block' }} />
    <img
      src={require('/img/chrip-talk.png').default}
      width="60"
      height="60"
      style={{
        position: 'absolute',
        bottom: '15px',
        right: '25%',
        pointerEvents: 'none',
      }}
    />
    <p></p>
  </div>
</div>


:::info
Hayride is built on WebAssembly and its supporting specifications. If you're new to WebAssembly—especially WebAssembly System Interfaces (WASI)—review the [WebAssembly documentation](./architecture/wasm.md) to understand the underlying technologies.
:::

At its core, Hayride is a secure runtime that combines a WebAssembly execution engine with a set of host-defined capabilities. It executes **Morphs**—portable, composable WebAssembly components—within a sandboxed environment.

### Core Concepts

Hayride introduces two key abstractions:

1. **Host Functions**  
   Host functions are capabilities provided by the platform and imported by Morphs. These functions enable access to external systems—such as file systems, databases, inference engines, or other stateful services—and are exposed in a controlled, secure way.

2. **Morphs**  
   Morphs are standard WebAssembly components that import host functions and can expose their own capabilities. They are designed to be portable, composable, and language-agnostic. Morphs can be combined into dependency graphs to build everything from isolated functions to full applications.

<div style={{ textAlign: 'center' }}>
  <img src={require('/img/morph-hf-interfaces.png').default} width="300" height="300" />
</div>

Morphs describe their interfaces using **WebAssembly Interface Types (WIT)**—a format that defines a clear contract for what each component imports and exports. This makes Morphs interoperable across languages, architectures, and runtimes.

:::info
Hayride publicly defines its host functions using WebAssembly Interface Types. This provides a stable, transparent contract between platform and components.

Other platforms can implement these interfaces themselves or adopt Hayride's default implementations.

Explore Hayride’s interfaces [here](./interfaces/).
:::

## Start Your Journey

To get started, explore the [guides](../guides/) that walk through building your first Morph, deploying it to the Hayride runtime, and using the platform’s core capabilities.

For deeper dives, check out:
- [Concepts](./concepts.md)
- [WebAssembly](./wasm.md)
- [Interfaces](./interfaces/)
- [SDK Reference](../sdk-reference/)
