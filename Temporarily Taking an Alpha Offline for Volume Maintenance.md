# Temporarily Taking an Alpha Offline for Volume Maintenance

**Revision date**: 05/21/2026

### Use Cases for This Runbook

This procedure is needed when an underlying volume backing an Alpha pod must be taken offline for routine maintenance (filesystem check, resize, storage migration, hypervisor maintenance, etc.) **without removing the Alpha from the RAFT group and without forcing a full snapshot rebuild**.

This is distinct from [Remove and re-add a bad Alpha to a RAFT Group](Remove%20and%20re-add%20a%20bad%20Alpha%20to%20a%20RAFT%20Group.md), which intentionally discards the local `p`, `w`, `t` directories and forces the returning Alpha to re-replicate from the leader. That procedure is for **bad** data; this one preserves good data so the returning Alpha catches up cheaply via the RAFT log.

**Symptoms / triggers**:

- Scheduled storage-backend maintenance affecting a specific node or volume.
- Filesystem-level operation (fsck, resize, snapshot restore from a known-good snapshot) on a healthy Alpha's PVC.
- Planned migration of a PV to a different storage class or zone.

**Prerequisites**:

- HA cluster with **at least 3 Alpha replicas per group**. Taking one offline leaves a healthy 2/3 RAFT quorum.
- No other Alpha in the same group is currently degraded, restarting, or behind on log replication.
- No backup, bulk-load, or tablet move is currently in progress.

> ⚠️ Do **not** run this procedure on a non-HA cluster (single Alpha per group). Taking the only replica offline makes that group unavailable for the duration of the maintenance window.

## Overview of the procedure:

1. Verify cluster health and confirm the target group has a healthy 3-replica quorum.
2. Take a defensive backup.
3. Prevent Kubernetes from rescheduling the Alpha pod onto a different node (cordon the node, or apply an unsatisfiable `nodeSelector`).
4. Delete the Alpha pod so it shuts down gracefully via SIGTERM; the PVC is retained.
5. Perform the volume maintenance.
6. Allow the pod to be rescheduled (uncordon / remove the selector).
7. Verify the Alpha rejoins the group and catches up via RAFT log replication.

## Detailed Steps:

Below steps assume `alphastatefulset-x` is the Alpha whose underlying volume requires maintenance.

### 1. Set the NS variable to point to the cluster

```yaml
export NS=<namespace>
example-  export NS=cluster-0x12345
```

### 2. Verify cluster health and quorum

```yaml
kubectl exec -it zerostatefulset-0 -n $NS -- bash
curl -s localhost:6080/state | jq '{alphas: .groups."1".members, removed: .removed, zeros: .zeros}'
```

Confirm:

- All three Alpha replicas in the affected group are present and **not** in the `removed` array.
- All three have a recent `lastUpdate` timestamp (within the last few seconds).
- One is `leader: true` and two are `leader: false`. Note which one is `alphastatefulset-x` — if it is currently the leader, the cluster will elect a new leader when you delete the pod; this is fine.

If any other Alpha in the group looks unhealthy, **stop** and resolve that first.

### 3. Take a defensive backup

From the `/admin` GraphQL endpoint of any Alpha:

```graphql
mutation {
  backup(input: {destination: "<your-destination>"}) {
    response { message code }
    taskId
  }
}
```

Wait for the backup to complete before continuing.

### 4. Prevent the pod from being rescheduled

Pick **one** of the following. Cordoning is usually cleaner; the `nodeSelector` approach is useful when other workloads must keep scheduling on the node.

**Option A — cordon the node (recommended):**

```bash
NODE=$(kubectl get pod alphastatefulset-x -n $NS -o jsonpath='{.spec.nodeName}')
kubectl cordon $NODE
```

After the pod is deleted in step 5, it will stay `Pending` until you uncordon. The PVC stays bound to the pod identity, so when scheduling resumes the Alpha gets its original volume back.

**Option B — patch the pod with an unsatisfiable `nodeSelector`:**

```bash
kubectl patch pod alphastatefulset-x -n $NS -p '{"spec":{"nodeSelector":{"maintenance-hold":"true"}}}'
```

