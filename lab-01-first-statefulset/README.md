# Lab 01 — Your First StatefulSet

## Objective

Create a StatefulSet and observe how it behaves differently from a Deployment. By the end, you'll understand **stable pod names** and **ordered creation/deletion**.

## Concept

When you create a StatefulSet with 3 replicas, Kubernetes creates pods **one at a time, in order**:

```
web-0 → (wait until Running) → web-1 → (wait until Running) → web-2
```

Each pod gets a **predictable, stable name** — not a random hash like Deployments give you.

## Step 1 — Deploy a StatefulSet

```bash
kubectl apply -f lab-01-first-statefulset/statefulset.yaml
```

Now watch the pods come up. Notice the **order**:

```bash
kubectl get pods -w -l app=nginx
```

You should see:
```
NAME    READY   STATUS    AGE
web-0   1/1     Running   5s
web-1   1/1     Running   10s
web-2   1/1     Running   15s
```

> **Key observation:** Pods are named `web-0`, `web-1`, `web-2` — not `web-7d9f8b6c4-xk2lp`. The name comes from the StatefulSet name (`web`) plus an ordinal index.

Press `Ctrl+C` to stop watching.

## Step 2 — Compare with a Deployment

```bash
kubectl apply -f lab-01-first-statefulset/deployment-comparison.yaml
```

```bash
kubectl get pods -l app=nginx-deploy
```

Notice the difference:
```
NAME                              READY   STATUS    AGE
web-deployment-7d9f8b6c4-abc12   1/1     Running   3s
web-deployment-7d9f8b6c4-def34   1/1     Running   3s
web-deployment-7d9f8b6c4-ghi56   1/1     Running   3s
```

Deployment pods have **random suffixes** and were created **all at once**.

## Step 3 — Test Pod Identity Stability

Delete a StatefulSet pod and watch what happens:

```bash
kubectl delete pod web-1
```

```bash
kubectl get pods -w -l app=nginx
```

The replacement pod is **still called `web-1`**. Same name, same ordinal. This is what "stable identity" means.

Now try the same with a Deployment pod:

```bash
# Pick one of your deployment pod names
kubectl delete pod <your-deployment-pod-name>
```

The replacement gets a **completely new random name**.

## Step 4 — Observe Ordered Deletion

Scale the StatefulSet down:

```bash
kubectl scale statefulset web --replicas=1
```

```bash
kubectl get pods -w -l app=nginx
```

Watch the order: `web-2` is deleted first, then `web-1`. Deletion happens in **reverse order** (highest ordinal first).

Scale back up:

```bash
kubectl scale statefulset web --replicas=3
```

Creation happens in **forward order** again: `web-1` first, then `web-2`.

## Exercises

1. **Why does ordered creation matter?** Think about a database cluster. The primary (pod-0) needs to be running before replicas (pod-1, pod-2) can connect to it. What would happen if all three started at the same time?

2. **Why do stable names matter?** Imagine pod-1 is a replica configured to replicate from `web-0.nginx`. If the pod restarted with a random new name, how would it know where to find the primary?

## Clean Up

```bash
kubectl delete -f lab-01-first-statefulset/statefulset.yaml
kubectl delete -f lab-01-first-statefulset/deployment-comparison.yaml
```

## Next Step

Now that you understand stable identities, let's dive deeper into **how** pods find each other → [Lab 02 — Headless Services](../lab-02-headless-services/README.md).
