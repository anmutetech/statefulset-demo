# Lab 00 — Why StatefulSets?

## The Problem

You already know Deployments. They're great for **stateless** apps — any pod can handle any request, pods are interchangeable, and you don't care which pod is which.

But what happens when your app **needs to remember things**?

Think about a database like PostgreSQL:
- Each instance needs its **own dedicated storage** that survives pod restarts
- The **primary** instance handles writes; **replicas** handle reads — they are NOT interchangeable
- When a pod restarts, it needs to come back with the **same identity** (same hostname, same storage)
- You can't just kill all pods at once — you need to bring them down **one at a time, in order**

Deployments can't do any of this. **StatefulSets can.**

## Deployment vs StatefulSet

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| Pod names | Random (e.g., `web-7d9f8b6c4-xk2lp`) | Predictable (e.g., `web-0`, `web-1`, `web-2`) |
| Pod creation | All at once (parallel) | One at a time, in order (0 → 1 → 2) |
| Pod deletion | Any order | Reverse order (2 → 1 → 0) |
| Storage | Shared or none | Each pod gets its own persistent volume |
| Network identity | Random, changes on restart | Stable DNS name per pod |
| Best for | Stateless apps (web servers, APIs) | Stateful apps (databases, message queues, caches) |

## Real-World Examples

Organizations use StatefulSets for:

- **Databases** — PostgreSQL, MySQL, MongoDB, Cassandra
- **Message brokers** — Kafka, RabbitMQ, NATS
- **Search engines** — Elasticsearch, OpenSearch
- **Caches** — Redis (cluster mode)
- **Distributed coordination** — ZooKeeper, etcd

In all these cases, each instance has a unique role or unique data, so it needs a stable identity and its own storage.

## Key Concepts You'll Learn

1. **Stable Pod Identity** — Pods get names like `app-0`, `app-1`, `app-2` that never change
2. **Ordered Operations** — Pods are created, updated, and deleted in a predictable sequence
3. **Headless Services** — A special Service type that gives each pod its own DNS name
4. **VolumeClaimTemplates** — Automatically create a unique PersistentVolumeClaim for each pod
5. **Stable Network IDs** — Each pod gets a DNS record: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`

## Next Step

Ready? Head to [Lab 01 — Your First StatefulSet](../lab-01-first-statefulset/README.md).
