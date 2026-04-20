# Lab 05 — Real-World: Kafka on EKS

## Objective

Deploy a **3-broker Kafka cluster with ZooKeeper** on EKS. This lab demonstrates why StatefulSets are essential for distributed messaging systems — each broker has a unique ID, its own storage, and needs to advertise its own DNS address.

## Architecture

```
  Producers ──────────────────────────────────────── Consumers
       │                                                 │
       │              kafka-headless (DNS)                │
       │    ┌─────────────┬─────────────┬────────────┐   │
       │    │             │             │             │   │
       ▼    ▼             ▼             ▼             ▼   ▼
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │ kafka-0   │     │ kafka-1   │     │ kafka-2   │
  │ broker 0  │     │ broker 1  │     │ broker 2  │
  │ EBS: 10Gi │     │ EBS: 10Gi │     │ EBS: 10Gi │
  └─────┬─────┘     └─────┬─────┘     └─────┬─────┘
        │                  │                  │
        └────────┬─────────┴─────────┬────────┘
                 │   ZooKeeper       │
          ┌──────▼──┐  ┌──────┐  ┌──▼──────┐
          │  zk-0    │  │ zk-1  │  │  zk-2   │
          │  EBS:2Gi │  │ EBS:2Gi│  │  EBS:2Gi │
          └─────────┘  └───────┘  └─────────┘
```

### Why Kafka Needs StatefulSets

1. **Broker ID** — Each broker has a unique integer ID (0, 1, 2). We derive this from the pod ordinal (`kafka-0` → broker 0).
2. **Advertised Listeners** — Each broker tells clients its own address: `kafka-0.kafka-headless:9092`. Without stable DNS, clients couldn't reconnect after a restart.
3. **Log Storage** — Kafka stores messages on disk. Each broker needs its own persistent volume so messages survive restarts.
4. **Ordered Startup** — Brokers need ZooKeeper to be running first. ZooKeeper itself needs ordered startup for leader election.

## Step 1 — Deploy ZooKeeper First

Kafka depends on ZooKeeper, so deploy it first:

```bash
kubectl apply -f lab-05-kafka/zookeeper.yaml
```

Wait for all 3 ZooKeeper pods:

```bash
kubectl get pods -w -l app=zookeeper
```

Verify the ensemble is healthy:

```bash
kubectl exec zk-0 -- bash -c "echo ruok | nc localhost 2181"
```

Expected: `imok`

## Step 2 — Deploy Kafka

```bash
kubectl apply -f lab-05-kafka/kafka.yaml
```

Watch the brokers start (this takes a minute or two):

```bash
kubectl get pods -w -l app=kafka
```

Notice the ordered startup: `kafka-0` → `kafka-1` → `kafka-2`.

## Step 3 — Verify the Cluster

Check that all brokers registered with ZooKeeper:

```bash
kubectl exec kafka-0 -- kafka-broker-api-versions --bootstrap-server kafka-0.kafka-headless:9092
```

List the brokers:

```bash
kubectl exec kafka-0 -- kafka-metadata --snapshot /var/lib/kafka/data/__cluster_metadata-0/00000000000000000000.log --broker-list 2>/dev/null || \
kubectl exec kafka-0 -- bash -c "echo dump | nc zk-0.zk-headless 2181 | grep broker"
```

## Step 4 — Create a Topic

```bash
kubectl exec kafka-0 -- kafka-topics --create \
  --topic orders \
  --bootstrap-server kafka-0.kafka-headless:9092 \
  --partitions 3 \
  --replication-factor 3
```

Verify the topic:

```bash
kubectl exec kafka-0 -- kafka-topics --describe \
  --topic orders \
  --bootstrap-server kafka-0.kafka-headless:9092
```

You'll see output like:

```
Topic: orders   Partitions: 3   ReplicationFactor: 3
  Partition 0:  Leader: 1   Replicas: 1,0,2   Isr: 1,0,2
  Partition 1:  Leader: 2   Replicas: 2,1,0   Isr: 2,1,0
  Partition 2:  Leader: 0   Replicas: 0,2,1   Isr: 0,2,1
```

Each partition has a **leader** (handles reads/writes) and **replicas** (for fault tolerance). Leaders are distributed across brokers — this is Kafka's way of spreading the load.

## Step 5 — Produce and Consume Messages

Open **two terminals**.

**Terminal 1 — Start a consumer:**

