---
sidebar_position: 4
title: Tools
---

Hayride defines a set of WebAssembly Interfaces Types (WIT) that allow you to build components that export the `tools` interface.

A component that exports the `tools` interface defines what tools are available and how they are called.
This provides a way for custom tools to be made available to an agent.

In this example we will create a simple tool that can provide the date and time to an agent.

:::tip
Hayride leverages the WebAssembly Component Model, which allows you to implement your own `tools` component. However, Hayride ships with a default implementation of the `tools` interface that you can use to get started quickly. 

This example is specifically to help guide in the process of creating your own tools implementations.
:::


## Prerequisites
- [Go](https://go.dev/doc/install) version 1.23.0+
- [TinyGo](https://tinygo.org/) version 0.33.0+

## Step 1: WIT Definition

Below is the WIT definition for the `tools` interface. 

```wit
package hayride:mcp@0.0.65;

interface tools {
    use types.{tool, content, call-tool-params, call-tool-result, list-tools-result};

    enum error-code {
        tool-call-failed,
        tool-not-found,
        unknown
    }

    resource error {
        /// return the error code.
        code: func() -> error-code;
        /// errors can propagated with backend specific status through a string value.
        data: func() -> string;
    }

    resource tools {
        constructor();
        list-tools: func(cursor: string) -> result<list-tools-result, error>;
        call-tool: func(params: call-tool-params) -> result<call-tool-result, error>;
    }
}
```

The `tools` resource is defined by components that implement the `tools` interface. 

The error resource provides a way to return an error code and additional data if needed.

The constructor creates a new instance of tools with a `list-tools` method to get tools, and a `call-tool` method to use a tool.

How you implement the `tools` resource is up to you, but it must provide a way to list and call tools.

Using the `tools` interface, you can define a component that exports the tools interface like the following:

```wit
package hayride:contexts@0.0.1;

world in-memory {
    include hayride:wasip2/imports@0.0.65;
    export hayride:mcp/tools@0.0.65;
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

## Step 4: Implement the Tools

Now that we have the WIT definition and the bindings dependency, we can implement the tools in Go.

Next, create a file called `main.go` in the root directory of your project. This file will contain the implementation of your Morph:

```bash
go mod init tools-example
touch main.go
```

### Step 4.1 Exports 

In the main.go file, implement the Morph:

We will start by setting the exported functions for the `tools` resource by calling the bindings `export.Tools` function with a constructor.


```go
package main

import (
    "github.com/hayride-dev/bindings/go/hayride/mcp/tools"
	"github.com/hayride-dev/bindings/go/hayride/mcp/tools/export"
	"github.com/hayride-dev/bindings/go/hayride/types"
)

func init() {
    export.Tools(constructor)
}
```

Note, the `export.Tools` function takes a constructor, which is a function type defined in bindings: `type Constructor func() (tools.Tools, error)`

This constructor will be called whenever the user of our Tools calls the tools constructor function. So we will return a new object that satisfies the 
`tools.Tools` interface:

```go
type Tools interface {
	Call(params types.CallToolParams) (*types.CallToolResult, error)
	List(cursor string) (*types.ListToolsResult, error)
}
```

### Step 4.2 Implement the Constructor

The constructor is called when the tools is created. It initializes the tools and returns a struct that satisfies the `tools.Tools` interface.

```go
var _ tools.Tools = (*defaultTools)(nil)

type defaultTools struct{}

func constructor() (tools.Tools, error) {
	return &defaultTools{}, nil
}
```

The `constructor` function creates a new instance of the `defaultTools` struct, which for this example is just an empty struct.

### Step 4.3 Implement the List and Call Methods

We can see above that our `defaultTools` struct needs to satisfy the `tools.Tools` interface which has a `List` method that returns a ListToolsResult and a `Call` method that calls a tool with params and returns a CallToolResult.

```go

func (n *defaultTools) Call(params types.CallToolParams) (*types.CallToolResult, error) {
	if params.Name != "datetime" {
		return nil, fmt.Errorf("unknown tool: %s", params.Name)
	}

	now := time.Now().Format(time.RFC3339)

	content := types.NewContent(types.TextContent{
		ContentType: "text",
		Text:        fmt.Sprintf("Current date and time: %s", now),
	})

	return &types.CallToolResult{
		Content: cm.ToList([]types.Content{content}),
	}, nil
}

func (n *defaultTools) List(cursor string) (*types.ListToolsResult, error) {
	return &types.ListToolsResult{
		Tools: cm.ToList([]types.Tool{
			{
				Name:        "datetime",
				Title:       "Datetime",
				Description: "Provides the current date and time.",
			},
		}),
	}, nil
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

See the [bindings](https://github.com/hayride-dev/bindings) and [coven](https://github.com/hayride-dev/coven) repositories for more details on the types that are being used.

## Step 5: Build the Morph

To build the Morph, we will use TinyGo to compile the Go code into a WebAssembly component.

```bash
tinygo build -target wasip2 --wit-package ./wit/ --wit-world default -o default.wasm main.go
```

This command will compile the Morph to a WebAssembly binary. The `--wit-package` flag specifies the directory containing the WIT files, and the `--wit-world` flag specifies the name of the WIT world to use.

## Step 6: Register the Morph

```bash
hayride register --bin default.wasm --package hayride-ai:tools@0.0.1
```

# Next steps    

You can now use the Morph in your Hayride applications. 

The morph can be composed with another Morph that imports the `hayride:ai/tools` interface, allowing you to build more complex agents that can interact with other components in the Hayride ecosystem.

For more examples, checkout the following examples: 
- Agents: [agents](./agents.md)
- Context: [context](./context.md)
- Model: [model formatting](./model-formatting.md)
