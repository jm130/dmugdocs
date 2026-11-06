# Heriot-Watt Student Cluster User Guide

Welcome to the Heriot-Watt Student Cluster! This guide will help you get started with high-performance computing (HPC) on our cluster, from your first login to running complex simulations efficiently. Whether you're new to HPC or an experienced user, you'll find everything you need to make the most of this resource.

---

## 0. The Workflow at a Glance

To ensure the cluster remains fast for everyone, we follow a specific data lifecycle. **We strongly discouraged to run simulations directly in your `/home` directory.**

```text
[ Your Laptop ] 
      |
      | (1) scp/rsync (Upload)
      v
[ Cluster: /home/user/ ]  <-- (Safe storage, slow for compute)
      |
      | (2) sbatch (Automated Copy)
      v
[ Cluster Node: /tmp/job_id/ ] <-- (High-speed local disk, used for compute)
      |
      | (3) Run Simulation
      v
[ Cluster Node: /tmp/job_id/ ]
      |
      | (4) sbatch (Automated Copy-back)
      v
[ Cluster: /home/user/results/ ] <-- (Results saved safely)
      |
      | (5) scp/rsync (Download)
      v
[ Your Laptop ]
```

---

## 1. Getting Started (Access)

### On-Campus Access
If you are on the university network, you can access the cluster directly via SSH:
```bash
ssh <your_username>@dmug.hw.ac.uk 
```

### Remote Access (Off-Campus)

If you are accessing the cluster from home or another external network, you **must** connect through our **SSH Gateway** first.

**Important**: Before using the SSH gateway, you need to:
1. Request access via **[ISHelp@hw.ac.uk](mailto:ISHelp@hw.ac.uk)**

Once access is granted:

1. Connect to the gateway (note the non-standard port 44788):
   ```bash
   ssh -p 44788 <your_username>@sshgw.hw.ac.uk
   ```
2. From the gateway, SSH into the cluster:
   ```bash
   ssh <your_username>@<cluster_hostname>
   ```

### The Login Node
When you first connect, you are on the **Login Node**. 
- **DO NOT** run heavy simulations or data processing directly on the login node.
- The login node is for editing code, managing files, and submitting jobs.
- Heavy tasks will be killed to ensure the system remains responsive for everyone.

---

## 2. Environment Setup (Modules)

We use the `module` system to manage software. This allows you to load different versions of compilers, libraries, and tools without them conflicting.

*   **`module avail`**: Lists all software currently available to you.
*   **`module load <module_name>`**: Loads a specific software package (e.g., `module load scalapack/2.2.3`).
*   **`module list`**: Shows which modules you currently have loaded.
*   **`module purge`**: Removes all currently loaded modules (useful for starting fresh).

### Installing New Software

You have two options for adding new software to your environment:

1. **Local Installation (User Space)**: You can install software in your own user directory without requiring root access. This is the recommended approach for individual tools and dependencies. Simply download and compile software with a prefix pointing to your home directory (e.g., `./configure --prefix=$HOME/local`).

2. **System-Wide Installation**: If you'd like software to be available to all cluster users, please contact **[ISHelp@hw.ac.uk](mailto:ISHelp@hw.ac.uk)** with details of the software and its version. Our team will evaluate and install it system-wide.

---

## 3. Data Management & Performance

### Storage Quotas

While there are currently no strict storage quotas enforced on the cluster, please remember that this is a **shared resource**. Be considerate of other users by:
- Regularly cleaning up completed job outputs you no longer need
- Archiving old projects to external storage
- Avoiding unnecessary duplication of large files

### The Golden Rule: Use Local Disks for Computation

Running simulations directly in your `/home` directory is **inefficient**. The `/home` directory is a network-mounted filesystem, and high-frequency reads/writes can slow down the entire cluster.

**For maximum performance, always run your simulations using the node's local disk.**

### Recommended Data Transfer

**Uploading Data to the Cluster:**

Use `rsync` from your **local machine** to upload data to your cluster home directory:
```bash
# Copy a folder from your laptop to the cluster (on-campus)
rsync -avz ./my_project <your_username>@dmug.hw.ac.uk:/home/<your_username>/
```

**Downloading Data from the Cluster:**

To retrieve your results, use `scp` or `rsync` to download files. When off-campus, connect through the SSH gateway first:

```bash
# Download from cluster (on-campus direct access)
rsync -avz <your_username>@dmug.hw.ac.uk:/home/<your_username>/results/ ./local_results/

# Download via SSH gateway (off-campus) - use ProxyCommand or SSH config
# Option 1: Using SSH config (~/.ssh/config)
Host cluster
    HostName dmug.hw.ac.uk
    User <your_username>
    ProxyCommand ssh -p 44788 %r@sshgw.hw.ac.uk -W %h:%p

# Then simply:
rsync -avz cluster:/home/<your_username>/results/ ./local_results/
```

