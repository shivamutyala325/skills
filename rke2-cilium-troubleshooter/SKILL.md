---
name: rke2-cilium-troubleshooter
description: >
  Use this skill whenever the user is troubleshooting an RKE2 Kubernetes cluster with Cilium CNI —
  even if they only describe a symptom, not a root cause. Triggers include: pods stuck in
  Pending/CrashLoopBackOff/OOMKilled/Terminating, services unreachable, DNS failures,
  ingress not routing, nodes NotReady, Cilium policy drops, Hubble flow errors, container
  runtime failures, scheduling problems, or any "my cluster is broken" statement involving RKE2
  or Cilium. Apply a structured SRE workflow: triage → scope → diagnose → isolate → resolve →
  verify. Always consult this skill before improvising Kubernetes debugging steps.
---

# RKE2 + Cilium Troubleshooting Skill

## Core SRE Workflow

Every incident follows this sequence. Never skip triage — symptoms mislead without scope.

```
1. TRIAGE    → What is broken? What is the user impact?
2. SCOPE     → Which layer is failing? (node / runtime / network / policy / app)
3. DIAGNOSE  → Collect targeted evidence from the right layer
4. ISOLATE   → Narrow to a single root cause
5. RESOLVE   → Apply fix with rollback awareness
6. VERIFY    → Confirm recovery; check for regressions
```

---

## Layer Map — Where to Start

| Symptom | Start at |
|---|---|
| Pod stuck Pending | Node readiness → Scheduling |
| Pod CrashLoopBackOff / OOMKilled | Container runtime → App logs |
| Pod Running but service unreachable | Cilium connectivity → kube-proxy replacement |
| DNS NXDOMAIN / timeout | CoreDNS → Cilium DNS proxy |
| Ingress 502 / 503 | Ingress controller → backend service → Cilium policy |
| Node NotReady | kubelet / containerd → RKE2 agent → Cilium node agent |
| Cilium policy drop | Hubble → CiliumNetworkPolicy → identity |
| Intermittent packet loss | Cilium BPF maps → MTU → encapsulation |

---

## 1. Node Readiness

```bash
# Node status overview
kubectl get nodes -o wide

# Describe a NotReady node
kubectl describe node <node-name>

# RKE2 agent status on the node
systemctl status rke2-agent
journalctl -u rke2-agent -n 100 --no-pager

# Containerd status
systemctl status containerd
crictl info

# Cilium node agent
kubectl -n kube-system get pod -l k8s-app=cilium -o wide
kubectl -n kube-system logs -l k8s-app=cilium --tail=100

# Check BPF filesystem mount
mount | grep bpf
ls /sys/fs/bpf/
```

**Common causes:** BPF filesystem not mounted, containerd socket missing, RKE2 agent not rejoined after reboot, kernel version incompatible with Cilium eBPF features.

---

## 2. Pod Failures

```bash
# Quick status scan
kubectl get pods -A | grep -v Running | grep -v Completed

# Detailed pod state
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous   # prior crash
kubectl logs <pod> -n <ns> -c <init-container>

# Resource pressure (OOMKilled / evictions)
kubectl top nodes
kubectl top pods -A --sort-by=memory

# Events (last 30min)
kubectl get events -n <ns> --sort-by='.lastTimestamp' | tail -30
```

**CrashLoopBackOff checklist:**
- Liveness probe misconfigured → check `failureThreshold` and timing
- Config/secret missing → `kubectl describe` shows `CreateContainerConfigError`
- Image pull failure → `ErrImagePull`, check `imagePullSecrets`
- OOMKilled → increase memory limit or profile heap leak

---

## 3. Scheduling Problems

```bash
# Why is a pod Pending?
kubectl describe pod <pod> -n <ns> | grep -A 20 "Events:"

# Node capacity / allocatable
kubectl describe nodes | grep -A 5 "Allocatable:"

# Taint/toleration mismatch
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints

# Affinity / topology
kubectl get pod <pod> -n <ns> -o jsonpath='{.spec.affinity}' | jq .

# PVC pending (common blocker)
kubectl get pvc -A | grep -v Bound
kubectl describe pvc <pvc> -n <ns>
```

