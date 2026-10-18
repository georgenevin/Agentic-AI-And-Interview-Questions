# AI Agents: Reviewed Study Notes

## 1. What is an AI agent?

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

## 2. Components

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

## 3. Reasoning patterns

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

## 4. Tool definition essentials (e.g. LangChain)

- `@tool` wraps a function so the framework can register it as a callable tool (decorator name varies by framework)
- **Type hints** (e.g. `a: float`) define parameter types in the schema
- **Docstring** becomes the tool description, which is how the LLM decides when and how to use the tool. **Vague docstring leads to wrong tool choice.**
- The framework converts the function into a JSON schema that is sent to the LLM with the user message. The LLM sees the schema and decides whether a tool call is needed.
- The **return value** is not part of what the LLM sees up front; it comes back as the observation after execution.

---

## 5. Safety and guardrails

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



