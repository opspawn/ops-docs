# Ops-Core Overview

`ops-core` is the orchestration engine of the Opspawn Core Foundation. It is responsible for managing the lifecycle of agent tasks, coordinating workflows, and interacting with external systems.

## Key Components

1.  **API Layer (REST & gRPC):**
    -   Provides external interfaces (primarily REST via FastAPI) for submitting tasks, querying status, and managing workflows.
    -   Offers internal interfaces (gRPC) for potentially higher-performance communication between `ops-core` components or tightly coupled services.

2.  **Scheduler (`scheduler/engine.py`):**
    -   The central component responsible for receiving task requests (via API/gRPC).
    -   Validates requests and prepares tasks for execution.
    -   For agent tasks (`agent_run` type), it instantiates the required `agentkit` components (Agent, Planner, LLM Client, Memory, Tools) based on configuration.
    -   Injects the `MCPProxyTool` if required.
    -   Dispatches tasks to the asynchronous execution system (Dramatiq).

3.  **Metadata Store (`metadata/store.py`):**
    -   Tracks the state and history of all tasks managed by `ops-core`.
    -   Stores task definitions, status (Pending, Running, Success, Failed), input data, results, and timestamps.
    -   Currently implemented as an `InMemoryMetadataStore` (MVP limitation), with plans for a persistent backend (e.g., PostgreSQL).

4.  **Asynchronous Task Execution (Dramatiq + RabbitMQ):**
    -   Uses the Dramatiq library with a RabbitMQ broker for reliable, scalable background task processing.
    -   The `execute_agent_task_actor` (`scheduler/engine.py`) is defined as a Dramatiq actor.
    -   The Scheduler sends agent task messages to the queue.
    -   Dedicated Worker processes (`tasks/worker.py`) consume messages from the queue and execute the `execute_agent_task_actor` logic.
    -   The actor logic involves running the configured `agentkit` Agent and updating the Metadata Store with the final status and results.

5.  **MCP Client (`mcp_client/`):**
    -   Manages connections to external Model Context Protocol (MCP) servers based on configuration.
    -   Provides the `OpsMcpClient` class used by the `MCPProxyTool` (injected into agents) to make `call_tool` requests to external MCP servers.
    -   Handles MCP server discovery, connection management, and communication.

6.  **Configuration (`config/loader.py`):**
    -   Manages loading configuration settings for `ops-core`, including MCP server details and potentially LLM provider settings, typically from environment variables or configuration files.

## Core Workflow (Agent Task)

1.  An external client submits a task request (e.g., goal description, task type `agent_run`) via the REST API.
2.  The API endpoint validates the request and calls the `Scheduler.submit_task` method.
3.  The Scheduler creates a task record in the Metadata Store (Status: PENDING).
4.  The Scheduler sends a message containing task details to the Dramatiq queue via `execute_agent_task_actor.send()`.
5.  A Worker process picks up the message.
6.  The Worker executes the `execute_agent_task_actor` logic:
    -   Updates task status to RUNNING in the Metadata Store.
    -   Instantiates the necessary `agentkit` components (Agent, Planner, LLM Client, Memory, Tools, potentially MCPProxyTool) based on the task details and system configuration.
    -   Calls the `agent.run_async()` method with the task goal.
    -   Waits for the agent execution to complete.
    -   Updates the task status (SUCCESS/FAILED) and stores the result/error in the Metadata Store.

*(This is an initial draft based on current understanding. It will be expanded as the project evolves.)*
