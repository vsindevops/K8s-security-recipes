# Day 2 — Kubernetes Security & Platform Hardening  
## Seccomp Profiles & Runtime Restrictions on GKE

This document is a **detailed, step-by-step teaching guide** for Day 2 of the Kubernetes Security & Platform Hardening series.

On Day 1, we learned:

- How to create a secure GKE cluster
- What Pod Security Admission (PSA) is
- How `restricted` mode blocks privileged pods

On Day 2, we go deeper into **runtime security** using **seccomp** and see how it works together with PSA.

---

## 1. What You Will Learn Today

By the end of this lab, you will:

- Understand **what seccomp is** in plain language  
- See **how containers use system calls** and why we care about them  
- Learn how Kubernetes exposes seccomp through `securityContext.seccompProfile`  
- Understand the three seccomp modes:
  - `RuntimeDefault`
  - `Unconfined`
  - `Localhost`
- Know exactly **what the `restricted` Pod Security profile expects**:
  - non-root users
  - no privilege escalation
  - dropping all capabilities
  - safe seccomp profile
- Deploy a **secure pod** that passes all `restricted` checks
- Deploy an **insecure pod** that is **rejected by PSA** because of bad seccomp config
- See how all of this fits into real-world platform engineering

This is written like a teacher’s lesson, not just a list of commands.

---

## 2. Concept: How Containers Talk to the Kernel (Syscalls & Seccomp)

Before running any command, it’s important to understand what we are protecting.

### 2.1 System Calls (Syscalls)

When a process inside a container wants to:

- open a file
- create a network connection
- allocate memory
- change permissions

it does not access hardware directly.  
Instead, it asks the **Linux kernel** to do it on its behalf using **system calls** (syscalls).

Examples of syscalls:

- `open`
- `read`
- `write`
- `chmod`
- `mount`
- `ptrace`
- `clone`

If an attacker compromises a container, they may try to use dangerous syscalls (like `ptrace` or `mount`) to escape the container or exploit the kernel.

### 2.2 What Is Seccomp?

**Seccomp** (Secure Computing Mode) is a Linux feature that allows us to **filter** which syscalls a process is allowed to use.

We can say things like:

> *“This container is allowed to do normal I/O calls, but it is **not** allowed to call `ptrace`, `mount`, or other risky syscalls.”*

So even if an attacker gets a shell inside the container, many harmful actions are simply blocked by the kernel.

### 2.3 Seccomp in Kubernetes

Kubernetes lets us attach seccomp policy to a **pod** or **container** using `securityContext.seccompProfile`.

There are three main types:

- `RuntimeDefault`  
  - Use the container runtime’s default seccomp profile  
  - This is usually a safe, well-maintained deny list of dangerous syscalls  
- `Unconfined`  
  - **Disable seccomp**  
  - No syscall filtering at all → not suitable for `restricted` environments  
- `Localhost`  
  - Use a custom seccomp JSON file stored on the **node** filesystem  
  - More advanced use-case (we will only mention it conceptually today)

We will focus on **RuntimeDefault** (good) vs **Unconfined** (bad) and see how Pod Security Admission reacts.

---

## 3. Prerequisites

You need:

- A Google Cloud project with **billing enabled**
- Google Cloud SDK (`gcloud`) installed
- `kubectl` installed
- A GKE **Standard** cluster (we reuse the Day 1 pattern)

If you still have your Day 1 cluster and `kubectl get nodes` works, you can **skip Section 4** and go to Section 5.

---

## 4. (Optional) Create or Recreate the GKE Cluster

This section is to recreate the environment from scratch, like a real platform engineer.

### 4.1 Authenticate & Set the Project

Explanation:

- `gcloud auth login` → log into GCP with your Google account  
- `gcloud config set project` → tell gcloud which project to use for all subsequent commands  

Commands:

    gcloud auth login
    gcloud config set project <PROJECT_ID>

Replace `<PROJECT_ID>` with your actual GCP project ID.

### 4.2 Enable Required APIs

Explanation:

- Kubernetes on GCP is provided by GKE → requires `container.googleapis.com`
- Nodes are Compute Engine VMs → requires `compute.googleapis.com`

