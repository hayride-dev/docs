---
sidebar_position: 2
title: Context
---

Hayride defines a set of WebAssembly Interfaces Types (WIT) that allow you to build componenets that export the `context` interface.

A component that exports the `context` inferface defines how messages that are processed by and agent are store/retrieved. Typically, this is used to store the history of messages that have been processed by the agent, but it can also be used to store other information that is relevant to the agent's processing.

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
package hayride:ai@0.0.60;

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
    include hayride:wasip2/imports@0.0.60;
    export hayride:ai/context@0.0.60;
}
```

Place this WIT definition in a file called `world.wit` in a directory called `wit` at the root of your project.

## Step 2: Create a WIT Dependencies file

Since the world imports wasip2 and the context interface, we need to create a `deps.toml` file to manage the dependencies for our WIT files.

In the `wit` directory, create a `deps.toml` file with the following content:

```toml
wasip2 = "https://github.com/hayride-dev/coven/releases/download/v0.0.60/hayride_wasip2_v0.0.60.tar.gz"
ai = "https://github.com/hayride-dev/coven/releases/download/v0.0.60/hayride_ai_v0.0.60.tar.gz"
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

## Step 3: Build the WebAssembly Component

To build the WebAssembly component, we will use the `wit-bindgen-go` tool to generate the necessary code from the WIT definition.

```bash
wit-bindgen-go generate --world hayride:contexts/in-memory --out ./internal/gen ./wit
```

This command will generate the Go code for the `context` interface in the `internal/gen` directory.

## Step 4: Implement the Context

Now that we have the WIT definition and the generated code, we can implement the agent in Go.

Next, create a file called `main.go` in the root directory of your project. This file will contain the implementation of your Morph:

```bash
go mod init context-example
touch main.go
```

### Step 4.1 Exports 
In the main.go file, implement the Morph:

We will start by setting the exported functions for the `context` resource.


```go
package main

import (
	"unsafe"

	"github.com/hayride-dev/morphs/components/ai/contexts/internal/gen/hayride/ai/context"
	"github.com/hayride-dev/morphs/components/ai/contexts/internal/gen/hayride/ai/types"
	"go.bytecodealliance.org/cm"
)

func init() {
	context.Exports.Context.Constructor = constructor
	context.Exports.Context.Push = push
	context.Exports.Context.Messages = messages
	context.Exports.Context.Destructor = destructor
}
```

Note, the `init` function is setting the exported functions for the `context` resource to the functions we will implement next (i.e `constructor`, `push`, `messages`, and `destructor`).

### Step 4.2 Implement the Constructor

The constructor is called when the context is created. It initializes the context and returns a pointer to the context resource.

```go
package main

import (
	"unsafe"

	"github.com/hayride-dev/morphs/components/ai/contexts/internal/gen/hayride/ai/context"
	"github.com/hayride-dev/morphs/components/ai/contexts/internal/gen/hayride/ai/types"
	"go.bytecodealliance.org/cm"
)

type resources struct {
	ctx map[cm.Rep]*inMemoryContext
}

type inMemoryContext struct {
	context []types.Message
}

func (c *inMemoryContext) push(msg context.Message) error {
	c.context = append(c.context, msg)
	return nil
}

func (c *inMemoryContext) messages() []context.Message {
	return c.context
}

var resourceTable = resources{
	ctx: make(map[cm.Rep]*inMemoryContext),
}

func init() {
	context.Exports.Context.Constructor = constructor
	context.Exports.Context.Push = push
	context.Exports.Context.Messages = messages
	context.Exports.Context.Destructor = destructor
}

func constructor() context.Context {
	ctx := &inMemoryContext{
		context: make([]types.Message, 0),
	}

	key := cm.Rep(uintptr(unsafe.Pointer(ctx)))
	v := context.ContextResourceNew(key)
	resourceTable.ctx[key] = ctx
	return v
}
```

The `constructor` function creates a new instance of the `inMemoryContext` struct, which will hold the messages in memory. We store the context in a resourceTable and returnt he resource handle to the caller.

### Step 4.3 Implement the Destructor

The destructor is called when the context is no longer needed. It cleans up the resources associated with the context.

```go
func destructor(self cm.Rep) {
	delete(resourceTable.ctx, self)
}
```

