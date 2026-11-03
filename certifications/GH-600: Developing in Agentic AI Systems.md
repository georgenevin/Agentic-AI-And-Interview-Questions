# GH-600: Developing in Agentic AI Systems
## Revised certification study guide — easy-English edition

**Certification:** GitHub Certified: Agentic AI Developer  
**Exam:** GH-600  
**Primary authority:** Official Microsoft Learn / GitHub GH-600 study guide  
**Purpose:** Exam preparation and practical understanding

> **Study-guide rule:** The six domains and their weights below come from the official GH-600 study guide. Explanations, examples, checklists, and practice questions in this document are teaching aids designed to make those objectives easier to understand. They are not claims about the exact questions on the exam.

---

# 1. Exam overview and study strategy

GH-600 assesses the ability to operate, integrate, supervise, evaluate, and govern AI agents inside production software-development workflows. GitHub is the system of record and control plane.

The official study guide groups the exam into six domains:

| Domain | Official domain | Weight |
|---|---|---:|
| 1 | Prepare agent architecture and SDLC processes | 15–20% |
| 2 | Implement tool use and environment interaction | 20–25% |
| 3 | Manage memory, state, and execution | 10–15% |
| 4 | Perform evaluation, error analysis, and tuning | 15–20% |
| 5 | Orchestrate multi-agent coordination | 15–20% |
| 6 | Implement guardrails and accountability | 10–15% |

**Highest-weight domain:** Domain 2. Give tool selection, tool permissions, MCP, GitHub integration, execution scope, error handling, and traceability extra practice.

The official certification page lists a 120-minute exam. The official study guide states that a score of 700 or greater is required to pass. Confirm current details on the certification page before booking because exam policies can change.

## How to study each topic

For every concept, be able to:
1. Define it in simple language.
2. Explain why it is needed.
3. Choose it in a practical scenario.
4. Identify its security and reliability risks.
5. Explain how GitHub controls or artifacts help manage it.

---

# 2. Agentic AI fundamentals

## 2.1 What is an AI agent?

An AI agent is a software system that uses an AI model to pursue a goal. Depending on its design, it can:
- Interpret an objective.
- Plan one or more steps.
- Select and use tools.
- Observe tool results.
- Decide what to do next.
- Evaluate whether the objective has been achieved.
- Stop, retry, or ask a human for help.

An agent is not necessarily fully autonomous. Its autonomy should be deliberately limited by permissions, policies, and approval requirements.

### Example: fixing a bug

A developer asks an agent to fix a bug in a repository.

1. **Understand:** Read the issue and relevant project instructions.
2. **Plan:** Identify likely files, changes, and tests.
3. **Act:** Inspect code, edit files, and run tests using permitted tools.
4. **Observe:** Read test results and error messages.
5. **Evaluate:** Decide whether the fix meets the acceptance criteria.
6. **Report:** Produce a summary and, if authorized, create a branch or pull request.
7. **Escalate:** Ask a human when requirements are unclear or a risky action needs approval.

## 2.2 Agent vs. LLM vs. workflow

| Term | Meaning | Example |
|---|---|---|
| LLM | Model that generates or interprets content | Explains a code error |
| Tool | A capability the agent can invoke | Search repository, run tests, create an issue |
| Workflow | A predefined sequence of steps | Build → test → deploy |
| Agent | Model-driven system that can choose steps and use tools toward a goal | Investigates a bug, edits code, tests, and reports |
| Multi-agent system | Several agents with defined roles coordinated toward a shared outcome | Planner, developer, tester, security reviewer |

**Exam distinction:** A fixed workflow normally follows configured steps. An agent can select actions based on the objective and observations. An agent can still operate inside a controlled workflow.

## 2.3 Characteristics of agentic systems

- Goal-directed: works toward a defined outcome.
- Multi-step: can perform a sequence of actions.
- Tool-using: can interact with external systems.
- Context-aware: uses relevant instructions and information.
- Adaptive: can change its next step based on results.
- Evaluated: checks results against success criteria.
- Governed: operates within permission and policy boundaries.

Do not assume that every agent has persistent memory, unlimited autonomy, or the ability to learn permanently from every interaction. Those capabilities depend on its design.

## 2.4 The Plan–Act–Evaluate loop

A useful high-level mental model is:

**Plan → Act → Observe → Evaluate → Continue / Stop / Escalate**

- **Plan:** Break the goal into executable steps.
- **Act:** Use approved tools to perform a step.
- **Observe:** Collect the result of the action.
- **Evaluate:** Compare the result with the expected outcome.
- **Continue:** Move to the next step if appropriate.
- **Stop or escalate:** Stop when the goal is met, the plan is invalid, a policy blocks the action, or human judgment is required.

Planning, execution, and evaluation should be distinguishable. This makes the work easier to inspect and control.

---

# 3. Domain 1 — Prepare agent architecture and SDLC processes (15–20%)

Official objectives:
- Integrate agents into the software development lifecycle.
- Identify and mitigate agent anti-patterns.
- Define agent inputs, outputs, and success criteria.
- Separate planning, reasoning, and action.
- Produce structured plans and validate them before execution.
- Configure observability, autonomy, artifacts, and human intervention.

## 3.1 Integrating agents into the SDLC

The software development lifecycle (SDLC) includes activities such as:
1. Requirements and issue analysis.
2. Design and planning.
3. Implementation.
4. Testing.
5. Security review.
6. Code review.
7. Release and deployment.
8. Monitoring and maintenance.

Agents can assist at different stages. For example:

| SDLC stage | Possible agent task | Human responsibility |
|---|---|---|
| Issue analysis | Summarize issue, identify acceptance criteria | Confirm business intent |
| Planning | Propose implementation steps | Review scope and risk |
| Development | Suggest or implement code changes | Review code and design |
| Testing | Run tests and summarize failures | Decide whether coverage is sufficient |
| Security | Run or interpret security checks | Assess material risk and exceptions |
| Pull request | Draft description and summarize changes | Review and approve |
| Deployment | Prepare release information or initiate a controlled workflow | Approve protected deployment when required |

Use agents where tasks are repeatable, bounded, and measurable. Keep human judgment for ambiguous, high-impact, irreversible, or policy-sensitive decisions.

