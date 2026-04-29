# Ephemeral containers in real life

Folder name uses “ephermal”; Kubernetes and docs use **ephemeral**.

## What they are (and what this folder is for)

**Ephemeral containers** are extra containers attached to an **existing** pod through the **EphemeralContainers** API. They are **not** the same as listing another container under `spec.containers` in your Deployment. This repo’s [deployment.yaml](deployment.yaml) is a **normal app** (one `nginx` container) that you attach debug shells to **after** the pod is running.

Official reference: [Ephemeral containers](https://kubernetes.io/docs/concepts/workloads/pods/ephemeral-containers/).

## Prerequisites

- Cluster supports ephemeral containers (Kubernetes 1.23+, feature stable in recent releases).
- Your user needs permission to create subresources on pods (for example `pods/ephemeralcontainers` patch).
- The pod must exist (running or not ready—ephemeral containers are often used when the main container is unhealthy).

## Deploy the sample app

```bash
kubectl apply -f deployment.yaml
kubectl wait --for=condition=available deployment/nginx-deployment --timeout=60s
POD=$(kubectl get pod -l app=nginx -o jsonpath='{.items[0].metadata.name}')
```

## Real-life scenarios

### 1. Minimal or distroless app—no shell in the main container

Many production images ship **without** `sh`, `bash`, or common tools. `kubectl exec` cannot give you an interactive shell. You attach a debug image that **shares the target container’s namespaces** so you see the same filesystem and processes.

```bash
kubectl debug "$POD" -it --image=busybox:1.36 --target=nginx
```

Inside the ephemeral shell you can inspect paths under the **target** container’s view (for example process list, mounted volumes, files under `/etc/nginx` as visible from that namespace).

### 2. CrashLoopBackOff or failing startup—inspect before the next restart

When the app container exits repeatedly, you may still patch ephemeral containers onto the pod (depending on cluster/version) or catch a running window. Ephemeral containers can use a **different** image with `curl`, `dig`, or editors without changing your production image.

Typical flow: `kubectl describe pod "$POD"`, then attach a debugger container to gather **one-off** evidence (config files, DNS, upstream connectivity from the **pod network**).

### 3. Network debugging from the pod’s network namespace

Traffic problems (DNS, TLS, upstream) are easiest to debug from **inside** the pod’s network stack. Attach an image that includes networking tools:

```bash
kubectl debug "$POD" -it --image=nicolaka/netshoot --target=nginx
```

Use `curl`, `dig`, `tcpdump`, etc. The traffic path matches what the `nginx` container would use when you share the target’s network namespace (default behavior with `--target` for process/filesystem/network coupling—confirm flags for your `kubectl` version).

### 4. Read-only root filesystem and locked-down main container

If the main container runs as non-root, has **read-only root**, and **dropped capabilities**, `kubectl exec` may be too limited to install tools or write temp files. An ephemeral container can be created with a **debug** profile (where allowed) so operators can investigate without rebuilding the production image.

### 5. One-off inspection without changing the Deployment

Security and change control often block “add a sidecar just for this incident.” Ephemeral containers **do not** change your Git manifest or rolling update; they disappear when the pod is deleted or recreated. That matches **production incident** workflows: debug the live pod, capture notes, then let the ReplicaSet replace the pod.

## Useful `kubectl debug` patterns

Interactive shell sharing the `nginx` container’s namespaces:

```bash
kubectl debug "$POD" -it --image=busybox:1.36 --target=nginx
```

Non-interactive one-shot (hits `nginx` on localhost from the shared network namespace):

```bash
kubectl debug "$POD" -q --image=busybox:1.36 --target=nginx -- wget -qO- http://127.0.0.1:80 | head -n 5
```

For filesystem layout, prefer the interactive attach and explore with `ls` (paths follow the target container’s mounts).

List ephemeral containers on a pod:

```bash
kubectl get pod "$POD" -o jsonpath='{.spec.ephemeralContainers[*].name}{"\n"}'
```

## Relationship to [deployment.yaml](deployment.yaml)

| In Git (`deployment.yaml`)          | At runtime (`kubectl debug`)                             |
| ----------------------------------- | -------------------------------------------------------- |
| Declares the **workload** (`nginx`) | Adds a **temporary** container to an existing pod        |
| Long-lived, versioned               | Not in your YAML; gone when the pod goes                 |
| Same image as production            | Can use **heavy** debug images (netshoot, busybox, etc.) |

Use this folder to practice **operational** debugging. Use a **second regular container** in the pod spec only when you want a **permanent** sidecar, not when you want ephemeral investigation.
