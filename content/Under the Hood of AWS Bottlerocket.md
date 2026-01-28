---
title: Under the Hood of AWS Bottlerocket - A Filesystem Analysis
tags:
  - AWS
  - Bottlerocket
  - Kubernetes
  - Containerd
  - DevOps
---

Bottlerocket is a Linux-based open-source operating system built by AWS specifically for running containers. It's known for its security (read-only root filesystem) and atomic updates.

Recently, I took a look **inside** a fresh Bottlerocket node to understand how it handles storage and configuration under the hood. Here are my raw observations and notes.

### 1. The Containerd Configuration

The heart of a container OS is its runtime. On Bottlerocket, `containerd` is configured via `/etc/containerd/config.toml` (or similar paths depending on the setup).

Here is what the configuration looks like on a fresh node:

```toml
version = 2
root = "/var/lib/containerd"
state = "/run/containerd"
disabled_plugins = [
    "io.containerd.internal.v1.opt",
    "io.containerd.snapshotter.v1.aufs",
    "io.containerd.snapshotter.v1.devmapper",
    "io.containerd.snapshotter.v1.native",
    "io.containerd.snapshotter.v1.zfs",
]

[grpc]
address = "/run/containerd/containerd.sock"

[plugins."io.containerd.grpc.v1.cri"]
device_ownership_from_security_context = false
enable_selinux = true
sandbox_image = "localhost/kubernetes/pause:0.1.0"

[plugins."io.containerd.grpc.v1.cri".containerd]
default_runtime_name = "shimpei"

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.shimpei]
runtime_type = "io.containerd.runc.v2"
base_runtime_spec = "/etc/containerd/cri-base.json"
```

Notice the specific runtime configuration (`shimpei`) and the explicit disabling of unused snapshotters to keep the footprint small.

### 2. Disk Partitioning & Layout

Bottlerocket uses a unique partition scheme to support its atomic updates. It uses two partition sets (A/B) for the OS image, but separates persistent data.

Running `lsblk` reveals the structure:

```bash
NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
nvme1n1      259:0    0   14G  0 disk
`-nvme1n1p1  259:15   0   14G  0 part /var
                                      /opt
                                      /mnt
                                      /local
nvme0n1      259:1    0    5G  0 disk
|-nvme0n1p3  259:4    0   40M  0 part /boot
|-nvme0n1p12 259:12   0   41M  0 part /var/lib/bottlerocket
```

*   **`nvme0n1`**: The OS disk. It has many small partitions (for the A/B bank system).
*   **`nvme1n1`**: The data disk. This is where `/var`, `/opt`, and `/local` live. This is persistent and separate from the OS updates.

### 3. Storage Usage & Image Pulling

I ran some tests to see how disk usage changes when pulling images.

**Before pulling a backend image:**
The data partition (`/dev/nvme1n1p1` mounted at `/local`) was sitting at **7% usage** (approx 924MB used).

**After pulling a backend image:**
Usage jumped to **38%** (5.2GB used).

```bash
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme1n1p1   14G  5.2G  8.8G  38% /local
```

This confirms that heavy container images live on that separate data partition.

### 4. An Optimization Strategy: Pre-cached Snapshots

Based on this analysis, I sketched out a potential workflow for faster node startup times by pre-baking images into EBS snapshots.

**The Idea:**
Instead of pulling large images every time a node scales up, we can "seed" the data volume.

1.  **Create Volume:** Start with a snapshot of a standard Bottlerocket data volume (empty).
2.  **Mount & Pull:** Attach this volume to a builder instance (like a Jenkins worker).
3.  **Cache Images:** Use `ctr` (or Docker) to pull your heavy backend images directly into this volume.
4.  **Snapshot:** Take a snapshot of this now-populated volume.
5.  **Deploy:** Update Kubernetes manifests or Launch Templates to map new nodes' data volumes to this "pre-warmed" snapshot.

This allows new nodes to come online with the images already "present" on disk, skipping the network pull and speeding up startup significantly.

---

*Notes: Always ensure the role attached to the Bottlerocket instance allows SSM access for debugging (`AmazonSSMManagedInstanceCore`).*
