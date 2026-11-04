# Heriot-Watt Student Cluster User Guide

Welcome to the Heriot-Watt student cluster. This guide is designed to help you transition from your local machine to running high-performance computing (HPC) jobs on this cluster.

---

## 0. The Workflow at a Glance

To ensure the cluster remains fast for everyone, we follow a specific data lifecycle. **Do not run simulations directly in your `/home` directory.**

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
ssh <your_username>@<cluster_hostname>
```

### Remote Access (Off-Campus)
If you are accessing the cluster from home or another external network, you **must** connect through our **SSH Gateway** first.
1. Connect to the gateway:
   ```bash
   ssh <your_username>@<gateway_address>
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
*   **`module load <module_name>`**: Loads a specific software package (e.g., `module load python/3.10`).
*   **`module list`**: Shows which modules you currently have loaded.
*   **`module purge`**: Removes all currently loaded modules (useful for starting fresh).

---

## 3. Data Management & Performance

### The Golden Rule: Use Local Disks for Computation
Running simulations directly in your `/home` directory is **extremely inefficient**. The `/home` directory is a network-mounted filesystem, and high-frequency reads/writes can slow down the entire cluster.

**For maximum performance, always run your simulations using the node's local disk.**

### Recommended Data Transfer
Use `rsync` from your **local machine** to upload data to your cluster home directory:
```bash
# Copy a folder from your laptop to the cluster
rsync -avz ./my_project <your_username>@<cluster_hostname>:/home/<your_username>/
```

---

## 4. Running Jobs (Slurm)

We use **Slurm** to manage job scheduling and resource allocation.

### Partitions (Queues)
1.  **`nodes`**: For standard CPU-intensive tasks.
2.  **`gpu`**: For tasks requiring GPU acceleration.

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
module load python/3.10

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