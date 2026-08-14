# AbrNOC-TASK

## Phase 3: Highly Available Kubernetes API Endpoint

This phase is one of the most important parts of the task.

Before running `kubeadm init`, I need a highly available endpoint that will later be used by the Kubernetes API Server.

The target endpoint is:

```text
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
```

I intentionally do not use port `6443` for the HAProxy frontend.

The design is:

```text
Frontend = 8443
Backend  = 6443
```

Later, Kubernetes will use:

```yaml
controlPlaneEndpoint: "api.k8s.lab:8443"
```

This avoids a port conflict between HAProxy and the local `kube-apiserver`, because each Kubernetes API Server will listen on port `6443` on its own control-plane node.

---

### Phase 3 Result

After completing this phase, the HA layer contains:

```text
Pacemaker        ✓
Corosync         ✓
Keepalived       ✓
HAProxy          ✓

3-node HA cluster       ✓
Corosync quorum         ✓
VIP ownership           ✓
VIP failover            ✓
VIP failback            ✓
HAProxy TCP frontend    ✓
HAProxy health checks   ✓

VIP:
10.10.10.100            ✓

API Endpoint:
api.k8s.lab:8443        ✓
```

> **Note:** At this stage the HA TCP endpoint is ready, but Kubernetes has not been bootstrapped yet.  
> Real Kubernetes API Server continuity through this endpoint will be validated after `kubeadm init` and the remaining control-plane nodes join the cluster.

---

## HA Architecture

The HA design for the Kubernetes API endpoint is:

```text
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
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
         k8s-cp-01      k8s-cp-02      k8s-cp-03
        10.10.10.11    10.10.10.12    10.10.10.13
            :6443          :6443          :6443
```

The three control-plane nodes are:

| Node | IP Address | Keepalived Priority |
|---|---:|---:|
| `k8s-cp-01` | `10.10.10.11` | `150` |
| `k8s-cp-02` | `10.10.10.12` | `140` |
| `k8s-cp-03` | `10.10.10.13` | `130` |

Virtual IP:

```text
10.10.10.100
```

Kubernetes API endpoint:

```text
api.k8s.lab:8443
```

---

# HA Components and Responsibilities

## 1. Corosync

Corosync is responsible for cluster communication and membership between the control-plane nodes.

The cluster consists of three nodes:

```text
Nodes    = 3
Majority = 2
```

Therefore, the cluster can maintain quorum while one node is unavailable.

The final cluster name is:

```text
abrnoc-ha
```

---

## 2. Pacemaker

Pacemaker provides the cluster state and resource-management foundation.

Its responsibilities include:

- Cluster state management
- Resource monitoring
- Service recovery logic
- Failure handling
- Integration with Corosync membership and quorum

For this lab, STONITH is disabled because there is no IPMI, Redfish, hypervisor fencing, or other fencing device available.

```bash
pcs property set stonith-enabled=false
```

> **Important:** Disabling STONITH is acceptable only for this lab.  
> A production HA cluster should use a proper fencing mechanism.

---

## 3. Keepalived

Keepalived is responsible for:

- VRRP
- VIP ownership
- VIP election
- VIP failover
- VIP failback
- Health-based priority adjustment

The VIP is:

```text
10.10.10.100
```

All nodes are configured as `BACKUP`.

The node priority decides which server becomes MASTER:

```text
k8s-cp-01 → 150
k8s-cp-02 → 140
k8s-cp-03 → 130
```

Normally:

```text
10.10.10.100
      │
      ▼
 k8s-cp-01
```

If Keepalived fails on `k8s-cp-01`:

```text
10.10.10.100
      │
      ▼
 k8s-cp-02
```

When `k8s-cp-01` becomes healthy again, its higher priority allows the VIP to fail back to it.

---

## 4. HAProxy

HAProxy is responsible for:

- TCP load balancing
- Backend health checking
- Kubernetes API Server distribution
- Providing the frontend endpoint on port `8443`

Frontend:

```text
*:8443
```