For easier off-campus access, we recommend setting up an SSH config file at `~/.ssh/config` with the gateway proxy settings.

---

## 4. Running Jobs (Slurm)

We use **Slurm** to manage job scheduling and resource allocation.

### Partitions (Queues)
1.  **`nodes`**: For standard CPU-intensive tasks.
2.  **`gpu`**: For tasks requiring GPU acceleration.

---

## 4.1 GPU Job Resource Policy

GPU nodes are shared between jobs to maximise use of the available GPUs, CPUs and memory. Each GPU node has **4 GPUs**, **28 CPU cores** and approximately **63 GB RAM**.

Jobs submitted to the `gpu` partition must explicitly request at least one GPU using `--gres=gpu:N`.

### Resource Limits

For each GPU requested, a job may request up to:
- **7 CPU cores** per GPU
- **32 GiB RAM** per GPU

| GPUs Requested | Max CPUs | Max RAM |
|----------------|----------|---------|
| 1 GPU | 7 | 32 GiB |
| 2 GPUs | 14 | 64 GiB* |
| 3 GPUs | 21 | 96 GiB* |
| 4 GPUs | 28 | 128 GiB* |

*Physical node memory (63 GB) remains the ultimate limit.

These limits prevent a job using a single GPU from reserving all of the node's CPU or memory, which would make the remaining GPUs unavailable to other users.

### Examples

**A typical GPU job might request:**
```bash
srun -p gpu \
  --gres=gpu:1 \
  --cpus-per-task=4 \
  --mem=16G \
  hostname
```

**Maximum allowed for a one-GPU job:**
```bash
--gres=gpu:1 --cpus-per-task=7 --mem=32G
```

**These requests will be REJECTED:**
```bash
# Too many CPUs for 1 GPU
--gres=gpu:1 --cpus-per-task=8

# Too much memory for 1 GPU
--gres=gpu:1 --mem=40G
```

### Using `--mem-per-cpu`

Be careful when using `--mem-per-cpu`, as this value is multiplied by the number of CPUs requested.

**This is allowed (4 CPUs × 8 GiB = 32 GiB):**
```bash
--gres=gpu:1 --cpus-per-task=4 --mem-per-cpu=8G
```

**This will be REJECTED (7 CPUs × 8 GiB = 56 GiB, exceeds 32 GiB limit):**
```bash
--gres=gpu:1 --cpus-per-task=7 --mem-per-cpu=8G
```

### Why We Do This

The GPU nodes are configured so that several jobs can run simultaneously on the same node while using different GPUs, CPU cores and memory.

Accurate resource requests allow Slurm to make the unused resources available to other jobs rather than leaving GPUs idle simply because another job has unnecessarily reserved all of the CPU cores or memory.

**Please request only the resources your workload needs.** This improves GPU utilisation, reduces unnecessary queueing and makes the cluster more readily available to everyone.

---

### Batch Jobs (`sbatch`)
For most work, you will create a shell script and submit it using `sbatch`. The templates below include **safety features**: they use `set -e` to stop on errors and `trap` to ensure local temporary files are cleaned up even if the job fails.

#### **Template 1: Standard CPU Job**
Save this as `cpu_job.sh`:
```bash
#!/bin/bash
#SBATCH --job-name=cpu_sim          # Job name
#SBATCH --partition=nodes           # Use the 'nodes' partition
#SBATCH --cpus-per-task=4           # Number of CPU cores
#SBATCH --mem=8G                    # Total memory
#SBATCH --time=01:00:00             # Max runtime (HH:MM:SS)
#SBATCH --output=res_%j.out         # Standard output file (%j = job ID)

# --- SAFETY SETTINGS ---
set -e              # Exit immediately if a command fails
trap 'rm -rf "$LOCAL_DIR"; exit' EXIT  # Ensure cleanup happens on exit/error

# 1. Load necessary modules
module purge
module load scalapack/2.2.3

# 2. Setup local workspace (Performance!)
LOCAL_DIR="/tmp/job_$SLURM_JOB_ID"
mkdir -p "$LOCAL_DIR"
# Copy everything from the directory where you ran 'sbatch' to the local disk
rsync -a "$SLURM_SUBMIT_DIR/" "$LOCAL_DIR/"
cd "$LOCAL_DIR"

# 3. Run the simulation
python my_simulation.py

# 4. Move results back to home
mkdir -p "$SLURM_SUBMIT_DIR/results"
rsync -a "$LOCAL_DIR/output/" "$SLURM_SUBMIT_DIR/results/"

# 5. Cleanup (Handled by 'trap' above)
```
**Submit it with:** `sbatch cpu_job.sh`

#### **Template 2: GPU Job**

