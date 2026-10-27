# AI Agents, LangChain, MCP, OpenAI Agents, and Oracle AI Database


This guide consolidates the uploaded notes into a cleaner reference. It preserves the main concepts, corrects misleading statements, and flags details that depend on framework/API version. Check official documentation before copying code into a project.

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

```python
from langchain.agents import create_agent

def get_weather(city: str) -> str:
    # Replace this demo with a real, authenticated weather provider.
    return f"Weather data for {city} is not connected in this example."

agent = create_agent(
    model="provider:model-name",
    tools=[get_weather],
    system_prompt="You are a helpful assistant.",
)

result = agent.invoke({
    "messages": [{"role": "user", "content": "What is the weather in Paris?"}]
})
```

The model string and provider integration are placeholders. Configure the correct provider package and credentials.

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

**Version warning:** MCP evolves. Newer protocol revisions may change initialization, headers, session handling, or transport details. The sequence above describes the 2025-06-18 style used in the source sample, not every later revision. Pin the protocol/SDK version and use its matching official specification.

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

## Autonomous AI Database MCP Server and Select AI Agent
MCP standardizes how a compatible AI client discovers and calls tools. Oracle's managed Autonomous AI Database MCP Server exposes approved database tools through an MCP endpoint for supported environments.

- **MCP client:** the AI application.
- **MCP server:** exposes tools through MCP.
- **Select AI Agent framework:** defines/manages built-in or custom tools that can be exposed.
- **Database security:** governs what the authenticated identity can access and execute.

Typical setup:
1. Confirm database release and service availability.
2. Enable/configure the MCP Server according to Oracle instructions.
3. Select or define approved tools.
4. Configure the client endpoint and supported authentication.
5. Discover and invoke tools.
6. Ensure database permissions and audit controls are effective.

Enabling MCP does not give every client access to all tools. Authentication depends on supported configuration. Database roles, network ACLs/private endpoints, VPD policies, and auditing have different purposes.

---

# 8. High-priority distinctions

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
| Are all embedding vectors 1024-dimensional? | No. Dimensions depend on the model. |
| Are rerank scores always probabilities from 0 to 1? | No. Scores are model-specific. |
| Does a private endpoint replace IAM? | No. Network reachability and identity authorization are different controls. |
| Does enabling MCP authorize every database user to every tool? | No. Tool exposure, authentication, privileges, network controls, and data policies still apply. |
| Does “no-code” mean no engineering responsibility? | No. Integrations, permissions, evaluation, monitoring, and maintenance still matter. |

---

---

# Official documentation

- [LangChain agents](https://docs.langchain.com/oss/python/langchain/agents)
- [LangChain `create_agent` reference](https://reference.langchain.com/python/langchain/agents/factory/create_agent)
- [MCP specification — tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)
- [MCP specification — transports](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports)
- [OpenAI Responses API](https://platform.openai.com/docs/guides/responses)
- [OpenAI Agents SDK](https://platform.openai.com/docs/guides/agents)
- [OpenAI Assistants API migration guide](https://platform.openai.com/docs/assistants/migration)
- [Oracle AI Database Private Agent Factory](https://docs.oracle.com/en/database/oracle/agent-factory/26.7/paias/introduction.html)
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
9. https://github.com/oracle-university-labs/ou-generativeai-pro/tree/main/agentic-ai-foundations 
