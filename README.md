# Exited Container Pruning for OpenShift

Prune exited containers on OpenShift worker nodes to prevent nodes from entering `NotReady` state due to gRPC `ResourceExhausted` errors.

## Problem

When workloads (Tekton Pipelines, CronJobs, CI/CD) generate exited containers faster than the kubelet garbage collector can remove them, the CRI-O container list response exceeds the 16 MiB gRPC message limit. This causes:

- `ContainerGCFailed` events with `rpc error: code = ResourceExhausted desc = grpc: received message larger than max`
- Nodes transitioning to `NotReady` state
- Cascading failure where GC can no longer list or remove containers

See [KCS 7045904](https://access.redhat.com/solutions/7045904) and [KCS 7018759](https://access.redhat.com/solutions/7018759) for details.

## Solution

A CronJob that runs every 12 hours, iterates over all worker nodes via `oc debug`, and uses `crictl` to remove the oldest exited containers when the count exceeds a configurable threshold (default: **2000**).

### How it works

1. Lists all worker nodes (`node-role.kubernetes.io/worker`)
2. For each node, spawns an `oc debug` pod in the `cleanup` namespace
3. Uses `chroot /host` to access the node's CRI-O runtime
4. Counts exited containers using `crictl ps -q --state exited`
5. If count exceeds the threshold, removes the oldest excess containers via `crictl rm`

## Deployment

```bash
# 1. Create the cleanup namespace
oc create namespace cleanup

# 2. Deploy RBAC resources (ServiceAccount, ClusterRole, ClusterRoleBinding)
oc apply -f manifest.yaml

# 3. Grant the privileged SCC to the service account
oc adm policy add-scc-to-user privileged -z openshift-cleanup -n cleanup

# 4. Deploy the CronJob
oc apply -f cronjob.yaml

# 5. Verify
oc get cronjob -n cleanup
```

## Configuration

| Parameter | Default | How to change |
|-----------|---------|---------------|
| Container threshold | 2000 | Set `CONTAINER_THRESHOLD` env var in `cronjob.yaml` |
| Schedule | Every 12 hours (`0 */12 * * *`) | Edit `spec.schedule` in `cronjob.yaml` |
| Active deadline | 3600s (1 hour) | Edit `activeDeadlineSeconds` in `cronjob.yaml` |
| Target nodes | Workers only | Change label selector in the `oc get no -l` command |

## Immediate Recovery

If a node is already `NotReady`, manually prune exited containers:

```bash
oc debug node/<node-name> -- chroot /host crictl ps -q --state exited | xargs -r crictl rm
```

## Related

- [KCS 7045904 - Node reporting NotReady state related to ResourceExhausted](https://access.redhat.com/solutions/7045904)
- [KCS 7018759 - Completed pods are not cleaned up in OpenShift 4](https://access.redhat.com/solutions/7018759)
- [OCPBUGS-17141](https://issues.redhat.com/browse/OCPBUGS-17141) - RFE for automatic exited container cleanup (not yet implemented)
- [OCPBUGS-18256](https://issues.redhat.com/browse/OCPBUGS-18256) - Exited containers accumulation
- [kubernetes#131407](https://github.com/kubernetes/kubernetes/issues/131407) - kubelet not cleaning up exited containers
- [kubernetes#134750](https://github.com/kubernetes/kubernetes/issues/134750) - Inability for kubelet to clean large number of containers

## License

Apache License 2.0
