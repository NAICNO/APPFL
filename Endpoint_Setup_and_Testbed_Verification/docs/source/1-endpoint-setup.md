# Part 1 — Setting up your endpoint

You are being asked to set up a **multi-user Globus Compute endpoint (MEP)** on your
cluster. When a task arrives from Argonne National Laboratory -- Leadership Computing Facility, your cluster runs it
**locally** and returns only results — nothing else leaves your machine. You authorize
one Globus identity; no secret is shared, and you never run the experiment yourself.

**This is an admin task.** You need privileged (root) access on a login node, a scheduler
account and partition, and `conda`.

Everything you need is in this bundle: this guide, Part 2, and the files they name. Nothing
is sent separately.

> **Already run another Globus Compute MEP on this cluster?** Your account, cluster and
> conda installation carry over, but do **not** reuse that endpoint or its environment.
> Build a separate one (`flamby-ep` with service account `flamby_svc`,
> running in environment `appfl_env`) so the two stay independent, with separate
> scheduler accounting.

This is Part 1 of 2 and is the same for every experiment. Part 2 covers what differs for
yours.

---

## Your details

| | |
| --- | --- |
| Your institution | University of Oslo |
| Your site ID | `Site2` |
| Endpoint to create | `flamby-ep` |
| Service account to create | `flamby_svc` |
| Scheduler | slurm |
| Identity to authorize | `mjoel@anl.gov` |

---

## Step 1 — Get the software

Create a conda environment named **`appfl_env`**:

```bash
conda create -n appfl_env python=3.12 -y
conda activate appfl_env
python -V                        # must print 3.12.x
```

Use exactly that name, and create it in a conda installation that `flamby_svc`
(Step 3) also uses. The worker template in Step 4c activates it by that name, as that
account, on a compute node. An environment the account cannot see does not fail here —
your workers just never start. If an environment called `appfl_env` already exists
on your cluster from an earlier project, do not reuse or replace it; tell us, and we will
give this one a different name.

Python must be **3.12**. We ship function bytecode between machines, and
bytecode is only portable within a minor version — a mismatch fails before any of your
code runs. The patch level does not need to match ours.

```bash
pip install --extra-index-url https://download.pytorch.org/whl/cpu \
    "appfl-bio-suite[flamby] @ git+https://github.com/mfjoel01/appfl-bio-suite.git@v0.2.1"
```

The extra index gives you the CPU-only build of PyTorch. Everything this federation runs
on your cluster runs on CPU, and the default build adds about 3 GB of GPU libraries to the
download that would never be used. Every version is otherwise identical.

The suite is installed from a git tag, not from PyPI. Every site in this federation
installs that same tag, so every site is running identical source. Please do not
substitute a branch, a newer tag, or `pip install --upgrade` — if you think you need a
different version, tell us and we will move the whole federation at once.

Verify:

```bash
globus-compute-endpoint version
```

It **must** print `4.9.0`. If it prints something else, stop and
tell us. Every site in the federation runs the same version; a mismatch surfaces as a
deserialization failure several rounds into a run, which is the most expensive kind of
bug for everyone involved.

## Step 2 — Check that multi-user mode works

```bash
globus-compute-endpoint configure --multi-user true _probe
globus-compute-endpoint delete --yes _probe
```

Both lines must finish with **no error**. If either errors, stop and send us the output
before continuing — we will align the whole federation on one version rather than have
you work around it.

## Step 3 — Create the service account

Create a local POSIX account named **`flamby_svc`**. It must be able to:

- activate `appfl_env` from Step 1
- submit slurm jobs
- read and write `/home/flamby_svc/fedlearn/outputs`

Every task we send runs as this account and nothing else. Check the first of these the
way your workers will do it:

```bash
sudo -iu flamby_svc bash -lc 'source "$(conda info --base)/etc/profile.d/conda.sh" && conda activate appfl_env && python -V'
```

It must print Python 3.12.x.

## Step 4 — Configure the endpoint

> If your centre already operates a Globus Compute MEP, do not build your own — send them
> this whole section and ask them to add it.

Run as the **privileged (root)** user:

```bash
globus-compute-endpoint configure --multi-user true flamby-ep
```

That creates `~/.globus_compute/flamby-ep/` with several files already filled
in. You only need to change two of them.

### 4a — `config.yaml` (usually no change)

