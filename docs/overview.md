---
sidebar_position: 1
title: Overview
slug: /
description: Hayride is an open source WebAssembly-based platform for building secure, polyglot, and composable AI workflows and agentic applications.
---

# Overview

Hayride is an open source platform for building secure, portable, and composable polyglot applications using WebAssembly. It offers a foundation for executing workloads across languages and environments with strong isolation, modularity, and scalability.

<div style={{ display: 'flex', justifyContent: 'center' }}>
  <div style={{ position: 'relative', display: 'inline-block' }}>
    <img
      src={require('/img/hayride-overview.png').default}
      width="350"
      height="400"
      style={{ display: 'block' }}
    />
    <img
      src={require('/img/chrip-skate.png').default}
      width="60"
      height="60"
      style={{
        position: 'absolute',
        bottom: '15px',
        right: '5px',
        pointerEvents: 'none',
      }}
    />
    <p></p>
  </div>
</div>

Hayride helps developers build scalable applications while maintaining strict boundaries around execution, resource usage, and language interoperability.

Whether you're composing autonomous AI agents, introducing sandboxed execution into existing systems, or creating modular applications, Hayride accelerates innovation through simplicity and security.

:::info
Hayride’s polyglot support is made possible by [WebAssembly](https://webassembly.org/) and [WASI](https://github.com/WebAssembly/WASI/blob/main/Proposals.md), enabling safe execution across a range of supported languages.

Learn more in the [WebAssembly documentation](./platform/wasm.md).
:::

## Key Features for Developers

Hayride offers a practical foundation for building modern applications with modular execution and cross-platform consistency.

| Feature                         | Description                                                                                                                             |
|----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Deployment Easy                 | Deploy locally or remotely.                                                                                                             |
| Resource Management             | Automatic caching, parallel execution, and memory management for optimized performance.                                                |
| Performance & Portability       | Built on WebAssembly (Wasm) for lightweight, efficient execution across environments.                                                  |
| Modular & Flexible Architecture | Built on morphological principles to maximize modularity, reusability, and maintainability.                                            |
| Low Cost to Build & Deploy      | Infrastructure designed to reduce complexity and cost in both development and deployment.                                              |
| Separation of Concerns          | Enables innovation without the burden of system constraints, with support for multiple languages via WebAssembly.                     |
| Developer-First Ecosystem       | Extensible and secure platform with powerful tooling and integration support.                                                          |
| Built on Trusted Standards      | Built on the open WebAssembly ecosystem to ensure interoperability, transparency, and community-driven evolution.                     |

:::tip
Hayride follows the principles of [**Morphological Architecture**](./platform/concepts/design-patterns.md), emphasizing modularity, composability, and secure execution.

This design supports extension for varied use cases—from edge computing to AI-driven workflows—without sacrificing clarity or control.
:::

## AI-Native by Design

While Hayride supports general-purpose application development, it is purpose-built for AI agentic workflows. The platform provides lightweight execution and fine-grained resource control for composing, orchestrating, and executing agents securely and efficiently.

Today’s AI systems face critical limitations in execution efficiency, data freshness, and secure automation. LLMs are powerful, but struggle with structured API interactions and secure environments for dynamic code execution.

<div style={{ textAlign: 'center' }}>
  <img src={require('/img/hayride-ai-example.png').default} width="80%"  />
  <p></p>
</div>


Hayride addresses these challenges with:

- **Function Calling** – Allows LLMs to automate workflows by interacting with structured APIs.
- **Retrieval-Augmented Generation (RAG)** – Enables real-time access to external data sources for relevant, contextual responses.
- **WebAssembly Execution** – Offers a portable, sandboxed environment for executing both LLM-generated and developer-authored code.

Together, these capabilities form a secure and extensible foundation for building adaptive, AI-native systems.

### AI Capabilities

Hayride includes features that simplify and strengthen AI agent development:

| Feature                        | Description                                                                                         |
|-------------------------------|-----------------------------------------------------------------------------------------------------|
| Agent Framework               | Multi-language framework for building and managing agents.                                          |
| Secure AI Execution           | Sandboxed runtime ensures safe execution without external dependencies.                             |
| AI Capabilities               | Native support for RAG, Function Calling, and model-based orchestration.                            |
| AI Model Portability          | Run inference against local or remote models for maximum deployment flexibility.                    |
| Tool Discovery & Extensibility| Access a growing tool registry to build secure, modular workflows.                                  |

---

Hayride is built for developers who need modularity, portability, and secure AI execution. Whether you're optimizing AI workloads, building multi-language systems, or enabling next-generation agentic applications, Hayride provides a solid, forward-looking foundation.
