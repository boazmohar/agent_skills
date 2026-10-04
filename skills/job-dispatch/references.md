# Janelia Research Campus compute cluster (LSF) -- cheat sheet

Cached summary of the HPC team's "AI Agent Hints" page, **last updated 2026-09-29**.
Canonical, authoritative source: <https://hpc.int.janelia.org/docs/ai-agent-hints>
(internal network / VPN). If this file is more than 30 days old, fetch the live page:
queue limits, runtime caps, GPU fleet and prices change through the year.

Contact: clusteradmins@janelia.hhmi.org. Helpdesk via IT Services.

## Cluster identity

| Item | Value |
|---|---|
| Scheduler | IBM Spectrum LSF 10.1 (not Slurm / PBS / SGE / Kubernetes) |
| OS | RHEL 9.8 on all nodes |
| CUDA | 13.3 default at `/usr/local/cuda`; `/usr/local/cuda-11`, `/usr/local/cuda-12` also present on compute nodes (not login nodes) |
| Containers | Apptainer 1.4.x for trusted images (`--nv` for GPU). Docker does not exist. Rootless podman only with a per-user subuid range from the HPC team. |
| Storage | NFS-based (PRFS primary, NRS non-redundant). No parallel filesystem. |
| Interconnect | Ethernet everywhere; InfiniBand only on H100/H200 (400 Gbit NDR) and B300 (800 Gbit XDR) nodes, and only across nodes of the same generation. |
| Billing | $0.05 per slot-hour plus a per-GPU rate; charged from start to end, failed and killed jobs included. Cycle: 15th to 14th. |

Dashboards (internal network): cluster status <https://cluster-status.int.janelia.org>, GPU utilisation <https://gpustats.int.janelia.org>, RTM job monitoring <https://lsf-rtm.int.janelia.org>, storage usage <https://storage-usage.int.janelia.org>, Starfish <https://starfish.int.janelia.org>, wiki <https://hpc.int.janelia.org>.

## Critical rules

Always:
- Submit all computation as LSF jobs with `bsub`, even small tasks. Login nodes (`login1`, `login2`) and the submit host (`submit.int.janelia.org`, 1 core / 16 GB shared) are for `bsub`/`bjobs` and light orchestration only.
- Set `-W` on every job. Defaults without it: 14 d on `local`, 8 h on `interactive`, **2 h on named GPU queues**.
- Size with the slot model (below). Jobs exceeding their memory are killed with no recovery.
- Use `-o` and `-e` (with `%J`, and `%I` for arrays) so LSF does not e-mail per job. `-o /dev/null` if the code saves its own results.
- `bkill` interactive and GPU jobs when done.
- Clean up `/scratch/$USER/` before the job ends (node-local SSD, swept periodically, ~25 GB per slot).
- Irreplaceable data on PRFS (`/groups/...`), nightly backups, 30-day retention.
- `module load` for software, especially MATLAB (hardcoded `/misc/local/matlab-*/bin/matlab` paths leave orphaned daemons).
- Check RTM after test runs and right-size.
- Set threading env vars in every Python/ML job (see below).
- **Sandbox unsupervised agentic loops** (required since 2026-08-28): bwrap when no GPU, rootless podman when GPU; never Apptainer for this. Examples: <https://github.com/JaneliaScientificComputingSystems/agentic-sandbox>.

Never:
- Run computation, compiles, `pip install`, `conda create` or heavy file operations on login nodes or the submit host.
- Submit more than 15,000 jobs at once; use array jobs.
- `watch bjobs` with short intervals; `watch -n 60 bjobs` at minimum, RTM preferred.
- Use `/tmp` on compute nodes; use `/scratch/$USER/`.
- VS Code Remote SSH to compute or login nodes; use NoMachine + interactive job, or code-server as a job.
- Put passwords, tokens or usernames in job names (`bjobs -u all` shows them to everyone).
- Assume a program is parallel; 48 slots do not speed up single-threaded code.
- Set `CUDA_VISIBLE_DEVICES` in job scripts; LSF sets it (`$CUDA_VISIBLE_DEVICES_ORIG` holds host IDs).
- Submit partial-node jobs to `*_parallel` queues.
- SSH to compute nodes to poll `nvidia-smi` or jobs. One-off human debugging on a node is fine; scripted polling is not.

