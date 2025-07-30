---
sidebar_position: 3
title: Model Formatting
---

Hayride defines a set of WebAssembly Interfaces Types (WIT) that allow you to build components that export the `model` interface.

A component that exports the `model` interface defines how messages that are are encoded into data to send to an LLM and how the data returning from an LLM can be decoded. This is important as it provides a way to perform model specific encoding of messages and a way to detect the various model specific tool calling features.

In this example we will show how you can create a Llama 3.1 model implementation.

:::tip
Hayride leverages the WebAssembly Component Model, which allows you to implement your own `model` component. However, Hayride ships with a various implementations of the `model` interface that you can use to get started quickly. 

This example is specifically to help guide in the process of creating your own model implementations.
:::


## Prerequisites
- [Go](https://go.dev/doc/install) version 1.23.0+
- [TinyGo](https://tinygo.org/) version 0.33.0+

## Step 1: WIT Definition

Below is the WIT definition for the `model` interface. 

```wit
package hayride:ai@0.0.61;

interface model {
    use types.{message};

    enum error-code {
        context-error,
        context-encode,
        context-decode,
        compute-error,
        unknown
    }

    resource error {
        /// return the error code.
        code: func() -> error-code;
        /// errors can propagated with backend specific status through a string value.
        data: func() -> string;
    }

    resource format {
        constructor();
        encode: func(messages: list<message>) -> result<list<u8>,error>;
        decode: func(raw: list<u8>) -> result<message, error>;
    }
}
```

The `format` resource is defined by components that implement the `model` interface. 

The error resource provides a way to return an error code and additional data if needed.

The constructor creates a new format with a `encode` method to convert a message to bytes, and a `decode` method to turn bytes into a message.

How you implement the `format` resource is up to you, but it should provide a way to encode and decode messages.

Using the `model` interface, you can define a component that exports the model interface like the following:

```wit
package hayride:models@0.0.1;

world llm {
    include hayride:wasip2/imports@0.0.61;
    export hayride:ai/model@0.0.61;
}
```

Place this WIT definition in a file called `world.wit` in a directory called `wit` at the root of your project.

## Step 2: Create a WIT Dependencies file

Since the world imports wasip2 and the context interface, we need to create a `deps.toml` file to manage the dependencies for our WIT files.

In the `wit` directory, create a `deps.toml` file with the following content:

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

## Step 4: Implement the Model

Now that we have the WIT definition and the bindings dependency, we can implement the model in Go.

Next, create a file called `main.go` in the root directory of your project. This file will contain the implementation of your Morph:

```bash
go mod init model-example
touch main.go
```

### Step 4.1 Exports 

In the main.go file, implement the Morph:

We will start by setting the exported functions for the `context` resource by calling the bindings `export.Context` function with a constructor.


```go
package main

import (
    "github.com/hayride-dev/bindings/go/hayride/ai/models"
    "github.com/hayride-dev/bindings/go/hayride/ai/models/export"
    "github.com/hayride-dev/bindings/go/hayride/types"
)

func init() {
    export.Format(constructor)
}
```

Note, the `export.Format` function takes a constructor, which is a function type defined in bindings: `type Constructor func() (models.Format, error)`

This constructor will be called whenever the user of our Model calls the format constructor function. So we will return a new object that satisfies the 
`models.Format` interface:

```go
type Format interface {
	Encode(messages ...types.Message) ([]byte, error)
	Decode(b []byte) (*types.Message, error)
}
```

### Step 4.2 Implement the Constructor

The constructor is called when the format is created. It initializes the format and returns a struct that satisfies the `models.Format` interface.

```go
var _ models.Format = (*llama3)(nil)

type llama3 struct{}

func constructor() (models.Format, error) {
	return &llama3{}, nil
}
```

The `constructor` function creates a new instance of the `llama3` struct, which for this example is just an empty struct.

### Step 4.3 Implement the Encode and Decode Methods

We can see above that our `llama3` struct needs to satisfy the `models.Format` interface which has an `Encode` method that encodes a message into bytes and a `Decode` method that decodes bytes into a message.

This is where custom model logic will be required to handle various model formats. Below is an example of how [Hayride Morphs](https://github.com/hayride-dev/morphs) implemented these functions for llama 3.1:

```go
func (m *llama3) Decode(data []byte) (*types.Message, error) {
	msg := string(data)
	if strings.Contains(msg, pythonTag) {
		// remove python tags
		content := strings.TrimPrefix(msg, pythonTag)
		content = strings.TrimSuffix(content, endOfMessage)

		matches := parseFunc.FindStringSubmatch(strings.TrimSpace(content))
		if len(matches) != 3 {
			return nil, fmt.Errorf("failed to parse assistant message, invalid function formation")
		}

		name := matches[0]
		argsString := matches[1]

		paramsList := parseFuncParams.FindAllString(argsString, -1)
		var args [][2]string
		for _, param := range paramsList {
			// Split each parameter into name and value
			paramParts := strings.SplitN(param, "=", 2)
			if len(paramParts) != 2 {
				return nil, fmt.Errorf("parameter format is invalid: %s", param)
			}

			// Trim spaces and quotes from key
			paramKey := strings.TrimSpace(paramParts[0])
			paramKey = strings.Trim(paramKey, "'")
			paramKey = strings.Trim(paramKey, "\"")

			// Trim spaces and quotes from value
			paramValue := strings.TrimSpace(paramParts[1])
			paramValue = strings.Trim(paramValue, "'")
			paramValue = strings.Trim(paramValue, "\"")

			args = append(args, [2]string{paramKey, paramValue})
		}
		return &types.Message{
			Role: types.RoleAssistant,
			Content: cm.ToList([]types.MessageContent{
				types.NewMessageContent(types.CallToolParams{
					Name:      name,
					Arguments: cm.ToList(args),
				}),
			}),
		}, nil
	} else if strings.Contains(msg, "<function") {
		// Custom function definition
		matches := customFunc.FindStringSubmatch(msg)
		if len(matches) != 3 {
			return nil, fmt.Errorf("failed to parse assistant message, invalid function formation")
		}

		if matches != nil {
			result := make(map[string]string)
			for i, name := range customFunc.SubexpNames() {
				if i > 0 && name != "" {
					result[name] = matches[i]
				}
			}

			input := [][2]string{}
			if values, ok := result["input"]; ok && values != "" {
				// Unmarshal into a map
				var m map[string]string
				err := json.Unmarshal([]byte(values), &m)
				if err != nil {
					return nil, fmt.Errorf("failed to parse input parameters: %v", err)
				}

				for k, v := range m {
					input = append(input, [2]string{k, v})
				}
			}

			return &types.Message{
				Role: types.RoleAssistant,
				Content: cm.ToList([]types.MessageContent{
					types.NewMessageContent(types.CallToolParams{
						Name:      result["name"],
						Arguments: cm.ToList(input),
					}),
				}),
			}, nil
		} else {
			return nil, fmt.Errorf("failed to parse assistant message, invalid function formation")
		}
	}

	message := &types.Message{
		Role: types.RoleAssistant,
		Content: cm.ToList([]types.MessageContent{
			types.NewMessageContent(types.Text(msg)),
		}),
	}

	return message, nil
}

func (m *llama3) Encode(messages ...types.Message) ([]byte, error) {
	builder := &strings.Builder{}
	last := len(messages) - 1
	for i, msg := range messages {
		//builder.WriteString("<|begin_of_text|>")
		switch msg.Role {
		case types.RoleSystem:
			// set the system message
			builder.WriteString(fmt.Sprintf("%s%s%s\n", startHeaderId, system, endHeaderId))
			// add environment token to enable tools by default
			// TODO :: add ability to disable tool support in system prompt
			builder.WriteString(fmt.Sprintf("%s\n", env))

			// message body, collect tool schema definitions
			tools := []types.Tool{}
			for _, content := range msg.Content.Slice() {
				switch content.String() {
				case "text":
					c := content.Text()
					builder.WriteString(fmt.Sprintf("%s\n", *c))
				case "tools":
					c := content.Tools().Slice()
					tools = c
				}
			}

			if len(tools) > 0 {
				toolString := customToolEncode(tools)
				if toolString != "" {
					builder.WriteString(fmt.Sprintf("%s\n", toolString))
				}
			}

			// end system message turn
			builder.WriteString(endOfTurn)
		case types.RoleUser:
			// header
			builder.WriteString(fmt.Sprintf("%s%s%s\n", startHeaderId, user, endHeaderId))
			// message body ( user prompt )
			for _, content := range msg.Content.Slice() {
				if content.String() == "text" {
					c := content.Text()
					builder.WriteString(fmt.Sprintf("%s\n", *c))
				}
			}
			// end turn
			builder.WriteString(endOfTurn)
		case types.RoleAssistant:
			// header
			builder.WriteString(fmt.Sprintf("%s%s%s\n", startHeaderId, assistant, endHeaderId))
			// message body ( assistant response )
			for _, content := range msg.Content.Slice() {
				switch content.String() {
				case "text":
					c := content.Text()
					builder.WriteString(fmt.Sprintf("%s\n", *c))
					builder.WriteString(endOfTurn)
				case "tool-input":
					c := content.ToolInput()
					// Parse arguments into a json format
					args := make(map[string]string)
					for _, arg := range c.Arguments.Slice() {
						if len(arg) != 2 {
							return nil, fmt.Errorf("invalid tool input argument format: %v", arg)
						}
						args[arg[0]] = arg[1]
					}

					input := ""
					if len(args) > 0 {
						jsonArgs, err := json.Marshal(args)
						if err != nil {
							return nil, fmt.Errorf("failed to marshal tool input arguments: %v", err)
						}
						input = string(jsonArgs)
					}

					builder.WriteString(fmt.Sprintf("<function=%s>%v</function>\n", c.Name, input))
					// end turn
					builder.WriteString(endOfMessage)
				}
			}
		case types.RoleTool:
			// header
			builder.WriteString(fmt.Sprintf("%s%s%s\n", startHeaderId, tool, endHeaderId))
			// message body ( tool output )
			for _, content := range msg.Content.Slice() {
				if content.String() == "tool-output" {
					output := content.ToolOutput()
					for _, c := range output.Content.Slice() {
						switch c.String() {
						case "text":
							builder.WriteString(fmt.Sprintf("%s\n", c.Text().Text))
						case "image":
							image := c.Image()
							builder.WriteString(fmt.Sprintf("Image Data: %v\n", image.Data))
						case "audio":
							audio := c.Audio()
							builder.WriteString(fmt.Sprintf("Audio Data: %v\n", audio.Data))
						case "resource-link":
							resource := c.ResourceLink()
							builder.WriteString(fmt.Sprintf("Resource Link: %s\n", resource.URI))
						case "resource-content":
							content := c.ResourceContent()
							switch content.ResourceContents.String() {
							case "text":
								builder.WriteString(fmt.Sprintf("Resource Content (Text): %s\n", content.ResourceContents.Text()))
							case "blob":
								builder.WriteString(fmt.Sprintf("Resource Content (Blob): %v\n", content.ResourceContents.Blob()))
							}
						}
					}
				}
			}
			// end turn
			builder.WriteString(endOfTurn)
		default:
			return nil, fmt.Errorf("unknown supported message role: %v", msg.Role)
		}
		if i == last {
			if msg.Role != types.RoleAssistant {
				builder.WriteString(fmt.Sprintf("%s%s%s\n", startHeaderId, assistant, endHeaderId))
			}
		}
	}
	return []byte(builder.String()), nil
}
```

## Full Implementation

Here is the full implementation of the `main.go` file:

```go
package llama3

import (
	"encoding/json"
	"fmt"
	"regexp"
	"strings"

	"github.com/hayride-dev/bindings/go/hayride/ai/models"
    "github.com/hayride-dev/bindings/go/hayride/ai/models/export"
	"github.com/hayride-dev/bindings/go/hayride/types"
	"go.bytecodealliance.org/cm"
)

var (
	// Define a regular expression to match the function name and the parameter list that may appear anywhere in the input string
	// Example: [func_name1(params_name1=params_value1, params_name2=params_value2)]
	parseFunc = regexp.MustCompile(`\[([a-zA-Z_][a-zA-Z0-9_@:.\-\/ ]*)\((.*)\)\]`)

	// The function definition for a custom defined function for llama 3.1
	// Example: <function=example_function_name>{"example_name": "example_value"}</function>
	customFunc = regexp.MustCompile(`<function=(?P<name>[^\s>]+)>\s*(?P<input>\{.*?\})?\s*<.*function>`)

	// Split the parameters by comma
	parseFuncParams = regexp.MustCompile(`[a-zA-Z_][a-zA-Z0-9_]*=('[^']*'|"[^"]*"|[^,]+)`)
)

const (
	// llama3.1 special tokens

	// specifies the start of the prompt
	beginOfText = "<|begin_of_text|>"

	// model will cease to generate more tokens. This token is generated only by the base models.
	endOfText = "<|end_of_text|>"

	// this token is used for padding text sequences to the same length in a batch
	finetuneRightPadID = "<|finetune_right_pad_id|>"

	// these token is used to enclose the role for a particular message.
	// the possible roles are: [system, user, assistant, ipython]
	startHeaderId = "<|start_header_id|>"
	endHeaderId   = "<|end_header_id|>"

	// end of message. A message represents a possible stopping point for execution where the
	// can inform the executor that a tool call needs to be made. This is used for multi-step interactions
	// between the model and any avilable tools. This token is emitted by the model when the environment: ipython
	// instruction is used in the system prompt, or if the model calls for a built-in tool.
	endOfMessage = "<|eom_id|>"

	// end of turn.  Represents when the model has determined that it has finished interacting with
	// the user message that initiated its response. This is used in two scenarios:
	//
	// at the end of a direct interaction between the model and the user
	// at the end of multiple interactions between the model and any available tools
	//
	// this token signals to the executor that the model has finished generating a response.
	endOfTurn = "<|eot_id|>"

	// special tag used in the model’s response to signify a tool call.
	pythonTag = "<|python_tag|>"

	//There are 4 different roles that are supported by Llama text models:
	//
	// system: Sets the context in which to interact with the AI model. It typically includes rules, guidelines, or necessary information that help the model respond effectively.
	// user: Represents the human interacting with the model. It includes the inputs, commands, and questions to the model.
	// ipython: A new role introduced in Llama 3.1. Semantically, this role means "tool". This role is used to mark messages with the output of a tool call when sent back to the model from the executor.
	// assistant: Represents the response generated by the AI model based on the context provided in the system, ipython and user prompts.
	system    = "system"
	user      = "user"
	tool      = "ipython"
	assistant = "assistant"

	// environment token
	env = "Environment: ipython"
)

var _ models.Format = (*llama3)(nil)

func constructor() (models.Format, error) {
	return &llama3{}, nil
}

type llama3 struct{}

func init() {
    export.Format(constructor)
}

func (m *llama3) Decode(data []byte) (*types.Message, error) {
	msg := string(data)
	if strings.Contains(msg, pythonTag) {
		// remove python tags
		content := strings.TrimPrefix(msg, pythonTag)
		content = strings.TrimSuffix(content, endOfMessage)

		matches := parseFunc.FindStringSubmatch(strings.TrimSpace(content))
		if len(matches) != 3 {
			return nil, fmt.Errorf("failed to parse assistant message, invalid function formation")
		}

		name := matches[0]
		argsString := matches[1]

		paramsList := parseFuncParams.FindAllString(argsString, -1)
		var args [][2]string
		for _, param := range paramsList {
			// Split each parameter into name and value
			paramParts := strings.SplitN(param, "=", 2)
			if len(paramParts) != 2 {
				return nil, fmt.Errorf("parameter format is invalid: %s", param)
			}

			// Trim spaces and quotes from key
			paramKey := strings.TrimSpace(paramParts[0])
			paramKey = strings.Trim(paramKey, "'")
			paramKey = strings.Trim(paramKey, "\"")

			// Trim spaces and quotes from value
			paramValue := strings.TrimSpace(paramParts[1])
			paramValue = strings.Trim(paramValue, "'")
			paramValue = strings.Trim(paramValue, "\"")

			args = append(args, [2]string{paramKey, paramValue})
		}
		return &types.Message{
			Role: types.RoleAssistant,
			Content: cm.ToList([]types.MessageContent{
				types.NewMessageContent(types.CallToolParams{
					Name:      name,
					Arguments: cm.ToList(args),
				}),
			}),
		}, nil
	} else if strings.Contains(msg, "<function") {
		// Custom function definition
		matches := customFunc.FindStringSubmatch(msg)
		if len(matches) != 3 {
			return nil, fmt.Errorf("failed to parse assistant message, invalid function formation")
		}

		if matches != nil {
			result := make(map[string]string)
			for i, name := range customFunc.SubexpNames() {
				if i > 0 && name != "" {
					result[name] = matches[i]
				}
			}

			input := [][2]string{}
			if values, ok := result["input"]; ok && values != "" {
				// Unmarshal into a map
				var m map[string]string
				err := json.Unmarshal([]byte(values), &m)
				if err != nil {
					return nil, fmt.Errorf("failed to parse input parameters: %v", err)
				}

				for k, v := range m {
					input = append(input, [2]string{k, v})
				}
			}

			return &types.Message{
				Role: types.RoleAssistant,
				Content: cm.ToList([]types.MessageContent{
					types.NewMessageContent(types.CallToolParams{
						Name:      result["name"],
						Arguments: cm.ToList(input),
					}),
				}),
			}, nil
		} else {
			return nil, fmt.Errorf("failed to parse assistant message, invalid function formation")
		}
	}

	message := &types.Message{
		Role: types.RoleAssistant,
		Content: cm.ToList([]types.MessageContent{
			types.NewMessageContent(types.Text(msg)),
		}),
	}

	return message, nil
}

func (m *llama3) Encode(messages ...types.Message) ([]byte, error) {
	builder := &strings.Builder{}
	last := len(messages) - 1
	for i, msg := range messages {
		//builder.WriteString("<|begin_of_text|>")
		switch msg.Role {
		case types.RoleSystem:
			// set the system message
			builder.WriteString(fmt.Sprintf("%s%s%s\n", startHeaderId, system, endHeaderId))
			// add environment token to enable tools by default
			// TODO :: add ability to disable tool support in system prompt
			builder.WriteString(fmt.Sprintf("%s\n", env))

			// message body, collect tool schema definitions
			tools := []types.Tool{}
			for _, content := range msg.Content.Slice() {
				switch content.String() {
				case "text":
					c := content.Text()
					builder.WriteString(fmt.Sprintf("%s\n", *c))
				case "tools":
					c := content.Tools().Slice()
					tools = c
				}
			}

			if len(tools) > 0 {
				toolString := customToolEncode(tools)
				if toolString != "" {
					builder.WriteString(fmt.Sprintf("%s\n", toolString))
				}
			}

			// end system message turn
			builder.WriteString(endOfTurn)
		case types.RoleUser:
			// header
			builder.WriteString(fmt.Sprintf("%s%s%s\n", startHeaderId, user, endHeaderId))
			// message body ( user prompt )
			for _, content := range msg.Content.Slice() {
				if content.String() == "text" {
					c := content.Text()
					builder.WriteString(fmt.Sprintf("%s\n", *c))
				}
			}
			// end turn
			builder.WriteString(endOfTurn)
		case types.RoleAssistant:
			// header
			builder.WriteString(fmt.Sprintf("%s%s%s\n", startHeaderId, assistant, endHeaderId))
			// message body ( assistant response )
			for _, content := range msg.Content.Slice() {
				switch content.String() {
				case "text":
					c := content.Text()
					builder.WriteString(fmt.Sprintf("%s\n", *c))
					builder.WriteString(endOfTurn)
				case "tool-input":
					c := content.ToolInput()
					// Parse arguments into a json format
					args := make(map[string]string)
					for _, arg := range c.Arguments.Slice() {
						if len(arg) != 2 {
							return nil, fmt.Errorf("invalid tool input argument format: %v", arg)
						}
						args[arg[0]] = arg[1]
					}

					input := ""
					if len(args) > 0 {
						jsonArgs, err := json.Marshal(args)
						if err != nil {
							return nil, fmt.Errorf("failed to marshal tool input arguments: %v", err)
						}
						input = string(jsonArgs)
					}

					builder.WriteString(fmt.Sprintf("<function=%s>%v</function>\n", c.Name, input))
					// end turn
					builder.WriteString(endOfMessage)
				}
			}
		case types.RoleTool:
			// header
			builder.WriteString(fmt.Sprintf("%s%s%s\n", startHeaderId, tool, endHeaderId))
			// message body ( tool output )
			for _, content := range msg.Content.Slice() {
				if content.String() == "tool-output" {
					output := content.ToolOutput()
					for _, c := range output.Content.Slice() {
						switch c.String() {
						case "text":
							builder.WriteString(fmt.Sprintf("%s\n", c.Text().Text))
						case "image":
							image := c.Image()
							builder.WriteString(fmt.Sprintf("Image Data: %v\n", image.Data))
						case "audio":
							audio := c.Audio()
							builder.WriteString(fmt.Sprintf("Audio Data: %v\n", audio.Data))
						case "resource-link":
							resource := c.ResourceLink()
							builder.WriteString(fmt.Sprintf("Resource Link: %s\n", resource.URI))
						case "resource-content":
							content := c.ResourceContent()
							switch content.ResourceContents.String() {
							case "text":
								builder.WriteString(fmt.Sprintf("Resource Content (Text): %s\n", content.ResourceContents.Text()))
							case "blob":
								builder.WriteString(fmt.Sprintf("Resource Content (Blob): %v\n", content.ResourceContents.Blob()))
							}
						}
					}
				}
			}
			// end turn
			builder.WriteString(endOfTurn)
		default:
			return nil, fmt.Errorf("unknown supported message role: %v", msg.Role)
		}
		if i == last {
			if msg.Role != types.RoleAssistant {
				builder.WriteString(fmt.Sprintf("%s%s%s\n", startHeaderId, assistant, endHeaderId))
			}
		}
	}
	return []byte(builder.String()), nil
}

func customToolEncode(tools []types.Tool) string {
	if len(tools) == 0 {
		return ""
	}

	result := `
	# Tool Instructions
	- Calling a tool is not necessary, use relevant functions only if needed
	- If you call a function, put the entire function call reply on one line
	- Only add parameters when the params are specified in the tool schema
	- When you get a response from a tool, use that information to answer the user query

	You have access to the following functions:
	{
	`
	for _, tool := range tools {
		result += fmt.Sprintf(`"name": "%s",\n`, tool.Name)
		result += fmt.Sprintf(`"description": "%s",\n`, tool.Description)
		// Add input schema properties
		if len(tool.InputSchema.Properties.Slice()) > 0 {
			result += `"parameters": {`
			for i, prop := range tool.InputSchema.Properties.Slice() {
				result += fmt.Sprintf(`"%s":`, prop[0])
				result += fmt.Sprintf(`"%s"`, prop[1])
				if i < len(tool.InputSchema.Properties.Slice())-1 {
					result += ", "
				}
			}
			result += "},\n"
		} else {
			result += `"parameters": {},\n`
		}
	}

	// remove last comma
	result = strings.TrimSuffix(result, ",\n")

	result += `
	}
		
	If a you choose to call a function ONLY reply in the following format:
	<{start_tag}={function_name}>{parameters}{end_tag}
	where

	start_tag => <function
	parameters => a JSON dict with the function argument name as key and function argument value as value.
	end_tag => </function>

	Here is an example,
	<function=example_function_name>{"example_name": "example_value"}</function>

	Reminder:
	- Function calls MUST follow the specified format
	- Required parameters MUST be specified
	- Only call one function at a time
	- Put the entire function call reply on one line
	- Always add your sources when using search results to answer the user query
	`
	return result
}
```

## Step 5: Build the Morph

To build the Morph, we will use TinyGo to compile the Go code into a WebAssembly component.

```bash
tinygo build -target wasip2 --wit-package ./wit/ --wit-world llm -o llama31.wasm main.go
```

This command will compile the Morph to a WebAssembly binary. The `--wit-package` flag specifies the directory containing the WIT files, and the `--wit-world` flag specifies the name of the WIT world to use.

## Step 6: Register the Morph

```bash
hayride register --bin llama31.wasm --package hayride-ai:llama31@0.0.1
```

# Next steps    

You can now use the Morph in your Hayride applications. 

The morph can be composed with another Morph that imports the `hayride:ai/model` interface, allowing you to build more complex agents that can interact with other components in the Hayride ecosystem.

For more examples, checkout the following examples: 
- Agents: [agents](./agents.md)
- Tools: [tools](./tools.md)
- Context: [context](./context.md)
