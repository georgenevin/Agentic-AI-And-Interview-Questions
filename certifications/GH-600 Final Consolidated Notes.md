# GH-600 Final Governance and Agentic Workflow Notes

This note consolidates the practical GH-600 guidance discussed for governance, approval boundaries, GitHub enforcement, traceability, and agent operations. It is intended as a practical study and implementation summary, not as a replacement for the official exam guide or the current GitHub documentation.

## 1. Copilot CLI interactive mode

To start Copilot CLI in interactive mode with all permissions allowed and all GitHub MCP tools enabled, run:

```bash
copilot --allow-all --enable-all-github-mcp-tools
```

This launches the interactive CLI; it is not a one-shot prompt invocation.

Security warning:
- These flags grant broad capability and may allow consequential actions without normal per-action confirmation.
- Enable all GitHub MCP tools only when they are needed.
- Use this mode only in a trusted environment with a suitable account and repository scope.
- These flags do not override GitHub identity permissions or organization policies.
- Prefer the least-privileged tool set for routine work.

Before running:
- Confirm the current repository and working directory.
- Review which account the CLI is authenticated as and what it can access.
- Avoid using this mode with untrusted repository content or in environments containing production credentials.
- Review any proposed tool actions, changes, and outputs before accepting them.

## 2. End-to-end evaluation flow

A typical GitHub evaluation flow for a task looks like this:

1. An issue defines the task and success criteria.
2. An agent creates a branch and opens a pull request.
3. Workflows run automatically.
4. Status checks appear in the pull request.
5. Required checks must pass.
6. A reviewer approves the change.
7. The pull request is merged.

This ensures that all changes are validated before being accepted.

### Why this matters

This is the practical mechanism that turns a proposal into a reviewable, auditable change. A pull request is not just a way to merge code; it is a control boundary and a record of review, validation, and accountability.

## 3. Example: an idea pull request

Suppose a team proposes adding a repository health report generated weekly by an agent.

A good idea pull request should include:

- Problem and motivation
- Proposal summary
- Scope and non-goals
- Alternatives considered
- Risks and safeguards
- Evaluation plan
- Decision record and ownership

Example structure:

```markdown
## Goal
Propose an automated weekly repository health report for maintainers.

## Motivation
We need a summary of key repository health signals without manual review.

## Scope
- Read-only status summary
- Use scheduled workflow and artifact output
- No automatic issue creation or deployment

## Risks
- Workflow permissions must be narrow
- Report must not expose secrets or untrusted content
- Should not make repository changes without review

## Validation plan
- Check sample repositories and edge cases
- Verify report output is readable and useful
- Require human review before implementation proceeds
```

This type of PR is a proposal record, not an implementation claim. It helps teams assess value, risk, and design before the code is built.

## 4. How the PR acts as the source of truth

Treat the pull request as the authoritative, reviewable record of a proposed change and its validation. The PR brings together:

- the issue or task definition
- the proposal or design context
- the actual diff and commit history
- workflow results and required checks
- review comments and approvals
- the final merge or closure outcome

Important rule:
- The issue is the source of truth for the task, audience, and acceptance criteria.
- The design or proposal record is the source of truth for approved design decisions.
- The pull request is the source of truth for what was actually proposed, reviewed, validated, and accepted.

Example PR summary:

```markdown
## Goal
Implement the approved weekly repository health report from #42.

## Scope
- Add a scheduled read-only workflow
- Publish a summary artifact for maintainers
- No automatic merge or deployment

## Validation
- [ ] Required workflow checks pass
- [ ] Read-only permissions verified
- [ ] Report checked against agreed sample cases

## Risks and limitations
- Report may omit information unavailable to the workflow identity
- It does not change repository content or deploy code

## Decision and evidence
- Approved proposal: #42
- Workflow run: [link]
- Deviations: none
```

This makes the PR practical as both a review boundary and an audit record.

## 5. Autonomy levels

A useful autonomy model for agentic systems:

| Autonomy level | What the agent can do | Example |
|---|---|---|
| Read-only autonomy | Inspect, summarize, classify, and recommend changes; cannot modify anything | Summarize an issue and identify likely files |
| Propose-only autonomy | Create branches and pull requests, but cannot merge or deploy | Draft a change and open a PR |
| Execute with guardrails | Run pre-approved workflows such as tests, builds, or staging deploys within defined limits | Run CI or limited staging validation |
| Human-authorized execution | Perform high-impact actions only after a human approves | Production deployment, workflow changes, secret access |

This model is a teaching aid, not an official GH-600 taxonomy. Use it as risk-based guidance.

## 6. Risk-based approval model

Treat changes to `.github/workflows/`, `infra/`, and `security/` as high risk by default. These are “small diff, big consequence” areas.

### Risk tiers

| Risk | Default handling | Example |
|---|---|---|
| Low risk | Plans can run automatically within approved scope | Documentation and harmless read-only tasks |
| Medium risk | Requires validation through PRs and checks | Feature code and application logic changes |
| High risk | Requires explicit human approval | Workflow edits, infrastructure changes, security policy changes |

Why these paths are high risk by default:
- A small diff can change CI or deployment behavior.
- Workflow and infrastructure edits can widen or weaken access.
- Security rules affect trust boundaries and real operational risk.

## 7. GitHub enforcement model

GitHub enforces governance by applying rules and controls directly within the development workflow.

### Apply rulesets and branch protection

For protected branches such as `main`, enforce:

