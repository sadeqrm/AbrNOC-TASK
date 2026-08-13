# AbrNOC-TASK

## Phase 1: Repository & Ansible Foundation

The purpose of this phase is to successfully run:

```bash
ansible all -m ping
```

Then:

```bash
ansible-playbook playbooks/prepare.yml
```

After completing this phase, all **5 nodes** will be ready for the next phase.

---

# Initial Repository Structure

Initial structure:

```text
AbrNOC-TASK/
├── README.md
├── .gitignore
│
├── docs/
│   └── architecture.md
│
└── ansible/
    ├── ansible.cfg
    │
    ├── inventories/
    │   └── lab/
    │       ├── hosts.yml
    │       │
    │       └── group_vars/
    │           ├── all.yml
    │           ├── control_plane.yml
    │           └── workers.yml
    │
    ├── playbooks/
    │   ├── prepare.yml
    │   └── site.yml
    │
    └── roles/
        └── common/
            ├── defaults/
            │   └── main.yml
            ├── handlers/
            │   └── main.yml
            └── tasks/
                └── main.yml
```

Then we need to create the directories below:

```bash
mkdir -p ansible/inventories/lab/group_vars
mkdir -p ansible/playbooks
mkdir -p ansible/roles/common/{tasks,handlers,defaults}
mkdir -p docs
```

I did it with **MobaXterm** using its Cygwin capability on Windows.

---

# `.gitignore`

Create the `.gitignore` file with the following contents:

```gitignore
# Ansible
*.retry

# SSH Keys
*.pem
*.key
id_rsa
id_ed25519

# Secrets
*.secret
*.vault
.env
.env.*

# Ansible Vault temporary files
.vault_pass
vault-password*

# Kubernetes
*.kubeconfig
kubeconfig
admin.conf

# IDE
.idea/
.vscode/

# OS
.DS_Store
Thumbs.db

# Logs
*.log

# Temporary files
*.tmp
*.swp
*~

# Python
__pycache__/
*.pyc
.venv/
venv/
```

> **No PRIVATE KEY, KUBECONFIG, or PASSWORD should be committed to GitHub.**

---

# Ansible Configuration

With the following `ansible.cfg` file, there is no need to write:

```bash
-i inventories/lab/hosts.yml
```

every time.

Create:

```bash
nano ansible/ansible.cfg
```

Contents:

```ini
[defaults]
inventory = inventories/lab/hosts.yml
roles_path = roles

host_key_checking = False
retry_files_enabled = False

interpreter_python = auto_silent

stdout_callback = default

forks = 10
timeout = 30

[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = False

[ssh_connection]
pipelining = True
```

---

# Inventory Setup

Create the inventory file:

```bash
nano ansible/inventories/lab/hosts.yml
```

```yaml
---
all:
  children:

    control_plane:
      hosts:

        k8s-cp-01:
          ansible_host: 10.10.10.11

        k8s-cp-02:
          ansible_host: 10.10.10.12

        k8s-cp-03:
          ansible_host: 10.10.10.13

    workers:
      hosts:

        k8s-worker-01:
          ansible_host: 10.10.10.21

        k8s-worker-02:
          ansible_host: 10.10.10.22
```

Our logical inventory structure will be:

```text
all
├── control_plane
│   ├── k8s-cp-01
│   ├── k8s-cp-02
│   └── k8s-cp-03
│
└── workers
    ├── k8s-worker-01
    └── k8s-worker-02
```

Then we can easily call:

```yaml
hosts: control_plane
```

or:

```yaml
hosts: workers
```

---

# Global Variables

Global variables will be configured in:

```text
ansible/inventories/lab/group_vars/all.yml
```

Create:

```bash
nano ansible/inventories/lab/group_vars/all.yml
```

```yaml
---
# --------------------------------------------------
# General
# --------------------------------------------------

cluster_name: abrnoc-k8s

timezone: Asia/Tehran

ansible_user: ubuntu


# --------------------------------------------------
# Kubernetes API
# --------------------------------------------------

kubernetes_api_domain: api.k8s.lab
kubernetes_api_vip: 10.10.10.100
kubernetes_api_port: 6443


# --------------------------------------------------
# Kubernetes Networking
# --------------------------------------------------

node_network_cidr: 10.10.10.0/24

pod_network_cidr: 10.244.0.0/16

service_network_cidr: 10.96.0.0/12


# --------------------------------------------------
# Container Runtime
# --------------------------------------------------

container_runtime: containerd


# --------------------------------------------------
# CNI
# --------------------------------------------------

kubernetes_cni: calico
```

---

# Control Plane Variables

Create:

```bash
nano ansible/inventories/lab/group_vars/control_plane.yml
```

