# AbrNOC-TASK

## Phase 2: Container Runtime + Kubernetes Packages + Bootstrap Preparation

I will set this phase based on **Kubernetes v1.36** because I do not want any inconsistency during the bootstrap process.

Our purpose in this phase is:

```text
containerd        ✓
CRI               ✓
systemd cgroup    ✓

kubelet           ✓
kubeadm           ✓
kubectl           ✓

packages held     ✓
services ready    ✓
```

But there is **no cluster initialization** in this phase:

```text
kubeadm init      ✗
kubeadm join      ✗
Calico install    ✗
```

---

# Repository Structure

The new repository structure for this phase will be:

```text
ansible/
├── playbooks/
│   ├── prepare.yml
│   ├── containerd.yml
│   ├── kubernetes.yml
│   └── site.yml
│
└── roles/
    ├── common/
    │
    ├── containerd/
    │   ├── defaults/
    │   │   └── main.yml
    │   ├── handlers/
    │   │   └── main.yml
    │   └── tasks/
    │       └── main.yml
    │
    └── kubernetes_packages/
        ├── defaults/
        │   └── main.yml
        └── tasks/
            └── main.yml
```

So I am going to create these folders and files:

```bash
mkdir -p ansible/roles/containerd/{tasks,handlers,defaults}
mkdir -p ansible/roles/kubernetes_packages/{tasks,defaults}

touch ansible/playbooks/containerd.yml
touch ansible/playbooks/kubernetes.yml

touch ansible/roles/containerd/tasks/main.yml
touch ansible/roles/containerd/defaults/main.yml
touch ansible/roles/containerd/handlers/main.yml

touch ansible/roles/kubernetes_packages/tasks/main.yml
touch ansible/roles/kubernetes_packages/defaults/main.yml
```

---

# Containerd

There are two common ways to install containerd on Ubuntu:

```text
Ubuntu package
containerd
```

or:

```text
Docker repository
containerd.io
```

I decided to use the Ubuntu package.

After installation, the default configuration will be generated and I will verify:

```text
CRI
Systemd Cgroup
```

It is advised by Kubernetes that if the system is using `systemd` and `cgroup v2`, both kubelet and the container runtime should use the `systemd` cgroup driver.

---

# Containerd Defaults

Create:

```bash
nano ansible/roles/containerd/defaults/main.yml
```

```yaml
---
containerd_package_name: containerd

containerd_config_dir: /etc/containerd

containerd_config_file: /etc/containerd/config.toml

containerd_service_name: containerd

containerd_cri_socket: unix:///run/containerd/containerd.sock
```

Default CRI socket path for containerd:

```text
/run/containerd/containerd.sock
```

---

# Containerd Role

Create:

```bash
nano ansible/roles/containerd/tasks/main.yml
```

```yaml
---
# --------------------------------------------------
# Install containerd
# --------------------------------------------------

- name: Install containerd
  ansible.builtin.apt:
    name: "{{ containerd_package_name }}"
    state: present
    update_cache: true


# --------------------------------------------------
# Configuration directory
# --------------------------------------------------

- name: Ensure containerd configuration directory exists
  ansible.builtin.file:
    path: "{{ containerd_config_dir }}"
    state: directory
    owner: root
    group: root
    mode: "0755"


# --------------------------------------------------
# Generate default configuration
# --------------------------------------------------

- name: Check whether containerd configuration exists
  ansible.builtin.stat:
    path: "{{ containerd_config_file }}"
  register: containerd_config


- name: Generate default containerd configuration
  ansible.builtin.shell:
    cmd: containerd config default > {{ containerd_config_file }}
  when: not containerd_config.stat.exists
  notify:
    - Restart containerd


# --------------------------------------------------
# Configure systemd cgroup
# --------------------------------------------------

- name: Enable SystemdCgroup for containerd
  ansible.builtin.replace:
    path: "{{ containerd_config_file }}"
    regexp: 'SystemdCgroup = false'
    replace: 'SystemdCgroup = true'
  notify:
    - Restart containerd


# --------------------------------------------------
# Enable CRI
# --------------------------------------------------

- name: Ensure CRI plugin is not disabled
  ansible.builtin.replace:
    path: "{{ containerd_config_file }}"
    regexp: 'disabled_plugins\s*=\s*\["cri"\]'
    replace: 'disabled_plugins = []'
  notify:
    - Restart containerd


# --------------------------------------------------
# Service
# --------------------------------------------------

- name: Ensure containerd is enabled and running
  ansible.builtin.service:
    name: "{{ containerd_service_name }}"
    enabled: true
    state: started
```

