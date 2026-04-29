# Init container deployment

This example is in [deployment.yaml](deployment.yaml).

## What the Deployment does

- **Workload**: `Deployment/nginx-deployment` with **one replica** and label `app: nginx`.
- **Pod template** defines:
  1. An **init container** named `init-container` (`busybox`) that runs a shell once and writes `Hello, World!` into `/mnt/data/hello.txt`.
  2. A **volume** `data-volume` of type `emptyDir` shared by mounting it at `/mnt/data` on the init container.
  3. The **main container** `nginx` that starts **only after** the init container exits successfully (exit code 0). It mounts the same `data-volume` (you would add a `volumeMounts` entry on `nginx` if you want it to read that file; the current YAML only mounts the volume on the init container).

## Lifecycle

1. Kubernetes schedules the pod.
2. **Init phase**: runs `init-container` → creates the file on the shared volume → exits.
3. **App phase**: starts the `nginx` container with the configured CPU/memory requests and limits.

If the init container fails or never completes, the main container does not start.

## Apply and inspect

```bash
kubectl apply -f deployment.yaml
kubectl get pods -l app=nginx
kubectl describe pod -l app=nginx   # init container status and events
```

To have `nginx` actually serve or see the file from the init step, add a matching `volumeMounts` on the `nginx` container pointing at `data-volume`.