## 3.2 Define agent inputs, outputs, and success criteria

A good agent task specification should define:

- **Objective:** What outcome is required?
- **Inputs:** Issue, repository, files, requirements, instructions, test data.
- **Scope:** Which repositories, branches, files, and environments are allowed?
- **Tools:** Which tools may be used?
- **Constraints:** What must not happen?
- **Outputs:** Code changes, test results, report, issue comment, or pull request.
- **Success criteria:** How will completion be measured?
- **Escalation:** When must the agent stop and ask a person?

### Example task specification

```text
Objective:
Fix the input validation bug described in issue #123.

Inputs:
Issue #123 and the repository's contribution instructions.

Scope:
Work only in the specified repository and create a feature branch.
Do not change deployment configuration.

Allowed actions:
Read source files, edit application code, run unit tests,
and prepare a pull request.

Prohibited actions:
Do not merge the pull request or deploy to production.

Success criteria:
The reported bug is fixed, relevant tests pass, and the
pull request explains the change and test results.

Escalation:
Ask for clarification if the acceptance criteria conflict
or the fix requires a database schema change.
```

## 3.3 Structured planning and plan validation

A structured plan makes an agent's intended work inspectable before it changes anything.

A plan can contain:
- Step number.
- Action.
- Required input.
- Tool to be used.
- Expected output.
- Risk or permission needed.
- Validation condition.

Example:

| Step | Action | Expected evidence |
|---|---|---|
| 1 | Read issue and repository instructions | Requirements summary |
| 2 | Inspect relevant code | List of affected files |
| 3 | Propose a fix | Reviewable implementation plan |
| 4 | Apply approved changes | Diff or branch |
| 5 | Run tests | Test output |
| 6 | Report result | Summary and pull request |

**Important:** A plan is not proof that the task was completed. Validate actual tool results and artifacts. If the plan is invalid, revise it before execution.

## 3.4 Autonomy levels and human intervention

Think of autonomy as a risk-based scale:

| Level | Agent behavior | Example |
|---|---|---|
| Assist | Suggests; human performs action | Suggest a code change |
| Supervised execution | Performs bounded work; human reviews | Create a branch and draft a PR |
| Conditional autonomy | Performs low-risk actions automatically; approval for risky steps | Run tests automatically, request approval before deployment |
| High autonomy | Performs a broader set of approved actions | Automated remediation in a tightly controlled environment |

These labels are a teaching model, not a required GH-600 taxonomy. The exam objective is to classify actions by operational, security, and compliance risk and assign suitable autonomy.

## 3.5 Observability and inspectable artifacts

An agent's work should leave evidence that people and other systems can inspect.

Useful artifacts include:
- Structured plan.
- Changed files and code diff.
- Test output.
- Security scan results.
- Pull request description.
- Workflow logs.
- Approval record.
- Final execution summary.

A pull request is a useful collaboration and review point. It allows humans to inspect changes, discuss them, run required checks, and approve or reject them.

GitHub Actions can automate validation tasks such as builds, tests, and scans. A workflow run and its logs provide evidence of what happened.

## 3.6 Common agent anti-patterns

| Anti-pattern | Why it is a problem | Better design |
|---|---|---|
| One agent does everything | Hard to test, understand, and govern | Use focused responsibilities |
| No explicit objective | Agent may solve the wrong problem | Define objective and acceptance criteria |
| Execute before plan validation | Risky or irrelevant actions may occur | Inspect and validate the plan |
| Unlimited repository access | Increases blast radius | Restrict repository and branch scope |
| No evaluation | Incorrect output may be accepted | Validate outputs and process |
| Too many human approvals | Slows low-risk work | Use risk-based approval |
| No audit trail | Hard to investigate failures | Record plans, tools, outputs, and approvals |
| Repeated retries without limits | Can waste resources or repeat harmful actions | Use bounded retries and escalation |

### Domain 1 exam checklist

You should be able to:
- Select suitable SDLC tasks for agents.
- Identify inputs, outputs, constraints, and success criteria.
- Distinguish planning from execution.
- Validate a plan before allowing actions.
- Select appropriate autonomy and human checkpoints.
- Choose artifacts that make execution inspectable.
- Identify agent anti-patterns.

---

# 4. Domain 2 — Implement tool use and environment interaction (20–25%)

This is the highest-weight domain.

Official objectives:
- Identify and configure tools.
- Configure tool permissions.
- Add MCP servers to agents.
- Configure GitHub remote MCP servers, registries, and allow lists.
- Evaluate and configure execution context.
- Scope agents to repositories and branches.
- Invoke agents in CI workflows.
- Enable controlled actions such as branch and pull request creation.
- Handle environment constraints, errors, retries, rollbacks, escalation, and traceability.

## 4.1 What is a tool?

A tool is an external capability an agent can invoke to retrieve information or perform an action.

Examples:
- Read a repository file.
- Search code.
- Create or update an issue.
- Read a pull request.
- Run a test command.
- Call an approved API.
- Retrieve information from an MCP server.

A model may generate a tool-call request, but the tool implementation performs the actual operation. Tool permissions and execution controls determine whether the operation is allowed.

## 4.2 Selecting and configuring tools

Use this process:

1. Identify the task the agent must perform.
2. Identify the minimum information and actions needed.
3. Select the smallest suitable set of tools.
4. Configure authentication and permissions.
5. Validate tool inputs and outputs.
6. Handle errors and timeouts.
7. Log important tool calls and results.

### Example

For an agent that investigates a GitHub issue:

| Need | Possible tool |
|---|---|
| Read issue details | GitHub issue read capability |
| Search repository | Repository/code search |
| Inspect files | Repository content access |
| Run tests | Approved command or CI workflow |
| Share proposed change | Pull request creation capability |

Do not give an agent write access merely because it needs read access. Separate read, write, merge, and deployment capabilities.

## 4.3 Tool permissions and least privilege

**Least privilege** means granting only the permissions required for the task, for only the necessary scope and duration.

Examples:
- Read-only access for an analysis agent.
- Contents write permission only when code changes are required.
- Pull request permission when the agent must create a PR.
- No production deployment permission for a code-review agent.
- Separate credentials for development and production environments.