(Remember to undo this in step 7.)

### 5. Delete the Alpha pod gracefully

```bash
kubectl delete pod alphastatefulset-x -n $NS
```

> Do **not** use `--force` or `--grace-period=0`. SIGTERM allows Dgraph to flush Badger and the WAL cleanly. Ensure `terminationGracePeriodSeconds` on the StatefulSet is generous (60–120s recommended for large datasets) so Badger isn't SIGKILLed mid-flush.

Confirm the pod has actually terminated and is `Pending` (cordon path) or stuck unscheduled (nodeSelector path):

```bash
kubectl get pod alphastatefulset-x -n $NS
```

> ⚠️ Do **not** delete the PVC (`alphapvc-alphastatefulset-x`). The data must be preserved.

### 6. Perform the volume maintenance

The PV is no longer mounted by a running pod. You can now:

- Unmount it and run `fsck` from the host (use `kubectl node-shell` or Lens).
- Resize it via the CSI driver.
- Snapshot/restore at the storage layer.
- Migrate it to a different storage class (this generally requires a separate PV migration runbook — out of scope here).

Whatever you do, leave the `p`, `w`, and `t` directories intact. If they are altered or removed, the Alpha will need a full snapshot rebuild — use the [Remove and re-add a bad Alpha to a RAFT Group](Remove%20and%20re-add%20a%20bad%20Alpha%20to%20a%20RAFT%20Group.md) procedure instead.

### 7. Allow the pod to be rescheduled

**If you cordoned the node:**

```bash
kubectl uncordon $NODE
```

**If you applied a nodeSelector:**

```bash
kubectl patch pod alphastatefulset-x -n $NS --type=json -p='[{"op":"remove","path":"/spec/nodeSelector/maintenance-hold"}]'
```

(In practice the pod will likely have already been deleted and the StatefulSet will recreate it; just ensure the StatefulSet template doesn't carry the selector forward.)

### 8. Verify the Alpha rejoins and catches up

```bash
kubectl get pods -n $NS -w
```

Wait for `alphastatefulset-x` to reach `Running` and `1/1 READY`.

Repeat **step 2** and confirm:

- `alphastatefulset-x` is back in the Alpha group members with the **same Raft ID** as before (not a new incremented ID — that would indicate it was treated as a fresh replica).
- Its `lastUpdate` timestamp is recent.
- It is **not** in the `removed` array.

Also check the Alpha leader's logs — you should see RAFT log replication catching the returning replica up, **not** a full snapshot stream:

```bash
kubectl logs alphastatefulset-<ldr> -n $NS -f | grep -iE "snapshot|raft.*catching up"
```

A full "Sending Snapshot Streaming about N GiB" message indicates the on-disk state was lost or diverged enough that the leader had to send a full snapshot. The procedure still recovers correctly, but if this happens regularly the maintenance step is corrupting state and should be investigated.

## Possible problems

1. **Pod schedules back onto the maintenance node before you finished**
    1. Cause: the cordon or nodeSelector wasn't applied before the pod was deleted, or the StatefulSet had a tolerations/affinity rule that bypassed it.
    2. Apply the cordon **first**, then delete the pod. Verify the pod is `Pending` before starting maintenance.

2. **Returning Alpha gets a new (incremented) Raft ID**
    1. This means Zero treated it as a new member. Usually caused by the pod being away longer than the Zero's tolerance, or by the `w` directory being modified during maintenance.
    2. Recovery is automatic — the new replica will snapshot-stream from the leader as in [Remove and re-add a bad Alpha to a RAFT Group](Remove%20and%20re-add%20a%20bad%20Alpha%20to%20a%20RAFT%20Group.md) — but it will be slower than expected. To avoid: keep the maintenance window short and leave `w` untouched.

3. **`terminationGracePeriodSeconds` too short**
    1. Symptom: Badger logs at next startup show WAL replay taking unusually long, or errors about partial vlog/sst files.
    2. Recovery is usually automatic via WAL replay. To avoid recurrence, raise `terminationGracePeriodSeconds` on the Alpha StatefulSet to 120s or more.
