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

The server is part of the core Hayride configuration.

Hayride installs with default values, which can be customized as needed:

```
core:
  server:
    bin: "hayride-core:server-cfg@0.0.1"
    http:
      address: "http://localhost:8080"
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
        [
          "hayride:ai-server@0.0.1",
          "e30739ec-8a38-4d3c-8e0c-b07f3e3e1548"
        ],
        [
          "hayride:cli-agent@0.0.1",
          "e92d2baa-c0e9-43a2-9d47-ac59408693cd"
        ]
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
    "session-status": {
      "active": true
    }
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
    "session-status": {
      "active": false
    }
  }
}
```
