# AbrNOC-TASK
Phase 0 :
Basic Architecture :
(3 Control-plane + 2 Worker-node)
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
        ┌────────▼───────┐   ┌────────▼───────┐   ┌──────▼─────────┐
        │ k8s-cp-01      │   │ k8s-cp-02      │   │ k8s-cp-03     │
        │ 10.10.10.11    │   │ 10.10.10.12    │   │ 10.10.10.13   │
        │ kube-apiserver │   │ kube-apiserver │   │ kube-apiserver│
        │ scheduler      │   │ scheduler      │   │ scheduler     │
        │ controller     │   │ controller     │   │ controller    │
        │ etcd-1         │   │ etcd-2         │   │ etcd-3        │
        └────────┬───────┘   └────────┬───────┘   └──────┬─────────┘
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
                │ Apps            │       │ Apps            │
                │ Istio           │       │ Istio           │
                │ Monitoring      │       │ Monitoring      │
                └─────────────────┘       └─────────────────┘

This topology is covering STACKED_ETCD which means each control-plane consist of an ETCD member.
Acceptable for HA which is supported by kubeadm with 3 contol-plane QUORUM.

| Hostname        |    IP پیشنهادی | Role                 |    CPU |   RAM |   Disk |
| --------------- | -------------: | -------------------- | -----: | ----: | -----: |
| `k8s-cp-01`     |  `10.10.10.11` | Control Plane + etcd | 4 vCPU |  8 GB |  60 GB |
| `k8s-cp-02`     |  `10.10.10.12` | Control Plane + etcd | 4 vCPU |  8 GB |  60 GB |
| `k8s-cp-03`     |  `10.10.10.13` | Control Plane + etcd | 4 vCPU |  8 GB |  60 GB |
| `k8s-worker-01` |  `10.10.10.21` | Worker               | 8 vCPU | 16 GB | 100 GB |
| `k8s-worker-02` |  `10.10.10.22` | Worker               | 8 vCPU | 16 GB | 100 GB |
| VIP             | `10.10.10.100` | Kubernetes API       |      — |     — |      — |

We need these resources because of tools like : KAFKA+ISTIO+JAEGER+PROMETHEUS+VAULT on worker nodes which makes more PRESSURE on workers.
Selected OS is : Ubuntu Server 24.04 LTS.
---
All Nodes has bellow :
    containerd
    kubelet
    kubeadm
    kubectl   # In all CP nodes

I select CONTAINERD as a runtime. K8s need a CRI for each node so I choose that.
---
So important to INIT our cluster on VIP as an ENDPOINT > kubeadm init --apiserver-advertise-address=10.10.10.100
api.k8s.lab
       │
       ▼
10.10.10.100

So KUBEADM will recieve this : controlPlaneEndpoint: "api.k8s.lab:6443"
This is a SHARED endpoint (LOAD-BALANCER)  for all CP nodes

For all HOSTS we need bellow :
nano /etc/hosts:
    10.10.10.100 api.k8s.lab
    10.10.10.11  k8s-cp-01
    10.10.10.12  k8s-cp-02
    10.10.10.13  k8s-cp-03
    10.10.10.21  k8s-worker-01
    10.10.10.22  k8s-worker-02
After we set-up a DNS server, All will happen through out of that(DNS sever Resolver).
---
TASK1 : 
Pacemaker+Keepalived
KEEPALIVED will support both VIP failover and LOADBALNCING mechanism but for task I asked for BOTH implementation.
So I suggest bellow architecture:
Pacemaker / Corosync
        │
        │ supervises HA resource state
        ▼
   Keepalived
        │
        │ VRRP
        ▼
10.10.10.100 VIP

It means KEEPALIVED responsible for :
    * VIP ownership
    * VRRP election
    * health checking
and PACEMAKER/COROSYNC responsible for :
    * cluster membership
    * resource state
    * failure handling

VIP is not just and IP ! Its responsible for sending request to 3 KUBE-APISERVER in correct manner. Logical view is :
                     10.10.10.100:6443
                            │
                         VIP/LB
                  ┌─────────┼─────────┐
                  ▼         ▼         ▼
              cp01:6443 cp02:6443 cp03:6443
In HA_KUBEADM documents you see the needs of LOAD_BALANCING_ENDPOINT in front of all API's with HEALTCH_CHECK on 6443 port.
For this purpose we need a REVERSE PROXY such as HA_PROXY too.(software TCP loadbalancer)
Keepalived + HAProxy + Pacemaker

Final Technical View will be :
Pacemaker/Corosync
        │
   manages HA services
        │
        ├── Keepalived → VIP
        │
        └── HAProxy → TCP :6443
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
             CP1        CP2        CP3

---
K8s Networking:
CIDRS will be fixed in bellow :
    Node Network:
    10.10.10.0/24

    Kubernetes API VIP:
    10.10.10.100

    Pod CIDR:
    10.244.0.0/16

    Service CIDR:
    10.96.0.0/12

- Important tip : No overlap between network ranges.

CNI:
For task I prefer CALICO. 
For complex purpose we use CILIUM(for better SECURITY and OBSERVABILITY).

Because of BOOTSTRAP_CLUSTER process which should be done before ANSIBLE, we need to select CNI because after INIT by CP, a network plugin shoud be selected and install for cluster to be UP healthy.

---
Architecure Points : 
No APP_WORKLOADS on CP nodes ! We do like bellow :
    cp01 ─┐
    cp02 ─┼─ Kubernetes control plane
    cp03 ─┘

    worker01 ─┐
            ├─ Application workloads
    worker02 ─┘

APP_WORKLOADS on WR nodes will be :
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

* There an issue beyond the task for KAFKA clustering, which we need 3 HOST for clustering in HOST_LEVEL FAULT DOMAIN. But one HOST will be hosting two
Of KAFKA brokers. With podAntiAffinity we tell scheduler to handle it.

3 Kafka Pods <-> 3 independent failure domains

---
NAMESPACES design :
    argocd
    istio-system
    vault
    kafka
    apps
    monitoring
    tracing
Which will be like that :
    kafka/
    kafka-0
    kafka-1
    kafka-2

    apps/
    producer
    consumer

    monitoring/
    prometheus
    alertmanager

    tracing/
    jaeger

---
Project WORKFLOW :
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
      └── Kubernetes cluster bootstrap
      │
      ▼
HA Kubernetes Cluster
      │
      ▼
ArgoCD bootstrap
      │
      ▼
GitOps repository
      │
      ├── Kafka
      ├── Producer
      ├── Consumer
      ├── Istio
      ├── Vault
      ├── Prometheus
      └── Jaeger

then :

failure tests
      │
      ├── Kafka leader killed
      ├── Vault active killed
      └── VIP holder killed
      │
      ▼
metrics + logs + traces + recovery times
      │
      ▼
final report

So we locked in our PHASE 0 like that :
    OS             Ubuntu Server 24.04 LTS

    Control Plane
    cp01            10.10.10.11
    cp02            10.10.10.12
    cp03            10.10.10.13

    Workers
    worker01        10.10.10.21
    worker02        10.10.10.22

    API VIP
    api.k8s.lab     10.10.10.100:6443

    etcd            stacked / 3 members
    Runtime         containerd
    Bootstrap       kubeadm
    HA              Pacemaker + Corosync + Keepalived
    API balancing   HAProxy
    CNI             Calico
    GitOps          ArgoCD

