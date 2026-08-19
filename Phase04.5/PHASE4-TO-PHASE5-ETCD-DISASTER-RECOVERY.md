# Incident Case Study: Kubernetes Control Plane & etcd Disaster Recovery

## Production-Style Failure Scenario After Abrupt Infrastructure Restart

After completing Phase 4, the Kubernetes cluster was healthy and fully operational:

```text
3 × Control Plane     Ready
2 × Worker            Ready
3 × etcd Members      Healthy
Calico                Running
HAProxy + VIP         Healthy
Kubernetes API HA     PASS
```

The control-plane topology was:

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
                  │             │             │
                  ▼             ▼             ▼
                etcd          etcd          etcd

                         3-member quorum
```

Then the VMware infrastructure was abruptly restarted.

This represents an important **production-style failure scenario**.

Examples of similar real-world events include:

```text
Hypervisor power loss
Host crash
Storage interruption
Datacenter power failure
Unexpected VM termination
Filesystem/storage inconsistency
```

> The failure happened immediately after the VMware restart. The logs prove that etcd data corruption existed after the restart, but they do not by themselves prove that the restart was the direct physical cause of the corruption.

This incident became an opportunity to test actual Kubernetes control-plane disaster recovery rather than only planned HA failover.

---

# 1. Initial Symptom

After the infrastructure came back online, the first command executed on `k8s-cp-01` was:

```bash
kubectl get nodes
```

Instead of returning the cluster nodes, kubectl initially attempted:

```text
http://localhost:8080
```

and failed.

The kubeconfig was then verified:

```bash
echo $KUBECONFIG
```

Result:

```text
/etc/kubernetes/admin.conf
```

The file also existed:

```bash
ls -l /etc/kubernetes/admin.conf
```

So kubeconfig itself was not the underlying problem.

The next kubectl attempt returned:

```text
Unable to connect to the server: EOF
```

At this point the problem had moved from:

```text
kubectl configuration
```

to:

```text
Kubernetes API availability
```

---

# 2. Validate the HA API Path

The configured API endpoint was verified:

```bash
kubectl config view \
  --minify \
  -o jsonpath='{.clusters[0].cluster.server}'; echo
```

Result:

```text
https://api.k8s.lab:8443
```

DNS was also correct:

```bash
getent hosts api.k8s.lab
```

Result:

```text
10.10.10.100    api.k8s.lab
```

The VIP existed on `k8s-cp-01`:

```bash
ip -br addr | grep 10.10.10.100
```

Result:

```text
ens33  UP  10.10.10.11/24 10.10.10.100/24
```

HAProxy was listening:

```bash
ss -lntp | grep 8443
```

Result:

```text
0.0.0.0:8443
```

This meant:

```text
DNS        ✓
VIP        ✓
HAProxy    ✓
kubeconfig ✓
```

The problem was therefore deeper in the Kubernetes control-plane backend.

---

# 3. HAProxy Reported No Healthy API Backend

HAProxy logs showed repeated entries containing:

```text
kubernetes-api-backend/<NOSRV>
```

This was an important observation.

HAProxy itself was alive and accepting traffic, but it could not find a healthy Kubernetes API backend.

The path therefore looked like:

```text
kubectl
   │
   ▼
api.k8s.lab
   │
   ▼
VIP
   │
   ▼
HAProxy
   │
   └────── X No healthy kube-apiserver backend
```

The HA layer was functioning.

The Kubernetes control plane was not.

---

# 4. Direct kube-apiserver Health Check

To remove HAProxy from the equation, the API server was tested directly:

```bash
curl -k https://127.0.0.1:6443/readyz
```

and:

```bash
curl -k https://10.10.10.11:6443/readyz
```

Both returned:

```text
Could not connect to server
```

The VIP endpoint also failed:

```bash
curl -k https://10.10.10.100:8443/readyz
```

Result:

```text
Send failure: Broken pipe
```

This confirmed:

```text
HAProxy was not the root cause.

kube-apiserver itself was unavailable.
```

---

# 5. kubelet Was Running

The kubelet service was inspected:

```bash
systemctl status kubelet
```

It was:

```text
active (running)
```

However, kubelet repeatedly logged:

```text
Unable to register mirror pod because node is not registered yet
node "k8s-cp-01" not found
```

This was a **symptom**, not the root cause.

Kubelet was alive, but it could not communicate with a functioning Kubernetes API.

---

# 6. Static Control-Plane Containers Were Crashing

The static Pod manifests were still present:

```bash
ls -l /etc/kubernetes/manifests/
```

Result:

```text
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

