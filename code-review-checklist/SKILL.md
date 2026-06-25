---
name: code-review-checklist
description: >
  Use this skill whenever a user wants to review code, a pull request, a diff, or any infrastructure/application file for quality, safety, or correctness. Triggers include: "review this code", "check this PR", "review my YAML/Helm/Terraform", "give me feedback on this", "what's wrong with this", "code review", "review before merge", or when a user pastes code and asks for feedback. Covers Python and Kubernetes/Infrastructure (YAML, Helm, Terraform). Always produces inline GitHub/GitLab-style review comments with severity ratings. Apply this skill even if the user just pastes code without explicitly asking for a review — if it looks like they want feedback, use it.
---

# Code Review Checklist Skill

Produces inline GitHub/GitLab-style review comments for **Python** and **Kubernetes/Infrastructure (YAML, Helm, Terraform)** code. Every comment includes a severity, location, and actionable suggestion.

---

## Output Format

Always produce comments in this structure:

\`\`\`
**[SEVERITY] filename.ext, line N (or lines N–M)**
**Category:** <category name>
**Issue:** <what's wrong and why it matters>
**Suggestion:**
\`\`\`<language>
<corrected code snippet>
\`\`\`
\`\`\`

Severity levels:
- 🔴 **CRITICAL** — Must fix before merge (security exposure, data loss risk, broken rollback)
- 🟠 **WARNING** — Should fix (performance, missing tests, unclear logic)
- 🟡 **SUGGESTION** — Nice to have (style, docs, minor improvements)

End every review with a **Summary Table**:

| Severity | Count |
|----------|-------|
| 🔴 Critical | N |
| 🟠 Warning | N |
| 🟡 Suggestion | N |
| **Total** | **N** |

And a one-line **Merge Recommendation**: ✅ Approve / ⚠️ Approve with fixes / 🚫 Request changes.

---

## Review Checklist by Language

### Python

Read \`references/python-checklist.md\` for the full checklist.

Key areas to always check:
1. **Security & Secrets** — hardcoded credentials, unsafe deserialization, injection risks
2. **Performance** — N+1 queries, unbounded loops, missing pagination
3. **Test Coverage** — untested public functions, missing edge cases, mocked vs real assertions
4. **Documentation** — missing docstrings on public functions/classes, unclear variable names
5. **Coding Standards** — PEP8 compliance, type hints, error handling patterns
6. **Rollback Safety** — DB migrations reversible, feature flags present for risky changes

### Kubernetes / Infrastructure (YAML, Helm, Terraform)

Read \`references/infra-checklist.md\` for the full checklist.

Key areas to always check:
1. **Security & Secrets** — secrets in plaintext, overly permissive RBAC, missing NetworkPolicy
2. **Resource Limits** — missing CPU/memory requests and limits, no HPA defined
3. **Rollback Safety** — missing RollingUpdate strategy, no PodDisruptionBudget, no readiness probe
4. **Documentation** — undocumented Helm values, missing description fields in Terraform variables
5. **Coding Standards** — consistent label schema, DRY Helm templates, Terraform module structure
6. **Performance** — oversized resource requests, missing node affinity for large workloads

---

## Workflow

1. **Identify the file type(s)** — Python, YAML, Helm, Terraform, or mixed PR
2. **Load the relevant reference checklist** for that file type
3. **Scan top-to-bottom**, applying every checklist item
4. **Emit one comment block per issue** in the output format above
5. **Group comments by file** if reviewing a multi-file PR
6. **Append the Summary Table and Merge Recommendation** at the end

---

## Tone & Style

- Be direct and specific — cite the exact line and what to change
- Explain *why* something is an issue, not just *what* is wrong
- For CRITICAL issues, always include a corrected code example
- For SUGGESTION items, keep it brief — one sentence is fine
- Match the assertiveness of the severity — don't soften CRITICALs

---

## Edge Cases

- **No code provided**: Ask the user to paste the code or diff
- **Mixed PR (Python + YAML)**: Apply both checklists, group comments by file
- **Terraform modules**: Check for hardcoded values that should be variables
- **Helm subcharts**: Note if a value override is missing from the parent values.yaml
- **Large diffs (>300 lines)**: Prioritize CRITICAL and WARNING; note that SUGGESTION pass may be partial
