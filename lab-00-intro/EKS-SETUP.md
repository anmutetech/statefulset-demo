# EKS Cluster Setup Guide

This guide walks you through setting up an EKS cluster from scratch for the StatefulSets lab. By the end, you will have a 3-node Kubernetes cluster on AWS with everything needed to run Labs 01-05.

## What You Will Set Up

```
Your Laptop                          AWS (us-east-1)
┌──────────────┐                     ┌────────────────────────────────────┐
│ AWS CLI      │                     │  EKS Cluster: statefulset-lab      │
│ eksctl       │ ──── kubectl ────>  │  ┌──────────────────────────────┐  │
│ kubectl      │                     │  │  Node 1 (t3.medium)          │  │
└──────────────┘                     │  │  Node 2 (t3.medium)          │  │
                                     │  │  Node 3 (t3.medium)          │  │
                                     │  └──────────────────────────────┘  │
                                     │  EBS CSI Driver (for PVC volumes)  │
                                     └────────────────────────────────────┘
```

**Estimated time:** 20-30 minutes (most of it is waiting for AWS to provision resources).

**Estimated cost:** ~$0.30/hour for the cluster + nodes. Delete the cluster when done to stop charges.

---

## Step 1 -- Install the Required Tools

You need three CLI tools on your laptop. Check if you already have them:

```bash
aws --version
eksctl version
kubectl version --client
```

If any of these fail, install the missing tool(s) below.

### AWS CLI

The AWS CLI lets you interact with AWS services from your terminal.

**macOS:**
```bash
brew install awscli
```

**Linux:**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**Windows:**

Download and run the installer from https://aws.amazon.com/cli/

### eksctl

`eksctl` is the official CLI for creating and managing EKS clusters. It automates what would otherwise be dozens of CloudFormation steps.

**macOS:**
```bash
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

**Linux:**
```bash
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
tar -xzf eksctl_$PLATFORM.tar.gz -C /tmp && sudo mv /tmp/eksctl /usr/local/bin
```

**Windows:**
```powershell
choco install eksctl
```

### kubectl

`kubectl` is the Kubernetes command-line tool. You will use it for every lab.

**macOS:**
```bash
brew install kubectl
```

**Linux:**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

**Windows:**
```powershell
choco install kubernetes-cli
```

---

## Step 2 -- Configure AWS Credentials

If you haven't configured the AWS CLI yet, run:

```bash
aws configure
```

It will prompt you for:

```
AWS Access Key ID:     <your-access-key>
AWS Secret Access Key: <your-secret-key>
Default region name:   us-east-1
Default output format: json
```

> **Where do I get these keys?** Go to the AWS Console > IAM > Users > your user > Security credentials > Create access key. If you are using an organization account, ask your admin.

Verify it works:

```bash
aws sts get-caller-identity
```

You should see your account ID and user ARN. If this fails, double-check your credentials.

---

## Step 3 -- Create the EKS Cluster

This single command creates the entire cluster: VPC, subnets, security groups, the EKS control plane, and a managed node group with 3 worker nodes.

```bash
eksctl create cluster \
  --name statefulset-lab \
  --region us-east-1 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 3 \
  --managed
```

**This takes 15-20 minutes.** `eksctl` will print progress as it creates CloudFormation stacks. Don't close your terminal.

You will see output like:

```
2024-01-15 10:30:00 [ℹ]  eksctl version 0.167.0
2024-01-15 10:30:00 [ℹ]  using region us-east-1
2024-01-15 10:30:02 [ℹ]  setting availability zones to [us-east-1a us-east-1b]
2024-01-15 10:30:02 [ℹ]  subnets for us-east-1a - public:192.168.0.0/19 private:192.168.64.0/19
...
2024-01-15 10:47:30 [✔]  EKS cluster "statefulset-lab" in "us-east-1" region is ready
```

When it finishes, `eksctl` automatically configures `kubectl` to talk to your new cluster.

### Verify the cluster is running

```bash
kubectl get nodes
```

Expected output:

```
NAME                             STATUS   ROLES    AGE   VERSION
ip-192-168-25-118.ec2.internal   Ready    <none>   2m    v1.29.0-eks-xxxxx
ip-192-168-48-205.ec2.internal   Ready    <none>   2m    v1.29.0-eks-xxxxx
ip-192-168-71-42.ec2.internal    Ready    <none>   2m    v1.29.0-eks-xxxxx
```

Three nodes, all `Ready`. Your cluster is up.

> **Why 3 nodes?** Labs 04 and 05 deploy multiple StatefulSet pods (3 PostgreSQL instances, 3 Kafka brokers, 3 ZooKeeper nodes). Spreading them across 3 nodes gives you a realistic distributed setup and prevents resource contention.

> **Why t3.medium?** Each t3.medium node has 2 vCPUs and 4 GiB memory -- enough for the lab workloads. In production, you would size nodes based on your actual database and Kafka resource requirements.

---

## Step 4 -- Install the EBS CSI Driver

**This step is critical.** Labs 03, 04, and 05 use PersistentVolumeClaims to create EBS volumes. On EKS 1.23+, the EBS CSI (Container Storage Interface) driver must be installed separately -- it does not come pre-installed.

Without it, your PVCs will stay stuck in `Pending` and pods won't start.

### 4a -- Create an IAM role for the driver

The EBS CSI driver needs permission to create and manage EBS volumes in your AWS account. This command creates an IAM role with the right permissions and associates it with your cluster:

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster statefulset-lab \
  --region us-east-1 \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --role-only \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve
```

