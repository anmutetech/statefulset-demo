# Lab 04 — Real-World: PostgreSQL on EKS

## Objective

Deploy a **PostgreSQL primary-replica cluster** using a StatefulSet. This lab shows you how real organizations run databases on Kubernetes — combining everything you've learned: stable identities, headless services, and persistent storage.

## Architecture

```
                    ┌──────────────────────┐
                    │   postgres-primary    │
                    │   (regular Service)   │
                    │   Writes go here      │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │     postgres-0        │
                    │     (PRIMARY)         │
                    │     Read + Write      │
                    │     EBS: 5Gi          │
                    └──────────┬───────────┘
                               │ WAL Replication
                    ┌──────────┴───────────┐
              ┌─────▼──────┐        ┌──────▼─────┐
              │ postgres-1  │        │ postgres-2  │
              │ (REPLICA)   │        │ (REPLICA)   │
              │ Read-only   │        │ Read-only   │
              │ EBS: 5Gi    │        │ EBS: 5Gi    │
              └─────────────┘        └─────────────┘

    Headless Service: postgres-headless
    DNS: postgres-0.postgres-headless.default.svc.cluster.local
          postgres-1.postgres-headless.default.svc.cluster.local
          postgres-2.postgres-headless.default.svc.cluster.local
```

### Why StatefulSets Are Essential Here

- **postgres-0** is always the primary — it needs a stable identity
- Replicas connect to `postgres-0.postgres-headless` — they need stable DNS
- Each instance has its own data directory — they need per-pod storage
- The primary must be running before replicas start — they need ordered startup

A Deployment simply cannot provide any of this.

## Step 1 — Deploy the PostgreSQL Cluster

```bash
kubectl apply -f lab-04-postgresql/secret.yaml
kubectl apply -f lab-04-postgresql/configmap.yaml
kubectl apply -f lab-04-postgresql/statefulset.yaml
```

Watch the pods come up in order:

```bash
kubectl get pods -w -l app=postgres
```

This will take a minute or two. `postgres-0` (primary) starts first, then `postgres-1` and `postgres-2` (replicas) follow.

## Step 2 — Verify the Primary

```bash
kubectl exec -it postgres-0 -- psql -U postgres -c "SELECT pg_is_in_recovery();"
```

Expected output: `f` (false) — this means it's the **primary** (not in recovery mode).

Run the init script to set up the database:

```bash
kubectl exec -it postgres-0 -- bash /scripts/init-primary.sh
```

## Step 3 — Verify the Replicas

```bash
kubectl exec -it postgres-1 -- psql -U postgres -c "SELECT pg_is_in_recovery();"
kubectl exec -it postgres-2 -- psql -U postgres -c "SELECT pg_is_in_recovery();"
```

Expected output: `t` (true) — these are **replicas** in recovery (streaming replication) mode.

Check replication status from the primary:

```bash
kubectl exec -it postgres-0 -- psql -U postgres -c "SELECT client_addr, state, sent_lsn, replay_lsn FROM pg_stat_replication;"
```

You should see two connected replicas with `streaming` state.

## Step 4 — Test Write → Replicate → Read

**Write data on the primary:**

```bash
kubectl exec -it postgres-0 -- psql -U postgres -d appdb -c "
  INSERT INTO orders (product, quantity) VALUES
    ('Widget A', 10),
    ('Widget B', 25),
    ('Widget C', 5);
"
```

**Read data from a replica:**

```bash
kubectl exec -it postgres-1 -- psql -U postgres -d appdb -c "SELECT * FROM orders;"
```

The data replicated from the primary to the replica automatically.

**Try writing to a replica (it should fail):**

```bash
kubectl exec -it postgres-1 -- psql -U postgres -d appdb -c "
  INSERT INTO orders (product, quantity) VALUES ('Should Fail', 1);
"
```

You'll get: `ERROR: cannot execute INSERT in a read-only transaction` — replicas are read-only.

## Step 5 — Test Data Durability

Delete the primary pod:

```bash
kubectl delete pod postgres-0
```

Watch it come back:

```bash
kubectl get pods -w -l app=postgres
```

Verify data survived:

```bash
kubectl exec -it postgres-0 -- psql -U postgres -d appdb -c "SELECT * FROM orders;"
```

All three orders are still there — the PVC kept the data safe.

## Step 6 — Understand the Service Architecture

```bash
kubectl get svc -l app=postgres
```

There are **two** services:

| Service | Type | Purpose |
|---------|------|---------|
| `postgres-headless` | Headless (ClusterIP: None) | Per-pod DNS for replication |
| `postgres-primary` | ClusterIP | Route app write traffic to the primary |

In a real application:
- Your app connects to `postgres-primary:5432` for writes
- For read scaling, you could create a `postgres-replicas` service targeting replicas

## Exercises

1. **Scale the replicas.** Run `kubectl scale statefulset postgres --replicas=4`. Watch `postgres-3` appear and bootstrap from the primary. Check `pg_stat_replication` — you should see 3 replicas.

2. **Simulate a failure scenario.** Delete `postgres-0` and immediately check if the replicas notice. What happens? (Note: This simple setup doesn't have automatic failover — that's what tools like Patroni handle in production.)

3. **Check the EBS volumes.** Run `kubectl get pvc` and note the volume names. These are real EBS volumes in your AWS account that will persist even if you delete the entire StatefulSet.

4. **Think about production.** What's missing from this setup that you'd need in production? (Hints: automatic failover, backups, monitoring, resource tuning, pod disruption budgets)

## Clean Up

```bash
kubectl delete -f lab-04-postgresql/statefulset.yaml
kubectl delete -f lab-04-postgresql/configmap.yaml
kubectl delete -f lab-04-postgresql/secret.yaml

# Clean up PVCs
kubectl delete pvc -l app=postgres
```

## Next Step

You've deployed a database — now let's do something completely different → [Lab 05 — Kafka on EKS](../lab-05-kafka/README.md).