## Important Tip

The important part here is **CRI**.

For Kubernetes, `cri` should not be present inside:

```text
disabled_plugins
```

---

# Containerd Handler

Create:

```bash
nano ansible/roles/containerd/handlers/main.yml
```

```yaml
---
- name: Restart containerd
  ansible.builtin.service:
    name: containerd
    state: restarted
```

---

# Containerd Playbook

Create:

```bash
nano ansible/playbooks/containerd.yml
```

```yaml
---
- name: Install and configure containerd
  hosts: all

  become: true

  roles:
    - containerd
```

---

# Containerd Installation

Now install containerd:

```bash
ansible-playbook playbooks/containerd.yml
```

Output:

```console
PLAY [Install and configure containerd]

TASK [Gathering Facts]
ok: [k8s-cp-01]
ok: [k8s-cp-02]
ok: [k8s-worker-02]
ok: [k8s-worker-01]
ok: [k8s-cp-03]

TASK [containerd : Install containerd]
changed: [k8s-cp-02]
changed: [k8s-worker-02]
changed: [k8s-cp-03]
changed: [k8s-cp-01]
changed: [k8s-worker-01]

TASK [containerd : Ensure containerd configuration directory exists]
changed: [k8s-worker-01]
changed: [k8s-cp-02]
changed: [k8s-cp-01]
changed: [k8s-worker-02]
changed: [k8s-cp-03]

TASK [containerd : Check whether containerd configuration exists]
ok: [k8s-cp-02]
ok: [k8s-worker-01]
ok: [k8s-cp-01]
ok: [k8s-cp-03]
ok: [k8s-worker-02]

TASK [containerd : Generate default containerd configuration]
changed: [k8s-worker-01]
changed: [k8s-cp-01]
changed: [k8s-worker-02]
changed: [k8s-cp-02]
changed: [k8s-cp-03]

TASK [containerd : Enable SystemdCgroup for containerd]
changed: [k8s-worker-01]
changed: [k8s-cp-03]
changed: [k8s-cp-02]
changed: [k8s-cp-01]
changed: [k8s-worker-02]

TASK [containerd : Ensure CRI plugin is not disabled]
ok: [k8s-worker-01]
ok: [k8s-cp-01]
ok: [k8s-cp-02]
ok: [k8s-cp-03]
ok: [k8s-worker-02]

TASK [containerd : Ensure containerd is enabled and running]
ok: [k8s-worker-01]
ok: [k8s-cp-01]
ok: [k8s-worker-02]
ok: [k8s-cp-02]
ok: [k8s-cp-03]

RUNNING HANDLER [containerd : Restart containerd]
changed: [k8s-cp-01]
changed: [k8s-cp-02]
changed: [k8s-cp-03]
changed: [k8s-worker-01]
changed: [k8s-worker-02]
```

Play recap:

```console
k8s-cp-01     : ok=9 changed=5 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
k8s-cp-02     : ok=9 changed=5 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
k8s-cp-03     : ok=9 changed=5 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
k8s-worker-01 : ok=9 changed=5 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
k8s-worker-02 : ok=9 changed=5 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

Containerd installation completed successfully on all five nodes.

---

# Containerd Validation

## Containerd Version

```bash
ansible all -a "containerd --version"
```

Output:

```console
k8s-cp-01 | CHANGED | rc=0 >>
containerd github.com/containerd/containerd/v2 2.2.2

k8s-cp-03 | CHANGED | rc=0 >>
containerd github.com/containerd/containerd/v2 2.2.2

k8s-cp-02 | CHANGED | rc=0 >>
containerd github.com/containerd/containerd/v2 2.2.2

k8s-worker-01 | CHANGED | rc=0 >>
containerd github.com/containerd/containerd/v2 2.2.2