Commands:

    gcloud services enable container.googleapis.com
    gcloud services enable compute.googleapis.com

### 4.3 Create a Dedicated VPC and Subnet

Explanation:

- We create a **separate VPC** instead of using the default one  
- This is a common **security best practice** for production environments  
- You now control networking for the cluster more explicitly  

Commands:

```bash
    gcloud compute networks create k8s-secure-vpc --subnet-mode=custom

    gcloud compute networks subnets create k8s-subnet \
      --network=k8s-secure-vpc \
      --region=asia-south1 \
      --range=10.10.0.0/24
```

### 4.4 Create the GKE Cluster

We will:

- Use 2 small nodes (`e2-micro`)
- Attach them to our custom VPC
- Enable shielded nodes (extra OS hardening)
- Enable Workload Identity (better IAM model)

Variables:

```bash
    REGION=asia-south1
    ZONE=asia-south1-a
    CLUSTER_NAME=k8s-security-lab
    PROJECT_ID=<PROJECT_ID>
```

Cluster creation command:
    
```bash
    gcloud container clusters create $CLUSTER_NAME \
      --zone $ZONE \
      --cluster-version latest \
      --machine-type e2-micro \
      --num-nodes 2 \
      --network=k8s-secure-vpc \
      --subnetwork=k8s-subnet \
      --enable-shielded-nodes \
      --workload-pool="$PROJECT_ID.svc.id.goog"
```

**What these flags give you:**

- A small but functional learning cluster  
- Good security defaults (shielded nodes, identity via Workload Identity)  
- Environment that behaves similarly to real production clusters  

### 4.5 Configure kubectl

We now connect your local `kubectl` to this cluster.

    gcloud container clusters get-credentials $CLUSTER_NAME --zone $ZONE
    kubectl get nodes

You should see 2 nodes in `Ready` state.

---

## 5. Pod Security Admission (PSA) Refresher

On Day 1 you learned:

- PSA validates **pod specs** before they are created.
- It enforces one of three standard profiles:
  - `privileged`
  - `baseline`
  - `restricted` ← our focus
- PSA is enabled at the **namespace level** via labels.

Today we will use PSA again, in `restricted` mode, together with seccomp.

---

## 6. Create a `restricted` Namespace for Seccomp Labs

We don’t want to test on random namespaces. Instead, we isolate work into `seccomp-lab`.

### 6.1 Create Namespace

    kubectl create namespace seccomp-lab

### 6.2 Label Namespace for PSA Restricted

These labels tell Kubernetes:

> “In this namespace, enforce the **restricted** Pod Security profile.”
```bash
    kubectl label namespace seccomp-lab \
      pod-security.kubernetes.io/enforce=restricted \
      pod-security.kubernetes.io/enforce-version=latest
```

Check:

```bash
    kubectl get ns seccomp-lab --show-labels
```

You should see `pod-security.kubernetes.io/enforce=restricted` among the labels.

From now on, **every pod in this namespace** must follow `restricted` rules.

---

## 7. What Does `restricted` Actually Require?

This is important, because it explains why you saw errors like:

- “must drop capabilities”
- “must not run as root”

Under `restricted:latest`, Kubernetes expects:

1. **Non-root user**
   - Either `runAsNonRoot: true` + image that does not default to root  
   - Or explicitly set `runAsUser` to a non-root UID (e.g. `1000`)

2. **No privilege escalation**
   - `allowPrivilegeEscalation: false`

3. **Drop all capabilities**
   - `capabilities.drop` must include `"ALL"`

4. **Safe seccomp**
   - `seccompProfile.type` must be `RuntimeDefault` or `Localhost`  
   - `Unconfined` is **not allowed**

5. **No privileged container**
   - `privileged: true` is rejected

In Day 1, you saw this with `privileged: true`.  
In Day 2, you see it with seccomp + capabilities + non-root.

---

## 8. Lab 1 — Secure Pod Using `RuntimeDefault` (Passes Restricted)

Now we create a pod that fully respects the `restricted` profile.

### 8.1 Design of the Pod

We will:

