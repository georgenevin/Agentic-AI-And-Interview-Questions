# AI Agents, LangChain, MCP and the OpenAI Agent Stack: Reviewed Study Notes

**Contents**
- Part 1: AI agent fundamentals
- Part 2: LangChain agent and what `invoke()` does
- Part 3: Sample conversation JSON (one full tool-calling loop)
- Part 4: Model Context Protocol (MCP), including a full JSON-RPC session sample
- Part 5: OpenAI agent stack (Responses API, Agents SDK), including sample flows (function call, handoff, tripwire)

---

# Part 1: AI agent fundamentals

## 1.1 What is an AI agent?

**Agent = LLM + Tools + Memory + Loop**, wrapped in an orchestration layer.

| Part | Role | Analogy |
|---|---|---|
| LLM | Understands, plans, decides | Brain |
| Tools | Read data and take actions in the outside world | Hands |
| Loop (orchestrator) | Runs the cycle, routes calls, manages state | Nervous system |

**Key properties**
- **Goal-directed**: works toward an objective, not just answering one prompt
- **Autonomous**: decides the next step dynamically
- **Iterative**: runs in a loop until done or stopped
- **Self-correcting**: observes results and retries on error
- **Stateful**: keeps context across steps
- **Knowledge** = training data + live tools
- **Memory** = working (context window) + persistent (external store, e.g. vector DB or database)
- **Planning**: multi-step planning and iteration
- **Actions**: search, compute, call APIs, write code

**Cycle:** Perceive → Reason → Act → Observe → repeat

- **Perceive**: receive input or observation
- **Reason**: choose the next step
- **Act**: call a tool or respond to the user
- **Observe**: read the tool result or feedback

**What can go wrong:** infinite loops, hallucinated tool calls or arguments, cost explosion, context overflow, compounding errors over long chains, goal drift.

**Mitigations:** max iterations, token/cost budget, timeouts, loop detection, validated tool schemas.

---

## 1.2 Components

### LLM (decision-maker)
- Understands user intent, plans multi-step work, picks the tool and its arguments, interprets results, writes the final answer
- Can be one or several models of any size, prompted with CoT and ReAct strategies
- The same LLM can power different agents just by changing tools and prompts (no retraining)
- Needs: reliable structured output / native tool calling, strong reasoning, large context window
- Each iteration makes **at least 1 LLM call**, so there is a capability vs cost vs latency tradeoff
- **Two-tier pattern:** cheap model for routing, capable model for hard reasoning
- Newer reasoning models have built-in chain-of-thought training, which can improve agent reasoning

### Tools (bridge to the real world)
- APIs, database queries, search, code execution, etc.
- Enable real-time data access and real-world actions
- Turn an LLM from a text generator into an actor

**Tool-calling flow**
1. Developer defines tools as JSON schemas (name, description, parameters)
2. Agent sends the schemas + user query to the LLM
3. LLM decides: plain text answer **or** a tool call (name + arguments)
4. **Your code** executes the tool. The LLM never runs it.
5. Result goes back to the LLM as an observation

### Loop / orchestrator
- Runs reason → act → observe; decides whether to continue or stop
- **Reasoning strategy**: applies CoT, ReAct, or another prompting framework to break the goal into steps
- **Memory management**: short-term scratchpad for the session; long-term store for history and documents
- **Tool routing**: select the tool, format the call, handle errors, feed results back
- **State machine**: tracks position, completed steps, loops, and failures
- **Safety and guardrails**: hard-coded policy rules that override model reasoning when needed

---

## 1.3 Reasoning patterns

| Pattern | How it works | Best for |
|---|---|---|
| **Chain of Thought (CoT)** | Step-by-step intermediate reasoning before the answer | Math, logic, step-by-step analysis |
| **ReAct** | Interleaves Thought → Action → Observation | Tool selection, multi-step tasks, external API calls |
| **Tree of Thoughts (ToT)** | Explores multiple branches, evaluates each, picks one (costly: many LLM calls) | Creative tasks, strategic planning, exploration |

### CoT details
- Prompting technique: instruct the LLM to reason step by step before answering
- **Zero-shot:** add "think step by step" to the prompt
- **Few-shot:** provide worked examples showing the reasoning
- Benefits: better accuracy on complex tasks, easier debugging
- Caveats: the trace is not guaranteed to be a faithful explanation of why the model answered; modern reasoning models already do this internally, so explicit prompting is often unnecessary
- **Limitation (CoT alone):** uses only internal knowledge, so wrong knowledge gives wrong reasoning. It cannot verify against reality or act in the real world. This is the gap ReAct fills.

### ReAct details
- Uses intermediate reasoning (Thought) to decide when and how to act (Action), then reads the result (Observation)
- Pros:
  - Transparent, debuggable reasoning traces
  - Tool calls ground reasoning in real data, which **reduces** (does not eliminate) hallucination
  - Prevents the model from making up answers when it should look things up
- Typical prompt: *"Think step by step. If you need information, call a tool. When the final answer is ready, present it."*

---

## 1.4 Tool definition essentials (e.g. LangChain)

- `@tool` wraps a function so the framework can register it as a callable tool (decorator name varies by framework)
- **Type hints** (e.g. `a: float`) define parameter types in the schema
- **Docstring** becomes the tool description, which is how the LLM decides when and how to use the tool. **Vague docstring leads to wrong tool choice.**
- The framework converts the function into a JSON schema that is sent to the LLM with the user message. The LLM sees the schema and decides whether a tool call is needed.
- The **return value** is not part of what the LLM sees up front; it comes back as the observation after execution.

