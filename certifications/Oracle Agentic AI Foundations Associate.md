# AI Agents, LangChain, MCP, OpenAI Agents, and Oracle AI Database


This guide combines the main ideas from the notes into one study reference. It uses plain English where possible, explains technical terms when they first appear, gives examples, and points out details that depend on a software version or setup.

**How to use this guide:** Read the short explanation first, then study the example and the technical details. Code samples are for learning unless they are explicitly described as complete. Before using code in a real system, check the official documentation for your database, framework, and installed version.

---

# 1. AI agent fundamentals

## What is an AI agent?

A practical mental model is:

**Agent = model + instructions + tools + context/state + orchestration**

An agent receives a goal, decides what to do next, can call approved tools, observes the results, and continues until it finishes or reaches a limit. Not every agent is highly autonomous; autonomy depends on the workflow and permissions provided.

| Component | Responsibility |
|---|---|
| Model (LLM) | Interprets the request, proposes actions/tool calls, interprets results, generates output |
| Instructions | Define role, scope, behavior, and constraints |
| Tools | Provide access to APIs, databases, search, calculations, or actions |
| Orchestrator/runtime | Executes the loop, routes calls, manages state, errors, retries, and stop conditions |
| Context and memory | Supplies conversation history and, where configured, durable facts or retrieved knowledge |
| Guardrails/application logic | Enforces permissions, validation, limits, approvals, and output checks |
| Observability | Records information needed to diagnose errors, latency, usage, and cost |

Typical cycle: **Receive input → decide → request action → execute tool → observe result → continue or respond.**

### Key distinctions
- Training data, retrieved documents, and live tool results are different information sources; “knowledge = training data + tools” is too simplistic.
- The context window is the information supplied to a model call; it is not automatically persistent memory.
- Conversation history stores turns. Long-term memory stores selected information for later retrieval. RAG retrieves external source material to ground a response.
- The model may request a tool call; the application/framework or remote tool server executes it.
- A tool call is not safe just because the model selected it. Validate arguments and authorize actions outside the model.

### Failure modes and controls
Common failures include loops, incorrect tool selection, malformed arguments, stale retrieval, prompt injection, context overflow, cascading errors, excessive cost/latency, and unsafe side effects.

Use bounded iterations, timeouts, rate limits, token/cost budgets, schema validation, least-privilege credentials, idempotent tools where possible, human approval for high-impact actions, and evaluation/tracing.

## Reasoning patterns

| Pattern | Description | Caution |
|---|---|---|
| Chain-of-thought prompting | Asks a model to work through a problem before answering | A generated explanation is not guaranteed to be a faithful record of the model's internal process |
| ReAct | Alternates between reasoning, an action, and an observation | Tool results can improve grounding but do not guarantee correctness |
| Tree of Thoughts | Explores and evaluates multiple candidate paths | Often more expensive and slower |

These are conceptual patterns, not required parts of every agent framework. Native tool calling can implement an agent loop without exposing a literal Thought/Action/Observation transcript.

## Tool-calling flow
1. The application supplies conversation context and tool schemas.
2. The model returns either a normal response or a structured tool-call request.
3. The application/framework validates and executes the tool.
4. The result is returned to the model as an observation.
5. The model responds or requests another tool call.

Tool schemas commonly contain a name, description, and parameter schema. In Python frameworks, type hints and a docstring may be used to construct the schema. Vague descriptions can lead to poor tool selection.

**Security rule:** prompts and model-level guardrails help, but are not a security boundary. Enforce permissions and critical business rules in code and the underlying systems.

---

# 2. LangChain agents and `invoke()`

In LangChain v1, `create_agent` creates an agent graph that calls a model and executes tools in a loop until a stopping condition is reached. It is built on LangGraph. Older tutorials may use older APIs such as `create_react_agent`; verify your installed version.

### Minimal example: understand the agent loop

This first example deliberately uses a deterministic tool. It teaches how a tool is defined and passed to an agent without needing a weather service.

Install the packages:

```bash
python -m pip install -U langchain langchain-openai
```

Set `OPENAI_API_KEY` in your environment before running the example. The sample uses `gpt-5.5` as the model identifier; select a model enabled for your account if that model is unavailable.

```python
from langchain.agents import create_agent
from langchain.tools import tool

@tool
def lookup_office_hours(day: str) -> str:
    """Return the office opening hours for a weekday."""
    hours = {
        "monday": "9:00 AM to 5:00 PM",
        "tuesday": "9:00 AM to 5:00 PM",
        "wednesday": "9:00 AM to 5:00 PM",
        "thursday": "9:00 AM to 5:00 PM",
        "friday": "9:00 AM to 4:00 PM",
    }
    return hours.get(day.lower(), "Hours are not available for that day.")

agent = create_agent(
    model="openai:gpt-5.5",
    tools=[lookup_office_hours],
    system_prompt="Answer clearly. Use the office-hours tool for questions about opening hours.",
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "When does the office close on Friday?"}]
})

print(result["messages"][-1].content)
```

**What happens:** LangChain sends the user question and tool description to the model. If the model requests `lookup_office_hours`, the runtime executes the Python function, gives the result back to the model, and obtains the final answer. The model name and provider integration can change; check the current LangChain model integration documentation for your installed version.

### More complete example: inspect messages and handle failures

```python
from langchain.agents import create_agent
from langchain.tools import tool

@tool
def get_order_status(order_id: str) -> str:
    """Look up an order in this demo data. Do not use for real customer records."""
    demo_orders = {
        "A100": "Packed",
        "A101": "Shipped",
    }
    status = demo_orders.get(order_id.upper())
    if status is None:
        return "Order not found in demo data."
    return status

agent = create_agent(
    model="openai:gpt-5.5",
    tools=[get_order_status],
    system_prompt=(
        "Help users check demo order status. Ask for an order ID if it is missing. "
        "Never invent an order status."
    ),
)

try:
    result = agent.invoke({
        "messages": [{"role": "user", "content": "Can you check order A101?"}]
    })
    for message in result["messages"]:
        print(f"{type(message).__name__}: {message.content}")
except Exception as exc:
    # In production, log a safe error identifier; do not expose secrets.
    print(f"Agent run failed: {type(exc).__name__}: {exc}")
```

The second example is complete for a local demonstration, but it is not a production order system. Real systems need authentication, authorization, input validation, reliable database access, tests, and safe error handling.

## What `agent.invoke()` does
1. Accepts the current input/state.
2. Calls the model with message history and available tool schemas.
3. If the model returns tool calls, resolves each tool name to a registered tool.
4. Executes tool code in the application/runtime.
5. Adds results as tool messages associated with their tool-call IDs.
6. Calls the model again with updated history.
7. Continues until there are no more tool calls or a limit/error stops the workflow.

A single `invoke()` can therefore perform multiple model calls and tool executions.

- **Tool name** identifies which registered tool to execute.
- **Tool-call ID** correlates the result with the request, especially when multiple calls are returned.
- Independent tool calls may run in parallel.
- The model is not running Python when it returns a tool call; the runtime executes the function.
- `invoke()` alone does not guarantee memory across separate invocations. Persistent state typically requires a checkpointer/thread ID or explicitly supplied history.
- Set sensible graph/recursion limits. Decide whether tool errors should be returned to the model for recovery or stop the workflow.
- Long histories increase latency and token use; consider trimming, summarization, and prompt caching where supported.

---

# 3. Tool-call message example

This simplified example shows the logic, not one provider's exact wire format.

1. `system`: instructions.
2. `user`: asks a question.
3. `assistant`: returns a structured request to call `get_weather`.
4. `tool`: contains the result and matching call ID.
5. `assistant`: reads the result and answers.

```json
{
  "id": "call_abc123",
  "name": "get_weather",
  "arguments": {
    "city": "Paris",
    "units": "celsius"
  }
}
```

The actual API shape differs by provider. OpenAI Responses API function-call arguments, for example, are represented as a JSON string; frameworks may normalize them into a dictionary.

Evaluation example: if a user asks about next week's weather but a tool returns only current weather, the agent must not present current conditions as a forecast. It should obtain forecast data or explain the limitation.

---

# 4. Model Context Protocol (MCP)