But containerd showed no active etcd or kube-apiserver tasks.

This meant:

```text
Manifest exists
      │
      ▼
kubelet creates static Pod
      │
      ▼
container starts
      │
      ▼
container crashes
      │
      ▼
restart loop
```

So the next investigation moved to etcd and API server container logs.

---

# 7. kube-apiserver Was Failing Because etcd Was Down

The `kube-apiserver` logs showed repeated attempts to connect to:

```text
127.0.0.1:2379
```

and failures such as:

```text
dial tcp 127.0.0.1:2379: connect: connection refused
```

Eventually the API server terminated with:

```text
error creating storage factory: context deadline exceeded
```

This established the dependency chain:

```text
etcd failure
    │
    ▼
kube-apiserver cannot initialize storage
    │
    ▼
kube-apiserver exits
    │
    ▼
HAProxy marks backend unhealthy
    │
    ▼
kubectl receives EOF
```

The API server itself was therefore **not the root cause**.

The investigation moved one level deeper to etcd.

---

# 8. Root Cause on k8s-cp-01: Corrupted etcd bbolt Database

The etcd logs on `k8s-cp-01` showed it opening:

```text
/var/lib/etcd/member/snap/db
```

and then panicking:

```text
panic: assertion failed:
Page expected to be: 664,
but self identifies as ...
```

This was a serious persistent database integrity problem inside the etcd/bbolt database.

The state of `cp-01` was therefore:

```text
k8s-cp-01
    │
    ├── kubelet           Running
    ├── static manifests  Present
    │
    ├── etcd              CRASH
    │       │
    │       └── corrupted bbolt DB
    │
    └── kube-apiserver    CRASH
            │
            └── etcd unavailable
```

---

# 9. Inspecting k8s-cp-02

Because etcd is a distributed system, recovery must not be attempted blindly from a single node.

The second control-plane member was investigated.

Its etcd instance could start, but it could not communicate with either peer:

```text
10.10.10.11:2380 → connection refused
10.10.10.13:2380 → connection refused
```

The member repeatedly attempted elections:

```text
starting a new election
became pre-candidate
received 1 MsgPreVoteResp
```

But it only had its own vote.

For a three-member etcd cluster:

```text
Members = 3

Required majority:

floor(3 / 2) + 1 = 2
```

Therefore:

```text
cp-02 alone = 1 vote
required     = 2 votes

No quorum.
```

The etcd health check reflected exactly this situation:

```text
non_learner        ok
data_corruption    ok
serializable_read  ok
linearizable_read  FAILED
```

So `cp-02` appeared to contain usable data, but could not safely operate alone.

---

# 10. Inspecting k8s-cp-03

The third etcd member was then inspected.

Unlike `cp-01`, its main bbolt database opened successfully.

However etcd attempted snapshot recovery and expected:

```text
/var/lib/etcd/member/snap/000000000003e287.snap.db
```

The file did not exist.

etcd then terminated with:

```text
failed to recover v3 backend from snapshot
```

and:

```text
panic: failed to recover v3 backend from snapshot
```

The failure state was now fully understood:

```text
┌──────────────┬──────────────────────────────────┐
│ Node         │ etcd State                       │
├──────────────┼──────────────────────────────────┤
│ k8s-cp-01    │ Corrupted bbolt database         │
│ k8s-cp-02    │ Usable data, but no quorum       │
│ k8s-cp-03    │ Missing required snapshot state  │
└──────────────┴──────────────────────────────────┘
```

This was no longer a simple single-member failure.

It was an **etcd majority failure**.

---

# 11. Why We Did NOT Run kubeadm reset

At this stage it would have been easy to make the situation worse with commands such as:

```bash
kubeadm reset
```

or:

```bash
rm -rf /var/lib/etcd
```

or by reinitializing Kubernetes.

None of these were performed.

The Kubernetes object state was still potentially recoverable from etcd.

The recovery principle was:

```text
STOP
 │
 ├── Do not reset Kubernetes
 ├── Do not destroy etcd state
 ├── Do not recreate certificates
 ├── Do not re-run kubeadm init
 │
 ▼
Preserve the best surviving etcd state first
```

---

# 12. Selecting the Best etcd Survivor

`k8s-cp-02` was selected as the recovery source because:

