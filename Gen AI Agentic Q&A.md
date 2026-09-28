# Gen AI Specialist — Interview Q&A: Agentic AI, RAG at Scale & Advanced Topics

---

**Q: How would you prevent an agent from leaking sensitive information in its response that it picked up earlier in the conversation or from a tool call?**

Scope what data a tool call is allowed to return to the model in the first place, rather than trying to filter it out after the fact. Scan the response before it goes back to the user. Minimize how much sensitive data the model has access to at any one time. Be deliberate about what's allowed to persist in conversation memory — don't let sensitive data linger in context longer than the task actually needs it.

---

**Q: How would you design permission scoping for an agent that has multiple tools?**

The agent should only have access to exactly the tools and data it needs for the current task — never broad, account-wide access by default.

Explicitly define the allowed list of tools available for a given task or agent role, rather than exposing every tool to every agent.

Scope each tool's own permissions narrowly — a tool that reads customer data shouldn't also be able to delete records. Read, write, and delete should be separate permission grants, not bundled into one tool.

---

**Q: How would you prevent an agent from running away with excessive tool calls?**

Don't rely on the model to self-regulate — enforce hard runtime limits instead:
- A maximum number of tool calls allowed per task or session
- A spending/budget cap per session
- A timeout per individual step, and an overall timeout for the task

---

**Q: What does it mean to sandbox an agent's actions, and why would you do this even for actions that seem safe?**

Sandboxing means running an agent's actions in an isolated environment — separate from production systems, real user data, and real external side effects — so that anything the agent does is contained and reversible rather than directly hitting live systems.

You sandbox even "safe-looking" actions because the risk isn't really about whether a single action looks dangerous — it's about blast radius and the fact that "safe" is often an assumption, not a guarantee. A tool description or the model's own judgment about what's safe can be wrong, an action can have side effects you didn't anticipate (rate limits, cascading triggers, unintended writes), and a chain of individually "safe" actions can combine into something harmful. Sandboxing means a bad decision costs you a rollback, not an incident.

---

**Q: How would you design an escalation system that hands a task to a human when the agent isn't confident?**

Escalate after the agent has made a reasonable attempt and clearly failed or hit a confidence threshold — not on the first sign of uncertainty, or you'll escalate everything. When it does escalate, it should hand over everything it's already figured out: what it tried, what it found, and why it got stuck — so the human isn't starting from zero and re-doing work the agent already did.

---

**Q: Why do we need full audit logs?**

Every action taken needs a timestamp and enough detail to reconstruct what happened — the context around the decision and any human interventions — so that an incident is actually debuggable after the fact, rather than relying on memory or guesswork about what the agent did and why.

---

**Q: How do you decide the right balance between letting an agent act fully autonomously versus requiring human approval for everything?**

I'd base it on **reversibility and blast radius**, not just task complexity. If an action is easily reversible and low-impact (e.g., drafting a response, querying read-only data), let the agent act autonomously. If an action is irreversible, high-impact, or touches sensitive data or money (e.g., sending an email externally, deleting a record, making a payment), require human approval regardless of how confident the agent is.

In practice I'd design this as tiers rather than a single on/off switch:
- **Fully autonomous** — low-risk, reversible, high-volume actions (read-only lookups, internal drafts)
- **Human-in-the-loop / approve-before-execute** — medium-risk or irreversible actions (sending something externally, modifying a record)
- **Human-only** — high-risk actions the agent should never take unsupervised (financial transactions above a threshold, anything affecting production systems directly)

And I'd revisit the tiering over time — as an agent's error rate on a given action type proves low in practice, it can graduate from approval-required to autonomous, rather than deciding the tiering once and leaving it static.

---

**Q: Red teaming**

Deliberately attempting realistic prompt injection and jailbreak techniques against the system, testing edge cases and unusual inputs, to try to trick the agent into behavior it shouldn't exhibit — done proactively, before real users or bad actors find the same gaps.

---

**Q: How do you monitor guardrails?**

Track the rate at which guardrails are actually triggering over time — a sudden spike or drop is itself a signal worth investigating. Check for confirmed guardrail bypasses found through red teaming or real user reports, and feed those back into strengthening the guardrail rather than treating a bypass as a one-off.

---

**Q: Design a RAG system for over 10 million documents**

Split the system into two halves: **ingestion** and **query serving**.

**Ingestion:**
- Parse PDFs/webpages, check for duplicates so you're not wasting retrieval capacity on redundant content
- Use semantic chunking rather than arbitrary chunk boundaries — arbitrary boundaries reduce retrieval quality by splitting related content apart
- Attach metadata to every chunk — tenant ID, access permissions, timestamp, source URL — so retrieval can filter correctly and results stay traceable
- Generate embeddings in batches for efficiency
- Store in an ANN (Approximate Nearest Neighbor) index for fast retrieval at scale — ANN is a data structure/algorithm for finding items in a large collection that are closest to a query item without doing an exhaustive comparison against every item