This `destructor` function cleans up the context resource when it is no longer needed and removes the agent from the resource table.


### Step 4.4 Implement the Push and Messages Methods

We can see above that our `inMemoryContext` struct has a `push` method that adds a message to the context and a `messages` method that retrieves all messages stored in the context.

We can access these through our resource table using the `cm.Rep` key.

```go
func push(self cm.Rep, msg context.Message) cm.Result[context.Error, struct{}, context.Error] {
	ctx, ok := resourceTable.ctx[self]
	if !ok {
		wasiErr := context.ErrorResourceNew(cm.Rep(context.ErrorCodePushError))
		return cm.Err[cm.Result[context.Error, struct{}, context.Error]](wasiErr)
	}

	if err := ctx.push(msg); err != nil {
		wasiErr := context.ErrorResourceNew(cm.Rep(context.ErrorCodePushError))
		return cm.Err[cm.Result[context.Error, struct{}, context.Error]](wasiErr)
	}
	return cm.Result[context.Error, struct{}, context.Error]{}
}

func messages(self cm.Rep) (result cm.Result[cm.List[context.Message], cm.List[context.Message], context.Error]) {
	ctx, ok := resourceTable.ctx[self]
	if !ok {
		wasiErr := context.ErrorResourceNew(cm.Rep(context.ErrorCodeMessageNotFound))
		return cm.Err[cm.Result[cm.List[context.Message], cm.List[context.Message], context.Error]](wasiErr)
	}

	return cm.OK[cm.Result[cm.List[context.Message], cm.List[context.Message], context.Error]](cm.ToList(ctx.messages()))
}
```

These functions handle the logic of access the context resource and performing the necessary operations.

## Full Implementation

Here is the full implementation of the `main.go` file:

```go
package main

import (
	"unsafe"

	"github.com/hayride-dev/morphs/components/ai/contexts/internal/gen/hayride/ai/context"
	"github.com/hayride-dev/morphs/components/ai/contexts/internal/gen/hayride/ai/types"
	"go.bytecodealliance.org/cm"
)

type resources struct {
	ctx map[cm.Rep]*inMemoryContext
}

type inMemoryContext struct {
	context []types.Message
}

func (c *inMemoryContext) push(msg context.Message) error {
	c.context = append(c.context, msg)
	return nil
}

func (c *inMemoryContext) messages() []context.Message {
	return c.context
}

var resourceTable = resources{
	ctx: make(map[cm.Rep]*inMemoryContext),
}

func init() {
	context.Exports.Context.Constructor = constructor
	context.Exports.Context.Push = push
	context.Exports.Context.Messages = messages
	context.Exports.Context.Destructor = destructor
}

func constructor() context.Context {
	ctx := &inMemoryContext{
		context: make([]types.Message, 0),
	}

	key := cm.Rep(uintptr(unsafe.Pointer(ctx)))
	v := context.ContextResourceNew(key)
	resourceTable.ctx[key] = ctx
	return v
}

func push(self cm.Rep, msg context.Message) cm.Result[context.Error, struct{}, context.Error] {
	ctx, ok := resourceTable.ctx[self]
	if !ok {
		wasiErr := context.ErrorResourceNew(cm.Rep(context.ErrorCodePushError))
		return cm.Err[cm.Result[context.Error, struct{}, context.Error]](wasiErr)
	}

	if err := ctx.push(msg); err != nil {
		wasiErr := context.ErrorResourceNew(cm.Rep(context.ErrorCodePushError))
		return cm.Err[cm.Result[context.Error, struct{}, context.Error]](wasiErr)
	}
	return cm.Result[context.Error, struct{}, context.Error]{}
}

func messages(self cm.Rep) (result cm.Result[cm.List[context.Message], cm.List[context.Message], context.Error]) {
	ctx, ok := resourceTable.ctx[self]
	if !ok {
		wasiErr := context.ErrorResourceNew(cm.Rep(context.ErrorCodeMessageNotFound))
		return cm.Err[cm.Result[cm.List[context.Message], cm.List[context.Message], context.Error]](wasiErr)
	}

	return cm.OK[cm.Result[cm.List[context.Message], cm.List[context.Message], context.Error]](cm.ToList(ctx.messages()))
}

func destructor(self cm.Rep) {
	delete(resourceTable.ctx, self)
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