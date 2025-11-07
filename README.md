# How-to-Deploy-Rook-Ceph-on-Kubernetes---The-Ultimate-Step-by-Step-Guide-for-Cloud-Native-Storage
How to Deploy Rook Ceph on Kubernetes - The Ultimate Step-by-Step Guide for Cloud-Native Storage
# 🚀 How to Deploy Rook Ceph on Kubernetes — The Ultimate Step-by-Step Guide for Cloud-Native Storage

Kubernetes makes deploying applications easy.  
But what about **persistent storage** — the data that actually matters?  
That’s where things get tricky.  

If you’ve ever struggled to make databases or stateful workloads reliable on Kubernetes, you’ve probably heard of **Ceph** — a powerful, distributed storage system that can handle block, file, and object storage at scale.  

But deploying Ceph manually is complex.  
That’s where **Rook** comes in — a Kubernetes operator that automates the entire Ceph lifecycle: deployment, scaling, healing, and upgrades — all using native Kubernetes resources.  

In this guide, I’ll walk you through deploying **Rook + Ceph** from scratch and show you how to verify and manage it like a pro.  

---

## 🧠 What You’ll Learn
By the end of this guide, you’ll know how to:
- Deploy the **Rook Operator** and **Ceph Cluster**
- Verify Ceph cluster health from inside Kubernetes  
- Choose and configure **one** storage type: RBD, CephFS, or Object Store  
- Apply production-grade best practices  

---

## ⚙️ Prerequisites
You’ll need:
- A running **Kubernetes cluster** (3+ nodes recommended)  
- `kubectl` access  
- Raw disks or block devices attached to your nodes (for OSDs)  
- Basic knowledge of YAML and Kubernetes resources  

> 💡 **Pro tip:** Use Minikube or KIND for learning, and real multi-node clusters with physical disks for production.

---

## 🏗️ Step 1: Install Rook Operator and CRDs

Rook extends Kubernetes with **Custom Resource Definitions (CRDs)** representing Ceph components like clusters, pools, filesystems, and gateways.  

We’ll install those first, then deploy the Rook Operator.

### 🧩 1. Apply CRDs
```bash
kubectl create -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/crds.yaml
```
Registers Ceph-related APIs with Kubernetes.

### 🧩 2. Apply Common Resources
```bash
kubectl create -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/common.yaml
```
Creates the `rook-ceph` namespace and RBAC roles required by the operator.

### 🧠 3. Deploy the Operator
```bash
kubectl create -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/operator.yaml
```

Check the pod:
```bash
kubectl -n rook-ceph get pods
```

```
NAME                                  READY   STATUS    RESTARTS   AGE
rook-ceph-operator-6b5d6bb79f-h8x6j   1/1     Running   0          2m
```

The operator now watches for Ceph CRDs and orchestrates deployment.

---

## 🧱 Step 2: Create the Ceph Cluster

The `CephCluster` resource defines how Rook should build and configure Ceph — monitors, managers, and OSDs.

```yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v18
  dataDirHostPath: /var/lib/rook
  mon:
    count: 3
    allowMultiplePerNode: false
  mgr:
    modules:
      - name: pg_autoscaler
        enabled: true
  dashboard:
    enabled: true
  storage:
    useAllNodes: false
```

> ⚠️ **Important:** Avoid `useAllDevices: true` in production — it will claim all disks.

Apply:
```bash
kubectl apply -f ceph-cluster.yaml
```

Check pods:
```bash
kubectl -n rook-ceph get pods
```

```
NAME                                 READY   STATUS    RESTARTS   AGE
rook-ceph-mon-a-6f6b5d79f7-pt7qx     1/1     Running   0          3m
rook-ceph-mgr-a-5d79bcbf7c-8b6dh     1/1     Running   0          2m
rook-ceph-osd-0-7cc94b8b9c-hjs2t     1/1     Running   0          1m
rook-ceph-osd-1-7cc94b8b9c-mtlxm     1/1     Running   0          1m
```

Your Ceph Cluster is now running — let’s verify it.

---

## 🧰 Step 3: Deploy the Rook Toolbox (and Check Cluster Health)

The **Rook Toolbox** is an administrative pod that includes the `ceph` CLI.

Deploy:
```bash
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/deploy/examples/toolbox.yaml
```

When it’s ready:
```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
```

Check cluster health:
```bash
ceph status
```

```
cluster:
  id: a1b2c3d4
  health: HEALTH_OK
  services:
    mon: 3 daemons, quorum a,b,c
    mgr: active: a
    osd: 3 osds: 3 up, 3 in
```

✅ **Your Ceph cluster is healthy and operational!**

---

## 🧭 Step 4: Choose Your Storage Type

Once healthy, choose **one** storage interface to deploy (per cluster).

| Storage Type | Description | Ideal Use Case |
|--------------|--------------|----------------|
| 🧊 **RBD (Block Storage)** | Dynamic PVCs backed by RBD volumes | Databases, StatefulSets |
| 📁 **CephFS (Filesystem)** | Shared POSIX-compliant filesystem | Shared app data, Prometheus |
| ☁️ **Object Store (RGW)** | S3-compatible object storage | Backups, S3 API apps |