## Slot model

1 slot = 1 physical core + fixed RAM:

| Queue family | RAM per slot |
|---|---|
| CPU queues, `gpu_l4*`, `gpu_rtx6000`, `gpu_short` | 15 GB |
| `gpu_a100`, `gpu_h100`, `gpu_h200`, `gpu_b300` | 40 GB |

slots = max(cores needed, ceil(RAM_GB / RAM_per_slot)). Cores and memory cannot be requested separately. Examples at 15 GB/slot: 4 cores + 30 GB -> `-n 4`; 2 cores + 100 GB -> `-n 7`; 16 cores + 500 GB -> `-n 34`. At 40 GB/slot: 4 cores + 100 GB -> `-n 4`; 2 cores + 200 GB -> `-n 5`.

Hyperthreading is on, so the OS shows 2x the physical cores. LSF slots map to physical cores; do not read `lscpu` and assume more are available, and do not thread to logical cores.

`$LSB_DJOB_NUMPROC` = slots allocated. Use it for thread counts.

## CPU queues

| Queue | Max runtime | Default runtime | Per-user limit | Notes |
|---|---|---|---|---|
| `short` | 1 h | 1 h | none | Jobs with `-W <= 60` auto-route here. Best turnaround. |
| `local` | 14 d | 14 d | 5,999 slots | Default batch queue. |
| `interactive` | 48 h | 8 h | 128 slots or 4 jobs | `bsub -Is /bin/bash`; GUI apps and testing. |
| `cpu_parallel` | 14 d | -- | -- | Multi-node MPI, whole nodes (128 slots) only. |

## GPU queues

| Queue | GPU | $/GPU/h | GPUs/node | Slots/GPU | Node slots | RAM/slot | VRAM |
|---|---|---|---|---|---|---|---|
| `gpu_l4` | L4 | 0.10 | 8 | 8 | 64 | 15 GB | 24 GB |
| `gpu_l4_16` | L4 | 0.10 | 8 | 16 | 128 | 15 GB | 24 GB |
| `gpu_l4_large` | L4 | 0.10 | 1 | 64 | 64 | 15 GB | 24 GB |
| `gpu_a100` | A100 | 0.20 | 4 | 12 | 48 | 40 GB | 80 GB |
| `gpu_rtx6000` | RTX PRO 6000 | 0.50 | 8 | 16 | 128 | 15 GB | 96 GB |
| `gpu_h100` | H100 | 0.65 | 8 | 12 | 96 | 40 GB | 80 GB |
| `gpu_h200` | H200 | 0.80 | 8 | 12 | 96 | 40 GB | 141 GB |
| `gpu_b300` | B300 | 2.40 | 8 | 12 | 96 | 40 GB | 268 GB |
| `gpu_short` | any | 0.10 | varies | varies | varies | 15 GB | varies |

- GPU queues default to **2 h** without `-W`; max 14 d. `gpu_short` hard 1 h limit, any GPU type (size slots for the smallest ratio, 8).
- `bsub -gpu "num=N"` needs an explicit `-q gpu_*`; it does not infer one.
- Per-user GPU limit is about 50% of each queue.
- Parallel variants (`gpu_l4_parallel`, `gpu_a100_parallel`, `gpu_h100_parallel`, `gpu_h200_parallel`, `gpu_b300_parallel`): whole nodes, relaxed fencing, `-app parallel-<slots>`, `-gpu "num=<GPUs per node>"` is per host.

