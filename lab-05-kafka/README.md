# Lab 05 -- QuickBite's Event Streaming: Kafka on EKS

## The Scenario

Your team lead is impressed with the PostgreSQL deployment. Now comes the second piece of QuickBite's backend:

> "When a customer places an order, a lot of things need to happen at once -- the restaurant gets notified, a driver gets dispatched, the customer sees a live tracker, and the analytics pipeline logs the event. We can't have the order service call each of these systems one by one. Instead, it publishes an event to Kafka, and every downstream service consumes it independently. Deploy a 3-broker Kafka cluster. I want to see real order events flowing through it."

This is event-driven architecture in action. Kafka is the backbone that decouples QuickBite's services.

### QuickBite's Event Flow

```
Customer places order
        │
        ▼
  Order Service
  (writes to PostgreSQL + publishes to Kafka)
        │
        ▼
  ┌─────────────────────────────────────────┐
  │          Kafka: order.events topic       │
  │  ┌───────────┬───────────┬───────────┐  │
  │  │Partition 0 │Partition 1 │Partition 2 │  │
  │  └───────────┴───────────┴───────────┘  │
  └─────────┬──────────┬──────────┬─────────┘
            │          │          │
     ┌──────▼───┐ ┌────▼─────┐ ┌─▼──────────┐
     │Notification│ │ Driver   │ │ Analytics  │
     │ Service   │ │ Dispatch │ │ Pipeline   │
     └──────────┘ └──────────┘ └────────────┘
```

Each downstream service reads from the same topic independently. If the analytics pipeline is slow, it doesn't block driver dispatch. If the notification service crashes, it can catch up later by replaying events.

## Architecture

```
  Order Service                              Downstream Consumers
  (produces events)                          (notification, dispatch, analytics)
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

### New Concepts in This Lab

| Concept | What It Is |
|---------|-----------|
| **ZooKeeper** | A distributed coordination service. Kafka uses it to track which brokers are alive, which broker leads each partition, and cluster configuration. Think of it as the "control plane" for the Kafka cluster. |
| **Broker** | A single Kafka server. Each broker stores a subset of the data (partitions) and serves producer/consumer requests. |
| **Topic** | A named stream of events (like `order.events`). Producers write to topics, consumers read from them. |
| **Partition** | A topic is split into partitions for parallelism. Each partition lives on a specific broker and has its own ordered log of events. |
| **Replication Factor** | How many copies of each partition exist across brokers. Factor 3 means every event is stored on 3 brokers for fault tolerance. |
| **ISR (In-Sync Replicas)** | The set of replicas that are fully caught up with the leader. If a broker fails, a new leader is elected from the ISR. |

### Why Kafka Needs StatefulSets

1. **Broker ID** -- Each broker has a unique integer ID (0, 1, 2). We derive this from the pod ordinal (`kafka-0` becomes broker 0). If a broker restarted with a random ID, it would lose ownership of its partitions.
2. **Advertised Listeners** -- Each broker tells clients its own address: `kafka-0.kafka-headless:9092`. When the order service produces an event and Kafka says "send it to broker 2", the service needs to resolve `kafka-2.kafka-headless` to an actual IP. Without stable DNS from a headless service, this falls apart.
3. **Log Storage** -- Kafka stores QuickBite's order events on disk. Each broker needs its own persistent volume so events survive restarts. If broker 1 loses its data, every event it was responsible for is gone -- that means missed notifications, lost analytics, and drivers never dispatched.
4. **Ordered Startup** -- Brokers need ZooKeeper to be running first for cluster coordination. ZooKeeper itself needs ordered startup for leader election.

## Step 1 -- Deploy ZooKeeper First

Kafka depends on ZooKeeper for cluster coordination, so deploy it first:

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

Expected: `imok` (ZooKeeper's way of saying "I'm OK").

> **Troubleshooting:** If you get no response, the pod may still be starting. Wait 10 seconds and try again. Check `kubectl logs zk-0` if it persists.

ZooKeeper is itself a StatefulSet -- each node needs a unique server ID (derived from pod ordinal), its own data directory, and stable DNS so the other nodes can find it for leader election.

## Step 2 -- Deploy Kafka

```bash
kubectl apply -f lab-05-kafka/kafka.yaml
```

Watch the brokers start (this takes a minute or two):

```bash
kubectl get pods -w -l app=kafka
```

Notice the ordered startup: `kafka-0` then `kafka-1` then `kafka-2`. Each broker registers with ZooKeeper using its stable DNS name before the next one starts.

## Step 3 -- Verify the Cluster

Check that all brokers registered with ZooKeeper:

```bash
kubectl exec kafka-0 -- kafka-broker-api-versions --bootstrap-server kafka-0.kafka-headless:9092
```

If the command produces output (a list of API versions) without errors, the broker is healthy. Any timeout or connection error means the broker isn't ready yet.

List the brokers registered in ZooKeeper:

```bash
kubectl exec kafka-0 -- bash -c "echo dump | nc zk-0.zk-headless 2181 | grep broker"
```

Expected output (3 lines, one per broker):

```
/brokers/ids/0
/brokers/ids/1
/brokers/ids/2
```

## Step 4 -- Create the Order Events Topic

This is the topic that powers QuickBite's real-time event flow. 3 partitions allow parallel processing, and replication factor 3 ensures no data loss if a broker fails:

```bash
kubectl exec kafka-0 -- kafka-topics --create \
  --topic order.events \
  --bootstrap-server kafka-0.kafka-headless:9092 \
  --partitions 3 \
  --replication-factor 3