## What MCP is
The **Model Context Protocol (MCP)** is an open protocol for connecting AI applications to external tools and context. It uses JSON-RPC messages and standardizes tool discovery and invocation. MCP complements model function calling; it does not replace the model or agent orchestrator.

- **Host:** AI application managing the model, user experience, and tool-use policy.
- **Client:** component within the host that connects to an MCP server.
- **Server:** exposes tools, resources, and/or prompts.

The model does not directly speak MCP. The host supplies tool definitions to the model, interprets its choice, and asks the MCP client to invoke the selected server tool.

### MCP vs function calling
- Function calling is how the model returns a structured request to call a function.
- MCP standardizes how an application discovers and calls tools exposed by a server.
- A host can convert MCP tool schemas into the model provider's tool schema.

## Capabilities
- **Tools:** actions the model can request, such as querying an API or performing a calculation.
- **Resources:** contextual data identified by URI that the host/application may choose to read and provide.
- **Prompts:** reusable prompt templates offered by the server.

Tool choice may be model-driven, but the protocol does not require every host to expose every tool automatically. Hosts can restrict tools or require approval.

## Lifecycle and JSON-RPC
For protocol versions using the classic initialization handshake:
1. Client sends `initialize` with protocol version, capabilities, and client information.
2. Server responds with its version, capabilities, and server information.
3. Client sends `notifications/initialized`.
4. During operation, client discovers tools with `tools/list` and calls tools with `tools/call`.
5. Client closes the transport when finished.

**Important current-version warning:** MCP evolves. The 2026-07-28 specification introduces a stateless protocol core and retires the classic `initialize` / `notifications/initialized` handshake for the new protocol version. Older clients and servers may still use earlier versions. The lifecycle described above explains the classic 2025-06-18 style, not every current revision. The Python SDK may abstract parts of the handshake, but do not copy raw JSON-RPC examples across protocol versions without checking the matching specification. Pin and test compatible client/server SDK versions.

- A JSON-RPC request has an `id` and expects a response.
- A response echoes the request ID.
- A notification has no ID and does not expect a response.
- JSON-RPC request IDs correlate protocol messages; they differ from model tool-call IDs.

### Error categories
- **Tool execution error:** server returns a normal tool result with `isError: true`.
- **Protocol error:** server returns a JSON-RPC `error` object, e.g. for an unknown method or invalid parameters.

## Transports
| Transport | How it works | Typical use |
|---|---|---|
| stdio | Host launches a server process and exchanges newline-delimited JSON-RPC over stdin/stdout | Local developer tools |
| Streamable HTTP | Server exposes an HTTP MCP endpoint; responses may stream using SSE | Remote/shared services |

For stdio, do not write logs or arbitrary text to stdout; reserve stdout for protocol messages and use stderr for logs. For HTTP servers, implement authentication, validate the `Origin` header, and use safe binding/network defaults. The older HTTP+SSE transport was superseded by Streamable HTTP in the 2025-03-26 protocol revision, although implementations may retain compatibility.

## LangChain MCP adapter example
```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient({
    "math": {
        "transport": "stdio",
        "command": "python",
        "args": ["/path/to/math_server.py"],
    },
    "weather": {
        "transport": "streamable_http",
        "url": "http://localhost:8000/mcp",
    },
})

tools = await client.get_tools()
# Pass these tools to create_agent(...)
```

This is illustrative. Check your installed adapter version for accepted transport names, session behavior, and client configuration.

### `uvx` and supply-chain security
`uvx` (alias for `uv tool run`) can run a Python package in an isolated environment. It can download and execute package code, so it is not risk-free. Verify the package source, permissions, and documentation; pin versions for reproducibility. Example package names and environment variables must be verified against the server's README.

### MCP security
Treat third-party servers and tool descriptions as untrusted. Validate inputs and outputs, apply authentication/authorization, rate limits, timeouts, and least-privilege credentials. Require human confirmation for sensitive actions where appropriate. Avoid logging secrets or unnecessary personal data. MCP does not automatically grant safe access to a database or cloud account.

### Simplified JSON-RPC examples

List tools:
```json
{"jsonrpc":"2.0","id":2,"method":"tools/list"}
```

Call a tool:
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {"name": "add", "arguments": {"a": 3, "b": 4}}
}
```

Successful result:
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {"content": [{"type": "text", "text": "7"}], "isError": false}
}
```

Exact schemas and transport headers depend on protocol version; optional fields are omitted here for clarity.

---

# 5. OpenAI Responses API and Agents SDK

## Responses API vs Agents SDK
| Component | Responsibility | Who manages the multi-step loop? |
|---|---|---|
| Models | Generate output and tool-call requests | Not applicable |
| Responses API | Model requests, hosted tools, function calling, streaming, supported state features | Application manages multi-step loops when required |
| Agents SDK | Higher-level agent orchestration, tool execution, handoffs, guardrails, sessions, tracing | SDK Runner |

A Responses API request can include tools and return a tool call, but for **application-defined function tools**, your application normally executes the function and submits its result. Certain hosted tools execute on the provider side.

## Responses API concepts
- Endpoint: `/v1/responses`.
- Requests contain input and may include instructions, tools, output constraints, and supported conversation-state fields.
- Responses contain structured output items, such as messages and function-call items.
- Hosted tools (for example, web search, file search, or code interpreter) run within the provider's hosted service when supported and configured.
- Application-defined function calls are requests, not proof that the function has run.

### Conversation state
- A stateless request requires the caller to provide needed context.
- `previous_response_id` can link a new response to a prior response when supported and enabled.
- The Conversations API provides a conversation object.
- State linking simplifies application-side history management; do not assume it makes earlier context free of token billing.
- Supply tools/settings as required by the API on each request; do not assume they are automatically inherited.

**Migration note:** OpenAI's official migration guide says the Assistants API was sunset on 26 August 2026. New implementations should use the current Responses API guidance.

## Agents SDK
Core concepts:
- **Agent:** instructions, model, tools, optional output type, handoffs, and guardrails.
- **Runner:** executes the agent loop.
- **Function tools:** application-defined functions.
- **Agents as tools:** a specialist agent called by another agent.
- **Handoffs:** transfer control to another agent.
- **Sessions:** preserve conversation state across runs using a supported session implementation.
- **Tracing:** records model calls, tool activity, handoffs, and supported events.

The SDK API, supported providers, and parameter names can change. Use docs for the exact Python or JavaScript package version. Avoid blanket provider-compatibility claims unless current SDK documentation supports them.

Example (Python):
```python
import os
from agents import Agent, Runner, function_tool

@function_tool
def lookup_order(order_id: str) -> str:
    # Replace with an authenticated, permission-checked service call.
    return f"Demo result for order {order_id}"

agent = Agent(
    name="Order assistant",
    instructions="Answer order questions. Use the tool when a lookup is required.",
    tools=[lookup_order],
)

result = Runner.run_sync(agent, "Look up order A123")
print(result.final_output)
```

This is a teaching example. Configure a supported model/provider, credentials, and SDK version before running it.

## Handoff vs manager pattern
| Pattern | What happens | Good fit |
|---|---|---|
| Handoff | Triage agent transfers control to a specialist that continues the conversation | Routing to a specialist who owns the next phase |
| Manager / agents as tools | Manager calls specialists as tools and keeps responsibility for the final answer | Combining specialist outputs into one response |

Each additional model/agent call can increase latency and cost. Start with one agent and well-defined tools unless separation provides a clear benefit.

## Guardrails and tripwires
Guardrails can validate input, output, or tool calls. A tripwire signals a guardrail condition and may stop the run by raising a documented exception.

- Guardrails do not replace authorization or deterministic business rules.
- If an input guardrail runs in parallel with the main agent, the agent may already have started and could have consumed tokens or attempted tool calls before the guardrail finishes.
- For side-effecting workflows, use a blocking/pre-execution check where supported by the installed SDK.
- Catch documented exceptions and return a safe response.
- Validate exact decorator parameters and behavior against the SDK version.

Tracing can record model calls, tool calls, handoffs, and guardrail activity. Traces may contain prompts, tool arguments, or returned data, so configure retention and sensitive-data controls.

