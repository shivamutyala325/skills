---
name: pr-description-generator
description: >
  Use this skill whenever a user wants to write, generate, or improve a pull request or merge
  request description. Triggers include: "write a PR description", "generate a PR description",
  "help me write a pull request", "draft a merge request description", "write up this PR",
  "PR description for this diff", "describe this change", "write PR notes", or when a user
  pastes a git diff, bullet points, or mentions a JIRA ticket and wants a formatted PR description.
  Targets Bitbucket Markdown format. Always produces a structured description with What Changed,
  Why, How to Test, and Linked Ticket sections. Apply this skill even if the user just pastes
  a diff or says "write this up" — if it looks like they need a PR description, use it.
---

# PR Description Generator Skill

Generates clean, structured **Bitbucket pull request descriptions** in Markdown from a git diff,
bullet points, or a JIRA ticket reference. Every description follows a consistent team template.

---

## Input Detection

Identify what the user has provided and select the appropriate path:

| Input type | How to handle |
|---|---|
| Git diff / code paste | Extract changes from the diff, infer purpose from context |
| Bullet points / plain description | Expand and structure into the PR template |
| JIRA ticket reference (e.g. `PROJ-123`) | Use ticket title/description as the basis; ask for diff/bullets if not provided |
| Mix of the above | Combine all inputs for maximum context |

If input is too vague to produce a meaningful description, ask one focused question before proceeding.

---

## PR Description Template

Always output descriptions in this exact structure (Bitbucket Markdown):

```markdown
## Summary
<2–3 sentence overview of what this PR does and why. Written for a reviewer who has no prior context.>

## What Changed
<Bulleted list of concrete changes. Be specific — file names, function names, config keys where relevant.>
- 
- 
- 

## Why
<1–3 sentences explaining the motivation. Link to the problem being solved, not just the solution.>

## How to Test
<Step-by-step instructions a reviewer can follow to validate the change works correctly.>
1. 
2. 
3. 

## Linked Ticket
<JIRA ticket link or reference. If not provided, output a placeholder.>
- Ticket: [PROJ-XXX](https://jira.example.com/browse/PROJ-XXX)

## Notes for Reviewer
<Optional. Include only if there is something specific the reviewer should pay attention to —
tricky logic, known limitations, follow-up tickets, or areas of uncertainty. Omit this section
if there is nothing notable.>
```

---

## Writing Guidelines

### Summary
- Written for someone with no context on the change
- Covers: what changed, at what scope, and the expected outcome
- Avoid vague openers like "This PR fixes..." — be specific
- Good: "Adds cursor-based pagination to the `/v1/deployments` list endpoint to prevent memory exhaustion on large clusters."
- Bad: "Fixed some issues with the API."

### What Changed
- One bullet per logical change (not per file)
- Start each bullet with a past-tense verb: Added, Updated, Removed, Fixed, Refactored, Replaced
- Include component/file context where it helps: `Updated HelmChart validator (validators/helm.py) to reject missing readinessProbe`
- Group related changes under a sub-heading if the PR is large (>5 bullets)

### Why
- Focus on the problem, not the implementation
- Reference an incident, customer request, performance issue, or tech debt item if applicable
- If it's a Kubernetes/infrastructure change, note the operational impact

### How to Test
- Numbered steps, written so any team member can follow them
- Include specific commands, endpoints, or config values where relevant
- For infra changes: include kubectl/helm commands to verify
- For API changes: include curl or HTTP client examples
- End with the expected outcome: "You should see X"

### Linked Ticket
- Always include even if placeholder
- Format: `[PROJ-XXX](https://jira.example.com/browse/PROJ-XXX)`
- If multiple tickets: list each on its own line

### Notes for Reviewer
- Only include if genuinely useful
- Good uses: flagging a non-obvious design decision, noting a known limitation, pointing to a follow-up ticket
- Do NOT use for general explanations that belong in Why or What Changed

---

## PR Title

Always suggest a PR title above the description in this format:

```
**Suggested PR Title:** <type>(<scope>): <short description>
```

Types: `feat`, `fix`, `refactor`, `chore`, `docs`, `test`, `infra`
Scope: component or service name (e.g. `api`, `helm`, `cilium`, `auth`)
Example: `feat(api): add cursor pagination to deployments list endpoint`

---

## Tone & Style

- Professional but direct — no marketing language
- Active voice, past tense for changes ("Added", "Removed", not "Adding", "We added")
- Concise — reviewers are busy; every sentence should earn its place
- Specific over generic — "Updated `values.yaml` to set `resources.limits.memory: 512Mi`" beats "Updated config"

---

## Edge Cases

- **Diff is very large (>200 lines)**: Summarize by component/module, don't list every file
- **No JIRA ticket provided**: Output `- Ticket: [PROJ-XXX](https://jira.example.com/browse/PROJ-XXX) ← update this`
- **Pure refactor with no functional change**: Make that explicit in Summary and Why
- **Infra / Kubernetes change**: How to Test should include `kubectl` or `helm` verification steps
- **Breaking change**: Add a `## ⚠️ Breaking Change` section above Notes for Reviewer, clearly stating what breaks and the migration path