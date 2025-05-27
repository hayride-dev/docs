---
title: Hello World
sidebar_position: 1
description: Hello World example for Hayride
---

This is a simple example of how to create a "Hello World" application using Hayride. We make the assumption that you have already installed the Hayride CLI and TinyGo. If you haven't done so, please refer to the [installation guide](../installation.md) for instructions.

If you are not familiar with the Hayride platform or the various WebAssembly tools, we recommend checking out the [Hayride Overview](../../overview.md) to get a better understanding of the platform and its capabilities. 

## Prerequisites
- [Go](https://go.dev/doc/install) version 1.23.0+
- [TinyGo](https://tinygo.org/) version 0.33.0+

:::tip
See [Tools](../tools.md) for additional information on these tools.
:::

# Step 1: Create a new directory

Create a new directory for your project and navigate into it:

```bash
mkdir hello-world
cd hello-world
```

# Step 2: Create a WIT file

First, create the necessary WIT files. In the root level of the project, create a directory called `wit` and create a file called `world.wit` inside it:

```bash
mkdir wit
touch wit/world.wit
```

In the `wit/world.wit` file, define the interface for your Morph. This interface will include a function to return a greeting message:

```wit
package hayride-examples:hello-world@0.0.1;

world hello-world {
    include hayride:wasip2/imports@0.0.51;
    include hayride:wasip2/exports@0.0.51;
}
```

Since we simply be returning a string, we don't need to include any additional imports or exports. The `wasip2` imports and exports are sufficient for this example.

Our morph will be a `Command` component that returns a string. Which means, we will be exported the cli:run function from wasip2. 

The the above example, we are using the `hayride:wasip2/imports` and `hayride:wasip2/exports` modules to provide the necessary WASI functionality. These are shorthand references to the WASI imports and exports that are required for the Morph to function correctly.

To include these files we will use `wit-dep` to pull in the necessary dependencies. The `wit-dep` tool is a dependency manager for WIT files. 

## WIT Dependencies

In the wit directory, create a `deps.toml` file to manage the dependencies for your WIT files. 

This file will specify the dependencies required for your Morph:

```toml
wasip2 = "https://github.com/hayride-dev/coven/releases/download/v0.0.51/hayride_wasip2_v0.0.51.tar.gz"
```

Using `wit-dep`, we can pull in the dependencies for our WIT files. 

From the root directory of your project, run the following command:

```bash
wit-dep update
```

This will download the dependencies specified in the `deps.toml` file and place them in the `wit/deps` directory.

## Step 3: Create the Morph

Next, create a file called `main.go` in the root directory of your project. This file will contain the implementation of your Morph:

```bash
go mod init hello-world
touch main.go
```

In the `main.go` file, implement the Morph:
```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, World!")
	fmt.Println("This is a simple example to demonstrate the usage of the Hayride Platform.")
}
```
This is a simple Go program that prints "Hello, World!" and a message to the console.

Under the hood, we are using TinyGo's support for wasip2 in order to use the standard library functions of the fmt package.

### Step 4: Build the Morph

We can build the Morph using TinyGo. Run the following command in the root directory of your project:

```bash
tinygo build -target wasip2 --wit-package ./wit/ --wit-world hello-world -o hello-world.wasm
```

This command will compile the Morph to a WebAssembly binary called `hello-world.wasm`. The `--wit-package` flag specifies the directory containing the WIT files, and the `--wit-world` flag specifies the name of the WIT world to use.


### Step 5: Register the Morph

To register the Morph with the Hayride platform, we can use the `hayride` CLI. First, initialize the Hayride environment:

```bash
hayride register --bin hello-world.wasm --package hayride-examples:hello-world@0.0.1
```

### Step 6: Cast the Morph

To execute the Morph, we can use the `hayride` CLI to cast the Morph. Run the following command:

```bash
hayride cast --package hayride-examples:hello-world@0.0.1 -it 
```

This command will execute the Morph and print the output to the console. The `-it` flag indicates that we want to run the Morph in interactive mode.

You should see the following output:

```bash
Hello, World!
This is a simple example to demonstrate the usage of the Hayride Platform.
```
This indicates that the Morph has been successfully executed and the greeting message has been printed to the console.

# What's Next?
Now that you have created a simple "Hello World" Morph using Hayride, you can explore more advanced features and capabilities of the platform. Here are some suggestions for what to do next:

- [Bindings](./bindings/): Explore how to define and use WIT generated binding in your Morphs.
- [Http](./http/): Learn how to write a HTTP Server or Client using the Hayride platform.
- [AI](./ai/): Explore how to integrate AI capabilities into your Morphs.

