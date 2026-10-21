# AI Agents and LangChain Agents: Reviewed Study Notes

**Contents**
- Part 1: AI agent fundamentals
- Part 2: LangChain agent and what `invoke()` does


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



---

