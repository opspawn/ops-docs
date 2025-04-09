# Opspawn Core Foundation: System Architecture

This document provides an overview of the high-level architecture for the Opspawn Core Foundation.

## Core Components

The system follows a modular, service-oriented architecture composed of two primary components:

1.  **`agentkit`**: A Python toolkit/library designed for building Large Language Model (LLM)-powered agents. It focuses on providing modular components for agent capabilities like memory, planning, and tool use.
2.  **`ops-core`**: A Python-based orchestration engine responsible for managing the lifecycle of agent tasks. This includes scheduling task execution, tracking state, coordinating workflows, and managing interactions with external systems (including MCP servers).

## Interaction Model

-   `ops-core` acts as the primary entry point for initiating agent tasks.
-   It invokes functionalities within `agentkit` (e.g., running an agent for a specific goal) via internal APIs or function calls.
-   Communication between distributed `ops-core` components (like scheduler and workers) utilizes asynchronous messaging (Dramatiq + RabbitMQ).
-   External interactions are managed via REST APIs (provided by `ops-core`). Internal high-performance communication may leverage gRPC.

## Key Architectural Patterns

-   **Modularity & Composability:** Both `agentkit` and `ops-core` are designed with distinct, replaceable components.
-   **Service Orientation:** Clear separation of concerns between the agent building toolkit (`agentkit`) and the orchestration engine (`ops-core`).
-   **API-Driven:** Interactions between components and with external systems are mediated through well-defined APIs (REST, potentially gRPC).
-   **Asynchronous Processing:** Leverages `asyncio` within components and a message queue (`Dramatiq`/`RabbitMQ`) for scalable task execution.
-   **Stateful Orchestration:** `ops-core` maintains persistent state about tasks and workflows in a metadata store.
-   **Dynamic Proxy for MCP:** `ops-core` manages MCP interactions, injecting a proxy tool into `agentkit` agents for controlled access to external tools/resources via MCP.

*(This is an initial draft based on current understanding. It will be expanded as the project evolves.)*
