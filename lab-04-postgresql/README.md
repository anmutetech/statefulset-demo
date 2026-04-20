# Lab 04 -- QuickBite's Order Database: PostgreSQL on EKS

## The Scenario

It is Wednesday morning and your team lead messages you:

> "Great work on the fundamentals. Time for the real thing. We need QuickBite's order database running on EKS before the end of the sprint. The setup is a PostgreSQL primary-replica cluster -- the primary handles all writes from the order service, and the replicas serve read traffic to the mobile app (restaurant listings, order history, etc). Deploy it, seed it with test data, and prove that we don't lose orders if a pod crashes."

This is what you have been building toward. Every concept from the previous labs -- stable identities, headless services, persistent storage -- comes together here.

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
    └────────┬─────────┘
             │ WAL Replication
    ┌────────┴─────────┐
┌───▼──────┐     ┌─────▼────┐
│postgres-1 │     │postgres-2 │
│(REPLICA)  │     │(REPLICA)  │
│Read-only  │     │Read-only  │
│EBS: 5Gi   │     │EBS: 5Gi   │
└───────────┘     └───────────┘

  Headless Service: postgres-headless
  DNS: postgres-0.postgres-headless.default.svc.cluster.local
       postgres-1.postgres-headless.default.svc.cluster.local
       postgres-2.postgres-headless.default.svc.cluster.local
```

### Why StatefulSets Are Essential Here

- **postgres-0** is always the primary -- it needs a stable identity so replicas know where to stream WAL from
- Replicas connect to `postgres-0.postgres-headless` -- they need stable DNS that doesn't change on restart
- Each instance has its own data directory -- they need per-pod storage so a crashed replica doesn't lose its data
- The primary must be running before replicas start -- they need ordered startup because replicas bootstrap from the primary

A Deployment simply cannot provide any of this.

## Step 1 -- Deploy the PostgreSQL Cluster

```bash
kubectl apply -f lab-04-postgresql/secret.yaml
kubectl apply -f lab-04-postgresql/configmap.yaml
kubectl apply -f lab-04-postgresql/statefulset.yaml
```

Watch the pods come up in order:

```bash
kubectl get pods -w -l app=postgres
```

This will take a minute or two. `postgres-0` (primary) starts first, then `postgres-1` and `postgres-2` (replicas) follow -- just like the ordered creation you saw in Lab 01, but now it actually matters.

## Step 2 -- Verify the Primary

```bash
kubectl exec -it postgres-0 -- psql -U postgres -c "SELECT pg_is_in_recovery();"
```

Expected output: `f` (false) -- this means it is the **primary** (not in recovery mode).

Run the init script to create the QuickBite database and seed it with restaurant and customer data:

```bash
kubectl exec -it postgres-0 -- bash /scripts/init-primary.sh
```

## Step 3 -- Verify the Replicas

```bash
kubectl exec -it postgres-1 -- psql -U postgres -c "SELECT pg_is_in_recovery();"
kubectl exec -it postgres-2 -- psql -U postgres -c "SELECT pg_is_in_recovery();"
```

Expected output: `t` (true) -- these are **replicas** in recovery (streaming replication) mode.

Check replication status from the primary:

```bash
kubectl exec -it postgres-0 -- psql -U postgres -c "SELECT client_addr, state, sent_lsn, replay_lsn FROM pg_stat_replication;"
```

You should see two connected replicas with `streaming` state. This is the QuickBite database replicating in real time.

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

All three orders are still there. Alice's Pad Thai, Bob's Sashimi, Carol's Pizza -- none of it was lost. The PVC kept the data safe across the pod restart. This is what you'll show your team lead.

## Step 6 -- Understand the Service Architecture

```bash
kubectl get svc -l app=postgres
```

There are **two** services, each serving a different purpose:

| Service | Type | Who Uses It |
|---------|------|-------------|
| `postgres-headless` | Headless (ClusterIP: None) | Replicas use per-pod DNS for replication setup |
| `postgres-primary` | ClusterIP | The order service connects here for writes |

In the full QuickBite architecture:
- The **order service** connects to `postgres-primary:5432` for writes (placing orders, updating status)
- The **mobile app API** connects to replicas through `postgres-headless` for reads (browsing restaurants, viewing order history)
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

# Clean up PVCs
kubectl delete pvc -l app=postgres
```

## Next Step

The order database is running. But QuickBite also needs **real-time event streaming** -- when an order status changes from "placed" to "preparing" to "out for delivery" to "delivered", the notification service, driver dispatch, and analytics pipeline all need to know immediately. That is Kafka's job. Head to [Lab 05 -- Kafka on EKS](../lab-05-kafka/README.md).