```bash
kubectl exec -it kafka-0 -- kafka-console-consumer \
  --topic orders \
  --bootstrap-server kafka-0.kafka-headless:9092 \
  --from-beginning
```

**Terminal 2 — Produce some messages:**

```bash
kubectl exec -it kafka-1 -- kafka-console-producer \
  --topic orders \
  --bootstrap-server kafka-1.kafka-headless:9092
```

Type messages and press Enter:
```
order-001: 5x Widget A
order-002: 10x Widget B
order-003: 3x Widget C
```

Watch them appear in Terminal 1. Press `Ctrl+C` in both terminals when done.

> **Notice:** We produced from `kafka-1` and consumed from `kafka-0`. The headless service lets us talk to any specific broker, and Kafka routes messages through the cluster.

## Step 6 — Test Broker Failure and Recovery

Kill a broker:

```bash
kubectl delete pod kafka-1
```

Check the topic — partitions that had `kafka-1` as leader will elect a new leader:

```bash
kubectl exec kafka-0 -- kafka-topics --describe \
  --topic orders \
  --bootstrap-server kafka-0.kafka-headless:9092
```

Wait for `kafka-1` to come back:

```bash
kubectl get pods -w -l app=kafka
```

Check the topic again — `kafka-1` rejoins the ISR (In-Sync Replicas):

```bash
kubectl exec kafka-0 -- kafka-topics --describe \
  --topic orders \
  --bootstrap-server kafka-0.kafka-headless:9092
```

**This is the power of StatefulSets for Kafka:**
- `kafka-1` came back with the **same broker ID** (1)
- It reattached to the **same EBS volume** with its message data
- It re-registered with ZooKeeper using the **same DNS name**
- It resumed replication and rejoined the ISR

With a Deployment, the replacement pod would have a new random name, no data, and no idea it was ever "broker 1."

## Step 7 — Inspect the Storage

```bash
kubectl get pvc -l app=kafka
kubectl get pvc -l app=zookeeper
```

You'll see 6 PVCs total — 3 for Kafka (10Gi each) and 3 for ZooKeeper (2Gi each). That's 6 EBS volumes in your AWS account, each dedicated to a specific pod.

## Exercises

1. **Scale up a broker.** Run `kubectl scale statefulset kafka --replicas=4`. Watch `kafka-3` appear. Create a new topic with `replication-factor 4` and verify all 4 brokers participate.

2. **Observe partition reassignment.** After adding `kafka-3`, existing partitions are still only on brokers 0-2. New brokers don't automatically get existing data. In production, you'd use `kafka-reassign-partitions` to rebalance. Why doesn't Kafka do this automatically?

3. **Kill ZooKeeper and observe.** Delete `zk-1` and watch Kafka. Does Kafka keep working? (Yes — ZooKeeper has quorum with 2/3 nodes.) Now delete `zk-0` too. What happens when ZooKeeper loses quorum?

4. **Think about production Kafka.** What would you add for a production deployment? Consider:
   - Pod anti-affinity (spread brokers across nodes/AZs)
   - Pod Disruption Budgets (prevent too many brokers going down at once)
   - Monitoring (JMX metrics, Prometheus)
   - KRaft mode (Kafka without ZooKeeper — the future)

## Clean Up

```bash
kubectl delete -f lab-05-kafka/kafka.yaml
kubectl delete -f lab-05-kafka/zookeeper.yaml

# Clean up all PVCs
kubectl delete pvc -l app=kafka
kubectl delete pvc -l app=zookeeper
```

## Congratulations!

You've completed all 5 labs. Here's what you've learned:

| Lab | Concept | Real-World Relevance |
|-----|---------|---------------------|
| 01 | Stable pod names, ordered creation | Pods have predictable identities |
| 02 | Headless services, per-pod DNS | Pods can find each other by name |
| 03 | VolumeClaimTemplates, per-pod storage | Each pod keeps its own data |
| 04 | PostgreSQL primary-replica | Database clusters on Kubernetes |
| 05 | Kafka multi-broker cluster | Distributed messaging on Kubernetes |

### When to Use StatefulSets (Decision Guide)

Ask yourself:
- Does each instance need its **own storage**? → StatefulSet
- Do instances need to **find each other by name**? → StatefulSet
- Does it matter **which instance** handles a request? → StatefulSet
- Do instances need to start in a **specific order**? → StatefulSet
- Is the app completely **stateless** and interchangeable? → Deployment
