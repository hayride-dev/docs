---
sidebar_position: 2
---

# Installation

The easiest way to install Hayride is through our installation script.

You can download our installation script, but visiting our [releases repository](https://github.com/hayride-dev/releases/releases), or directly install with the following command:

```bash
curl https://raw.githubusercontent.com/hayride-dev/releases/refs/heads/main/install.sh -sSf | bash
```

:::info
Currently only MacOS x86_64 and ARM targets are supported. Cross platform installations coming soon!
:::

## Validating Installation

To validate that Hayride has been installed correctly, you can run the following command:

```bash
hayride help
```

If you installation was successful, you should see the Hayride help menu and a list of available commands.

## Hayride Directory 

By Default, the installation script will create a directory called `.hayride` in your home directory.

This directory will contain all of the necessary files and directories for Hayride to run.

The directory structure will look like this:

```bash
.hayride
├── ai
│   └── models
│       └── Meta-Llama-3.1-8B-Instruct-Q5_K_M.gguf
├── bin
│   ├── LICENSE.txt
│   └── hayride
├── config.yaml
├── logs
│   └── hayride.log
└── registry
    └── morphs
        ├── hayride
        │   └── 0.0.1
        │       ├── inmemory.wasm
        │       └── llama31.wasm
        └── hayride-core
            └── 0.0.1
                ├── ai-server-cfg.wasm
                ├── cli.wasm
                └── server-cfg.wasm
```     

:::warning
You can change the default directory by setting the `HAYRIDE_DIR` environment variable to a different path. However, this is not recommend. 
:::

Hayride ships with a set of core morphs and AI models. Refer to the [core](./platform/core/) documentation for more information on why these are included. 