```

Verify the topic:

```bash
kubectl exec kafka-0 -- kafka-topics --describe \
  --topic order.events \
  --bootstrap-server kafka-0.kafka-headless:9092
```

You'll see output like:

```
Topic: order.events   Partitions: 3   ReplicationFactor: 3
  Partition 0:  Leader: 1   Replicas: 1,0,2   Isr: 1,0,2
  Partition 1:  Leader: 2   Replicas: 2,1,0   Isr: 2,1,0
  Partition 2:  Leader: 0   Replicas: 0,2,1   Isr: 0,2,1
```

Each partition has a **leader** (handles reads/writes) and **replicas** (for fault tolerance). Leaders are distributed across brokers so no single broker is a bottleneck during lunch rush.

## Step 5 -- Simulate the QuickBite Event Flow

Open **two terminals** to simulate the order service producing events and a downstream service consuming them.

**Terminal 1 -- Start the notification service (consumer):**

```bash
kubectl exec -it kafka-0 -- kafka-console-consumer \
  --topic order.events \
  --bootstrap-server kafka-0.kafka-headless:9092 \
  --from-beginning
```

**Terminal 2 -- Simulate the order service (producer):**

```bash
kubectl exec -it kafka-1 -- kafka-console-producer \
  --topic order.events \
  --bootstrap-server kafka-1.kafka-headless:9092
```

Type these order events and press Enter after each:
```
{"order_id": 1001, "customer": "Alice Chen", "restaurant": "Thai Palace", "status": "placed", "total": 32.50}
{"order_id": 1001, "customer": "Alice Chen", "restaurant": "Thai Palace", "status": "preparing"}
{"order_id": 1002, "customer": "Bob Martinez", "restaurant": "Sushi Express", "status": "placed", "total": 28.00}
{"order_id": 1001, "customer": "Alice Chen", "restaurant": "Thai Palace", "status": "out_for_delivery", "driver": "Dave"}
{"order_id": 1001, "customer": "Alice Chen", "restaurant": "Thai Palace", "status": "delivered"}
```

Watch them appear in Terminal 1 in real time. This is exactly how QuickBite's notification service would receive events to push alerts to the customer's phone: "Your order is being prepared", "Your driver Dave is on the way", "Your food has arrived!"

Press `Ctrl+C` in both terminals when done.

> **Notice:** We produced from `kafka-1` and consumed from `kafka-0`. The headless service lets us talk to any specific broker, and Kafka routes messages through the cluster. In production, the order service would connect to any broker as a bootstrap and Kafka handles the rest.

## Step 6 -- Test Broker Failure and Recovery

It is lunch rush and broker 1 crashes. What happens to the order events?

```bash
kubectl delete pod kafka-1
```

Check the topic -- partitions that had `kafka-1` as leader will elect a new leader:

```bash
kubectl exec kafka-0 -- kafka-topics --describe \
  --topic order.events \
  --bootstrap-server kafka-0.kafka-headless:9092