Review permissions periodically. Remove unused integrations and credentials.

**Exam trap:** Authentication answers "who or what is connecting?" Authorization answers "what is that identity allowed to do?" A valid credential does not automatically mean the requested action should be permitted.

## 4.4 Model Context Protocol (MCP)

MCP is an open protocol that standardizes how AI applications connect to external context and capabilities.

### Main MCP components

- **MCP host:** The AI application that uses MCP capabilities.
- **MCP client:** The component inside the host that maintains a connection to an MCP server.
- **MCP server:** Exposes capabilities to the client.
- **Tools:** Actions the model can request, such as searching or creating an item.
- **Resources:** Context or data that can be read.
- **Prompts:** Reusable prompt templates exposed by a server.

An MCP server does not automatically grant unlimited access. The host/client, server, authentication system, and underlying service permissions all matter.

### MCP interaction — simplified

```text
User
  |
  v
AI application / MCP host
  |
  v
MCP client
  |
  | MCP request
  v
MCP server
  |
  v
External system / API / data source
  |
  v
MCP response
  |
  v
Agent evaluates result and decides next step
```

### MCP tools, resources, and prompts

| Capability | Purpose | Example |
|---|---|---|
| Tool | Performs an action | Create a GitHub issue |
| Resource | Provides data/context | Read repository documentation |
| Prompt | Provides a reusable prompt template | Generate a code review checklist |

**Remember:** Tools do things; resources provide information; prompts provide reusable instructions or templates.

### MCP transports

MCP supports different ways for clients and servers to communicate. Common concepts include:
- **stdio:** Client launches or communicates with a local server through standard input/output. Often used for local development and processes running on the same machine.
- **Streamable HTTP:** A network transport for remote MCP servers.

Do not assume every MCP server uses the same transport or deployment model. Follow the current MCP and platform documentation for exact configuration.

## 4.5 MCP registries and allow lists

An MCP registry helps users or administrators discover and manage MCP servers. A registry is not the same thing as an authorization system.

An allow list restricts which servers or integrations an agent or organization is permitted to use.

Good governance:
- Use trusted and approved servers.
- Verify server identity and source.
- Allow only required servers and tools.
- Review permissions and usage.
- Remove unused or untrusted integrations.
- Do not expose sensitive systems through an unreviewed public server.

**Important correction to the original notes:** Registering an MCP server in an inventory or registry does not, by itself, deploy the server or make it secure. Hosting, networking, authentication, authorization, and client configuration are separate concerns.

## 4.6 GitHub MCP and agent configuration

For exam preparation, understand the concepts rather than memorizing one configuration file:
- A GitHub remote MCP server exposes GitHub capabilities through MCP.
- An agent must be configured to connect to the server.
- Only approved tools should be enabled.
- The user's or service identity's permissions still constrain access.
- Organization policies and allow lists can limit available integrations.
- Tool results must be validated before the agent relies on them.

Platform-specific configuration details can change. Check the current GitHub documentation before copying JSON or YAML examples into a real environment.

## 4.7 Execution environment

An execution environment is where the agent's code, commands, and tools run.

Examples include:
- Developer workstation.
- Isolated cloud development environment.
- GitHub-hosted runner.
- Self-hosted runner.
- Container or sandbox.
- Controlled enterprise environment.

Choose the environment based on:
- Required repository access.
- Network connectivity.
- Required tools and dependencies.
- Sensitivity of data.
- Required isolation.
- Credential handling.
- Compliance and audit requirements.

### GitHub-hosted vs. self-hosted runners

| Area | GitHub-hosted runner | Self-hosted runner |
|---|---|---|
| Infrastructure | Managed by GitHub | Managed by your organization |
| Maintenance | Less runner infrastructure to maintain | Organization maintains the machine |
| Network access | Depends on supported configuration and connectivity | Can be configured for approved internal network access |
| Control | Less control over underlying machine | More control over machine and network |
| Security responsibility | Still need workflow and permission controls | Additional responsibility for patching, isolation, and hardening |

**Important correction:** A self-hosted runner is not automatically safer. It can increase risk if it has broad access to internal systems or is not isolated and maintained. A public repository running untrusted code on a self-hosted runner can expose the machine and connected network. Apply strict repository trust, runner isolation, and workflow controls.

## 4.8 Repository, branch, and environment scope

An agent's execution context should specify:
- Which repository it may access.
- Which branch it may read or modify.
- Which files or directories are in scope.
- Which workflow can invoke it.
- Which environment it can access.
- Which secrets and tools are available.

Separate development, test, and production environments. Do not give an agent direct production API access when a controlled, audited API or deployment workflow can provide the required operation.

## 4.9 GitHub Actions and agent workflows

GitHub Actions can trigger automation from repository events, such as:
- Push.
- Pull request.
- Issue activity.
- Scheduled event.
- Manual workflow dispatch.

A CI workflow can invoke an approved agent or agent-related process, then run tests, scans, or other validation.

A safe high-level flow:

```text
Pull request opened
       |
       v
Workflow starts
       |
       v
Validate event and permissions
       |
       v
Run agent in scoped environment
       |
       v
Collect output / artifact
       |
       v
Run tests and security checks
       |
       v
Human review if required
       |
       v
Merge or deployment through protected controls
```

A pull request event does not mean the agent should be allowed to merge automatically. Use branch protection or repository rulesets, required checks, code-owner reviews, and protected environment approvals as appropriate.

## 4.10 Credentials and secrets

Use secure credential practices:
- Store secrets in approved secret stores.
- Never hard-code credentials in source code or prompts.
- Use short-lived, narrowly scoped credentials where supported.
- Prefer workload identity/OIDC or managed identity where the platform and target service support it.
- Use GitHub tokens with the minimum required permissions.
- Avoid reusing personal access tokens for automation when a more appropriate identity is available.
- Rotate and revoke credentials when necessary.
- Avoid printing secrets in logs.

**Important correction:** "Always use managed identity" is not a universal rule. Managed identity is an option in supported environments. In GitHub workflows, OIDC federation or appropriately scoped GitHub tokens may be more suitable. Choose the credential mechanism supported by the target service and use least privilege.

## 4.11 Error handling, retries, rollback, and escalation

