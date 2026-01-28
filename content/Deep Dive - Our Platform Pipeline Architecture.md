---
title: Instant K8s Scaling - Baking Docker Images into Bottlerocket Volumes
tags:
  - AWS
  - Bottlerocket
  - Kubernetes
  - EKS
  - Scaling
  - DevOps
---

Scaling Kubernetes nodes usually comes with a tax: **startup latency**.

When a new node joins the cluster, it's just an empty shell. Before it can run your application, it has to pull your Docker images from the registry. For our monolithic backend services, these images can be massive (2GB+). This means waiting 3–5 minutes for network I/O before a single pod can start.

We solved this by leveraging the unique architecture of **AWS Bottlerocket OS**.

Instead of pulling images *at runtime*, we bake them into the node's data volume *at build time*. Here is how we engineered an instant-start scaling workflow.

### The Bottlerocket Advantage

[Bottlerocket](https://github.com/bottlerocket-os/bottlerocket) separates its OS partition from its data partition.
*   **OS Partition:** Read-only, updated atomically.
*   **Data Partition (`/local`):** Where persistent data and **container images** live.

This separation allows us to swap out the data partition without touching the OS. We realized we could "pre-warm" this partition with our latest Docker images.

### The Workflow: From Build to Instant Scale

We integrated this baking process directly into our CI/CD pipeline. Here is the streamlined flow:

#### 1. The Build Phase 🏗️
Standard CI stuff. We build our application code and push the Docker image to ECR.
*   **Output:** `ecr.io/my-app:v1.2.3`

#### 2. The Baking Phase 🍪
This is where the magic happens. We run a specialized pipeline job that acts as a "bakery":

1.  **Spin Up Volume:** We create a temporary EBS volume from a "clean" Bottlerocket data snapshot.
2.  **Mount:** We attach this volume to a builder instance (e.g., a Jenkins worker).
3.  **Pull Images:** We use `ctr` (containerd CLI) to pull the new image `v1.2.3` *directly* into the mounted volume's filesystem.
    ```bash
    ctr -n k8s.io images pull ecr.io/my-app:v1.2.3 --root /mnt/bottlerocket-data
    ```
4.  **Snapshot:** We unmount the volume and take a new **EBS Snapshot**.
    *   **Result:** Snapshot ID `snap-0abc123...` (contains the OS data structure + the cached image).

#### 3. The Deploy Phase 🚀
We don't just update the Kubernetes Deployment; we update the infrastructure itself.

1.  **Update Launch Template:** Our pipeline updates the AWS Launch Template for our EKS Node Group. We point the data volume mapping to our new snapshot `snap-0abc123...`.
2.  **Scale Out:** When Karpenter or Cluster Autoscaler triggers a scale-up, AWS launches a new EC2 instance.
3.  **Instant Start:** The node boots up. It mounts the data volume. `containerd` looks at its storage and sees... **the image is already there.**
4.  **Zero Pull:** The pod starts immediately.

### Why This Matters

*   **Startup Time:** Reduced from ~5 minutes to **<45 seconds**.
*   **Cost:** Less data transfer from ECR (images are pulled once during baking, not 100 times by 100 nodes).
*   **Reliability:** No more "ImagePullBackOff" errors during critical scaling events due to registry throttling.

By treating the node's storage as an immutable artifact—just like the Docker image itself—we achieved true "instant" elasticity on EKS.

---
*Curious about the filesystem details? Check out my analysis: [[Under the Hood of AWS Bottlerocket]].*