```

The cluster is still operational. Producers and consumers keep working because the surviving brokers take over leadership of affected partitions. No order events are lost because every partition has replicas on other brokers.

Wait for `kafka-1` to come back:

```bash
kubectl get pods -w -l app=kafka
```

Check the topic again -- `kafka-1` rejoins the ISR (In-Sync Replicas):

```bash
kubectl exec kafka-0 -- kafka-topics --describe \
  --topic order.events \
  --bootstrap-server kafka-0.kafka-headless:9092
```

**This is the power of StatefulSets for Kafka:**
- `kafka-1` came back with the **same broker ID** (1)
- It reattached to the **same EBS volume** with its event data
- It re-registered with ZooKeeper using the **same DNS name** (`kafka-1.kafka-headless`)
- It resumed replication and rejoined the ISR

With a Deployment, the replacement pod would have a new random name, no data, and no idea it was ever "broker 1." Every consumer connected to that broker would be permanently disconnected.

## Step 7 -- Inspect the Storage

```bash
kubectl get pvc -l app=kafka
kubectl get pvc -l app=zookeeper
```

You'll see 6 PVCs total -- 3 for Kafka (10Gi each) and 3 for ZooKeeper (2Gi each). That is 6 EBS volumes in your AWS account, each dedicated to a specific pod. For QuickBite's production cluster, these volumes would hold days or weeks of order events (configurable via `KAFKA_LOG_RETENTION_HOURS`).

## Exercises

1. **Scale up for Black Friday.** QuickBite is expecting 10x normal traffic. Run `kubectl scale statefulset kafka --replicas=4`. Watch `kafka-3` appear. Create a new topic `order.events.v2` with `replication-factor 4` and verify all 4 brokers participate. Could you do this with a Deployment? (No -- the new pod would need a predictable broker ID and its own storage.)

2. **Observe partition reassignment.** After adding `kafka-3`, existing partitions of `order.events` are still only on brokers 0-2. The new broker sits idle for old topics. In production, you'd use `kafka-reassign-partitions` to rebalance load. Why doesn't Kafka do this automatically? (Hint: moving partitions means moving data, which costs network bandwidth and disk I/O.)

3. **Kill ZooKeeper and observe.** Delete `zk-1` and watch Kafka. Does Kafka keep working? (Yes -- ZooKeeper has quorum with 2/3 nodes.) Now delete `zk-0` too. What happens when ZooKeeper loses quorum? What would this mean for QuickBite during lunch rush?

4. **Think about production Kafka for QuickBite.** What would you add before going live?
   - **Pod anti-affinity** -- spread brokers across nodes and AZs so a single AZ outage doesn't take down all brokers
   - **Pod Disruption Budgets** -- prevent Kubernetes from evicting too many brokers at once during node upgrades
   - **Monitoring** -- JMX metrics exported to Prometheus, alerts on consumer lag (if the notification service falls behind, customers stop getting updates)
   - **KRaft mode** -- Kafka's replacement for ZooKeeper, removing an entire StatefulSet from the architecture

## Clean Up

```bash
kubectl delete -f lab-05-kafka/kafka.yaml
kubectl delete -f lab-05-kafka/zookeeper.yaml

# Clean up all PVCs
kubectl delete pvc -l app=kafka
kubectl delete pvc -l app=zookeeper
```

## Congratulations!

You have completed your onboarding at QuickBite. Here is what you built and learned:

| Lab | What You Built | StatefulSet Concept |
|-----|---------------|---------------------|
| 01 | Compared Deployments vs StatefulSets | Stable pod names, ordered creation |
| 02 | Explored DNS-based service discovery | Headless services, per-pod DNS |
| 03 | Proved data survives pod crashes | VolumeClaimTemplates, per-pod storage |
| 04 | Deployed QuickBite's PostgreSQL cluster | Primary-replica database on EKS |
| 05 | Deployed QuickBite's Kafka cluster | Multi-broker event streaming on EKS |

### When to Use StatefulSets (Decision Guide)

Next time your team needs to deploy a service on Kubernetes, ask yourself:

- Does each instance need its **own storage**? (like each Kafka broker's event logs) -- StatefulSet
- Do instances need to **find each other by name**? (like PostgreSQL replicas finding the primary) -- StatefulSet
- Does it matter **which instance** handles a request? (like Kafka partition leaders) -- StatefulSet
- Do instances need to start in a **specific order**? (like ZooKeeper before Kafka) -- StatefulSet
- Is the app completely **stateless** and interchangeable? (like the QuickBite web frontend) -- Deployment

You are now ready to take this to production. Go tell your team lead.