| Failure | Appropriate response |
|---|---|
| Temporary network timeout | Retry with a limit and backoff |
| Rate limit | Respect retry guidance and wait |
| Invalid tool input | Correct input; do not blindly retry |
| Permission denied | Stop and request authorized permission change |
| Test failure | Inspect output, diagnose, and revise if safe |
| Partial execution | Record completed steps and resume safely |
| Unsafe or irreversible action | Stop and require human authorization |
| Repeated failure | Escalate with logs and evidence |

A retry is suitable for transient failures, not for every error. Use bounded retries, backoff, and idempotency where possible.

A rollback restores a previous known-good state. It is different from retrying. Some operations cannot be safely rolled back, so prevention and approval are important.

### Domain 2 exam checklist

You should be able to:
- Choose tools for a task.
- Apply least-privilege tool permissions.
- Explain MCP host, client, server, tools, resources, and prompts.
- Distinguish stdio and remote HTTP transport at a conceptual level.
- Explain registries and allow lists.
- Select a suitable execution environment.
- Scope an agent to repositories, branches, and environments.
- Explain GitHub Actions integration.
- Select suitable credentials and protect secrets.
- Choose between retry, rollback, and escalation.

---

# 5. Domain 3 — Manage memory, state, and execution (10–15%)

Official objectives:
- Choose short-term, long-term, and external memory.
- Scope memory to task-relevant information.
- Define expiration, pruning, and reset rules.
- Persist progress and decisions as durable artifacts.
- Resume work without repeating steps or diverging from decisions.
- Detect and correct context drift.
- Share state and prevent conflicting or stale context across tools and environments.

## 5.1 Memory vs. state vs. context

These terms are related but not identical.

| Term | Meaning | Typical lifetime |
|---|---|---|
| Context | Information currently available to the model | Current request or context window |
| Working memory | Information needed during the current task | Short-lived |
| Long-term memory | Useful information retained across tasks or sessions | Longer-lived |
| External memory | Information stored outside the model context | Depends on storage policy |
| State | Current execution progress and status | For the duration of a workflow, possibly persisted |

Memory is retained information that may help future work. State records where the current task is and what has already happened.

## 5.2 Types of memory

### Working or short-term memory
Holds information needed for the current task, such as:
- Current objective.
- Relevant files.
- Recent tool outputs.
- Pending decisions.

### Long-term memory
Retains information that is useful across tasks, such as:
- Project conventions.
- Stable user preferences relevant to the agent.
- Approved architectural decisions.

Long-term memory needs governance. Incorrect or stale information can harm future tasks.

### External memory
Stores information outside the model's immediate context, such as:
- Database.
- Search index.
- Document store.
- Repository files.
- Issue or pull request records.

External storage can support persistence and retrieval, but access controls and data lifecycle rules still apply.

### Retrieved memory
The agent retrieves only information relevant to the current task rather than loading all stored information.

## 5.3 Memory lifecycle

1. Decide what information is useful to retain.
2. Define where it will be stored.
3. Apply access controls.
4. Set expiration or review rules.
5. Retrieve only relevant information.
6. Correct, prune, or delete stale information.
7. Reset memory when it is unreliable or no longer appropriate.

**Principle:** Treat memory as curated knowledge, not as an unlimited transcript of every conversation.

## 5.4 Persisting execution state

A durable state record can include:

```json
{
  "task_id": "issue-123",
  "objective": "Fix input validation bug",
  "status": "testing",
  "completed_steps": [
    "Read issue",
    "Inspected validation code",
    "Implemented proposed fix"
  ],
  "pending_steps": [
    "Run integration tests",
    "Prepare pull request summary"
  ],
  "decisions": [
    "Keep the public API contract unchanged"
  ],
  "artifacts": [
    "feature/issue-123-validation"
  ],
  "last_successful_checkpoint": "implementation-complete"
}
```

This is an illustrative structure, not a required GH-600 schema.

Persisting state helps an agent:
- Resume after interruption.
- Avoid repeating completed steps.
- Preserve decisions.
- Coordinate handoffs.
- Explain progress to a human.

## 5.5 Context drift

Context drift occurs when an agent gradually loses alignment with the original objective, constraints, or decisions during a long task.

Common causes:
- Very long interaction history.
- Irrelevant information added to context.
- Old or contradictory instructions.
- Tool outputs that change the agent's assumptions.
- Handoffs that omit important decisions.
- Stale retrieved memory.
- Summaries that lose critical details.

Ways to reduce drift:
- Keep the objective and constraints explicit.
- Use concise, structured checkpoints.
- Persist important decisions outside the conversation.
- Retrieve only task-relevant information.
- Revalidate assumptions at key stages.
- Use versioned requirements and artifacts.
- Ask a human when requirements conflict.

## 5.6 Continuity across tools and environments

When work moves between agents, workflows, or environments:
- Share a stable task identifier.
- Share the latest approved state.
- Include completed and pending steps.
- Link to artifacts and evidence.
- Record decisions and their rationale.
- Identify the owner of the next action.
- Avoid sharing secrets in handoff documents.
- Check that the receiving agent's context is current.

### Domain 3 exam checklist

You should be able to:
- Distinguish context, memory, and state.
- Choose short-term, long-term, and external memory.
- Explain pruning, expiration, and reset.
- Design a useful execution checkpoint.
- Explain how to resume a task.
- Identify and prevent context drift.
- Share state safely between agents.

---

# 6. Domain 4 — Perform evaluation, error analysis, and tuning (15–20%)

Official objectives:
- Define success criteria and operational constraints.
- Identify qualitative and quantitative evaluation signals.
- Align evaluation with development intent.
- Use automated scanning tools to generate evaluation signals.
- Analyze failures through logs, plans, traces, outputs, and artifacts.
- Classify reasoning, tool, context, and environment failures.
- Tune instructions, workflows, memory, tools, and access based on results.

## 6.1 Why evaluate agents?

An agent can produce a plausible answer while making incorrect changes or using an inappropriate tool. Evaluation should check both:
1. **Outcome:** Did the agent achieve the goal?
2. **Process:** Did it act correctly, safely, and efficiently?

## 6.2 Define success criteria before execution