---

## 1.5 Safety and guardrails

### Threats
1. **Prompt injection**: hijacking the agent via crafted input
   - *Direct*: the user types it
   - *Indirect*: hidden in web pages, documents, emails, or tool output (the bigger risk for agents)
2. **Tool misuse**: agent calls a tool with wrong or dangerous arguments
3. **Memory poisoning**: malicious content stored in memory affects future behavior, including for other users
4. **Data exfiltration**: agent tricked into leaking sensitive internal data through tool calls or responses
5. **Runaway execution**: endless loops or excessive API calls cause cost overruns and system strain

### Defense layers
1. **Input validation**
   - Treat all external or retrieved content as untrusted data
   - PII detection, rate limiting
2. **LLM-level guardrails**
   - Safety system prompts and explicit instructions help, but are **not a security boundary** (they can be bypassed by injection)
   - Enforce critical rules (permissions, spend limits, irreversible actions) **deterministically in code**
3. **Tool boundaries**
   - Least privilege: minimum necessary permission per tool
   - Input validation on tool arguments
   - Sandbox code execution
   - **Human-in-the-loop** for irreversible actions
4. **Output filtering**
   - PII screening, content policy, relevance checks, citation and attribution checks
   - Low-confidence outputs go to human review
5. **Observability**
   - Log traces, inputs, tool calls, outputs, errors, and cost

**Rule of thumb:** match guardrails to the **impact of the actions the agent can take**, not how risky the query sounds. A read-only FAQ bot needs fewer controls than a finance agent that can move money, which needs all layers.

---

# Part 2: LangChain agent

## 2.1 Overview

**Cycle:** Reason → Act → Observe → Repeat
- **Reason**: decide what to do next (which tool, or answer directly)
- **Act**: call a tool or respond to the user
- **Observe**: read the tool result

Key points:
- Decides which tool to use, reasons across multiple steps, iterates until the goal is met
- **The LLM never executes tools.** It generates a tool call request; LangChain (your process) runs the code.
- `create_agent` (`from langchain.agents import create_agent`) builds the whole loop: call model → run tool if requested → feed result back → repeat
- Built on LangGraph. The loop is a graph with a model node and a tools node.
- It uses the provider's **native tool-calling API**, not the old text-based "Thought/Action" prompt format. It replaced the older `create_react_agent`, so older tutorials may show different imports.
- With native tool calling, "Reason" is the model's decision to call a tool. Explicit thoughts appear only if the model emits them or is a reasoning model.

