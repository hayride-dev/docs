---
sidebar_position: 2
title: Context
---

Hayride defines a set of WebAssembly Interfaces Types (WIT) that allow you to build components that export the `context` interface.

A component that exports the `context` interface defines how messages that are processed by an agent are stored and retrieved. Typically, this is used to store the history of messages that have been processed by the agent, but it can also be used to store other information that is relevant to the agent's processing.

In this example we will create a simple context that stores messages in memory.

:::tip
Hayride leverages the WebAssembly Component Model, which allows you to implement your own `context` component. However, Hayride ships with a various implementations of the `context` interface that you can use to get started quickly. 

This example is specifically to help guide in the process of creating your own context implementations.
:::


## Prerequisites
- [Go](https://go.dev/doc/install) version 1.23.0+
- [TinyGo](https://tinygo.org/) version 0.33.0+

## Step 1: WIT Definition

Below is the WIT definition for the `context` interface. 

```wit
package hayride:ai@0.0.65;

interface context {
    use types.{message};

    enum error-code {
        unexpected-message-type,
        push-error,
        message-not-found,
        unknown
    }

    resource error {
        /// return the error code.
        code: func() -> error-code;
        /// errors can propagated with backend specific status through a string value.
        data: func() -> string;
    }


    resource context {
        constructor();
        push: func(msg: message) -> result<_, error>;
        messages: func() -> result<list<message>, error>;
    }
}
```

The `context` resource is defined by components that implement the `context` interface. 

The error resource provides a way to return an error code and additional data if needed.

The constructor creates a new context, a `push` method to add messages to the context, and a `messages` method to retrieve all messages stored in the context.

The push method takes a `message` as an argument and returns a result that can either be a success or an error. 

The `messages` method returns a list of messages stored in the context.

How you implement the `context` resource is up to you, but it should provide a way to store and retrieve messages.

Using the `context` interface, you can define a component that exports the context interface like the following:

```wit
package hayride:contexts@0.0.1;

world in-memory {
    include hayride:wasip2/imports@0.0.65;
    export hayride:ai/context@0.0.65;
}
```

Place this WIT definition in a file called `world.wit` in a directory called `wit` at the root of your project.

## Step 2: Create a WIT Dependencies file

Since the world imports wasip2 and the context interface, we need to create a `deps.toml` file to manage the dependencies for our WIT files.

In the `wit` directory, create a `deps.toml` file with the following content:

```toml
wasip2 = "https://github.com/hayride-dev/coven/releases/download/v0.0.65/hayride_wasip2_v0.0.65.tar.gz"
ai = "https://github.com/hayride-dev/coven/releases/download/v0.0.65/hayride_ai_v0.0.65.tar.gz"
mcp = "https://github.com/hayride-dev/coven/releases/download/v0.0.65/hayride_mcp_v0.0.65.tar.gz"
```

Using `wit-deps`, we can pull in the dependencies for our WIT files.

`wit-deps update` will download the dependencies specified in the `deps.toml` file and place them in the `wit/deps` directory.

The directory structure should look like this:

```
root/wit/
root/wit/deps/
root/wit/world.wit
root/wit/deps.toml
```

## Step 3: Import Bindings

Generally, to build the WebAssembly component, you would need to generate bindings that provide language specific code to interact with the WIT defined objects.
In Go you can do this with the `wit-bindgen-go` tool, however, Hayride has provided a repo that has pregenerated the necessary objects with helpful wrappers.
So all you need to do is add it as a go dependency:

```bash
go get github.com/hayride-dev/bindings
```

## Step 4: Implement the Context

Now that we have the WIT definition and the bindings dependency, we can implement the context in Go.

Next, create a file called `main.go` in the root directory of your project. This file will contain the implementation of your Morph:

```bash
go mod init context-example
touch main.go
```

### Step 4.1 Exports 

In the main.go file, implement the Morph:

We will start by setting the exported functions for the `context` resource by calling the bindings `export.Context` function with a constructor.


```go
package main

import (
	"github.com/hayride-dev/bindings/go/hayride/ai/ctx"
	"github.com/hayride-dev/bindings/go/hayride/ai/ctx/export"
	"github.com/hayride-dev/bindings/go/hayride/types"
)

func init() {
	export.Context(constructor)
}
```

Note, the `export.Context` function takes a constructor, which is a function type defined in bindings: `type Constructor func() (ctx.Context, error)`

This constructor will be called whenever the user of our Context calls the context constructor function. So we will return a new object that satisfies the 
`ctx.Constructor` interface:

```go
type Context interface {
	Push(messages ...types.Message) error
	Messages() ([]types.Message, error)
}
```

### Step 4.2 Implement the Constructor

The constructor is called when the context is created. It initializes the context and returns a struct that satisfies the `ctx.Context` interface.

```go
var _ ctx.Context = (*inMemoryContext)(nil)

type inMemoryContext struct {
	context []types.Message
}

func constructor() (ctx.Context, error) {
	return &inMemoryContext{
		context: make([]types.Message, 0),
	}, nil
}
```

The `constructor` function creates a new instance of the `inMemoryContext` struct, which will hold the messages in memory.

### Step 4.3 Implement the Push and Messages Methods

We can see above that our `inMemoryContext` struct needs to satisfy the `ctx.Context` interface which has a `Push` method that adds messages to the context and a `Messages` method that retrieves all messages stored in the context.

These are straightforward to implement for our in-memory example.

```go
func (c *inMemoryContext) Push(msg ...types.Message) error {
	c.context = append(c.context, msg...)
	return nil
}

func (c *inMemoryContext) Messages() ([]types.Message, error) {
	return c.context, nil
}
```

## Full Implementation

Here is the full implementation of the `main.go` file:

```go

package main

import (
	"github.com/hayride-dev/bindings/go/hayride/ai/ctx"
	"github.com/hayride-dev/bindings/go/hayride/ai/ctx/export"
	"github.com/hayride-dev/bindings/go/hayride/types"
)

var _ ctx.Context = (*inMemoryContext)(nil)

type inMemoryContext struct {
	context []types.Message
}

func (c *inMemoryContext) Push(msg ...types.Message) error {
	c.context = append(c.context, msg...)
	return nil
}

func (c *inMemoryContext) Messages() ([]types.Message, error) {
	return c.context, nil
}

func constructor() (ctx.Context, error) {
	return &inMemoryContext{
		context: make([]types.Message, 0),
	}, nil
}

func init() {
	export.Context(constructor)
}

func main() {}
```

## Step 5: Build the Morph

To build the Morph, we will use TinyGo to compile the Go code into a WebAssembly component.

```bash
tinygo build -target wasip2 --wit-package ./wit/ --wit-world in-memory -o in-memory-ctx.wasm main.go
```

This command will compile the Morph to a WebAssembly binary. The `--wit-package` flag specifies the directory containing the WIT files, and the `--wit-world` flag specifies the name of the WIT world to use.

## Step 6: Register the Morph

```bash
hayride register --bin in-memory-ctx.wasm --package hayride-ai:inmemory@0.0.1
```

# Next steps    

You can now use the Morph in your Hayride applications. 

The morph can be composed with another Morph that imports the `hayride:ai/context` interface, allowing you to build more complex agents that can interact with other components in the Hayride ecosystem.

For more examples, checkout the following examples: 
- Agents: [agents](./agents.md)
- Tools: [tools](./tools.md)
- Model: [model formatting](./model-formatting.md)
