# Phase 4: Kubernetes HA Cluster Bootstrap with kubeadm

## Objective

In this phase, I bootstrap the Kubernetes cluster on top of the HA API endpoint prepared in Phase 3.

The target topology is:

```text
                       api.k8s.lab:8443
                              │
                              ▼
                       10.10.10.100
                              │
                           HAProxy
                              │
                 ┌────────────┼────────────┐
                 │            │            │
                 ▼            ▼            ▼
             k8s-cp-01    k8s-cp-02    k8s-cp-03
               :6443        :6443        :6443
                 │            │            │
                etcd         etcd         etcd
```

I use a **stacked etcd topology**, which means every Kubernetes control-plane node also runs one local etcd member.

Final control-plane layout:

```text
k8s-cp-01  → kube-apiserver + controller-manager + scheduler + etcd
k8s-cp-02  → kube-apiserver + controller-manager + scheduler + etcd
k8s-cp-03  → kube-apiserver + controller-manager + scheduler + etcd
```

Kubeadm manages the stacked etcd topology automatically when local etcd is used. :contentReference[oaicite:3]{index=3}

---

## Component Versions

```text
Kubernetes : v1.36.3
kubeadm    : v1.36.3
kubelet    : v1.36.3
kubectl    : v1.36.3
containerd : 2.2.2
Calico     : v3.32.1
```

For Kubernetes v1.36, the kubeadm configuration API used in this lab is:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
```

The cluster-wide control-plane endpoint is:

```yaml
controlPlaneEndpoint: "api.k8s.lab:8443"
```

This endpoint points to the HAProxy/VIP layer created in Phase 3.

---

# Phase 4 Repository Structure

A new Ansible role is created for Kubernetes cluster bootstrap:

```text
ansible/
├── playbooks/
│   ├── prepare.yml
│   ├── containerd.yml
│   ├── kubernetes.yml
│   ├── ha.yml
│   ├── bootstrap.yml
│   └── site.yml
│
└── roles/
    ├── common/
    ├── containerd/
    ├── kubernetes_packages/
    ├── ha/
    │
    └── kubernetes_cluster/
        ├── defaults/
        │   └── main.yml
        │
        ├── tasks/
        │   ├── main.yml
        │   ├── preflight.yml
        │   ├── init.yml
        │   ├── join_control_plane.yml
        │   └── join_workers.yml
        │
        └── templates/
            └── kubeadm-init.yaml.j2
```

Create the structure:

```bash
mkdir -p ansible/roles/kubernetes_cluster/{defaults,tasks,templates}

touch ansible/roles/kubernetes_cluster/defaults/main.yml
touch ansible/roles/kubernetes_cluster/tasks/main.yml
touch ansible/roles/kubernetes_cluster/tasks/preflight.yml
touch ansible/roles/kubernetes_cluster/tasks/init.yml
touch ansible/roles/kubernetes_cluster/tasks/join_control_plane.yml
touch ansible/roles/kubernetes_cluster/tasks/join_workers.yml
touch ansible/roles/kubernetes_cluster/templates/kubeadm-init.yaml.j2

touch ansible/playbooks/bootstrap.yml
```

---

# Kubernetes Cluster Variables

File:

```text
ansible/roles/kubernetes_cluster/defaults/main.yml
```

```yaml
---
kubernetes_version: "v1.36.3"

kubernetes_api_domain: "api.k8s.lab"
kubernetes_api_vip: "10.10.10.100"
kubernetes_api_port: 8443
kubernetes_api_backend_port: 6443

kubernetes_control_plane_endpoint: >-
  {{ kubernetes_api_domain }}:{{ kubernetes_api_port }}

kubernetes_pod_subnet: "10.244.0.0/16"
kubernetes_service_subnet: "10.96.0.0/12"

containerd_socket: "unix:///run/containerd/containerd.sock"

kubeadm_config_dir: "/etc/kubernetes"
kubeadm_config_file: "/etc/kubernetes/kubeadm-init.yaml"
```

The network ranges are the same values selected during the initial architecture design:

```text
Pod CIDR     : 10.244.0.0/16
Service CIDR : 10.96.0.0/12
```

---

# kubeadm Configuration

Template:

```text
ansible/roles/kubernetes_cluster/templates/kubeadm-init.yaml.j2
```

```yaml
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration

localAPIEndpoint:
  advertiseAddress: "{{ hostvars[groups['control_plane'][0]]['ansible_host'] }}"
  bindPort: {{ kubernetes_api_backend_port }}

nodeRegistration:
  name: "{{ groups['control_plane'][0] }}"
  criSocket: "{{ containerd_socket }}"
  imagePullPolicy: IfNotPresent

---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration

clusterName: "{{ cluster_name }}"

kubernetesVersion: "{{ kubernetes_version }}"

controlPlaneEndpoint: "{{ kubernetes_control_plane_endpoint }}"

networking:
  podSubnet: "{{ kubernetes_pod_subnet }}"
  serviceSubnet: "{{ kubernetes_service_subnet }}"

apiServer:
  certSANs:
    - "{{ kubernetes_api_domain }}"
    - "{{ kubernetes_api_vip }}"
    - "{{ hostvars['k8s-cp-01']['ansible_host'] }}"
    - "{{ hostvars['k8s-cp-02']['ansible_host'] }}"
    - "{{ hostvars['k8s-cp-03']['ansible_host'] }}"

etcd:
  local:
    dataDir: /var/lib/etcd
```

There are two important endpoint concepts:

```text
localAPIEndpoint
└── 10.10.10.11:6443
    Local kube-apiserver instance

controlPlaneEndpoint
└── api.k8s.lab:8443
    Cluster-wide HA endpoint
```

The API certificate also includes the VIP, DNS name, and all control-plane IP addresses as SANs.

---

# kubeadm Configuration Validation

Before initializing the cluster, the rendered kubeadm configuration is validated.

File:

```text
ansible/roles/kubernetes_cluster/tasks/init.yml
```

```yaml
---
- name: Deploy kubeadm init configuration
  ansible.builtin.template:
    src: kubeadm-init.yaml.j2
    dest: "{{ kubeadm_config_file }}"
    owner: root
    group: root
    mode: "0600"

- name: Validate kubeadm configuration
  ansible.builtin.command:
    cmd: "kubeadm config validate --config {{ kubeadm_config_file }}"
  changed_when: false
```

Initial playbook:

```yaml
---
- name: Bootstrap Kubernetes cluster
  hosts: control_plane
  become: true
  gather_facts: true

  roles:
    - kubernetes_cluster
```

Validation completed successfully before running `kubeadm init`.

---

# Preflight Validation

Before cluster bootstrap, several preflight checks are performed.

File:

```text
ansible/roles/kubernetes_cluster/tasks/preflight.yml
```

```yaml
---
- name: Check swap is disabled
  ansible.builtin.command:
    cmd: swapon --show
  register: swap_status
  changed_when: false
  failed_when: swap_status.stdout != ""

- name: Check containerd is active
  ansible.builtin.command:
    cmd: systemctl is-active containerd
  register: containerd_status
  changed_when: false
  failed_when: containerd_status.stdout != "active"

- name: Check kubelet is installed
  ansible.builtin.command:
    cmd: kubelet --version
  changed_when: false

- name: Check kubeadm is installed
  ansible.builtin.command:
    cmd: kubeadm version -o short
  changed_when: false

- name: Check API endpoint DNS resolution
  ansible.builtin.command:
    cmd: getent hosts {{ kubernetes_api_domain }}
  changed_when: false

- name: Check HAProxy endpoint is reachable
  ansible.builtin.shell:
    cmd: "timeout 3 bash -c '</dev/tcp/{{ kubernetes_api_vip }}/{{ kubernetes_api_port }}'"
    executable: /bin/bash
  changed_when: false
```

The following conditions must be satisfied before bootstrap:

```text
Swap disabled               ✓
containerd active           ✓
kubelet installed           ✓
kubeadm installed           ✓
api.k8s.lab resolves        ✓
10.10.10.100:8443 reachable ✓
```

---

# Initialize the First Control Plane

The first Kubernetes control-plane node is initialized only once.

```yaml
- name: Initialize first Kubernetes control-plane node
  ansible.builtin.command:
    cmd: "kubeadm init --config {{ kubeadm_config_file }} --upload-certs"
  args:
    creates: /etc/kubernetes/admin.conf
  register: kubeadm_init