Examples:
- Correctness: Does the change fix the reported issue?
- Completeness: Are all acceptance criteria met?
- Test quality: Did the required tests pass?
- Tool use: Were the appropriate tools selected?
- Policy compliance: Did the agent follow repository and security rules?
- Efficiency: Was the work completed within acceptable time and resource limits?
- Traceability: Can a reviewer understand what happened?

Avoid vague criteria such as "the code looks good." Define observable evidence.

## 6.3 Evaluation signals

| Signal | What it measures | Example evidence |
|---|---|---|
| Task success | Whether the objective was achieved | Acceptance criteria met |
| Accuracy | Correctness of output | Test results and reviewer findings |
| Completeness | Whether required work is present | Checklist coverage |
| Tool correctness | Whether suitable tools were used | Tool-call trace |
| Efficiency | Time and resource use | Duration, token and tool-call counts |
| Policy compliance | Whether rules were followed | Permission and policy logs |
| Reliability | Whether results are consistent | Repeated test runs |
| Security | Whether vulnerabilities or risky changes exist | Security scans and review |

Automated signals can include:
- Unit and integration tests.
- Build checks.
- Static analysis.
- Dependency scanning.
- Secret scanning.
- Code scanning.
- Linting and formatting.
- Policy checks.

A passing scan is useful evidence, but it does not prove that every risk is absent.

## 6.4 Analyze agent failures

Useful evidence:
- Original input and instructions.
- Generated plan.
- Tool calls and parameters.
- Tool responses.
- Execution logs and traces.
- Changed files and diffs.
- Test and scan results.
- Workflow artifacts.
- Human review comments.

### Failure categories

| Category | Meaning | Example |
|---|---|---|
| Reasoning/planning error | Agent chose an incorrect approach | Misunderstood acceptance criteria |
| Tool misuse | Wrong tool or incorrect parameters | Used an update operation instead of read |
| Context error | Missing, stale, or conflicting information | Used outdated API requirements |
| Environment error | Runtime or dependency problem | Test command fails because dependency is missing |
| Permission error | Identity lacks required access | API returns forbidden |
| Evaluation error | Success criteria are incomplete or incorrect | Tests pass but required behavior is missing |

### Root-cause analysis sequence

1. Reconstruct the timeline.
2. Identify the first incorrect assumption or action.
3. Inspect the plan and tool call.
4. Check the tool response and environment.
5. Compare the final result with the success criteria.
6. Classify the root cause.
7. Apply a targeted improvement.
8. Re-run evaluation.

Do not change the prompt automatically when the real issue is a permission, network, or environment problem.

## 6.5 Tuning agent behavior

Possible improvements:
- Clarify instructions and acceptance criteria.
- Improve the structured planning format.
- Add or remove workflow steps.
- Improve memory retrieval.
- Restrict or replace tools.
- Correct tool schemas or input validation.
- Adjust permissions and constraints.
- Add tests and evaluation checks.
- Improve failure handling.

### Continuous improvement loop

```text
Define success criteria
        |
        v
Run agent task
        |
        v
Collect logs, traces, scans, and artifacts
        |
        v
Evaluate outcome and process
        |
        v
Identify root cause
        |
        v
Change prompt / workflow / memory / tools
        |
        v
Re-test against the same criteria
        |
        v
Monitor results
```

Change one important factor at a time when possible. This helps identify which change improved or degraded performance.

### Domain 4 exam checklist

You should be able to:
- Define measurable success criteria.
- Distinguish outcome evaluation from process evaluation.
- Select qualitative and quantitative signals.
- Identify useful GitHub and CI scan artifacts.
- Diagnose reasoning, tool, context, environment, and permission failures.
- Choose the correct tuning action.
- Re-evaluate after every meaningful change.

---

# 7. Domain 5 — Orchestrate multi-agent coordination (15–20%)

Official objectives:
- Apply orchestration patterns.
- Configure isolation for parallel execution.
- Detect and resolve overlapping changes, duplicated work, and contradictory outputs.
- Produce reviewable and auditable artifacts.
- Document decisions, handoffs, and outcomes.
- Diagnose failed, partial, or stalled executions.
- Implement rollback and human-in-the-loop recovery.
- Add, update, replace, and retire agents without losing continuity.

## 7.1 What is a multi-agent system?

A multi-agent system uses multiple agents with defined responsibilities to complete a larger objective.

Example for a software change:

| Agent | Responsibility | Output |
|---|---|---|
| Planner | Break requirement into tasks | Structured plan |
| Developer | Implement code changes | Branch or code diff |
| Tester | Run and analyze tests | Test report |
| Security reviewer | Identify security issues | Findings report |
| Reviewer | Check completeness and quality | Review summary |

The human remains accountable for decisions that require judgment or approval.

## 7.2 When to use multiple agents

Use multiple agents when:
- The task has distinct specialist responsibilities.
- Independent tasks can run in parallel.
- Different tools or expertise are required.
- Validation is needed between stages.
- Work can be divided without creating excessive coordination cost.

Do not create multiple agents just because the technology supports it. A single agent or deterministic workflow may be simpler and more reliable for a small, sequential task.

## 7.3 Orchestration patterns

| Pattern | How it works | Best fit |
|---|---|---|
| Sequential | Agents run in a defined order | Each stage depends on the previous output |
| Parallel | Independent agents work at the same time | Independent analysis or test tasks |
| Hierarchical | Coordinator delegates and gathers results | Complex work with multiple specialists |
| Event-driven | An event triggers an agent or workflow | Issue, PR, or workflow events |

### Sequential example

```text
Planner → Developer → Tester → Security reviewer → Human review
```

### Parallel example

```text
                 ┌→ Test agent ──────────┐
Issue → Planner ─┼→ Security agent ──────┼→ Coordinator
                 └→ Documentation agent ─┘
```

Parallel work is useful only when tasks are sufficiently independent. If agents modify the same files, conflict risk increases.

## 7.4 Coordination and handoffs

A reliable handoff should contain:
- Task ID and objective.
- Assigned responsibility.
- Inputs and constraints.
- Completed work.
- Pending work.
- Decisions and rationale.
- Artifact locations.
- Validation evidence.
- Known risks.
- Next owner or agent.

Prefer structured artifacts over long free-form conversations. Structured artifacts reduce ambiguity and make work easier to audit.