Backends:

```text
10.10.10.11:6443
10.10.10.12:6443
10.10.10.13:6443
```

TCP health checks are configured for the Kubernetes API Server backends.

---

# Repository Structure for Phase 3

```text
ansible/
├── playbooks/
│   ├── prepare.yml
│   ├── containerd.yml
│   ├── kubernetes.yml
│   ├── ha.yml
│   └── site.yml
│
└── roles/
    ├── common/
    ├── containerd/
    ├── kubernetes_packages/
    │
    └── ha/
        ├── defaults/
        │   └── main.yml
        │
        ├── handlers/
        │   └── main.yml
        │
        ├── tasks/
        │   ├── main.yml
        │   ├── packages.yml
        │   ├── haproxy.yml
        │   ├── keepalived.yml
        │   └── pacemaker.yml
        │
        └── templates/
            ├── haproxy.cfg.j2
            └── keepalived.conf.j2
```

Create the required directories and files:

```bash
mkdir -p ansible/roles/ha/{defaults,handlers,tasks,templates}

touch ansible/roles/ha/defaults/main.yml
touch ansible/roles/ha/handlers/main.yml

touch ansible/roles/ha/tasks/main.yml
touch ansible/roles/ha/tasks/packages.yml
touch ansible/roles/ha/tasks/haproxy.yml
touch ansible/roles/ha/tasks/keepalived.yml
touch ansible/roles/ha/tasks/pacemaker.yml

touch ansible/roles/ha/templates/haproxy.cfg.j2
touch ansible/roles/ha/templates/keepalived.conf.j2

touch ansible/playbooks/ha.yml
```

---

# HA Variables

The global HA variables are defined in the inventory variables.

For the current inventory layout:

```text
ansible/inventories/lab/group_vars/all/
├── main.yml
└── vault.yml
```

Example HA configuration:

```yaml
kubernetes_api_domain: api.k8s.lab
kubernetes_api_vip: 10.10.10.100

ha_packages:
  - haproxy
  - keepalived
  - pacemaker
  - corosync
  - pcs
  - resource-agents-base

ha_cluster_name: abrnoc-ha

haproxy_frontend_port: 8443
haproxy_backend_port: 6443

keepalived_virtual_router_id: 51
```

Sensitive values such as the `hacluster` password are stored using **Ansible Vault** instead of plain text.

Example:

```yaml
hacluster_user: hacluster
hacluster_password: "{{ vault_hacluster_password }}"
```

---

# Install HA Packages

```yaml
- name: Install HA packages
  ansible.builtin.apt:
    name: "{{ ha_packages }}"
    state: present
    update_cache: true

- name: Enable pcsd
  ansible.builtin.service:
    name: pcsd
    state: started
    enabled: true
```

---

# HAProxy Configuration

Template:

```text
ansible/roles/ha/templates/haproxy.cfg.j2
```

```haproxy
global
    log /dev/log local0
    log /dev/log local1 notice
    daemon

defaults
    log global
    mode tcp
    option tcplog

    timeout connect 5s
    timeout client  50s
    timeout server  50s

frontend kubernetes-api
    bind *:{{ haproxy_frontend_port }}
    mode tcp

    default_backend kubernetes-api-backend

backend kubernetes-api-backend
    mode tcp
    option tcp-check
    balance roundrobin

{% for host in groups['control_plane'] %}
    server {{ host }} {{ hostvars[host]['ansible_host'] }}:{{ haproxy_backend_port }} check inter 2s fall 3 rise 2
{% endfor %}
```

HAProxy tasks:

```yaml
- name: Deploy HAProxy configuration
  ansible.builtin.template:
    src: haproxy.cfg.j2
    dest: /etc/haproxy/haproxy.cfg
    owner: root
    group: root
    mode: "0644"
  notify:
    - Restart HAProxy

- name: Validate HAProxy configuration
  ansible.builtin.command:
    cmd: haproxy -c -f /etc/haproxy/haproxy.cfg
  changed_when: false

- name: Ensure HAProxy is enabled and running
  ansible.builtin.service:
    name: haproxy
    state: started
    enabled: true
```

