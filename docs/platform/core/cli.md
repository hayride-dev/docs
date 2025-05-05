---
sidebar_position: 1
title: CLI
---

# Hayride CLI 

The Hayride CLI moprh is a command-line tool that provides a convenient way to interact with the Hayride platform. It allows you to manage Morphs, run commands, and perform various operations related to the Hayride ecosystem. 

## WIT Definition

The CLI moprh itself is a WebAssembly Component that can be represented through the following WIT definition:

```wit
world cli {
    include hayride:wasip2/imports@0.0.46;
    include hayride:wasip2/exports@0.0.46;

    include hayride:silo/imports@0.0.46;

    include hayride:http/client@0.0.46;
    include hayride:wac/imports@0.0.46;

    import hayride:core/config@0.0.46;
}
```

The cli world is made up of the wasip2 imports and exports as well as mixture of public and private imports from the Hayride. 

:::caution
The hayride platfrom is still in development and the CLI is subject to change. The WIT definition may not be fully stable and may be subject to change in future releases. 

Always refer to the git repository, [Coven](https://github.com/hayride-dev/coven), for the latest version of the CLI WIT definition.
:::

### Public Imports

The CLI includes the following `public` imports from the Hayride platform:
- `hayride:http/client@0.0.46`: A WebAssembly HTTP client that allows the CLI to make HTTP requests to the Hayride platform.
- `hayride:wac/imports@0.0.46`: WebAssembly Composition imports that provide `WAC` functionality for composing WebAssembly components.
- `hayride:core/config@0.0.46`: Core configuration management for the Hayride platform, allowing the CLI to access and manage configuration settings.
- `hayride:wasip2/imports@0.0.46`: WASI (WebAssembly System Interface) imports that provide standard system-level functionality for the CLI.
- `hayride:wasip2/exports@0.0.46`: WASI exports that provide standard system-level functionality for the CLI.
These imports provide the necessary functionality for the CLI to interact with the Hayride platform, including HTTP client capabilities, WAC (WebAssembly Component) imports, and core configuration management. Each of these imports is defined publicly in our [Coven](https://github.com/hayride-dev/coven) repository, allowing for additional implementations. 

### Private Imports/Exports

Hayride has `private` or `reserved` imports that are not intended for `public` use. These imports are restrcited to the Hayride platform and are used internally by the CLI. 

They include:
- `hayride:silo/imports@0.0.46`: Silo implements basic parallelism and concurrency primitives for the Hayride platform. It provides a way to run Morphs in parallel and manage their execution. This is not intended for public use and is reserved for the Hayride platform. More holistic Aysnc support is being discussed by the WebAssembly community and may replace this in the future.

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
