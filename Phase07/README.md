# Phase 07 — Vault HA, Kubernetes Auth and Secret Rotation

## Goal
Provide HA secret management with:
- Vault HA
- Raft storage
- Kubernetes authentication
- least-privilege policies
- short-lived credentials
- Vault Agent injection
- secret rotation

## Storage
The cluster initially had no StorageClass.

For the lab, Rancher Local Path Provisioner was installed:
```text
StorageClass: local-path
Provisioner:  rancher.io/local-path
```

> This is lab-grade storage, not production-grade replicated storage.

## Vault Deployment
HashiCorp chart:
```text
Chart: 0.34.1
Vault: 2.0.4
```

Configuration:
```text
3 replicas
HA enabled
Integrated Storage / Raft
```

PVCs:
```text
5Gi data per node
2Gi audit per node
```

## Scheduling Issue
Default Vault required anti-affinity allowed only one Vault server per node.

With two schedulable workers and three tainted control-plane nodes:
```text
vault-0 -> worker-01
vault-1 -> worker-02
vault-2 -> Pending
```

Scheduler:
```text
2 node(s) didn't match pod anti-affinity rules
3 node(s) had untolerated taint(s)
```

For the lab, affinity was disabled.

### Important Kubernetes Detail
Removing affinity from the StatefulSet template did not modify existing pods. Old `vault-0` and `vault-1` still carried the old scheduling rules and blocked `vault-2`.

Recreating old pods solved the issue.

## PVC Final State
All six PVCs became Bound:
```text
data-vault-0/1/2   5Gi
audit-vault-0/1/2  2Gi
```

## Initialization and Unseal
Shamir:
```text
shares:    5
threshold: 3
```

Initialization output was stored at:
```text
/root/vault-init.json
```

and protected:
```bash
chmod 600 /root/vault-init.json
```

Never commit unseal keys/root tokens to Git.

## Raft HA
Final peers:
```text
vault-0 leader
vault-1 follower
vault-2 follower
```

All are voters in the same Vault cluster.

## Kubernetes Auth
Enabled:
```text
kubernetes/
```

API endpoint:
```text
https://kubernetes.default.svc:443
```

Vault ServiceAccount:
```text
vault:vault
```

It was granted `system:auth-delegator` for TokenReview.

Validation:
```text
can-i create tokenreviews.authentication.k8s.io -> yes
```

## App Identities
Dedicated ServiceAccounts:
```text
apps/producer-vault
apps/consumer-vault
```

## KV v2
Mount:
```text
abrnoc/
```

Application path:
```text
abrnoc/data/apps
```

## Least-Privilege Policy
Applications only get read access to:
```text
abrnoc/data/apps
```

## Kubernetes Role
Role:
```text
abrnoc-apps
```

Bound to:
```text
producer-vault
consumer-vault
namespace: apps
```

Token lifetime:
```text
TTL:     15m
Max TTL: 30m
```

A real Kubernetes login successfully returned a 15-minute Vault token.

## Token Hygiene
A test token was printed during validation and treated as exposed. Its accessor was revoked.

## Vault Agent Injector
The injector was temporarily disabled while working around a Helm webhook ownership conflict, then re-enabled after server configuration was stable.

Final injector:
```text
Deployment: 1/1 Running
Webhook:    vault-agent-injector-cfg
```

## Producer Injection
Producer uses:
```text
serviceAccountName: producer-vault
```

Vault annotations inject:
```text
/vault/secrets/abrnoc.env
```

The resulting pod contained:
```text
istio-init
istio-proxy
vault-agent-init
producer
vault-agent
```

This confirms Istio and Vault Agent injection coexist correctly.

## Secret Rotation
The secret moved from KV version 1 to version 2.

The injected file later reflected the rotated secret without embedding it in the Deployment manifest.

For faster lab validation, the static secret render interval can be configured explicitly with the Vault Agent annotation.

## Important Application Caveat
Updating a mounted secret file does not automatically mean an application reloads it into memory.

A production application should explicitly:
- watch/re-read the file, or
- use an agent exec/reload hook, or
- restart safely under a rotation controller.

## Final State
```text
vault-0 Running / active
vault-1 Running / standby
vault-2 Running / standby
```

Capabilities proven:
- Raft HA
- Kubernetes Auth
- short-lived credentials
- least-privilege policy
- Vault Agent injection
- rotated secret delivery
- no static Vault token in workload manifests

## Result
The platform now has an HA secret-management layer based on Kubernetes identity and short-lived Vault credentials.
