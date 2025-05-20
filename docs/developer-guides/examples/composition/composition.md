---
title: Composition
sidebar_position: 1
description: WebAssembly composition using Hayride
---

## WebAssembly Composition with Hayride

Hayride supports composing modular WebAssembly components using [WAC](https://github.com/bytecodealliance/wac),
a tool developed by the Bytecode Alliance. This enables developers to break down complex applications into smaller, 
reusable components that expose their functionality through [WIT interfaces](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md).

These components can then be composed into a complete application that runs on the Hayride runtime.

:::tip
For more tools related to WebAssembly composition, see the [Tools Reference](../../tools.md).
:::

## Hayride CLI Integration with WAC

Hayride includes native support for WAC commands in its CLI, providing a streamlined developer experience for composing and managing components via the Hayride registry.

### hayride wac plug

The plug command provides a fast way to wire together components by resolving interfaces (sockets) and implementations (plugs) automatically. Components can be fetched from the Hayride registry using a package reference.

**Example**

To plug a datetime utility into the appropriate import of an AI agent component:

```
hayride wac plug --socket agent.wasm --plugs hayride:datetime@0.0.1
```

This automatically connects the datetime component to the matching interface in the agent.wasm component.

### hayride wac compose

The compose command allows you to build more complex applications by explicitly defining how multiple components are connected using a WAC composition file.

Registered Hayride components can be referenced using their package names and versions.

**Example Composition File**

Below is a WAC composition file for an AI agent that connects memory, model, and datetime components, and exports a complete server:

```
package hayride:agent;

let context = new hayride:inmemory@0.0.1 {...};

let llama = new hayride:llama31@0.0.1 {...};

let datetime = new hayride:datetime@0.0.1 {...};

let agent = new hayride:basic@0.0.1 {
  context: context.context,
  model: llama.model,
  datetime: datetime.datetime,
  ...
};

let server = new hayride-core:ai-server-cfg@0.0.1 {
  agent: agent.agent,
  ...
};

// Export the server handler and config
export server...;
```

**Building the Composed Component**

To build the composed component into a standalone `.wasm` file:

```
hayride wac compose --path basic-agent.wac --out server.wasm
```

This will resolve all referenced components, apply the specified wiring, and produce a deployable server.wasm.
