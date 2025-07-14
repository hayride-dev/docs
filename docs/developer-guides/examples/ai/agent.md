---
sidebar_position: 1
title: Agents
---

Hayride defines a set of WebAssembly Interfaces Types (WIT) that allow you to build componenets that export the `agent` interface.

This interface allows you to build agents that can be used in a variety of contexts, such as command line tools, or server-side applications.

In this example, we will create a simple agent loop. 

This example heavily uses generated bindings from `wit-bindgen-go`, which is a tool that generates Go code from WIT definitions. Unfortunately, can be a little difficult to follow, however once you start to understand using the bindings, you will find that it is fairly straightforward to fill in the implementation details.

In the future, we will be extending our bindings to provide more functionality and make it easier to work with the `agent` interface.

:::tip
Hayride leverages the WebAssembly Component Model, which allows you to implement your own `agent` interfaces. However, Hayride ships with a default implementation of the `agent` interface that you can use to get started quickly. 

This example is specifically to help guide in the process of creating your own agent implementations.
:::

## Prerequisites
- [Go](https://go.dev/doc/install) version 1.23.0+
- [TinyGo](https://tinygo.org/) version 0.33.0+


## Step 1: WIT Definition

Below is the WIT definition for the `agent` interface:

```wit
package hayride:ai@0.0.60;

interface agents {
    use types.{message};
    use context.{context};
    use model.{format};
    use tools.{tools};
    use graph-stream.{graph-stream};
    use inference-stream.{graph-execution-context-stream};
    use wasi:io/streams@0.2.0.{output-stream};

    enum error-code {
        invoke-error,
        unknown
    }

    resource error {
        /// return the error code.
        code: func() -> error-code;
        /// errors can propagated with backend specific status through a string value.
        data: func() -> string;
    }

    resource agent {
        constructor(name: string, instruction: string, tools: tools, context: context, format: format, graph: graph-execution-context-stream);
        invoke: func(input: message) -> result<list<message>, error>;
        invoke-stream: func(message: message, writer: output-stream) -> result<_,error>;
    }            
}
```

The agent resource is the main entry point for components implementing the `agent` interface. 

From the constructor, you can see it takes the following parameters:
- `name`: A string representing the name of the agent.
- `instruction`: A string that provides instructions for the agent.
- `tools`: A set of tools that the agent can use.
- `context`: A context that provides additional information for the agent.
- `format`: A format that specifies how messages should be formatted.
- `graph`: A graph execution context stream that allows the agent to execute graphs.

The `use` keywords indicate that the agent interface depends on other interfaces, such as `types`, `context`, `model`, `tools`, `graph-stream`, and `inference-stream`. These dependencies provide the necessary types and functionality for the agent to operate effectively. 

For now, we can treat them as normal WebAssembly imports and leave the implementation details for later. 

:::info
Using the WebAssembly Component Model, we can build other WebAssembly components that satisfy various interfaces, such as `tools`, meaning that the imports do not need to be strictly implemented by the host environement, or even the same language.
:::

Using this WIT definition, we can create a WebAssembly component that exports the `agent` interface.

```wit
package hayride:agents@0.0.1;

world default {
    include hayride:wasip2/imports@0.0.59;
    export hayride:ai/agents@0.0.59;
}
```

Place this WIT definition in a file called `world.wit` in a directory called `wit` at the root of your project.

## Step 2: Create a WIT Dependencies file

In the `wit` directory, create a `deps.toml` file to manage the dependencies for your WIT files.

This file will specify the dependencies required for your Morph:

```toml
wasip2 = "https://github.com/hayride-dev/coven/releases/download/v0.0.59/hayride_wasip2_v0.0.59.tar.gz"
ai = "https://github.com/hayride-dev/coven/releases/download/v0.0.59/hayride_ai_v0.0.59.tar.gz"
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
wit-bindgen-go generate --world hayride:agents/default --out ./internal/gen ./wit
```

This command will generate the Go code for the `agent` interface in the `internal/gen` directory.

## Step 4: Implement the Agent

Now that we have the WIT definition and the generated code, we can implement the agent in Go.

Next, create a file called `main.go` in the root directory of your project. This file will contain the implementation of your Morph:

```bash
go mod init agent-example
touch main.go
```


### Step 4.1 Exports 
In the main.go file, implement the Morph:

We will start by setting the exported functions for the `agent` interface.

```go
package main

import (
	"github.com/hayride-dev/morphs/components/ai/agents/internal/gen/hayride/ai/agents"
)

func init() {
	agents.Exports.Agent.Constructor = constructor
	agents.Exports.Agent.Invoke = invoke
	agents.Exports.Agent.InvokeStream = invokeStream
	agents.Exports.Agent.Destructor = destructor
}
```

Note, the `init` function is setting the exported functions for the `agent` interface to the functions we will implement next (i.e `constructor`, `invoke`, `invokeStream`, and `destructor`).

### Step 4.2 Implement the Constructor

Next, we will implement the `constructor` function. This function will be called when the agent is created.

```go

const maxturn = 10

var resourceTable = resources{
	agents: make(map[cm.Rep]*agent),
}

func init() {
	agents.Exports.Agent.Constructor = constructor
	agents.Exports.Agent.Invoke = invoke
	agents.Exports.Agent.InvokeStream = invokeStream
	agents.Exports.Agent.Destructor = destructor
}

type resources struct {
	agents map[cm.Rep]*agent
}

type agent struct {
	name    string
	tools   agents.Tools
	context agents.Context
	format  agents.Format
	graph   agents.GraphExecutionContextStream
}

func constructor(name string, instruction string, tools_ agents.Tools, context_ agents.Context, format agents.Format, graph agents.GraphExecutionContextStream) agents.Agent {
	agent := &agent{
		name:    name,
		tools:   tools_,
		context: context_,
		format:  format,
		graph:   graph,
	}

	content := []types.Content{}
	content = append(content, types.ContentText(types.TextContent{
		Text: instruction,
	}))

	result := tools_.Capabilities()
	if result.IsErr() {
		return cm.ResourceNone
	}
	for _, t := range result.OK().Slice() {
		content = append(content, types.ContentToolSchema(cm.Reinterpret[types.ToolSchema](t)))
	}

	msg := types.Message{Role: 1, Content: cm.ToList(content)}

	agent.context.Push(cm.Reinterpret[agents.Message](msg))

	key := cm.Rep(uintptr(unsafe.Pointer(agent)))
	v := agents.AgentResourceNew(key)
	resourceTable.agents[key] = agent
	return v
}
```

This `constructor` function initializes a new agent with the provided parameters and pushes an initial message to the agent's context. It also stores the agent in a resource table for later use.

The `constructor` function is called when a new agent is created, and it sets up the agent's initial state. Any component that imports the `agent` interface can call this function to create a new agent instance.

### Step 4.3 Implement the Destructor

Next we will implement the `destructor` function, which will be called when the agent is no longer needed.

```go
func destructor(self cm.Rep) {
	delete(resourceTable.agents, self)
}

```

This `destructor` function cleans up the agent's resources when it is no longer needed and removes the agent from the resource table.

### Step 4.4 Implement the Invoke Function

Next, we will implement the `invoke` function, which will be called to process a message with the agent.

```go
func invoke(self cm.Rep, input agents.Message) cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error] {
	agent, ok := resourceTable.agents[self]
	if !ok {
		wasiErr := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
		return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](wasiErr)
	}

	result := agent.context.Push(input)
	if result.IsErr() {
		err := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
		return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](err)
	}

	var messages []agents.Message

	for i := 0; i <= maxturn; i++ {
		result := agent.context.Messages()
		if result.IsErr() {
			err := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
			return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](err)
		}
		msgs := result.OK().Slice()

		encodedResult := agent.format.Encode(cm.ToList(msgs))
		if encodedResult.IsErr() {
			err := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
			return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](err)
		}

		d := tensor.TensorDimensions(cm.ToList([]uint32{1}))
		td := tensor.TensorData(cm.ToList(encodedResult.OK().Slice()))
		t := tensor.NewTensor(d, tensor.TensorTypeU8, td)
		inputs := []inferencestream.NamedTensor{
			{
				F0: "user",
				F1: t,
			},
		}
		computeResult := agent.graph.Compute(cm.ToList(inputs))
		if computeResult.IsErr() {
			err := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
			return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](err)
		}

		stream := computeResult.OK().F1
		ts := tensorStream(stream)
		text := make([]byte, 0)
		for {
		
			p := make([]byte, 100)
			len, err := ts.Read(p)
			if len == 0 || err == io.EOF {
				break
			} else if err != nil {
				err := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
				return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](err)
			}
			text = append(text, p[:len]...)
		}

		decodeResult := agent.format.Decode(cm.ToList(text))
		if decodeResult.IsErr() {
			err := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
			return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](err)
		}

		msg := decodeResult.OK()
		pushResponse := agent.context.Push(*msg)
		if pushResponse.IsErr() {
			err := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
			return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](err)
		}

		// Add the message to the messages list
		messages = append(messages, *msg)

		calledTool := false
		switch msg.Role {
		case types.RoleAssistant:
			for _, c := range msg.Content.Slice() {
				switch c.String() {
				case "tool-input":
					toolresult := agent.tools.Call(*c.ToolInput())
					if toolresult.IsErr() {
						err := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
						return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](err)
					}
					calledTool = true

					toolCall := agents.Message{Role: types.RoleTool, Content: cm.ToList([]types.Content{types.ContentToolOutput(*toolresult.OK())})}

					messages = append(messages, toolCall)

					agent.context.Push(toolCall)
				default:
					continue
				}
			}
		default:
			return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError)))
		}
		if !calledTool {
			break
		}
	}
	return cm.OK[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](cm.ToList(messages))
}
```

While this implementation is a bit complex, it essentially does the following:
- adds the input message to the agent's context
- retrieves all messages from the context
- encodes the messages using the agent's format
- computes the response using the agent's graph
- decodes the response using the agent's format
- looks for tool calls in the response
- if a tool call is found, it calls the tool and adds the tool output to the context
- continues processing until no more tool calls are found or the maximum number of turns is reached
- pushes the response back to the agent's context
- returns the list of messages

### Step 4.5 Implement the InvokeStream Function

Finally, we will implement the `invokeStream` function, which will be called to process a message with the agent in a streaming manner.

Since the `invokeStream` is similar to the `invoke` function, we will keep it as an empty placeholder for now. 

```go
func invokeStream(self cm.Rep, input agents.Message, writer wasi_io.OutputStream) cm.Result[cm.List[agents.Message], agents.Error] {
    // Placeholder for streaming implementation
    return cm.Err[cm.Result[cm.List[agents.Message], agents.Error]](agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError)))
}
```

### Full Implementation

Here is the full implementation of the `main.go` file:

```go
package main

import (
	"encoding/json"
	"fmt"
	"io"
	"unsafe"

	"github.com/hayride-dev/morphs/components/ai/agents/internal/gen/hayride/ai/agents"
	inferencestream "github.com/hayride-dev/morphs/components/ai/agents/internal/gen/hayride/ai/inference-stream"
	"github.com/hayride-dev/morphs/components/ai/agents/internal/gen/hayride/ai/types"
	"github.com/hayride-dev/morphs/components/ai/agents/internal/gen/wasi/nn/tensor"
	"go.bytecodealliance.org/cm"
)

const maxturn = 10

var resourceTable = resources{
	agents: make(map[cm.Rep]*agent),
}

func init() {
	agents.Exports.Agent.Constructor = constructor
	agents.Exports.Agent.Invoke = invoke
	agents.Exports.Agent.InvokeStream = invokeStream
	agents.Exports.Agent.Destructor = destructor
}

type resources struct {
	agents map[cm.Rep]*agent
}

type tensorStream cm.Resource

func (t tensorStream) Read(p []byte) (int, error) {
	ts := cm.Reinterpret[inferencestream.TensorStream](t)
	ts.Subscribe().Block()
	data := ts.Read(uint64(len(p)))
	if data.IsErr() {
		if data.Err().Closed() {
			return 0, nil
		}
		return 0, fmt.Errorf("%s", data.Err().String())
	}
	n := copy(p, data.OK().Slice())
	p = p[:n]
	return len(p), nil
}

type agent struct {
	name    string
	tools   agents.Tools
	context agents.Context
	format  agents.Format
	graph   agents.GraphExecutionContextStream
}

func constructor(name string, instruction string, tools_ agents.Tools, context_ agents.Context, format agents.Format, graph agents.GraphExecutionContextStream) agents.Agent {
	agent := &agent{
		name:    name,
		tools:   tools_,
		context: context_,
		format:  format,
		graph:   graph,
	}

	content := []types.Content{}
	content = append(content, types.ContentText(types.TextContent{
		Text: instruction,
	}))

	result := tools_.Capabilities()
	if result.IsErr() {
		return cm.ResourceNone
	}
	for _, t := range result.OK().Slice() {
		content = append(content, types.ContentToolSchema(cm.Reinterpret[types.ToolSchema](t)))
	}

	msg := types.Message{Role: 1, Content: cm.ToList(content)}

	agent.context.Push(cm.Reinterpret[agents.Message](msg))

	key := cm.Rep(uintptr(unsafe.Pointer(agent)))
	v := agents.AgentResourceNew(key)
	resourceTable.agents[key] = agent
	return v
}

func invoke(self cm.Rep, input agents.Message) cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error] {
	agent, ok := resourceTable.agents[self]
	if !ok {
		wasiErr := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
		return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](wasiErr)
	}

	result := agent.context.Push(input)
	if result.IsErr() {
		err := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
		return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](err)
	}

	var messages []agents.Message

	for i := 0; i <= maxturn; i++ {
		result := agent.context.Messages()
		if result.IsErr() {
			err := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
			return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](err)
		}
		msgs := result.OK().Slice()

		encodedResult := agent.format.Encode(cm.ToList(msgs))
		if encodedResult.IsErr() {
			err := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
			return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](err)
		}

		d := tensor.TensorDimensions(cm.ToList([]uint32{1}))
		td := tensor.TensorData(cm.ToList(encodedResult.OK().Slice()))
		t := tensor.NewTensor(d, tensor.TensorTypeU8, td)
		inputs := []inferencestream.NamedTensor{
			{
				F0: "user",
				F1: t,
			},
		}
		computeResult := agent.graph.Compute(cm.ToList(inputs))
		if computeResult.IsErr() {
			err := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
			return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](err)
		}

		stream := computeResult.OK().F1
		ts := tensorStream(stream)
		text := make([]byte, 0)
		for {
		
			p := make([]byte, 100)
			len, err := ts.Read(p)
			if len == 0 || err == io.EOF {
				break
			} else if err != nil {
				err := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
				return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](err)
			}
			text = append(text, p[:len]...)
		}

		decodeResult := agent.format.Decode(cm.ToList(text))
		if decodeResult.IsErr() {
			err := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
			return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](err)
		}

		msg := decodeResult.OK()
		pushResponse := agent.context.Push(*msg)
		if pushResponse.IsErr() {
			err := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
			return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](err)
		}

		messages = append(messages, *msg)

		calledTool := false
		switch msg.Role {
		case types.RoleAssistant:
			for _, c := range msg.Content.Slice() {
				switch c.String() {
				case "tool-input":
					toolresult := agent.tools.Call(*c.ToolInput())
					if toolresult.IsErr() {
						err := agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError))
						return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](err)
					}
					calledTool = true

					toolCall := agents.Message{Role: types.RoleTool, Content: cm.ToList([]types.Content{types.ContentToolOutput(*toolresult.OK())})}

					messages = append(messages, toolCall)

					agent.context.Push(toolCall)
				default:
					continue
				}
			}
		default:
			return cm.Err[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError)))
		}
		if !calledTool {
			break
		}
	}
	return cm.OK[cm.Result[cm.List[agents.Message], cm.List[agents.Message], agents.Error]](cm.ToList(messages))
}

func invokeStream(self cm.Rep, message agents.Message, writer agents.OutputStream) cm.Result[agents.Error, struct{}, agents.Error] {
        // Placeholder for streaming implementation
    return cm.Err[cm.Result[cm.List[agents.Message], agents.Error]](agents.ErrorResourceNew(cm.Rep(agents.ErrorCodeInvokeError)))
}

func destructor(self cm.Rep) {
	delete(resourceTable.agents, self)
}

func main() {}
```

## Step 5: Build the Morph

To build the Morph, we will use TinyGo to compile the Go code into a WebAssembly component.

```bash
tinygo build -target wasip2 --wit-package ./wit/ --wit-world default default.go
```

This command will compile the Morph to a WebAssembly binary. The `--wit-package` flag specifies the directory containing the WIT files, and the `--wit-world` flag specifies the name of the WIT world to use.

## Step 6: Register the Morph

```bash
hayride register --bin default.wasm --package hayride-ai:default@0.0.1
```

# Next steps    

You can now use the Morph in your Hayride applications. You can create agents that can process messages, call tools, and interact with the context.

The morph can be composed with another Morph that imports the `hayride:ai/agents` interface, allowing you to build more complex agents that can interact with other components in the Hayride ecosystem.