**Query serving — hybrid retrieval:**
- Combine BM25 (keyword/lexical match) with dense vector retrieval (semantic similarity), then merge and re-rank the combined results
- For citation, every chunk should carry its document ID, chunk ID, and exact character offsets so an answer can be traced back precisely
- Keep re-ranking scoped to a small candidate set per query rather than re-ranking everything — re-rank the top-N from initial retrieval, not the full corpus

---

**Q: Your RAG chatbot gives a fluent but wrong answer — how do you diagnose it?**

First, isolate whether the failure is at the **retrieval** stage or the **generation** stage.

**Check retrieval first:** is the correct source actually present in the top-K retrieved results? If it's not there, it's a retrieval problem — investigate the chunking strategy, embedding quality, or metadata filters that may be excluding the right content.

**If retrieval is fine, it's a generation problem:** check prompt grounding (is the prompt actually instructing the model to rely only on retrieved context?), context ordering (is the most relevant chunk positioned where the model weighs it appropriately?), and whether there are too many distracting/irrelevant chunks in the context competing with the correct one. Run evals to confirm the fix actually resolved it rather than assuming.

---

**Q: AI agent keeps calling the wrong tool — how do you diagnose and fix it?**

Check the full agent trajectory: which tool it selected, the arguments it generated, the tool's response, and the final answer — don't just look at the end result in isolation.

**Common causes:**
- Poor tool design — vague tool descriptions
- Overlapping responsibilities between tools, so the model can't clearly distinguish which one to use
- Weak parameter schemas that don't constrain what a valid call looks like

**Fixes:**
- Make tool boundaries much cleaner — each tool should have one specific, unambiguous responsibility
- Strengthen schema design — required arguments, enums instead of free text where possible, input validation before execution
- Add few-shot examples showing correct tool-selection patterns in similar situations

---

**Q: LLM inference — is it parallel?**

Not fully. LLMs are autoregressive during generation — each next token depends on the tokens generated before it, so token-by-token decoding is inherently **sequential**, not parallel. Where parallelism does happen: the **prefill** stage (processing the input prompt) can be parallelized across all input tokens at once, since they're all already known. And at the system level, you can batch multiple *separate* requests together to improve throughput, even though each individual response is still generated token-by-token.

---

**Q: How can smaller models improve an AI system?**

Use them for narrower, well-defined sub-tasks where a large model is overkill — intent classification, request routing, information extraction, validation/guardrail checks. This is essentially multi-model orchestration: routing tasks to the smallest model capable of handling them, which improves both latency and cost-effectiveness compared to sending everything to the largest model by default.

---

**Q: How do you measure LLM latency?**

