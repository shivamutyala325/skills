# Kubernetes / Infrastructure Code Review Checklist

Covers: Kubernetes YAML manifests, Helm charts, Terraform modules

## 1. Security & Secrets Exposure

### Kubernetes / Helm
- [ ] No plaintext secrets in manifests — use Secret objects or external secret managers
- [ ] Secret values are base64-encoded, not plaintext
- [ ] No secrets passed as env vars directly — reference secretKeyRef
- [ ] RBAC roles follow least-privilege — no * verbs on sensitive resources
- [ ] ClusterRoleBinding used only when namespace-scoped RoleBinding won't suffice
- [ ] NetworkPolicy defined to restrict ingress/egress (Cilium environments)
- [ ] Containers not running as root (runAsNonRoot: true)
- [ ] allowPrivilegeEscalation: false in securityContext
- [ ] readOnlyRootFilesystem: true where possible
- [ ] No hostNetwork/hostPID/hostIPC without justification

### Terraform
- [ ] No hardcoded credentials or passwords in .tf files
- [ ] Sensitive variables marked sensitive = true
- [ ] State backend configured with encryption at rest
- [ ] IAM policies follow least-privilege

## 2. Performance & Resource Limits

### Kubernetes / Helm
- [ ] All containers have resources.requests and resources.limits defined
- [ ] CPU/memory limits are realistic
- [ ] HPA defined for workloads expecting variable load
- [ ] LimitRange or ResourceQuota in place at namespace level
- [ ] Storage class and volume sizes appropriate

### Terraform
- [ ] Instance types right-sized for the workload
- [ ] Auto-scaling groups configured where applicable

## 3. Rollback Safety & Operational Readiness

### Kubernetes / Helm
- [ ] Deployment uses RollingUpdate strategy with maxUnavailable and maxSurge set
- [ ] readinessProbe defined
- [ ] livenessProbe defined
- [ ] startupProbe defined for slow-starting containers
- [ ] PodDisruptionBudget defined for critical workloads
- [ ] terminationGracePeriodSeconds set appropriately (not 0)
- [ ] revisionHistoryLimit set to retain at least 3 previous ReplicaSets
- [ ] ConfigMap/Secret changes trigger pod restarts

### Terraform
- [ ] prevent_destroy = true on critical resources
- [ ] Plan reviewed for unexpected resource replacements

## 4. Documentation & Comments

### Kubernetes / Helm
- [ ] Helm values.yaml has comments explaining non-obvious parameters
- [ ] Chart.yaml has accurate description, version, and appVersion
- [ ] README present for non-trivial Helm charts

### Terraform
- [ ] All input variables have description fields
- [ ] All output values have description fields
- [ ] Module README documents usage, inputs, outputs, examples

## 5. Coding Standards & Style

### Kubernetes / Helm
- [ ] Consistent label schema (app, component, version, managed-by)
- [ ] app.kubernetes.io/ label prefix used
- [ ] Helm templates use {{ include }} not {{ template }}
- [ ] No hardcoded values in templates that should be in values.yaml
- [ ] _helpers.tpl used for repeated template fragments
- [ ] Indentation consistent (2-space YAML)

### Terraform
- [ ] Resources, variables, outputs follow snake_case naming
- [ ] Repeated blocks use for_each or count, not copy-paste
- [ ] terraform fmt applied

## 6. Cilium-Specific (RKE2 environments)
- [ ] CiliumNetworkPolicy preferred over plain NetworkPolicy for L7 rules
- [ ] toEndpoints uses label selectors, not IP-based rules where possible
- [ ] Egress rules explicitly allow DNS (port 53) where needed
- [ ] hubble annotations present if flow observability needed
- [ ] ClusterMesh service references use correct namespace and cluster name
- [ ] WireGuard/IPSec encryption settings not accidentally overridden
