---
title: HTTP Server
sidebar_position: 2
description: Simple HTTP Server using Hayride
---

In this example we will create a simple HTTP server using Hayride. The server will listen on a specified port and respond with a "Hello, World!" message when accessed. 

The WebAssembly component that is produced is a Command component that can be run in the Hayride environment.

## Prerequisites
- [Go](https://go.dev/doc/install) version 1.23.0+
- [TinyGo](https://tinygo.org/) version 0.33.0+

## Step 1: Create a new directory
Create a new directory for your project and navigate into it:

```bash
mkdir http-server
cd http-server
```

## Step 2: Create a WIT file
First, create the necessary WIT files. In the root level of the project, create a directory called `wit` and create a file called `client.wit` inside it:

```bash
mkdir wit
touch wit/world.wit
```

In the `wit/world.wit` file, define a world that will include the necessary imports and exports for a HTTP Server.

In this case, we will be using the `hayride:wasip2/imports` and `hayride:wasip2/exports` modules to provide the necessary WASI functionality. 

Addtionally we will be using the `hayride:http/server` module to provide the HTTP Server functionality. 

```wit
package hayride-examples:http@0.0.1;

world server {
    include hayride:wasip2/imports@0.0.47;
    include hayride:wasip2/exports@0.0.47;
 
    include hayride:http/server@0.0.47;
}
```

## Step 3: Create a WIT Dependencies file

In the wit directory, create a `deps.toml` file to manage the dependencies for your WIT files. 

This file will specify the dependencies required for your Morph:

```toml
wasip2 = "https://github.com/hayride-dev/coven/releases/download/v0.0.47/hayride_wasip2_v0.0.47.tar.gz"
hayride-http = "https://github.com/hayride-dev/coven/releases/download/v0.0.47/hayride_http_v0.0.47.tar.gz""
```

Using `wit-dep`, we can pull in the dependencies for our WIT files. 

From the root directory of your project, run the following command:

```bash
wit-dep update
```

This will download the dependencies specified in the `deps.toml` file and place them in the `wit/deps` directory.

The directory structure should look like this:

```wit/deps
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
go mod init http-server
touch main.go
```


In the `main.go` file, implement the Morph:

```go
package main

import (
	"net/http"

	"github.com/hayride-dev/bindings/go/exports/net/http/server"
)

func init() {
	mux := http.NewServeMux()
	mux.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "text/plain")
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("Hello, World!"))
	})

	server.Export(mux, server.Config{Address: "localhost:9000"})
}

func main() {}
```

This is a simple Go program that creates a new HTTP server and registers a handler for the `/hello` endpoint. When accessed, it responds with "Hello, World!".

The `server.Export` function is a Hayride binding that allows the HTTP server to be exported as a WebAssembly component. This is equivalent to http.ListenAndServe in the standard library, but it is specifically designed to work with Hayride's WebAssembly environment.

While the hayride are being used in this example, any http package that implements the `wasi-http` interface can be used.

## Step 5: Build the Morph
To build the Morph, run the following command:

```bash
tinygo build -target wasip2 --wit-package ./wit/ --wit-world server
```

This command will compile the Morph to a WebAssembly binary. The `--wit-package` flag specifies the directory containing the WIT files, and the `--wit-world` flag specifies the name of the WIT world to use.

## Step 6: Register the Morph 

To register the Morph with the Hayride platform, we can use the `hayride` CLI. First, initialize the Hayride environment:

```bash
hayride register --bin server.wasm --package hayride-examples:server@0.0.1
```

## Step 7: Cast the Morph

To cast the Morph, use the `hayride` CLI:

```bash
hayride cast --package hayride-examples:server@0.0.1
```

using `curl` we can test the server:

```bash
curl -X GET http://localhost:9000/hello
```
You should see the following output:

```bash
Hello, World!
```

# What's Next? 

Now that you have created a simple HTTP server using Hayride, you can explore more advanced features and functionalities. Here are some ideas for what you can do next:

- [HTTP Client](./client.md): Learn how to create an HTTP client using Hayride.
- [HTTP Client Reactor](./client-reactor.md): Explore how to create a Reactor component that uses the HTTP client.
- [HTTP Server + Client](./server-client.md): Learn how to create a HTTP Server Client using Hayride.



