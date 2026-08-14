# AbrNOC-TASK

## Phase 3: Highly Available Kubernetes API Endpoint
This phase is one of important part of the TASK I think. Because before kubeadm init we need an ENDPOINT which has been ready for API SERVER.
I am going to setup PHASE03 like this :
  api.k8s.lab:8443
          │
          ▼
  10.10.10.100:8443
          │
          ▼
      HAProxy
          │
          ├── k8s-cp-01:6443
          ├── k8s-cp-02:6443
          └── k8s-cp-03:6443
HAPROXY is better not to listen on the same PORT.
It means :
  Frontend = 8443
  Backend  = 6443
and after that :
  controlPlaneEndpoint: "api.k8s.lab:8443"

After this PHASE we have something bellow :
  Pacemaker        ✓
  Corosync         ✓
  Keepalived       ✓
  HAProxy          ✓

  3-node HA cluster       ✓
  VIP ownership           ✓
  HAProxy health checks   ✓

  VIP:
  10.10.10.100            ✓

  API Endpoint:
  api.k8s.lab:8443        ✓

  VIP Failover Test       ✓

Our HA design will be :
                      api.k8s.lab:8443
                              │
                              ▼
                    10.10.10.100:8443
                              │
                        Keepalived VIP
                              │
                              ▼
                          HAProxy
                              │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        k8s-cp-01       k8s-cp-02       k8s-cp-03
        :6443           :6443           :6443

For HA level :
                Pacemaker / Corosync
                          │
                Cluster Membership
                Resource Monitoring
                          │
            ┌────────────┴────────────┐
            │                         │
            ▼                         ▼
        Keepalived                  HAProxy
          VRRP                     TCP Proxy
            │
            ▼
      10.10.10.100 VIP
---
RESPONSIBILITIES:
1. COROSYNC
  * connection and membership between CP nodes.
I am using 3 nodes so QUORUM will be there:
  3 Nodes
  Majority = 2
2. PACEMAKER
  * responsible for :
      Resource monitoring
      Service recovery
      Cluster state
      Failure handling
3. KEEPALIVED
  * responsible for :
      VRRP
      VIP ownership
      VIP failover
      Health-based election
  VIP : 10.10.10.100
4. HAPROXY
  * responsible for :
      TCP Load Balancing
      Health Checking
      API Server Distribution
  * It's BACKENDs:
      10.10.10.11:6443
      10.10.10.12:6443
      10.10.10.13:6443
TIP :HEALTHCHEK set for TCP check on 6443.
