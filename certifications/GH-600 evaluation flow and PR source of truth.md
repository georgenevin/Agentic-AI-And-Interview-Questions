# GH-600 addendum: Evaluation flow and pull requests as the project record

This companion section expands the GH-600 study guide's coverage of evaluation, GitHub Actions checks, pull-request review, and traceability.

## End-to-end evaluation flow

A typical issue-to-pull-request evaluation flow in GitHub looks like this:

1. **An issue defines the task.** Record the problem or idea, context, scope, constraints, and measurable success or acceptance criteria. Link relevant design decisions and supporting evidence.
2. **An agent works in an isolated branch.** It inspects the task and repository instructions, proposes a plan when appropriate, makes a bounded change, and opens a pull request. It must not bypass repository protections.
3. **Workflows run automatically.** GitHub Actions can build and test the proposed changes and run configured lint, security, or policy checks.
4. **Status checks appear on the pull request.** Reviewers can see the outcome and inspect workflow logs or artifacts for evidence. A reported pass should be verified against the actual checks and their scope.
5. **Required checks must pass.** Branch protection or repository rulesets can prevent merging until required checks and any required reviews are complete. A workflow that merely runs is not necessarily a required check.
6. **A reviewer evaluates the change.** The reviewer checks the goal, plan, diff, test and scan evidence, risks, and whether acceptance criteria are met. CODEOWNERS or other required reviewers may apply.
7. **The pull request is merged through the protected process.** The merge records the accepted change in repository history. Deployment, if any, remains subject to its own protected environment and approval controls.

This process provides validation evidence and review before a change is accepted. Passing checks alone do not prove that the change meets the business goal, and a reviewer should still assess the change itself.

## Example: an idea pull request

Suppose a team proposes adding a repository health report generated weekly by an agent. The idea is initially only a proposal, not an authorized implementation request.

An **idea or RFC pull request** can document:

- **Problem and motivation:** What maintainer need does the report address?
- **Proposal:** What should the report include, and where should it appear?
- **Alternatives:** What other approaches were considered, and why were they not selected?
- **Scope and non-goals:** Which repositories, data, and actions are included or excluded?
- **Risks and safeguards:** What permissions, MCP tools, workflow triggers, and outputs are needed? How will secrets and untrusted issue content be handled?
- **Evaluation plan:** What representative cases, success criteria, and failure handling will demonstrate that the report is useful and safe?
- **Decision record:** Who reviewed the proposal, what was approved, and what remains unresolved?

For an idea-only PR, keep the change to proposal documents and examples. Do not silently include production automation or privileged credentials. Mark the PR clearly as a proposal and require the appropriate maintainers to approve the design before implementation begins.

After the proposal is accepted, implementation can be tracked in a linked issue (or a clearly scoped implementation PR). The implementation PR should link back to the accepted proposal and state any deviations. Required checks, review, and merge controls still apply.

## How the PR acts as a source of truth

Treat the pull request as the **authoritative, reviewable record of a proposed code change and its validation**. It brings the relevant material together:

- links to the issue and the accepted design or RFC;
- the implementation plan, assumptions, and scope;
- the actual diff and commits;
- workflow status checks, test output, and scan evidence;
- reviewer feedback, approvals, and any resolved decisions;
- the final merge or closure outcome.

Keep the issue as the source of truth for the task's problem, requirements, and acceptance criteria. Keep an accepted RFC or design record as the source of truth for approved design decisions. The PR is the source of truth for what code was proposed, reviewed, validated, and ultimately merged. If requirements or design change, update the authoritative issue or design record and link the change; do not rely on an undocumented PR comment or agent conversation as the only record.

### Example PR summary

```markdown
## Goal
Implement the approved weekly repository health report from #42.

## Scope
- Add a scheduled, read-only report workflow.
- Publish the report as a workflow artifact for maintainer review.
- No automatic issue creation, merging, or deployment.

## Validation
- [ ] Required workflow checks pass
- [ ] Read-only permissions verified
- [ ] Report checked against the agreed sample cases

## Risks and limitations
- The report may omit information unavailable to the workflow identity.
- It does not make repository changes.

## Decision and evidence
- Approved proposal: #42
- Relevant workflow run: [link]
- Deviations from proposal: None
```

This structure makes the PR a practical handoff and audit record while preserving clear ownership of requirements, design, and implementation.
