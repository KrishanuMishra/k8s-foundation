# Istio and the service mesh (topic 02)

**Service mesh** tools help you manage **east–west (E–W)** traffic: requests between workloads *inside* the cluster (service to service). That is different from **north–south (N–S)** traffic, which is traffic from **outside** the cluster (for example users hitting an Ingress).

## What / why / how

| Question | Answer |
|----------|--------|
| **What** | A control plane plus data-plane proxies that sit in the traffic path between services. |
| **Why** | Uniform policy, security, and observability without every app reimplementing the same concerns. |
| **How** | A **sidecar** proxy container is injected into each workload pod; traffic is steered through those proxies. |

## Traffic at a glance

```text
                    North–South (user → cluster)
                              │
                              ▼
                         [ Ingress ]
                              │
    East–West (inside cluster)│
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
    [ Login ]  ──────►  [ Catalog ]  ──────►  [ Payment ]  ──►  [ Notification ]
         │                    │                    │
         └────────────────────┴────────────────────┘
              All service-to-service hops can be
              mediated by mesh proxies (sidecars).
```

## Why use a service mesh?

Kubernetes alone can route traffic with Services and Ingress. A mesh **adds a dedicated layer** for policies and telemetry that apply consistently across many languages and teams.

Common mesh capabilities (Istio is one implementation):

| Feature | What it gives you |
|---------|-------------------|
| **mTLS** | Encrypted service-to-service traffic with **mutual** authentication (each side proves identity with certificates). |
| **Advanced rollout** | Patterns such as **canary**, **A/B**, and **blue/green** at the traffic layer. |
| **Observability** | For example **Kiali** for topology and traffic health, plus metrics/traces when wired to your stack. |

### TLS vs mTLS (short)

- **TLS**: Usually the **client** checks the **server’s** identity (typical HTTPS).
- **mTLS**: **Both** client and server present certificates and verify each other—common for **service-to-service** identity inside a cluster.

## How Istio fits pods

In a meshed namespace, workloads often get an extra container: the **sidecar proxy**. Application containers still talk to `localhost` or cluster DNS as before, but the mesh **intercepts** and **enforces** policy on that traffic.

```text
  Pod                          Pod
┌──────────────────┐        ┌──────────────────┐
│ app container    │        │ app container    │
│ sidecar (proxy)  │◄──────►│ sidecar (proxy)  │
└──────────────────┘        └──────────────────┘
```

## Admission controllers (concept Istio also relies on)

Kubernetes **admission controllers** can **validate** or **mutate** objects before they are persisted. Two ideas tied to learning meshes and platform behavior:

1. **Sidecar injection** — Istio commonly uses a **mutating** admission webhook so new Pods in selected namespaces get the proxy sidecar without hand-editing every manifest.
2. **General platform examples** — Below are small manifests you can apply to see **mutation** (default StorageClass on a PVC) and **validation** (Pod blocked by ResourceQuota).

### Example A: PVC with no `storageClassName` (DefaultStorageClass)

If your cluster has a **default** StorageClass, the **DefaultStorageClass** admission controller can set it on PVCs that omit `storageClassName`.

- Manifest: [pvc-no-storageclass.yaml](pvc-no-storageclass.yaml)

```bash
kubectl apply -f pvc-no-storageclass.yaml
kubectl get pvc demo-pvc-no-sc -o wide
kubectl get pvc demo-pvc-no-sc -o jsonpath='{.spec.storageClassName}{"\n"}'
```

You should see a storage class name populated after creation (when defaults are configured).

### Example B: ResourceQuota and a Pod that exceeds it

Apply a **ResourceQuota**, then try a Pod whose **requests** exceed the quota. The API server should **reject** creating the Pod.

```bash
kubectl apply -f resourcequota.yaml
kubectl describe resourcequota demo-quota
kubectl apply -f pod-exceeds-quota.yaml
# Expect an error such as exceeding cpu quota
kubectl delete -f pod-exceeds-quota.yaml --ignore-not-found
kubectl delete -f resourcequota.yaml --ignore-not-found
kubectl delete -f pvc-no-storageclass.yaml --ignore-not-found
```

- Quota manifest: [resourcequota.yaml](resourcequota.yaml)  
- Failing Pod manifest: [pod-exceeds-quota.yaml](pod-exceeds-quota.yaml)

---

*Note: The folder name `02-isitio` is kept as in the repo layout; the product name is **Istio**.*