`configure` already wrote this and already pointed it at the identity-mapping file it
created alongside. **Leave that path exactly as generated** — do not retype it. If
`display_name` came out as `null`, set it to `flamby-ep`. That is the only edit.

### 4b — the identity mapping

Replace the entire contents of the file that `identity_mapping_config_path` names —
`example_identity_mapping_config.json`, in the same folder — with the block below. We have
included it pre-filled as `example_identity_mapping_config.json` in this bundle.

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

This says: *if the submitter is `mjoel@anl.gov`, run the task as
`flamby_svc`.* That mapping is the entire gatekeeper. No other identity is
accepted.

> ### Do not add `^` or `$` anchors to `match`
>
> This is the single most expensive mistake on this project — it cost one site over a
> week, and a wrong diagnosis was sent to another before it was understood.
>
> `match` is **not** a normal regular expression. The Globus expression mapper *escapes*
> any `^` or `$` you write, turning them into literal characters, and then wraps your
> pattern in its own `^...$`. So anchoring the pattern as `^mjoel@anl\.gov$`
> compiles to a pattern that matches only a literal string beginning with a caret — it
> can never match anything, and every submission is rejected with
> `422 SEMANTICALLY_INVALID … Identity failed to map to a local user name`.
>
> The unanchored form above is still an exact whole-string match, because the mapper adds
> the anchors for you. Omitting them is not a loosening.
>
> Keep the backslash before each `.` — it escapes the dot so it matches only a literal
> dot. In the JSON file it is written doubled, `\\.`, because JSON itself needs a
> backslash escaped; a single `\.` is invalid JSON. Do **not** escape anything else: the
> mapper accepts only a small set of escapes and rejects the whole file if it sees one it
> does not recognize.

The endpoint **polls this file** and picks up changes within about five seconds. A restart
is not required after editing it.

### 4c — the worker template

Install the pre-filled `user_config_template.yaml.j2` from this bundle as
`~/.globus_compute/flamby-ep/user_config_template.yaml.j2`, replacing the one
`configure` wrote. This is its full content:

```yaml
# Globus Compute multi-user endpoint template for University of Oslo (Site2).
#
# Generated by `appfl-bio-suite partner-bundle` from the coordinator's federation config.
# Almost every value is filled in for your cluster. What we could not know from here is
# marked REPLACE_WITH_ below -- fill those in, then install this as
#     ~/.globus_compute/flamby-ep/user_config_template.yaml.j2
#
# Each worker_init line prevents a specific failure. Please keep all of them; see Part 1
# of your setup guide for what each one is for.

engine:
  type: GlobusComputeEngine
  max_workers_per_node: 1
  strategy: simple
  provider:
    type: SlurmProvider
    # Your scheduler account. We do not know it, so it is left explicit rather than
    # blank: an empty value here is not a default, it submits with no account and fails
    # at the scheduler rather than here.
    account: REPLACE_WITH_YOUR_SLURM_ACCOUNT
    # The partition these jobs run in. Prefer one with short queue waits: a
    # synchronous federated round advances at the speed of its slowest site.
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
      # Scheduler $TMPDIR paths overflow the ~108-char Unix socket limit and workers
      # then never register ("AF_UNIX path too long").
      export TMPDIR=/tmp
      # numpy's BLAS starts one thread per core on import; on a busy node that exhausts
      # the per-user thread budget ("can't start new thread").
      export OPENBLAS_NUM_THREADS=1
      export OMP_NUM_THREADS=1
      export MKL_NUM_THREADS=1
      export PYTHONUNBUFFERED=1

# Keep the worker block warm between tasks. Without these the block is torn down and
# re-queued each time, which on a busy queue turns minutes into hours.
idle_heartbeats_soft: 0
idle_heartbeats_hard: 5760
```

**Replace each `REPLACE_WITH_` value** with your own slurm account and
partition — we do not know them from here. Leave nothing else changed.

Every line in `worker_init` prevents a specific failure we have already hit. Please keep
all of them:

- **`export TMPDIR=/tmp`** — schedulers often set `$TMPDIR` to a long path. The worker
  pool opens a Unix socket underneath it, and the socket path limit is about 108
  characters. Overflow it and workers never register: you see
  *"workers failed to register"* or *"AF_UNIX path too long"*.
