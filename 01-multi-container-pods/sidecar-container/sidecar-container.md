# Sidecar container deployment

This example is in [deployment.yaml](deployment.yaml). The file begins with a YAML comment describing the intent: a sidecar that can work with the main container (for example log handling).

## What the Deployment does

- **Workload**: `Deployment/nginx-deployment`, **one replica**, selector `app: nginx`.
- **Two app containers** in the same pod (both are normal `containers`, not init):
  1. **`main-container`**: `nginx`, exposes port **80**.
  2. **`sidecar-container`**: `busybox` running `tail -f /dev/null` so the process stays alive, with **`log-volume`** (`emptyDir`) mounted at **`/mnt/logs`**.

## How the sidecar pattern works here

Both containers share the **same pod**: same network namespace (localhost between containers), and any volume mounted in both appears at the same paths you configure.

In this manifest, only the sidecar mounts `log-volume`. A typical log sidecar would also mount that volume on the main container—for example at Nginx’s log directory—so the sidecar could read or ship logs. As written, the sidecar has an empty directory at `/mnt/logs` unless you extend the main container with a matching `volumeMounts` entry.

## Apply and inspect

```bash
kubectl apply -f deployment.yaml
kubectl get pods -l app=nginx
kubectl exec -it deploy/nginx-deployment -c sidecar-container -- ls -la /mnt/logs
```

Use `kubectl exec` with `-c main-container` or `-c sidecar-container` to run commands in each container.