---

## 4. Cilium Connectivity

> Read `references/cilium.md` for deep Cilium diagnostics.

```bash
# Cilium overall health
cilium status --verbose                          # from cilium-cli
kubectl -n kube-system exec -ti ds/cilium -- cilium status

# Connectivity self-test (runs ~2 min)
cilium connectivity test

# Check Cilium endpoints
kubectl -n kube-system exec ds/cilium -- cilium endpoint list

# BPF load balancer map
kubectl -n kube-system exec ds/cilium -- cilium bpf lb list

# Policy verdict for a specific flow
kubectl -n kube-system exec ds/cilium -- cilium policy trace \
  --src-k8s-pod <ns>/<pod> --dst-k8s-pod <ns>/<pod> --dport <port>
```

---

## 5. Hubble Observability

```bash
# Enable Hubble if not running
cilium hubble enable

# Port-forward Hubble relay
cilium hubble port-forward &

# Watch live flows
hubble observe --follow

# Filter to a specific pod (drops only)
hubble observe --pod <ns>/<pod> --verdict DROPPED --follow

# Check policy drop reason
hubble observe --namespace <ns> --verdict DROPPED -o json | \
  jq '.flow | {src: .source, dst: .destination, reason: .drop_reason_desc}'

# Service map (requires Hubble UI)
cilium hubble ui
```

---

## 6. Service Connectivity

```bash
# Service and endpoint health
kubectl get svc,ep -n <ns>
kubectl describe svc <svc> -n <ns>

# Verify kube-proxy replacement (RKE2+Cilium uses eBPF lb)
kubectl -n kube-system exec ds/cilium -- cilium status | grep KubeProxyReplacement

# Test connectivity from a debug pod
kubectl run tmp-debug --image=nicolaka/netshoot --restart=Never --rm -it -- \
  curl -v http://<service>.<ns>.svc.cluster.local:<port>

# Check Cilium service entries
kubectl -n kube-system exec ds/cilium -- cilium service list

# NodePort / ExternalTrafficPolicy issues
kubectl get svc <svc> -n <ns> -o jsonpath='{.spec.externalTrafficPolicy}'
```

---

## 7. DNS Resolution

```bash
# Quick DNS test from cluster
kubectl run dns-test --image=busybox:1.28 --restart=Never --rm -it -- \
  nslookup kubernetes.default

# CoreDNS pod health
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=50

# CoreDNS config
kubectl -n kube-system get configmap coredns -o yaml

# Cilium DNS proxy (if enabled)
kubectl -n kube-system exec ds/cilium -- cilium status | grep DNS
kubectl -n kube-system exec ds/cilium -- cilium fqdn cache list

# Check ndots / resolv.conf inside pod
kubectl exec <pod> -n <ns> -- cat /etc/resolv.conf
```

**Common DNS failures in RKE2+Cilium:**
- Cilium DNS proxy intercept misconfigured for external FQDNs
- CoreDNS `forward` plugin pointing to unreachable upstream
- `ndots:5` causing excessive search-domain lookups → increase timeout or reduce ndots
- FQDN NetworkPolicy blocking DNS egress on port 53

---

## 8. Ingress Routing

```bash
# Ingress status
kubectl get ingress -A
kubectl describe ingress <name> -n <ns>

# Ingress controller pods (NGINX / Traefik)
kubectl get pods -n <ingress-ns>
kubectl logs -n <ingress-ns> <controller-pod> --tail=100

# Backend service reachable?
kubectl get ep -n <ns> <backend-svc>

# TLS cert issues
kubectl get secret <tls-secret> -n <ns>
openssl s_client -connect <host>:443 -servername <host> </dev/null 2>&1 | grep -E "subject|issuer|expire"

# Cilium policy blocking ingress controller → backend
hubble observe --namespace <ns> --verdict DROPPED --follow
```

