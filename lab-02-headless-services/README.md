# Lab 02 -- Headless Services & Stable Network IDs

## The Scenario

You are sketching out QuickBite's order service architecture on a whiteboard. The order service runs as multiple instances -- a primary that processes payments and writes to the database, and replicas that serve read queries to the mobile app.

Your lead walks by and asks: "So how does replica-1 find primary-0 to set up replication? You can't just point it at a load balancer -- that might route to another replica."

Good question. The answer is **headless services**. This lab shows you how they work.

## Objective

Understand how **headless services** give each StatefulSet pod its own DNS name, and why this is critical for stateful applications like QuickBite's PostgreSQL and Kafka.

## Concept

A normal Kubernetes Service has a **ClusterIP** -- a single virtual IP that load-balances traffic across all pods. This is great for the QuickBite web frontend where you don't care which pod handles a request.

But for stateful apps, you **need to talk to a specific pod**. For example:
- A PostgreSQL replica needs to connect to the **primary** instance specifically to stream WAL records
- A Kafka consumer needs to talk to the **broker** that owns a specific partition of the `order.events` topic

A **headless service** (`clusterIP: None`) skips the load balancer and instead creates **individual DNS records** for each pod:

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

So with a StatefulSet named `web` and a headless service named `web-headless`:
```
web-0.web-headless.default.svc.cluster.local → 10.0.1.5
web-1.web-headless.default.svc.cluster.local → 10.0.2.8
web-2.web-headless.default.svc.cluster.local → 10.0.3.2
```

Each pod has a **stable, predictable DNS name** that other pods can use to reach it directly. This is how QuickBite's replicas will find the primary, and how Kafka consumers will connect to specific brokers.

## Step 1 -- Deploy the StatefulSet and Services

```bash
kubectl apply -f lab-02-headless-services/statefulset.yaml
kubectl apply -f lab-02-headless-services/dns-test-pod.yaml
```

Wait for everything to be running:

```bash
kubectl get pods -w
```

## Step 2 -- Compare the Two Services

```bash
kubectl get svc
```

You should see:
```
NAME           TYPE        CLUSTER-IP      PORT(S)   AGE
web-headless   ClusterIP   None            80/TCP    30s
web-regular    ClusterIP   10.100.45.123   80/TCP    30s
```

Notice `web-headless` has **no ClusterIP** -- that's what makes it headless.

## Step 3 -- Test DNS Resolution

Exec into the DNS test pod:

```bash
kubectl exec -it dns-test -- bash
```

Now run these DNS lookups inside the pod:

**Regular service -- returns one ClusterIP:**
```bash
nslookup web-regular
```

This is what the QuickBite mobile app would use to reach the stateless API gateway -- it does not care which instance answers.

**Headless service -- returns individual pod IPs:**
```bash
nslookup web-headless
```

You'll see multiple IP addresses, one for each pod.

**Individual pod DNS -- the real power of headless services:**
```bash
nslookup web-0.web-headless
nslookup web-1.web-headless
nslookup web-2.web-headless
```

Each resolves to the **specific IP** of that pod. This is how a PostgreSQL replica would know exactly where to find the primary -- it connects to `postgres-0.postgres-headless`, not a random instance.

Type `exit` to leave the pod.

## Step 4 -- Verify DNS Survives Pod Restart

```bash
kubectl delete pod web-1
```

Wait for it to come back:

```bash
kubectl get pods -w -l app=web
```

Now check DNS again:

```bash
kubectl exec dns-test -- nslookup web-1.web-headless
```

The DNS name `web-1.web-headless` still works -- it just points to the new pod's IP. The **name is stable** even though the underlying IP may change. Applications connect by DNS name, not IP.

This is exactly what you need for QuickBite: even if `kafka-2` crashes and restarts on a different node with a different IP, consumers configured to talk to `kafka-2.kafka-headless` will automatically find it again.

## Step 5 -- Understand the serviceName Field

Look at the StatefulSet YAML -- notice the `serviceName: "web-headless"` field. This is what connects the StatefulSet to the headless service and enables the per-pod DNS records.

```bash
kubectl get statefulset web -o jsonpath='{.spec.serviceName}'
```

> **Important:** If `serviceName` doesn't match an actual headless service, the per-pod DNS records won't be created.

## Exercises

1. **Why can't a regular Service do this?** A regular service gives you `web-regular.default.svc.cluster.local` which round-robins across all pods. Try `nslookup web-regular` multiple times -- you always get the same ClusterIP. How would a replica know which IP is the primary?

2. **Construct the full DNS name.** If you have a StatefulSet called `db` with a headless service called `postgres` in the `production` namespace, what's the full DNS name for the third pod? (Answer: `db-2.postgres.production.svc.cluster.local`)

3. **Think about QuickBite's Kafka cluster.** Each Kafka broker needs to advertise its own address so that when a producer sends an order event to partition 3, it connects directly to the broker that owns that partition. Why is a headless service essential for this?

## Clean Up

```bash
kubectl delete -f lab-02-headless-services/statefulset.yaml
kubectl delete -f lab-02-headless-services/dns-test-pod.yaml
```

## Next Step

Pods now have stable names and stable DNS -- but what about their **data**? If QuickBite's order database pod crashes, you need the order records to still be there when it comes back. Head to [Lab 03 -- Persistent Storage](../lab-03-persistent-storage/README.md).
