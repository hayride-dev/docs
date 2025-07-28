---
sidebar_position: 1
title: Agents
---

Hayride defines a set of WebAssembly Interfaces Types (WIT) that allow you to build components that export the `agent` interface.

An Agent is a component that interacts with an AI model, has the ability to use tools, and stores context of any interactions. 

This interface allows you to build agents that can be used in a variety of contexts, such as command line tools, or server-side applications.

In this example, we will create a simple agent.

:::tip
Hayride leverages the WebAssembly Component Model, which allows you to implement your own `agent` component. However, Hayride ships with a default implementation of the `agent` interface that you can use to get started quickly. 

This example is specifically to help guide in the process of creating your own agent implementations.
:::

## Prerequisites
- [Go](https://go.dev/doc/install) version 1.23.0+
- [TinyGo](https://tinygo.org/) version 0.33.0+


## Step 1: WIT Definition

Below is the WIT definition for the `agent` interface:

```wit
package hayride:ai@0.0.61;


interface agents {
    use types.{message};
    use context.{context};
    use model.{format};
    use hayride:mcp/tools@0.0.61.{tools};
    use hayride:mcp/types@0.0.61.{tool, call-tool-params, call-tool-result};
    use graph-stream.{graph-stream};
    use inference-stream.{graph-execution-context-stream};
    use wasi:io/streams@0.2.0.{output-stream};

    enum error-code {
        capabilities-error,
        context-error,
        compute-error,
        execute-error,
        unknown
    }

    resource error {
        /// return the error code.
        code: func() -> error-code;
        /// errors can propagated with backend specific status through a string value.
        data: func() -> string;
    }

    resource agent {
        constructor(name: string, instruction: string, format: format, graph: graph-execution-context-stream, tools: option<tools>, context: option<context>);
        name: func() -> string;
        instruction: func() -> string;
        capabilities: func() -> result<list<tool>, error>;
        context: func() -> result<list<message>, error>;
        compute: func(message: message) -> result<message, error>;
        execute: func(params: call-tool-params) -> result<call-tool-result, error>;
    }
}
```

The agent resource is defined by components implementing the `agent` interface. 

From the constructor, you can see it takes the following parameters:
- `name`: A string representing the name of the agent.
- `instruction`: A string that provides instructions for the agent.
- `format`: A format that specifies how messages should be formatted.
- `graph`: A graph execution context stream that allows the agent to execute graphs.
- `option<tools>`: An optional set of tools that the agent can use.
- `option<context>`: An optional context that provides additional information for the agent.

The `use` keywords indicate that the agent interface depends on other interfaces, such as `types`, `context`, `model`, `tools`, `graph-stream`, and `inference-stream`. These dependencies provide the necessary types and functionality for the agent to operate effectively. 

For now, we can treat them as normal WebAssembly imports and leave the implementation details for later. 

:::info
Using the WebAssembly Component Model, we can build other WebAssembly components that satisfy various interfaces, such as `tools`, meaning that the imports do not need to be strictly implemented by the host environment, or even the same language.
:::

Using this WIT definition, we can create a WebAssembly component that exports the `agent` interface.

```wit
package hayride:agents@0.0.1;

world default {
    include hayride:wasip2/imports@0.0.61;
    export hayride:ai/agents@0.0.61;
}
```

Place this WIT definition in a file called `world.wit` in a directory called `wit` at the root of your project.

## Step 2: Create a WIT Dependencies file

In the `wit` directory, create a `deps.toml` file to manage the dependencies for your WIT files.

This file will specify the dependencies required for your Morph:

```toml
wasip2 = "https://github.com/hayride-dev/coven/releases/download/v0.0.61/hayride_wasip2_v0.0.61.tar.gz"
ai = "https://github.com/hayride-dev/coven/releases/download/v0.0.61/hayride_ai_v0.0.61.tar.gz"
mcp = "https://github.com/hayride-dev/coven/releases/download/v0.0.61/hayride_mcp_v0.0.61.tar.gz"
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

## Step 4: Implement the Agent

Now that we have the WIT definition and the bindings dependency, we can implement the agent in Go.

Next, create a file called `main.go` in the root directory of your project. This file will contain the implementation of your Morph:

```bash
go mod init agent-example
touch main.go
```

### Step 4.1 Exports 

In the main.go file, implement the Morph:

We will start by setting the exported functions for the `agent` resource by calling the bindings `export.Agent` function with a constructor.


```go
package main

import (
	"github.com/hayride-dev/bindings/go/hayride/ai/agents/export"
)

func init() {
	export.Agent(constructor)
}
```

Note, the `export.Agent` function takes a constructor, which is a function type defined in bindings: 
`type Constructor func(name string, instruction string, format models.Format, graph graph.GraphExecutionContextStream, tools tools.Tools, context ctx.Context) (agents.Agent, error)`

This constructor will be called whenever the user of our Agent calls the agent constructor function. So we will return a new object that satisfies the 
`agents.Agent` interface:

```go
type Agent interface {
	Name() string
	Instruction() string
	Capabilities() ([]types.Tool, error)
	Context() ([]types.Message, error)
	Compute(message types.Message) (*types.Message, error)
	Execute(params types.CallToolParams) (*types.CallToolResult, error)
}
```

### Step 4.2 Implement the Constructor

Next, we will implement the `constructor` function. This function will be called when the agent is created.

```go
var _ agents.Agent = (*defaultAgent)(nil)

type defaultAgent struct {
	name        string
	instruction string
	format      models.Format
	graph       graph.GraphExecutionContextStream

	// Tools and Context are optional
	tools   tools.Tools
	context ctx.Context
}

func constructor(name string, instruction string, format models.Format, graph graph.GraphExecutionContextStream, tools tools.Tools, context ctx.Context) (agents.Agent, error) {
	if format == nil {
		return nil, fmt.Errorf("format is required for agent")
	}

	if graph == nil {
		return nil, fmt.Errorf("graph is required for agent")
	}

	agent := &defaultAgent{
		name:        name,
		instruction: instruction,
		tools:       tools,
		context:     context,
		format:      format,
		graph:       graph,
	}

	// If context is set, push the initial instruction message
	if context != nil {
		content := []types.MessageContent{}
		content = append(content, types.NewMessageContent(types.Text(instruction)))

		// If tools are set, list them and append to content
		if tools != nil {
			result, err := tools.List("")
			if err != nil {
				return nil, err
			}

			if result.Tools.Len() > 0 {
				// Append the list of tools to the content
				content = append(content, types.NewMessageContent(result.Tools))
			}
		}

		// Push message to the context
		msg := types.Message{Role: types.RoleSystem, Content: cm.ToList(content)}
		agent.context.Push(cm.Reinterpret[types.Message](msg))
	}

	return agent, nil
}
```

This `constructor` function initializes a new agent with the provided parameters and, if context is set, pushes an initial message to the agent's context.

The `constructor` function is called when a new agent is created, and it sets up the agent's initial state. Any component that imports the `agent` interface can call this function to create a new agent instance.

### Step 4.3 Implement the Interface Functions

Next, we will implement the `agents.Agent` functions, which may be called on an individual agent instance.

```go
func (a *defaultAgent) Name() string {
	return a.name
}

func (a *defaultAgent) Instruction() string {
	return a.instruction
}

func (a *defaultAgent) Capabilities() ([]types.Tool, error) {
	if a.tools == nil {
		return nil, fmt.Errorf("tools are not set for agent %s", a.name)
	}

	result, err := a.tools.List("")
	if err != nil {
		return nil, err
	}

	return result.Tools.Slice(), nil
}

func (a *defaultAgent) Context() ([]types.Message, error) {
	if a.context == nil {
		return nil, fmt.Errorf("context is not set for agent %s", a.name)
	}

	msgs, err := a.context.Messages()
	if err != nil {
		return nil, err
	}

	return msgs, nil
}

func (a *defaultAgent) Compute(message types.Message) (*types.Message, error) {
	var msgs []types.Message
	// Push message to context
	if a.context != nil {
		if err := a.context.Push(message); err != nil {
			return nil, fmt.Errorf("failed to push message to context: %w", err)
		}
		// Get all context messages
		m, err := a.context.Messages()
		if err != nil {
			return nil, fmt.Errorf("failed to get context messages: %w", err)
		}

		msgs = m
	} else {
		msgs = []types.Message{message}
	}

	// Format encode the messages
	data, err := a.format.Encode(msgs...)
	if err != nil {
		return nil, fmt.Errorf("failed to encode context messages: %w", err)
	}

	// Call Graph Compute
	d := graph.TensorDimensions(cm.ToList([]uint32{1}))
	td := graph.TensorData(cm.ToList(data))
	t := graph.NewTensor(d, graph.TensorTypeU8, td)
	inputs := []graph.NamedTensor{
		{
			F0: "user",
			F1: t,
		},
	}
	namedTensorStream, err := a.graph.Compute(inputs)
	if err != nil {
		return nil, fmt.Errorf("failed to compute graph: %w", err)
	}

	stream := namedTensorStream.F1
	ts := graph.TensorStream(stream)
	// read the output from the stream
	text := make([]byte, 0)
	for {
		// Read up to 100 bytes from the output
		// to get any tokens that have been generated and push to socket
		p := make([]byte, 100)
		len, err := ts.Read(p)
		if len == 0 || err == io.EOF {
			break
		} else if err != nil {
			return nil, fmt.Errorf("failed to read from tensor stream: %w", err)
		}
		text = append(text, p[:len]...)

		// TODO:: Optionally write RAW output to a writer
		// to get the raw output back faster, but would require an updated interface for agent compute
	}

	// Decode Message
	msg, err := a.format.Decode(text)
	if err != nil {
		return nil, fmt.Errorf("failed to decode message: %w", err)
	}

	// Push to Context if set
	if a.context != nil {
		if err := a.context.Push(*msg); err != nil {
			return nil, fmt.Errorf("failed to push message to context: %w", err)
		}
	}

	// Return the final message
	return msg, nil
}

func (a *defaultAgent) Execute(params types.CallToolParams) (*types.CallToolResult, error) {
	if a.tools == nil {
		return nil, fmt.Errorf("tools are not set for agent %s", a.name)
	}

	result, err := a.tools.Call(params)
	if err != nil {
		return nil, err
	}

	return result, nil
}
```

The `Name`, `Instruction`, `Capabilities`, and `Context` functions are mostly just used to get information from the agent.

The `Compute` function is one that does most of the work.

While this implementation is a bit complex, it essentially does the following:
- adds the input message to the agent's context
- retrieves all messages from the context
- encodes the messages using the agent's format
- computes the response using the agent's graph
- decodes the response using the agent's format
- pushes the response back to the agent's context
- returns the final message

Generally this Compute function would be called by an agent runner, which may additionally decide to use a tool through the Agent's `Execute` function.

Execute just takes the parameters of a tool call and uses the `Call` function of it's tools and returns the result or any error.

### Full Implementation

Here is the full implementation of the `main.go` file:

```go
package main

import (
	"fmt"
	"io"

	"github.com/hayride-dev/bindings/go/hayride/ai/agents"
	"github.com/hayride-dev/bindings/go/hayride/ai/agents/export"
	"github.com/hayride-dev/bindings/go/hayride/ai/ctx"
	"github.com/hayride-dev/bindings/go/hayride/ai/graph"
	"github.com/hayride-dev/bindings/go/hayride/ai/models"
	"github.com/hayride-dev/bindings/go/hayride/mcp/tools"
	"github.com/hayride-dev/bindings/go/hayride/types"
	"go.bytecodealliance.org/cm"
)

var _ agents.Agent = (*defaultAgent)(nil)

type defaultAgent struct {
	name        string
	instruction string
	format      models.Format
	graph       graph.GraphExecutionContextStream

	// Tools and Context are optional
	tools   tools.Tools
	context ctx.Context
}

func init() {
	export.Agent(constructor)
}

func constructor(name string, instruction string, format models.Format, graph graph.GraphExecutionContextStream, tools tools.Tools, context ctx.Context) (agents.Agent, error) {
	if format == nil {
		return nil, fmt.Errorf("format is required for agent")
	}

	if graph == nil {
		return nil, fmt.Errorf("graph is required for agent")
	}

	agent := &defaultAgent{
		name:        name,
		instruction: instruction,
		tools:       tools,
		context:     context,
		format:      format,
		graph:       graph,
	}

	// If context is set, push the initial instruction message
	if context != nil {
		content := []types.MessageContent{}
		content = append(content, types.NewMessageContent(types.Text(instruction)))

		// If tools are set, list them and append to content
		if tools != nil {
			result, err := tools.List("")
			if err != nil {
				return nil, err
			}

			if result.Tools.Len() > 0 {
				// Append the list of tools to the content
				content = append(content, types.NewMessageContent(result.Tools))
			}
		}

		// Push message to the context
		msg := types.Message{Role: types.RoleSystem, Content: cm.ToList(content)}
		agent.context.Push(cm.Reinterpret[types.Message](msg))
	}

	return agent, nil
}

func (a *defaultAgent) Name() string {
	return a.name
}

func (a *defaultAgent) Instruction() string {
	return a.instruction
}

func (a *defaultAgent) Capabilities() ([]types.Tool, error) {
	if a.tools == nil {
		return nil, fmt.Errorf("tools are not set for agent %s", a.name)
	}

	result, err := a.tools.List("")
	if err != nil {
		return nil, err
	}

	return result.Tools.Slice(), nil
}

func (a *defaultAgent) Context() ([]types.Message, error) {
	if a.context == nil {
		return nil, fmt.Errorf("context is not set for agent %s", a.name)
	}

	msgs, err := a.context.Messages()
	if err != nil {
		return nil, err
	}

	return msgs, nil
}

func (a *defaultAgent) Compute(message types.Message) (*types.Message, error) {
	var msgs []types.Message
	// Push message to context
	if a.context != nil {
		if err := a.context.Push(message); err != nil {
			return nil, fmt.Errorf("failed to push message to context: %w", err)
		}
		// Get all context messages
		m, err := a.context.Messages()
		if err != nil {
			return nil, fmt.Errorf("failed to get context messages: %w", err)
		}

		msgs = m
	} else {
		msgs = []types.Message{message}
	}

	// Format encode the messages
	data, err := a.format.Encode(msgs...)
	if err != nil {
		return nil, fmt.Errorf("failed to encode context messages: %w", err)
	}

	// Call Graph Compute
	d := graph.TensorDimensions(cm.ToList([]uint32{1}))
	td := graph.TensorData(cm.ToList(data))
	t := graph.NewTensor(d, graph.TensorTypeU8, td)
	inputs := []graph.NamedTensor{
		{
			F0: "user",
			F1: t,
		},
	}
	namedTensorStream, err := a.graph.Compute(inputs)
	if err != nil {
		return nil, fmt.Errorf("failed to compute graph: %w", err)
	}

	stream := namedTensorStream.F1
	ts := graph.TensorStream(stream)
	// read the output from the stream
	text := make([]byte, 0)
	for {
		// Read up to 100 bytes from the output
		// to get any tokens that have been generated and push to socket
		p := make([]byte, 100)
		len, err := ts.Read(p)
		if len == 0 || err == io.EOF {
			break
		} else if err != nil {
			return nil, fmt.Errorf("failed to read from tensor stream: %w", err)
		}
		text = append(text, p[:len]...)

		// TODO:: Optionally write RAW output to a writer
		// to get the raw output back faster, but would require an updated interface for agent compute
	}

	// Decode Message
	msg, err := a.format.Decode(text)
	if err != nil {
		return nil, fmt.Errorf("failed to decode message: %w", err)
	}

	// Push to Context if set
	if a.context != nil {
		if err := a.context.Push(*msg); err != nil {
			return nil, fmt.Errorf("failed to push message to context: %w", err)
		}
	}

	// Return the final message
	return msg, nil
}

func (a *defaultAgent) Execute(params types.CallToolParams) (*types.CallToolResult, error) {
	if a.tools == nil {
		return nil, fmt.Errorf("tools are not set for agent %s", a.name)
	}

	result, err := a.tools.Call(params)
	if err != nil {
		return nil, err
	}

	return result, nil
}

func main() {}
```

## Step 5: Build the Morph

To build the Morph, we will use TinyGo to compile the Go code into a WebAssembly component.

```bash
tinygo build -target wasip2 --wit-package ./wit/ --wit-world default -o default.wasm main.go
```

This command will compile the Morph to a WebAssembly binary. The `--wit-package` flag specifies the directory containing the WIT files, and the `--wit-world` flag specifies the name of the WIT world to use.

## Step 6: Register the Morph

```bash
hayride register --bin default.wasm --package hayride-ai:default@0.0.1
```

# Next steps    

You can now use the Morph in your Hayride applications. 

The morph can be composed with another Morph that imports the `hayride:ai/agents` interface, allowing you to build more complex agents that can interact with other components in the Hayride ecosystem.

For more examples on extending the `agent` interface, checkout the following examples: 
- Context: [context](./context.md)
- Tools: [tools](./tools.md)
- Model: [model formatting](./model-formatting.md)