k8s-worker-02 | CHANGED | rc=0 >>
containerd github.com/containerd/containerd/v2 2.2.2
```

---

## Containerd Service Status

```bash
ansible all -a "systemctl is-active containerd"
```

Result on all nodes:

```text
active
```

Check if containerd is enabled:

```bash
ansible all -a "systemctl is-enabled containerd"
```

Result:

```text
enabled
```

Containerd is active and enabled on all five nodes.

---

# Systemd Cgroup Validation

Run:

```bash
ansible all -m shell -a "grep -n 'SystemdCgroup' /etc/containerd/config.toml"
```

Output:

```console
k8s-worker-01 | CHANGED | rc=0 >>
109:            SystemdCgroup = true

k8s-cp-03 | CHANGED | rc=0 >>
109:            SystemdCgroup = true

k8s-worker-02 | CHANGED | rc=0 >>
109:            SystemdCgroup = true

k8s-cp-02 | CHANGED | rc=0 >>
109:            SystemdCgroup = true

k8s-cp-01 | CHANGED | rc=0 >>
109:            SystemdCgroup = true
```

---

# CRI Validation

Run:

```bash
ansible all -m shell -a "grep disabled_plugins /etc/containerd/config.toml || true"
```

Output:

```console
k8s-cp-03 | CHANGED | rc=0 >>
disabled_plugins = []

k8s-cp-02 | CHANGED | rc=0 >>
disabled_plugins = []

k8s-worker-01 | CHANGED | rc=0 >>
disabled_plugins = []

k8s-cp-01 | CHANGED | rc=0 >>
disabled_plugins = []

k8s-worker-02 | CHANGED | rc=0 >>
disabled_plugins = []
```

> CRI is not present in `disabled_plugins`.

---

# Kubernetes Version Variable

Now let us move to Kubernetes packages.

Create:

```bash
nano ansible/roles/kubernetes_packages/defaults/main.yml
```

```yaml
---
kubernetes_minor_version: "v1.36"

kubernetes_repo_url: >-
  https://pkgs.k8s.io/core:/stable:/{{ kubernetes_minor_version }}/deb/

kubernetes_keyring_file: /etc/apt/keyrings/kubernetes-apt-keyring.gpg

kubernetes_repo_file: /etc/apt/sources.list.d/kubernetes.list

kubernetes_packages:
  - kubelet
  - kubeadm
  - kubectl
```

---

# Kubernetes Package Role

Create:

```bash
nano ansible/roles/kubernetes_packages/tasks/main.yml
```

```yaml
---
# --------------------------------------------------
# Kubernetes repository prerequisites
# --------------------------------------------------

- name: Install Kubernetes repository dependencies
  ansible.builtin.apt:
    name:
      - ca-certificates
      - curl
      - gpg
      - apt-transport-https
    state: present
    update_cache: true


# --------------------------------------------------
# APT keyrings
# --------------------------------------------------

- name: Ensure APT keyrings directory exists
  ansible.builtin.file:
    path: /etc/apt/keyrings
    state: directory
    owner: root
    group: root
    mode: "0755"


# --------------------------------------------------
# Kubernetes repository signing key
# --------------------------------------------------

- name: Download Kubernetes repository signing key
  ansible.builtin.get_url:
    url: >-
      https://pkgs.k8s.io/core:/stable:/{{ kubernetes_minor_version }}/deb/Release.key
    dest: /tmp/kubernetes-release.key
    mode: "0644"


- name: Install Kubernetes repository signing key
  ansible.builtin.shell:
    cmd: >-
      gpg --dearmor --yes
      -o {{ kubernetes_keyring_file }}
      /tmp/kubernetes-release.key
  args:
    creates: "{{ kubernetes_keyring_file }}"


# --------------------------------------------------
# Kubernetes repository
# --------------------------------------------------

- name: Configure Kubernetes APT repository
  ansible.builtin.copy:
    dest: "{{ kubernetes_repo_file }}"
    owner: root
    group: root
    mode: "0644"
    content: |
      deb [signed-by={{ kubernetes_keyring_file }}] {{ kubernetes_repo_url }} /


# --------------------------------------------------
# Update repositories
# --------------------------------------------------