## When to use each approach
- **Responses API directly:** explicit control over requests and application workflow logic.
- **Agents SDK:** higher-level loop, tool orchestration, handoffs, guardrails, sessions, and tracing.
- **LangChain:** agent orchestration and integrations across supported model providers.
- **MCP:** standard discovery/invocation of tools exposed by external or reusable servers.

These options can be combined: an agent framework can connect to MCP tools and use a provider's model API.

---

# 6. OCI Enterprise AI Agents

This section preserves the OCI concepts from the source notes. Check current Oracle documentation for exact product names, availability, and configuration.

## Generic enterprise agent stack
| Layer | Responsibility |
|---|---|
| User experience / agent definition | Persona, instructions, tools, knowledge sources, business outcome |
| Request handling | Authentication, authorization, validation, rate limiting, routing |
| Runtime | Execution environment for the agent application |
| Orchestration and state | Multi-step workflows, retries, conversation state |
| Tool sandbox | Isolated code execution and resource limits where supported |
| Integration | Connectors, APIs, databases, credentials |
| Scaling and reliability | Replica scaling, load distribution, timeout/retry behavior |
| Observability | Logs, metrics, traces where instrumented, audits |

This is a conceptual architecture, not an Oracle product catalogue. A deployment may place these responsibilities in different services.

## Model capabilities and access modes
- **Chat/generative models:** generate conversational or structured output; tool use depends on model/API support.
- **Embedding models:** convert compatible input into vectors for semantic retrieval. Dimensions depend on the chosen model.
- **Rerank models:** reorder candidate documents by query relevance. Scores are model-specific and should not automatically be interpreted as calibrated probabilities.

**On-demand:** shared managed model-serving infrastructure, generally usage-based. Suitable for experimentation and supported workloads that fit its limits.

**Dedicated AI cluster:** dedicated GPU capacity with cluster-based commitment/pricing. Used for workloads needing dedicated capacity and supported fine-tuning or custom/imported model workflows.

Do not memorise a fixed endpoint-per-cluster number without checking current Oracle limits. Model availability, modes, regions, and quotas vary.

## OCI Responses API and hosted applications

### Responses API
An API-first route for supported model interaction and agentic tools. Supported tools may include file search, code interpreter, function calling, and MCP calling depending on model/API configuration.

For an application-defined function, the model requests the call; application/runtime code executes it and returns the result.

### Hosted agentic application
Typical lifecycle:
1. Develop the agent application.
2. Package it into a container image.
3. Push the image to OCIR.
4. Configure the OCI Generative AI application and runtime.
5. Create/activate a deployment.
6. Invoke the provisioned endpoint.

OCI manages supported hosting infrastructure, but developers remain responsible for code, dependencies, permissions, safe tool logic, testing, and operational readiness.

### Memory terminology
- **Conversation history:** turns associated with a conversation.
- **Context compaction:** reduces the size of earlier history where supported/configured; important details may be lost.
- **Long-term memory:** selected information retained for later retrieval, subject to configuration and retention.
- **RAG:** retrieval from knowledge sources, distinct from conversation memory.

## Governance and reliability
Use IAM and least-privilege policies. Private endpoints/network rules do not replace authentication or authorization. Configure guardrails explicitly where supported; they reduce risk but do not guarantee safety. Validate tool arguments, restrict credentials, apply timeouts/retries, and require approval for high-impact operations. Enable logs, metrics, audit records, and application tracing as needed. Autoscaling does not automatically make stateful workflows or external side effects safe.

---

# 7. Oracle AI Database Agentic AI

## Oracle AI Vector Search
Oracle AI Vector Search is integrated into Oracle Database and allows vectors to be stored and searched alongside relational data.

Typical workflow:
1. Extract text from source documents.
2. Chunk the text and preserve source/permission metadata.
3. Generate embeddings with a compatible model.
4. Store embeddings in a `VECTOR` column.
5. Create an appropriate vector index when useful.
6. Embed the query and retrieve similar chunks.
7. Use retrieved passages to ground an LLM response.

### Indexes and hybrid search
- **HNSW:** graph-based approximate nearest-neighbor indexing.
- **IVF:** partitions vector space and searches selected partitions.
- **Hybrid search:** combines semantic retrieval with text/keyword retrieval.

Index type, syntax, supported metrics, and hybrid-search features depend on database release. A vector index is not required for every small dataset.

### Distance functions
Oracle's documented functions include `COSINE_DISTANCE`, `L1_DISTANCE`, `L2_DISTANCE`, `INNER_PRODUCT`, `HAMMING_DISTANCE`, and `JACCARD_DISTANCE`, subject to supported vector types/metrics.

`VECTOR_DISTANCE(vector1, vector2, metric)` compares compatible vectors. Oracle's `DOT` metric uses the **negative dot product**; do not treat it as the same sign convention as `INNER_PRODUCT`. Lower distance generally means closer vectors for distance metrics.

Illustrative SQL:
```sql
SELECT document_id,
       chunk_text,
       VECTOR_DISTANCE(embedding, :query_embedding, COSINE) AS distance
FROM document_chunks
WHERE department = :allowed_department
ORDER BY distance
FETCH FIRST 5 ROWS ONLY;
```

A user-supplied filter is not sufficient authorization by itself. Enforce access through trusted application logic and database security.

### Text extraction and embeddings
The source notes refer to `DBMS_VECTOR_CHAIN`:
- `UTL_TO_TEXT` for text extraction in supported workflows.
- `UTL_TO_CHUNKS` to split text into chunks.
- `UTL_TO_EMBEDDINGS` to generate embeddings with a configured model.

Check exact package/API syntax, supported file types, credentials, and model setup for your database release. ONNX models must meet Oracle's supported format and compatibility requirements. An external embedding API also requires network, credential, data-egress, and privacy controls.

## Oracle AI Database Private Agent Factory
Agent Factory is a deployable no-code/low-code platform for building, testing, and publishing enterprise agents and workflows. It is not the same as Vector Search and should not be described as an agent that automatically lives inside the database.

### Knowledge Agent
Designed for enterprise search and question answering over approved unstructured sources. Typical flow:
1. Connect and ingest approved sources.
2. Extract/chunk content and index it.
3. Retrieve relevant passages.
4. Generate an answer grounded in those passages.
5. Return source references/citations where supported.

Sources and connectors vary by release/configuration. Connecting a source does not automatically prove that source-system permissions are enforced for each end user.

### Data Analysis Agent
Designed for structured database data. It can help translate natural-language questions into SQL, understand schema/table context, explain results, and support visualizations or suggested questions where available.

Generated SQL must be controlled. Restrict schemas and operations, use least-privilege database accounts, apply query limits, and validate results.

### Agent Builder
The visual builder can support simple workflows, tool-enabled agents, custom components (including Python where supported), and multi-agent orchestration. “No-code” does not remove the need for design, testing, credentials, access control, and maintenance. Pre-built agents and deployment options vary by release.

## Oracle Select AI and Select AI Agent

These names are related, but they do different jobs. Understanding the difference is important.

### 1. What is Select AI?

**Select AI** is an Oracle Database feature that lets people use natural language—ordinary sentences instead of SQL code—to work with data and AI models.

For example, instead of writing a SQL query, a user might ask:

> “Show the five products that generated the most revenue last month.”

Select AI can use an AI model and information about the database tables to generate SQL. Depending on the action selected, it can show the generated SQL, run it, explain it, or describe the results in normal language.

Select AI also supports other workflows, including chat, retrieval-augmented generation (RAG), and synthetic data generation. Which features are available depends on the Oracle Database product and version.

The main database package for Select AI is `DBMS_CLOUD_AI`.

### 2. What is Select AI Agent?

**Select AI Agent** is Oracle's agent framework for building workflows that can take several steps and use tools to complete a goal. It is more than asking an LLM to generate one SQL statement.

An agent can:
1. Read the user's request.
2. Decide what steps are needed.
3. Choose an available tool for the next step.
4. Run that tool.
5. Inspect the result (called an *observation*).
6. Decide whether it needs another step.
7. Return a final answer or action result.

The framework can maintain conversation context across turns when configured to do so. It uses an AI model selected through an AI profile. Oracle describes the agent pattern as ReAct (reasoning and acting): the model alternates between deciding what to do and using tools. This does not mean the model's decisions are always correct, so permissions and validation are still required.