- Use the `ubuntu` image
- Run a simple `sleep 3600` so the pod stays alive
- Explicitly configure:
  - `seccompProfile: RuntimeDefault`  
  - `runAsUser: 1000`, `runAsGroup: 1000` (non-root)  
  - `runAsNonRoot: true`  
  - `allowPrivilegeEscalation: false`  
  - `capabilities.drop: ["ALL"]`

This is exactly what a secure, restricted pod should look like.

### 8.2 YAML Manifest

File: `seccomp-runtime-default.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-runtime-default
  namespace: seccomp-lab
spec:
  containers:
    - name: ubuntu
      image: ubuntu
      command: ["sleep", "3600"]
      securityContext:
        seccompProfile:
          type: RuntimeDefault
        runAsUser: 1000
        runAsGroup: 1000
        runAsNonRoot: true
        allowPrivilegeEscalation: false
        capabilities:
          drop:
          - ALL
```

### 8.3 Apply the Manifest

    kubectl apply -f seccomp-runtime-default.yaml
    kubectl get pods -n seccomp-lab

Expected:

- `seccomp-runtime-default` is in `Running` state.

If you ever see `CreateContainerConfigError`, use:

    kubectl describe pod seccomp-runtime-default -n seccomp-lab

On images like `ubuntu`, you must specify non-root user (`runAsUser`), otherwise it assumes root by default and violates `runAsNonRoot`.

Here we explicitly fix that.

### 8.4 What This Pod Tells Kubernetes

- “I will only use the **runtime’s default seccomp profile**.”
- “I will run as **user 1000**, not root.”
- “I will **not** try to escalate privileges.”
- “I’m dropping **all Linux capabilities**.”

PSA checks all these and says:

> “This pod is acceptable for `restricted` mode.”

---

## 9. Lab 2 — Insecure Pod Using `Unconfined` (Blocked by PSA)

Now we intentionally misconfigure seccomp to see PSA protect us.

### 9.1 Idea

We will:

- Keep all good settings:
  - non-root
  - no privilege escalation
  - drop all capabilities
- But change **only one thing**: `seccompProfile.type: Unconfined`

This is like telling Kubernetes:

> “Allow this container to call any syscall it wants.”

In a `restricted` namespace, this is not acceptable.

### 9.2 YAML Manifest

File: `seccomp-unconfined.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: seccomp-unconfined
  namespace: seccomp-lab
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sleep", "3600"]
    securityContext:
      seccompProfile:
        type: Unconfined
      runAsUser: 1000
      runAsGroup: 1000
      runAsNonRoot: true
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
```

### 9.3 Apply and Observe

    kubectl apply -f seccomp-unconfined.yaml

You should see an error similar to:

```text
Error from server (Forbidden): error when creating “seccomp-unconfined.yaml”:
pods “seccomp-unconfined” is forbidden:
violates PodSecurity “restricted:latest”:
seccompProfile.type “Unconfined” is not allowed
```

This is the **exact behavior** we want:

- The pod never gets created.
- The node never runs it.
- PSA acts as a security gate at admission time.

---

## 10. (Optional) Try the Unconfined Pod in Namespace Without PSA

To see how **namespace-scoped** PSA is, you can create another namespace:

    kubectl create namespace seccomp-no-psa

Now, apply the same `seccomp-unconfined.yaml` there:

    kubectl apply -f seccomp-unconfined.yaml -n seccomp-no-psa

Possible outcomes (depends on your GKE defaults):

- It might be **allowed** and the pod runs.
- Or your cluster might have additional policies.

The main lesson:

> PSA enforcement depends on the **labels on the namespace**.  
> Different namespaces can have different security levels.

This is how platform teams can give stricter rules to `prod` vs `dev` or to different squads.

---

## 11. (Optional) Inspect the Pod and Events

Even if `kubectl exec` is not working (due to GKE tunnel / agent issues), you can still inspect the pod.

### 11.1 View Full Pod Spec

For the good pod:

    kubectl get pod seccomp-runtime-default -n seccomp-lab -o yaml

Look under `securityContext` and verify:

- `seccompProfile.type: RuntimeDefault`
- `runAsUser: 1000`, `runAsGroup: 1000`
- `runAsNonRoot: true`
- `allowPrivilegeEscalation: false`
- `capabilities.drop: ["ALL"]`

This is how a **restricted** pod should look in a modern cluster.

### 11.2 View Events For Debugging

For any pod:

    kubectl describe pod seccomp-runtime-default -n seccomp-lab

At the bottom, under **Events**, you see all scheduling and admission events.  
For rejected pods (like `seccomp-unconfined`), PSA errors appear in `kubectl apply` output itself.

---

## 12. Custom Seccomp Profiles (`Localhost`) — Conceptual Overview

So far we used:

- `RuntimeDefault` → runtime’s default seccomp profile
- `Unconfined` → no seccomp (blocked in restricted namespaces)

The third type is:

- `Localhost` → use a custom profile file from the node’s filesystem

Example (concept only):

```yaml
securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: my-profiles/custom-seccomp.json
```

This tells the kubelet:

> “Load this JSON seccomp profile from the node and use it for this container.”

On GKE Standard, this is an advanced topic, usually involving:

- Baking profiles into a custom node image, or
- Running a privileged DaemonSet that writes profile files into `/var/lib/kubelet/seccomp/profiles/`.

We won’t implement it in this beginner lab, but you now know where it fits.

---

## 13. Mental Model: PSA + Seccomp + Node Security

Think of security in three layers:

1. **Pod Security Admission (PSA) — Policy at the API level**
   - Validates pod specs before they exist
   - Enforces:
     - no privileged containers
     - no `Unconfined` seccomp
     - non-root users
     - `allowPrivilegeEscalation: false`
     - drop all capabilities

2. **Seccomp — Runtime syscall filter**
   - Limits which syscalls containers can use
   - `RuntimeDefault` is usually strong enough for most workloads
   - Even if app is compromised, certain kernel actions are blocked

3. **Node-level hardening (OS & infrastructure)**
   - Shielded nodes
   - OS patches and minimal packages
   - Secure kubelet configuration
   - IAM roles for nodes
   - Firewall rules / network policies

Combining these gives a **defense-in-depth** approach:

- PSA stops many bad pods from ever running.
- Seccomp restricts what running pods can do.
- Node-level controls limit damage if something still breaks through.

Day 1 + Day 2 together already give you a surprisingly strong security baseline.

---

## 14. Cleanup

You have two options: keep the cluster or tear it all down.

### 14.1 Keep Cluster, Remove Only Lab Resources

Useful if you’ll use the same cluster tomorrow.

    kubectl delete namespace seccomp-lab --ignore-not-found=true
    

(Remember to also delete any Day 1 namespaces if you don’t need them.)

### 14.2 Full Cleanup (Cluster + Network)

If you want to stop all costs:

    CLUSTER_NAME=k8s-security-lab
    ZONE=asia-south1-a
    REGION=asia-south1
    VPC_NAME=k8s-secure-vpc

    kubectl delete namespace seccomp-lab --ignore-not-found=true
    

    gcloud container clusters delete $CLUSTER_NAME --zone $ZONE --quiet

    gcloud compute networks subnets delete k8s-subnet --region=$REGION --quiet
    gcloud compute networks delete $VPC_NAME --quiet

Deleting the cluster removes the control plane and worker nodes, which are the main cost components.

---

## 15. Summary of Day 2

Today you:

- Learned **what seccomp is** and why syscall filtering matters  
- Understood how Kubernetes exposes seccomp via `securityContext.seccompProfile`  
- Saw what the `restricted` Pod Security standard actually demands:
  - non-root users
  - no privilege escalation
  - drop all capabilities
  - safe seccomp profile (no `Unconfined`)
- Created a **secure pod** that:
  - runs as non-root (UID 1000)
  - drops all capabilities
  - uses `RuntimeDefault` seccomp
  - passes PSA restricted validation
- Created an **insecure pod** with `seccompProfile.type: Unconfined` and watched PSA **block it**
- Built a mental model of how PSA + seccomp + node hardening work together

You are now comfortable with **pod-level runtime restrictions**, which is a major part of real-world Kubernetes security and platform engineering.
