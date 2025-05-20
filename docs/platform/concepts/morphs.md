---
sidebar_position: 3
title: Morphs
---

**Morphs** are portable, composable WebAssembly components. In Hayride, Morphs are the fundamental building blocks of applications. They can import host functions to access external capabilities and can also expose their own capabilities to other Morphs. 

The term **Morph** simply refers to a WebAssembly component that is designed to be composable and portable across different environments. 

Morphs are defined using WebAssembly Interface Types (WIT), which specify the imports and exports of the Morph. This allows Morphs to be portable across architectures, runtimes, and languages.

## Morph Attributes

- **Component Model-Based ( WASI 0.2 ):** Morphs are built using the Wasm Component Model, not just traditional Wasm modules.
    - This enables typed imports/exports, interface abstraction, and composition of components.
    - Interoperability between components is handled through canonical ABI and interface types (e.g. strings, records, lists) instead of raw byte buffers.
- **Guest Language-Agnostic:** Morphs can be written in any language that compiles to Wasm and supports WIT (WebAssembly Interface Types), including:
    - Rust
    - Go/TinyGo
    - JavaScript/AssemblyScript
    - C/C++ 
    - Python 
- **Import/Export via WIT Definitions:** Morphs define and expose typed interfaces via .wit files (WebAssembly Interface Types).
    - Imports are declared capabilities, allowing the Hayride host or other Morphs to fulfill them (e.g., logging, HTTP, clock).
    - Exports are commands or functions a Morph that can be imported by the runtime or other Morphs.
- **Sandboxed:** Morphs execute in sandboxed environments with no implicit access to the host system.
    - All host access (files, time, networking) must be granted explicitly and comes via WASI or custom host imports.
- **Composable:** Multiple Morphs can be linked together at compile time or programmatically using the component model’s support for composition.
    - This enables reusable, modular behaviors like filters, validators, or data transformers.
- **Runtime Agnostic, Hayride Native**
    - While Morphs are portable across any runtime that supports WASI 0.2, Hayride provides a runtime environment tailored for executing Morphs that import native Hayride host functions.
- **Versioned:** Morphs can be versioned using semantic versioning, allowing for backward compatibility and safe upgrades.

:::info
The component model is rapidly evolving, and while many languages have support for compiling to WebAssembly Components, the ecosystem is still maturing. Please refer to the [component model documentation](https://component-model.bytecodealliance.org/introduction.html) for the latest updates on language support and best practices.
:::


## Developing Morphs
Morphs can be developed in any language that compiles to WebAssembly, such as Rust, AssemblyScript, or C. This flexibility allows developers to choose the best tools for their specific use cases.

<div style={{ textAlign: 'center' }}>
    <img src={require('./assets/morph-langs.png').default} width="500" height="300" />
</div>

:::info
WebAssembly Components are apart of the WASI 0.2 specification, which defines the component model and how components can interact with each other and with the host environment. Many languages have support for compiling to WebAssembly Components, including Rust, AssemblyScript, C, and more, however, support is still evolving. 
:::

Check out our [language guide](../languages/) for more information on how to develop Morphs in different languages.

## Morph Composition

Morphs can be composed together to form a composition. In the simplest terms, a composition is a collection of components that are instantiated in a topological order and certain exports from those instances are made available as exports of the composition itself. Composition can happen at compile time or programmatically, allowing for flexible and dynamic application architectures.

Since morphs are WebAssembly components, they can be linked together using the component model's support for composition.

Hayride relies heavily on existing tools to aid in the composition of Morphs. 

Currently, we recommend using `wac`, which is a tool that uses the WAC (pronounced "whack") language to define how components composed together. WAC is a superset of the WIT language.


### Example Composition 

Below we will walk through a simple example of how to compose morphs using the `wac` tool.

First we will take a look the wit definition of the Morphs we want to compose.

### Hello Morph WIT

The hello component exports a function hello which returns a string:
```
package hayride:components;

world hello {
  export hello: func() -> string;
}
```

### Greeter Morph WIT

```
package hayride:components;

world greeter {
  import hello: func() -> string;

  export greet: func() -> string;
}
```

The greeter component imports the hello function and exports a greet function that returns a string. In this example, the greeter world imports a function signature that matches the hello function signature defined by our hello component. This allows the greeter component to be composed with the hello component, using the hello function exported by the hello component.

The `wac` tool support multiple was to compose components together but a simple script example of the composition would look like this:

```
package hayride:composition;

// Instantiate the `hello` component
let hello = new example:hello {};

// Instantiate the `greeter` component plugging its one `hello` import with
// the `hello` export of the `hello` component.
let greeter = new example:greeter {
  hello: hello.hello,
};

// Export the greet function from the greeter component
export greeter.greet;
```

### Compiling and Executing the Composition
We can then compile this composition using the `wac` tool, which will generate a WebAssembly component that can be executed in the Hayride runtime.

```sh
wac compose script.wac -o composed.wasm
```

We can execute the composed component in Hayride using the `hayride` CLI:

```sh
hayride init 
hayride register --morph composed.wasm --package hayride
hayride cast --morph hayride:composed  --function greet
```

:::info 
The above example was pulled from [WAC examples](https://github.com/bytecodealliance/wac/tree/main/examples), for a more complete overview, please refer to the WAC documentation.
:::

