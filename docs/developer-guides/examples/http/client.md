---
title: HTTP Client
sidebar_position: 1
description: Simple HTTP client using Hayride
---

In this example, we will create a simple HTTP client using Hayride. The client will make a GET request to a specified URL and print the response. 

The WebAssembly component that is produced is a Command component that can be run in the Hayride environment.

## Prerequisites
- [Go](https://go.dev/doc/install) version 1.23.0+
- [TinyGo](https://tinygo.org/) version 0.33.0+

:::tip
See [Tools](../../tools.md) for additional information on these tools.
:::

## Step 1: Create a new directory

Create a new directory for your project and navigate into it:

```bash
mkdir http-client
cd http-client
```
## Step 2: Create a WIT file

First, create the necessary WIT files. In the root level of the project, create a directory called `wit` and create a file called `world.wit` inside it:

```bash
mkdir wit
touch wit/world.wit
```

In the `wit/world.wit` file, define a world that will include the necessary imports and exports for a HTTP client.

In this case, we will be using the `hayride:wasip2/imports` and `hayride:wasip2/exports` modules to provide the necessary WASI functionality. 

Additionally we will be using the `hayride:http/client` module to provide the HTTP client functionality. 

```wit
package hayride-examples:http@0.0.1;

world client {
    include hayride:wasip2/imports@0.0.59;
    include hayride:wasip2/exports@0.0.59;
 
    include hayride:http/client@0.0.59;
}
```

## Step 3: Create a WIT Dependencies file

In the wit directory, create a `deps.toml` file to manage the dependencies for your WIT files. 

This file will specify the dependencies required for your Morph:

```toml
wasip2 = "https://github.com/hayride-dev/coven/releases/download/v0.0.59/hayride_wasip2_v0.0.59.tar.gz"
hayride-http = "https://github.com/hayride-dev/coven/releases/download/v0.0.59/hayride_http_v0.0.59.tar.gz""
```

Using `wit-deps`, we can pull in the dependencies for our WIT files. 

From the root directory of your project, run the following command:

```bash
wit-deps update
```

This will download the dependencies specified in the `deps.toml` file and place them in the `wit/deps` directory.

The directory structure should look like this:

```
wit/deps
├── cli
│   ├── command.wit
│   ├── environment.wit
│   ├── exit.wit
│   ├── imports.wit
│   ├── run.wit
│   ├── stdio.wit
│   └── terminal.wit
├── clocks
│   ├── monotonic-clock.wit
│   ├── wall-clock.wit
│   └── world.wit
├── filesystem
│   ├── preopens.wit
│   ├── types.wit
│   └── world.wit
├── hayride-http
│   └── world.wit
├── http
│   ├── handler.wit
│   ├── proxy.wit
│   └── types.wit
├── io
│   ├── error.wit
│   ├── poll.wit
│   ├── streams.wit
│   └── world.wit
├── random
│   ├── insecure-seed.wit
│   ├── insecure.wit
│   ├── random.wit
│   └── world.wit
├── sockets
│   ├── instance-network.wit
│   ├── ip-name-lookup.wit
│   ├── network.wit
│   ├── tcp-create-socket.wit
│   ├── tcp.wit
│   ├── udp-create-socket.wit
│   ├── udp.wit
│   └── world.wit
└── wasip2
    └── world.wit
```
## Step 4: Create the Morph

Next, create a file called `main.go` in the root directory of your project. This file will contain the implementation of your Morph:

```bash
go mod init http-client
touch main.go
```

In the `main.go` file, implement the Morph:

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

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		fmt.Println("error reading response body:", err)
		return
	}

	fmt.Println(string(body))
}
```

In this code, we create a new HTTP client using the `http.Client` struct and set the transport to `transport.New()`. This allows us to make HTTP requests using the WASI environment. The transport is provided by the `github.com/hayride-dev/bindings/go/wasi/net/http/transport` package, which is part of the Hayride HTTP bindings. However, any HTTP client that has implemented the `wasi-http` interface can be used.

In this example, we are making a GET request to `https://postman-echo.com/get?foo1=bar1&foo2=bar2` and printing the response body. You can modify the URL to test with different endpoints.

## Step 5: Build the Morph

To build the Morph, run the following command:

```bash
tinygo build -target wasip2 --wit-package ./wit/ --wit-world client
```

This command will compile the Morph to a WebAssembly binary. The `--wit-package` flag specifies the directory containing the WIT files, and the `--wit-world` flag specifies the name of the WIT world to use.

## Step 6: Register the Morph 

To register the Morph with the Hayride platform, we can use the `hayride` CLI. First, initialize the Hayride environment:

```bash
hayride register --bin client.wasm --package hayride-examples:client@0.0.1
```

## Step 7: Cast the Morph

To cast the Morph, use the `hayride` CLI:

```bash
hayride cast --package hayride-examples:http@0.0.1:client -it
```

This command will execute the Morph and print the output to the console. The `-it` flag indicates that we want to run the Morph in interactive mode.

You should see the output of the HTTP GET request printed to the console.
```
{
  "args": {
    "foo1": "bar1",
    "foo2": "bar2"
  },
  "headers": {
    "host": "postman-echo.com",
    "x-request-start": "t1747088411.159",
    "connection": "close",
    "x-forwarded-proto": "https",
    "x-forwarded-port": "443",
    "x-amzn-trace-id": "Root=1-6822741b-691d7eea1cec4fb95c7b704d"
  },
  "url": "https://postman-echo.com/get?foo1=bar1&foo2=bar2"
}
```

This indicates that the Morph has been successfully executed and the HTTP response has been printed to the console.

# What's Next? 

Now that you have created a simple HTTP client using Hayride, you may want to check out the following resources:

- [Hayride HTTP Server](./server.md): Learn how to create an HTTP server using Hayride.
- [HTTP Client Reactor](./client-reactor.md): Explore how to create a Reactor component that uses the HTTP client.