```text
cp-01 → database corruption panic

cp-03 → snapshot recovery panic

cp-02 → database readable
        raft running
        no local corruption error
        only quorum unavailable
```

Before doing anything destructive, kubelet was stopped:

```bash
systemctl stop kubelet
```

The etcd process was confirmed stopped.

Then the complete data directory was backed up:

```bash
cp -a /var/lib/etcd \
      /var/lib/etcd.backup-20260815
```

The survivor database itself was preserved separately:

```bash
cp -a /var/lib/etcd/member/snap/db \
      /root/etcd-cp02-survivor.db
```

Result:

```text
Survivor DB             26 MB
Original etcd directory 267 MB
Backup etcd directory   267 MB
```

This was a critical safety checkpoint.

---

# 13. Validate the Survivor Before Restore

The host did not contain the `etcdutl` binary.

Instead of installing another package during the incident, the already-present Kubernetes etcd image was used:

```text
registry.k8s.io/etcd:3.6.8-0
```

The snapshot was inspected using `etcdutl` from inside that image:

```bash
ctr -n k8s.io run --rm \
  --mount type=bind,src=/root,dst=/recovery,options=rbind:ro \
  registry.k8s.io/etcd:3.6.8-0 \
  etcdutl-check \
  /usr/local/bin/etcdutl snapshot status \
  /recovery/etcd-cp02-survivor.db \
  -w table
```

Result:

```text
+----------+----------+------------+------------+---------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE | VERSION |
+----------+----------+------------+------------+---------+
| b8ee2cac |   168055 |        525 |     9.8 MB |   3.6.0 |
+----------+----------+------------+------------+---------+
```

The survivor database was therefore readable by etcd tooling.

---

# 14. Non-Destructive Restore Test

Before touching the real data directory, a restore was performed into:

```text
/var/lib/etcd-restore-test
```

instead of:

```text
/var/lib/etcd
```

This was intentionally non-destructive.

Example:

```bash
ctr -n k8s.io run --rm \
  --mount type=bind,src=/root,dst=/recovery,options=rbind:ro \
  --mount type=bind,src=/var/lib,dst=/var/lib,options=rbind:rw \
  registry.k8s.io/etcd:3.6.8-0 \
  etcdutl-restore-test \
  /usr/local/bin/etcdutl snapshot restore \
  /recovery/etcd-cp02-survivor.db \
  --skip-hash-check \
  --name k8s-cp-02 \
  --initial-cluster \
  "k8s-cp-01=https://10.10.10.11:2380,k8s-cp-02=https://10.10.10.12:2380,k8s-cp-03=https://10.10.10.13:2380" \
  --initial-cluster-token "abrnoc-etcd-recovery-20260815" \
  --initial-advertise-peer-urls \
  "https://10.10.10.12:2380" \
  --data-dir /var/lib/etcd-restore-test
```

The restore succeeded and produced:

```text
member/
├── snap/
│   ├── *.snap
│   └── db
│
└── wal/
    └── *.wal
```

This validated the recovery source before any original etcd directory was replaced.

---

# 15. Freeze All Control Planes

Before the real cluster restore, kubelet was stopped on every control-plane node:

```bash
systemctl stop kubelet
```

This effectively stopped the kubeadm static Pods:

```text
etcd
kube-apiserver
kube-controller-manager
kube-scheduler
```

The original etcd directories on all three nodes were backed up:

```bash
cp -a /var/lib/etcd \
      /var/lib/etcd.backup-20260815
```

This ensured that recovery remained reversible.

---

# 16. Copy the Survivor Database to Every Control Plane

The database from `cp-02` was copied to:

```text
cp-01
cp-03
```

Then SHA-256 hashes were compared.

All three produced:

```text
2eac9582cbf3cde6208a8eab4c0e45c89693c41de5acad1039df3a1e4d3987f3
```

This proved that:

```text
cp-01 recovery DB
        =
cp-02 recovery DB
        =
cp-03 recovery DB
```

Every restored member would therefore start from exactly the same Kubernetes keyspace.

---

# 17. Revision-Bump Restore Attempt

For Kubernetes, revision bump is useful because controllers maintain informer/watch caches.

Therefore a restore was attempted with:

```text
--bump-revision 1000000000
--mark-compacted
```

The restore began correctly and calculated:

```text
latest revision:     168055
bump amount:         1000000000
new latest revision: 1000168055
```

However, bbolt then detected another internal database inconsistency:

```text
panic: freepages: failed to get all reachable pages
```

The restore did **not** complete.

Only:

```text
member/snap/db
```

was created; the required snapshot and WAL structure was absent.

Therefore the partially restored directories were rejected.

This was an important operational rule:

```text
A restore command starting successfully
does NOT mean the restore completed successfully.
```

Validation of the resulting directory matters.

---

# 18. Recovery Decision: Restore Without Revision Bump

The same source database had already completed a clean non-bumped restore.

Therefore recovery continued without:

```text
--bump-revision
--mark-compacted
```

This was a deliberate trade-off forced by the actual condition of the surviving database.

The immediate priority was obtaining a consistent etcd cluster from the only usable surviving state.

---

# 19. Restore a New Logical Three-Member etcd Cluster

The membership used for the restored cluster was:

```text
k8s-cp-01=https://10.10.10.11:2380
k8s-cp-02=https://10.10.10.12:2380
k8s-cp-03=https://10.10.10.13:2380
```

with cluster token:

```text
abrnoc-etcd-recovery-20260815
```

Each node used:

```text
same snapshot
same initial-cluster
same cluster token

but:

different --name
different --initial-advertise-peer-urls
```

## Restore k8s-cp-01

```bash
ctr -n k8s.io run --rm \
  --mount type=bind,src=/root,dst=/recovery,options=rbind:ro \
  --mount type=bind,src=/var/lib,dst=/var/lib,options=rbind:rw \
  registry.k8s.io/etcd:3.6.8-0 \
  etcdutl-restore-cp01 \
  /usr/local/bin/etcdutl snapshot restore \
  /recovery/etcd-recovery.db \
  --skip-hash-check \
  --name k8s-cp-01 \
  --initial-cluster \
  "k8s-cp-01=https://10.10.10.11:2380,k8s-cp-02=https://10.10.10.12:2380,k8s-cp-03=https://10.10.10.13:2380" \
  --initial-cluster-token \
  "abrnoc-etcd-recovery-20260815" \
  --initial-advertise-peer-urls \
  "https://10.10.10.11:2380" \
  --data-dir /var/lib/etcd-restored
```

## Restore k8s-cp-02

```bash
ctr -n k8s.io run --rm \
  --mount type=bind,src=/root,dst=/recovery,options=rbind:ro \
  --mount type=bind,src=/var/lib,dst=/var/lib,options=rbind:rw \
  registry.k8s.io/etcd:3.6.8-0 \
  etcdutl-restore-cp02 \
  /usr/local/bin/etcdutl snapshot restore \
  /recovery/etcd-cp02-survivor.db \
  --skip-hash-check \
  --name k8s-cp-02 \
  --initial-cluster \
  "k8s-cp-01=https://10.10.10.11:2380,k8s-cp-02=https://10.10.10.12:2380,k8s-cp-03=https://10.10.10.13:2380" \
  --initial-cluster-token \
  "abrnoc-etcd-recovery-20260815" \
  --initial-advertise-peer-urls \
  "https://10.10.10.12:2380" \
  --data-dir /var/lib/etcd-restored
```

## Restore k8s-cp-03

```bash
ctr -n k8s.io run --rm \
  --mount type=bind,src=/root,dst=/recovery,options=rbind:ro \
  --mount type=bind,src=/var/lib,dst=/var/lib,options=rbind:rw \
  registry.k8s.io/etcd:3.6.8-0 \
  etcdutl-restore-cp03 \
  /usr/local/bin/etcdutl snapshot restore \
  /recovery/etcd-recovery.db \
  --skip-hash-check \
  --name k8s-cp-03 \
  --initial-cluster \
  "k8s-cp-01=https://10.10.10.11:2380,k8s-cp-02=https://10.10.10.12:2380,k8s-cp-03=https://10.10.10.13:2380" \
  --initial-cluster-token \
  "abrnoc-etcd-recovery-20260815" \
  --initial-advertise-peer-urls \
  "https://10.10.10.13:2380" \
  --data-dir /var/lib/etcd-restored
```

Every restore ended with:

```text
restored snapshot
```

and produced:

```text
/var/lib/etcd-restored/member/snap/*.snap
/var/lib/etcd-restored/member/snap/db
/var/lib/etcd-restored/member/wal/*.wal
```

---

# 20. Replace the Broken etcd Data Directories

Only after the restored directories had been verified were they activated.

On every control-plane node:

```bash
mv /var/lib/etcd \
   /var/lib/etcd.pre-recovery

mv /var/lib/etcd-restored \
   /var/lib/etcd

chown -R root:root /var/lib/etcd
```

At this point every node contained three safety layers:

```text
/var/lib/etcd
    └── restored active state

/var/lib/etcd.pre-recovery
    └── original failed state

/var/lib/etcd.backup-20260815
    └── separate incident backup
```

Nothing was irreversibly destroyed during recovery.

---

# 21. Controlled Startup

The control planes were not started simultaneously.

First:

```text
k8s-cp-02
k8s-cp-03
```

were started:

```bash
systemctl start kubelet
```

The purpose was to first obtain:

```text
2 / 3 etcd members
      │
      ▼
    Quorum
```

before bringing the final member online.

---

# 22. etcd Quorum Returned

After starting `cp-02` and `cp-03`, `cp-02` showed:

```text
10.10.10.12:2380   LISTEN
10.10.10.12:2379   LISTEN
127.0.0.1:2379     LISTEN
*:6443              LISTEN
```

More importantly, the Raft log showed:

```text
received 2 MsgVoteResp votes
became leader at term 2
```

This was the critical recovery event:

```text
cp-02
  │
  ├── vote from cp-02
  └── vote from cp-03
          │
          ▼
       Majority = 2
          │
          ▼
      Leader elected
          │
          ▼
      Quorum restored
```

Immediately afterwards etcd reported:

```text
ready to serve client requests
```

and:

```text
grpc service status changed → SERVING
```

This confirmed that the restored etcd cluster was operational.

---

# 23. Kubernetes API Recovered Automatically

Once etcd became functional, the static kube-apiserver Pod was able to initialize normally.

There was no need to:

```text
re-run kubeadm init
recreate PKI
regenerate kubeconfig
rejoin Kubernetes nodes
reinstall Calico
recreate Kubernetes objects
```

The recovery chain was:

```text
Restored etcd
     │
     ▼
Raft quorum
     │
     ▼
etcd leader
     │
     ▼
etcd client service
     │
     ▼
kube-apiserver storage available
     │
     ▼
kube-apiserver starts
     │
     ▼
HAProxy backend healthy
     │
     ▼
VIP/API becomes available
     │
     ▼
kubectl works again
```

---

# 24. Bring Back the Third Member

After quorum was established between `cp-02` and `cp-03`, the final member was started:

```bash
systemctl start kubelet
```

on:

```text
k8s-cp-01
```

The intended final membership validation is:

```bash
kubectl -n kube-system exec etcd-k8s-cp-02 -- \
  etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  member list -w table
```

Expected state:

```text
k8s-cp-01   started   learner=false
k8s-cp-02   started   learner=false
k8s-cp-03   started   learner=false
```

And:

```bash
kubectl get nodes -o wide
```

should return the complete five-node Kubernetes cluster.

---

# 25. Incident Timeline

```text
Abrupt VMware restart
        │
        ▼
Control-plane VMs reboot
        │
        ▼
etcd persistence problems appear
        │
        ├── cp-01 → corrupted bbolt database
        │
        ├── cp-02 → usable but isolated
        │
        └── cp-03 → snapshot recovery failure
        │
        ▼
etcd loses majority
        │
        ▼
No Raft quorum
        │
        ▼
kube-apiserver cannot access storage
        │
        ▼
All API backends become unhealthy
        │
        ▼
HAProxy reports <NOSRV>
        │
        ▼
kubectl → EOF
        │
        ▼
Incident investigation
        │
        ▼
cp-02 selected as best survivor
        │
        ▼
Freeze control planes
        │
        ▼
Backup all etcd data
        │
        ▼
Extract cp-02 member/snap/db
        │
        ▼
Validate with etcdutl
        │
        ▼
Non-destructive restore test
        │
        ▼
Copy identical DB to all CPs
        │
        ▼
Restore new 3-member logical etcd cluster
        │
        ▼
Replace old data directories
        │
        ▼
Start cp-02 + cp-03
        │
        ▼
Raft majority = 2
        │
        ▼
Leader elected
        │
        ▼
etcd operational
        │
        ▼
kube-apiserver recovers
        │
        ▼
HA API recovers
        │
        ▼
Start cp-01
        │
        ▼
3-member cluster restored
```

---

# 26. Why HAProxy and Keepalived Could Not Save This Failure

This incident demonstrates an important distinction:

```text
Load-Balancer HA
        ≠
Control-Plane State HA
```

Keepalived can move:

```text
10.10.10.100
```

between nodes.

HAProxy can route:

```text
:8443 → healthy :6443 backend
```

But neither can fix:

```text
all kube-apiserver backends unavailable
```

And kube-apiserver cannot function without usable etcd storage.

Therefore:

```text
VIP HA
  +
HAProxy HA
  +
Multiple kube-apiserver instances
  +
etcd quorum
  =
Complete Kubernetes control-plane HA
```

A load balancer only helps if at least one viable backend exists.

---

# 27. Why Three etcd Members Still Matter

A healthy three-member etcd cluster tolerates one member failure:

```text
3 members
majority = 2

1 failed
2 healthy
→ cluster continues
```

But:

```text
2 failed
1 healthy
→ no majority
→ cluster stops
```

The failure tolerance of the architecture was:

```text
3-member etcd
Failure tolerance = 1 member
```

---

# 28. Production Lessons Learned

## 28.1 Automated etcd Snapshots

The most important improvement is scheduled etcd backup.

A production backup should normally use:

```bash
etcdctl snapshot save
```

rather than relying on emergency recovery from:

```text
member/snap/db
```

Recommended operational model:

```text
etcd snapshot
      │
      ├── scheduled
      ├── encrypted
      ├── off-node
      ├── off-cluster
      ├── retention policy
      └── periodic restore test
```

A backup that has never been restored is not enough evidence of recoverability.

## 28.2 Store Backups Outside the Kubernetes Cluster

A snapshot stored only on:

```text
/var/lib/etcd
```

does not protect against:

```text
disk loss
VM loss
datastore corruption
host destruction
operator error
```

Production snapshots should be copied to independent storage.

## 28.3 Test Disaster Recovery Regularly

This incident showed that a theoretically valid recovery procedure can encounter unexpected storage behavior.

The restore test prevented damage to the only viable recovery copy.

Production DR exercises should explicitly test:

```text
snapshot creation
snapshot transfer
snapshot validation
clean restore
member reconstruction
quorum formation
Kubernetes API recovery
application validation
```

## 28.4 Monitor etcd Health

Important signals include:

```text
etcd member health
leader changes
proposal failures
database size
fsync latency
WAL latency
peer RTT
corruption alarms
quorum availability
```

## 28.5 Storage Quality Is Critical

etcd is one of the most storage-sensitive components in Kubernetes.

For production, etcd storage should be treated as critical infrastructure:

```text
Reliable persistent storage
Stable filesystem
Low fsync latency
Correct shutdown behavior
UPS / power protection
Hypervisor storage protection
Filesystem monitoring
Disk health monitoring
Backup validation
```

---

# 29. Operational Recovery Principles Learned

```text
1. Diagnose before resetting.

2. Follow the dependency chain:
   kubectl
      ↓
   VIP
      ↓
   HAProxy
      ↓
   kube-apiserver
      ↓
   etcd

3. Do not assume the load balancer is the problem.

4. Preserve surviving etcd data before modifying anything.

5. Stop writers before copying persistent state.

6. Back up every member before destructive recovery.

7. Validate the recovery source before restoring.

8. Restore into a temporary directory first.

9. Use the same snapshot for every reconstructed member.

10. Verify checksums after transferring recovery data.

11. Restore quorum before focusing on Kubernetes.

12. Keep the original failed data until recovery is validated.

13. Record failed recovery attempts as part of the incident.

14. Test backup restore before an actual disaster.
```

---

# 30. Failure Classification

| Layer | State |
|---|---|
| VMware infrastructure | Abrupt restart |
| Linux VMs | Recovered |
| containerd | Running |
| kubelet | Running |
| Keepalived | Running |
| VIP | Available |
| HAProxy | Running |
| etcd cp-01 | Failed — database corruption |
| etcd cp-02 | Usable — no quorum |
| etcd cp-03 | Failed — snapshot recovery |
| etcd quorum | **Lost** |
| kube-apiserver | Failed due to etcd |
| Kubernetes API | **Unavailable** |
| Worker nodes | Existing but unable to communicate with API |
| Recovery source | cp-02 |
| etcd disaster recovery | **Successful** |
| Raft quorum restoration | **Successful** |
| kube-apiserver recovery | **Successful** |

---

# 31. Recovery Safety Checkpoints

