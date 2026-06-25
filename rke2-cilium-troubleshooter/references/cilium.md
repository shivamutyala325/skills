# Cilium Deep Reference

## Table of Contents
1. [Identity System](#identity-system)
2. [BPF Maps](#bpf-maps)
3. [Policy Enforcement Modes](#policy-enforcement-modes)
4. [Network Policy Advanced](#network-policy-advanced)
5. [eBPF Load Balancer](#ebpf-load-balancer)
6. [Hubble Advanced](#hubble-advanced)
7. [Encryption (WireGuard / IPSec)](#encryption)
8. [Cluster Mesh](#cluster-mesh)
9. [Common Error Messages](#common-error-messages)

---

## Identity System

Cilium assigns a numeric **security identity** to every endpoint based on its Kubernetes labels.

```bash
# List all identities in the cluster
kubectl -n kube-system exec ds/cilium -- cilium identity list

# Get identity for a specific endpoint
kubectl -n kube-system exec ds/cilium -- cilium endpoint list
# Find endpoint ID, then:
kubectl -n kube-system exec ds/cilium -- cilium endpoint get <id>

# Lookup identity by label selector
kubectl -n kube-system exec ds/cilium -- cilium identity get \
  --labels "k8s:app=myapp,k8s:io.kubernetes.pod.namespace=production"
```

**Identity sources (in priority order):**
1. `k8s:` — Kubernetes pod labels
2. `reserved:` — Built-in (world, host, remote-node, kube-apiserver)
3. `cidr:` — IP-based (for entities outside cluster)

**Debugging identity mismatch:**
If policy says `allow app=frontend → app=backend` but traffic is dropped:
```bash
# Confirm backend pod has the right labels
kubectl get pod <backend> -n <ns> --show-labels

# Confirm Cilium endpoint has picked up those labels
kubectl -n kube-system exec ds/cilium -- cilium endpoint list | grep <pod-ip>
# Note the IDENTITY column

# Inspect that identity
kubectl -n kube-system exec ds/cilium -- cilium identity get <identity-id>
```

---

## BPF Maps

```bash
# List all BPF maps
kubectl -n kube-system exec ds/cilium -- cilium bpf maps list

# Connection tracking table
kubectl -n kube-system exec ds/cilium -- cilium bpf ct list global | head -50

# NAT table
kubectl -n kube-system exec ds/cilium -- cilium bpf nat list

# Policy map for an endpoint
kubectl -n kube-system exec ds/cilium -- cilium bpf policy get <endpoint-id>

# Masquerade / SNAT
kubectl -n kube-system exec ds/cilium -- cilium bpf ipmasq list

# LB service/backend maps
kubectl -n kube-system exec ds/cilium -- cilium bpf lb list
kubectl -n kube-system exec ds/cilium -- cilium bpf lb list --backends
```

**Map exhaustion** (drops without obvious policy reason):
```bash
# Check map sizes
kubectl -n kube-system exec ds/cilium -- cilium metrics | grep bpf_map
# Look for: cilium_bpf_map_pressure > 0.9 = near-full map
```

Increase via `cilium-config` ConfigMap:
```yaml
# Example: increase CT table
bpf-ct-global-tcp-max: "524288"   # default 262144
bpf-ct-global-any-max: "262144"   # default 131072
```

---

## Policy Enforcement Modes

| Mode | Behavior |
|---|---|
| `disabled` | No policy enforcement (allow all) |
| `default` | Enforce only if a CNP selects the endpoint |
| `always` | Enforce for all endpoints; default deny if no policy |

```bash
# Check current mode
kubectl -n kube-system get cm cilium-config -o jsonpath='{.data.policy-enforcement-mode}'

# Per-namespace override
kubectl annotate namespace <ns> policy.cilium.io/enforcement-mode=always
```

**Default deny pattern** — the most common source of confusion:
```yaml
# This CNP creates a default-deny for selected pods
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: default-deny
spec:
  endpointSelector: {}   # selects ALL pods in namespace
  ingress: []            # empty = deny all ingress
  egress: []             # empty = deny all egress
```

After applying a default-deny, you must add explicit allow rules for:
- DNS egress (port 53 to kube-dns)
- Health checks from kubelet (port varies)
- Inter-service communication

---

## Network Policy Advanced

### L7 HTTP Policy
Requires Cilium Envoy integration (enabled by default in recent Cilium versions).

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: frontend
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: GET
          path: "/api/v1/.*"
```

Debugging L7 drops:
```bash
# L7 proxy port
kubectl -n kube-system exec ds/cilium -- cilium proxy-stats

# Envoy access log (if enabled)
kubectl -n kube-system exec ds/cilium -- cilium monitor --type l7
```

### FQDN Policy
```yaml
egress:
- toFQDNs:
  - matchName: "api.external.com"
  toPorts:
  - ports:
    - port: "443"
      protocol: TCP
```

```bash
# Check FQDN cache
kubectl -n kube-system exec ds/cilium -- cilium fqdn cache list

# Flush stale FQDN cache
kubectl -n kube-system exec ds/cilium -- cilium fqdn cache clean
```

### toEntities (world, host, kube-apiserver)
```yaml
egress:
- toEntities:
  - world   # allow all external traffic
- toEntities:
  - kube-apiserver  # allow k8s API access
```

---

## eBPF Load Balancer

RKE2 + Cilium replaces kube-proxy entirely. All service load balancing runs in BPF.

```bash
# Verify kube-proxy replacement is active
kubectl -n kube-system exec ds/cilium -- cilium status | grep -i "kube-proxy"
# Expected: KubeProxyReplacement: True

# List services in BPF LB
kubectl -n kube-system exec ds/cilium -- cilium service list

# Show backends for a specific service
kubectl -n kube-system exec ds/cilium -- cilium service list | grep <svc-name>
# Note the service ID, then:
kubectl -n kube-system exec ds/cilium -- cilium bpf lb list | grep <svc-id>

# Maglev consistent hashing (if enabled)
kubectl -n kube-system get cm cilium-config -o jsonpath='{.data.load-balancer-algorithm}'
```

**NodePort DSR (Direct Server Return):** reduces latency for NodePort traffic.
```bash
kubectl -n kube-system get cm cilium-config -o jsonpath='{.data.nodeport-mode}'
# Options: snat (default), dsr, hybrid
```

---

## Hubble Advanced

### Flow filtering
```bash
# All drops in namespace
hubble observe -n <ns> --verdict DROPPED

# Specific source → destination
hubble observe \
  --ip-src <pod-ip> \
  --ip-dst <svc-clusterip> \
  --follow

# By protocol
hubble observe --protocol TCP --port 5432 --follow

# Policy audit mode (see what WOULD be dropped)
hubble observe --verdict AUDIT

# JSON output for grep/jq
hubble observe -n <ns> -o json | \
  jq 'select(.flow.verdict=="DROPPED") | {src: .flow.source, dst: .flow.destination, reason: .flow.drop_reason_desc}'
```

### Drop reason codes (most common)
| Code | Meaning |
|---|---|
| `POLICY_DENIED` | CiliumNetworkPolicy blocked the flow |
| `CT_MISSING_ENTRIES` | Conntrack entry missing (asymmetric routing) |
| `UNSUPPORTED_L3_PROTOCOL` | Non-IP protocol not handled |
| `ENCAPSULATION_TRAFFIC_IS_PROHIBITED` | VXLAN/Geneve misconfiguration |
| `NO_MAPPING_FOR_NAT_MASQUERADE` | BPF NAT table full or SNAT misconfigured |

### Hubble metrics (Prometheus)
```bash
# Metrics endpoint
kubectl -n kube-system port-forward svc/hubble-metrics 9965:9965 &
curl localhost:9965/metrics | grep hubble_drop

# Key metrics to alert on:
# hubble_drop_total{reason="POLICY_DENIED"} — policy violations
# hubble_flows_processed_total — flow volume
```

---

## Encryption

### WireGuard (recommended for RKE2 + Cilium)
```bash
# Check WireGuard status
kubectl -n kube-system exec ds/cilium -- cilium status | grep Encryption
# Expected: Encryption: Wireguard [NodeEncryption: Disabled, cilium_wg0 (Pubkey: ...)]

# WireGuard interface on node
wg show cilium_wg0

# Verify node-to-node encryption
hubble observe --verdict FORWARDED -o json | \
  jq 'select(.flow.is_reply==false) | .flow.ethernet'
```

Enable in cilium-config:
```yaml
encryption: wireguard
enable-wireguard-userspace-fallback: "false"
```

### IPSec
```bash
# Check IPSec keys
kubectl -n kube-system get secret cilium-ipsec-keys

# Rotate keys (without downtime)
kubectl -n kube-system exec ds/cilium -- cilium encrypt status
```

---

## Cluster Mesh

```bash
# Status of all connected clusters
cilium clustermesh status

# Remote endpoints visible
kubectl -n kube-system exec ds/cilium -- cilium endpoint list | grep -i remote

# Global services (span clusters)
kubectl get service -A -l service.cilium.io/global=true

# Connectivity test across clusters
cilium connectivity test --multi-cluster <remote-cluster-name>
```

---

## Common Error Messages

| Message | Cause | Fix |
|---|---|---|
| `Failed to create endpoint: unable to create veth pair` | Stale netns from previous pod | Drain and reboot node; check `ip netns list` |
| `BPF map pressure high` | CT / policy map near-full | Increase map sizes in cilium-config |
| `Unable to contact k8s api-server` | Cilium agent lost API connectivity | Check apiserver CiliumNetworkPolicy + network |
| `Rejecting connection due to missing policy` | Default-deny active, no allow rule | Add explicit ingress/egress CNP |
| `FQDN cache miss` | DNS proxy didn't cache the name | Check CoreDNS → Cilium DNS proxy chain |
| `WireGuard: no such device` | Kernel module missing | `modprobe wireguard`; check kernel ≥ 5.6 |
| `Failed to restore endpoints: missing map` | Cilium restarted, stale BPF maps | Normal on upgrade; verify endpoints recover |
| `Conntrack is full` | High connection rate | Increase CT map size; tune GC interval |
