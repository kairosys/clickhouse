# ClickHouse

Analytical database serving as the backend for [Langfuse](https://langfuse.com) v3 self-hosted observability stack. This directory contains a single-node, stateful ClickHouse deployment configured to ingest and serve Langfuse traces, evaluations, and logs.

---
## Overview
ClickHouse stores all Langfuse analytical data (traces, observations, scores, prompts). It is deployed as a Kubernetes `StatefulSet` with a fixed DNS name per pod, local persistent storage, and two exposed ports consumed by the rest of the Langfuse stack:

- HTTP/REST on port **`8123`** to support its APIs here
- Native TCP protocol connections at port **`9000`**.

This is a single-replica (`replicas: 1`) deployment using an Atomic database engine.
## Architecture & Specifications
### Layout
StatefulSet named `clickhouse`, headless Service of the same name (ClusterIP None; names like `<pod>.clickhouse.furseal.svc.cluster.local` since every object below lives in that one namespace where this data applies).

### Storage configuration
No PVC/PV. The worker node hostPath `/mnt/workspaces/clickhouse/data` mounts into the container at `/var/lib/clickhouse` with `type: DirectoryOrCreate`, so existing content is preserved and recreated when absent. This local `./data/` tree mirrors engine files (`metadata/*.sql`, `preprocessed_configs/`, runtime markers like `status`/`uuid`) for inspection only, not source-of-truth schema to edit (overwritten by the running server).

### Image & runtime
`clickhouse/clickhouse-server:latest` (unpinned tag — pin before production apply); environment sourced from Secret via envFrom plus override CLICKHOUSE_DB=default. No compatibility toggles such as `CLICKHOUSE_CLUSTER_ENABLED` are declared here; single-node is compatible with Langfuse client defaults.

---
## Configuration & Environment Variables
Declared across k8s/clickhouse-secret.yaml and the Pod spec's env/envFrom block in clickhouse-statefulset.yaml:

| Variable | Source                  | Default value   / effect   | Notes |
|----------|------------------------|----------------------------|-------|
| CLICKHOUSE_USER        | Secret via pod `envFrom`    | `clickhouse`            | Credentials committed plaintext in k8s/clickhouse-secret.yaml; rotate before pushing if exposed. Password kept out of this README itself.|
| CLICKHOUSE_PASSWORD     | Secret (stringData)          | <redacted>              | Not printed here.|
| CLICKHOUSE_DB          | Pod env override              | `default`                  | Selected when a Langfuse client connects unqualified; schema appears live under metadata/.|

`.gitignore` ignores data/ and k8s/*-secret.yaml at commit, but does not broadly exclude secrets treat anything already committed as leaked.
## Deployment Guide (namespace furseal)
Apply in order so references resolve on first reconcile:

```bash
kubectl get ns       furseal || kubectl create    s      furseal          # namespace must exist before Secret/Service; all manifests below hardcode it

# Credentials + defaults via envFrom apply Secret first, else the Pod restarts later to pick up new values under environment for the container at runtime here:
kubectl -n        apply           -f k8s/clickhouse-secret.yaml      # auth credentials as a Kubernetes resource named per manifest below (check by name) so all manifests are namespaced to furseal and may be applied with that context already in place for each object via the kubectl default namespace or explicit flags:

kubectl apply    kfss/   clickhouse-statefulset.yaml          # headless Service (`---` separator creates both objects in one command, since this file is split into two parts) and wait until Running plus readiness green before relying on ports (HTTP 8123 here at least):
```