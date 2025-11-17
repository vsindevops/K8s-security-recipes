# Day 1 — Kubernetes Security & Platform Hardening  
## Pod Security Admission (PSA) Basics on GKE — End-to-End Lab

This document is a **complete, step-by-step guide** for Day 1 of your Kubernetes Security & Platform Hardening journey.

You will:

- Create a **GKE cluster from scratch** on GCP  
- Understand **pod-level vs node-level** security  
- Enable **Pod Security Admission (PSA)** using the `restricted` profile  
- Deploy a **privileged pod** and see it **blocked by PSA**  
- Learn where **Seccomp**, **AppArmor**, and **PodSecurityPolicy (PSP)** fit in  

Everything you need for Day 1 is in this one file.

---

## 1. High-Level Objective

By the end of this lab you will:

- Have a **GKE cluster** running with 2 small worker nodes  
- Have a namespace with **PSA enforced in `restricted` mode**  
- Understand why a **privileged pod** is rejected  
- Grasp the big picture of:
  - Pod-level security
  - Node-level security
  - PSA vs legacy PSP
  - Seccomp & AppArmor at a basic level

---

## 2. Prerequisites

You only need:

- A **GCP project** with **billing enabled**
- **Google Cloud SDK (gcloud)** installed  
- `kubectl` installed (comes with gcloud if you run `gcloud components install kubectl`)

### 2.1. Authenticate with GCP

```bash
gcloud auth login
```
This opens a browser window → log in using your Google account.

### 2.2. Set the Active Project

```bash
gcloud config set project <PROJECT_ID>
```
Replace <PROJECT_ID> with your actual GCP project ID.

You can verify:

```bash
gcloud config get-value project
```

## 3. Enable Required APIs

GKE runs on top of several GCP services. At minimum, we must enable:
	•	GKE API → container.googleapis.com
	•	Compute Engine API → compute.googleapis.com

Run:
```bash
gcloud services enable container.googleapis.com
gcloud services enable compute.googleapis.com
```

Why?
	•	container.googleapis.com allows you to create and manage Kubernetes clusters.
	•	compute.googleapis.com is needed because GKE worker nodes are actually Compute Engine VMs.

## 4. (Recommended) Create a Dedicated VPC & Subnet

You could use the default VPC, but security best practice is to create a separate network for your cluster.

### 4.1. Create a Custom VPC

```bash
gcloud compute networks create k8s-secure-vpc --subnet-mode=custom
```

	•	--subnet-mode=custom means you will explicitly create subnets (no automatic ones).

### 4.2. Create a Subnet for the Cluster

```bash
gcloud compute networks subnets create k8s-subnet \
  --network=k8s-secure-vpc \
  --region=us-central1 \
  --range=10.10.0.0/24
```

	•	VPC: k8s-secure-vpc
	•	Subnet: k8s-subnet
	•	Region: asia-south1
	•	CIDR block: 10.10.0.0/24

You can change region and CIDR to your preference, but then stay consistent later.

## 5. Create a Secure GKE Cluster

Now we’ll create the cluster that we’ll use for security labs.

### 5.1. Define Some Variables

```bash
REGION=asia-south1
ZONE=asia-south1-a
CLUSTER_NAME=k8s-security-lab
PROJECT_ID=<YOUR_PROJECT_ID>
```

### 5.2. Create the Cluster (Standard, 2 Nodes, Small Instances)

gcloud container clusters create $CLUSTER_NAME \
  --zone $ZONE \
  --cluster-version latest \
  --machine-type e2-micro \
  --num-nodes 2 \
  --network k8s-secure-vpc \
  --subnetwork k8s-subnet \
  --enable-shielded-nodes \
  --workload-pool="$PROJECT_ID.svc.id.goog"

This gives you: 1 control plane (managed by GKE) + 2 worker nodes (e2-micro VMs).

## 6. Connect kubectl to the Cluster

After cluster creation completes, get cluster credentials:

```bash
gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE
```

This configures your kubeconfig so kubectl talks to your new cluster.

### 6.1. Verify Connectivity

```bash
kubectl get nodes
```

You should be able to see something like:

```bash
NAME                                         STATUS   ROLES    AGE   VERSION
gke-k8s-security-lab-default-pool-...       Ready    <none>   ...   v1.xx.x-gke.x
gke-k8s-security-lab-default-pool-...       Ready    <none>   ...   v1.xx.x-gke.x
```

If you see the nodes with STATUS = Ready, your cluster is good to go.


## 7. Concept: Pod-Level vs Node-Level Security (Read and Understand this first)

Before configuring PSA, it’s crucial to understand where different security mechanisms apply.

### 7.1. Pod-Level Security Controls

These control what a pod is allowed to request or do.

Examples:
	•	Pod Security Admission (PSA)
	•	Pod securityContext fields:
		-  runAsUser, runAsNonRoot
		-  privileged
		-  allowPrivilegeEscalation
		-  readOnlyRootFilesystem
		-  Linux capabilities (capAdd, capDrop)
	•   Seccomp profiles
	•	AppArmor profiles