## 7.5 Isolation and concurrency

Agents working in parallel need controlled isolation.

Possible isolation mechanisms:
- Separate branches.
- Separate worktrees.
- Separate containers or sandboxes.
- Separate execution contexts.
- Scoped permissions and credentials.

### Branches vs. worktrees

- A **branch** is a separate line of Git history.
- A **worktree** is a separate working directory connected to the same Git repository, allowing multiple branches to be checked out in separate directories.

Worktrees can reduce working-directory collisions when multiple agents work on separate branches. They do not replace permission isolation, credential isolation, or review.

Apply concurrency controls when multiple agent instances may update shared resources. Use locking, ownership rules, branch protections, or serialized workflow steps when needed.

## 7.6 Coordination artifacts in GitHub

Useful artifacts include:
- Issues for task definition and tracking.
- Pull requests for proposed code changes and review.
- Workflow outputs and logs for execution evidence.
- Repository files for shared specifications and reports.
- Build and test artifacts for validation evidence.
- Discussions or comments for recorded decisions.

A GitHub artifact is not automatically trustworthy just because it exists. Validate its source, contents, and relationship to the task.

## 7.7 Agent conflicts

Common conflicts:
- Two agents edit the same file.
- Agents implement contradictory designs.
- Two agents perform the same work.
- One agent uses stale requirements.
- A downstream agent receives incomplete output.

Mitigation:
- Assign clear file or task ownership.
- Use separate branches or worktrees.
- Define outputs and interfaces before parallel work.
- Require integration checks.
- Use pull request review and branch protection.
- Escalate unresolved design conflicts to a human.

## 7.8 Multi-agent observability and recovery

Record:
- Which workflow started.
- Which agents participated.
- Which inputs each agent received.
- Which tools each agent used.
- Which artifacts each agent produced.
- Which decisions and handoffs occurred.
- Which tests and scans ran.
- Which approvals were granted.
- Where execution failed or stalled.

Recovery patterns:
- Retry a transient failure with a limit.
- Resume from the last valid checkpoint.
- Reassign a failed task to another agent when safe.
- Roll back invalid changes.
- Stop dependent work if its input is invalid.
- Escalate critical or ambiguous failures to a human.

## 7.9 Agent lifecycle management

Agents are software components and should be managed accordingly:
- Version agent instructions and configurations.
- Test changes before production use.
- Preserve execution history and artifacts.
- Document configuration changes.
- Update or replace agents carefully.
- Retire unused agents and revoke their access.
- Preserve audit records needed for investigation.

### Domain 5 exam checklist

You should be able to:
- Choose sequential, parallel, hierarchical, or event-driven orchestration.
- Decide whether a task should use one or multiple agents.
- Isolate parallel work.
- Prevent duplicate effort and conflicting changes.
- Design structured handoffs.
- Use GitHub artifacts for coordination and audit.
- Recover from partial or failed agent execution.
- Manage agent versions and lifecycle.

---

# 8. Domain 6 — Implement guardrails and accountability (10–15%)

Official objectives:
- Classify actions by operational, security, and compliance risk.
- Assign appropriate autonomy levels.
- Identify actions that require human judgment.
- Block actions that violate policy.
- Enforce least-privilege permissions and execution contexts.
- Require authorization or controlled paths for irreversible or sensitive changes.
- Avoid unnecessary approvals that do not materially reduce risk.

## 8.1 Guardrails vs. governance

| Concept | Meaning |
|---|---|
| Guardrails | Technical and procedural boundaries that constrain agent behavior |
| Governance | The policies, roles, oversight, and accountability used to manage agents |

Examples of guardrails:
- Restrict repository and branch access.
- Limit available tools.
- Block access to production secrets.
- Require approval before deployment.
- Prevent merging without required checks.
- Validate inputs and outputs.

Governance defines who sets the rules, who approves exceptions, who monitors activity, and who is accountable.

## 8.2 Risk-based autonomy

Classify actions according to their impact.

| Risk | Example | Typical control |
|---|---|---|
| Low | Read documentation or summarize an issue | Automatic execution with logging |
| Medium | Create a branch or draft a pull request | Scoped permissions and review |
| High | Change security configuration or access sensitive data | Strong controls and explicit approval |
| Critical | Production deployment, destructive operation, or irreversible change | Protected workflow and explicit authorization |

The exact risk level depends on organizational context. A seemingly simple action can be high risk if it affects sensitive data or production systems.

## 8.3 Human-in-the-loop

Human review is important when:
- Production systems may be affected.
- Sensitive information may be exposed.
- Financial, legal, security, or compliance risks exist.
- Requirements are ambiguous.
- The agent's confidence or evidence is insufficient.
- An action is irreversible or difficult to recover from.
- A policy requires human authorization.

Avoid adding approval gates to every harmless step. The goal is to place human judgment at meaningful risk boundaries.

## 8.4 GitHub governance controls

| GitHub control | Purpose |
|---|---|
| Repository rulesets | Enforce repository and branch rules |
| Branch protection | Protect important branches |
| CODEOWNERS | Identify owners who should review specified paths |
| Required status checks | Require CI or validation checks before merge |
| Required pull request reviews | Require review before merging |
| Environments | Control access to deployment environments and configure protection rules |
| Environment approvals | Require reviewers before protected deployment jobs proceed |
| Workflow permissions | Limit the permissions available to workflow jobs or tokens |
| Audit logs | Support investigation of relevant organization and repository activity |

Exact availability and configuration can vary by GitHub plan and organization settings. Verify current product documentation for implementation.

## 8.5 Accountability and auditability

Be able to answer:
- Who initiated the workflow?
- Which agent or identity performed each action?
- Which tools were used?
- What inputs and instructions were supplied?
- What decisions were recorded?
- Which files or resources changed?
- Which checks ran?
- Who approved the action?
- What was the final outcome?

Keep evidence in appropriate logs and artifacts. Do not put secrets or unnecessary personal data in logs.

## 8.6 Responsible operation

A responsible agent system should:
- Protect confidential information.
- Use approved data sources.
- Respect access boundaries.
- Avoid unauthorized actions.
- Provide reviewable evidence.
- Escalate uncertainty and policy conflicts.
- Support human oversight.
- Be monitored and improved over time.