Choosing: start with `gpu_l4`. Move up only for >24 GB VRAM, NVLink or multi-GPU in a node. `gpu_l4_large` / `gpu_l4_16` when CPU/memory-bound with occasional GPU use. Single-node fine-tuning or inference: `gpu_rtx6000` if the model fits in 96 GB (no NVLink, no IB, native FP4), `gpu_a100` for NVLink within a node. Multi-node: `gpu_h100` / `gpu_h200` (`_parallel`). Largest models, Blackwell-native FP4/FP8 kernels (e.g. Kimi-K3): `gpu_b300`. FP64: H200 > H100 > A100, never B300. Check bf16 before requesting a bigger GPU; fp32 on a B300 uses ~2% of its compute.

Slot-to-GPU ratio: requesting more slots than the ratio strands GPUs for other users (`-n 24 -gpu "num=1"` on `gpu_l4` holds 3 GPUs' worth of slots). Fewer is fine for GPU-bound code.

## bsub templates

```bash
# CPU batch
bsub -n 4 -W 2:00 -q local -J myjob -o logs/out_%J.log -e logs/err_%J.log bash run_my.sh <args>

# Quick CPU test (auto-routes to short when -W <= 60)
bsub -n 2 -W 0:30 -J smoke -o logs/smoke_%J.log bash run_my.sh <args>

# 1x L4 GPU, 2 h
bsub -n 8 -gpu "num=1" -q gpu_l4 -W 2:00 -J train -o logs/train_%J.log -e logs/train_%J.err bash run_my.sh <args>

# 2x A100 with the standard ratio
bsub -n 24 -gpu "num=2" -q gpu_a100 -W 8:00 -J train2 -o logs/train2_%J.log bash run_my.sh <args>

# Interactive CPU shell on a compute node (install packages, debug)
bsub -n 4 -W 1:00 -Is /bin/bash

# Interactive GPU session
bsub -n 8 -gpu "num=1" -q gpu_l4 -W 4:00 -Is /bin/bash

# Array job: 1000 tasks, 15 concurrent, 1-based $LSB_JOBINDEX
bsub -n 1 -W 1:00 -J "sweep[1-1000]%15" -o "logs/sweep_%J_%I.log" bash run_array.sh /groups/<lab>/project/runs/table.tsv

# Multi-node GPU (2 nodes x 8 H100)
bsub -n 192 -app parallel-96 -q gpu_h100_parallel -gpu "num=8:mode=shared" -W 24:00 mpirun -np 16 --map-by ppr:8:node ./gpu_app

# MATLAB batch
module load matlab && bsub -n 4 -W 2:00 -o output.log matlab -nojvm -batch "my_script;"
```

Array worker skeleton (row `N` of a TSV, columns: name, arg_a, dir_save):

```bash
TABLE=$1
IFS=$'\t' read -r RUN_NAME ARG_A DIR_SAVE < <(sed -n "${LSB_JOBINDEX}p" "$TABLE")
```

Most users should **not** pass `-P`; LSF assigns the project from the user. `lsfgroup <username>` shows the default.

## Environment variables inside a job

| Variable | Contents |
|---|---|
| `LSB_JOBID` | Job ID |
| `LSB_JOBINDEX` | Array task index (1-based) |
| `LSB_DJOB_NUMPROC` | Slots allocated |
| `LSB_JOBNAME` | Name from `-J` |
| `LSB_BATCH_JID` | Combined ID, e.g. `13234[3]` |
| `CUDA_VISIBLE_DEVICES` | GPUs assigned by LSF; read, never write |

## Threading

Set all of these in the job script; unset, each library spawns node-core-count threads per process. Layered threading multiplies (framework threads x BLAS threads), so size so that total <= slots:

```bash
export OMP_NUM_THREADS=$LSB_DJOB_NUMPROC
export MKL_NUM_THREADS=$LSB_DJOB_NUMPROC
export OPENBLAS_NUM_THREADS=$LSB_DJOB_NUMPROC
export TBB_NUM_THREADS=$LSB_DJOB_NUMPROC
export OPENMP_NUM_THREADS=$LSB_DJOB_NUMPROC
export NUM_MKL_THREADS=$LSB_DJOB_NUMPROC
# PyTorch DataLoader workers x threads per worker <= $LSB_DJOB_NUMPROC
```

## Storage

| Tier | Path | Backed up | $/TB/mo | Cluster access | Use |
|---|---|---|---|---|---|
| PRFS | `/groups/<lab>/...` | nightly, 30 d | 12 (1 TB free per lab) | yes | active data, home dirs, irreplaceable files |
| NRS | `/nrs/<lab>/...` | no | 10 | yes | reproducible or temporary data |
| nearline | `/nearline/<lab>/...` | nightly, 30 d | 3 | **no** (desktops, `dtn.int.janelia.org`) | cold archive |
| scratch | `/scratch/$USER/` | no (node-local SSD) | free | job's node only | temp files during a job |
| misc/public | `/misc/public/` | no, purged after 30 d | free | yes | short-term sharing |

Home: `/groups/<groupname>/home/<username>`. Quotas are hard limits in 5 TB steps. POSIX permissions only, default umask 022; `umask 002` in job scripts for group write. `sudo` does nothing on network storage.

Transfers: from a Mac mount `smb://prfs.hhmi.org/<lab>/...` (NRS: `nrs.hhmi.org`) and copy directly; do not route through login nodes. Inside Janelia `rsync`/`cp` as a job if it is not near-instant; >500 GB or >10,000 files: Globus or a helpdesk ticket.

## Monitoring

```bash
bjobs                         # your jobs
bjobs -l <jobid>              # details incl. GPU assignment; works ~7 days after completion, prefer over bhist/bacct
bjobs -u all                  # everyone
bjobs ID1 ID2 ID3             # one call for many jobs (do this, not one call per job)
bjobs <arrayid>               # all elements of an array in one call
bqueues / bhosts              # queue state, host availability
bpeek <jobid>                 # live stdout/stderr
bkill <jobid> ; bkill -J "prefix*" ; bkill 0   # one, a named sweep, everything
lsload -gpuload <hostname>    # GPU use on a node, from a login node, <= every 10-15 s
```

`bhist` / `bacct` scan ~149 M records: slow, submit them as jobs, only for jobs older than 7 days.

Polling schedule (agent-driven monitoring must follow it):
- Pending: first check after ~1 min, then 2, 4, 8 min... never under 1 min.
- Running: wait about half the walltime, then halve the remaining time each check, never under 1 min.
- Consolidate: one `bjobs` call for all tracked jobs at the soonest next-check time.

RTM is the right-sizing tool: max memory used vs reserved, job efficiency (CPU vs slots), minute-by-minute graphs.

## Slurm -> LSF translation

| Slurm | LSF |
|---|---|
| `sbatch script.sh` | `bsub < script.sh` (parses `#BSUB`) or `bsub ./script.sh` (argv only) |
| `srun --pty bash` | `bsub -Is /bin/bash` |
| `squeue` / `squeue -u all` | `bjobs` / `bjobs -u all` |
| `scancel JOBID` | `bkill JOBID` |
| `sinfo` | `bhosts`, `bqueues` |
| `sacct` | RTM, or `bhist` |
| `--cpus-per-task` + `--mem` | `-n <slots>` (atomic core+RAM unit) |
| `--gres=gpu:N` | `-gpu "num=N" -q gpu_*` |
| `--time` | `-W H:MM` |
| `--array=0-99%15` | `-J "name[1-100]%15"` (1-based) |
| `$SLURM_ARRAY_TASK_ID` | `$LSB_JOBINDEX` |
| `$SLURM_CPUS_PER_TASK` | `$LSB_DJOB_NUMPROC` |

## Containers (trusted images, not agent sandboxes)

```bash
apptainer exec --nv --bind /groups/mylab:/data:ro /misc/local/singularity/pytorch_25.09-py3.sif python train.py
apptainer build my_container.sif docker://nvcr.io/nvidia/pytorch:24.01-py3
```

`--nv` is required for GPU. Home, `/tmp` and `/scratch` are auto-mounted. Pre-built: `/misc/local/singularity/pytorch_25.09-py3.sif`, `tensorflow_25.02-tf2-py3.sif`. Pre-downloaded models under `/misc/hf/<org>/<model>`.

## NCCL / multi-node

```bash
# H100 / H200 / B300: InfiniBand present, leave enabled
export NCCL_BUFFSIZE=8388608
# A100 / L4 / RTX 6000 Pro: Ethernet only
export NCCL_IB_DISABLE=1 NCCL_NET_GDR_LEVEL=0 NCCL_BUFFSIZE=8388608
export RAY_TMPDIR="/scratch/$(whoami)/ray_$(whoami)"
```

Recipes: <https://github.com/JaneliaScientificComputingSystems/gpu-cluster-recipes>. Always-on model serving (Kimi-K3 endpoint behind LiteLLM with a Haiku-based safeguard, everything logged): request a key from clusteradmins.

## Agentic sandbox command reference

Repo: <https://github.com/JaneliaScientificComputingSystems/agentic-sandbox>. Clone to shared storage so compute nodes see it. Both wrappers live in `scripts/`, resolve their helpers from their own location, and always bind the toolchain read-only plus `$PWD` read-write; `$HOME` only if explicitly requested (never do that).

```
sandbox-run.sh [options] -- <command...>     # bwrap, no GPU (default choice)
podman-run.sh  [options] -- <command...>     # podman + NVIDIA CDI, GPU-capable
  --ro PATH      read-only bind (repeatable)
  --rw PATH      read-write bind (repeatable)
  --allow HOST   allowed egress host (repeatable); omit for no network at all
  --scratch      --rw /scratch/$USER
  --claude       bind ~/.claude, ~/.claude.json and the CLI install (claude auth login first)
  --opencode     bind opencode's four XDG state dirs (one such sandbox per account at a time)
  --gpu          podman only: --device nvidia.com/gpu=all
  --image NAME   podman only: default ghcr.io/janeliascientificcomputingsystems/agentic-sandbox-lite:latest
                 (CUDA base + Node + Claude Code + opencode, 1.24 GB); -gpu:latest has PyTorch
  --keep-id      podman only: run as your real uid/gid (needs a subuid range wider than your GID)
```

Hosts to allow: Claude Code `api.anthropic.com`, `claude.ai`, `platform.claude.com`; opencode + Kimi K3 `litellm.int.janelia.org`. Nothing for pure compute.

Facts verified by the HPC team (see `ADMIN-NOTES.md` in the repo): LSF cgroup limits apply unchanged inside the sandbox; `bkill` kills the whole tree; bwrap needs `user.max_user_namespaces > 0`; bwrap masks `.ssh`, `.aws`, `.azure`, `.kube`, `.config/gcloud`, `.docker`, `.git-credentials`, `.npmrc`, `.pypirc`, `.netrc`, `.env*` wherever they appear, podman cannot mask anything; podman storage must be pointed at `/scratch/$USER` once per user (`~/.config/containers/storage.conf`) and needs a subuid/subgid range from the HPC team; `bsub -gpu` needs an explicit GPU queue; backgrounded helpers can outlive a job that ends normally, both wrappers clean theirs up.

## Billing example

`-n 12 -gpu "num=1" -q gpu_h100` for 10 h: slots 12 x 10 x 0.05 = $6.00, GPU 1 x 10 x 0.65 = $6.50, total $12.50. Accurate `-W` and killing idle interactive jobs is what keeps this down.