```yaml
---
control_plane: true

etcd_member: true

kubernetes_control_plane_endpoint: >-
  {{ kubernetes_api_domain }}:{{ kubernetes_api_port }}
```

---

# Worker Variables

Create:

```text
ansible/inventories/lab/group_vars/workers.yml
```

```yaml
---
control_plane: false

etcd_member: false
```

---

# First Role: Common OS Preparation

The first role is responsible for:

```text
Basic OS preparation
├── hostname
├── packages
├── swap
├── kernel modules
├── sysctl
└── hosts resolution
```

Create:

```bash
nano ansible/roles/common/defaults/main.yml
```

```yaml
---
common_packages:
  - curl
  - wget
  - vim
  - git
  - jq
  - unzip
  - tar
  - ca-certificates
  - gnupg
  - lsb-release
  - apt-transport-https
  - software-properties-common
  - chrony
```

---

# Common Tasks

Create:

```bash
nano ansible/roles/common/tasks/main.yml
```

```yaml
---
- name: Set system hostname
  ansible.builtin.hostname:
    name: "{{ inventory_hostname }}"


- name: Update apt cache
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600


- name: Install common packages
  ansible.builtin.apt:
    name: "{{ common_packages }}"
    state: present


- name: Ensure chrony is enabled and running
  ansible.builtin.service:
    name: chrony
    state: started
    enabled: true


# --------------------------------------------------
# Swap
# Kubernetes nodes must run without swap for the
# configuration we will use.
# --------------------------------------------------

- name: Disable swap immediately
  ansible.builtin.command:
    cmd: swapoff -a
  when: ansible_swaptotal_mb > 0
  changed_when: true


- name: Disable swap permanently in fstab
  ansible.builtin.replace:
    path: /etc/fstab
    regexp: '^([^#].*\s+swap\s+.*)$'
    replace: '# \1'


# --------------------------------------------------
# Kernel modules required by Kubernetes networking
# --------------------------------------------------

- name: Configure required kernel modules
  ansible.builtin.copy:
    dest: /etc/modules-load.d/k8s.conf
    owner: root
    group: root
    mode: "0644"
    content: |
      overlay
      br_netfilter


- name: Load overlay kernel module
  community.general.modprobe:
    name: overlay
    state: present


- name: Load br_netfilter kernel module
  community.general.modprobe:
    name: br_netfilter
    state: present


# --------------------------------------------------
# Kernel networking parameters
# --------------------------------------------------

- name: Configure Kubernetes sysctl parameters
  ansible.builtin.copy:
    dest: /etc/sysctl.d/99-kubernetes-cri.conf
    owner: root
    group: root
    mode: "0644"
    content: |
      net.bridge.bridge-nf-call-iptables  = 1
      net.bridge.bridge-nf-call-ip6tables = 1
      net.ipv4.ip_forward                 = 1
  notify:
    - Reload sysctl


# --------------------------------------------------
# Local name resolution
# --------------------------------------------------

- name: Add Kubernetes nodes to /etc/hosts
  ansible.builtin.blockinfile:
    path: /etc/hosts
    marker: "# {mark} ANSIBLE MANAGED KUBERNETES HOSTS"
    block: |
      10.10.10.100 api.k8s.lab
      10.10.10.11  k8s-cp-01
      10.10.10.12  k8s-cp-02
      10.10.10.13  k8s-cp-03
      10.10.10.21  k8s-worker-01
      10.10.10.22  k8s-worker-02
```

---

# Handler

Create:

```bash
nano ansible/roles/common/handlers/main.yml
```

```yaml
---
- name: Reload sysctl
  ansible.builtin.command:
    cmd: sysctl --system
  changed_when: true
```

---

# Ansible Collection Requirements

We are using the following Ansible collection:

```text
community.general
```

Therefore, we need a `requirements.yml` file.

Create:

```bash
nano ansible/requirements.yml
```

```yaml
---
collections:
  - name: community.general
```

Then install the required collection:

```bash
ansible-galaxy collection install -r requirements.yml
```

---

# Prepare Playbook

Create:

```bash
nano ansible/playbooks/prepare.yml
```

```yaml
---
- name: Prepare all Kubernetes nodes
  hosts: all

  become: true

  roles:
    - common
```

The playbook should remain as simple as possible.

The implementation logic should stay inside the role files.

---

# Master Playbook

Create the master playbook:

```bash
nano ansible/playbooks/site.yml
```

Current content:

```yaml
---
- import_playbook: prepare.yml
```

In the next phases, it will eventually look similar to:

```yaml
---
- import_playbook: prepare.yml
- import_playbook: containerd.yml
- import_playbook: kubernetes.yml
- import_playbook: ha.yml
- import_playbook: bootstrap.yml
```

It means that all infrastructure components will eventually be deployed using:

```bash
ansible-playbook playbooks/site.yml
```

---

# Ansible Connectivity Test

Now all nodes are reachable.

My favorite output is:

```console
(ansible-venv) root@Utravs-A215:~/projects/ansible# ansible all -m ping

k8s-cp-03 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.14"
    },
    "changed": false,
    "ping": "pong"
}

k8s-worker-01 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.14"
    },
    "changed": false,
    "ping": "pong"
}

k8s-worker-02 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.14"
    },
    "changed": false,
    "ping": "pong"
}

k8s-cp-02 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.14"
    },
    "changed": false,
    "ping": "pong"
}

k8s-cp-01 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.14"
    },
    "changed": false,
    "ping": "pong"
}
```

All five hosts successfully returned `pong`.

---

# First Preparation Execution

The first preparation playbook was applied using:

```bash
ansible-playbook playbooks/prepare.yml
```

Result:

```console
PLAY [Prepare all Kubernetes nodes]

TASK [Gathering Facts]
ok: [k8s-cp-01]
ok: [k8s-worker-02]
ok: [k8s-worker-01]
ok: [k8s-cp-02]
ok: [k8s-cp-03]

TASK [common : Set system hostname]
ok: [k8s-cp-01]
ok: [k8s-cp-03]
ok: [k8s-worker-01]
ok: [k8s-worker-02]
ok: [k8s-cp-02]

TASK [common : Update apt cache]
ok: [k8s-cp-03]
changed: [k8s-cp-01]
changed: [k8s-cp-02]
changed: [k8s-worker-01]
changed: [k8s-worker-02]

TASK [common : Install common packages]
changed: [k8s-cp-02]
changed: [k8s-worker-01]
changed: [k8s-worker-02]
changed: [k8s-cp-03]
changed: [k8s-cp-01]

TASK [common : Ensure chrony is enabled and running]
ok: [k8s-cp-02]
ok: [k8s-worker-02]
ok: [k8s-cp-03]
ok: [k8s-cp-01]
ok: [k8s-worker-01]

TASK [common : Disable swap immediately]
skipping: [k8s-cp-01]
skipping: [k8s-cp-02]
skipping: [k8s-cp-03]
skipping: [k8s-worker-01]
skipping: [k8s-worker-02]

TASK [common : Disable swap permanently in fstab]
ok: [k8s-cp-01]
ok: [k8s-cp-03]
ok: [k8s-cp-02]
ok: [k8s-worker-02]
ok: [k8s-worker-01]

TASK [common : Configure required kernel modules]
changed: [k8s-worker-02]
changed: [k8s-cp-02]
changed: [k8s-cp-01]
changed: [k8s-cp-03]
changed: [k8s-worker-01]

TASK [common : Load overlay kernel module]
changed: [k8s-worker-02]
changed: [k8s-worker-01]
changed: [k8s-cp-03]
changed: [k8s-cp-01]
changed: [k8s-cp-02]

TASK [common : Load br_netfilter kernel module]
changed: [k8s-cp-02]
changed: [k8s-worker-01]
changed: [k8s-cp-03]
changed: [k8s-worker-02]
changed: [k8s-cp-01]

TASK [common : Configure Kubernetes sysctl parameters]
changed: [k8s-cp-03]
changed: [k8s-cp-01]
changed: [k8s-cp-02]
changed: [k8s-worker-01]
changed: [k8s-worker-02]

TASK [common : Add Kubernetes nodes to /etc/hosts]
changed: [k8s-cp-01]
changed: [k8s-worker-02]
changed: [k8s-cp-03]
changed: [k8s-worker-01]
changed: [k8s-cp-02]

RUNNING HANDLER [common : Reload sysctl]
changed: [k8s-cp-01]
changed: [k8s-cp-03]
changed: [k8s-worker-02]
changed: [k8s-worker-01]
changed: [k8s-cp-02]
```

The final play recap was:

```console
PLAY RECAP

k8s-cp-01     : ok=12 changed=8 unreachable=0 failed=0 skipped=1 rescued=0 ignored=0
k8s-cp-02     : ok=12 changed=8 unreachable=0 failed=0 skipped=1 rescued=0 ignored=0
k8s-cp-03     : ok=12 changed=7 unreachable=0 failed=0 skipped=1 rescued=0 ignored=0
k8s-worker-01 : ok=12 changed=8 unreachable=0 failed=0 skipped=1 rescued=0 ignored=0
k8s-worker-02 : ok=12 changed=8 unreachable=0 failed=0 skipped=1 rescued=0 ignored=0
```

