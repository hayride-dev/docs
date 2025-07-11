---
sidebar_position: 2
title: CLI
---

# Hayride CLI 

The Hayride CLI morph is a command-line tool that provides a convenient way to interact with the Hayride platform. It allows you to manage Morphs, run commands, and perform various operations related to the Hayride ecosystem. 

## WIT Definition

The CLI morph itself is a WebAssembly Component that can be represented through the following WIT definition:

```wit
world cli {
    include hayride:wasip2/imports@0.0.59;

    // silo imports
    import hayride:silo/threads@0.0.59;
    import hayride:silo/process@0.0.59;

    // wasi imports
    import wasi:http/outgoing-handler@0.2.0;

    // wac imports
    import hayride:wac/wac@0.0.59;

    // exports
    include hayride:wasip2/exports@0.0.59;
}
```

The cli world is made up of the wasip2 imports and exports as well as mixture of public and private imports from the Hayride. 

:::caution
The hayride platform is still in development and the CLI is subject to change. The WIT definition may not be fully stable and may be subject to change in future releases. 

Always refer to the git repository, [Coven](https://github.com/hayride-dev/coven), for the latest version of the CLI WIT definition.
:::

### Public Imports

The CLI includes the following `public` imports from the Hayride platform:
- `wasi:http/outgoing-handler@0.2.0`: A WebAssembly HTTP client that allows the CLI to make HTTP requests to the Hayride platform.
- `hayride:wac/wac@0.0.59`: WebAssembly Composition import that provides `WAC` functionality for composing WebAssembly components.

These imports provide the necessary functionality for the CLI to interact with the Hayride platform, including HTTP client capabilities, WAC (WebAssembly Component) imports, and core configuration management. Each of these imports is defined publicly in our [Coven](https://github.com/hayride-dev/coven) repository, allowing for additional implementations. 

### Private Imports/Exports

Hayride has `private` or `reserved` imports that are not intended for `public` use. These imports are restricted to the Hayride platform and are used internally by the CLI. 

They include:
- `hayride:silo/threads@0.0.59` and `hayride:silo/process@0.0.59`: Silo implements basic parallelism and concurrency primitives for the Hayride platform. It provides a way to run Morphs in parallel and manage their execution. This is not intended for public use and is reserved for the Hayride platform. More holistic Async support is being discussed by the WebAssembly community and may replace this in the future.

:::info 
For a complete overview Hayride's WIT definition, please refer to the our [reference documentation](../../reference/interfaces/)
:::

### Future Improvements
The CLI is still in development and will be improved over time. Some of the planned improvements include:
- Moving CLI Command definitions to WIT to support better interoperability and extensibility
- Adding more commands and options for better usability
- Improving error handling and user feedback

## CLI Commands

To learn about the available commands, run the following command:

```bash
hayride help
```
This will display a list of available commands and their descriptions.

Each command has its own set of subcommands and options. You can get more information about a specific command by running:

```bash
hayride <command> --help
```

### Learn More 
To learn more about the CLI and its usage, refer to the following sections:
- [Developer Guide](../../developer-guides/examples/)
