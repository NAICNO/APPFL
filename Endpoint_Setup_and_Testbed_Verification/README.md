# APPFL Endpoint Setup and Testbed Verification Guide

This repository contains step-by-step setup instructions, environment specifications, configuration templates, and verification scripts for configuring a **Multi-User Globus Compute Endpoint (MEP)** and running initial validation tests within the Advanced Privacy-Preserving Federated Learning (APPFL) testbed framework.

The deployment configures a local HPC compute node (e.g., University of Oslo / Site2) to participate in federated training. Tasks sent by the federation coordinator (`mjoel@anl.gov`) are executed locally under a dedicated POSIX service account (`flamby_svc`), ensuring zero exposure of raw dataset files or local cluster credentials.

---

## Technical Overview & System Requirements

### Cluster Requirements
* **Operating System**: Linux HPC cluster login/compute environment (POSIX compliant).
* **Privilege Level**: Privileged (`root`) access on the login node for endpoint deployment, identity mapping configuration, and multi-user service mapping.
* **Job Scheduler**: Slurm Workload Manager with an active user account and target compute partition.
* **Environment Manager**: Conda package manager.
* **Python Runtime**: Python 3.12 (exact minor version required for cross-node function bytecode compatibility).

---

## Phase 1: Environment Preparation and Dependency Installation

### Step 1.1: Create and Activate Conda Environment
Create a Conda environment named `appfl_env` configured specifically with Python 3.12. Globus Compute function bytecode requires exact minor version matching across all participating federation nodes.

```bash
conda create -n appfl_env python=3.12 -y
conda activate appfl_env
python -V
```
*Verification*: Must output `Python 3.12.x`.

### Step 1.2: Install Pinned APPFL Bio-Suite and CPU PyTorch
Install CPU-only PyTorch binaries alongside the pinned release of the `appfl-bio-suite` framework directly from Git tag `v0.2.1`:

```bash
pip install --extra-index-url https://download.pytorch.org/whl/cpu \
    "appfl-bio-suite[flamby] @ git+https://github.com/mfjoel01/appfl-bio-suite.git@v0.2.1"
```

### Step 1.3: Verify Endpoint Version
Confirm that the installed Globus Compute endpoint version matches the federation-wide pinned standard:

```bash
globus-compute-endpoint version
```
*Requirement*: The command **must output `4.9.0`**. Any version mismatch will lead to deserialization errors during federated execution rounds.

### Step 1.4: Validate Multi-User Mode Capability
Confirm that multi-user endpoint configuration operates properly on your cluster by configuring and deleting a probe endpoint:

```bash
globus-compute-endpoint configure --multi-user true _probe
globus-compute-endpoint delete --yes _probe
```
Both commands must complete without errors.

### Step 1.5: Create Dedicated POSIX Service Account
Create a local POSIX service account named `flamby_svc` to execute worker tasks under isolated privileges.

1. Provision the local service account:
   ```bash
   useradd -m flamby_svc
   ```
2. Verify that `flamby_svc` can submit Slurm jobs, activate the `appfl_env` Conda environment, and read/write to `/home/flamby_svc/fedlearn/outputs`.
3. Test environment activation under the service account context:
   ```bash
   sudo -iu flamby_svc bash -lc 'source "$(conda info --base)/etc/profile.d/conda.sh" && conda activate appfl_env && python -V'
   ```
*Verification*: Output must confirm `Python 3.12.x`.

---

## Phase 2: Multi-User Endpoint Configuration and Service Mapping

### Step 2.1: Initialize Multi-User Endpoint
As the **root** user, initialize the multi-user endpoint instance named `flamby-ep`:

```bash
globus-compute-endpoint configure --multi-user true flamby-ep
```
This generates configuration files in `~/.globus_compute/flamby-ep/`.

### Step 2.2: Configure Main Endpoint Settings (`config.yaml`)
Inspect `~/.globus_compute/flamby-ep/config.yaml`:
* Retain the automatically generated `identity_mapping_config_path` value without modification.
* Verify that `display_name` is explicitly set to `flamby-ep`.

### Step 2.3: Configure Strict Identity Mapping
Replace the contents of `~/.globus_compute/flamby-ep/example_identity_mapping_config.json` with the identity mapping block authorizing `mjoel@anl.gov` to run tasks as `flamby_svc`:

```json
[
  {
    "DATA_TYPE": "expression_identity_mapping#1.0.0",
    "mappings": [
      {
        "source": "{username}",
        "match": "mjoel@anl\\.gov",
        "output": "flamby_svc"
      }
    ]
  }
]
```

> **CRITICAL CONFIGURATION WARNING**: Do NOT add regex start or end anchors (`^` or `$`) to the `match` field (e.g., do not write `^mjoel@anl\.gov$`). The Globus expression mapper automatically wraps expressions in anchors. Adding explicit anchors causes literal escaping that results in permanent authorization failures (`422 SEMANTICALLY_INVALID ... Identity failed to map to a local user name`).

### Step 2.4: Deploy Slurm Worker Configuration (`user_config_template.yaml.j2`)
Install the worker template at `~/.globus_compute/flamby-ep/user_config_template.yaml.j2`:

```yaml
engine:
  type: GlobusComputeEngine
  max_workers_per_node: 1
  strategy: simple
  provider:
    type: SlurmProvider
    account: REPLACE_WITH_YOUR_SLURM_ACCOUNT
    partition: REPLACE_WITH_YOUR_PARTITION
    nodes_per_block: 1
    init_blocks: 0
    min_blocks: 0
    max_blocks: 1
    walltime: 00:30:00
    launcher:
      type: SrunLauncher
    worker_init: |
      source "$(conda info --base)/etc/profile.d/conda.sh"
      conda activate appfl_env
      export TMPDIR=/tmp
      export OPENBLAS_NUM_THREADS=1
      export OMP_NUM_THREADS=1
      export MKL_NUM_THREADS=1
      export PYTHONUNBUFFERED=1

idle_heartbeats_soft: 0
idle_heartbeats_hard: 5760
```

#### Key Configuration Directives:
* **`account` & `partition`**: Replace `REPLACE_WITH_YOUR_SLURM_ACCOUNT` and `REPLACE_WITH_YOUR_PARTITION` with your site's specific Slurm allocation details.
* **`export TMPDIR=/tmp`**: Prevents Unix socket length overflow (>108 characters) caused by long scheduler paths (`workers failed to register` / `AF_UNIX path too long`).
* **`*_NUM_THREADS=1`**: Restricts NumPy/BLAS multi-threading to 1 thread per worker, preventing per-user thread limit exhaustion (`RuntimeError: can't start new thread`).
* **`idle_heartbeats_hard: 5760`**: Keeps compute worker blocks warm between tasks, avoiding repetitive Slurm queue waiting times.
* **Compute Network Interface Note**: If workers start but fail to connect back to the endpoint engine, add the explicit network interface under `engine`:
  ```yaml
    address:
      type: address_by_interface
      ifname: <your_compute_node_interface>
  ```

### Step 2.5: Mechanically Validate Identity Mapping
As the **root** user, execute the identity validation tool before launching the endpoint service:

```bash
appfl-bio-suite identity validate \
    /root/.globus_compute/flamby-ep/example_identity_mapping_config.json \
    --identity 'mjoel@anl.gov' \
    --expect 'flamby_svc'
```
*Requirement*: Must print **`PASS`**.

### Step 2.6: Start Endpoint Service
Launch the endpoint as the **root** user and obtain the Endpoint UUID:

```bash
globus-compute-endpoint start flamby-ep
globus-compute-endpoint list
```
Record the assigned **Endpoint UUID**.

> **SECURITY NOTICE**: Always start the endpoint as the `root` user. Starting the endpoint as an ordinary user silently bypasses identity mapping and exposes the endpoint to authorization failure (`403 ENDPOINT_ACCESS_FORBIDDEN`). Furthermore, Endpoint UUIDs are user-account specific; the UUID printed under `root` must be used.

---

## Phase 3: FLamby Benchmark Setup and Data Acquisition

### Step 3.1: Switch to Service Account Context
Perform all dataset downloads and benchmark setup commands as `flamby_svc`:

```bash
sudo -iu flamby_svc
conda activate appfl_env
```

### Step 3.2: Clone FLamby and Create Safe Module Link
Clone the FLamby benchmark repository and write a `.pth` path reference to expose the package without triggering `setup.py`:

```bash
git clone https://github.com/owkin/FLamby.git
cd FLamby
python -c "import site, pathlib; pathlib.Path(site.getsitepackages()[0], 'flamby.pth').write_text(str(pathlib.Path.cwd()))"
```

> **DO NOT RUN `pip install -e .` OR `make install`**: FLamby's `setup.py` hooks shell out to external package indexes that pull unpinned dependencies, which can overwrite the pinned `globus-compute-endpoint` version and break federation compatibility.

Verify module importability:
```bash
python -c "from flamby.datasets.fed_heart_disease import FedHeartDisease; import flamby; print('flamby importable from', flamby.__file__)"
```

Confirm that the endpoint version remains `4.9.0`:
```bash
globus-compute-endpoint version
```

### Step 3.3: Download Fed-Heart-Disease Dataset (Centre 1)
Download the benchmark dataset into `/home/flamby_svc/fedlearn/data`:

```bash
mkdir -p /home/flamby_svc/fedlearn/data
cd flamby/datasets/fed_heart_disease/dataset_creation_scripts
python download.py --output-folder /home/flamby_svc/fedlearn/data
```
Accept the CC BY 4.0 data terms by entering `y` when prompted.

*Note*: If the dataset is moved to a new directory later, update the configuration using:
```bash
python update_config.py --new-path <new_absolute_path>
```

---

## Phase 4: Initial Verification Test Execution

Run the dataset validation script under `flamby_svc` in the `appfl_env` environment to verify data loading and sample integrity:

```bash
python -c "
from flamby.datasets.fed_heart_disease import FedHeartDisease
d = FedHeartDisease(train=True, center=1, pooled=False)
print('OK, samples:', len(d))
"
```

### Expected Output
The verification test must return exactly:
```text
OK, samples: 172
```

---

## Phase 5: Federation Handoff

Upon successful verification, transmit the handoff confirmation to the federation coordinator (`mjoel@anl.gov`) containing:

1. **Endpoint UUID**: The UUID obtained from `globus-compute-endpoint list` (run as `root`).
2. **Identity Mapping Confirmation**: Statement confirming `mjoel@anl.gov` maps to `flamby_svc`.
3. **Version Confirmation**: Statement confirming `globus-compute-endpoint version` returns `4.9.0`.
4. **Dataset Verification Confirmation**: The explicit phrase `"Fed-Heart-Disease centre 1 loads, 172 training samples."`

Leave the endpoint process active. Subsequent federated training rounds will be orchestrated remotely by the coordinator.

---

## Troubleshooting Guide

| Error / Symptom | Primary Cause | Solution / Fix Procedure |
| :--- | :--- | :--- |
| `422 SEMANTICALLY_INVALID ... Identity failed to map` | Anchors (`^` or `$`) included in identity mapping JSON regex match string. | Edit `example_identity_mapping_config.json` to remove explicit anchors (`mjoel@anl\\.gov`). Validate using `appfl-bio-suite identity validate`. |
| `403 ENDPOINT_ACCESS_FORBIDDEN` | Endpoint process launched by standard user rather than `root`. | Stop non-root endpoint process, restart as `root`, and copy the newly generated root Endpoint UUID. |
| `404 NOT FOUND` | Mismatched Endpoint UUID or endpoint instance deleted. | Verify current active Endpoint UUID using `globus-compute-endpoint list` under `root`. |
| `workers failed to register` / `AF_UNIX path too long` | Scheduler setting long `$TMPDIR` path exceeding 108-character Unix socket limit. | Confirm `export TMPDIR=/tmp` is present in worker template `worker_init`. |
| `RuntimeError: can't start new thread` | BLAS library exhausting per-user thread limit. | Ensure `OPENBLAS_NUM_THREADS=1`, `OMP_NUM_THREADS=1`, and `MKL_NUM_THREADS=1` are present in `worker_init`. |
| `ModuleNotFoundError: flamby` | Execution running in wrong environment or user context. | Switch to `flamby_svc` user, activate `appfl_env`, and re-apply `.pth` module path linking. |
| Dataset Sample Count Mismatch | Incorrect dataset download or centre assignment. | Report actual sample count to `mjoel@anl.gov` before proceeding. |