- name: Show kubeadm init output
  ansible.builtin.debug:
    var: kubeadm_init.stdout_lines
  when: kubeadm_init is changed
```

Using:

```yaml
creates: /etc/kubernetes/admin.conf
```

prevents Ansible from running `kubeadm init` again on an already initialized control plane.

The first control-plane node was initialized successfully:

```text
k8s-cp-01
├── kube-apiserver
├── kube-controller-manager
├── kube-scheduler
├── etcd
└── kubelet
```

---

# Troubleshooting: Kubernetes Registry Access

During the first bootstrap attempt, kubeadm failed while pulling images from:

```text
registry.k8s.io
```

The error was:

```text
403 Forbidden
```

The failure affected:

```text
kube-apiserver
kube-controller-manager
kube-scheduler
kube-proxy
CoreDNS
pause
etcd
```

The issue was not related to kubeadm configuration.

Direct validation showed:

```bash
curl -I -L https://registry.k8s.io/v2/
```

Initially returned:

```text
HTTP/2 403
```

After fixing the network path, it returned:

```text
HTTP/2 200
docker-distribution-api-version: registry/2.0
```

After network connectivity was restored, `kubeadm init` completed successfully.

---

# Install Calico CNI

After the first control plane was initialized, the node was initially:

```text
NotReady
```

and CoreDNS pods were:

```text
Pending
```

This was expected because no CNI plugin had been installed yet.

Calico `v3.32.1` was selected.

The Tigera Operator and custom resources were downloaded:

```bash
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/tigera-operator.yaml

curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.32.1/manifests/custom-resources.yaml
```

The default Calico IP pool was:

```yaml
cidr: 192.168.0.0/16
```

Because the Kubernetes Pod CIDR is:

```text
10.244.0.0/16
```

the Calico configuration was changed to:

```yaml
cidr: 10.244.0.0/16
```

Calico documentation explicitly recommends ensuring the configured Calico IP pool matches the Pod network selected for the cluster. :contentReference[oaicite:4]{index=4}

The operator was then installed:

```bash
kubectl apply -f tigera-operator.yaml
```

After the operator CRDs became available:

```bash
kubectl apply -f custom-resources.yaml
```

The created resources included:

```text
Installation
APIServer
Goldmane
Whisker
```

Calico components eventually reached:

```text
calico-node              Running
calico-kube-controllers  Running
calico-apiserver         Running
calico-typha             Running
csi-node-driver          Running
CoreDNS                  Running
```

and the first control-plane node changed from:

```text
NotReady
```

to:

```text
Ready
```

The Tigera Operator installs Calico resources under the `calico-system` namespace. :contentReference[oaicite:5]{index=5}

---

# Troubleshooting: Calico Image Pull

While adding `k8s-cp-02`, Calico temporarily entered:

```text
Init:ImagePullBackOff
```

The event showed:

```text
Failed to pull image "quay.io/calico/node:v3.32.1"

short read
unexpected EOF
```

This was a partial image-download/network issue, not a Calico configuration problem.

After the image pull completed successfully, the node automatically recovered:

```text
calico-node        1/1 Running
csi-node-driver    2/2 Running
k8s-cp-02          Ready
```

---

# Join Additional Control Plane Nodes

The additional control-plane nodes were initially joined manually to validate the procedure.

The generic command is:

```bash
kubeadm join api.k8s.lab:8443 \
  --token <BOOTSTRAP_TOKEN> \
  --discovery-token-ca-cert-hash sha256:<CA_HASH> \
  --control-plane \
  --certificate-key <CERTIFICATE_KEY>
```

> Bootstrap tokens and certificate keys are intentionally not stored in the repository or README.

Control-plane nodes were joined one at a time:

```text
k8s-cp-02
k8s-cp-03
```

Joining control-plane nodes sequentially is also the recommended kubeadm HA workflow. :contentReference[oaicite:6]{index=6}

---

# Stacked etcd Validation

After joining all three control-plane nodes:

```bash
kubectl -n kube-system exec etcd-k8s-cp-01 -- \
  etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  member list -w table
```

Final membership:

```text
k8s-cp-01   started   learner=false
k8s-cp-02   started   learner=false
k8s-cp-03   started   learner=false
```

This confirms:

```text
3 etcd members
3 started members
0 learners
```

The final stacked etcd topology is:

```text
k8s-cp-01
└── etcd member 1