**Important**: See the [GPU Job Resource Policy](#41-gpu-job-resource-policy) section above for resource limits. Each GPU allows up to 7 CPUs and 32 GiB RAM.

Save this as `gpu_job.sh`:
```bash
#!/bin/bash
#SBATCH --job-name=gpu_sim
#SBATCH --partition=gpu             # Use the 'gpu' partition
#SBATCH --gres=gpu:1                # Request 1 GPU
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --time=02:00:00
#SBATCH --output=gpu_res_%j.out

# --- SAFETY SETTINGS ---
set -e
trap 'rm -rf "$LOCAL_DIR"; exit' EXIT

# 1. Load modules (including CUDA)
module purge
module load python/3.10 cuda/12.1

# 2. Setup local workspace
LOCAL_DIR="/tmp/gpu_job_$SLURM_JOB_ID"
mkdir -p "$LOCAL_DIR"
rsync -a "$SLURM_SUBMIT_DIR/" "$LOCAL_DIR/"
cd "$LOCAL_DIR"

# 3. Run the simulation
python gpu_simulation.py

# 4. Move results back
mkdir -p "$SLURM_SUBMIT_DIR/results"
rsync -a "$LOCAL_DIR/output/" "$SLURM_SUBMIT_DIR/results/"
```
**Submit it with:** `sbatch gpu_job.sh`

### Interactive Sessions (`srun`)
If you need to debug code or test if a module loads correctly, use `srun` for a "live" session:
```bash
# Request an interactive CPU session
srun --partition=nodes --cpus-per-task=2 --mem=4G --pty bash
```

---

## 5. Monitoring & Troubleshooting

### Common Commands
*   **`squeue`**: Check the status of your jobs.
*   **`sinfo`**: Check which nodes/partitions are available.
*   **`scancel <job_id>`**: Stop a job that is running too long or is broken.
*   **`list_nodes`**: View detailed information about all compute nodes (custom command).

### Cluster Topology

The cluster consists of CPU and GPU compute nodes. The node configuration may change regularly as resources are added or maintained. Use the `list_nodes` command to see the current status:

```bash
list_nodes
```

This will display output similar to:

```
NODELIST                                STATE       REASON                        CPUS S:C:T   MEMORY    AVAIL_FEATURES                          GRES                NODE_ADDR           TIMESTAMP
node01                                  idle        none                          32   2:8:2   191000    (null)                      (null)              node01              Unknown
node02                                  idle        none                          32   2:8:2   191000    (null)                      (null)              node02              Unknown
node03                                  idle        none                          32   2:8:2   191000    (null)                      (null)              node03              Unknown
node04                                  idle        none                          32   2:8:2   191000    (null)                      (null)              node04              Unknown
node11                                  idle        none                          48   2:12:2  257000    (null)                      (null)              node11              Unknown
node12                                  idle        none                          48   2:12:2  225000    (null)                      (null)              node12              Unknown
gpu01                                   idle        none                          28   1:14:2  63000     (null)                      gpu:4               gpu01               Unknown
gpu02                                   idle        none                          28   1:14:2  63000     (null)                      gpu:4               gpu02               Unknown
```

**Node Specifications:**

| Node Type | CPUs | Socket:Core:Thread | Memory (MB) | GPUs | GPU Model |
|-----------|------|-------------------|-------------|------|-----------|
| Standard (node01-04) | 32 | 2:8:2 | ~191 GB | - | - |
| High-CPU (node11-12) | 48 | 2:12:2 | ~225-257 GB | - | - |
| GPU Nodes (gpu01-02) | 28 | 1:14:2 | ~63 GB | 4 | NVIDIA RTX 2080 Ti |

**Note**: The list of available nodes is expected to change regularly. Always use `list_nodes` to check current availability before submitting jobs.

### Troubleshooting FAQ
*   **"My job is stuck in PD (Pending) state"**
    *   This usually means the cluster is full. Check `sinfo` to see if nodes are `idle` or `alloc`. Your job will run automatically once resources are free.
*   **"I got an 'Out of Memory' (OOM) error"**
    *   Your job requested too little RAM. Edit your script to increase `--mem` (e.g., change `8G` to `16G`) and resubmit.
*   **"My simulation failed immediately"**
    *   Check your output file (e.g., `res_12345.out`). It will contain the error message from Python or your code.
*   **"I can't find my output files"**
    *   Ensure your script includes the `rsync` step to copy files from the local disk back to your `/home` directory.

---

## 6. Support & Pilot Status

### Important Notice
This cluster is a **pilot project**. It is an experimental environment used for research and educational purposes. 
- The system may experience unexpected downtime.
- Performance may vary.
- Hardware may be restarted for maintenance without extensive notice.

### Need Help?
If you encounter cluster-level issues (e.g., cannot log in, Slurm is down, node failures), please contact:
**[ISHelp@hw.ac.uk](mailto:ISHelp@hw.ac.uk)**

*Note: Support staff can assist with cluster access and availability, but they cannot debug your personal code or research logic.*