- PR required to merge
- required checks must pass
- required approving reviews by risk level
- CODEOWNERS review for sensitive paths
- restrict direct pushes
- block force pushes and branch deletion

Example workflow trigger:

```yaml
on:
  pull_request:
    branches: [main]
```

This trigger starts validation; it does not itself guarantee approval or merge enforcement. Use required checks and branch protections to enforce policy.

## 8. Use required checks as validation

Required checks should map to governance goals:

| Goal | Example check |
|---|---|
| Quality | build and unit tests |
| Security risk reduction | secret scans, dependency scans, security checks |
| Policy compliance | workflow or infrastructure validation |

Important guidance:
- Treat renaming or removing required checks as high risk.
- If checks drift, governance drifts.
- Review workflow and repository rules together.

## 9. Route ownership using CODEOWNERS

A CODEOWNERS file can route review to responsible teams:

```text
/security/          @security-team
/infra/             @platform-team
/.github/workflows/ @platform-team
*                   @core-team
```

Important:
- CODEOWNERS is not the same as enforcement.
- Require CODEOWNERS review in rulesets or branch protection, or the ownership map is merely advisory.

## 10. Defense-in-depth guardrails

Do not rely on a single instruction or a single workflow as your only protection. Use a defense-in-depth model:

- least-privilege tool access
- protected branches and rulesets
- required checks
- required human reviews
- CODEOWNERS for sensitive paths
- protected environments for deployment
- job logs and artifacts for traceability

### Example: guardrail workflow to block agent-authored PRs and require human approval

```yaml
name: agent-guardrails

on:
  pull_request:
    branches: ["main"]

permissions:
  contents: read
  pull-requests: read

jobs:
  enforce-human-review:
    runs-on: ubuntu-latest
    steps:
      - name: Fail if PR is authored by Copilot agent
        if: ${{ github.event.pull_request.user.login == 'github-copilot[bot]' }}
        run: |
          echo "PRs authored by github-copilot[bot] must be opened or owned by a human."
          exit 1

      - name: Require at least one human approval
        uses: actions/github-script@v7
        with:
          script: |
            const { owner, repo } = context.repo;
            const pull_number = context.payload.pull_request.number;
            const reviews = await github.paginate(
              github.rest.pulls.listReviews,
              { owner, repo, pull_number, per_page: 100 }
            );

            const approvedByHuman = reviews.some((review) =>
              review.state === "APPROVED" &&
              review.user?.type === "User" &&
              review.user.login !== "github-copilot[bot]"
            );

            if (!approvedByHuman) {
              core.setFailed("Human approval is required before merge.");
            }
```

Important caveats:
- This sample checks one specific bot identity and is illustrative.
- Confirm the actual agent identity or bot naming in your environment.
- A workflow like this should be a required status check in branch rules.
- A human approval must reflect actual review judgment; automation identities should not count as humans.

## 11. Gate deployments with environments

Use environments to:

- require reviewers for production deploys
- restrict production secrets to the production environment
- maintain a clear approval record
- keep deployment behavior separate from general repository access

This reduces accidental exposure and gives explicit deployment boundaries.

## 12. Tool approval and friction reduction

Low-risk, read-only tools often need no extra friction. High-risk tools should require explicit approval at use time.

Example idea:

```bash
copilot --agent=CodeAgent --allow-tool 'read,search'
```

This is only illustrative syntax. Always check the current Copilot CLI documentation and platform behavior. Tool availability is not authorization. The authenticated identity, repo rules, and platform permissions still apply.

The real principle:
- low-risk actions can be streamlined
- high-risk actions require explicit approval and scoping
- broad flags should be avoided for routine work

## 13. Observability model

Every meaningful action should generate evidence.

Required evidence may include:

- PR and commit history
- workflow results and job logs
- uploaded artifacts (plans, test results, execution reports)
- approval and merge events
- environment approval records for production deploys

Observability means an action is visible, audit-friendly, and explainable. It supports debugging, accountability, and trust.

## 14. Operational governance

Governance is not static. It evolves as systems, teams, and risks change.

Operational governance means regularly reviewing:

- permissions
- rulesets and protected branch settings
- workflow behavior
- CODEOWNERS ownership
- environment reviewers and secrets
- failed runs and recurring policy violations

### Common governance drift risks

- required checks renamed or removed
- CODEOWNERS file ignored or outdated
- permissions broadened without review
- secrets exposed to broader scopes
- review requirements weakened over time

### Recommended cadence

- Weekly: review failed runs and recurring policy violations
- Monthly: review workflow permissions and secret scopes
- Quarterly: audit rulesets, CODEOWNERS, environment reviewers, and retention/evidence practices

## 15. Final decision rules for GH-600 exam and real work

Use these rules when making design decisions:

- Prefer least privilege for tools and execution contexts.
- Require review and approval for consequential actions.
- Use GitHub rulesets, required checks, and CODEOWNERS as actual enforcement boundaries.
- Treat `.github/workflows/`, `infra/`, and `security/` as high risk by default.
- Keep agent behavior bounded by repository scope, branch scope, environment scope, and credential scope.
- Preserve auditable evidence for every significant action.
- Review governance periodically; controls drift over time.

## 16. Short summary

A strong agentic workflow has four layers:

1. Clear task definition and success criteria
2. Controlled tool permissions and repository/environment scope
3. Validated execution through pull request checks, reviews, and approvals
4. Observability and governance that keep the system explainable and adjustable over time

This is the practical model behind a safe GitHub-based agent workflow and a solid GH-600 answer framework.