The main package for managing agents, tools, tasks, and teams in the SQL/PL/SQL interface is `DBMS_CLOUD_AI_AGENT`. Select AI and Select AI Agent work together, but they are not the same feature.

### 3. Important building blocks

Think of these parts as a small team with clear responsibilities:

| Part | In plain English | Example |
|---|---|---|
| **AI profile** | Settings that tell Oracle which AI provider/model to use and what database objects are in scope | Use a configured model and allow access to approved sales tables |
| **Agent** | A worker with a defined role and instructions | “You help employees check order status” |
| **Tool** | An action the agent is allowed to perform | Query approved tables, search approved knowledge, or call a PL/SQL function |
| **Task** | A specific job with instructions and rules | “Check the order, confirm the return reason, and prepare the next step” |
| **Agent team** | One or more agents arranged to work together | One agent fetches logs; another analyzes them |
| **Observation** | The result returned by a tool | Database rows, search results, a message, or an error |
| **Team process** | The order in which the tasks run | Run tasks one after another in a sequential workflow |
| **Conversation context** | Information retained so the agent can understand follow-up questions | The order number discussed in the previous turn |

An agent is the worker; a task describes the job; a tool is the action it can use; and a team coordinates one or more workers. These parts are configured rather than appearing automatically.

### 4. Tools Select AI Agent can use

The exact tool set depends on the database release and configuration. Oracle documentation describes built-in tools such as:

- **SQL:** Generate and run SQL for approved database objects.
- **RAG:** Find relevant passages in a configured knowledge source and use them to help answer a question. Availability differs by release.
- **WEBSEARCH:** Search the web through the supported provider and configured credentials.
- **NOTIFICATION:** Send notifications through supported channels, such as email or Slack, when configured.
- **Custom tools:** Call a database PL/SQL function or another supported integration, such as an external REST service.

A tool is a capability the agent can use, not permission to do anything it wants. For example, a SQL tool should be limited to approved tables and safe operations. A notification tool should have clear rules about who can be contacted and when.

### 5. Example: checking an order return

Imagine an online store wants an assistant to help handle return requests.

1. **User request:** “I want to return order 125.”
2. **Agent:** Understands the request and identifies that it needs the order details and return reason.
3. **SQL tool:** Retrieves the order status from approved database tables.
4. **Agent:** Reads the returned rows and asks the user for any missing information.
5. **Task instructions:** Check the return rules and decide which supported step should happen next.
6. **Custom update tool:** If the rules allow it, call a controlled database function to update the return status.
7. **Observation:** The tool returns success or an error.
8. **Agent:** Explains the result to the user.

This example illustrates the design; it is not a ready-to-run implementation. A production system must verify the user's identity, confirm that the user is allowed to access that order, validate the return rules in code, and protect any action that changes data.

### 6. A simple mental model for the architecture

```text
User's request
      |
      v
Select AI Agent / Agent Team
      |
      v
Plan the next step
      |
      v
Choose an allowed tool
      |
      v
Run the tool against a database, knowledge source, or service
      |
      v
Read the observation (result or error)
      |
      +---- Need another step? ---- Yes ----> Plan again
      |
      No
      v
Return the final response
```

The AI model helps decide what to do. The Oracle framework coordinates the workflow. The tool performs the actual operation. The database and application security controls determine what the operation is allowed to access or change.

### 7. How do you create and run a Select AI Agent?

At a high level, the setup process is:

1. **Check prerequisites.** Confirm that your Oracle Database product and version support the features you need. Set up the required AI provider credentials, network access, and database privileges.
2. **Create an AI profile.** Configure the model/provider and limit the database objects that may be used.
3. **Create tools.** Configure the built-in tools you need and/or register approved custom functions.
4. **Create tasks.** Write clear instructions for each job. State the goal, required inputs, allowed tools, expected output, and restrictions.
5. **Create agents.** Give each agent a clear role and associate it with an AI profile.
6. **Create an agent team.** Assign tasks to agents and choose the supported execution process, such as sequential execution.
7. **Run the workflow.** Use the Select AI agent action in a supported SQL interface or call `DBMS_CLOUD_AI_AGENT.RUN_TEAM` from PL/SQL.
8. **Test and monitor.** Review tool calls, results, errors, and run history. Test both expected and unexpected user requests before releasing the workflow.

A team does not have to contain multiple agents. A team can coordinate a single agent, or multiple specialized agents. Use multiple agents only when separating responsibilities makes the workflow clearer or more manageable.

### 8. Conceptual PL/SQL example

The following shows the *shape* of an agent definition. It is not a complete deployment script; it assumes that the required grants, AI profile, credentials, and other prerequisites have already been configured. Check the current Oracle documentation for the exact attributes supported by your database version.

```sql
BEGIN
  DBMS_CLOUD_AI_AGENT.CREATE_AGENT(
    agent_name => 'ORDER_HELPER',
    attributes => '{
      "profile_name": "MY_AI_PROFILE",
      "role": "Help users check order status using only approved tools."
    }'
  );
END;
/
```

A task, tool, and team must also be configured before the workflow can perform the intended job. Creating an agent alone does not automatically connect it to every table or API.

To run a configured team from PL/SQL, Oracle documents the `DBMS_CLOUD_AI_AGENT.RUN_TEAM` interface. The required team name, prompt, optional parameters, and exact usage depend on the workflow and database version. Another supported option is to use the Select AI agent action in an appropriate SQL interface.

### 9. How to inspect what happened

Oracle provides history views that can help you understand an agent run:

- `USER_AI_AGENT_TEAM_HISTORY`: information about team runs.
- `USER_AI_AGENT_TASK_HISTORY`: information about tasks run by agents.
- `USER_AI_AGENT_TOOL_HISTORY`: information about tools used during tasks.

Use these views to investigate which tasks and tools ran and whether they succeeded. Check the official view definitions for the columns available in your release. Logs are useful for debugging and auditing, but they do not replace application-level security or testing.

### 10. Select AI vs Select AI Agent vs Agent Factory vs MCP

| Technology | Main purpose | Easy way to remember it |
|---|---|---|
| **Select AI** | Uses natural language for database-related AI actions, such as generating/running SQL, chat, and supported RAG workflows | “Ask the database a question in ordinary language” |
| **Select AI Agent** | Coordinates multi-step work by using agents, tasks, and tools | “Give the system a goal and let it work through approved steps” |
| **Oracle AI Database Private Agent Factory** | A separate platform for building, testing, and publishing enterprise agents and workflows with a visual builder and supported components | “Build and manage enterprise agents in a visual environment” |
| **Oracle AI Database MCP Server** | Exposes supported tools through the Model Context Protocol so compatible external clients can discover and call them | “Make approved capabilities available through a standard tool interface” |
| **Oracle AI Vector Search** | Stores and searches vector embeddings to find semantically similar content | “Find content by meaning, not only exact keywords” |

These technologies can work together, but they are not interchangeable. For example, a Select AI Agent tool may use SQL or RAG; configured tools may be exposed through an MCP Server; and Vector Search can support the retrieval part of a RAG workflow.

### 11. Security and reliability rules

- **Limit database access.** Use least-privilege accounts and expose only the tables, views, and operations required for the job.
- **Validate changes.** Do not rely on the model alone to decide whether a transaction, refund, email, or other important action is allowed.
- **Separate reading from writing.** Use read-only tools where possible. Put approval and validation checks around tools that change data or contact people.
- **Protect credentials.** Do not put API keys or passwords in prompts, examples, or source control.
- **Control external calls.** Network ACLs, credentials, private endpoints, and database privileges solve different security problems; configure each one that applies.
- **Test incorrect requests.** Test attempts to access another user's data, query unapproved tables, bypass rules, or trigger unwanted updates.
- **Monitor tool use.** Review run history, tool errors, cost, latency, and unexpected actions.
- **Check version support.** Select AI Agent capabilities and built-in tools vary by Oracle Database product and release. Do not assume a feature available in Autonomous AI Database 26ai is available in every Oracle Database version.

### 12. SQL/PLSQL and Python interfaces

