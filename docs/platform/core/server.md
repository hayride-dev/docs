---
sidebar_position: 3
title: Server
---

# Hayride Server

The Hayride server moprh is a HTTP server that exposes a REST API for the Hayride platform. It is responsible for handling incoming HTTP requests and routing them to the appropriate handlers. 

The [Hayride CLI](./cli.md) make requests to the server moprh to perform various tasks, such as casting morphs, starting and stopping morphs, and managing the Hayride platform.

## WIT Definition

The server moprh itself is a WebAssembly Component that can be represented through the following WIT definition:

```wit
world server {
    import wasi:config/store@0.2.0-draft;

    include hayride:wasip2/imports@0.0.48;
    include hayride:silo/imports@0.0.48;

    include hayride:http/client-server@0.0.48;
}
```
The server world is made up of the wasip2 imports, but also includes the `hayride:http/client-server@0.0.46` export, which is wrapper around the wasi-http incoming and outgoing HTTP Handlers. 

The HTTP server implemented is provided by the Hayride runtime. HTTP Request are passed to the server moprh, which then routes them to the appropriate handlers. 

### Public Imports
- `hayride:http/client-server@0.0.46`
- `hayride:core/config@0.0.46`

### Private Imports
- `hayride:silo/imports@0.0.46`

## Server Routes 

The server moprh exposes the following routes:

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