- name: Update APT cache after adding Kubernetes repository
  ansible.builtin.apt:
    update_cache: true


# --------------------------------------------------
# Kubernetes packages
# --------------------------------------------------

- name: Install Kubernetes packages
  ansible.builtin.apt:
    name: "{{ kubernetes_packages }}"
    state: present


# --------------------------------------------------
# Prevent uncontrolled upgrades
# --------------------------------------------------

- name: Hold Kubernetes packages
  ansible.builtin.dpkg_selections:
    name: "{{ item }}"
    selection: hold
  loop: "{{ kubernetes_packages }}"


# --------------------------------------------------
# kubelet
# --------------------------------------------------

- name: Ensure kubelet is enabled
  ansible.builtin.service:
    name: kubelet
    enabled: true
```

---

# Kubernetes Playbook

Run:

```bash
ansible-playbook playbooks/kubernetes.yml
```

Output:

```console
PLAY [Install Kubernetes packages]

TASK [Gathering Facts]
ok: [k8s-cp-01]
ok: [k8s-cp-02]
ok: [k8s-cp-03]
ok: [k8s-worker-02]
ok: [k8s-worker-01]

TASK [kubernetes_packages : Install Kubernetes repository dependencies]
ok: [k8s-cp-03]
ok: [k8s-cp-02]
ok: [k8s-worker-02]
ok: [k8s-cp-01]
ok: [k8s-worker-01]

TASK [kubernetes_packages : Ensure APT keyrings directory exists]
ok: [k8s-cp-01]
ok: [k8s-worker-01]
ok: [k8s-cp-02]
ok: [k8s-cp-03]
ok: [k8s-worker-02]

TASK [kubernetes_packages : Download Kubernetes repository signing key]
changed: [k8s-worker-01]
changed: [k8s-cp-01]
changed: [k8s-cp-02]
changed: [k8s-worker-02]
changed: [k8s-cp-03]

TASK [kubernetes_packages : Install Kubernetes repository signing key]
changed: [k8s-cp-01]
changed: [k8s-cp-02]
changed: [k8s-worker-02]
changed: [k8s-worker-01]
changed: [k8s-cp-03]

TASK [kubernetes_packages : Configure Kubernetes APT repository]
changed: [k8s-worker-01]
changed: [k8s-cp-01]
changed: [k8s-cp-03]
changed: [k8s-worker-02]
changed: [k8s-cp-02]

TASK [kubernetes_packages : Update APT cache after adding Kubernetes repository]
changed: [k8s-worker-01]
changed: [k8s-cp-02]
changed: [k8s-cp-01]
changed: [k8s-cp-03]
changed: [k8s-worker-02]

TASK [kubernetes_packages : Install Kubernetes packages]
changed: [k8s-cp-02]
changed: [k8s-worker-02]
changed: [k8s-cp-01]
changed: [k8s-cp-03]
changed: [k8s-worker-01]

TASK [kubernetes_packages : Hold Kubernetes packages]
changed: [k8s-cp-03] => (item=kubelet)
changed: [k8s-cp-01] => (item=kubelet)
changed: [k8s-worker-01] => (item=kubelet)
changed: [k8s-worker-02] => (item=kubelet)
changed: [k8s-cp-03] => (item=kubeadm)
changed: [k8s-worker-01] => (item=kubeadm)
changed: [k8s-cp-01] => (item=kubeadm)
changed: [k8s-worker-02] => (item=kubeadm)
changed: [k8s-cp-02] => (item=kubelet)
changed: [k8s-cp-03] => (item=kubectl)
changed: [k8s-worker-01] => (item=kubectl)
changed: [k8s-cp-01] => (item=kubectl)
changed: [k8s-worker-02] => (item=kubectl)
changed: [k8s-cp-02] => (item=kubeadm)
changed: [k8s-cp-02] => (item=kubectl)

