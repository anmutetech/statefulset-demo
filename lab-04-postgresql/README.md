# Lab 04 -- QuickBite's Order Database: PostgreSQL on EKS

## The Scenario

It is Wednesday morning and your team lead messages you:

> "Great work on the fundamentals. Time for the real thing. We need QuickBite's order database running on EKS before the end of the sprint. The setup is a PostgreSQL primary-replica cluster -- the primary handles all writes from the order service, and the replicas serve read traffic to the mobile app (restaurant listings, order history, etc). Deploy it, seed it with test data, and prove that we don't lose orders if a pod crashes."

This is what you have been building toward. Every concept from the previous labs -- stable identities, headless services, persistent storage -- comes together here.

## New Kubernetes Concepts in This Lab

Before we dive in, this lab introduces a few Kubernetes concepts you haven't seen yet. Here's a quick overview -- you'll see each one in action shortly.

| Concept | What It Is | Why We Need It |
|---------|-----------|----------------|
| **ConfigMap** | A Kubernetes object that stores configuration files and scripts as key-value pairs. Pods can mount them as files. | We store the PostgreSQL setup scripts here so every pod can access them. |
| **Secret** | Like a ConfigMap, but for sensitive data (passwords, API keys). Values are base64-encoded. | We store the database password here instead of hardcoding it in the YAML. |
| **initContainer** | A container that runs **once** before the main container starts. It must complete successfully before the main container launches. | Replicas use an initContainer to copy data from the primary (via `pg_basebackup`) before starting PostgreSQL in replica mode. |
| **Readiness Probe** | A health check that tells Kubernetes "this pod is ready to accept traffic." | Kubernetes won't send traffic to a PostgreSQL pod until `pg_isready` confirms it's accepting connections. |
| **Liveness Probe** | A health check that tells Kubernetes "this pod is still alive." If it fails, the pod gets restarted. | If PostgreSQL crashes or hangs, the liveness probe detects it and restarts the pod automatically. |

## Architecture

```
     QuickBite Order Service              QuickBite Mobile App
     (writes new orders)                  (reads menus, order history)
              │                                     │
              ▼                                     ▼
    ┌──────────────────┐                  Reads go to any replica
    │  postgres-primary │                  via postgres-headless
    │  (regular Service)│
    └────────┬─────────┘
             │
    ┌────────▼─────────┐
    │    postgres-0     │
    │    (PRIMARY)      │
    │    Read + Write   │
    │    EBS: 5Gi       │
    ���────────┬─────────┘
             │ WAL Replication
    ┌────────┴─────────┐
┌───▼──────┐     ┌─────▼────┐
│postgres-1 │     │postgres-2 │
│(REPLICA)  │     │(REPLICA)  │
│Read-only  │     ��Read-only  │
│EBS: 5Gi   │     │EBS: 5Gi   │
��───────────┘     └───────────┘

  Headless Service: postgres-headless
  DNS: postgres-0.postgres-headless.default.svc.cluster.local
       postgres-1.postgres-headless.default.svc.cluster.local
       postgres-2.postgres-headless.default.svc.cluster.local
```

### How Replication Works (Simplified)

PostgreSQL uses **streaming replication** to keep replicas in sync:

1. The **primary** (postgres-0) processes all writes and records every change in a **Write-Ahead Log (WAL)**
2. Each **replica** connects to the primary and continuously streams these WAL records
3. The replica applies the WAL records to its own copy of the data, staying in sync

For this to work, replicas need to:
- Know the primary's address -- **headless service** (Lab 02)
- Have their own copy of the data -- **per-pod storage** (Lab 03)
- Copy the primary's initial data before starting -- **initContainer** (new in this lab)

### Why StatefulSets Are Essential Here

- **postgres-0** is always the primary -- it needs a stable identity so replicas know where to stream WAL from
- Replicas connect to `postgres-0.postgres-headless` -- they need stable DNS that doesn't change on restart
- Each instance has its own data directory -- they need per-pod storage so a crashed replica doesn't lose its data
- The primary must be running before replicas start -- they need ordered startup because replicas bootstrap from the primary

A Deployment simply cannot provide any of this.

## Step 1 -- Deploy the PostgreSQL Cluster

The deployment requires three files. Take a moment to understand what each one does:

