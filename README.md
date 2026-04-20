# Kubernetes StatefulSets Lab

A hands-on lab series that teaches you how organizations use **StatefulSets** in the real world — from the basics all the way to running databases and message brokers on EKS.

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

## How to Use This Lab

Work through the labs **in order**. Each lab builds on concepts from the previous one.

Every lab follows the same format:
1. **Concept** — Short explanation of what you're learning
2. **Deploy** — Apply the manifests and observe behavior
3. **Explore** — Guided exercises to deepen understanding
4. **Clean Up** — Tear down resources before moving on

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