---

## 9. CiliumNetworkPolicy

> Read `references/cilium.md` → "Network Policy" section for policy syntax and identity debugging.

```bash
# List all CNPs and CCNPs
kubectl get ciliumnetworkpolicy,ciliumclusterwidenetworkpolicy -A

# Policy affecting a pod
kubectl -n kube-system exec ds/cilium -- cilium endpoint get <endpoint-id> | \
  jq '.[].status.policy'

# Trace policy decision
kubectl -n kube-system exec ds/cilium -- cilium policy trace \
  --src-k8s-pod <ns>/<src-pod> \
  --dst-k8s-pod <ns>/<dst-pod> \
  --dport <port>/TCP

# Identity labels for a pod
kubectl -n kube-system exec ds/cilium -- cilium identity get <identity-id>
```

**Policy debugging tips:**
- Default deny? Check for a `CiliumNetworkPolicy` with empty `spec.ingress` or `spec.egress`
- Label mismatch: Cilium uses pod labels for identity — verify with `cilium endpoint list`
- L7 policy (HTTP/gRPC) requires Envoy proxy sidecar injection by Cilium

---

## 10. Container Runtime (containerd / crictl)

```bash
# List running containers
crictl ps

# Pod sandbox status
crictl pods
crictl inspectp <pod-id>

# Container logs via crictl
crictl logs <container-id>

# Image pull issues
crictl pull <image>
crictl images | grep <image>

# RKE2 embedded registry mirrors (if configured)
cat /etc/rancher/rke2/registries.yaml

# containerd config
cat /var/lib/rancher/rke2/agent/etc/containerd/config.toml
```

---

## 11. Cluster Networking Failures (MTU / Encapsulation)

```bash
# Cilium encapsulation mode
kubectl -n kube-system exec ds/cilium -- cilium status | grep Encapsulation

# MTU configured
kubectl -n kube-system get configmap cilium-config -o yaml | grep -i mtu

# Test MTU between nodes (run from two different nodes)
ping -M do -s 1450 <node-ip>   # adjust size; drop = MTU issue

# Check for ICMP black holes (cloud environments)
kubectl -n kube-system exec ds/cilium -- cilium bpf tunnel list

# IPSec / WireGuard encryption status
kubectl -n kube-system exec ds/cilium -- cilium status | grep Encryption
```

---

## Quick Reference: Key File Paths on RKE2 Nodes

| Resource | Path |
|---|---|
| RKE2 config | `/etc/rancher/rke2/config.yaml` |
| kubeconfig (server) | `/etc/rancher/rke2/rke2.yaml` |
| RKE2 data dir | `/var/lib/rancher/rke2/` |
| containerd socket | `/run/k3s/containerd/containerd.sock` |
| Cilium config map | `kubectl -n kube-system get cm cilium-config` |
| etcd snapshots | `/var/lib/rancher/rke2/server/db/snapshots/` |
| RKE2 server logs | `journalctl -u rke2-server` |
| RKE2 agent logs | `journalctl -u rke2-agent` |

---

## Escalation Checklist

Before escalating, confirm you have collected:
- [ ] `kubectl get nodes -o wide` output
- [ ] `cilium status --verbose` output
- [ ] Hubble flows for the affected namespace (DROPPED verdict)
- [ ] `kubectl describe pod` for failing pods
- [ ] `journalctl -u rke2-{server,agent}` from affected nodes
- [ ] Cilium version: `cilium version`
- [ ] RKE2 version: `rke2 --version`
- [ ] Kernel version: `uname -r` (BPF features are kernel-dependent)

---

## Reference Files

- `references/cilium.md` — Deep Cilium internals: BPF maps, identity system, policy enforcement modes, Hubble advanced, eBPF load balancer, WireGuard/IPSec, Cluster Mesh