```python
from langchain.agents import create_agent

agent = create_agent(model="...", tools=[my_tool], system_prompt="...")
result = agent.invoke({"messages": [{"role": "user", "content": "..."}]})
```
*(Check your installed version's docs, as the API changed between v0.x and v1.)*

## 2.2 Under the hood: what `agent.invoke()` does

`invoke()` looks like one call but runs a **full loop** internally.

1. Your code calls `agent.invoke(...)`
2. LangChain builds the request: system prompt + messages + tool definitions
3. **Tool schemas are auto-generated** from each `@tool` function: name, docstring (description), and type hints (parameters)
4. LangChain sends the request to the model API with two things:
   - **messages**: the entire conversation so far
   - **tools**: the list of functions the model is allowed to call
5. The model reads the **whole conversation** (not just the last user message) and returns either:
   - a **tool call request** (structured), or
   - a **final text answer**
6. LangChain inspects the response: **are there tool calls?**
   - **Yes** → extract each call's `name`, `args`, `id`, then continue to step 7
   - **No** → return the text answer to the user (loop ends)
7. **Lookup:** LangChain keeps an internal **registry (a dict)** mapping tool names → tool objects (the function plus its schema). It finds the one matching `name`.
8. **Execute:** the actual Python code runs in **your application process**, with `args`
9. **Return the result** to the model as a **tool message** carrying the same `id`, so the model knows which call it answers
10. LangChain sends the **full history again** (the model is **stateless**), including the new tool result
11. Repeat from step 5 until the model responds with no tool calls

**Why the `id` matters:** the model may request several tools at once (parallel calls). The **name** maps to the function; the **id** links each result back to its request. Without a framework, you would build this name-to-function mapping and id matching yourself.

## 2.3 Things to remember

- **Each step depends on the previous one**, since the growing history is the agent's working memory. Independent parallel tool calls are an exception.
- **Cost and latency grow each iteration** because the full history is resent. Long loops risk hitting the context limit. Prompt caching and trimming or summarizing history help.
- **Stop conditions:** the loop ends when the model returns no tool calls, or when a limit is hit (`recursion_limit`). Always set sensible limits.
- **Memory across invocations** needs a checkpointer + `thread_id` (or manual history), not just `invoke()`.
- **Tool errors:** decide whether to feed the error back to the model (lets it self-correct) or fail hard.
- **Security:** tools run on your machine with your permissions, so apply the least-privilege, validation, and sandboxing rules from section 1.5.
- **Unified interface:** one API across providers (OpenAI, Anthropic, etc.), but tool-calling reliability differs per model. Test the one you choose.

---

# Part 3: Sample conversation JSON

This is the **state of the request after one full loop**. It contains the system prompt, tool schema, earlier conversation history, the user's new prompt, the assistant's tool call, the tool result, and the final answer.

> Field names are simplified. Real providers differ: OpenAI sends `arguments` as a JSON **string** inside `function`, while LangChain normalizes tool calls to `{name, args, id}` with `args` as a dict. A tool result is a message with `role: "tool"` (`ToolMessage` in LangChain).

## 3.1 Walkthrough

| Message | Role | What it is |
|---|---|---|
| 1 | `system` | System prompt: sets behavior and rules |
| 2 | `user` | Earlier user turn (history) |
| 3 | `assistant` | Earlier assistant reply (history), no tool calls |
| 4 | `user` | New prompt for the agent |
| 5 | `assistant` | **Tool call request**: `name`, `args`, `id`. No text yet |
| 6 | `tool` | **Tool call response**: carries `tool_call_id` matching message 5 |
| 7 | `assistant` | **Final answer**: no tool calls, so the loop ends |

**Loop trace**
- **LLM call 1:** receives messages 1-4 + tools → returns message 5 (tool call)
- LangChain looks up `get_weather` in the registry and runs it with `{"city": "Paris", "units": "celsius"}`
- **LLM call 2:** receives messages 1-6 (full history, since the model is stateless) → returns message 7 (no tool calls, loop stops)

Note that message 7 reports **current** weather while the user travels **next week**. A good answer flags that mismatch, which is a good habit to test for in evaluation.

## 3.2 JSON

```json
{
  "model": "your-model-name",
  "tools": [
    {
      "name": "get_weather",
      "description": "Get the current weather for a city. Use when the user asks about current conditions or what to wear/pack. Returns temperature, condition and rain chance.",
      "parameters": {
        "type": "object",
        "properties": {
          "city": {
            "type": "string",
            "description": "City name, e.g. 'Paris'"
          },
          "units": {
            "type": "string",
            "enum": ["celsius", "fahrenheit"],
            "description": "Temperature units"
          }
        },
        "required": ["city"]
      }
    }
  ],
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful travel assistant. Use the available tools for live data such as weather. Never guess current conditions. If a tool fails, tell the user instead of inventing an answer."
    },
    {
      "role": "user",
      "content": "I'm planning a trip to Paris next week. Any tips?"
    },
    {
      "role": "assistant",
      "content": "Great choice! Book popular museums in advance, use the Metro for getting around, and carry a light jacket. Want help with an itinerary?",
      "tool_calls": []
    },
    {
      "role": "user",
      "content": "What's the weather like there right now? Should I pack an umbrella?"
    },
    {
      "role": "assistant",
      "content": "",
      "tool_calls": [
        {
          "id": "call_abc123",
          "name": "get_weather",
          "args": {
            "city": "Paris",
            "units": "celsius"
          },
          "type": "tool_call"
        }
      ]
    },
    {
      "role": "tool",
      "tool_call_id": "call_abc123",
      "name": "get_weather",
      "content": "{\"temp_c\": 12, \"condition\": \"Light rain\", \"rain_chance_pct\": 80}"
    },
    {
      "role": "assistant",
      "content": "Right now Paris is 12°C with light rain and an 80% chance of more. Since you travel next week, check the forecast closer to your dates, but a compact umbrella and a waterproof jacket are a safe bet.",
      "tool_calls": []
    }
  ]
}
```

A standalone copy is in `agent_conversation_sample.json`.

---

# Part 4: Model Context Protocol (MCP)

## 4.1 What is MCP?

- **Model Context Protocol**: an open standard, introduced by Anthropic, that gives AI applications one universal way to connect to external tools, data sources, and systems ("USB-C for AI")
- Messages use **JSON-RPC 2.0**; the specification itself is open
- Solves the **N×M integration problem**: instead of every app writing a custom integration for every tool, each app implements MCP once and each tool exposes MCP once
- **MCP vs function calling:** they complement each other. MCP standardizes how tools are **discovered and served**. Function calling is how the **LLM requests** a tool. The host converts MCP tool definitions (`inputSchema`) into the LLM's tool schemas.

## 4.2 Architecture

| Role | What it is |
|---|---|
| **Host** | The AI application (Claude Desktop, VS Code, your own agent app). Manages the LLM, the UI, and user consent. |
| **Client** | Created by the host. Each client has a **dedicated 1:1 connection** to one server. |
| **Server** | Exposes tools, resources, and prompts. Can run **locally or remotely**. |

The LLM never talks to MCP directly. Flow: user → host → LLM (with tool schemas from MCP) → tool call → client → `tools/call` → **server executes** → result → host → LLM.

**How it fits your LangChain notes (Part 2):** the agent loop is the same. The only difference is who runs the tool. With plain `@tool` functions, **your process** runs the code. With MCP, the **MCP server** runs it, and the client just forwards the request and returns the result.

## 4.3 Core components

| Component | Controlled by | What it is |
|---|---|---|
| **Tools** | **Model** | Functions the AI can call to perform actions |
| **Resources** | **Application** | Read-only data for context (files, README.md, DB records, GET-style endpoints), identified by URI; the host decides what goes into the context window |
| **Prompts** | **User** | Reusable templates that structure LLM interactions (e.g. slash commands) |

Servers declare these as **capabilities**. Clients can also offer capabilities to servers: **sampling** (server asks the host's LLM), **roots**, and **elicitation** (server asks the user for input).

## 4.4 Connection lifecycle

1. **Initialize**
   - Client → `initialize` **request**: protocol version, capabilities, client info
   - Server → **response**: its version, capabilities, server info (versions are negotiated)
   - Client → `initialized` **notification**: ready to exchange messages
2. **Discover** (during the operation phase): client calls `tools/list`, `resources/list`, `prompts/list`
3. **Operate**: the LLM decides which tool to call; the client sends `tools/call`; **the server executes it** and returns the result
4. **Shutdown**: client closes the transport and ends the session (over stdio: close stdin, then the process exits)

The spec formally defines three phases (initialization, operation, shutdown); discovery happens inside operation. The 4-step view is a handy mental model.

## 4.5 JSON-RPC essentials

- Three message types: **request** (has `id`), **response** (same `id`), **notification** (no `id`, no response)
- The `id` **matches a response to its request** (like the tool call ids in Part 2)
- Communication is **bidirectional**: servers can send requests to clients too

**Methods**
- `tools/list`: called after connecting; returns every tool the server offers, so agents **discover tools without hardcoding them**. Servers can notify the client when the list changes.
- `tools/call`: sent each time the LLM decides it needs a tool; contains tool name + arguments. The LLM chooses based on the user's request and the **tool descriptions** (same rule as docstrings: vague description means wrong tool choice).

**Two kinds of failure**
- **Tool execution error:** the tool ran but failed. The response is a normal `result` with `isError: true`, so the model can read the error and recover.
- **Protocol error:** unknown tool, bad params, unknown method. The response is a JSON-RPC `error` object (e.g. code `-32602`).

## 4.6 Transports

| | **stdio** | **Streamable HTTP** |
|---|---|---|
| How | Host spawns the server as a **child process** on the same machine; messages flow over **stdin/stdout** (newline-delimited JSON-RPC) | Server runs as an **HTTP service** at one MCP endpoint; responses can stream via **SSE** |
| Network | None, so low overhead | Remote or cloud deployment |
| Clients | One client per spawned process | **Many clients** can connect |
| Best for | Local tools (files, git) | Shared, remote, or cloud-hosted servers |
| Watch out | **Never write logs to stdout**, use stderr | Auth (OAuth-based), validate `Origin`, bind to localhost for local servers, session handling |

Streamable HTTP replaced the older HTTP+SSE transport; SSE is now an optional streaming mechanism inside it.

## 4.7 Code: connecting with LangChain

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient({
    "math": {                         # label for this server
        "transport": "stdio",         # how to talk to it
        "command": "python",          # program to run
        "args": ["/path/to/math_server.py"],
    },
    "weather": {
        "transport": "streamable_http",
        "url": "http://localhost:8000/mcp",
    },
})
tools = await client.get_tools()      # pass these to create_agent(...)
```

- `MultiServerMCPClient` creates one connection per configured server. By default the adapter opens a fresh session per tool call; use an explicit session if a server must keep state (check the adapter docs for your version).
- **Responsibility shifts:** tool implementation, backend credentials, and schemas live in the **server**. Your app becomes the **orchestrator**.

## 4.8 `uvx` and `StdioServerParameters`

- `uvx` (alias for `uv tool run`) runs a Python tool from **PyPI** in an **isolated environment**, with no `pip install` into your own environment. It downloads the package, caches it, and runs its entry point.
- With stdio, the client launches that entry point as a child process and exchanges JSON-RPC over its **stdin/stdout**.

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

params = StdioServerParameters(
    command="uvx",
    args=["oracle.oci-usage-mcp-server"],        # verify package name in Oracle's MCP repo
    env={"OCI_CONFIG_PROFILE": "DEFAULT"},       # env var names depend on the server's README
)

async with stdio_client(params) as (read, write):
    async with ClientSession(read, write) as session:
        await session.initialize()               # handshake
        tools = await session.list_tools()       # discover
        result = await session.call_tool("tool_name", {"arg": "value"})  # operate
# exiting the context closes the transport (shutdown)
```

**Supply-chain caution:** `uvx` runs third-party code with your permissions. **Pin versions** (e.g. `pkg@1.2.3`) and vet sources. The package name and env var above are unverified placeholders; check the server's README.

## 4.9 When MCP shines

Use it when tools are **external or reusable across apps**:

- **Cloud (AWS, Azure, OCI):** manage infrastructure through a server instead of writing SDK integration code in your app (the server wraps the SDK)
- **Databases (Oracle DB, PostgreSQL):** query without embedding drivers and credentials in your agent app
- **Dev tools (GitHub, Jira):** create issues, review PRs from any MCP-enabled agent
- **Communication (Slack, email):** send messages without per-app integrations
- **File system:** read/write files on a machine, restricted to **allowed directories**

**When not to bother:** a single app with a few simple in-process tools is usually simpler and faster as plain function tools, with no extra process or protocol hop.

## 4.10 Security (ties into section 1.5)

- **Tool poisoning:** a malicious server can hide instructions in tool descriptions. **Treat third-party servers as untrusted.**
- **Indirect prompt injection** via tool results or resources
- **Over-permissioned servers:** apply least privilege and scope credentials narrowly
- **Human approval** for sensitive or irreversible tool calls (the host should show tool calls to the user)
- **Authorization:** remote servers should use OAuth-based auth; stdio servers typically take credentials from environment variables
- **Observability:** log every `tools/call` with arguments and results

## 4.11 Sample MCP session (JSON-RPC)

A full session between a client and a small `math-server`: handshake, discovery, a successful call, a tool execution error, and a protocol error.

> The `step`, `direction`, and `type` fields are **added for readability only**. On the wire, each message is just the object under `message`. The protocol version string is an example; use whichever version your SDK negotiates.

| Step | Direction | What happens |
|---|---|---|
| 1 | client → server | `initialize` request: protocol version, client capabilities |
| 2 | server → client | Response with server capabilities (`tools`, `resources`, `prompts`) |
| 3 | client → server | `notifications/initialized` (notification, no `id`, no reply) |
| 4-5 | client ↔ server | `tools/list` → list of tools with `name`, `description`, `inputSchema` |
| 6-7 | client ↔ server | `tools/call` `add` → result `7`, `isError: false` |
| 8-9 | client ↔ server | `tools/call` `divide` by 0 → **tool execution error**: `isError: true` |
| 10-11 | client ↔ server | `tools/call` unknown tool → **protocol error**: JSON-RPC `error` |

**Connecting it to the agent loop:** between steps 5 and 6, the host converts each tool's `inputSchema` into the LLM's tool schema and sends it with the conversation (Part 3 format). When the LLM returns a tool call, the client wraps it in a `tools/call` request (step 6), and the server's result (step 7) becomes the `tool` message sent back to the LLM.

```json
[
  {
    "step": 1,
    "direction": "client -> server",
    "type": "request",
    "message": {
      "jsonrpc": "2.0",
      "id": 1,
      "method": "initialize",
      "params": {
        "protocolVersion": "2025-06-18",
        "capabilities": {
          "roots": { "listChanged": true },
          "sampling": {}
        },
        "clientInfo": { "name": "my-agent-host", "version": "1.0.0" }
      }
    }
  },
  {
    "step": 2,
    "direction": "server -> client",
    "type": "response",
    "message": {
      "jsonrpc": "2.0",
      "id": 1,
      "result": {
        "protocolVersion": "2025-06-18",
        "capabilities": {
          "tools": { "listChanged": true },
          "resources": {},
          "prompts": {}
        },
        "serverInfo": { "name": "math-server", "version": "0.1.0" }
      }
    }
  },
  {
    "step": 3,
    "direction": "client -> server",
    "type": "notification",
    "message": {
      "jsonrpc": "2.0",
      "method": "notifications/initialized"
    }
  },
  {
    "step": 4,
    "direction": "client -> server",
    "type": "request",
    "message": {
      "jsonrpc": "2.0",
      "id": 2,
      "method": "tools/list"
    }
  },
  {
    "step": 5,
    "direction": "server -> client",
    "type": "response",
    "message": {
      "jsonrpc": "2.0",
      "id": 2,
      "result": {
        "tools": [
          {
            "name": "add",
            "description": "Add two numbers and return the sum. Use for any addition request.",
            "inputSchema": {
              "type": "object",
              "properties": {
                "a": { "type": "number", "description": "First number" },
                "b": { "type": "number", "description": "Second number" }
              },
              "required": ["a", "b"]
            }
          },
          {
            "name": "divide",
            "description": "Divide a by b and return the quotient.",
            "inputSchema": {
              "type": "object",
              "properties": {
                "a": { "type": "number", "description": "Numerator" },
                "b": { "type": "number", "description": "Denominator, must not be zero" }
              },
              "required": ["a", "b"]
            }
          }
        ]
      }
    }
  },
  {
    "step": 6,
    "direction": "client -> server",
    "type": "request",
    "message": {
      "jsonrpc": "2.0",
      "id": 3,
      "method": "tools/call",
      "params": {
        "name": "add",
        "arguments": { "a": 3, "b": 4 }
      }
    }
  },
  {
    "step": 7,
    "direction": "server -> client",
    "type": "response",
    "message": {
      "jsonrpc": "2.0",
      "id": 3,
      "result": {
        "content": [{ "type": "text", "text": "7" }],
        "isError": false
      }
    }
  },
  {
    "step": 8,
    "direction": "client -> server",
    "type": "request",
    "message": {
      "jsonrpc": "2.0",
      "id": 4,
      "method": "tools/call",
      "params": {
        "name": "divide",
        "arguments": { "a": 10, "b": 0 }
      }
    }
  },
  {
    "step": 9,
    "direction": "server -> client",
    "type": "response (tool execution error)",
    "message": {
      "jsonrpc": "2.0",
      "id": 4,
      "result": {
        "content": [{ "type": "text", "text": "Error: division by zero" }],
        "isError": true
      }
    }
  },
  {
    "step": 10,
    "direction": "client -> server",
    "type": "request (unknown tool)",
    "message": {
      "jsonrpc": "2.0",
      "id": 5,
      "method": "tools/call",
      "params": {
        "name": "multiply",
        "arguments": { "a": 2, "b": 5 }
      }
    }
  },
  {
    "step": 11,
    "direction": "server -> client",
    "type": "response (protocol error)",
    "message": {
      "jsonrpc": "2.0",
      "id": 5,
      "error": {
        "code": -32602,
        "message": "Unknown tool: multiply"
      }
    }
  }
]
```

A standalone copy is in `mcp_session_sample.json`.

---

# Part 5: OpenAI agent stack (Responses API + Agents SDK)

> Model names and API details change quickly. The facts here were checked in late October 2026. Verify the model lineup and SDK parameters against OpenAI's docs for your version.

## 5.1 Stack at a glance

| Layer | What it is | You write the loop? |
|---|---|---|
| **Models** | GPT-5.x reasoning family (GPT-5.6 Sol / Terra / Luna as of July 2026; GPT-5.5 is the previous generation); GPT-4.1 is older and non-reasoning | n/a |
| **Responses API** | Core API: model call + built-in tools + function calling + optional server-side state | **Yes** |
| **Agents SDK** | Framework: agent loop, handoffs, guardrails, sessions, tracing | **No** |

- "Reasoning vs non-reasoning" is a spectrum: GPT-5-series models take a `reasoning.effort` setting (from none up to high or higher). Raise it only when evals show a measurable gain, since it costs latency and tokens (reasoning tokens are billed as output).
- Tiered models map onto the **two-tier pattern** from Part 1: a cheap, fast tier for routing and guardrail checks, the flagship tier for hard reasoning. Check current pricing first.

## 5.2 Responses API

- Endpoint `/v1/responses`. You send `input` and get structured `output` items (messages, function calls, reasoning items).
- OpenAI recommends it for reasoning, tool-calling, and multi-turn use cases. Chat Completions remains supported, but new features land on Responses first.
- **Assistants API:** its shutdown date (26 Aug 2026) has passed, so nothing new should use it. Migration map: Assistants → Prompts, Threads → Conversations, Runs → Responses, Run-Steps → Items.
- **Single call, not "text only":** one request is one model call, but it supports built-in tools, function calling, structured output, streaming, and multi-turn state. The difference from the Agents SDK is **who owns the loop**.

**Conversation chaining**
- Chat Completions is stateless: you resend the full history every call.
- Responses: pass `previous_response_id` and the server links the earlier context (`store` must be enabled). A durable Conversations API also exists. For stateless or zero-data-retention flows, pass back the returned output items yourself.
- **Cost reality:** chaining saves code and bandwidth, not input tokens. Earlier context in the chain is still processed and billed as input (confirm in OpenAI's conversation-state docs). Real cost levers: prompt caching (static prompt first, dynamic last) and compaction.
- Tools are not inherited, so resend them on each call.

**Built-in (hosted) tools**
- `{"type": "web_search"}`, `{"type": "file_search"}`, `{"type": "code_interpreter"}`, plus others such as remote MCP and computer use
- The model decides when to use them (you can force it with `tool_choice`). They run **on OpenAI's side**, so you don't execute them.
- Available on Responses but not Chat Completions, and metered separately from tokens.

**Function calling** works as in Part 1: the model returns a `function_call` item with `call_id`, `name`, and `arguments` (a JSON **string**). Your code runs the tool and sends back a `function_call_output` with the same `call_id`. See sample 5.9.

## 5.3 Agents SDK

**Core concepts**
- **Agent**: LLM + instructions + tools, plus handoffs, guardrails, `output_type`
- **Runner**: executes the agent loop
- **Tools**: function tools, agents as tools, hosted tools
- **Handoffs**: transfer control between agents
- **Guardrails**: validate input, output, and tool calls
- **Sessions**: memory across runs (e.g. `SQLiteSession`)
- **Tracing**: records what the agent did

It is open source, available in Python and JS/TS, and **provider-agnostic** (Responses, Chat Completions, and 100+ other LLMs). It also supports MCP servers, which connects to Part 4.

**The loop (`Runner.run`)**: the runner calls the LLM with the agent's model, settings, and message history. Each turn has one of three outcomes:
1. **Final output** → loop ends
2. **Handoff** → current agent switches, loop continues
3. **Tool calls** → SDK runs them, appends results, loop continues

Whether to call a tool is the model's choice **every turn**; an agent with no tools simply answers directly. Use `max_turns` (default 10, check your version) to prevent runaway loops. It raises `MaxTurnsExceeded` when hit, which addresses the "runaway execution" threat from section 1.5.

```python
from agents import Agent, Runner, function_tool

@function_tool
def get_weather(city: str) -> str:
    """Get the current weather for a city."""   # docstring + type hints become the schema
    return f"Sunny in {city}"

agent = Agent(name="Assistant", instructions="Be concise.", tools=[get_weather])
result = Runner.run_sync(agent, "Weather in Paris?")   # async: await Runner.run(...)
print(result.final_output)
```

**Tool types**
| Type | Meaning |
|---|---|
| **Function tools** | Your Python functions; the LLM chooses based on the description (vague docstring means wrong choice) |
| **Agents as tools** | `agent.as_tool(...)`: one agent called by another |
| **Hosted tools** | Run by OpenAI (web search, file search, code interpreter), available with OpenAI Responses models |

**Responses API vs Agents SDK**
| | Responses API | Agents SDK |
|---|---|---|
| Loop | You write it | SDK runs it |
| Multi-agent, handoffs | Build yourself | Built in |
| Guardrails, tracing | Build yourself | Built in |
| Best for | One-shot tasks, simple tool use, full control | Multi-step loops, multiple agents, safety and observability |

## 5.4 Multi-agent patterns

| | **Handoff** (peer-to-peer) | **Manager** (centralized) |
|---|---|---|
| Control | Receiving agent **takes over** the conversation | Manager **keeps control**, calls specialists as tools |
| Mechanism | `handoffs=[...]` (a tool call like `transfer_to_<agent>`) | `agent.as_tool(...)` |
| Best for | Triage and routing to a specialist who owns the rest of the chat | Combining several specialists' outputs, consistent final answer, final guardrails |

```python
history_tutor = Agent(name="History Tutor", handoff_description="History questions",
                      instructions="Answer history questions.")
math_tutor = Agent(name="Math Tutor", handoff_description="Math questions",
                   instructions="Solve math problems step by step.")

triage_agent = Agent(
    name="Triage Agent",
    instructions="Route the user to the right specialist.",
    handoffs=[history_tutor, math_tutor],
    model="...",   # a cheap, fast model is usually enough for routing
)

manager = Agent(
    name="Manager",
    instructions="Use the tools to answer, then write one combined reply.",
    tools=[history_tutor.as_tool(tool_name="ask_history", tool_description="History expert")],
)
```

- A handoff is a specialized tool call; the specialist sees the conversation history by default.
- `handoff_description` on the specialists helps the triage agent choose.
- **Architect note:** every handoff or sub-agent call adds LLM calls, tokens, and latency. Start with **one agent and good tools**, and split only when instructions or tools get too large or domains need isolation.

## 5.5 Guardrails

- **Input guardrails**: run on the initial user input, **first agent only**
- **Output guardrails**: run on the final output, **last agent only**
- **Tool guardrails**: run before and after each custom function-tool call, which covers the gaps in multi-agent chains
- A guardrail is a **function**, not necessarily an agent. It often uses a small helper agent, but it can be plain code. It returns `GuardrailFunctionOutput(output_info, tripwire_triggered)`.
- **Tripwire:** when `tripwire_triggered` is true, the runner immediately raises `InputGuardrailTripwireTriggered` or `OutputGuardrailTripwireTriggered` and halts execution. Catch it with `try/except` and return a safe response.
- **Execution mode matters:** by default an input guardrail runs **in parallel** with the agent. That saves latency, but the agent may already have started (spent tokens, even called tools). For anything with side effects, use **blocking** mode (`run_in_parallel=False`) so the guardrail finishes first. Output guardrails always run after the agent completes, so they have no parallel option.
- Guardrails complement, not replace, the code-enforced controls from section 1.5.

```python
from pydantic import BaseModel
from agents import (Agent, Runner, GuardrailFunctionOutput, RunContextWrapper,
                    TResponseInputItem, input_guardrail,
                    InputGuardrailTripwireTriggered)

class HomeworkCheck(BaseModel):
    is_homework: bool
    reasoning: str

checker = Agent(name="Homework checker", output_type=HomeworkCheck,
                instructions="Is the user asking a homework question?",
                model="...")   # small, cheap model

@input_guardrail(run_in_parallel=False)   # blocking; verify param name in your SDK version
async def homework_only(ctx: RunContextWrapper, agent: Agent,
                        input: str | list[TResponseInputItem]) -> GuardrailFunctionOutput:
    result = await Runner.run(checker, input, context=ctx.context)
    return GuardrailFunctionOutput(
        output_info=result.final_output,
        tripwire_triggered=not result.final_output.is_homework,
    )

triage_agent.input_guardrails = [homework_only]   # or pass input_guardrails=[...] when creating

try:
    await Runner.run(triage_agent, "Write my holiday itinerary")
except InputGuardrailTripwireTriggered:
    print("Blocked: not a homework question")
```

## 5.6 Tracing

- On by default. View runs in the OpenAI dashboard (platform.openai.com → Traces).
- Records each step as spans: LLM generations, tool calls, handoffs, guardrail checks. This is the **observability** layer from section 1.5.
- Extensible: custom spans and external destinations such as Logfire, AgentOps, and Braintrust.
- Traces can contain prompts and tool data. Check the sensitive-data settings and your data-retention policy, as tracing may be limited under zero-data-retention setups.

## 5.7 When to use what

- **Responses API alone:** one-shot tasks, simple tool use, or when you want full control of the loop
- **Agents SDK:** multi-step loops, multiple agents, handoffs, guardrails, built-in tracing
- **LangChain (Part 2):** an alternative orchestrator with a unified interface across providers
- **MCP (Part 4):** how any of these frameworks connect to external tools and data

## 5.8 How it connects to earlier parts

| Concept | Part 1 / 2 | OpenAI stack |
|---|---|---|
| Loop | Reason → act → observe | `Runner` loop |
| Tool call id | `id` links result to request | `call_id` in `function_call` / `function_call_output` |
| Tool schema | `@tool` docstring + type hints | `@function_tool` docstring + type hints |
| Stateless model | Resend full history | Resend, or `previous_response_id` / sessions |
| Guardrails | Code-enforced, layered | Input / output / tool guardrails + tripwires |
| Observability | Log traces, tool calls, cost | Tracing dashboard |

## 5.9 Sample flows (JSON)

> Simplified for learning: IDs and token counts are illustrative, and the two Agents SDK traces are conceptual (not the real trace export format). The Responses API bodies follow the real request and response shapes.

### Responses API: function call round trip

| Step | What happens |
|---|---|
| 1 | Request with `instructions`, `input`, and tools (a function tool + hosted `web_search`) |
| 2 | Response contains a `function_call` item (`call_id`, `name`, `arguments` as a JSON string). The model only **requested** the tool |
| 3 | Your code runs the tool, then sends `function_call_output` with the same `call_id` and `previous_response_id` |
| 4 | Final `message` item with `output_text`. Note `input_tokens` rose, since the chain's context is still billed |

```json
[
  {
    "step": 1,
    "direction": "your app -> POST /v1/responses",
    "body": {
      "model": "your-model-name",
      "instructions": "You are a helpful travel assistant. Use tools for live data. Never guess current conditions.",
      "input": [
        {
          "role": "user",
          "content": "What's the weather in Paris right now? Do I need an umbrella?"
        }
      ],
      "tools": [
        {
          "type": "function",
          "name": "get_weather",
          "description": "Get the current weather for a city. Use when the user asks about current conditions.",
          "parameters": {
            "type": "object",
            "properties": {
              "city": {
                "type": "string",
                "description": "City name, e.g. 'Paris'"
              }
            },
            "required": [
              "city"
            ]
          }
        },
        {
          "type": "web_search"
        }
      ],
      "store": true
    }
  },
  {
    "step": 2,
    "direction": "OpenAI -> your app (response)",
    "body": {
      "id": "resp_001",
      "object": "response",
      "status": "completed",
      "output": [
        {
          "type": "reasoning",
          "id": "rs_001",
          "summary": []
        },
        {
          "type": "function_call",
          "id": "fc_001",
          "call_id": "call_abc123",
          "name": "get_weather",
          "arguments": "{\"city\": \"Paris\"}"
        }
      ],
      "usage": {
        "input_tokens": 180,
        "output_tokens": 95,
        "output_tokens_details": {
          "reasoning_tokens": 70
        }
      }
    },
    "note": "The model only REQUESTED the tool. Your code runs get_weather. Note 'arguments' is a JSON string here."
  },
  {
    "step": 3,
    "direction": "your app -> POST /v1/responses (send the tool result)",
    "body": {
      "model": "your-model-name",
      "previous_response_id": "resp_001",
      "input": [
        {
          "type": "function_call_output",
          "call_id": "call_abc123",
          "output": "{\"temp_c\": 12, \"condition\": \"Light rain\", \"rain_chance_pct\": 80}"
        }
      ],
      "tools": [
        {
          "type": "function",
          "name": "get_weather",
          "description": "Get the current weather for a city. Use when the user asks about current conditions.",
          "parameters": {
            "type": "object",
            "properties": {
              "city": {
                "type": "string",
                "description": "City name, e.g. 'Paris'"
              }
            },
            "required": [
              "city"
            ]
          }
        },
        {
          "type": "web_search"
        }
      ]
    },
    "note": "Only the NEW item is sent. The server links earlier context via previous_response_id. Resend tools each call (they are not inherited)."
  },
  {
    "step": 4,
    "direction": "OpenAI -> your app (final response)",
    "body": {
      "id": "resp_002",
      "object": "response",
      "status": "completed",
      "previous_response_id": "resp_001",
      "output": [
        {
          "type": "message",
          "role": "assistant",
          "content": [
            {
              "type": "output_text",
              "text": "It's 12°C in Paris with light rain and an 80% chance of more. Yes, take an umbrella."
            }
          ]
        }
      ],
      "usage": {
        "input_tokens": 310,
        "output_tokens": 40,
        "output_tokens_details": {
          "reasoning_tokens": 0
        }
      }
    },
    "note": "Illustrative numbers. input_tokens is higher than in step 2 because the earlier context in the chain is still processed and billed as input. Check OpenAI's conversation-state docs for current billing details."
  }
]
```

### Agents SDK: triage handoff

```json
{
  "scenario": "Triage agent hands off a math question to a specialist.",
  "note": "Simplified conceptual trace for learning. It is NOT the actual wire format or the trace export format.",
  "agents": [
    "Triage Agent (first agent)",
    "Math Tutor (specialist)",
    "History Tutor (specialist)"
  ],
  "user_input": "Can you help me solve 2x + 3 = 11?",
  "steps": [
    {
      "step": 1,
      "actor": "Runner",
      "action": "Run input guardrails (first agent only)",
      "result": "passed"
    },
    {
      "step": 2,
      "actor": "Triage Agent (LLM call 1)",
      "output": {
        "type": "function_call",
        "name": "transfer_to_math_tutor",
        "arguments": "{}"
      },
      "meaning": "A handoff is a specialized tool call."
    },
    {
      "step": 3,
      "actor": "Runner",
      "action": "Handoff detected: current agent becomes Math Tutor, conversation history carries over",
      "loop": "continues"
    },
    {
      "step": 4,
      "actor": "Math Tutor (LLM call 2)",
      "output": {
        "type": "message",
        "text": "Subtract 3 from both sides: 2x = 8. Divide by 2: x = 4."
      }
    },
    {
      "step": 5,
      "actor": "Runner",
      "action": "Final output (no tool calls, no handoff): loop ends. Output guardrails run for the LAST agent (Math Tutor).",
      "result": "passed"
    }
  ],
  "result.final_output": "Subtract 3 from both sides: 2x = 8. Divide by 2: x = 4."
}
```

### Agents SDK: guardrail tripwire

```json
{
  "scenario": "Blocking input guardrail rejects an off-topic request.",
  "note": "Simplified conceptual trace for learning.",
  "user_input": "Write my holiday itinerary",
  "guardrail": {
    "name": "homework_only",
    "mode": "blocking (run_in_parallel=False)"
  },
  "steps": [
    {
      "step": 1,
      "actor": "Runner",
      "action": "Run input guardrail homework_only on the first agent's input (agent has NOT started)"
    },
    {
      "step": 2,
      "actor": "Guardrail helper agent (small, cheap model)",
      "output": {
        "is_homework": false,
        "reasoning": "Request is about travel planning, not homework."
      }
    },
    {
      "step": 3,
      "actor": "Guardrail function",
      "output": {
        "tripwire_triggered": true
      },
      "meaning": "tripwire_triggered = not is_homework"
    },
    {
      "step": 4,
      "actor": "Runner",
      "action": "Raise InputGuardrailTripwireTriggered and halt. Triage Agent was never called, so no tokens spent and no tools run.",
      "handled_by": "Your try/except returns a safe message to the user."
    }
  ],
  "parallel_mode_difference": "With run_in_parallel=True (default), the Triage Agent starts at the same time as the guardrail. The tripwire still halts execution when it fires, but the agent may already have spent tokens or called tools. Use blocking mode for anything with side effects."
}
```

A standalone copy of all three is in `openai_stack_samples.json`.
