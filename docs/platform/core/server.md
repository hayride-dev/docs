---
sidebar_position: 3
title: Server
---

# Hayride Server

The Hayride server morph is a HTTP server that exposes a REST API for the Hayride platform. It is responsible for handling incoming HTTP requests and routing them to the appropriate handlers. 

The [Hayride CLI](./cli.md) make requests to the server morph to perform various tasks, such as casting morphs, starting and stopping morphs, and managing the Hayride platform.

## WIT Definition

The server morph itself is a WebAssembly Component that can be represented through the following WIT definition:

```wit
world server {
    import wasi:config/store@0.2.0-draft;

    include hayride:wasip2/imports@0.0.51;
    include hayride:silo/imports@0.0.51;

    include hayride:http/client-server@0.0.51;
}
```
The server world is made up of the wasip2 imports, but also includes the `hayride:http/client-server@0.0.51` export, which is wrapper around the wasi-http incoming and outgoing HTTP Handlers. 

The HTTP server implemented is provided by the Hayride runtime. HTTP Request are passed to the server morph, which then routes them to the appropriate handlers.

### Private Imports

Hayride has `private` or `reserved` imports that are not intended for `public` use. These imports are restricted to the Hayride platform and are used internally by the Server.

They include:

- `hayride:silo/imports@0.0.51`: Silo implements basic parallelism and concurrency primitives for the Hayride platform. It provides a way to run Morphs in parallel and manage their execution. This is not intended for public use and is reserved for the Hayride platform. More holistic Async support is being discussed by the WebAssembly community and may replace this in the future.

### Public Exports

The Server includes the following `public` exports from the Hayride platform:

- `hayride:http/client-server@0.0.51`: A hayride specific world that exports the wasi http/incoming-handler as well as a custom config interface to allow server implementations to expose their own server configuration through a `get` function. See below for an example of what this wit definition looks like.

```wit
interface config {
    record server {
        address: string,
        read-timeout: u32,
        write-timeout: u32,
        max-header-bytes: u32,
    }

    enum error-code {
        invalid,
        not-found,
        unknown
    }

    resource error {
        code: func() -> error-code;
        data: func() -> string;
    }

    get: func() -> result<server, error>;
}

world client {
    import wasi:http/types@0.2.0;
    import wasi:http/outgoing-handler@0.2.0;
}

world server {
    import wasi:http/types@0.2.0;
    export wasi:http/incoming-handler@0.2.0;

    export config;
}

world client-server {
    include client; 
    include server; 
}
```

## Server Routes 

The server morph exposes the following routes:

```
	castPath     = "/v1/cast"
	statusPath   = "/v1/sessions/status"
	killPath     = "/v1/sessions/kill"
```

The API objects are defined as WIT records. And can be found in the [Hayride Coven Repository](https://github.com/hayride-dev/core/blob/main/wit/deps/core/api.wit). 

### Cast
The cast route is used to execute a function on a morph. The function is typically the entry point of the morph, and the request body contains the input parameters for the function. The response body contains the output of the function. The response body also contains a session ID, which can be used to track the status of the function execution.

The cast route is used by the Hayride CLI to execute functions on morphs. The request body contains the input parameters for the function, and the response body contains the output of the function.

#### WIT Definition
```
record cast {
    name: string,
    function: string,
    args: list<string>,
}
```

The cast record is used as the data object in the request body. 

- **name field**, is the name of the morph, 
- **function field**, is the name of the function to execute
- **args** field, is a list of input parameters for the function. Represented as strings. More complex types are not supported yet.

#### Example Request
```json
{
  "data": {
    "cast": {
      "name": "string",
      "function": "",
      "args": [
        ""
      ]
    }
  }
}
```

#### Example Response
```json
{
  "data": {
    "session-id": "88290a7c-60e6-419a-bfc3-92f5a4f90fe0"
  }
}
```

### Session Status

The session status route is used to check if a morph is running. The request body contains the session ID, and the response body contains the status of the morph. 

#### WIT Definition
```   
record session-status {
    active: bool,
}
```

- **active field**, is a boolean value that indicates if the morph is running or not.

#### Example Request
```json
{
  "data": {
    "session-id": "88290a7c-60e6-419a-bfc3-92f5a4f90fe0"
  }
}
```
#### Example Response
```json
{
  "data": {
    "session-status": {
      "active": true
    }
  }
}
```

### Kill Session
The kill session route is used to stop a running morph. The request body contains the session ID, and the response body contains the status of the morph after it has been stopped.

#### WIT Definition
```
record session-status {
    active: bool,
}
```

- **active field**, is a boolean value that indicates if the morph is running or not.

#### Example Request
```json
{
  "data": {
    "session-id": "88290a7c-60e6-419a-bfc3-92f5a4f90fe0"
  }
}
```
#### Example Response
```json
{
  "data": {
    "session-status": {
      "active": false
    }
  }
}
```