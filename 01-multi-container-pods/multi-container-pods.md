# Multi-container pods

## What are multi-container pods?

When a single pod runs more than one container so the workload can share network, storage, and scheduling for its whole lifecycle, that is a **multi-container pod**.

## Patterns in this repo

Each pattern has its own folder with a `deployment.yaml` and a short guide:

| Pattern | Folder | Role |
|--------|--------|------|
| **Init containers** | [init-container](init-container/) | One-off setup before app containers start |
| **Sidecar** | [sidecar-container](sidecar-container/) | Helper container running alongside the main app |
| **Ephemeral debugging** | [ephermal-container](ephermal-container/) | Sample single-container app; attach short-lived debug shells with `kubectl debug` (see that doc) |

## Init containers

Init containers run **to completion in order** before any normal `containers` start. They are ideal for migrations, waiting on dependencies, or preparing files on a shared volume. Details: [init-container/init-container.md](init-container/init-container.md).

## Sidecar containers

Sidecars are normal containers in the same pod as the app, usually sharing volumes or localhost for logging, proxying, or sync. Details: [sidecar-container/sidecar-container.md](sidecar-container/sidecar-container.md).

## Ephemeral containers

Kubernetes **ephemeral containers** are attached at **runtime** for troubleshooting (for example `kubectl debug`). The sample **Deployment** in that folder is a normal one-container app you debug with ephemeral containers; it does not list debug containers in the manifest. Details: [ephermal-container/ephermal-container.md](ephermal-container/ephermal-container.md).
