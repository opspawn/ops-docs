# Opspawn Core Foundation: System Architecture

This document provides an overview of the high-level architecture for the Opspawn Core Foundation.

## Core Components

The system follows a modular, service-oriented architecture composed of two primary components:

1.  **`agentkit`**: A Python toolkit/library designed for building Large Language Model (LLM)-powered agents. It focuses on providing modular components for agent capabilities like memory, planning, and tool use.
2.  **`ops-core`**: A Python-based orchestration engine responsible for managing the lifecycle of agent tasks. This includes scheduling task execution, tracking state, coordinating workflows, and managing interactions with external systems (including MCP servers).

## Interaction Model & Communication

The interaction between the core components follows these patterns:

1.  **Task Initiation:** External clients or systems interact with `ops-core` via its **REST API Gateway** (built with FastAPI) to submit tasks (e.g., "run agent with goal X").
2.  **Scheduling:** The `ops-core` **Scheduler** receives the task request. For agent tasks, it determines the necessary configuration (LLM provider, model, etc.) and potentially injects dependencies like the MCP Proxy Tool.
3.  **Asynchronous Execution:** The Scheduler sends the task details as a message to a **Dramatiq queue** (backed by RabbitMQ).
4.  **Worker Processing:** An `ops-core` **Worker** picks up the message from the queue and executes the corresponding Dramatiq actor (`execute_agent_task_actor`).
5.  **Agent Invocation:** The actor logic within the worker instantiates the required `agentkit` **Agent**, providing it with the necessary configuration (LLM client, planner, memory, tool registry including the MCP proxy tool if applicable).
6.  **Agent Execution:** The `agentkit` Agent runs its internal loop (planning, acting, observing) to achieve the goal, potentially calling tools registered in its `ToolRegistry`.
7.  **MCP Proxy Handling (if used):** If the agent calls the `mcp_proxy_tool`, the call is handled by `ops-core`'s **MCP Client**, which communicates with the target external MCP Server. The result is returned to the agent.
8.  **Result Storage:** Upon completion (success or failure), the worker updates the task's final state and result in the `ops-core` **Metadata Store**.
9.  **Status Retrieval:** External clients can query the task status and results via the `ops-core` REST API.

**Internal Communication:**
-   **REST API:** Primary external interface (FastAPI).
-   **Dramatiq/RabbitMQ:** Used for asynchronous task queuing between the Scheduler and Workers.
-   **gRPC:** Defined (`proto/tasks.proto`) and implemented (`grpc_internal/task_servicer.py`) for potential high-performance internal communication needs, although the primary flow currently relies on Dramatiq.
-   **Direct Python Calls:** Within a worker process, `ops-core` directly instantiates and calls methods on `agentkit` components.

## System Diagram

```mermaid
flowchart TD
    subgraph External
        Client[External Client]
        MCP_Server[External MCP Server]
    end

    subgraph ops-core
        direction LR
        API[API Gateway (FastAPI)] --> Scheduler[Scheduler Engine]
        Scheduler -- Task Message --> MQ([Dramatiq Queue<br>(RabbitMQ)])
        MQ -- Task Message --> Worker[Worker (Dramatiq Actor)]
        Worker --> Metadata[(Metadata Store<br>InMemory/DB)]
        Worker --> AgentKit_Invoke(Invoke Agent)
        Worker --> MCP_Client[MCP Client Module]

        subgraph Agent Interaction
            AgentKit_Invoke -->|Instantiates| Agent[agentkit.Agent]
            Agent -->|Uses| Planner[agentkit.Planner]
            Agent -->|Uses| Memory[agentkit.Memory]
            Agent -->|Uses| ToolRegistry[agentkit.ToolRegistry]
            ToolRegistry -- Contains --> MCP_Proxy_Tool(MCP Proxy Tool)
            Agent -- Calls Tool --> MCP_Proxy_Tool -- Relays Call --> MCP_Client
        end
    end

    Client -- REST Request --> API
    API -- REST Response --> Client
    API -- Read/Write --> Metadata
    MCP_Client -- MCP Call --> MCP_Server
    MCP_Server -- MCP Response --> MCP_Client

    classDef opsCore fill:#D6EAF8,stroke:#333,stroke-width:2px;
    classDef agentKit fill:#D5F5E3,stroke:#333,stroke-width:2px;
    classDef external fill:#EBDEF0,stroke:#333,stroke-width:2px;

    class API,Scheduler,Worker,Metadata,MCP_Client opsCore;
    class Agent,Planner,Memory,ToolRegistry,MCP_Proxy_Tool agentKit;
    class Client,MCP_Server external;
```

## Key Architectural Patterns

-   **Modularity & Composability:** Both `agentkit` and `ops-core` are designed with distinct, replaceable components.
-   **Service Orientation:** Clear separation of concerns between the agent building toolkit (`agentkit`) and the orchestration engine (`ops-core`).
-   **API-Driven:** Interactions between components and with external systems are mediated through well-defined APIs (REST, potentially gRPC).
-   **Asynchronous Processing:** Leverages `asyncio` within components and a message queue (`Dramatiq`/`RabbitMQ`) for scalable task execution.
-   **Stateful Orchestration:** `ops-core` maintains persistent state about tasks and workflows in a metadata store.
-   **Dynamic Proxy for MCP:** This pattern centralizes external tool access control within `ops-core`.
    1.  `ops-core` acts as the MCP Host/Client, managing connections and configurations.
    2.  When an agent task requires external access, `ops-core` injects a special `mcp_proxy_tool` (defined in `agentkit`) into the agent's `ToolRegistry` during instantiation.
    3.  The `agentkit` agent's planner can decide to use this tool, specifying the target server, tool name, and arguments.
    4.  The call to `mcp_proxy_tool` is routed back to the `ops-core` worker process.
    5.  `ops-core`'s `OpsMcpClient` module handles the actual communication with the specified external MCP Server.
    6.  The result from the external server is returned to the agent via the proxy tool's result.
    7.  This keeps `agentkit` focused on agent logic and `ops-core` focused on orchestration and secure external interactions.