Think of pod-level controls as:

“What behaviors can workloads inside this pod perform?”

They are enforced inside the Kubernetes API during admission.


### 7.2. Node-Level Security Controls

These protect the node (VM) itself and the broader infrastructure.

Examples:
	•	Shielded nodes (OS/boot attestation)
	•	Kubelet authentication & authorization
	•	OS patching & hardening
	•	File system lockdowns
	•	Container runtime configuration (containerd, CRI-O)
	•	IAM roles attached to the node
	•	VPC firewall rules, network policies, etc.

Think of node-level controls as:

“If a pod is compromised, how hard is it to break out and own the host or cloud account?”

---

## 8. Pod Security Admission (PSA) — What & Why

Pod Security Admission (PSA) is a Kubernetes admission plugin that enforces Pod Security Standards at the namespace level.

Kubernetes defines three standard policy levels:
	1.	privileged
	2.	baseline
	3.	restricted  ← Most secure and our focus today

You enable PSA by adding labels to a namespace. When a Pod is created in that namespace:
	•	PSA evaluates the Pod spec against the configured policy (baseline or restricted).
	•	If the spec violates the policy in enforce mode → the Pod is rejected.

Why PSA instead of PSP?
	•	PSA is simpler to understand and use.
	•	PSP (PodSecurityPolicy) was confusing and is deprecated & removed in Kubernetes v1.25.
	•	PSA + tools like Gatekeeper/Kyverno are the modern replacement.

---

## 9. Create a Namespace with enforce: restricted PSA

Now we create a namespace and tell Kubernetes:

“In this namespace, only pods that meet the restricted security profile are allowed.”

### 9.1. Create the Namespace

```bash
kubectl create namespace psa-restricted-demo
```

### 9.2. Add PSA Labels for Restricted Enforcement

```bash
kubectl label namespace psa-restricted-demo \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/enforce-version=latest
```

	•	pod-security.kubernetes.io/enforce → selects the policy level (privileged, baseline, or restricted).
	•	pod-security.kubernetes.io/enforce-version → which version of the Pod Security Standard to use (latest is fine for labs).

### 9.3. Verify Namespace Labels

```bash
kubectl get namespace psa-restricted-demo --show-labels
```

You should see output similar to:

```bash
NAME                  STATUS   AGE   LABELS
psa-restricted-demo   Active   ...   pod-security.kubernetes.io/enforce=restricted,...,pod-security.kubernetes.io/enforce-version=latest
```

This namespace is now PSA-restricted.

## 10. Create a Privileged Pod Manifest (Intentionally Unsafe)

Now we try to deploy a pod that violates restricted rules.

### 10.1. Create privileged-pod.yaml

Create a file locally named privileged-pod.yaml with the following content:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-test
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true
```

Key piece:
	•	securityContext.privileged: true
→ This asks Kubernetes to run the container with full privileges, similar to root on the host. This is extremely powerful and generally unsafe in shared environments.

Under the restricted PSA profile, this is not allowed.

⸻

## 11. Apply the Privileged Pod in the Restricted Namespace

Now we test the PSA policy.

### 11.1. Try to Create the Pod

```bash
kubectl apply -f privileged-pod.yaml -n psa-restricted-demo
```

### 11.2. Expected Output

You should see an error similar to:

```text
Error from server (Forbidden): pods "privileged-test" is forbidden:
violates PodSecurity "restricted:latest": privileged containers are not allowed
```

This means:
	•	The Pod never got created.
	•	The API server rejected the request.
	•	The node never started this container.
	•	PSA is working as intended and enforcing restricted rules.

This is the core of today’s lesson:

Even in a small lab cluster, you can make the platform enforce strong security at the pod level.

⸻

## 12. Now let's try the Same Pod in a Non-Restricted Namespace

To understand scope, try this in the default namespace:

```bash
kubectl apply -f privileged-pod.yaml -n default
```

Depending on how your cluster is configured, this may:
	•	Be allowed, or
	•	Be subject to a different PSA policy, or
	•	Still be blocked if GKE set defaults.

Key takeaway:
	•	PSA is namespace-scoped.
	•	The platform team can enforce different policies per namespace:
	•	dev, staging, prod
	•	different teams or squads
	•	For maximum safety, sensitive namespaces (e.g., prod) should almost always be restricted.

---

Congratulations, Today you have learned below topics:
	•	✅ Created a GKE cluster on GCP with:
	•	Custom VPC & subnet
	•	Shielded nodes
	•	2 worker nodes (e2-micro)
	•	Workload Identity enabled
	•	✅ Understood pod-level vs node-level security
	•	✅ Configured a namespace with PSA restricted enforcement
	•	✅ Confirmed that a privileged pod is rejected by PSA
	•	✅ Learned where Seccomp, AppArmor, and PSP fit conceptually