```text
[✓] kubeconfig verified
[✓] DNS verified
[✓] VIP verified
[✓] HAProxy verified
[✓] direct API endpoint tested
[✓] kubelet inspected
[✓] static manifests verified
[✓] etcd failure identified
[✓] all three members inspected
[✓] majority failure confirmed
[✓] best survivor identified
[✓] kubelet stopped before data copy
[✓] full data directory backup created
[✓] survivor DB extracted
[✓] snapshot status verified
[✓] test restore completed
[✓] recovery DB checksum verified on all nodes
[✓] restored directories created separately
[✓] restored directory structure verified
[✓] original data preserved
[✓] restored data activated
[✓] two members started first
[✓] quorum restored
[✓] leader election confirmed
[✓] etcd SERVING confirmed
[✓] kube-apiserver returned
```

---

# 32. Recovery Architecture

Before the incident:

```text
                    Kubernetes API
                           │
                           ▼
                  3-member stacked etcd
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
            cp-01        cp-02        cp-03
```

During failure:

```text
              ┌────────────┼────────────┐
              ▼            ▼            ▼
            cp-01        cp-02        cp-03
              X            ✓            X
        DB corruption   no quorum   snapshot failure

                         │
                         ▼
                    quorum = LOST
                         │
                         ▼
                 Kubernetes API DOWN
```

During recovery:

```text
                   cp-02 survivor DB
                          │
                    snapshot copy
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
           restore     restore     restore
           cp-01       cp-02       cp-03
              │           │           │
              └───────────┼───────────┘
                          ▼
                  New logical etcd
                     3 members
                          │
                          ▼
                     quorum = 2
                          │
                          ▼
                    leader elected
                          │
                          ▼
                    Kubernetes API
                        restored
```

---

# 33. Final Incident Result

This incident demonstrated recovery from something substantially more serious than a simple Kubernetes Pod restart or API load-balancer failover.

The environment survived:

```text
Abrupt infrastructure restart
            +
Multiple etcd member failures
            +
Loss of etcd majority
            +
Complete Kubernetes API outage
```

without rebuilding the Kubernetes cluster from scratch.

The recovery preserved the surviving Kubernetes keyspace and reconstructed the etcd consensus cluster.

Final recovery chain:

```text
Corruption detected
       │
       ▼
Root cause isolated
       │
       ▼
Survivor protected
       │
       ▼
Snapshot validated
       │
       ▼
Restore tested
       │
       ▼
Cluster membership reconstructed
       │
       ▼
Quorum restored
       │
       ▼
API server recovered
       │
       ▼
Kubernetes control plane recovered
```

---

## Incident Status

```text
Infrastructure Restart              OCCURRED
etcd cp-01 Corruption               CONFIRMED
etcd cp-03 Recovery Failure         CONFIRMED
etcd Majority Loss                  CONFIRMED
Kubernetes API Outage               CONFIRMED

Survivor Database Recovery          PASS ✓
Snapshot Validation                 PASS ✓
Non-Destructive Restore Test        PASS ✓
Three-Member Restore                PASS ✓
Raft Leader Election                PASS ✓
etcd Quorum Recovery                PASS ✓
kube-apiserver Recovery             PASS ✓
```

---

# Key Takeaway

**High availability and disaster recovery are different capabilities.**

Phase 3 and Phase 4 proved that the Kubernetes API could survive a normal single-node/VIP failure.

This incident tested a different class of failure:

```text
Normal HA Failure
─────────────────────────────────
One component fails
Redundant component takes over
No data reconstruction required


Disaster Recovery Failure
─────────────────────────────────
Majority of stateful consensus members fail
Quorum disappears
Normal failover is impossible
Persistent state must be reconstructed
```

A production-ready Kubernetes platform therefore requires both:

```text
                    Reliability
                        │
             ┌──────────┴──────────┐
             │                     │
             ▼                     ▼
      High Availability     Disaster Recovery
             │                     │
      VIP / HAProxy          etcd snapshots
      Multiple API servers   Off-cluster backup
      3-member etcd          Restore procedures
      Automatic failover     Recovery testing
```

This incident is the bridge between **Phase 4 — Kubernetes HA Bootstrap** and the next phases of the platform:

```text
Phase 4
Kubernetes HA
    │
    │
    ├── Real failure discovered
    │
    ├── etcd disaster recovery performed
    │
    └── operational lessons captured
    │
    ▼
Phase 5+
GitOps / Security / Observability / Reliability
```