### Domain 6 exam checklist

You should be able to:
- Distinguish guardrails from governance.
- Classify actions by risk.
- Choose an appropriate autonomy level.
- Identify when human approval is necessary.
- Apply least privilege.
- Select relevant GitHub repository and deployment controls.
- Explain auditability and accountability.

---

# 9. Important comparisons for the exam

| Compare | Key distinction |
|---|---|
| Agent vs. workflow | An agent can choose actions based on observations; a workflow follows configured steps. They can work together. |
| Plan vs. execution | A plan describes intended actions; execution performs them. |
| Tool vs. resource in MCP | A tool performs an operation; a resource provides context or data. |
| MCP registry vs. allow list | A registry supports discovery/management; an allow list restricts what is permitted. |
| Authentication vs. authorization | Authentication identifies the caller; authorization determines allowed actions. |
| Memory vs. state | Memory retains useful information; state tracks current task progress. |
| Context vs. memory | Context is information currently available to the model; memory is information retained for possible future use. |
| Retry vs. rollback | Retry attempts an operation again; rollback restores a prior state. |
| Sequential vs. parallel orchestration | Sequential stages depend on order; parallel tasks can run independently. |
| Branch vs. worktree | Branch separates Git history; worktree provides a separate working directory. |
| Guardrails vs. governance | Guardrails constrain actions; governance defines policies, oversight, and accountability. |
| Outcome vs. process evaluation | Outcome checks whether the goal was achieved; process evaluation checks how the agent worked. |
| GitHub-hosted vs. self-hosted runner | GitHub manages hosted runner infrastructure; the organization manages self-hosted runner infrastructure. Neither is automatically secure without controls. |

---

# 10. Practical end-to-end project

## Project: Controlled GitHub issue-to-pull-request agent workflow

**Goal:** Build a conceptual workflow in which agents help resolve a GitHub issue while humans retain control over risky actions.

### Roles

1. Planner agent — analyzes the issue and creates a structured plan.
2. Developer agent — implements an approved change in an isolated branch.
3. Tester agent — runs tests and produces a test report.
4. Security reviewer agent — examines relevant security risks and scan output.
5. Human reviewer — reviews the pull request and decides whether it can be merged.

### Workflow

```text
Issue created
    |
    v
Planner reads issue and repository instructions
    |
    v
Structured plan created
    |
    v
Plan validated
    |
    v
Developer works in isolated branch
    |
    v
Tester runs checks
    |
    v
Security review and scan results
    |
    v
Pull request with evidence
    |
    v
Human review + required checks
    |
    v
Merge through protected repository controls
```

### Security requirements

- Limit the repository and branch scope.
- Give each agent only the permissions it needs.
- Do not expose production secrets.
- Validate tool outputs.
- Use protected branches and required checks.
- Require human approval for sensitive actions.
- Record plans, tool calls, logs, artifacts, and approvals.
- Use bounded retries and defined escalation paths.

### Deliverables

- Agent role definitions.
- Structured task plan.
- Permission matrix.
- Workflow diagram.
- Test report.
- Security findings report.
- Pull request summary.
- Audit and recovery checklist.

---

# 11. Troubleshooting scenarios

| Symptom | Likely area to investigate | Good response |
|---|---|---|
| Agent cannot read a repository | Identity, repository scope, permissions | Verify identity and grant only required read access |
| MCP server is unavailable | Server process, endpoint, transport, network | Check server health, endpoint, transport, and logs |
| MCP tool is not available to agent | Client configuration, allow list, tool exposure | Verify server connection and approved tool configuration |
| Tool returns forbidden | Authorization | Stop; inspect required permissions rather than repeatedly retrying |
| Agent repeats completed work | Missing or stale state | Persist checkpoints and completed steps |
| Agent changes unrelated files | Poor task scope or instructions | Restrict repository, branch, and file scope; improve plan |
| Parallel agents overwrite work | Shared working directory or overlapping ownership | Isolate branches/worktrees and define ownership |
| Agent says tests passed but CI fails | Incorrect evidence or environment mismatch | Inspect actual CI logs and reproduce in the relevant environment |
| Agent keeps retrying a bad request | Incorrect retry policy | Classify the error and use bounded retries only for transient failures |
| Agent's output is plausible but wrong | Weak evaluation criteria | Add measurable acceptance criteria and independent validation |
| Workflow exposes a secret | Secret handling or logging failure | Revoke/rotate the secret, inspect logs, and correct access controls |
| Deployment happens without approval | Environment or workflow protection gap | Add protected environment rules and required reviewers |

---

# 12. Practice questions — exam-style

These are original practice questions based on the official objectives, not real or recalled exam questions.

## Questions

### Q1. Planning and execution
An agent receives a request to update a payment service. The organization wants to inspect the intended changes before any files are modified. What should the system do?

A. Allow the agent to edit all files and review afterward  
B. Require the agent to produce a structured plan and validate it before execution  
C. Give the agent production credentials  
D. Disable evaluation

### Q2. Least privilege
An agent only needs to inspect issues and source files. Which permission approach is best?

A. Give repository administrator access  
B. Give write and deployment access  
C. Grant the minimum required read permissions  
D. Use a developer's personal token with broad access

### Q3. MCP concepts
An agent needs to create a GitHub issue through an MCP server. Which MCP capability is most directly suited to performing this action?

A. Resource  
B. Tool  
C. Prompt  
D. Context window

### Q4. MCP registry
An organization has registered an MCP server but wants to prevent agents from using unapproved servers. What should it configure?

A. A tool allow list or equivalent policy control  
B. More conversation memory  
C. A larger model context window  
D. More parallel agents

### Q5. Execution environment
A workflow needs access to an internal service that is reachable only from an organization's private network. What should be evaluated?

A. Whether a suitably secured self-hosted runner or other approved private execution environment is required  
B. Whether the agent should receive administrator access to every repository  
C. Whether all security checks can be disabled  
D. Whether the repository should be made public

### Q6. Retry behavior
An API call fails because the agent's identity is not authorized. What is the best response?

A. Retry forever  
B. Increase the model temperature  
C. Stop and investigate authorization; request an approved permission change if justified  
D. Roll back the Git repository

