# QuickBite Platform Engineering: StatefulSets Lab

Welcome to the QuickBite platform team. QuickBite is a food delivery platform that handles thousands of orders per day -- connecting hungry customers with local restaurants and delivery drivers in real time.

You just joined as a **platform engineer**. The company is migrating from VMs to Kubernetes on EKS, and your first project is deploying the stateful backend services: **PostgreSQL** (order/restaurant/customer database) and **Kafka** (real-time order event streaming).

Your team lead has one requirement before you touch production: **understand StatefulSets inside and out**. This lab series is your onboarding path.

## Prerequisites

- A running EKS cluster (1.27+)
- `kubectl` configured to talk to your cluster
- Familiarity with **Pods** and **Deployments**
- Basic understanding of YAML and the command line

## Lab Overview

| Lab | Title | Level | What You'll Learn |
|-----|-------|-------|-------------------|
| 00 | [Why StatefulSets?](lab-00-intro/README.md) | Beginner | The problem StatefulSets solve and how they differ from Deployments |
| 01 | [Your First StatefulSet](lab-01-first-statefulset/README.md) | Beginner | Create a StatefulSet, observe ordered pod creation, and stable pod names |
| 02 | [Headless Services & Stable Network IDs](lab-02-headless-services/README.md) | Beginner | DNS-based discovery and how pods get predictable hostnames |
| 03 | [Persistent Storage](lab-03-persistent-storage/README.md) | Intermediate | VolumeClaimTemplates, per-pod storage, and data survival across restarts |
| 04 | [Real-World: PostgreSQL on EKS](lab-04-postgresql/README.md) | Intermediate | Deploy a PostgreSQL primary-replica cluster using StatefulSets |
| 05 | [Real-World: Kafka on EKS](lab-05-kafka/README.md) | Intermediate | Deploy a multi-broker Kafka cluster with ZooKeeper on EKS |

## The QuickBite Architecture

Here is what you will eventually be responsible for deploying:

- **PostgreSQL** -- stores orders, restaurants, customers, and driver data. Runs as a primary-replica cluster where the primary handles writes and replicas serve read traffic from the mobile app.
- **Kafka** -- streams order lifecycle events (`order.placed`, `order.preparing`, `order.out_for_delivery`, `order.delivered`) so that the notification service, driver dispatch, and analytics pipeline all react in real time.

Both of these are stateful systems that cannot run as simple Deployments. By the end of this lab series, you will understand exactly why -- and you will have deployed both on EKS yourself.

## Kubernetes Concepts Reference

Labs 01-03 only require knowledge of Pods and Deployments. Labs 04-05 introduce a few more Kubernetes concepts. Here's a quick reference -- each concept is explained in context when it first appears, but this table is here if you need a refresher.

| Concept | One-Line Explanation | First Used In |
|---------|---------------------|---------------|
| **Service** | A stable network endpoint that routes traffic to a set of pods | Lab 01 |
| **Headless Service** | A Service with `clusterIP: None` that gives each pod its own DNS name instead of load-balancing | Lab 01 (explained in Lab 02) |
| **PersistentVolumeClaim (PVC)** | A request for storage -- Kubernetes finds or creates an actual disk (EBS volume on EKS) to satisfy it | Lab 03 |
| **VolumeClaimTemplate** | A PVC template inside a StatefulSet -- automatically creates a unique PVC for each pod | Lab 03 |
| **StorageClass (`gp2`)** | Tells Kubernetes what type of disk to create -- `gp2` is the default EBS volume type on EKS | Lab 03 |
| **ConfigMap** | Stores configuration files and scripts as key-value pairs that pods can mount as files | Lab 04 |
| **Secret** | Like ConfigMap but for sensitive data (passwords); values are base64-encoded | Lab 04 |
| **initContainer** | A container that runs once before the main container starts -- used for setup tasks | Lab 04 |
| **Readiness Probe** | A health check that tells Kubernetes "this pod is ready to accept traffic" | Lab 04 |
| **Liveness Probe** | A health check -- if it fails, Kubernetes restarts the pod | Lab 04 |

## How to Use This Lab

Work through the labs **in order**. Each lab builds on concepts from the previous one.

Every lab follows the same format:
1. **Concept** -- Short explanation of what you're learning
2. **Deploy** -- Apply the manifests and observe behavior
3. **Explore** -- Guided exercises to deepen understanding
4. **Clean Up** -- Tear down resources before moving on

## EKS Cluster Setup

If you don't have an EKS cluster yet, here's a quick setup using `eksctl`:

```bash
eksctl create cluster \
  --name statefulset-lab \
  --region us-east-1 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 3 \
  --managed
```

> **Note:** 3 nodes are recommended so you can see pods distributed across nodes, especially for the Kafka lab.

## Cleanup

When you're done with all labs:

```bash
eksctl delete cluster --name statefulset-lab --region us-east-1
```
