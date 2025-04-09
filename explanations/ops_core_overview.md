# Ops-Core Overview

`ops-core` is the orchestration engine of the Opspawn Core Foundation. It is responsible for managing the lifecycle of agent tasks, coordinating workflows, and interacting with external systems.

## Key Components

1.  **API Layer (REST & gRPC):**
    -   Provides external interfaces (primarily REST via FastAPI) for submitting tasks, querying status, and managing workflows.
    -   Offers internal interfaces (gRPC) for potentially higher-performance communication between `ops-core` components or tightly coupled services.

2.  **Scheduler (`scheduler/engine.py` - `InMemoryScheduler`):**
    -   The central component responsible for receiving task requests (via API/gRPC).
    -   Validates requests and prepares tasks for execution by creating a task record in the Metadata Store.
    -   For agent tasks (`agent_run` type), it determines the necessary configuration (LLM provider, model, etc.) based on environment variables or task input.
    -   It **does not** directly instantiate agent components but rather packages the task details (ID, goal, config) into a message.
    -   Dispatches agent tasks to the asynchronous execution system by sending a message via the `execute_agent_task_actor.send()` method.

3.  **Metadata Store (`metadata/store.py`):**
    -   Tracks the state and history of all tasks managed by `ops-core`.
    -   Stores task definitions, status (Pending, Running, Success, Failed), input data, results, and timestamps.
    -   Currently implemented as an `InMemoryMetadataStore`. **Important:** This implementation is for MVP purposes and is **not persistent** (data is lost on restart) and **not thread-safe**, making it unsuitable for multi-worker or production environments. A persistent backend (e.g., PostgreSQL) is planned (Task 6.1).

4.  **Asynchronous Task Execution (Dramatiq + RabbitMQ):**
    -   Uses the Dramatiq library with a RabbitMQ broker for reliable, scalable background task processing.
    -   The `execute_agent_task_actor` (`scheduler/engine.py`) is defined as a Dramatiq actor, containing the logic to actually run an agent task.
    -   The Scheduler sends agent task messages to the queue associated with this actor.
    -   Dedicated Worker processes (started via `ops_core/tasks/worker.py`) consume messages from the queue and execute the `execute_agent_task_actor` logic.
    -   **Actor Logic (`_run_agent_task_logic` helper function):**
        -   Retrieves task details (ID, goal, config).
        -   Updates task status to RUNNING in the Metadata Store.
        -   Uses helper functions (`get_llm_client`, `get_planner` in `scheduler/engine.py`) to instantiate the correct `agentkit.BaseLlmClient` and `agentkit.BasePlanner` based on configuration (read from environment variables like `AGENTKIT_LLM_PROVIDER`, `AGENTKIT_LLM_MODEL`).
        -   Instantiates other necessary `agentkit` components (`ShortTermMemory`, `ToolRegistry`).
        -   Injects the `MCPProxyTool` into the `ToolRegistry` if the task requires it (determined by scheduler/config).
        -   Instantiates the `agentkit.Agent` with all the configured components.
        -   Calls `agent.run_async()` with the task goal.
        -   Upon completion, updates the task status (SUCCESS/FAILED) and stores the final result or error message in the Metadata Store.

5.  **MCP Client (`mcp_client/`):**
    -   Manages connections to external Model Context Protocol (MCP) servers based on configuration.
    -   Provides the `OpsMcpClient` class used by the `MCPProxyTool` (injected into agents) to make `call_tool` requests to external MCP servers.
    -   Handles MCP server discovery, connection management, and communication.

6.  **Configuration (`config/loader.py`):**
    -   Manages loading configuration settings for `ops-core`, primarily MCP server details (`mcp_config.yaml` or defaults) and potentially other operational parameters. LLM provider settings are currently read directly from environment variables within the scheduler/actor logic.

## Core Workflow (Agent Task)

1.  An external client submits a task request (e.g., goal description, task type `agent_run`) via the REST API.
2.  The API endpoint validates the request and calls the `Scheduler.submit_task` method.
3.  The Scheduler creates a task record in the Metadata Store (Status: PENDING).
4.  The Scheduler sends a message containing task details to the Dramatiq queue via `execute_agent_task_actor.send()`.
5.  A Worker process picks up the message.
6.  The Worker executes the `execute_agent_task_actor` logic (specifically the `_run_agent_task_logic` helper function):
    -   Updates task status to RUNNING in the Metadata Store.
    -   Instantiates the necessary `agentkit` components (LLM Client, Planner, Agent, Memory, Tools, potentially MCPProxyTool) using helper functions and configuration derived from environment variables.
    -   Calls the `agent.run_async()` method with the task goal.
    -   Waits for the agent execution to complete.
    -   Updates the task status (SUCCESS/FAILED) and stores the final result (including memory history) or error in the Metadata Store.
