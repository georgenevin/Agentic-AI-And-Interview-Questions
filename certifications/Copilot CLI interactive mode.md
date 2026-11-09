# Copilot CLI interactive mode

To launch Copilot CLI interactively with all permissions allowed and all GitHub MCP tools enabled, run:

```bash
copilot --allow-all --enable-all-github-mcp-tools
```

This starts an interactive CLI session rather than submitting a one-shot prompt.

> **Security warning:** These flags enable broad capabilities and may allow consequential actions without the usual per-action confirmation. Use them only when the task genuinely requires them, in a trusted environment, with an account and repository scope appropriate to the work. They do not override the authenticated GitHub identity's permissions or organization policies. Prefer the least-privileged tool set for routine tasks.

## Before running

- Confirm the current repository and working directory.
- Check which account the CLI is authenticated as and what it can access.
- Avoid using this mode with untrusted repository content or in environments containing production credentials.
- Review proposed changes and tool actions; broad permissions do not make results safe or correct.
- Verify current options and behavior in the [Copilot CLI documentation](https://docs.github.com/en/copilot/github-copilot-in-the-cli), as CLI options can change.

## GH-600 relevance

This command illustrates Domain 2 (tool use and environment interaction) and Domain 6 (guardrails and accountability): enabling tools is not the same as limiting their authorization. Scope credentials and execution context carefully, and grant only the access needed for the task.