Oracle documents Select AI Agent interfaces for SQL/PLSQL and Python. In the SQL/PLSQL route, the framework is managed using `DBMS_CLOUD_AI` and `DBMS_CLOUD_AI_AGENT`. The Python interface provides classes such as `Tool`, `Task`, `Agent`, and `Team` through the `select_ai.agent` module.

Choose the interface that fits your application:
- Use SQL/PLSQL when you want to define and run the workflow close to the database.
- Use Python when you want to define or coordinate the workflow from a Python application.
- Confirm current package installation, database compatibility, authentication, and API syntax in the official documentation before coding.

### Official Oracle references for this section

- [About Select AI](https://docs.oracle.com/en/database/oracle/oracle-database/26/selai/select-ai-about.html)
- [Select AI Agent Framework](https://docs.oracle.com/en/database/oracle/oracle-database/26/selai/select-ai-agent1.html)
- [Select AI Agent Framework concepts](https://docs.oracle.com/en/database/oracle/oracle-database/26/selai/select-ai-agent-concepts.html)
- [How to use Select AI Agent Framework](https://docs.oracle.com/en/database/oracle/oracle-database/26/selai/how-do-i-use-select-ai-agent.html)
- [Select AI Agent Framework interfaces (SQL and Python)](https://docs.oracle.com/en/database/oracle/oracle-database/26/selai/select-ai-agent-interfaces.html)
- [Examples using Select AI Agent Framework](https://docs.oracle.com/en/database/oracle/oracle-database/26/selai/examples-using-select-ai-agent1.html)
- [Select AI and Select AI Agent capability matrix](https://docs.oracle.com/en/database/oracle/oracle-database/26/saicm/index.html)

## Autonomous AI Database MCP Server

MCP (Model Context Protocol) is a standard way for an AI application to discover and call tools provided by another system. Oracle's Autonomous AI Database MCP Server exposes supported database capabilities through an MCP endpoint in supported environments.

- **MCP client:** the AI application that connects to the server.
- **MCP server:** the service that lists available tools and handles tool requests.
- **Select AI Agent framework:** one way to define and manage tools that may be exposed through the MCP server.
- **Database security:** controls what the authenticated user or service is allowed to access or execute.

A typical setup is to confirm database support, configure the MCP Server according to Oracle's instructions, choose approved tools, configure the client endpoint and authentication, and then test tool discovery and invocation.

Enabling MCP does not automatically give every client access to every tool or every database object. Authentication, tool exposure, database privileges, network controls, data policies, and auditing still matter.


---

# 8. Practical coding examples by topic

This section gives short examples first, followed by more complete patterns. “Complete” means the example demonstrates the core workflow; it does not mean it is ready to expose to real users without security review, tests, deployment configuration, and monitoring.

## 8.1 AI agent fundamentals: a tiny tool loop

The following example shows the basic idea without a model or framework. It is not an AI agent by itself; it demonstrates the *runtime loop* that a real agent framework manages.

```python
def get_balance(account_id: str) -> dict:
    # Fake data only. A real service must authenticate the caller and authorize access.
    demo_accounts = {"AC100": 125.50, "AC200": 80.00}
    if account_id not in demo_accounts:
        return {"ok": False, "error": "Account not found"}
    return {"ok": True, "balance": demo_accounts[account_id]}

tool_result = get_balance("AC100")
if tool_result["ok"]:
    print(f"Balance: ${tool_result['balance']:.2f}")
else:
    print(tool_result["error"])
```

**Why include this?** Before learning an agent framework, understand that a tool is ordinary code with inputs, outputs, and rules. The model can request a tool, but application code runs it.

For a real tool, validate the input, check the caller's identity and permissions, handle service failures, and avoid returning information the caller is not allowed to see.

## 8.2 MCP: a minimal server and client

MCP is a protocol. An MCP server exposes a tool; an MCP client connects to the server, discovers tools, and calls them. The host application decides which tools the model may use.

### Minimal MCP server

Install the official Python MCP SDK in a virtual environment. This example targets the current v2 SDK API (`MCPServer` and `Client`), which implements the 2026-07-28 protocol and can also interoperate with legacy peers:

```bash
python -m pip install "mcp"
```

Save as `policy_server.py`:

```python
from mcp.server import MCPServer

mcp = MCPServer("company-policy-demo")

POLICIES = {
    "annual leave": "Employees should submit annual-leave requests through the HR system.",
    "sick leave": "Employees should follow the company's sick-leave reporting process.",
    "expenses": "Employees should submit itemized receipts through the expense system.",
}

@mcp.tool()
def search_policy(topic: str) -> str:
    """Find a short demo policy summary by topic."""
    topic = topic.strip().lower()
    for key, value in POLICIES.items():
        if topic in key or key in topic:
            return value
    return "No matching demo policy was found."

if __name__ == "__main__":
    # For stdio transport, do not print ordinary logs to stdout.
    mcp.run(transport="stdio")
```

This server uses fake, hard-coded policy summaries. It does not read company documents or check employee permissions.

### Minimal MCP client

Save as `policy_client.py` beside the server file:

```python
import asyncio
import sys
from pathlib import Path

from mcp import Client, StdioServerParameters

async def main() -> None:
    server_file = Path(__file__).with_name("policy_server.py")
    server = StdioServerParameters(
        command=sys.executable,
        args=[str(server_file)],
    )

    # The current SDK Client manages protocol discovery/version negotiation.
    async with Client(server) as client:
        tool_list = await client.list_tools()
        print("Available tools:", [tool.name for tool in tool_list.tools])

        result = await client.call_tool(
            "search_policy",
            {"topic": "annual leave"},
        )
        for item in result.content:
            if getattr(item, "text", None):
                print(item.text)

if __name__ == "__main__":
    asyncio.run(main())
```

Run the client:

```bash
python policy_client.py
```

Expected result: the tool list includes `search_policy`, and the result contains the demo annual-leave summary. The exact object representation can differ by SDK version.

### More complete MCP pattern

In a real application:
1. Keep the server process and client lifecycle controlled.
2. Add structured input validation and clear error results.
3. Authenticate and authorize any access to documents, databases, or APIs.
4. Pin and review package versions.
5. For remote HTTP transport, use the current SDK's Streamable HTTP support, authentication, TLS, and safe network configuration.
6. Treat tool output as data, not as instructions that override the user's or application's rules.

MCP server tool descriptions and returned content can be untrusted. A document might contain text such as “ignore previous rules”; the host must not treat that text as trusted system instructions.

## 8.3 OpenAI Agents SDK: a tool-backed agent

Install the package:

```bash
python -m pip install -U openai-agents
```

Set `OPENAI_API_KEY` in your environment. Save as `openai_policy_agent.py`:

```python
import os
from agents import Agent, Runner, function_tool

@function_tool
def search_policy(topic: str) -> str:
    """Return a demo policy summary for the supplied topic."""
    policies = {
        "annual leave": "Submit annual-leave requests through the HR system.",
        "expenses": "Submit itemized receipts through the expense system.",
    }
    return policies.get(topic.strip().lower(), "No matching demo policy was found.")

agent = Agent(
    name="Policy assistant",
    instructions=(
        "Answer employee policy questions in plain English. "
        "Use the search_policy tool when a policy detail is needed. "
        "If the tool does not find an answer, say that you could not find it."
    ),
    tools=[search_policy],
    model=os.environ.get("OPENAI_MODEL", "gpt-5.5"),
)

if __name__ == "__main__":
    result = Runner.run_sync(
        agent,
        "How should I submit an annual-leave request?",
    )
    print(result.final_output)
```

**What to notice:**
- `Agent` defines the agent's name, instructions, and tools.
- `@function_tool` exposes the Python function to the agent.
- `Runner.run_sync` runs the workflow and manages the agent loop.
- `final_output` contains the final response.

This is a local demo. It does not connect to an HR system or authenticate an employee. For asynchronous applications, use the SDK's async runner. For the current supported API, follow the official SDK quickstart.

## 8.4 Oracle Select AI: natural language to SQL

Select AI lets an authorized user ask a database question in ordinary language. The exact profile setup depends on the provider, database release, credentials, and objects the user is allowed to query.

A common interaction shape is:

```sql
-- Example only: MY_AI_PROFILE must already exist and be configured.
-- Set the active profile for this database session first.
EXEC DBMS_CLOUD_AI.SET_PROFILE('MY_AI_PROFILE');

-- Ask Select AI to show the SQL it generates; this does not execute that SQL.
SELECT AI SHOWSQL how many orders were placed last month;
```

The intent is to inspect the SQL generated for the natural-language request before executing it. Select AI command syntax and supported actions can differ by Oracle Database product and version, so confirm the exact syntax in the matching documentation.

A typical workflow is:
1. Configure an AI profile with the model/provider and approved database objects.
2. Ask a question using a supported Select AI action.
3. Inspect generated SQL where appropriate.
4. Run the query only with appropriate privileges.
5. Validate the result and explain it in plain English.

**Important:** generated SQL is not automatically correct. Review access to sensitive columns and tables, limit the profile's object list, and test queries that could return personal or confidential information.

## 8.5 Oracle Select AI Agent: a conceptual agent setup

Oracle's Select AI Agent framework supports agents, tasks, tools, and teams. The following pattern is based on the official Oracle SQL/PLSQL interface, but it is only the agent-creation step—not a complete working team.

```sql
BEGIN
  DBMS_CLOUD_AI_AGENT.CREATE_AGENT(
    agent_name => 'POLICY_HELPER',
    attributes => '{
      "profile_name": "MY_AI_PROFILE",
      "role": "Help users find approved policy information. Do not invent policy rules."
    }'
  );
END;
/
```

Before running it, configure the required AI profile and grants, and check that the database version supports the required Select AI Agent features.

A complete Select AI Agent workflow also needs:
- a tool, such as a permitted SQL tool or custom PL/SQL function;
- a task that explains when and how to use the tool;
- a team that assigns the task to the agent;
- a run through a supported Select AI Agent interface;
- validation, auditing, and permissions for every operation.

A practical learning path is to first create an agent using Oracle's official example, then create one tool, then one task, then one team. Test each stage before adding another capability. Avoid enabling write operations until the read-only workflow behaves as expected.

Official Oracle examples: [Select AI Agent examples](https://docs.oracle.com/en/database/oracle/oracle-database/26/selai/examples-using-select-ai-agent1.html).

## 8.6 OCI Generative AI: Responses API connection pattern

OCI Generative AI offers agent-building features and an OpenAI-compatible Responses API in supported configurations. The model endpoint, authentication method, region, and access policies must match your OCI tenancy and current service setup.

The official quick start documents an endpoint pattern similar to:

```text
https://inference.generativeai.<region>.oci.oraclecloud.com/openai/v1
```

A connection pattern using the OpenAI Python SDK is:

```python
import os
from openai import OpenAI

client = OpenAI(
    base_url=os.environ["OCI_RESPONSES_BASE_URL"],
    api_key=os.environ["OCI_RESPONSES_API_KEY"],
    project=os.environ["OCI_GENERATIVE_AI_PROJECT_OCID"],
)

response = client.responses.create(
    model=os.environ["OCI_MODEL_OR_DEPLOYMENT"],
    input="Explain the difference between an AI agent and a tool in simple English.",
)

print(response.output_text)
```

This example follows the OCI Generative AI API-key pattern and includes the required project OCID. Set `OCI_RESPONSES_BASE_URL` to the region-specific endpoint, `OCI_RESPONSES_API_KEY` to an OCI Generative AI API key, `OCI_GENERATIVE_AI_PROJECT_OCID` to the project OCID, and `OCI_MODEL_OR_DEPLOYMENT` to a model ID supported in that region. If your tenancy uses OCI IAM signing, use the official `oci-genai-auth` helper and the matching documented example instead. Do not assume that a public OpenAI API key works against OCI.

**Setup checklist:**
1. Enable access to the required OCI Generative AI service and model.
2. Choose the correct region and endpoint.
3. Configure the documented authentication method and IAM policies.
4. Set the environment variables locally; do not commit secrets.
5. Test a simple model response before adding tools or state.
6. Add logging, cost limits, error handling, and access controls.

OCI Generative AI Agents, OCI's OpenAI-compatible Responses API, and Oracle Database Select AI Agent are related parts of Oracle's AI ecosystem, but they are different services and are configured differently.

Official references: [OCI Generative AI Agents](https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/) and [OCI Responses API quick start](https://docs.oracle.com/en-us/iaas/Content/generative-ai/get-started-agents.htm).

## 8.7 Oracle AI Vector Search and RAG: the data flow

RAG means **retrieval-augmented generation**. The application first finds relevant source material and then asks the model to answer using that material.

The usual steps are:
1. Extract text from approved documents.
2. Split long text into smaller chunks.
3. Create an embedding for each chunk. An embedding is a numeric representation of meaning.
4. Store the chunk, its metadata, and its embedding.
5. Embed the user's question.
6. Search for similar chunks.
7. Send the most relevant chunks to the model with the question.
8. Return an answer with references to the source material when possible.

Conceptual SQL for vector similarity (the table, column names, vector bind, and distance metric must match your schema and embedding model):

```sql
SELECT document_id, chunk_text
FROM policy_chunks
ORDER BY VECTOR_DISTANCE(embedding, :query_embedding, COSINE)
FETCH FIRST 5 ROWS ONLY;
```

This ranks chunks by cosine distance in this example. Smaller distance generally means closer vectors for this metric, but distance values are not automatically calibrated probabilities. The query embedding must use the same compatible model/dimension as the stored embeddings.

### Simple Python example of the retrieval idea

This is a *toy example* using keyword overlap—not a real embedding or vector-search implementation. It helps explain the retrieve-then-answer sequence.

```python
documents = [
    {"id": "P1", "text": "Annual leave requests must be submitted in the HR portal."},
    {"id": "P2", "text": "Expense claims require itemized receipts."},
    {"id": "P3", "text": "Sick leave must be reported to the manager."},
]

def retrieve(query: str, limit: int = 2) -> list[dict]:
    words = set(query.lower().split())
    scored = []
    for doc in documents:
        overlap = len(words & set(doc["text"].lower().split()))
        scored.append((overlap, doc))
    scored.sort(key=lambda item: item[0], reverse=True)
    return [doc for score, doc in scored[:limit] if score > 0]

question = "How do I request annual leave?"
matches = retrieve(question)
context = "\n".join(doc["text"] for doc in matches)

print("Retrieved context:")
print(context)
print("\nUse this context as evidence when generating an answer.")
```

Real RAG uses robust chunking, embeddings, vector similarity, optional keyword/hybrid search, reranking, metadata filters, source citations, and evaluation. The toy example is not a substitute for Oracle AI Vector Search.

---

# 9. Troubleshooting guide

Start with the complete error message and identify which layer failed: package/import, authentication, model request, tool execution, MCP transport, database permissions, or retrieval. Do not immediately change several dependencies at once.

| Symptom | Common cause | What to check |
|---|---|---|
| `ModuleNotFoundError` | Package installed into a different Python environment | Run `python -c "import sys; print(sys.executable)"` and install using that interpreter |
| API key missing or unauthorized | Environment variable not set, wrong credential, or missing access policy | Check the variable name, endpoint, region, and provider's authentication guide; never print the secret |
| Model/deployment not found | Wrong model ID, deployment name, region, or unavailable model | Copy the exact identifier from the provider console/docs and check access/quota |
| Agent never calls a tool | Tool description is vague, tool not registered, or model decides it is unnecessary | Improve the description, test a direct function call, and inspect traces/messages |
| Tool call fails | Invalid arguments, service error, missing permissions, timeout | Validate inputs, test the function separately, and return safe, structured errors |
| LangChain `invoke()` fails | Provider integration mismatch, invalid model name, missing key, or recursion limit | Check package versions, provider docs, environment, and graph configuration |
| MCP server will not start | Wrong command/path, dependency missing, or server logs written to stdout | Run the server directly, check stderr, confirm the interpreter and package, keep stdout protocol-only |
| MCP tool list is empty | Server did not register tools, client did not initialize, or wrong server/transport | Check `initialize`, `list_tools`, server logs, and endpoint/command |
| MCP HTTP connection rejected | Wrong endpoint, authentication, TLS, Origin, firewall, or incompatible transport | Check current MCP transport docs and server configuration |
| Oracle Select AI reports profile errors | Profile missing, malformed attributes, credentials or provider setup incorrect | Verify profile name, JSON attributes, credentials, privileges, and supported provider |
| Oracle SQL tool returns wrong data | Wrong object list, generated SQL is invalid, or source schema differs | Inspect generated SQL, narrow the profile object list, and test with ordinary SQL |
| Select AI Agent cannot run a task | Missing grants, agent/task/tool/team not configured or enabled | Check each object and required privileges in order; inspect history views |
| Vector search returns poor results | Bad chunking, mismatched embedding models, wrong distance metric, no metadata filters | Inspect chunks and embedding dimensions, test retrieval independently, compare relevant results |
| RAG answer invents facts | Retrieved context is missing or ignored | Return “not found” when evidence is absent, cite sources, improve retrieval and evaluate |
| Output is slow or costly | Too many model calls, huge context, parallel tools, repeated retries | Set budgets/timeouts, reduce context, cache safe results, and trace the run |
| Version-specific example fails | Docs or code target a different release | Record package/database version and use the matching official reference; avoid blind downgrades |

### A repeatable debugging process

1. Reproduce the problem with the smallest possible example.
2. Record the Python package versions or database release.
3. Test credentials and connectivity independently of the agent.
4. Test the tool function independently of the model.
5. Inspect the model request, tool call, tool result, and final answer separately.
6. Add one feature at a time.
7. Write a regression test after fixing the issue.

For database examples, use a non-production schema until the workflow and permissions have been reviewed.

---

# 10. End-to-end learning project: Employee Policy Assistant

## Project goal

Build a small assistant that answers employee questions from approved policy information. It should explain its answer in plain English, use a tool to retrieve evidence, and admit when the information is not available.

This project is deliberately scoped as a learning exercise. It does **not** connect to a real HR system or contain real employee records.

## Requirements

- Accept a question such as “How do I request annual leave?”
- Retrieve a relevant policy excerpt.
- Answer using only the retrieved excerpt.
- Say when the policy source has no answer.
- Keep retrieval logic separate from the agent framework.
- Include tests for known and unknown questions.
- Do not implement leave-balance lookup or employee-specific access until authentication and authorization are designed.

## Suggested project layout

```text
policy-assistant/
├── policy_data.py       # demo policy content
├── policy_tools.py      # retrieval function/tool
├── app.py               # chosen agent framework
├── test_policy_tools.py # automated tests
├── .env.example         # variable names only; no real secrets
└── README.md            # setup and run steps
```

## Step 1: Create the policy data and retrieval function

Save as `policy_tools.py`:

```python
POLICIES = [
    {
        "id": "LEAVE-001",
        "title": "Annual leave",
        "text": "Submit annual-leave requests through the HR portal. Your manager must approve the request.",
    },
    {
        "id": "EXP-001",
        "title": "Expense claims",
        "text": "Submit expense claims with itemized receipts through the expense system.",
    },
    {
        "id": "SICK-001",
        "title": "Sick leave",
        "text": "Follow the company's sick-leave reporting process and inform your manager as required by policy.",
    },
]

def search_policy(query: str) -> str:
    """Return matching demo policy excerpts, or an explicit no-match message."""
    query_words = set(query.lower().replace("?", "").split())
    matches = []

    for policy in POLICIES:
        searchable = f"{policy['title']} {policy['text']}".lower()
        score = sum(1 for word in query_words if word in searchable)
        if score:
            matches.append((score, policy))

    matches.sort(key=lambda item: item[0], reverse=True)
    if not matches:
        return "NO_MATCH: No matching policy excerpt was found in the demo source."

    return "\n\n".join(
        f"Source: {policy['id']} — {policy['title']}\n{policy['text']}"
        for _, policy in matches[:2]
    )
```

This uses keyword overlap to keep the first version simple. It is not semantic search, and it may miss paraphrases. A later version can replace this function with Oracle AI Vector Search or another approved retrieval service without changing the overall responsibility of the tool.

## Step 2: Test retrieval before adding an LLM

Save as `test_policy_tools.py`:

```python
import unittest
from policy_tools import search_policy

class PolicySearchTests(unittest.TestCase):
    def test_annual_leave_returns_policy(self):
        result = search_policy("annual leave request")
        self.assertIn("LEAVE-001", result)
        self.assertIn("HR portal", result)

    def test_unknown_topic_returns_no_match(self):
        result = search_policy("company helicopter policy")
        self.assertIn("NO_MATCH", result)

if __name__ == "__main__":
    unittest.main()
```

Run:

```bash
python -m unittest -v
```

Both tests should pass. If retrieval fails, fix it before connecting an LLM. This separates retrieval bugs from model behavior.

## Step 3: Connect it to the OpenAI Agents SDK

Install `openai-agents`, set `OPENAI_API_KEY`, and save the following as `app.py`:

```python
from agents import Agent, Runner, function_tool
from policy_tools import search_policy as search_policy_data

@function_tool
def find_policy(query: str) -> str:
    """Search the approved demo policy source for excerpts related to a question."""
    return search_policy_data(query)

policy_agent = Agent(
    name="Employee Policy Assistant",
    instructions=(
        "Answer policy questions in plain English. Use find_policy to retrieve evidence. "
        "Base policy statements only on the returned excerpts. Include the source ID when "
        "available. If the tool returns NO_MATCH, clearly say the demo source did not "
        "contain an answer. Do not invent policy details or claim to check an employee's "
        "personal records."
    ),
    tools=[find_policy],
)

if __name__ == "__main__":
    question = input("Ask a policy question: ").strip()
    if not question:
        raise SystemExit("Please enter a question.")

    result = Runner.run_sync(policy_agent, question)
    print("\nAnswer:")
    print(result.final_output)
```

Run:

```bash
python app.py
```

Try:
- `How do I request annual leave?`
- `What do I need for an expense claim?`
- `What is the company's helicopter policy?`

Expected behavior: answers should use retrieved text, and the unsupported question should produce an explicit no-match response. Model wording may vary, so tests should verify behavior and evidence rather than require one exact sentence.

## Step 4: Add safety and quality checks

Before using real documents:
- Add document source, owner, version, and effective date metadata.
- Restrict retrieval by the caller's identity and allowed document scope.
- Remove or mask confidential content before logging.
- Keep retrieval output separate from trusted system instructions.
- Add tests for prompt injection in documents, ambiguous questions, no matches, and conflicting policies.
- Track answer quality, retrieval quality, latency, cost, and tool errors.
- Add a human escalation path for policy questions that affect pay, legal rights, or employment decisions.

## Step 5: Extend the project

Choose one extension at a time:
1. Replace keyword search with embeddings and Oracle AI Vector Search.
2. Expose `search_policy` through an MCP server.
3. Switch the orchestration layer to LangChain `create_agent`.
4. Deploy the approved agent workflow in an OCI environment.
5. Add document versioning, citations, and a repeatable evaluation set.

Do not build all integrations at once. Keep the retrieval interface stable and test each extension independently.

---

# 11. Practice exercises and interview questions

## Exercises

1. **Tool calling:** Write a Python function that validates an order ID and returns a structured result. Add tests for valid, invalid, and missing IDs.
2. **LangChain:** Add a second read-only tool to the agent. Explain when the model should choose each tool.
3. **MCP:** Add a second tool to `policy_server.py`. Verify that it appears in `list_tools()` and call it through the client.
4. **OpenAI Agents SDK:** Change the agent instructions so it refuses to answer when the retrieval tool returns `NO_MATCH`. Test this with supported and unsupported questions.
5. **RAG:** Explain why keyword overlap is not semantic search. Describe how embeddings and vector distance change the retrieval process.
6. **Oracle Select AI:** Describe how you would limit the objects available to a natural-language SQL profile and how you would inspect generated SQL.
7. **Oracle Select AI Agent:** Sketch an agent, a task, a tool, and a team for log analysis. State which steps are read-only and which require approval.
8. **OCI:** List the region, endpoint, model identifier, authentication method, and IAM permissions you would verify before debugging a failed model call.
9. **Evaluation:** Create five supported and five unsupported policy questions. Record whether retrieval returned useful evidence and whether the final answer was grounded.
10. **Security:** Describe how a malicious instruction hidden inside a retrieved document could affect an agent and what controls would reduce the risk.

## Interview questions with short answers

**1. What is the difference between an LLM and an AI agent?**  
An LLM generates outputs. An agent adds instructions, tools, state, and a runtime loop so it can work toward a goal over multiple steps.

**2. What is the difference between function calling and MCP?**  
Function calling is a model interface for requesting structured actions. MCP is a protocol for discovering and invoking tools offered by a server. They can be used together.

**3. What does LangChain `invoke()` do?**  
It runs the configured agent graph for the supplied input. The run may include several model calls and tool executions.

**4. Does RAG prevent hallucinations?**  
No. It can provide relevant evidence, but retrieval may be wrong or incomplete and the model may still misinterpret it.

**5. What is the difference between Select AI and Select AI Agent?**  
Select AI supports natural-language database interactions such as SQL generation and supported RAG workflows. Select AI Agent coordinates multi-step workflows using configured agents, tasks, and tools.

**6. Why is least privilege important for agents?**  
The model can select an inappropriate tool or produce incorrect arguments. Restricting tool permissions limits the damage a mistaken action can cause.

**7. How would you debug an agent that gives incorrect answers?**  
Check the retrieved evidence, tool arguments and results, prompt/instructions, model response, and evaluation tests separately.

**8. How should high-impact actions be handled?**  
Validate business rules in code, authorize the user, restrict tool permissions, use idempotency where possible, and require human confirmation when appropriate.

**9. What is an observation in a ReAct-style agent loop?**  
It is the result of an action or tool call, such as rows returned from a database, a search result, or an error.

**10. What is the difference between an agent and a team?**  
An agent has a role and available tools. A team coordinates tasks assigned to one or more agents according to a workflow.

---

# 12. Quick glossary

| Term | Plain-English meaning |
|---|---|
| Agent | A model-driven worker that can use tools and follow a workflow |
| Agent runtime/orchestrator | Software that manages model calls, tools, state, and stopping conditions |
| Tool | A function or service an agent is allowed to call |
| Tool schema | A description of a tool's name, inputs, and expected input types |
| Observation | A result returned after an action |
| MCP host | The AI application that manages the model and the user's interaction |
| MCP client | The component that connects an application to an MCP server |
| MCP server | A service that exposes tools, resources, or prompts |
| Function calling | A structured request from a model asking an application to run a function |
| RAG | Finding relevant source material and using it to help generate an answer |
| Embedding | A list of numbers representing aspects of the meaning of text or other data |
| Vector search | Searching for data whose embeddings are close to a query embedding |
| Reranking | Reordering retrieved results using a second relevance-scoring step |
| Guardrail | A check that blocks or flags disallowed inputs, actions, or outputs |
| Least privilege | Giving a tool or user only the permissions required for the task |
| Trace | A record of the steps, model calls, tool calls, and results in a workflow |
| Evaluation set | A repeatable set of questions and expected behavior used to measure quality |
| OCI IAM | Oracle Cloud Identity and Access Management, which controls identity and permissions in OCI |
| AI profile | Oracle database configuration connecting Select AI features to an AI provider and related settings |
| Select AI Agent task | A defined job with instructions, inputs, and permitted tools |
| Agent team | A configured workflow that coordinates one or more agents and tasks |


---

# 13. High-priority distinctions

| Question | Answer |
|---|---|
| Does an LLM execute a function call itself? | No. It returns a request; the runtime executes the function. |
| What does LangChain `invoke()` do? | Runs the agent graph, potentially including multiple model calls and tool executions. |
| Does `invoke()` automatically preserve memory between runs? | No. Persistent state needs a supported checkpointer/thread/session or supplied history. |
| What does MCP standardize? | Discovery and invocation of tools/resources/prompts between hosts/clients and servers. |
| Is MCP the same as function calling? | No. They complement each other. |
| Does Responses API always execute application-defined functions? | No. The application usually executes them; hosted tools are a separate case. |
| Does Agents SDK own the loop? | The Runner manages the agent loop and supported tool/handoff flow. |
| Does RAG guarantee correct answers? | No. Retrieval and generation both need evaluation. |
| Is Vector Search a separate database product? | No. It is integrated into Oracle Database. |
| Is Select AI the same as Select AI Agent? | No. Select AI supports natural-language AI actions; Select AI Agent coordinates multi-step tasks using configured agents and tools. |
| Is Select AI Agent the same as Agent Factory? | No. Select AI Agent is a database agent framework; Agent Factory is a separate platform for building and publishing enterprise agents and workflows. |
| Does creating an agent automatically grant database access? | No. Access depends on configured tools, credentials, database privileges, and application rules. |
| Are all embedding vectors 1024-dimensional? | No. Dimensions depend on the model. |
| Are rerank scores always probabilities from 0 to 1? | No. Scores are model-specific. |
| Does a private endpoint replace IAM? | No. Network reachability and identity authorization are different controls. |
| Does enabling MCP authorize every database user to every tool? | No. Tool exposure, authentication, privileges, network controls, and data policies still apply. |
| Does “no-code” mean no engineering responsibility? | No. Integrations, permissions, evaluation, monitoring, and maintenance still matter. |

---



---

# Official documentation and further reading

- [LangChain agents](https://docs.langchain.com/oss/python/langchain/agents)
- [LangChain tools](https://docs.langchain.com/oss/python/langchain/tools)
- [LangChain OpenAI integration](https://docs.langchain.com/oss/python/integrations/chat/openai)
- [MCP Python SDK — current v2 documentation](https://py.sdk.modelcontextprotocol.io/)
- [MCP Python SDK — what's new in v2](https://py.sdk.modelcontextprotocol.io/whats-new/)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [MCP server tools specification](https://modelcontextprotocol.io/specification/2026-07-28/server/tools)
- [OpenAI Agents SDK quickstart](https://developers.openai.com/api/docs/guides/agents/quickstart)
- [OCI Generative AI Agents](https://docs.oracle.com/en-us/iaas/Content/generative-ai-agents/)
- [OCI Responses API quick start](https://docs.oracle.com/en-us/iaas/Content/generative-ai/get-started-agents.htm)
- [LangChain `create_agent` reference](https://reference.langchain.com/python/langchain/agents/factory/create_agent)
- [MCP current specification](https://modelcontextprotocol.io/specification/2026-07-28)
- [MCP specification — tools, 2025-06-18 version](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)
- [MCP specification — transports, 2025-06-18 version](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports)
- [OpenAI Responses API](https://platform.openai.com/docs/guides/responses)
- [OpenAI Agents SDK](https://platform.openai.com/docs/guides/agents)
- [OpenAI Assistants API migration guide](https://platform.openai.com/docs/assistants/migration)
- [Oracle AI Database Private Agent Factory](https://docs.oracle.com/en/database/oracle/agent-factory/26.7/paias/introduction.html)
- [Oracle Select AI overview](https://docs.oracle.com/en/database/oracle/oracle-database/26/selai/select-ai-about.html)
- [Oracle Select AI Agent Framework](https://docs.oracle.com/en/database/oracle/oracle-database/26/selai/select-ai-agent1.html)
- [Oracle Select AI Agent concepts](https://docs.oracle.com/en/database/oracle/oracle-database/26/selai/select-ai-agent-concepts.html)
- [Oracle Select AI Agent examples](https://docs.oracle.com/en/database/oracle/oracle-database/26/selai/examples-using-select-ai-agent1.html)
- [Oracle AI Vector Search — `VECTOR_DISTANCE`](https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/vector_distance.html)
- [Autonomous AI Database MCP Server](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/mcp-server.html)
- [Oracle University Labs — Agentic AI Foundations](https://github.com/oracle-university-labs/ou-generativeai-pro/tree/main/agentic-ai-foundations)

## Recommended study order
1. Agent fundamentals, tool calling, and safety.
2. LangChain `create_agent` and what `invoke()` executes.
3. MCP architecture, JSON-RPC, transports, and tool security.
4. Responses API vs Agents SDK; function tools vs hosted tools.
5. OCI agent architecture, deployment modes, and memory terminology.
6. Oracle Vector Search, chunking, embeddings, distance functions, and indexes.
7. Agent Factory Knowledge/Data Analysis agents and MCP/Select AI Agent.
8. Revisit official docs before using code or memorizing version-specific behavior.