### Q7. Memory and state
An agent is interrupted after implementing code but before running tests. What helps it resume reliably?

A. Rely only on the model's hidden reasoning  
B. Persist completed steps, decisions, pending steps, and artifact references  
C. Delete the task history  
D. Restart from the beginning without checking progress

### Q8. Context drift
An agent begins following an old requirement after a long task. What is the best prevention?

A. Add every historical conversation to the prompt  
B. Use current authoritative requirements, structured checkpoints, and relevant retrieval  
C. Remove all success criteria  
D. Give the agent more write permissions

### Q9. Evaluation
The code compiles, but the original bug still occurs. Which conclusion is correct?

A. The task is successful because compilation passed  
B. The agent is successful if it used many tools  
C. The task failed its outcome criteria even though one validation check passed  
D. The pull request should be merged automatically

### Q10. Root-cause analysis
An agent selects the wrong tool because the tool description is ambiguous. What should be improved first?

A. Tool descriptions, instructions, or tool selection constraints  
B. Production database permissions  
C. The number of branches  
D. The runner's screen resolution

### Q11. Parallel agents
Two developer agents need to modify different features in the same repository at the same time. What reduces working-directory collisions?

A. Make both agents edit the same working directory  
B. Use isolated branches and separate worktrees or execution environments  
C. Remove all review requirements  
D. Give both agents repository administrator access

### Q12. Multi-agent handoff
What should a developer agent provide to a tester agent?

A. A long unstructured conversation only  
B. A structured handoff with task scope, changes, decisions, artifacts, and test expectations  
C. Its credentials and secrets  
D. An unrelated repository

### Q13. Governance
Which GitHub control can require successful CI checks before a protected branch accepts a pull request merge?

A. Required status checks in branch protection or repository rulesets  
B. Conversation memory  
C. MCP resource  
D. Model temperature

### Q14. Human approval
Which action most clearly warrants a controlled human approval step?

A. Reading a public README  
B. Summarizing an issue  
C. Deploying a high-impact change to production  
D. Formatting a local test report

### Q15. Accountability
A production defect is traced to an agent-generated code change. Which evidence is most useful?

A. Only the final agent response  
B. Plans, inputs, tool traces, changed files, test results, workflow logs, and approvals  
C. Only the model name  
D. Only the issue title

### Q16. Agent lifecycle
An organization replaces an agent used in a production workflow. What should it do?

A. Delete all previous execution history  
B. Version and validate the replacement, manage rollout, and preserve necessary audit history  
C. Give the new agent unrestricted access  
D. Disable monitoring

## Answer key and explanations

| Question | Answer | Explanation |
|---|---|---|
| 1 | B | A structured plan makes intended actions inspectable before execution. |
| 2 | C | Least privilege means only the access needed for the task. |
| 3 | B | An MCP tool performs an operation such as creating an issue. |
| 4 | A | An allow list restricts which integrations or tools are permitted. |
| 5 | A | Select an environment based on network need and security controls; self-hosted is not automatically safe. |
| 6 | C | Authorization failures are not transient network errors; investigate permissions. |
| 7 | B | Durable state enables reliable continuation from a checkpoint. |
| 8 | B | Current requirements and relevant structured context reduce drift. |
| 9 | C | Passing compilation does not prove the requested behavior works. |
| 10 | A | The cause is ambiguous tool guidance, so improve descriptions or selection constraints. |
| 11 | B | Isolation reduces concurrent working-directory conflicts. |
| 12 | B | Structured handoffs preserve context and evidence. |
| 13 | A | Required checks enforce validation before merge. |
| 14 | C | Production deployment is high impact and should follow protected controls. |
| 15 | B | A complete trace supports root-cause analysis and accountability. |
| 16 | B | Agent configurations should be versioned, validated, and managed as software components. |

---

# 13. Final revision sheet

Memorize these principles:

1. An agent works toward a goal through a loop of planning, action, observation, and evaluation.
2. Define inputs, outputs, scope, constraints, and success criteria before execution.
3. Validate structured plans before allowing risky actions.
4. Use tools only when required and grant minimum permissions.
5. MCP connects AI applications to external tools, resources, and prompts.
6. An MCP registry supports discovery; an allow list restricts permitted integrations.
7. Select execution environments according to workload, network, isolation, and sensitivity.
8. Scope agents to approved repositories, branches, files, and environments.
9. Protect credentials; never hard-code secrets or expose them in logs.
10. Memory retains useful information; state tracks current execution progress.
11. Persist checkpoints so work can resume without losing decisions.
12. Context drift can result from stale, irrelevant, or conflicting information.
13. Evaluate both the final result and the execution process.
14. Diagnose the root cause before tuning an agent.
15. Use multiple agents when responsibilities are meaningfully separable.
16. Use isolation and structured artifacts for multi-agent coordination.
17. Retry transient failures with limits; use rollback for recovery; escalate critical failures.
18. Apply guardrails and human approval according to risk.
19. Use GitHub rulesets, branch protection, CODEOWNERS, required checks, and protected environments where appropriate.
20. Preserve traceability: inputs, plans, tool calls, changes, tests, decisions, and approvals.

---

# 14. Official references

Use these as the source of truth for current exam objectives and product behavior:

1. **GH-600 certification page**  
   https://learn.microsoft.com/en-us/credentials/certifications/agentic-ai-developer/

2. **Official GH-600 study guide**  
   https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/gh-600

3. **Developing in Agentic AI Systems course**  
   https://learn.microsoft.com/en-us/training/courses/gh-600t00

4. **GitHub documentation**  
   https://docs.github.com/

5. **GitHub Actions documentation**  
   https://docs.github.com/en/actions

6. **Model Context Protocol documentation**  
   https://modelcontextprotocol.io/

7. **GitHub Copilot documentation**  
   https://docs.github.com/en/copilot

## Final study advice

Use the official study guide as the exam scope. Learn the decision behind each control, not just its name. In scenario questions, identify the objective, risk, required permissions, execution environment, evidence, and approval boundary before selecting an answer.

**Last verification note:** Product interfaces, configuration syntax, feature availability, and exam objectives can change. Verify implementation details against the current official documentation and the current GH-600 study guide before the exam.

