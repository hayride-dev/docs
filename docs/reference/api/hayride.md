---
sidebar_position: 1
title: Hayride
---

## Hayride API

The **Hayride Server** provides a dedicated API for interaction with the Hayride runtime. 
Most interactions with the Hayride Server can be facilitated through CLI commands, but the API does enable direct interaction if needed.

The server will be started when hayride is initialized:

```
hayride init
```

Once initialized, the server exposes HTTP endpoints that allow interaction with Hayride runtime using structured message payloads.

:::warning
The **Hayride API** is currently in **alpha**. Breaking changes may occur in upcoming releases. Please consult release notes and this documentation when integrating with this API.
:::

### Configuration

The server is part of the core Hayride configuration. The config specifies the root server binary, components used to compose the full server, and the http port.

Hayride installs with default values, which can be customized as needed:

```
core:
    server:
        bin: "hayride-core:server@0.0.1"
        plugs:
            - hayride-core:cfg@0.0.1
        http:
            port: 8080
```

:::tip
For detailed configuration options, see the [Configuration Guide](../../developer-guides/advanced/configuration.md) .
:::

### API Endpoints

> **Post /v1/cast**

Submits a request for the Hayride runtime to execute a registered morph. Can include option function name and arguments to use with the run.

**Request**

**Content-Type**: `application/json`

**Body**:
```JSON
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

**Response**

If successful, the server will respond with a session-id assigned for the running session.

```JSON
{
  "data": {
    "session-id": "31d55d91-122a-40fb-8f09-fcccff7ce065"
  }
}
```

> **GET /v1/sessions**

Submits a request to get all running sessions on the Hayride Server.

**Response**

If successful, the server will respond a list of sessions which includes the package name and session id.

```JSON
{
  "data": {
    "sessions": {
      "sessions": [
        {
          "id": "3d534634-b7de-4fe9-bebc-f19a18341aff",
          "pkg": "hayride-core:composed-ai-server@0.0.1",
          "function": "",
          "args": [
            "hayride-core:composed-ai-server@0.0.1"
          ],
          "output": [],
          "status": "processing"
        },
      ]
    }
  }
}
```

> **Post /v1/sessions/status**

Submits a request to get the current status of a running session.

**Request**

**Content-Type**: `application/json`

**Body**:
```JSON
{
  "data": {
    "session-id": "31d55d91-122a-40fb-8f09-fcccff7ce065"
  }
}
```

**Response**

If successful, the server will respond with the status of the session.

```JSON
{
  "data": {
    "session-status": "processing"
  }
}
```

> **Post /v1/sessions/stop**

Submits a request for the Hayride runtime to stop a running session.

**Request**

**Content-Type**: `application/json`

**Body**:
```JSON
{
  "data": {
    "session-id": "31d55d91-122a-40fb-8f09-fcccff7ce065"
  }
}
```

**Response**

If successful, the server will respond with the updated session status.

```JSON
{
  "data": {
    "session-status": "killed"
  }
}
```

> **GET /v1/version**

Submits a request to get the current configured version of Hayride.

**Response**

If successful, the server will respond with a version string.

```JSON
{
  "data": {
    "version": "v0.0.3-alpha"
  }
}
```

> **GET /v1/version/latest**

Submits a request to get the latest released version of Hayride.

**Response**

If successful, the server will respond with a version string.

```JSON
{
  "data": {
    "version": "v0.0.3-alpha"
  }
}
```
