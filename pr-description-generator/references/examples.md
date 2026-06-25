# PR Description Examples

Reference examples for common PR types your team will encounter.

---

## Example 1 — Python API Change

**Input:** "Added pagination to the deployments API endpoint, was causing OOM on large clusters. PROJ-442"

**Output:**

**Suggested PR Title:** `feat(api): add cursor pagination to deployments list endpoint`

---

## Summary
Adds cursor-based pagination to the `GET /v1/deployments` endpoint to prevent out-of-memory
errors when fetching large deployment sets. Previously the endpoint returned all records in a
single response, causing OOM kills on clusters with 500+ deployments.

## What Changed
- Added `cursor` and `limit` query parameters to `GET /v1/deployments`
- Updated `DeploymentRepository.list()` to accept cursor-based pagination args
- Added `PaginatedResponse` envelope to the endpoint response schema
- Updated OpenAPI spec to reflect new query parameters and response shape
- Added unit tests for pagination boundary conditions in `tests/test_deployments.py`

## Why
The `/v1/deployments` endpoint was loading all deployment records into memory before returning
a response. On clusters with 500+ deployments this caused consistent OOM kills on the API pod.
Cursor-based pagination caps memory usage per request regardless of total record count.

## How to Test
1. Deploy the change to a dev cluster: `helm upgrade api ./charts/api --set image.tag=<branch-tag>`
2. Seed the cluster with 100+ deployments or use the existing staging environment
3. Call the endpoint without pagination: `curl -H "Authorization: Bearer $TOKEN" https://api.dev.example.com/v1/deployments`
4. Verify the response includes a `pagination` object with `hasMore` and `nextCursor`
5. Call again with the cursor: `curl "...?cursor=<nextCursor>&limit=20"`
6. Confirm the second page returns the next set of deployments and a new cursor
7. Confirm the final page returns `"hasMore": false` and `"nextCursor": null`

## Linked Ticket
- Ticket: [PROJ-442](https://jira.example.com/browse/PROJ-442)

---

## Example 2 — Kubernetes / Helm Infrastructure Change

**Input:** "Updated the API Helm chart to add readiness and liveness probes. Fixes flapping during deploys."

**Output:**

**Suggested PR Title:** `infra(helm): add readiness and liveness probes to api deployment`

---

## Summary
Adds `readinessProbe` and `livenessProbe` to the API Helm chart to prevent traffic from
being routed to pods that aren't ready and to automatically restart stuck pods. This resolves
intermittent 502 errors observed during rolling deployments.

## What Changed
- Added `readinessProbe` (HTTP GET `/healthz`, initial delay 10s, period 5s) to `charts/api/templates/deployment.yaml`
- Added `livenessProbe` (HTTP GET `/healthz`, initial delay 30s, period 10s) to the same template
- Exposed probe configuration as `values.yaml` fields (`readinessProbe.*`, `livenessProbe.*`)
- Updated `charts/api/values.yaml` with sensible defaults for both probes
- Added probe config documentation to `charts/api/README.md`

## Why
Rolling deployments were routing traffic to new pods before they completed startup, causing
a spike of 502 errors on every deploy. Adding a readiness probe ensures Kubernetes only adds
pods to the load balancer once they pass the health check. The liveness probe ensures stuck
pods are automatically restarted without manual intervention.

## How to Test
1. Deploy to dev: `helm upgrade api ./charts/api --set image.tag=<branch-tag> -n dev`
2. Watch pod rollout: `kubectl rollout status deployment/api -n dev`
3. Confirm probes are configured: `kubectl describe pod -l app=api -n dev | grep -A5 "Liveness\|Readiness"`
4. Verify no 502s during rollout by watching ingress logs: `kubectl logs -l app=ingress -n dev -f`
5. Confirm a pod in non-ready state is not added to endpoints: `kubectl get endpoints api -n dev`

## Linked Ticket
- Ticket: [PROJ-XXX](https://jira.example.com/browse/PROJ-XXX) ← update this

---

## Example 3 — Pure Refactor

**Input:** diff showing rename of internal utility functions, no functional change

**Output:**

**Suggested PR Title:** `refactor(core): rename internal validator utilities for consistency`

---

## Summary
Renames internal validator utility functions across the `core/` module to follow the team's
established naming convention. No functional changes — this is a pure rename refactor to
improve codebase consistency before the v2 API work begins.

## What Changed
- Renamed `validate_helm_chart()` → `helm_chart_validator()` in `core/validators.py`
- Renamed `check_k8s_manifest()` → `k8s_manifest_validator()` in `core/validators.py`
- Updated all call sites across `api/`, `workers/`, and `tests/` to use new names
- No changes to function signatures, return types, or behaviour

## Why
The existing naming mixed verb-first (`validate_*`, `check_*`) and noun-first (`*_validator`)
conventions inconsistently. Standardising on noun-first (`<subject>_validator`) before the v2
API work makes the module easier to navigate and reduces onboarding friction.

## How to Test
1. Run the full test suite: `pytest tests/ -v`
2. Confirm all tests pass with no changes to test logic
3. Search for old names to confirm no call sites were missed: `grep -r "validate_helm_chart\|check_k8s_manifest" .`

## Linked Ticket
- Ticket: [PROJ-XXX](https://jira.example.com/browse/PROJ-XXX) ← update this