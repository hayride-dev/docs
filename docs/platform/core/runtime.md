---
sidebar_position: 1
title: Runtime
---

# Hayride Runtime

Hayride is a runtime for building and executing modular WebAssembly components, which we call **morphs**. It leverages the [WebAssembly Component Model](https://webassembly.github.io/component-model/) and the [WASI](https://wasi.dev/) standard to ensure interoperability and portability across environments.

<div style={{ display: 'flex', justifyContent: 'center' }}>
  <div style={{ position: 'relative', display: 'inline-block' }}>
    <img src={require('/img/hayride-runtime.png').default} width="90%" style={{ display: 'block' }} />
    <p></p>
  </div>
</div>

Hayride's primary focus is on supporting **WASIp2**, an evolution of the original WASI specification. WASIp2 introduces a richer and more composable interface for WebAssembly components, enabling Hayride to run morphs with greater flexibility, efficiency, and capability.

## WebAssembly Runtime

Hayride is built on top of the [Wasmtime](https://wasmtime.dev/) WebAssembly runtime.

Wasmtime provides a fast and secure execution engine for WebAssembly modules. Hayride extends this by embedding a minimal set of custom **host functions**, exposed via the WebAssembly Interface Types (WIT) system, to support morph functionality while remaining interoperable with other WASI-compliant environments.

## Host Function Philosophy

Hayride defines host functions through WIT interfaces, making them importable by morphs. These host functions enable powerful interactions such as networking, file I/O, and AI services.

However, Hayride intentionally limits the number of host functions in the runtime itself. Instead of coupling all logic to the host, we encourage developers to implement functionality as reusable morphs. This design promotes long-term portability and allows morphs to run in other WASI-compliant runtimes.

Ultimately, Hayride aims to **minimize vendor lock-in** by enabling morphs to be executed in any WebAssembly runtime that supports the Component Model and WASI.

All host functions are defined in our open [Coven repository](https://github.com/hayride-dev/coven).

### Runtime WIT

The Hayride runtime implements a collection of [WIT](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md) `world`s that compose various capabilities:

```wit
package hayride:runtime@0.0.1;

world hayride-server {
    include hayride:wasip2/imports@0.0.51;
    include hayride:http/client-server@0.0.51;
}

world hayride-cli {
    include hayride:wasip2/imports@0.0.51;
    include hayride:wasip2/exports@0.0.51;
}

world hayride-ws {
    include hayride:socket/exports@0.0.51;
}

world hayride-ai {
    include hayride:ai/imports@0.0.51;
}

world hayride-api {
    import hayride:core/ai-api@0.0.51;
}

world hayride-silo {
    include hayride:silo/imports@0.0.51;
}

world hayride-wac {
    include hayride:wac/imports@0.0.51;
}

```

:::info
Please refer to the [interface](../../reference/interfaces/) documentation for more details on each set of interfaces usage.
:::

## Runtime Morphs 

Hayride provides two primary morphs that are essential for its operation:
- **Server Morph**: The core component that handles incoming requests and manages the lifecycle of morphs.
- **CLI Morph**: The command-line interface that allows users to interact with the Hayride runtime, manage morphs, and perform various tasks.

As we continue to refine the cli and server, we plan to further expose their interfaces to allow for custom CLI and Server morphs to be created. 

