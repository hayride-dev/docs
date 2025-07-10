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

The plug command provides a fast way to wire together components by resolving interfaces (sockets) and implementations (plugs) automatically.

Components are fetched from the Hayride registry using a package reference.

**Example**

To plug the hayride core cfg component into the appropriate import of the hayride core server component:

```
hayride wac plug --socket hayride-core:server@0.0.1 --plugs hayride-core:cfg@0.0.1
```

This automatically connects the cfg component to the matching interface to produce a full component.

### hayride wac compose

The compose command allows you to build more complex applications by explicitly defining how multiple components are connected using a WAC composition file.

Components are fetched from the Hayride registry using a package reference.

**Example Composition File**

Below is a WAC composition file for a cli that use Hayride AI imports for context, model, tools, and agents components, and exports a complete cli component:

```
package hayride:agent;

let context = new hayride:inmemory@0.0.1 {...};

let llama = new hayride:llama31@0.0.1 {...};

let tools = new hayride:default-tools@0.0.1 {...};

let agent = new hayride:default-agent@0.0.1 {
  context: context.context,
  model: llama.model,
  tools: tools.tools,
  ...
};

let cli = new hayride:cli@0.0.1 {
  context: context.context,
  model: llama.model,
  tools: tools.tools,
  agents: agent.agents,
  ...
};

// Export the cli
export cli...;
```

**Building the Composed Component**

To build the composed component into a standalone `.wasm` file:

```
hayride wac compose --path default-agent-cli.wac --out cli.wasm
```

This will resolve all referenced components, apply the specified wiring, and produce a deployable cli.wasm.