The preparation completed without unreachable or failed nodes.

---

# Validation

After completing the preparation playbook, the nodes were validated.

## Hostname Validation

```bash
ansible all -a "hostname"
```

Output:

```console
k8s-cp-03 | CHANGED | rc=0 >>
k8s-cp-03

k8s-worker-01 | CHANGED | rc=0 >>
k8s-worker-01

k8s-cp-01 | CHANGED | rc=0 >>
k8s-cp-01

k8s-worker-02 | CHANGED | rc=0 >>
k8s-worker-02

k8s-cp-02 | CHANGED | rc=0 >>
k8s-cp-02
```

---

## Swap Validation

```bash
ansible all -a "swapon --show"
```

Output:

```console
k8s-cp-03 | CHANGED | rc=0 >>

k8s-worker-01 | CHANGED | rc=0 >>

k8s-cp-01 | CHANGED | rc=0 >>

k8s-cp-02 | CHANGED | rc=0 >>

k8s-worker-02 | CHANGED | rc=0 >>
```

No swap devices are active.

The memory output also confirms zero swap:

```console
Swap: 0B 0B 0B
```

on all five nodes.

---

## `br_netfilter` Validation

```bash
ansible all -m shell -a "lsmod | grep br_netfilter"
```

Output:

```console
k8s-cp-01 | CHANGED | rc=0 >>
br_netfilter           32768  0
bridge                425984  1 br_netfilter

k8s-worker-01 | CHANGED | rc=0 >>
br_netfilter           32768  0
bridge                425984  1 br_netfilter

k8s-cp-02 | CHANGED | rc=0 >>
br_netfilter           32768  0
bridge                425984  1 br_netfilter

k8s-cp-03 | CHANGED | rc=0 >>
br_netfilter           32768  0
bridge                425984  1 br_netfilter

k8s-worker-02 | CHANGED | rc=0 >>
br_netfilter           32768  0
bridge                425984  1 br_netfilter
```

---

## `overlay` Validation

```bash
ansible all -m shell -a "lsmod | grep overlay"
```

Output:

```console
k8s-cp-03 | CHANGED | rc=0 >>
overlay                233472  0

k8s-cp-02 | CHANGED | rc=0 >>
overlay                233472  0

k8s-cp-01 | CHANGED | rc=0 >>
overlay                233472  0

k8s-worker-01 | CHANGED | rc=0 >>
overlay                233472  0

k8s-worker-02 | CHANGED | rc=0 >>
overlay                233472  0
```

Both required kernel modules are loaded on all nodes.

---

## Memory and Swap Check

```bash
ansible all -a "free -h"
```

Example output:

```console
              total        used        free      shared  buff/cache   available
Mem:           5.2Gi       628Mi       4.2Gi       1.7Mi       681Mi       4.6Gi
Swap:             0B          0B          0B
```

All nodes report:

```text
Swap: 0B
```

---

## IP Forwarding Validation

```bash
ansible all -a "sysctl net.ipv4.ip_forward"
```

Output:

```console
k8s-worker-01 | CHANGED | rc=0 >>
net.ipv4.ip_forward = 1

k8s-cp-01 | CHANGED | rc=0 >>
net.ipv4.ip_forward = 1

k8s-cp-03 | CHANGED | rc=0 >>
net.ipv4.ip_forward = 1

k8s-cp-02 | CHANGED | rc=0 >>
net.ipv4.ip_forward = 1

k8s-worker-02 | CHANGED | rc=0 >>
net.ipv4.ip_forward = 1
```

IP forwarding is enabled on every Kubernetes node.

---

# Phase 1 Result

At the end of Phase 1:

* Ansible connectivity is working on all five nodes.
* Inventory groups are configured.
* Global, Control Plane, and Worker variables are configured.
* Base OS packages are installed.
* Hostnames are configured.
* Chrony is enabled.
* Swap is disabled.
* `overlay` is loaded.
* `br_netfilter` is loaded.
* Kubernetes-related sysctl parameters are configured.
* Local Kubernetes hostname resolution is configured.
* The base preparation playbook completes successfully.

```text
Phase 1
Repository & Ansible Foundation
        │
        ├── Repository Structure       ✓
        ├── Ansible Configuration      ✓
        ├── Inventory                  ✓
        ├── Group Variables            ✓
        ├── Common Role                ✓
        ├── Node Connectivity          ✓
        ├── OS Preparation             ✓
        └── Validation                 ✓
```

## Phase 1 Status

**COMPLETED**

Next:

```text
Phase 2
Container Runtime
+
Kubernetes Packages
+
Cluster Bootstrap Preparation
```