Handlers:

```yaml
- name: Restart HAProxy
  ansible.builtin.service:
    name: haproxy
    state: restarted

- name: Restart Keepalived
  ansible.builtin.service:
    name: keepalived
    state: restarted
```

Using a handler here is important.

During testing, HAProxy had been started before the new configuration was written. The service was reported as `active`, but port `8443` was not listening.

The configuration timestamp was newer than the HAProxy service start time.

After restarting HAProxy:

```text
LISTEN 0 4096 0.0.0.0:8443 0.0.0.0:*
```

This is why changes to `haproxy.cfg` now notify the restart handler automatically.

---

# Keepalived Configuration

Each control-plane node has a different priority.

Inventory example:

```yaml
control_plane:
  hosts:

    k8s-cp-01:
      ansible_host: 10.10.10.11
      keepalived_priority: 150

    k8s-cp-02:
      ansible_host: 10.10.10.12
      keepalived_priority: 140

    k8s-cp-03:
      ansible_host: 10.10.10.13
      keepalived_priority: 130
```

Keepalived template:

```text
ansible/roles/ha/templates/keepalived.conf.j2
```

```jinja2
global_defs {
    router_id {{ inventory_hostname }}
}

vrrp_script check_haproxy {
    script "/usr/bin/pgrep haproxy"

    interval 2
    timeout 2

    fall 2
    rise 2

    weight -20
}

vrrp_instance VI_K8S {
    state BACKUP

    interface {{ ansible_facts['default_ipv4']['interface'] }}

    virtual_router_id {{ keepalived_virtual_router_id }}

    priority {{ keepalived_priority }}

    advert_int 1

    authentication {
        auth_type PASS
        auth_pass {{ keepalived_auth_pass }}
    }

    virtual_ipaddress {
        {{ kubernetes_api_vip }}/24
    }

    track_script {
        check_haproxy
    }
}
```

> **TIP:** All control-plane nodes are configured as `BACKUP`.  
> The configured priority determines which node becomes MASTER.

Keepalived tasks:

```yaml
- name: Deploy Keepalived configuration
  ansible.builtin.template:
    src: keepalived.conf.j2
    dest: /etc/keepalived/keepalived.conf
    owner: root
    group: root
    mode: "0600"
  notify:
    - Restart Keepalived

- name: Ensure Keepalived is enabled and running
  ansible.builtin.service:
    name: keepalived
    state: started
    enabled: true
```

---

# Pacemaker and Corosync Cluster

Before creating the cluster, the `hacluster` account is configured and the nodes authenticate to each other using `pcs`.

The intended cluster is:

```text
abrnoc-ha
├── k8s-cp-01 / 10.10.10.11
├── k8s-cp-02 / 10.10.10.12
└── k8s-cp-03 / 10.10.10.13
```

The cluster is created using explicit Corosync addresses:

```bash
pcs cluster setup abrnoc-ha \
  k8s-cp-01 addr=10.10.10.11 \
  k8s-cp-02 addr=10.10.10.12 \
  k8s-cp-03 addr=10.10.10.13 \
  --start \
  --enable
```

Using explicit addresses avoids depending on ambiguous hostname resolution when Corosync builds its KNET links.

---

# Issue Found: Wrong Corosync Address

During the first cluster bootstrap, Corosync selected this address for `k8s-cp-01`:

```text
127.0.1.1
```

The result was:

```text
k8s-cp-01:
nodeid 2: disconnected
nodeid 3: disconnected
```

while `k8s-cp-02` and `k8s-cp-03` could communicate with each other.

The root cause was `/etc/hosts`:

```text
127.0.1.1       k8s-cp-01
10.10.10.11     k8s-cp-01
```

`getent` returned the loopback address first.

The incorrect mapping was removed so that:

```bash
getent hosts k8s-cp-01
```

returns:

```text
10.10.10.11 k8s-cp-01
```

The cluster was then recreated using explicit addresses.

After the fix:

```text
k8s-cp-01 → 10.10.10.11
k8s-cp-02 → 10.10.10.12
k8s-cp-03 → 10.10.10.13
```

and all Corosync links became connected.

---

# Corosync Connectivity Validation

Validation command:

```bash
ansible control_plane \
  -m shell \
  -a "corosync-cfgtool -s" \
  --ask-vault-pass
```

Final result:

```text
k8s-cp-01:
  nodeid 1: localhost
  nodeid 2: connected
  nodeid 3: connected

k8s-cp-02:
  nodeid 1: connected
  nodeid 2: localhost
  nodeid 3: connected

k8s-cp-03:
  nodeid 1: connected
  nodeid 2: connected
  nodeid 3: localhost
```

---

# Cluster and Quorum Validation

Pacemaker status:

```bash
ansible k8s-cp-01 \
  -a "pcs status" \
  --ask-vault-pass
```

Final state:

```text
Cluster name: abrnoc-ha

Cluster Summary:
  Stack: corosync
  3 nodes configured
  partition with quorum

Node List:
  Online: [ k8s-cp-01 k8s-cp-02 k8s-cp-03 ]

Daemon Status:
  corosync: active/enabled
  pacemaker: active/enabled
  pcsd: active/enabled
```

Quorum validation:

```bash
ansible k8s-cp-01 \
  -a "corosync-quorumtool -s" \
  --ask-vault-pass
```

Result:

```text
Nodes:            3
Expected votes:   3
Total votes:      3
Quorum:           2
Quorate:          Yes
```

This confirms that all three control-plane nodes participate in the same Corosync cluster.

---

# STONITH Configuration for the Lab

Initially Pacemaker reported:

```text
No stonith devices and stonith-enabled is not false
Resource start-up disabled since no STONITH resources have been defined
```

Because this lab does not have a fencing device, STONITH was disabled:

```bash
pcs property set stonith-enabled=false
```

After that, the cluster became valid and remained quorate.

Again, this is a **lab-only configuration**.

---

# VIP Ownership Validation

To identify the current VIP owner:

```bash
ansible control_plane \
  -m shell \
  -a "ip -br addr | grep 10.10.10.100 || true" \
  --ask-vault-pass
```

Initial state:

```text
k8s-cp-01:
ens33 UP 10.10.10.11/24 10.10.10.100/24

k8s-cp-02:
No VIP

k8s-cp-03:
No VIP
```

Therefore:

```text
VIP Owner = k8s-cp-01
```

---

# VIP Failover Test

To simulate failure of the active Keepalived instance:

```bash
ansible k8s-cp-01 \
  -a "systemctl stop keepalived" \
  --ask-vault-pass
```

The VIP was checked again:

```bash
ansible control_plane \
  -m shell \
  -a "ip -br addr | grep 10.10.10.100 || true" \
  --ask-vault-pass
```

Result:

```text
k8s-cp-02:
ens33 UP 10.10.10.12/24 10.10.10.100/24
```

Failover result:

```text
Before:
10.10.10.100 → k8s-cp-01

After failure:
10.10.10.100 → k8s-cp-02
```

**VIP Failover: PASS ✓**

---

# VIP Failback Test

Keepalived was started again on `k8s-cp-01`:

```bash
ansible k8s-cp-01 \
  -a "systemctl start keepalived" \
  --ask-vault-pass
```

Because `k8s-cp-01` has the highest priority, the VIP returned to it:

```text
10.10.10.100 → k8s-cp-01
```

**VIP Failback: PASS ✓**

---

# HAProxy Listener Validation

The HAProxy frontend was validated on all control-plane nodes:

```bash
ansible control_plane \
  -m shell \
  -a "ss -lntH 'sport = :8443'" \
  --ask-vault-pass
```

Final result:

```text
k8s-cp-01: LISTEN 0 4096 0.0.0.0:8443
k8s-cp-02: LISTEN 0 4096 0.0.0.0:8443
k8s-cp-03: LISTEN 0 4096 0.0.0.0:8443
```

Local TCP validation also succeeded:

```text
TCP_OK
```

---

# VIP TCP Endpoint Test

From the Ansible controller:

```bash
timeout 3 bash -c '</dev/tcp/10.10.10.100/8443' \
  && echo VIP_TCP_OK \
  || echo VIP_TCP_FAILED
```

Result:

```text
VIP_TCP_OK
```

This confirms that the HAProxy frontend can be reached through the VIP.

---

# HA Endpoint Failover Test

The active VIP owner was `k8s-cp-01`.

Keepalived was stopped:

```bash
ansible k8s-cp-01 \
  -a "systemctl stop keepalived" \
  --ask-vault-pass
```

The frontend was immediately tested again through the VIP:

```bash
timeout 3 bash -c '</dev/tcp/10.10.10.100/8443' \
  && echo VIP_FAILOVER_OK \
  || echo VIP_FAILOVER_FAILED
```

Result:

```text
VIP_FAILOVER_OK
```

At the same time:

```text
10.10.10.100 → k8s-cp-02
```

Therefore, the frontend remained reachable while VIP ownership changed from `k8s-cp-01` to `k8s-cp-02`.

**HA TCP Endpoint Failover: PASS ✓**

> This test validates TCP frontend availability.  
> End-to-end Kubernetes API availability will be tested after the API Servers are running on port `6443`.

---

# Ansible Idempotency

After fixing the HA role, the playbook was executed again:

```bash
ansible-playbook playbooks/ha.yml \
  --ask-vault-pass
```

Final recap:

```text
k8s-cp-01 : changed=0 failed=0
k8s-cp-02 : changed=0 failed=0
k8s-cp-03 : changed=0 failed=0
```

This confirms that the Phase 3 automation is idempotent in the current environment.

Several improvements were made during this process:

- HAProxy configuration changes trigger a handler restart.
- Keepalived uses `ansible_facts['default_ipv4']['interface']`.
- The `hacluster` password task does not report false changes on every run.
- `pcs host auth` does not report unnecessary Ansible changes.
- Corosync configuration detection checks the expected cluster name instead of only checking whether `/etc/corosync/corosync.conf` exists.

The last point is especially important because the operating system initially contained a default single-node Corosync configuration:

```text
cluster_name: debian
ring0_addr: 127.0.0.1
```

Simply checking for the existence of `corosync.conf` would incorrectly treat that configuration as the expected HA cluster.

---

# Phase 3 Final Validation

| Test | Result |
|---|:---:|
| HAProxy installed | ✓ |
| Keepalived installed | ✓ |
| Pacemaker installed | ✓ |
| Corosync installed | ✓ |
| `pcsd` active | ✓ |
| 3-node Corosync cluster | ✓ |
| All control-plane nodes online | ✓ |
| Cluster quorum | ✓ |
| VIP `10.10.10.100` active | ✓ |
| Single VIP owner | ✓ |
| VIP failover CP1 → CP2 | ✓ |
| VIP failback CP2 → CP1 | ✓ |
| HAProxy listens on `8443` | ✓ |
| VIP TCP endpoint reachable | ✓ |
| TCP endpoint reachable after VIP failover | ✓ |
| Ansible idempotency (`changed=0`) | ✓ |

---

## Phase 3 Status

```text
Phase 3: Highly Available Kubernetes API Endpoint

Corosync     PASS ✓
Pacemaker    PASS ✓
Keepalived   PASS ✓
HAProxy      PASS ✓
Quorum       PASS ✓
VIP          PASS ✓
Failover     PASS ✓
Failback     PASS ✓
Idempotency  PASS ✓
```

The infrastructure is now ready for the next phase:

```text
Phase 4
   │
   ▼
kubeadm bootstrap
   │
   ▼
controlPlaneEndpoint:
api.k8s.lab:8443
```