### 4b -- Install the EBS CSI add-on

```bash
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster statefulset-lab \
  --region us-east-1 \
  --service-account-role-arn "arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/AmazonEKS_EBS_CSI_DriverRole" \
  --force
```

### 4c -- Verify the driver is running

```bash
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
```

Expected output (you should see controller and node pods):

```
NAME                                  READY   STATUS    RESTARTS   AGE
ebs-csi-controller-5b8f48db76-xxxxx   6/6     Running   0          60s
ebs-csi-controller-5b8f48db76-yyyyy   6/6     Running   0          60s
ebs-csi-node-abc12                    3/3     Running   0          60s
ebs-csi-node-def34                    3/3     Running   0          60s
ebs-csi-node-ghi56                    3/3     Running   0          60s
```

### 4d -- Verify the gp2 storage class exists

```bash
kubectl get storageclass
```

Expected output:

```
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      AGE
gp2 (default)   kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   20m
```

`gp2` should be listed as the default storage class. This is what the lab manifests reference in their `storageClassName` field.

> **What if gp2 is not listed?** Create it manually:
> ```bash
> kubectl apply -f - <<EOF
> apiVersion: storage.k8s.io/v1
> kind: StorageClass
> metadata:
>   name: gp2
>   annotations:
>     storageclass.kubernetes.io/is-default-class: "true"
> provisioner: ebs.csi.aws.com
> volumeBindingMode: WaitForFirstConsumer
> parameters:
>   type: gp2
> EOF
> ```

---

## Step 5 -- Quick Smoke Test

Before starting the labs, verify that everything works end-to-end -- cluster, networking, and storage:

```bash
kubectl run test-pod --image=nginx:1.25 --restart=Never
kubectl wait --for=condition=Ready pod/test-pod --timeout=60s
kubectl exec test-pod -- nginx -v
kubectl delete pod test-pod
```

If you see the nginx version printed, your cluster is ready. Move on to [Lab 00 -- Why StatefulSets?](README.md).

---

## Troubleshooting

### "error: You must be logged in to the server"

Your kubeconfig is not set up. Run:

```bash
aws eks update-kubeconfig --name statefulset-lab --region us-east-1
```

### "Cannot create cluster: insufficient IAM permissions"

Your AWS user needs these permissions: `eks:*`, `ec2:*`, `iam:*`, `cloudformation:*`. Ask your admin to attach the `AdministratorAccess` policy for lab purposes, or create a scoped policy.

### PVCs stuck in "Pending"

This almost always means the EBS CSI driver is not installed. Go back to Step 4. You can also check:

```bash
kubectl describe pvc <pvc-name>
```

Look for events like `waiting for a volume to be created` or `no persistent volumes available`.

### Pods stuck in "Pending" (not PVC-related)

Check if nodes have enough resources:

```bash
kubectl describe nodes | grep -A 5 "Allocated resources"
```

If nodes are full, you may need to scale up:

```bash
eksctl scale nodegroup --cluster statefulset-lab --name workers --nodes 4 --region us-east-1
```

### "eksctl create cluster" times out or fails

- Check your internet connection
- Verify your AWS region has capacity: try `us-west-2` if `us-east-1` fails
- Check the CloudFormation console in AWS for detailed error messages

---

## Cluster Cleanup (When You Are Done With All Labs)

**Important:** Delete the cluster when you are finished to stop AWS charges. The cluster costs ~$0.30/hour.

First, make sure all lab PVCs are deleted (EBS volumes cost money even without a cluster):

```bash
kubectl delete pvc --all
```

Then delete the cluster:

```bash
eksctl delete cluster --name statefulset-lab --region us-east-1
```

This takes 5-10 minutes. It removes the EKS cluster, node group, VPC, and all associated resources.

Verify in the AWS console (EC2 > Volumes) that no orphaned EBS volumes remain from the labs.
