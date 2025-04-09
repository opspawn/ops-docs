# Agentkit Overview

`agentkit` is a Python toolkit designed for building modular and capable Large Language Model (LLM)-powered agents within the Opspawn ecosystem. It provides core components and interfaces to structure agent logic.

## Key Components & Interfaces

`agentkit` is built around a set of core interfaces (`agentkit/core/interfaces/`) and concrete implementations.

1.  **Agent (`core/agent.py`):**
    -   The central orchestrator for an individual agent's execution cycle.
    -   Takes a goal and uses its injected components (Planner, Memory, Tool Manager, Security Manager) to achieve it.
    -   Manages the step-by-step execution loop based on the planner's output.
    -   Interacts with memory to maintain context and history.
    -   Invokes tools via the Tool Manager.

2.  **Planner (`BasePlanner` interface, `planning/`):**
    -   Responsible for decomposing a high-level goal into a sequence of executable steps (a `Plan`).
    -   Implementations can use various strategies (e.g., ReAct, simple prompting).
    -   Requires an LLM client to generate the plan.
    -   Current implementation: `ReActPlanner` (`planning/react_planner.py`).

3.  **Memory (`BaseMemory` interface, `memory/`):**
    -   Manages the agent's conversational history and context window.
    -   Provides methods to add messages (human, AI, tool results) and retrieve the current context relevant for the LLM prompt.
    -   Current implementation: `ShortTermMemory` (`memory/short_term.py`) for in-memory context. (Long-term memory TBD).

4.  **Tool Manager (`BaseToolManager` interface, `tools/`):**
    -   Manages the tools available to the agent.
    -   **Tool Schemas (`tools/schemas.py`):** Defines `ToolSpec` (how a tool is described to the LLM) and `ToolResult`.
    -   **Tool Registry (`tools/registry.py`):** Concrete implementation (`ToolRegistry`) that holds available `Tool` objects. Provides `execute_tool` method.
    -   **Tool Execution (`tools/execution.py`):** Implements `execute_tool_safely` using `multiprocessing` to run tool code in a sandboxed process with timeouts for security and stability.
    -   **MCP Proxy Tool (`tools/mcp_proxy.py`):** Defines the `ToolSpec` for the special proxy tool injected by `ops-core` to enable controlled access to external MCP servers.

5.  **Security Manager (`BaseSecurityManager` interface):**
    -   Interface for defining security checks, primarily around tool usage.
    -   The `Agent` consults the security manager before executing a tool.
    -   Current implementation: `DefaultSecurityManager` (`core/security.py`) provides a basic placeholder.

6.  **LLM Clients (`BaseLlmClient` interface, `llm_clients/`):**
    -   Abstract interface (`core/interfaces/llm_client.py`) for interacting with different LLM providers.
    -   Defines a standard `generate` method and `LlmResponse` model.
    -   Concrete implementations exist for OpenAI, Anthropic, Google Gemini (`google-genai` SDK), and OpenRouter.
    -   These clients are typically injected into the Planner.

## Core Workflow (Agent Run)

1.  `ops-core` instantiates an `Agent` with specific implementations of Planner, Memory, Tool Manager, Security Manager, and potentially injects tools (like the MCP Proxy Tool).
2.  `ops-core` calls `agent.run_async(goal=...)`.
3.  The Agent retrieves initial context from Memory.
4.  The Agent calls `planner.plan(goal=..., history=...)` to get the first step(s).
5.  The Agent enters an execution loop based on the `Plan`:
    -   If the step is a thought/reasoning step, add it to Memory.
    -   If the step is a tool call:
        -   Check permissions with the Security Manager.
        -   Call `tool_manager.execute_tool(...)` (which uses `execute_tool_safely`).
        -   Add the `ToolResult` to Memory.
    -   If the step is the final answer, return it.
    -   Retrieve updated context from Memory.
    -   Call `planner.plan(...)` again to get the next step.
6.  The loop continues until the planner returns a final answer or an error occurs.
7.  The final result (or error) is returned to the caller (`ops-core`).

*(This is an initial draft based on current understanding. It will be expanded as the project evolves.)*
