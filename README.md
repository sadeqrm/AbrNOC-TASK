# AbrNOC-TASK

Senior DevOps Engineer Technical Task — **Phase 0: Architecture Design**

---

## Phase 0 — Basic Architecture

The initial Kubernetes architecture consists of:

* **3 Control Plane nodes**
* **2 Worker nodes**
* **1 Highly Available Kubernetes API VIP**
* **3-member stacked etcd cluster**
* HA API access using **Pacemaker, Corosync, Keepalived, and HAProxy**

```text
                         ┌─────────────────────────┐
                         │      Admin / Laptop     │
                         │ kubectl / ansible / git │
                         └────────────┬────────────┘
                                      │
                                      │
                           api.k8s.lab:6443
                                      │
                         ┌────────────▼────────────┐
                         │          VIP            │
                         │     10.10.10.100        │
                         │ Pacemaker + Keepalived  │
                         └────────────┬────────────┘
                                      │
                 ┌────────────────────┼────────────────────┐
                 │                    │                    │
        ┌────────▼───────┐   ┌────────▼───────┐   ┌────────▼───────┐
        │ k8s-cp-01      │   │ k8s-cp-02      │   │ k8s-cp-03      │
        │ 10.10.10.11    │   │ 10.10.10.12    │   │ 10.10.10.13    │
        │ kube-apiserver │   │ kube-apiserver │   │ kube-apiserver │
        │ scheduler      │   │ scheduler      │   │ scheduler      │
        │ controller     │   │ controller     │   │ controller     │
        │ etcd-1         │   │ etcd-2         │   │ etcd-3         │
        └────────┬───────┘   └────────┬───────┘   └────────┬───────┘
                 │                    │                    │
                 └────────────────────┼────────────────────┘
                                      │
                          Kubernetes Cluster Network
                                      │
                         ┌────────────┴────────────┐
                         │                         │
                ┌────────▼────────┐       ┌────────▼────────┐
                │ k8s-worker-01   │       │ k8s-worker-02   │
                │ 10.10.10.21     │       │ 10.10.10.22     │
                │ Kafka           │       │ Kafka           │
                │ Applications    │       │ Applications    │
                │ Istio           │       │ Istio           │
                │ Monitoring      │       │ Monitoring      │
                └─────────────────┘       └─────────────────┘
```

---

## Stacked etcd Topology

This architecture uses a **stacked etcd topology**, meaning that every Kubernetes Control Plane node also hosts an etcd member.

```text
k8s-cp-01 → etcd-1
k8s-cp-02 → etcd-2
k8s-cp-03 → etcd-3
```

A three-member etcd cluster provides quorum and allows the cluster to tolerate the failure of one etcd member.

This topology is suitable for the required highly available Kubernetes cluster and is supported by `kubeadm`.

---

## Server Layout

| Hostname        |     IP Address | Role                    |    CPU |   RAM |   Disk |
| --------------- | -------------: | ----------------------- | -----: | ----: | -----: |
| `k8s-cp-01`     |  `10.10.10.11` | Control Plane + etcd    | 4 vCPU |  8 GB |  60 GB |
| `k8s-cp-02`     |  `10.10.10.12` | Control Plane + etcd    | 4 vCPU |  8 GB |  60 GB |
| `k8s-cp-03`     |  `10.10.10.13` | Control Plane + etcd    | 4 vCPU |  8 GB |  60 GB |
| `k8s-worker-01` |  `10.10.10.21` | Worker                  | 8 vCPU | 16 GB | 100 GB |
| `k8s-worker-02` |  `10.10.10.22` | Worker                  | 8 vCPU | 16 GB | 100 GB |
| `VIP`           | `10.10.10.100` | Kubernetes API Endpoint |      — |     — |      — |

---

## Base Components

All Kubernetes nodes will have the following components installed:

```text
containerd
kubelet
kubeadm
```

The Control Plane nodes will also have:

```text
kubectl
```

---

## Container Runtime

I selected **containerd** as the Kubernetes container runtime.

Kubernetes requires a CRI-compatible container runtime on every node, and containerd will be used for this environment.

```text
Runtime: containerd
```

---

## Kubernetes API Endpoint

One of the most important architecture decisions is to use a **shared HA endpoint** for the Kubernetes API instead of exposing a specific Control Plane node directly.

The endpoint will be:

```text
api.k8s.lab:6443
        │
        ▼
10.10.10.100
```

The kubeadm configuration will therefore use:

```yaml
controlPlaneEndpoint: "api.k8s.lab:6443"
```

This endpoint will remain stable regardless of which Control Plane node is currently available.

> The `controlPlaneEndpoint` represents the shared Kubernetes API endpoint. Individual Control Plane nodes will still use their own node addresses internally when kubeadm initializes and joins them.

---

# Task 1 — Kubernetes Control Plane HA

The required HA stack contains:

```text
Pacemaker
Corosync
Keepalived
HAProxy
```

The proposed architecture is:

```text
Pacemaker / Corosync
        │
        │ manages HA resource state
        ▼
    Keepalived
        │
        │ VRRP / VIP ownership
        ▼
  10.10.10.100
        │
        ▼
      HAProxy
        │
        │ TCP :6443
        ▼
┌──────────────┬──────────────┬──────────────┐
│              │              │              │
▼              ▼              ▼
CP1            CP2            CP3
:6443          :6443          :6443
```

---

## Component Responsibilities

### Keepalived

Keepalived is responsible for:

* VIP ownership
* VRRP election
* VIP failover
* Node/service health checks used for VIP decisions

### Pacemaker / Corosync

Pacemaker and Corosync are responsible for:

* Cluster membership
* Resource state management
* Failure detection
* HA resource orchestration

### HAProxy