- **TTFT (Time to First Token)** — how long the user waits before seeing the first token appear
- **TPS (Tokens Per Second)** — how fast tokens stream once generation starts
- A large prompt increases TTFT, since the prefill stage has more input to process before generation can begin
- *(Worth also mentioning: total end-to-end latency, and inter-token latency for streaming UX — TTFT and TPS alone don't capture the full picture.)*

---

**Q: Your RAG system works fine at 10k documents but fails at 100 million documents — why?**

At that scale, a single-index approach breaks down on retrieval speed and recall quality. You need **distributed retrieval** — sharding the index across multiple nodes — and then merging and re-ranking results across shards. This generally comes with a tradeoff: better recall at scale comes at the cost of higher latency and higher infrastructure cost, so the system needs to be designed with that tradeoff explicitly in mind rather than assuming the 10k-document architecture just scales linearly.

---

**Q: Design memory for an agent**

- **Short-term memory** — the current conversation/task context
- **Long-term memory** — persisted externally (e.g., a vector store or database) so it survives beyond a single session
- **Shared memory** — used in multi-agent systems where multiple agents need visibility into the same state
- **Episodic memory** — recall of specific past events/interactions, useful for personalization and learning from prior outcomes

---

**Q: What if a retrieved document contains a prompt injection?**

Retrieved content should never be allowed to directly trigger actions — treat it as data, not instructions. Every tool call should be validated server-side against least-privilege access: if an agent only needs read permission, it should never have write or delete access available to it in the first place, regardless of what a retrieved document says. You can add a scanning layer that checks retrieved content for suspicious instructions before it's passed to the model, and keep a human in the loop for any action with real consequences.

---

**Q: Multi-tenant isolation**

- **Data isolation** — one tenant's data is never visible to another
- **Access isolation** — permission boundaries enforced per tenant
- **Compute isolation** — one tenant's workload can't starve or impact another's resources
- **Cache isolation** — cached results from one tenant never leak into another tenant's responses
- **Tool isolation** — tools/actions available to an agent are scoped per tenant, not shared globally

---

**Q: Agentic RAG**

Instead of doing a single retrieval pass, the system can plan and retrieve information from multiple sources across multiple steps, using different tools as needed, before generating a final response — turning retrieval from a one-shot lookup into a reasoning process that decides what to retrieve, when, and from where.

---

**Q: How does an AI agent recover from failure?**

**Temporary failure** (e.g., a transient API error) — retry, ideally with backoff.

**Permanent failure** — persist state and checkpoint progress as the agent works, so if the workflow fails partway through, it can resume from the last successful step instead of restarting the entire task from scratch.

---

**Q: Agentic AI vs. Gen AI**

**Gen AI** — a model that generates content in response to a prompt; it's reactive, single-shot.

**Agentic AI** — a system built on top of a model that autonomously pursues a goal through multi-step reasoning and action, using tools and making decisions along the way.

Example: Gen AI writes a marketing plan when asked. An agent researches the market, writes the plan, and sends the email — without being told each individual step.

---

**Q: How do you handle hallucination in an agent?**

- **Tool grounding** — use real tools for live/current data instead of relying on the model's internal memory, which can be stale or simply wrong
- **Confidence thresholds** — flag or escalate low-confidence outputs rather than presenting them as certain
- **Human-in-the-loop** for consequential decisions
- **Structured output schemas** — constrain what a valid response looks like, which reduces the surface area for ungrounded free-form claims

---

**Q: What is the role of memory in AI agents?**

- **Short-term** — scoped to the current context window, for the immediate task
- **Long-term** — persisted externally (e.g., a vector database like Chroma or Pinecone, or on-disk storage) so it survives across sessions
- **Episodic** — recall of specific past events, useful for personalization and continuity across interactions

---

**Q: How do you evaluate an AI application?**

- **Human evals** — manual review by domain experts
- **User evals** — real usage signals and feedback from actual users
- **Code-based evals** — deterministic checks against expected outputs where possible
- **LLM-as-a-judge** — using a model to score outputs against a rubric at scale
- **Continuous evaluation** — running evals on an ongoing basis in production, not just once before launch, since model and data drift mean a one-time eval goes stale

---

**Q: What is a prompt?**

An instruction given to an AI model to get a relevant response.

---

**Q: What is temperature?**

A parameter that controls the randomness/creativity of the model's output. High temperature → more varied, inventive output. Low temperature → more deterministic, consistent output.

---

**Q: What is an agent harness?**

The infrastructure surrounding an LLM that turns it into an agent — everything around the model itself (tool access, memory, orchestration, control loop) needed to make it behave correctly and reliably in a real environment. Examples: Codex, Claude Code.

---

**Q: What is prompt caching?**

In a long conversation with an agent, each new turn appends to the growing conversation and the entire thing gets sent to the LLM again. Prompt caching lets the model reuse the already-processed portion of that prompt from a previous request instead of reprocessing it from scratch every single time — if the model recognizes it's seen the same prefix before, it routes to a cache of that already-processed content. This is done using a `cache_control` parameter that marks where the reusable (stable) content ends and the new content begins. The benefit is both lower cost and lower latency, since the model isn't re-processing the same tokens repeatedly across turns.

---

**Q: Agent memory — types**

- **Working memory** — conversation state; short-term information needed for the current task
- **Episodic memory** — remembers *what* happened and *when*
- **Semantic memory** — general facts/knowledge that are true independent of any specific event
- **Procedural memory** — remembers *how* to do something (learned processes/skills)
- **Tool memory** — how to retrieve and use tools correctly

---

**Q: Why does an AI agent need memory?**

The LLM itself is stateless — it has no memory between calls. As a conversation grows, context windows are limited and more tokens cost more on every request, since each call to the LLM is processed fresh with no persistence built in. Everything the agent appears to "remember" is actually being managed by the system architecture around the model — memory has to be deliberately engineered, not assumed.

---

**Q: What is a ReAct agent?**

Agent + decision-making + tool calls, in a loop. "ReAct" = **Rea**son + **Act**. The agent reasons about what to do, takes an action (e.g., calls a tool/API), observes the result, and then decides what to do next based on that observation — repeating until the task is complete.

---

**Q: Loop engineering**

Designing the iterative feedback loop an AI agent runs to complete a task: **observe → think → act → observe** — and deciding how that loop terminates, how it recovers from bad steps, and how much autonomy it has at each iteration.

---

**Q: Greenfield project**

Starting from scratch — clearly define requirements and scope, choose the architecture, and set up the repo, environment, and CI/CD from zero, with no existing constraints to work around.

---

**Q: Brownfield project**

Working within an existing system — understand the current codebase, data flow, and dependencies first, identify what's reusable versus what needs to be replaced, and make incremental changes rather than a rewrite.

---

**Q: What is pruning?**

Reducing the size of a neural network or LLM by removing the parts (weights/parameters) that contribute little to its performance — used to make a model smaller, faster, and cheaper to run with minimal accuracy loss.