- **secret.yaml** -- stores the database password (so it's not hardcoded in the manifests)
- **configmap.yaml** -- stores the setup scripts and initialization SQL
- **statefulset.yaml** -- defines the PostgreSQL StatefulSet, services, and storage

```bash
kubectl apply -f lab-04-postgresql/secret.yaml
kubectl apply -f lab-04-postgresql/configmap.yaml
kubectl apply -f lab-04-postgresql/statefulset.yaml
```

Watch the pods come up in order:

```bash
kubectl get pods -w -l app=postgres
```

Here's what's happening behind the scenes:

1. **postgres-0** starts. Its initContainer runs `setup.sh`, which detects it's the primary and does nothing special. PostgreSQL initializes normally, runs `init.sql` (creates the replicator user, QuickBite database, and seed data), and begins accepting connections.
2. **postgres-1** starts after postgres-0 is ready. Its initContainer runs `setup.sh`, waits for the primary to be available, then runs `pg_basebackup` to copy all of postgres-0's data. Once done, the main PostgreSQL container starts in replica mode.
3. **postgres-2** starts after postgres-1 is ready. Same process as postgres-1.

> **This takes 2-3 minutes.** The replicas need to copy the entire database from the primary. You'll see pods go through `Init:0/1` (initContainer running) before reaching `Running`. This is normal.

Expected sequence:

```
NAME         READY   STATUS     AGE
postgres-0   0/1     Init:0/1   5s
postgres-0   1/1     Running    20s
postgres-1   0/1     Init:0/1   25s
postgres-1   1/1     Running    60s
postgres-2   0/1     Init:0/1   65s
postgres-2   1/1     Running    100s
```

Press `Ctrl+C` when all three pods show `1/1 Running`.

## Step 2 -- Verify the Primary

Check that postgres-0 is the primary:

```bash
kubectl exec -it postgres-0 -- psql -U postgres -c "SELECT pg_is_in_recovery();"
```

Expected output:

```
 pg_is_in_recovery
-------------------
 f
(1 row)
```

`f` (false) means this pod is **not** in recovery mode -- it's the primary. It accepts both reads and writes.

Verify the QuickBite database was created with seed data:

```bash
kubectl exec -it postgres-0 -- psql -U postgres -d quickbite -c "SELECT name, cuisine, rating FROM restaurants;"
```

Expected output:

```
     name      | cuisine  | rating
---------------+----------+--------
 Thai Palace   | Thai     |    4.5
 Pizza Planet  | Italian  |    4.2
 Sushi Express | Japanese |    4.8
 Burger Barn   | American |    4.0
(4 rows)
```

## Step 3 -- Verify the Replicas

```bash
kubectl exec -it postgres-1 -- psql -U postgres -c "SELECT pg_is_in_recovery();"
kubectl exec -it postgres-2 -- psql -U postgres -c "SELECT pg_is_in_recovery();"
```

Both should return `t` (true) -- these are **replicas** in recovery mode, streaming changes from the primary.

Check replication status from the primary:

```bash
kubectl exec -it postgres-0 -- psql -U postgres -c "SELECT client_addr, state, sent_lsn, replay_lsn FROM pg_stat_replication;"
```

Expected output (IPs will differ):

```
 client_addr |   state   |  sent_lsn  | replay_lsn
-------------+-----------+------------+------------
 10.0.1.45   | streaming | 0/3000148  | 0/3000148
 10.0.2.67   | streaming | 0/3000148  | 0/3000148
(2 rows)
```

Two rows, both with `streaming` state -- both replicas are connected and synced.

## Step 4 -- Simulate the Order Flow

**A customer places a lunch order on the primary:**

```bash
kubectl exec -it postgres-0 -- psql -U postgres -d quickbite -c "
  INSERT INTO orders (customer_id, restaurant_id, items, total_amount, status) VALUES
    (1, 1, 'Pad Thai, Spring Rolls, Mango Smoothie', 32.50, 'placed'),
    (2, 3, 'Salmon Sashimi, Edamame', 28.00, 'placed'),
    (3, 2, 'Margherita Pizza, Garlic Bread', 22.75, 'preparing');
"
```

**The mobile app reads order history from a replica:**

```bash
kubectl exec -it postgres-1 -- psql -U postgres -d quickbite -c "
  SELECT o.id, c.name AS customer, r.name AS restaurant, o.items, o.total_amount, o.status
  FROM orders o
  JOIN customers c ON o.customer_id = c.id
  JOIN restaurants r ON o.restaurant_id = r.id;
"
```

Expected output:

```
 id |    customer    |  restaurant   |                  items                  | total_amount | status
----+----------------+---------------+-----------------------------------------+--------------+-----------
  1 | Alice Chen     | Thai Palace   | Pad Thai, Spring Rolls, Mango Smoothie  |        32.50 | placed
  2 | Bob Martinez   | Sushi Express | Salmon Sashimi, Edamame                 |        28.00 | placed
  3 | Carol Johnson  | Pizza Planet  | Margherita Pizza, Garlic Bread          |        22.75 | preparing
(3 rows)
```

The data replicated from the primary to the replica automatically. The mobile app is reading from `postgres-1` while the order service writes to `postgres-0` -- read traffic and write traffic are separated.

**Try writing to a replica (it should fail):**

```bash
kubectl exec -it postgres-1 -- psql -U postgres -d quickbite -c "
  INSERT INTO orders (customer_id, restaurant_id, items, total_amount) VALUES (1, 2, 'Test', 10.00);
"
```

You'll get: `ERROR: cannot execute INSERT in a read-only transaction` -- replicas are read-only. This protects data integrity. All writes must go through the primary.

## Step 5 -- The Crash Test: Prove Orders Survive

This is the moment your team lead is waiting for. Simulate the primary crashing:

```bash
kubectl delete pod postgres-0
```

Watch it come back:

```bash
kubectl get pods -w -l app=postgres
```

Verify the orders survived:

```bash
kubectl exec -it postgres-0 -- psql -U postgres -d quickbite -c "
  SELECT o.id, c.name AS customer, o.items, o.total_amount, o.status
  FROM orders o
  JOIN customers c ON o.customer_id = c.id;
"
```

All three orders are still there. Alice's Pad Thai, Bob's Sashimi, Carol's Pizza -- none of it was lost. The PVC kept the data safe across the pod restart. This is what you will show your team lead.

## Step 6 -- Understand the Service Architecture

```bash
kubectl get svc -l app=postgres
```

There are **two** services, each serving a different purpose:

| Service | Type | Who Uses It |
|---------|------|-------------|
| `postgres-headless` | Headless (ClusterIP: None) | Replicas use per-pod DNS for replication setup |
| `postgres-primary` | ClusterIP | The order service connects here for database access |

In the full QuickBite architecture:
- The **order service** connects to `postgres-primary:5432` for writes (placing orders, updating status)
- The **mobile app API** queries replicas through their per-pod DNS for reads (browsing restaurants, viewing order history)
- This separation means lunch rush read traffic from thousands of app users doesn't slow down order processing

## Exercises

1. **Scale for lunch rush.** Run `kubectl scale statefulset postgres --replicas=4`. Watch `postgres-3` appear and bootstrap from the primary. Check `pg_stat_replication` -- you should see 3 replicas now. During peak hours, QuickBite could add read replicas to handle the traffic spike, then scale back down at night.

2. **Simulate a failure scenario.** Delete `postgres-0` and immediately check if the replicas notice. What happens? (Note: This simple setup doesn't have automatic failover -- in production, QuickBite would use a tool like Patroni to automatically promote a replica to primary if `postgres-0` goes down.)

3. **Check the EBS volumes.** Run `kubectl get pvc` and note the volume names. These are real EBS volumes in your AWS account that will persist even if you delete the entire StatefulSet. In production, these volumes would also be backed up with EBS snapshots on a schedule.

4. **Think about production.** What is missing from this setup that QuickBite would need before going live? (Hints: automatic failover with Patroni, scheduled backups, connection pooling with PgBouncer, monitoring with pg_stat_statements, Pod Disruption Budgets to prevent simultaneous pod evictions during node upgrades)

## Clean Up

```bash
kubectl delete -f lab-04-postgresql/statefulset.yaml
kubectl delete -f lab-04-postgresql/configmap.yaml
kubectl delete -f lab-04-postgresql/secret.yaml

# Clean up PVCs to release EBS volumes (and stop AWS charges!)
kubectl delete pvc -l app=postgres
```

## Next Step

The order database is running. But QuickBite also needs **real-time event streaming** -- when an order status changes from "placed" to "preparing" to "out for delivery" to "delivered", the notification service, driver dispatch, and analytics pipeline all need to know immediately. That is Kafka's job. Head to [Lab 05 -- Kafka on EKS](../lab-05-kafka/README.md).
