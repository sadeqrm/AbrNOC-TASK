# Phase 06 — Istio Service Mesh and STRICT mTLS

## Goal
Protect producer/consumer/Kafka communication with Istio-managed mTLS.

## Installation
Istio `1.30.3` was installed with `istioctl`.

Core components:
- `istiod`
- `istio-ingressgateway`

## Calico Compatibility
Istio detected:
```text
bpfConnectTimeLoadBalancing=TCP
```

and required:
```text
bpfConnectTimeLoadBalancing=Disabled
```

Felix configuration was patched accordingly.

## Apps Mesh Enrollment
Namespace:
```text
apps
```

was labeled:
```text
istio-injection=enabled
```

Producer/consumer were recreated and became `2/2 Running`.

## Native Sidecar Detail
In this Kubernetes/Istio combination, `istio-proxy` appears as a native sidecar under `initContainers`.

This initially caused confusion because:
```bash
.spec.containers[*].name
```
showed only the app container.

Full pod inspection showed:
- `istio-init`
- `istio-proxy`
- `security.istio.io/tlsMode=istio`
- Envoy Running/Ready
- interception mode `REDIRECT`

## STRICT mTLS for Apps
```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: apps
spec:
  mtls:
    mode: STRICT
```

Producer/consumer stayed connected to Istiod and event flow remained healthy.

## Kafka Mesh Enrollment
Kafka was added to the mesh after application injection was validated.

All three brokers received Istio sidecars.

## Temporary Kafka Convergence Errors
During Kafka rollout:
```text
NOT_LEADER_OR_FOLLOWER
NOT_ENOUGH_REPLICAS
```

appeared temporarily while broker leadership and ISR converged.

Event flow recovered after the rollout settled.

## STRICT mTLS for Kafka
A namespace-level `PeerAuthentication` with `STRICT` was applied to `kafka`.

`istioctl proxy-status` showed:
- producer
- consumer
- kafka-0
- kafka-1
- kafka-2

all connected to Istiod.

## Functional Proof
Producer kept generating events and consumer kept receiving the same events after STRICT mTLS was enabled.

## Envoy Proof
Producer outbound cluster:
```text
outbound|9092||kafka.kafka.svc.cluster.local
```

showed:
```text
tlsMode-istio
envoy.transport_sockets.tls
UpstreamTlsContext
TLSv1_2 ... TLSv1_3
SDS certificates
ROOTCA
```

`istioctl x describe pod` also reported:
```text
Workload mTLS mode: STRICT
```

## Troubleshooting Method
The Kafka/Istio path was tested layer by layer:
1. DNS
2. TCP reachability
3. Kafka protocol/API
4. advertised listeners
5. Envoy config

## Result
Producer, consumer and all Kafka brokers are inside the Istio mesh with strict mTLS enforced and proven at the Envoy configuration layer.
