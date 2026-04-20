# Lab 03 -- Persistent Storage

## The Scenario

Picture this: a customer places a $50 lunch order -- pad thai, spring rolls, and a mango smoothie. The order hits QuickBite's PostgreSQL database. Two seconds later, the pod handling it crashes.

If the order data disappears, that customer never gets their food. The restaurant never gets paid. The driver made a trip for nothing. That is a terrible experience AND lost revenue.

Your team lead pulls you aside: "Before we go live on EKS, I need you to prove to me that data survives pod restarts. Deploy a StatefulSet with persistent volumes, write some data, kill the pod, and show me the data is still there."

This lab is that proof.

## Objective

Learn how **VolumeClaimTemplates** automatically create per-pod storage, and verify that data survives pod deletion and restarts.

## Concept

With a Deployment, if you attach a PersistentVolumeClaim, **all pods share the same volume** (or you manually create separate PVCs). This doesn't work for databases -- each PostgreSQL instance needs its own data directory, and each Kafka broker needs its own log directory.

StatefulSets solve this with **VolumeClaimTemplates**. You define a template, and Kubernetes automatically creates a **unique PVC for each pod**:

```
datastore-0 → PVC: data-datastore-0 → EBS Volume (1Gi)
datastore-1 → PVC: data-datastore-1 → EBS Volume (1Gi)
datastore-2 → PVC: data-datastore-2 → EBS Volume (1Gi)
```

The naming pattern is: `<volumeClaimTemplate-name>-<statefulset-name>-<ordinal>`

**Critical behavior:** When a pod is deleted and recreated, it reattaches to the **same PVC**. Your data survives. The customer's $50 order is safe.

## Step 1 -- Deploy the StatefulSet

```bash
kubectl apply -f lab-03-persistent-storage/statefulset.yaml
```

Wait for all pods:

```bash
kubectl get pods -w -l app=datastore
```

## Step 2 -- Inspect the Auto-Created PVCs

```bash
kubectl get pvc
```

You should see three PVCs, one per pod:

```
NAME               STATUS   VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-datastore-0   Bound    pvc-abc123...  1Gi        RWO            gp2            30s
data-datastore-1   Bound    pvc-def456...  1Gi        RWO            gp2            25s
data-datastore-2   Bound    pvc-ghi789...  1Gi        RWO            gp2            20s
```

Each PVC is backed by its own EBS volume in AWS. In production, this is where QuickBite's order records, customer data, and Kafka event logs will live. Check the PersistentVolumes:

```bash
kubectl get pv
```

## Step 3 -- Write Data to Each Pod

Simulate writing order data to each instance:

```bash
kubectl exec datastore-0 -- sh -c "echo 'I am pod-0 data' >> /data/myfile.txt"
kubectl exec datastore-1 -- sh -c "echo 'I am pod-1 data' >> /data/myfile.txt"
kubectl exec datastore-2 -- sh -c "echo 'I am pod-2 data' >> /data/myfile.txt"
```

Verify:

```bash
kubectl exec datastore-0 -- cat /data/myfile.txt
kubectl exec datastore-1 -- cat /data/myfile.txt
kubectl exec datastore-2 -- cat /data/myfile.txt
```

Each pod has **its own separate file** on its own volume -- just like each PostgreSQL instance will have its own data directory.

## Step 4 -- Delete a Pod and Verify Data Survives

Now the moment of truth. Simulate a crash:

```bash
kubectl delete pod datastore-1
```

Wait for it to come back:

```bash
kubectl get pods -w -l app=datastore
```

Now check the data:

```bash
kubectl exec datastore-1 -- cat /data/myfile.txt
```

The data is still there. The new `datastore-1` pod reattached to `data-datastore-1` PVC. The customer's order would have survived. Also check:

```bash
kubectl exec datastore-1 -- cat /data/history.txt
```

You'll see **two entries** -- one from the original pod and one from the replacement. This proves the volume survived the restart.

## Step 5 -- Understand PVC Lifecycle

**Important:** PVCs are NOT deleted when you delete the StatefulSet or scale it down. This is a safety feature -- Kubernetes doesn't want to accidentally destroy your data. Imagine if scaling down QuickBite's database during off-peak hours wiped out all the order history. That would be catastrophic.

Scale down to 1:

```bash
kubectl scale statefulset datastore --replicas=1
```

Check PVCs:

```bash
kubectl get pvc
```

All three PVCs still exist! The volumes for pod-1 and pod-2 are preserved.

Scale back up:

```bash
kubectl scale statefulset datastore --replicas=3
```

```bash
kubectl exec datastore-1 -- cat /data/myfile.txt
kubectl exec datastore-2 -- cat /data/myfile.txt
```

Data is back. The pods reconnected to their original volumes.

## Step 6 -- The EBS Connection (EKS-Specific)

On EKS, each PVC creates an actual **EBS volume** in your AWS account. You can verify this:

```bash
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.csi.volumeHandle}{"\n"}{end}'
```

Each `volumeHandle` is an EBS volume ID (like `vol-0abc123...`). These volumes exist independently of your pods. In production, QuickBite will use `gp3` volumes with provisioned IOPS for the PostgreSQL instances to ensure consistent write latency during lunch rush.

## Exercises

1. **What happens if you delete the StatefulSet entirely?** Try `kubectl delete statefulset datastore` (without deleting the service). Are the PVCs still there? Recreate the StatefulSet -- do pods get their data back?

2. **Why is `ReadWriteOnce` the right access mode?** EBS volumes can only be attached to one node at a time. Since each pod has its own volume, `ReadWriteOnce` is the correct mode. When would you need `ReadWriteMany`?

3. **Manual PVC cleanup.** After you're done, you'll need to manually delete PVCs to free the EBS volumes. Why do you think Kubernetes makes this a manual step? (Hint: think about what would happen if a `kubectl delete statefulset` command accidentally ran in production during a Friday deploy.)

## Clean Up

```bash
kubectl delete -f lab-03-persistent-storage/statefulset.yaml

# IMPORTANT: Manually delete PVCs to free EBS volumes (and stop charges!)
kubectl delete pvc data-datastore-0 data-datastore-1 data-datastore-2
```

Verify PVs are released:

```bash
kubectl get pv
```

## Next Step

You now understand all the building blocks: stable identities, DNS-based discovery, and persistent storage. Time to put them together for the real thing -- deploying QuickBite's PostgreSQL cluster on EKS. Head to [Lab 04 -- PostgreSQL on EKS](../lab-04-postgresql/README.md).
