# Phase 05 — GitOps, Argo CD, Kafka, Producer and Consumer

## Goal
Build the event-driven application platform through GitOps.

Components:
- Argo CD
- 3-broker Kafka
- producer
- consumer
- automated sync/prune/self-heal

## GitOps Layout
```text
gitops/
├── argocd/
│   ├── applications/
│   └── projects/
├── kafka/
└── apps/
    ├── producer/
    └── consumer/
```

## Argo CD
Project:
```text
abrnoc-platform
```

Repository:
```text
https://github.com/sadeqrm/AbrNOC-TASK.git
```

Applications use:
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

### kubeconfig Issue
Initial `kubectl apply` from the Ansible controller returned:
```text
no matches for kind "AppProject"
no matches for kind "Application"
```

The YAML was valid; `kubectl` was using the wrong cluster/context. Copying the correct kubeconfig fixed the issue.

## Kafka
Kafka image:
```text
apache/kafka:3.9.0
```

### Image Failure
The first image was:
```text
bitnami/kafka:3.9.0
```

It failed with:
```text
ErrImagePull
ImagePullBackOff
```

The deployment was migrated to the official Apache image.

## KRaft
Three combined broker/controller nodes:
```text
0@kafka-0.kafka-headless:9093
1@kafka-1.kafka-headless:9093
2@kafka-2.kafka-headless:9093
```

Safety settings:
```text
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
default.replication.factor=3
min.insync.replicas=2
```

## Advertised Listener Problems

### Problem 1
Using:
```text
PLAINTEXT://$(POD_NAME).kafka-headless:9092
```
inside an env value produced a literal string and Kafka failed to parse it.

Fix: construct the listener in a shell wrapper using `$HOSTNAME`.

### Problem 2
Short names such as:
```text
kafka-0.kafka-headless:9092
```
worked inside the `kafka` namespace but not from `apps`.

Fix: advertise FQDNs:
```text
kafka-0.kafka-headless.kafka.svc.cluster.local:9092
kafka-1.kafka-headless.kafka.svc.cluster.local:9092
kafka-2.kafka-headless.kafka.svc.cluster.local:9092
```

## Kafka Validation
Quorum showed:
```text
LeaderId:       2
MaxFollowerLag: 0
CurrentVoters:  0,1,2
```

Topic:
```text
abrnoc-events
```

Configuration:
```text
Partitions:         3
Replication Factor: 3
min.insync.replicas: 2
```

All partitions had all replicas in ISR.

## End-to-End Test
Manual test:
```text
produce: abrnoc-test-event-1
consume: abrnoc-test-event-1
```

## GitOps Producer
Producer continuously emits:
```text
event-<counter>-<timestamp>
```

## GitOps Consumer
Consumer group:
```text
abrnoc-consumer-group
```

Producer events were successfully consumed.

## Final Argo State
```text
kafka      Synced Healthy
producer   Synced Healthy
consumer   Synced Healthy
```

## Follow-up for Production
Review:
- durable Kafka PVCs
- disruption budgets
- topology spread
- resource sizing
- Kafka auth/ACLs
- backup/restore
- upgrade policy

## Result
A Git-driven producer -> Kafka -> consumer pipeline was fully functional.
