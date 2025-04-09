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
    -   Responsible for decomposing a high-level goal into a sequence of executable steps (represented by a `Plan` object containing `PlanStep` instances).
    -   Implementations use an injected `BaseLlmClient` to interact with an LLM, providing context (goal, history, available tools) and parsing the response into structured plan steps (e.g., thought, tool call, final answer).
    -   **Current Implementation:** `ReActPlanner` (`planning/react_planner.py`). This planner implements the ReAct (Reason + Act) prompting strategy, iteratively generating thoughts and actions (tool calls) until a final answer is reached.

3.  **Memory (`BaseMemory` interface, `memory/`):**
    -   Manages the agent's conversational history and context window.
    -   Provides `async` methods to `add_message` (human input, AI responses, tool results) and `get_context` (retrieving a formatted history suitable for LLM prompts, potentially managing token limits). Also includes `clear`.
    -   **Current Implementation:** `ShortTermMemory` (`memory/short_term.py`). This provides basic, volatile in-memory storage of the conversation history as a list of messages. It does not yet implement sophisticated context window management or long-term persistence (Task 6.4 is optional for LTM MVP).

4.  **Tool Manager (`BaseToolManager` interface, `tools/`):**
    -   Manages the tools available to the agent.
    -   **Tool Schemas (`tools/schemas.py`):** Defines `ToolSpec` (how a tool is described to the LLM) and `ToolResult`.
    -   **Tool Registry (`tools/registry.py`):** Concrete implementation (`ToolRegistry`) that holds available `Tool` objects. Provides `execute_tool` method.
    -   **Tool Execution (`tools/execution.py`):** Implements the crucial `execute_tool_safely` function. This function takes a `Tool` object and its arguments, then uses the `multiprocessing` module to run the tool's `run` method in a separate, sandboxed process. This isolation prevents faulty or malicious tool code from crashing the main agent process. It also enforces a timeout to prevent tools from hanging indefinitely. The result or any exception from the tool process is captured and returned in a standardized `ToolResult` object.
    -   **MCP Proxy Tool (`tools/mcp_proxy.py`):** Defines the `ToolSpec` (named `mcp_proxy_tool_spec`) for the special proxy tool. This spec describes the tool's purpose and input schema (`MCPProxyToolInput` Pydantic model, requiring `server_name`, `tool_name`, `arguments`) to the agent's planner LLM. The actual execution logic for this tool resides within `ops-core`.

5.  **Security Manager (`BaseSecurityManager` interface, `core/security.py`):**
    -   Interface defining security checks, primarily `check_permissions` which the `Agent` calls before executing a tool action proposed by the planner.
    -   Allows for implementing policies like tool whitelisting/blacklisting, argument validation, or rate limiting.
    -   **Current Implementation:** `DefaultSecurityManager` (`core/security.py`). This is a basic implementation that currently allows all tool calls by default. More sophisticated implementations can be swapped in.

6.  **LLM Clients (`BaseLlmClient` interface, `llm_clients/`):**
    -   Abstract interface (`core/interfaces/llm_client.py`) defining a standard way to interact with different LLM providers.
    -   Defines an `async generate` method that takes parameters like `prompt`, `history`, `system_prompt`, etc., and returns a standardized `LlmResponse` Pydantic model containing the generated text.
    -   **Concrete Implementations:** Provided for major LLM providers, handling provider-specific API calls, authentication (via environment variables), and response mapping:
        -   `OpenAiClient` (`llm_clients/openai_client.py`)
        -   `AnthropicClient` (`llm_clients/anthropic_client.py`)
        -   `GoogleClient` (`llm_clients/google_client.py` - uses the modern `google-genai` SDK)
        -   `OpenRouterClient` (`llm_clients/openrouter_client.py` - uses OpenAI-compatible API)
    -   These clients are instantiated by `ops-core` based on configuration and injected into the selected `BasePlanner` implementation.

## Core Workflow (Agent Run)

1.  `ops-core` instantiates an `Agent` with specific implementations of Planner, Memory, Tool Manager, Security Manager, and potentially injects tools (like the MCP Proxy Tool).
2.  `ops-core` calls `agent.run_async(goal=...)`.
3.  The Agent retrieves initial context from Memory.
4.  The Agent calls `planner.plan(goal=..., history=...)` to get the first step(s).
5.  The Agent enters an execution loop based on the `Plan`:
    -   If the step is a thought/reasoning step, add it to Memory.
    -   If the step is a tool call (`action_type="tool_call"`):
        -   The Agent calls `security_manager.check_permissions(tool_name=...)`. If denied, an error is typically raised or added to memory.
        -   If permitted, the Agent calls `tool_manager.execute_tool(tool_name=..., arguments=...)`. The `ToolRegistry` finds the corresponding `Tool` object and invokes `execute_tool_safely`.
        -   The resulting `ToolResult` (containing output or error) is added to Memory using `memory.add_message(...)`.
    -   If the step is the final answer (`action_type="final_answer"`), the Agent extracts the answer from the step details and returns it, exiting the loop.
    -   After processing a step (thought or tool call), the Agent retrieves the updated context from Memory (`memory.get_context()`).
    -   The Agent calls `planner.plan(...)` again with the updated history/context to get the next step.
6.  The loop continues until the planner provides a `final_answer` step or a maximum step limit/error condition is met.
7.  The final result (the answer text or potentially an error object) is returned to the caller (`ops-core`'s actor logic).
