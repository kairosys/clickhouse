# Kind PID limit

The Podman-backed Kind node has a total PID limit of 2048. Its systemd default
was 307 tasks per unit, which prevented ClickHouse from creating threads.
`90-tasks-max.conf` sets the default to 512 for units that inherit this setting.
The node's total PID limit still applies across all workloads.

Apply to the existing node:

```sh
podman cp kind/90-tasks-max.conf mac-studio-control-plane:/etc/systemd/system.conf.d/90-tasks-max.conf
podman exec mac-studio-control-plane systemctl daemon-reexec
podman exec mac-studio-control-plane systemctl show -p DefaultTasksMax
kubectl --context kind-mac-studio -n default rollout restart statefulset/clickhouse
kubectl --context kind-mac-studio -n default rollout status statefulset/clickhouse --timeout=720s
kubectl --context kind-mac-studio -n default exec clickhouse-0 -- cat /sys/fs/cgroup/pids.max
```

Expected values are `DefaultTasksMax=512` and `512`. Existing container scopes
keep their previous limits until recreated. The file survives node restarts and
Pod recreation, but must be reapplied if the Kind node is deleted and recreated.