> ⚠️ **Note:** Deploy only one storage type per cluster based on your application needs.

---

## 🧩 Step 5: (Optional) Deploy Multiple Ceph Clusters

For separate environments (e.g., production and staging), apply a second cluster manifest:

```bash
kubectl apply -f cluster-second.yaml
```

Guidelines:
- Different namespace (e.g., `rook-ceph-test`)  
- Unique cluster name and FSID  
- Separate node and disk sets  

---

## 🧭 Rook Ceph Architecture Overview  

Here’s how **Rook** and **Ceph** integrate within a Kubernetes cluster 👇  

```
┌────────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                       │
│                                                            │
│   ┌────────────────────────────────────────────────────┐   │
│   │                   Rook Operator                    │   │
│   │   (Manages Ceph CRDs and orchestrates daemons)     │   │
│   └───────────────────────┬────────────────────────────┘   │
│                           │                                │
│           ┌───────────────▼────────────────┐               │
│           │           Ceph Cluster         │               │
│           │     (MON / MGR / OSD Pods)     │               │
│           └───────────────┬────────────────┘               │
│                           │                                │
│               Choose One Storage Interface                 │
│       ┌──────────────────┼───────────────────┐             │
│       │                  │                   │             │
│   ┌───▼───┐          ┌───▼────┐          ┌───▼────┐        │
│   │  RBD  │          │ CephFS │          │  RGW   │        │
│   │ Block │          │  File  │          │ Object │        │
│   │ Store │          │ System │          │  Store │        │
│   └───────┘          └────────┘          └────────┘        │
└────────────────────────────────────────────────────────────┘
```

🧩 **In short:**  
- **Rook Operator** manages Ceph lifecycle via CRDs.  
- **Ceph Cluster** runs daemons (MON, MGR, OSD).  
- You can expose one interface — **RBD**, **CephFS**, or **RGW** — depending on your needs.  

---

## ⚡ Performance & Best Practices

- **Separate roles:** Dedicate nodes to OSDs via taints/tolerations.  
- **Avoid `useAllDevices:true`** in production.  
- **Enable monitoring:** Expose Prometheus metrics to Grafana.  
- **Replication:** 3× replicas for critical data, erasure coding for bulk data.  
- **Routine checks:** Run `ceph health detail` regularly.  

---

## 🎯 Wrapping Up

You’ve built a **self-healing, scalable storage system** inside Kubernetes — powered by Rook and Ceph.  

Rook simplifies Ceph operations; Ceph delivers enterprise-grade resilience and performance.  
Together they form a foundation for stateful, cloud-native workloads.  

Next steps:
- Enable the **Ceph Dashboard**  
- Connect **Prometheus + Grafana**  
- Explore **CephFS or RGW**  
- Learn backup and recovery workflows  

---

## ✨ Final Thoughts

Kubernetes isn’t just for stateless apps anymore — with Rook Ceph, it becomes a true **data platform**.  

If you found this helpful, follow me for future posts in my **Rook Ceph Deep Dive Series**, where I’ll cover:  
- 🧰 Ceph Dashboard setup and metrics  
- 💾 CephFS and RGW performance tuning  
- 🛡️ Recovery from OSD or MON failures  

---

### 🧑‍💻 About the Author  

Hi there! I’m **Dhinakaran J**, a **Site Reliability Engineer at the National Payments Corporation of India (NPCI)** — where I help ensure the **availability, scalability, and reliability of India’s critical payment infrastructure**.  

I work extensively with **Kubernetes**, **Rook Ceph**, **Argo CD**, **Prometheus**, **Docker**, **RKE**, **Jenkins**, **MinIO**, and **Grafana**, building automated and observable cloud-native systems that keep payment services running smoothly at scale.  

My passion lies in **infrastructure reliability**, **monitoring**, and **distributed storage systems** — especially leveraging **Rook Ceph** to deliver highly resilient and transparent platforms in Kubernetes.  

When I’m not optimizing clusters or debugging production workloads, I enjoy exploring open-source technologies, mentoring engineers, and writing hands-on DevOps guides to make complex systems simple for everyone.  

> 💬 Follow me on Medium for practical guides on **Kubernetes**, **Ceph**, **Argo CD**, and **Observability** — drawn from real-world SRE experience.  

---

**Short footer bio:**  
> *Dhinakaran J is a Site Reliability Engineer at NPCI specializing in Kubernetes, Ceph, Argo CD, Prometheus, and cloud-native reliability.*  

---

### 🏷️ Suggested Tags for Medium  
`#Kubernetes`  `#DevOps`  `#Ceph`  `#Rook`  `#ArgoCD`  `#Prometheus`  `#CloudNative`  `#SRE`  `#Storage`

---

📸 **Suggested Image Placeholders for Medium**
1. *Header Image:* Kubernetes + Ceph architecture visual.  
2. *Step 1 Screenshot:* `kubectl get pods -n rook-ceph` showing operator running.  
3. *Step 3 Screenshot:* Ceph dashboard or `ceph status` output.  
4. *Architecture Diagram:* Use the above ASCII version converted to a PNG for a polished look.  