k8s-cp-02
└── etcd member 2

k8s-cp-03
└── etcd member 3
```

---

# Join Worker Nodes

Workers use a normal kubeadm join command:

```bash
kubeadm join api.k8s.lab:8443 \
  --token <BOOTSTRAP_TOKEN> \
  --discovery-token-ca-cert-hash sha256:<CA_HASH>
```

The worker nodes are:

```text
k8s-worker-01
k8s-worker-02
```

After Calico initialized on each worker, both reached:

```text
Ready
```

---

# Final Cluster State

The final Kubernetes cluster contains five nodes:

```text
NAME            STATUS   ROLES
k8s-cp-01       Ready    control-plane
k8s-cp-02       Ready    control-plane
k8s-cp-03       Ready    control-plane
k8s-worker-01   Ready    <none>
k8s-worker-02   Ready    <none>
```

Summary:

```text
3 × Control Plane   Ready
2 × Worker          Ready
3 × etcd members    Healthy
Kubernetes          v1.36.3
containerd          2.2.2
Calico              Running
CoreDNS             Running
```

Final topology:

```text
                       api.k8s.lab:8443
                              │
                              ▼
                    10.10.10.100:8443
                              │
                           HAProxy
                              │
                 ┌────────────┼────────────┐
                 │            │            │
                 ▼            ▼            ▼
             k8s-cp-01    k8s-cp-02    k8s-cp-03
               Ready         Ready         Ready
                etcd          etcd          etcd
                 │             │             │
                 └─────────────┼─────────────┘
                               │
                     3-member stacked etcd

                ┌─────────────────────────┐
                │                         │
                ▼                         ▼
          k8s-worker-01              k8s-worker-02
              Ready                      Ready
```

---

# Automating Control Plane Membership

The initial join procedure was validated manually.

The next step was making Phase 4 fully reproducible using Ansible.

File:

```text
ansible/roles/kubernetes_cluster/tasks/join_control_plane.yml
```

```yaml
---
- name: Check whether control-plane node is already joined
  ansible.builtin.stat:
    path: /etc/kubernetes/kubelet.conf
  register: control_plane_joined

- name: Generate worker-style join command
  ansible.builtin.command:
    cmd: kubeadm token create --print-join-command
  delegate_to: "{{ groups['control_plane'][0] }}"
  run_once: true
  register: kubeadm_join_command
  changed_when: false
  when: not control_plane_joined.stat.exists

- name: Upload control-plane certificates and generate certificate key
  ansible.builtin.command:
    cmd: kubeadm init phase upload-certs --upload-certs
  delegate_to: "{{ groups['control_plane'][0] }}"
  run_once: true
  register: kubeadm_upload_certs
  changed_when: false
  when: not control_plane_joined.stat.exists

- name: Set control-plane certificate key
  ansible.builtin.set_fact:
    kubeadm_certificate_key: "{{ kubeadm_upload_certs.stdout_lines | last }}"
  when: not control_plane_joined.stat.exists

- name: Join additional control-plane node
  ansible.builtin.command:
    cmd: >-
      {{ kubeadm_join_command.stdout }}
      --control-plane
      --certificate-key {{ kubeadm_certificate_key }}
  args:
    creates: /etc/kubernetes/kubelet.conf
  when: not control_plane_joined.stat.exists
```

This approach avoids storing temporary bootstrap tokens and certificate keys in Git.

---

# Automating Worker Membership

File:

```text
ansible/roles/kubernetes_cluster/tasks/join_workers.yml
```

```yaml
---
- name: Check whether worker node is already joined
  ansible.builtin.stat:
    path: /etc/kubernetes/kubelet.conf
  register: worker_joined

- name: Generate Kubernetes worker join command
  ansible.builtin.command:
    cmd: kubeadm token create --print-join-command
  delegate_to: "{{ groups['control_plane'][0] }}"
  run_once: true
  register: worker_join_command
  changed_when: false
  when: not worker_joined.stat.exists

- name: Join worker node to Kubernetes cluster
  ansible.builtin.command:
    cmd: "{{ worker_join_command.stdout }}"
  args:
    creates: /etc/kubernetes/kubelet.conf
  when: not worker_joined.stat.exists
