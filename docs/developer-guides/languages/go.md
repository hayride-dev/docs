---
sidebar_position: 1
title: Go
description: Building Morphs with Go
---

# Building Morphs with Go

Morphs can be built using Go tooling that supports WebAssembly (Wasm) and WASI. This guide covers how to get started with Go, TinyGo, and other Go-based tools.

- **Go** has native support for WebAssembly and WASI 0.1, but not yet for WASI 0.2. While WASI 0.2 support is in [progress](https://github.com/golang/go/issues/65333). 

- **TinyGo** is a lightweight Go compiler optimized for WebAssembly and embedded targets. It aims for compatibility with standard Go while adding features and optimizations specific to Wasm and WASI. Use the latest version of TinyGo to benefit from ongoing improvements and full WASI 0.2 support.

:::info
We recommend using TinyGo, which fully supports WASI 0.2 as of version [0.33.0](https://github.com/tinygo-org/tinygo/releases/tag/v0.33.0)
:::

## Tooling

There are several tools available for building and publishing morphs in Go. 

We recommend using the following tools for Go/TinyGo:  

- **[wit-bindgen-go](https://github.com/bytecodealliance/go-modules)**: A tool that generates Go bindings for WIT interfaces and worlds. It generates a Go package for each WIT world and interface, with the necessary types, functions, methods, and ABI glue code. 
- **[hayride bindings](https://github.com/hayride-dev/bindings)**: Official Go/TinyGo bindings for Hayride, providing optional, open-source support for Hayride-specific features like the agent framework and tool registry. Built on top of wit-bindgen-go, this package adds ergonomic wrappers for seamless integration with Hayride. Refer to the bindings guide [here](../../sdk-reference/go.md).

:::info 
For a complete list of recommended tools, see the [Tooling](../tools/tools.md) guide.
:::

## Getting Started with Go

The following steps will guide you through building a Morph using Go and TinyGo. We will use the [hayride bindings](https://github.com/hayride-dev/bindings),[wit-bindgen-go](https://github.com/bytecodealliance/go-modules), and the [TinyGo](https://tinygo.org/) toolchain to create a simple Morph that can be deployed to the Hayride runtime. 

This Morph will leverage the Hayride bindings to perform an HTTP request to a public API and return the response.  We will set up the Morph as a *command* module. This will allow us to execute the Morph directly through main, which can be useful for testing and debugging. Alternatively, we could define the Morph as a *reactor* module, which would allow it to be invoked through hayride or other Morphs.

### Prerequisites
The following tools are required to build and run this Morph:
- [Go](https://go.dev/doc/install) version 1.23.0+
- [TinyGo](https://tinygo.org/) version 0.33.0+
- [wit-bindgen-go](https://github.com/bytecodealliance/go-modules/) version 0.6.2+
- [hayride bindings](https://github.com/hayride-dev/bindings) version 0.0.25+

Please refer to the installation instructions for each tool to ensure you the correct installation and setup.

:::tip
See [Tools](../tools.md) for additional information on the Go tools.
:::

### Step 1: Create a new Go/TinyGo Project

Create a new directory for your Morph project and initialize a Go module:

```sh
mkdir http-client-morph

cd http-client-morph

go mod init http-client-morph
```

### Step 2: Define the WIT Interface

Next, in the root directory of your project, create a wit directory and define a WIT interface for your Morph. 

```sh
mkdir wit
touch wit/world.wit
```

In the `wit/world.wit` file, define the interface for your Morph. This interface will include a function to perform an HTTP request:

```wit
package hayride:http-morphs@0.0.1;

world http-client {
    include hayride:wasip2/imports@0.0.59;
    include hayride:wasip2/exports@0.0.59;
    include hayride:http/client@0.0.59;
}
```

In the above WIT file, we define a world called `http-client` that includes the necessary imports and exports for the Morph. Specifically, we include the `hayride:wasip2/imports` and `hayride:wasip2/exports` modules to provide the necessary WASI functionality, and the `hayride:http/client` module to provide HTTP client capabilities.

When compiling with TinyGo, we will include this WIT file to ensure the Wasm binary is generated with the correct imports and exports. 

The *include* statements reference additional worlds defined by Hayride. They provide a short hand way to include all of the necessary imports and exports for WASI 0.2 and HTTP client functionality.

The long hand version of the above WIT file would look like this:

```wit
package hayride:http-morphs@0.0.1;

world http-client {
  import wasi:clocks/wall-clock@0.2.0;
  import wasi:io/poll@0.2.0;
  import wasi:clocks/monotonic-clock@0.2.0;
  import wasi:random/random@0.2.0;
  import wasi:io/error@0.2.0;
  import wasi:io/streams@0.2.0;
  import wasi:filesystem/types@0.2.0;
  import wasi:filesystem/preopens@0.2.0;
  import wasi:cli/stdout@0.2.0;
  import wasi:cli/stderr@0.2.0;
  import wasi:cli/stdin@0.2.0;
  import wasi:cli/environment@0.2.0;

  import wasi:http/types@0.2.0;
  import wasi:http/outgoing-handler@0.2.0;

  export wasi:cli/run@0.2.0;
}
```

There are a number of ways to add dependencies to WIT, Hayride leverages wit-deps to manage dependencies. You can learn more about wit-deps [here](../tools/wit-deps.md).

### Step 3: Write the Morph Code

Create a new file called `main.go` in the root directory of your project. This file will contain the main logic for your Morph. 

```sh
touch main.go
```

In `main.go`, write the code to perform an HTTP request using the Hayride bindings:

```go
package main

import (
	"fmt"
	"io"

	"net/http"

	"github.com/hayride-dev/bindings/go/wasi/net/http/transport"
)

func main() {
	client := &http.Client{
		Transport: transport.New(),
	}

	resp, err := client.Get("https://postman-echo.com/get?foo1=bar1&foo2=bar2")
	if err != nil {
		fmt.Println("error making GET request:", err)
		return
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		fmt.Println("unexpected status code:", resp.StatusCode)
		return
	}
	fmt.Println("GET request successful:", resp.Status)
	body, err := io.ReadAll(resp.Body)
	if err != nil {
		fmt.Println("error reading response body:", err)
		return
	}

	fmt.Println(string(body))
}
```

In this code, we create an HTTP client using the `transport.New()` function from the Hayride bindings. This allows us to make HTTP requests within the Wasm environment. We then perform a GET request to a public API and print the response. 

HTTP is an example of a capability that is not yet supported by the Go standard library in Wasm. However, the Hayride bindings provide a way to use HTTP in a Wasm environment that support WASI 0.2.

### Step 4: Compile the Morph

Using TinyGo, we can compile the Morph to a Wasm binary. Run the following command in the root directory of your project:

```sh
tinygo build -target wasip2 --wit-package ./wit/ --wit-world http-client
```

This command compiles the `main.go` file into a Wasm binary, including the WIT interface defined in `wit/world.wit`. The `-target wasip2` flag specifies that we are targeting the WASI 0.2 environment.

The `--wit-package` flag specifies the directory containing the WIT files, and the `--wit-world` flag specifies the world to include in the compilation.


The compiled Wasm binary can be executed in a WASI 0.2 environment, such as `Wasmtime` or `Hayride`. 

### Step 5: Run the Morph with Wasmtime

While some Morphs require imported functions unique to Hayride, this Morph can be run using Wasmtime.

To run the Morph using Wasmtime, you can use the following command:

```sh   
 wasmtime -S http=y main.wasm
```

The output should look like this:

```json
GET request successful: OK
{
  "args": {
    "foo1": "bar1",
    "foo2": "bar2"
  },
  "headers": {
    "host": "postman-echo.com",
    "x-request-start": "t1745873359.754",
    "connection": "close",
    "x-forwarded-proto": "https",
    "x-forwarded-port": "443",
    "x-amzn-trace-id": "Root=1-680fe9cf-28c56a2c3b90957c466377ea"
  },
  "url": "https://postman-echo.com/get?foo1=bar1&foo2=bar2"
}
```

### Step 6: Run the Morph with Hayride

To run the Morph using Hayride, you can use the following command:

```sh
hayride register --bin main.wasm --package example:main@0.0.1
hayride cast --package example:main@0.0.1 --interactive
```

## Conclusion

In this guide, we have walked through the process of building a Morph using Go and TinyGo. We defined a WIT interface, implemented the Morph logic, and compiled it to a Wasm binary. We also demonstrated how to run the Morph using both Wasmtime and Hayride.

:::tip
For more detailed information on how to build WebAssembly components with Go, we highly recommend reading the [Go language guide](https://component-model.bytecodealliance.org/language-support/go.html).
:::