HAProxy acts as the TCP load balancer for the Kubernetes API servers.

```text
10.10.10.100:6443
        │
        ▼
     HAProxy
        │
   ┌────┼────┐
   ▼    ▼    ▼
 CP1   CP2   CP3
 6443  6443  6443
```

The VIP itself only provides a stable entry point. HAProxy is responsible for forwarding traffic to healthy `kube-apiserver` instances.

---

## Final HA Architecture

```text
Pacemaker / Corosync
        │
        │ manages HA resources
        │
        ├── Keepalived → VIP ownership / failover
        │
        └── HAProxy → Kubernetes API TCP load balancing
                           │
                   ┌───────┼───────┐
                   ▼       ▼       ▼
                  CP1     CP2     CP3
                 :6443   :6443   :6443
```

---

# Kubernetes Networking

The network ranges are defined as follows.

## Node Network

```text
10.10.10.0/24
```

## Kubernetes API VIP

```text
10.10.10.100
```

## Pod CIDR

```text
10.244.0.0/16
```

## Service CIDR

```text
10.96.0.0/12
```

> **Important:** The Node, Pod, and Service networks must not overlap.

---

# CNI

For this task, I selected:

```text
Calico
```

Calico keeps the initial Kubernetes networking architecture straightforward while still providing the networking capabilities required by the cluster.

For more advanced networking, security, and observability requirements, **Cilium** could also be considered.

The CNI must be selected before completing the Kubernetes bootstrap process because after the Control Plane is initialized, a network plugin must be installed before the cluster becomes fully operational.

---

# Workload Placement

Application workloads will **not** run on the Kubernetes Control Plane nodes.

```text
cp01 ─┐
cp02 ─┼─ Kubernetes Control Plane
cp03 ─┘
```

Application workloads will run on the Worker nodes:

```text
worker01 ─┐
          ├─ Application Workloads
worker02 ─┘
```

---

## Planned Application Workloads

The Worker nodes will eventually host:

```text
ArgoCD

Kafka Broker 1
Kafka Broker 2
Kafka Broker 3

Producer
Consumer

Istio

Vault

Prometheus
Alertmanager

Jaeger
```

---

# Kafka Failure Domain Consideration

There is an architectural limitation in the requested topology.

The task requires:

```text
3 Kafka Brokers
```

However, the Kubernetes cluster contains only:

```text
2 Worker Nodes
```

Therefore, one Worker node will inevitably host more than one Kafka broker.

This means that while we can run three Kafka broker Pods, we do not have three independent host-level failure domains.

```text
3 Kafka Pods != 3 Independent Host Failure Domains
```

Kubernetes scheduling rules such as:

```text
podAntiAffinity
```

will be used to distribute Kafka brokers across Worker nodes as much as possible.

This limitation will also be documented and considered during the failure-testing phase.

---

# Namespace Design

The planned Kubernetes namespaces are:

```text
argocd
istio-system
vault
kafka
apps
monitoring
tracing
```

The logical workload layout will look similar to:

```text
kafka/
├── kafka-0
├── kafka-1
└── kafka-2

apps/
├── producer
└── consumer

monitoring/
├── prometheus
└── alertmanager

tracing/
└── jaeger
```

---

# Project Workflow

The implementation workflow will follow this sequence:

```text
GitHub Repository
        │
        ▼
      Ansible
        │
        ├── OS preparation
        ├── containerd
        ├── Kubernetes packages
        ├── Pacemaker
        ├── Corosync
        ├── Keepalived
        ├── HAProxy
        └── Kubernetes cluster bootstrap
        │
        ▼
HA Kubernetes Cluster
        │
        ▼
ArgoCD Bootstrap
        │
        ▼
GitOps Repository
        │
        ├── Kafka
        ├── Producer
        ├── Consumer
        ├── Istio
        ├── Vault
        ├── Prometheus
        └── Jaeger
```

After deployment, failure tests will be performed:

```text
Failure Tests
      │
      ├── Kill Kafka broker leader
      ├── Kill active Vault node
      └── Kill active VIP holder
      │
      ▼
Metrics + Logs + Traces
      │
      ▼
Recovery Time Measurements
      │
      ▼
Final Reliability Report
```

---

# Phase 0 — Final Architecture Decisions

The following architecture decisions are now locked for Phase 0:

```text
Operating System
Ubuntu Server 24.04 LTS


Control Plane
k8s-cp-01       10.10.10.11
k8s-cp-02       10.10.10.12
k8s-cp-03       10.10.10.13


Workers
k8s-worker-01   10.10.10.21
k8s-worker-02   10.10.10.22


Kubernetes API VIP
api.k8s.lab     10.10.10.100:6443


etcd
3-member stacked etcd


Container Runtime
containerd


Kubernetes Bootstrap
kubeadm


High Availability
Pacemaker
Corosync
Keepalived


API Load Balancing
HAProxy


CNI
Calico


GitOps
ArgoCD
```

---

## Phase 0 Status

* [x] Kubernetes topology defined
* [x] Control Plane nodes defined
* [x] Worker nodes defined
* [x] IP addressing defined
* [x] Kubernetes API VIP defined
* [x] etcd topology defined
* [x] Container runtime selected
* [x] Kubernetes bootstrap method selected
* [x] Control Plane HA architecture defined
* [x] API load balancing architecture defined
* [x] Kubernetes network ranges defined
* [x] CNI selected
* [x] Workload placement strategy defined
* [x] Namespace strategy defined
* [x] Kafka failure-domain limitation identified

---

# Next Step

The next phase will focus on:

```text
Phase 1
Repository Design
+
Ansible Foundation
```

This includes defining the repository structure, Ansible inventory, variables, roles, and the first infrastructure preparation playbooks.
