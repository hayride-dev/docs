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
    include hayride:wasip2/imports@0.0.65;
    // core imports
    import hayride:core/version@0.0.65;
    
    // silo imports
    import hayride:silo/threads@0.0.65;
    import hayride:silo/process@0.0.65;

    // wasi imports
    import wasi:config/store@0.2.0-draft;
    
    // exports
    export wasi:http/incoming-handler@0.2.0;
    export hayride:http/config@0.0.65;
}
```
The server world is made up of the wasip2 imports and the `wasi:http/incoming-handler@0.2.0` export for http, but also includes the `hayride:http/config@0.0.65` export, which exposes a config `get` to enable the Hayride to get server configuration such as the address to serve.

The HTTP server implemented is provided by the Hayride runtime. HTTP requests are passed to the server morph, which then routes them to the appropriate handlers.

### Private Imports

Hayride has `private` or `reserved` imports that are not intended for `public` use. These imports are restricted to the Hayride platform and are used internally by the Server.

They include:

- `hayride:silo/threads@0.0.65` and `hayride:silo/process@0.0.65`: Silo implements basic parallelism and concurrency primitives for the Hayride platform. It provides a way to run Morphs in parallel and manage their execution. This is not intended for public use and is reserved for the Hayride platform. More holistic Async support is being discussed by the WebAssembly community and may replace this in the future.

### Public Imports

The Server includes the following `public` imports:
- `hayride:core/version@0.0.65`: A Hayride core interface which provides a function to check the latest version of Hayride releases.
- `wasi:config/store@0.2.0-draft`: The WASI config store which provides an interface to include a component that exports a config store, which is implemented the Hayride cfg morph to provide access to the Hayride config.

### Public Exports

The Server includes the following `public` exports:

- `wasi:http/incoming-handler@0.2.0`: The WASI http incoming-handler which provides an entry point for handling http requests.
- `hayride:http/config@0.0.65`: A hayride specific config interface to allow server implementations to expose their own server configuration through a `get` function. See below for an example of what this wit definition looks like.

```wit
interface types {
    enum error-code {
        invalid,
        not-found,
        unknown
    }

    record server-config {
        address: string,
        read-timeout: u32,
        write-timeout: u32,
        max-header-bytes: u32,
    }
}

interface config {
    use types.{server-config, error-code};
    resource error {
        code: func() -> error-code;
        data: func() -> string;
    }
    get: func() -> result<server-config, error>;
}
```

## Server Routes 

The server morph exposes the following routes:

```
	healthPath   = "/health"
	castPath     = "/v1/cast"
	sessionsPath = "/v1/sessions"
	statusPath   = "/v1/sessions/status"
	stopPath     = "/v1/sessions/stop"
	publishPath  = "/v1/registry/publish"
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

- **name** field, is the name of the morph, 
- **function** field, is the name of the function to execute
- **args** field, is a list of input parameters for the function. Represented as strings. More complex types are not supported yet.

#### Example Request

```json
{
  "data": {
    "cast": {
      "name": "example:echo-morph@0.0.1",
      "function": "echo",
      "args": [
        "arg1"
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
variant response-data {
    session-status(thread-status),
}

enum thread-status {
    unknown,
    processing,
    exited,
    killed
}
```

- **session-status** will contain the current thread status for the session, either processing, exited, or killed.

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
    "session-status": "processing"
  }
}
```

### Stop Session

The stop session route is used to stop a running morph. The request body contains the session ID, and the response body contains the status of the morph after it has been stopped.

#### WIT Definition

```
variant response-data {
    session-status(thread-status),
}

enum thread-status {
    unknown,
    processing,
    exited,
    killed
}
```

- **session-status** will contain the current thread status for the session, either processing, exited, or killed.

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
    "session-status": "killed"
  }
}
```