- **the three `*_NUM_THREADS=1` lines** — numpy's linear-algebra backend starts one
  thread per core on import. On a busy node that exhausts the per-user thread budget and
  you get `RuntimeError: can't start new thread`.
- **`idle_heartbeats_soft: 0` / `idle_heartbeats_hard: 5760`** — these keep your worker
  block warm between tasks. Without them the block is torn down and re-queued, which on a
  busy queue turns a run that should take minutes into one that takes hours.

> **Workers start but never connect back?** The engine could not auto-detect which network
> interface faces your compute nodes. Add this under `engine:`:
>
> ```yaml
>   address:
>     type: address_by_interface
>     ifname: <the interface your compute nodes use>
> ```

## Step 5 — Verify the mapping before you start

Reading the JSON by eye does not catch the anchor problem, because a wrong file looks
correct. Check it mechanically, in the same environment as the endpoint:

```bash
appfl-bio-suite identity validate \
    /root/.globus_compute/flamby-ep/example_identity_mapping_config.json \
    --identity 'mjoel@anl.gov' \
    --expect 'flamby_svc'
```

It prints what your `match` actually compiles to and says PASS or FAIL in about a second,
with no round trip to us. Get **PASS** before continuing. If it fails, no amount of
restarting will help — the message names the fix.

## Step 6 — Start the endpoint

Run as the **same privileged user** that ran Step 4:

```bash
globus-compute-endpoint start flamby-ep
globus-compute-endpoint list      # copy the Endpoint ID (UUID)
```

> **Two traps here, each of which has cost a site a week.**
>
> **1. Starting as an ordinary user silently disables identity mapping.** The endpoint
> still starts, with no error and nothing that looks wrong — but it ignores
> `identity_mapping_config_path` entirely and accepts only the identity that started it.
> Every submission from us is then rejected with `403`. "It started fine" is not evidence
> that Step 4b took effect.
>
> **2. Endpoint UUIDs are per-account.** The ID lives in
> `<that user's home>/.globus_compute/flamby-ep/endpoint.json`. An endpoint you
> built under your personal account and one built under `root` are *different endpoints
> with different UUIDs*. If you configured as yourself and later restarted as root, the
> UUID you send us must be the one `globus-compute-endpoint list` prints **as root**.

## Step 7 — Send us

1. Your **endpoint UUID** from Step 6.
2. One line confirming `mjoel@anl.gov` is mapped to `flamby_svc`.
3. Confirmation that `globus-compute-endpoint version` still prints
   `4.9.0`.

Then **leave the endpoint running**. Part 2 covers the data step for your experiment, and
after that we drive everything from our side.

---

## Troubleshooting

**`422 SEMANTICALLY_INVALID … Identity failed to map to a local user name`**
Good news: your endpoint is privileged and its mapping *is* being read. The expression
just did not match us. Almost always the `^`/`$` anchors — see the callout in Step 4b.
Run the Step 5 validator; it answers this in a second. Next most likely: you edited a
different file from the one `identity_mapping_config_path` names, which is easy to do if
that path still points into a personal home directory after you switched to root. No
restart is needed after a fix.

**`403 ENDPOINT_ACCESS_FORBIDDEN`**
This is *not* a mapping-expression problem — do not start editing the JSON. It almost
always means the endpoint was started by an ordinary user, so identity mapping was skipped
(see Step 6). Confirm who owns the running process, restart as the privileged user, then
**re-read the UUID** — it will have changed.

**`404`**
Wrong UUID, or the endpoint was deleted. Note that `delete` followed by `configure` issues
a **new** UUID; `stop`/`start`/`restart` preserve it.

**Version is not 4.9.0**
You installed without the pin. Reinstall and tell us — every site must match.

**"workers failed to register" / "AF_UNIX path too long"**
`export TMPDIR=/tmp` is missing from `worker_init`.

**`RuntimeError: can't start new thread`**
The three `*_NUM_THREADS=1` lines are missing from `worker_init`.

**First task works, later ones stall**
Your block is being torn down between tasks and re-queued. Check
`idle_heartbeats_soft: 0` and `idle_heartbeats_hard: 5760` are present, and prefer a
partition with short queue waits.

**Nothing happens at all**
The queue is busy and the job is waiting in line. That is normal; tell us if it persists.

Anything else — send us `~/.globus_compute/flamby-ep/*.log` and we will take it
from there. Contact: mjoel@anl.gov