TASK [kubernetes_packages : Ensure kubelet is enabled]
ok: [k8s-worker-01]
ok: [k8s-cp-02]
ok: [k8s-cp-03]
ok: [k8s-cp-01]
ok: [k8s-worker-02]
```

Play recap:

```console
k8s-cp-01     : ok=10 changed=6 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
k8s-cp-02     : ok=10 changed=6 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
k8s-cp-03     : ok=10 changed=6 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
k8s-worker-01 : ok=10 changed=6 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
k8s-worker-02 : ok=10 changed=6 unreachable=0 failed=0 skipped=0 rescued=0 ignored=0
```

Kubernetes packages were installed successfully on all five nodes.

---

# Master `site.yml`

Update:

```bash
nano ansible/playbooks/site.yml
```

```yaml
---
- import_playbook: prepare.yml
- import_playbook: containerd.yml
- import_playbook: kubernetes.yml
```

Now from Phase 0 to Phase 2, the infrastructure preparation flow is:

```text
OS Preparation
      │
      ▼
containerd
      │
      ▼
Kubernetes Packages
```

Run everything with:

```bash
ansible-playbook playbooks/site.yml
```

---

# Kubernetes Validation

## kubeadm Version

```bash
ansible all -a "kubeadm version"
```

Installed version:

```text
v1.36.3
```

---

## kubelet Version

```bash
ansible all -a "kubelet --version"
```

Output on all nodes:

```text
Kubernetes v1.36.3
```

---

## kubectl Version

```bash
ansible all -a "kubectl version --client"
```

Output:

```text
Client Version: v1.36.3
Kustomize Version: v5.8.1
```

All Kubernetes components are installed with version `v1.36.3`.

---

# Package Hold Validation

Run:

```bash
ansible all -m shell -a "apt-mark showhold | grep -E 'kubelet|kubeadm|kubectl'"
```

Output on all nodes:

```text
kubeadm
kubectl
kubelet
```

All three Kubernetes packages are held successfully.

---

# CRI Socket Verification

Run:

```bash
ansible all -a "ls -l /run/containerd/containerd.sock"
```

Output on all nodes confirms:

```text
/run/containerd/containerd.sock
```

exists and is available.

---

# Version Check

Run:

```bash
ansible all -m shell -a "containerd --version && kubeadm version -o short && kubelet --version"
```

Output:

```console
containerd github.com/containerd/containerd/v2 2.2.2
v1.36.3
Kubernetes v1.36.3
```

The same versions are installed on all five nodes.

---

# Cgroup v2 Validation

Kubernetes suggests using cgroup v2 on newer distributions.

Run:

```bash
ansible all -m shell -a "stat -fc %T /sys/fs/cgroup/"
```

Output on all nodes:

```text
cgroup2fs
```

This confirms cgroup v2 is enabled across the environment.

---

# Phase 2 Checklist

```bash
ansible all -m ping

ansible all -a "containerd --version"

ansible all -a "systemctl is-active containerd"

ansible all -m shell -a \
"grep 'SystemdCgroup = true' /etc/containerd/config.toml"

ansible all -a "ls -l /run/containerd/containerd.sock"

ansible all -a "kubeadm version -o short"

ansible all -a "kubelet --version"

ansible all -a "kubectl version --client"

ansible all -m shell -a \
"apt-mark showhold | grep -E 'kubelet|kubeadm|kubectl'"

ansible all -a "sysctl net.ipv4.ip_forward"

ansible all -m shell -a \
"lsmod | grep -E 'overlay|br_netfilter'"

ansible all -m shell -a \
"stat -fc %T /sys/fs/cgroup/"
```

---

# Definition of Done

```text
Phase 2
Container Runtime & Kubernetes Packages
        │
        ├── containerd installed             [✓]
        ├── containerd running               [✓]
        ├── CRI enabled                      [✓]
        ├── SystemdCgroup = true             [✓]
        ├── CRI socket available             [✓]
        │
        ├── Kubernetes repository added      [✓]
        ├── kubelet installed                [✓]
        ├── kubeadm installed                [✓]
        ├── kubectl installed                [✓]
        ├── packages held                    [✓]
        │
        └── all 5 nodes validated            [✓]
```

---

# Phase 2 Status

**COMPLETED**

After passing this phase, we need to configure the **HA API Endpoint** in Phase 3.

The next phase will prepare:

```text
Pacemaker / Corosync
        +
Keepalived
        +
HAProxy
```

and then perform a real test against:

```text
10.10.10.100:6443
```

before running:

```text
kubeadm init
```