```

---

# Final Task Flow

File:

```text
ansible/roles/kubernetes_cluster/tasks/main.yml
```

```yaml
---
- name: Run Kubernetes preflight checks
  ansible.builtin.import_tasks: preflight.yml

- name: Configure first control-plane node
  ansible.builtin.import_tasks: init.yml
  when: inventory_hostname == groups['control_plane'][0]

- name: Join additional control-plane nodes
  ansible.builtin.import_tasks: join_control_plane.yml
  when:
    - inventory_hostname in groups['control_plane']
    - inventory_hostname != groups['control_plane'][0]

- name: Join worker nodes
  ansible.builtin.import_tasks: join_workers.yml
  when: inventory_hostname in groups['workers']
```

The final bootstrap playbook must target both control-plane and worker groups:

```yaml
---
- name: Bootstrap Kubernetes cluster
  hosts: control_plane:workers
  become: true
  gather_facts: true

  roles:
    - kubernetes_cluster
```

---

# Ansible Idempotency Validation

The final bootstrap automation was executed again after all five nodes were part of the cluster.

Final recap:

```text
k8s-cp-01       changed=0 failed=0
k8s-cp-02       changed=0 failed=0
k8s-cp-03       changed=0 failed=0
k8s-worker-01   changed=0 failed=0
k8s-worker-02   changed=0 failed=0
```

This confirms that:

```text
kubeadm init is not repeated        ✓
control-plane nodes are not rejoined ✓
worker nodes are not rejoined        ✓
existing cluster state is preserved  ✓
Phase 4 automation is idempotent      ✓
```

---

# Kubernetes API HA Validation

The kubeconfig points to the shared HA endpoint:

```bash
kubectl config view \
  --minify \
  -o jsonpath='{.clusters[0].cluster.server}'
```

Result:

```text
https://api.k8s.lab:8443
```

This proves that `kubectl` is not directly using an individual control-plane node.

The VIP owner was then failed over using Keepalived.

During VIP failover:

```text
VIP:
k8s-cp-01
    │
    ▼
k8s-cp-02
```

`kubectl` continued to work through:

```text
https://api.k8s.lab:8443
```

Therefore:

```text
VIP Failover               PASS ✓
HAProxy Failover           PASS ✓
Kubernetes API Continuity  PASS ✓
```

This validates the Kubernetes API HA endpoint prepared in Phase 3 using the real kube-apiserver backends created in Phase 4.

---

# Phase 4 Final Validation

| Validation | Result |
|---|:---:|
| kubeadm config validation | ✓ |
| Swap disabled | ✓ |
| containerd active | ✓ |
| HA endpoint reachable | ✓ |
| First control plane initialized | ✓ |
| Calico CNI installed | ✓ |
| CoreDNS running | ✓ |
| `k8s-cp-01` Ready | ✓ |
| `k8s-cp-02` Ready | ✓ |
| `k8s-cp-03` Ready | ✓ |
| `k8s-worker-01` Ready | ✓ |
| `k8s-worker-02` Ready | ✓ |
| 3-member stacked etcd | ✓ |
| All etcd members started | ✓ |
| No etcd learners | ✓ |
| Dynamic control-plane join | ✓ |
| Dynamic worker join | ✓ |
| No hard-coded join token | ✓ |
| No hard-coded certificate key | ✓ |
| Ansible idempotency | ✓ |
| HA kubeconfig endpoint | ✓ |
| Kubernetes API continuity during VIP failover | ✓ |

---

## Phase 4 Status

```text
Phase 4: Kubernetes HA Cluster Bootstrap

kubeadm Bootstrap       PASS ✓
3 Control Planes        PASS ✓
2 Workers               PASS ✓
Stacked etcd            PASS ✓
Calico CNI              PASS ✓
CoreDNS                 PASS ✓
Dynamic Join            PASS ✓
Idempotency             PASS ✓
API HA Continuity       PASS ✓
```

Final result:

```text
5-node Kubernetes HA cluster
        │
        ├── 3 control-plane nodes
        ├── 3-member stacked etcd
        ├── 2 worker nodes
        ├── Calico CNI
        └── HA API endpoint
            api.k8s.lab:8443
```