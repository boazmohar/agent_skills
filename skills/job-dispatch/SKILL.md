---
name: job-dispatch
description: Job dispatch infrastructure for the Janelia LSF cluster. Use when packaging a Python script for batch / parallel execution (laptop or cluster), when submitting bsub jobs on the user's behalf, and to decide whether an agentic sandbox (bwrap / podman) is required for the run.
---

# Job Dispatch (Janelia LSF)

Package an existing Python analysis script for batch execution. This skill handles **infrastructure only**: the worker shell script, the Python submitter, the directory layout, and the sandbox wrapper when one is required. The core script must already conform to `script-opinions` (argparse, dual-handler logging, run dir). If it doesn't, fix it first.

The cluster is **IBM Spectrum LSF 10.1, not Slurm**. Never emit `sbatch`, `srun`, `squeue`, `scancel` or `#SBATCH`. The authoritative cluster document is <https://hpc.int.janelia.org/docs/ai-agent-hints> (internal network / VPN). `references.md` is a cached summary of it dated 2026-09-29; if that is more than 30 days old, fetch the live page before recommending queues, limits or prices.

## Step 0: which of the three contexts are you in?

Everything else in this skill depends on this answer. Decide it first and say which one you are in.

| Context | Who executes code, where | Human reviewing each step? | Sandbox |
|---|---|---|---|
| **1. Laptop** | The analysis script runs on the user's Mac, in the user's own session. | Yes (the user is in the chat) | **No** |
| **2. Dispatch to LSF** | You are on the laptop or on a login/submit host. You `bsub` a *reviewed, fixed* worker script; LSF runs it on compute nodes. | Yes (you are in the chat; the job runs a script, not an agent) | **No** |
| **3. Agent on the cluster** | An agent process (`claude -p`, `opencode run`, an interactive `claude` under `bsub -Is`, or any loop that executes agent-generated code) runs inside an LSF job on a compute node. | No, or only loosely | **Required**: `sandbox-run.sh` (bwrap, CPU) or `podman-run.sh --gpu` (GPU) |

Detect it:

```bash
uname -s                 # Darwin  -> context 1 or 2 (you are on the laptop)
hostname                 # login1 / login2 / submit.int.janelia.org -> submit host: context 2 only, never compute here
echo "${LSB_JOBID:-}"    # non-empty -> you are inside an LSF job on a compute node: context 3
```

The deciding question is **whose code runs on the cluster, and is anyone watching**. A script the user has reviewed, submitted by you from a chat, is context 2. Anything where the model's own output is executed on the cluster without per-step review is context 3, and the HPC policy (added 2026-08-28) requires that it be sandboxed: file, device and network access restricted to the minimum the task needs, never the user's full permissions. Common cases:

- User runs Claude Code on the Mac and asks you to launch a sweep: **context 2**.
- User launches `claude` inside `bsub -Is /bin/bash` on a compute node: **context 3** (sandbox repo's "Mode C"; a human is present, so the sandbox is defense in depth, but still use it).
- A `bsub ... claude -p "..."` job that runs unattended: **context 3** ("Mode A"). A loop of such calls in one job: "Mode B".
- A context-2 worker that merely *imports* a library is fine. A context-2 worker that `exec`s code the model wrote during the session, unreviewed, is context 3.

## Output layout

```text
<scripts_dir>/                       # on shared storage (/groups/<lab>/...) if the cluster must see it
+-- <script_name>.py                 # analysis script (argparse-based, script-opinions)
+-- runs/
    +-- run_<script_name>.sh         # worker (one file, all contexts)
    +-- submit_<script_name>.py      # Python submitter (stdlib-only)
    +-- logs/                        # stdout/stderr (-o / -e targets)
```

The submitter must import **only stdlib** so it runs under the base Python on the laptop *and* on the submit host.

## Worker script (`run_<name>.sh`)

One worker for all contexts. **No `#BSUB` header**: LSF only parses `#BSUB` lines when the script arrives on stdin (`bsub < script.sh`), which does not combine with positional arguments. All resource requests live in the submitter's `bsub` argv instead, so a wrong resource request is a one-line diff in review.

```bash
#!/bin/bash
# $1 is ALWAYS the absolute path to the analysis script.
# $4 is ALWAYS --dir_save (the run directory; see script-opinions).
set -u
ANALYSIS_SCRIPT=$1
RUN_NAME=$2
ARG_A=$3
DIR_SAVE=$4
[ -z "${ANALYSIS_SCRIPT:-}" ] || [ -z "${DIR_SAVE:-}" ] && { echo "Usage: $0 <script> <run_name> <arg_a> <dir_save>"; exit 1; }
[ ! -f "$ANALYSIS_SCRIPT" ] && { echo "Error: $ANALYSIS_SCRIPT not found"; exit 1; }

# One interpreter per machine, both explicit. Never `conda activate` in non-interactive bash.
case "$(uname -s)" in
  Darwin) PYTHON_EXEC="/Users/<user>/mambaforge/envs/<env>/bin/python" ;;
  *)      PYTHON_EXEC="/groups/<lab>/home/<user>/.conda/envs/<env>/bin/python" ;;
esac
[ ! -x "$PYTHON_EXEC" ] && { echo "Error: $PYTHON_EXEC not executable"; exit 1; }

# Threads = LSF slots. Unset locally, so fall back to a small fixed number.
# Every BLAS/OpenMP layer must be capped or each one spawns node-core-count threads.
NTHREADS=${LSB_DJOB_NUMPROC:-4}
export OMP_NUM_THREADS=$NTHREADS MKL_NUM_THREADS=$NTHREADS OPENBLAS_NUM_THREADS=$NTHREADS \
       NUMEXPR_NUM_THREADS=$NTHREADS TBB_NUM_THREADS=$NTHREADS OPENMP_NUM_THREADS=$NTHREADS

# Node-local scratch on the cluster (never /tmp), removed on exit.
if [ -n "${LSB_JOBID:-}" ]; then
  WORKDIR="/scratch/$USER/${LSB_JOBID}${LSB_JOBINDEX:+_$LSB_JOBINDEX}"
  mkdir -p "$WORKDIR" && trap 'rm -rf "$WORKDIR"' EXIT
  export TMPDIR="$WORKDIR"
fi

echo "Running $RUN_NAME (job=${LSB_JOBID:-local}, host=$(hostname), threads=$NTHREADS, gpus=${CUDA_VISIBLE_DEVICES:-none})"
"$PYTHON_EXEC" "$ANALYSIS_SCRIPT" \
    --arg-a "$ARG_A" \
    --dir_save "$DIR_SAVE"
exit $?
```

Never set `CUDA_VISIBLE_DEVICES` yourself; LSF sets it to the GPUs it assigned. For MATLAB, `module load matlab` instead of a hardcoded `/misc/local/...` path.

## Submitter (`submit_<name>.py`)

Backend and resources are module-level constants. The two dispatch helpers are the *only* places backend-specific argv lives, so the main loop reads backend-agnostic.

```python
#!/usr/bin/env python3
import argparse, shlex, subprocess
from pathlib import Path
from typing import Callable, TypedDict

BACKEND: str = "lsf"            # "local" (context 1) or "lsf" (context 2)
SSH_HOST: str | None = None     # e.g. "login1.int.janelia.org" when submitting from the laptop.
                                # Then every path below must be a CLUSTER path (/groups, /nrs),
                                # and <scripts_dir> must live on shared storage.

class LsfSpec(TypedDict):
    queue: str        # "short" (<=1h), "local" (CPU batch), "gpu_l4", "gpu_a100", ... see references.md
    slots: int        # 1 slot = 1 core + 15 GB (CPU/L4/RTX queues) or 40 GB (A100/H100/H200 queues)
    walltime: str     # "H:MM"; GPU queues default to 2 h, CPU to the queue max, so ALWAYS set it
    gpu: str | None   # None, or e.g. "num=1"; requires a gpu_* queue
    job_prefix: str   # stable prefix so `bkill -J "<prefix>*"` cancels the sweep

LSF: LsfSpec = {"queue": "local", "slots": 4, "walltime": "4:00", "gpu": None, "job_prefix": "<name>"}

class DatasetSpec(TypedDict):
    name: str
    arg_a: str

DATASETS: list[DatasetSpec] = [
    {"name": "run_001", "arg_a": "/groups/<lab>/project/input_a"},
]
OUTPUT_ROOT = "/groups/<lab>/project/output"

def dispatch_local(worker: Path, worker_args: list[str], _name: str, _logs: Path) -> list[str]:
    return ["bash", str(worker), *worker_args]

def dispatch_lsf(worker: Path, worker_args: list[str], name: str, logs: Path) -> list[str]:
    cmd = ["bsub", "-J", f"{LSF['job_prefix']}_{name}", "-n", str(LSF["slots"]), "-W", LSF["walltime"],
           "-q", LSF["queue"], "-o", f"{logs}/{name}_%J.out", "-e", f"{logs}/{name}_%J.err"]
    if LSF["gpu"]:
        cmd += ["-gpu", LSF["gpu"]]
    cmd += ["bash", str(worker), *worker_args]
    return ["ssh", SSH_HOST, shlex.join(cmd)] if SSH_HOST else cmd

DISPATCH: dict[str, Callable[[Path, list[str], str, Path], list[str]]] = {
    "local": dispatch_local,
    "lsf": dispatch_lsf,
}

def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--dry-run", action="store_true")
    args = ap.parse_args()

    here = Path(__file__).resolve().parent
    worker = here / "run_<name>.sh"
    analysis = (here / "../<script_name>.py").resolve()
    logs = here / "logs"
    assert worker.exists() and analysis.exists()
    logs.mkdir(exist_ok=True)
    Path(OUTPUT_ROOT).mkdir(parents=True, exist_ok=True)

    build_cmd = DISPATCH[BACKEND]
    for i, d in enumerate(DATASETS):
        dir_save = str(Path(OUTPUT_ROOT) / d["name"])
        worker_args = [str(analysis), d["name"], d["arg_a"], dir_save]
        cmd = build_cmd(worker, worker_args, d["name"], logs)
        print(f"[{i+1}/{len(DATASETS)}] {' '.join(cmd)}")
        if not args.dry_run:
            out = subprocess.run(cmd, check=True, capture_output=True, text=True, stdin=subprocess.DEVNULL)
            print(out.stdout.strip())     # "Job <123> is submitted to queue <local>." -- keep the IDs

if __name__ == "__main__":
    main()
```

**Short throwaway sweeps:** skip the `TypedDict`; hardcode config at the top and nest loops over the sweep axes. Worker stays unchanged.

**Array jobs for big sweeps (20+ datasets, hard cap 15,000 jobs at once):** the submitter writes a lookup table (one row per dataset) and submits one `bsub -J "<prefix>[1-N]%15"`; the worker reads `$LSB_JOBINDEX` (1-based) to pick its row; logs use `-o logs/<prefix>_%J_%I.out`. One `bjobs <JOBID>` then reports every element. Template in `references.md`.

## Rules (all contexts)

- **Absolute analysis-script path as `$1`.** The submitter computes it; the worker never resolves paths.
- **Explicit `PYTHON_EXEC`** per machine, no `conda activate`.
- **All worker args required**, no defaults, no optionals, no env vars between submitter and worker.
- **Propagate exit codes** (`exit $?`).
- **`stdin=subprocess.DEVNULL` + `capture_output=True` + `check=True`** on every `subprocess.run`; no `try/except` that hides a `CalledProcessError`.
- **Always support `--dry-run`** and run it first.
- **Secrets never on the `bsub` command line or in `-J` names.** `bjobs -l` / `bjobs -u all` show both to every user. Read keys from a `chmod 600` file inside the job.

## Context 1: laptop

*`BACKEND = "local"`.* Sequential by default; for fan-out wrap the loop in `concurrent.futures.ProcessPoolExecutor`. `check=True` stops at the first failure, which is what you want while smoke-testing. Cancel with `pkill -f run_<name>.sh`. Keep the `NTHREADS` fallback modest; the Mac is shared with the user's desktop. No sandbox: the user owns the machine and the session.

## Context 2: dispatching to LSF

*`BACKEND = "lsf"`.* You are on the laptop (`SSH_HOST` set) or on a login/submit host (`SSH_HOST = None`).

- **Only submit from login/submit hosts.** They are 1 core / 16 GB shared by everyone: `bsub`, `bjobs`, light orchestration scripts, nothing else. No compiles, no `pip install`, no `conda create`, no large `rsync`; those are jobs (`bsub -n 4 -W 1:00 -Is /bin/bash` for an interactive one).
- **Paths.** Cluster jobs see `/groups/<lab>/...` (PRFS, backed up), `/nrs/<lab>/...` (not backed up), `/scratch/$USER` (node-local). The laptop sees the same shares over SMB (`smb://prfs.hhmi.org/<lab>/...`), so stage data by copying to the share from the Mac, not by `scp` through a login node. Scripts and `runs/` must be on `/groups` so compute nodes can read them.
- **Size the job** with the slot model: slots = max(cores needed, ceil(RAM_GB / RAM_per_slot)). Jobs over their memory are killed with no recovery. Always pass `-W`; GPU queues default to 2 h.
- **GPU jobs** need an explicit `-q gpu_*` plus `-gpu "num=N"`. Start with `gpu_l4`; match slots to the queue's slots-per-GPU ratio (8 on L4/RTX, 12 on A100/H100/H200) and do not over-request at scale. Check whether bf16 is enough before asking for a bigger GPU.
- **Monitoring is rate-limited.** Never poll faster than once per minute, never `watch -n 1 bjobs`, never SSH to compute nodes to run `nvidia-smi`. Two-phase backoff: pending -> check at 1, 2, 4, 8 min; running -> wait ~half the walltime, then halve the remaining time. One `bjobs ID1 ID2 ...` call for all tracked jobs. Use RTM (<https://lsf-rtm.int.janelia.org>) and `bpeek <jobid>` for details; after a test run, read RTM's max-memory and efficiency to right-size the next submission.
- **Cancel**: `bkill -J "<prefix>*"` for the sweep, `bkill 0` for everything you own. Kill idle interactive jobs; billing runs from start to end, failed jobs included.
- **No sandbox** because the job executes a fixed, reviewed worker + analysis script under the user's identity, exactly as if the user had typed the `bsub` line. If the worker would run code generated during this session without review, you have crossed into context 3.

```python
import subprocess, time
def wait_for_jobs(job_ids: list[str], walltime_min: int, ssh_host: str | None = None) -> None:
    """Two-phase backoff per the HPC hints. One bjobs call per check, never under 60 s."""
    def states() -> dict[str, str]:
        cmd = ["bjobs", "-noheader", "-o", "jobid stat", *job_ids]
        if ssh_host:
            cmd = ["ssh", ssh_host, " ".join(cmd)]
        out = subprocess.run(cmd, check=True, capture_output=True, text=True, stdin=subprocess.DEVNULL).stdout
        return dict(line.split()[:2] for line in out.splitlines() if line.strip())
    delay = 60
    while True:
        time.sleep(delay)
        st = states()
        if all(s in ("DONE", "EXIT") for s in st.values()):
            return
        if any(s == "RUN" for s in st.values()):
            delay = max(60, int(walltime_min * 60 / 2)); walltime_min /= 2
        else:
            delay = min(delay * 2, 480)
```

## Context 3: you (an agent) are running on the cluster

The unit of work is no longer "a reviewed script" but "an agent loop", so the HPC policy applies: every agent process and every piece of agent-generated code executes inside a sandbox with default-deny filesystem and network. Use the HPC team's wrappers from <https://github.com/JaneliaScientificComputingSystems/agentic-sandbox>; do not improvise with raw `bwrap`, and never use Apptainer or Docker as the sandbox (Apptainer is too permissive, Docker does not exist on the cluster). If neither wrapper fits, ask before improvising.

**One-time setup** (done as the user, outside any sandbox):

```bash
git clone https://github.com/JaneliaScientificComputingSystems/agentic-sandbox.git /groups/<lab>/home/<user>/agentic-sandbox
claude auth login                       # once; the sandbox reuses ~/.claude via --claude
# podman (GPU) only: ask HPC for a /etc/subuid + /etc/subgid range, then point podman storage at scratch:
mkdir -p ~/.config/containers && printf '[storage]\ndriver = "overlay"\nrunroot = "/scratch/$USER/podman-run"\ngraphroot = "/scratch/$USER/podman-storage"\n[storage.options]\nmount_program = "/usr/bin/fuse-overlayfs"\n' > ~/.config/containers/storage.conf
```

**Wrapper flags** (identical for both): `--ro PATH` / `--rw PATH` (repeatable), `--scratch` (= `--rw /scratch/$USER`), `--allow HOST` (repeatable; omit for no network), `--claude` (binds `~/.claude` + the CLI), `--opencode`, and for `podman-run.sh` only `--gpu`, `--image`, `--keep-id`. Always mounted: the toolchain read-only and **`$PWD` read-write**, so `cd` into the project directory first, not into the sandbox repo. `$HOME` is never mounted unless you ask; never ask for all of it, and with `podman-run.sh` never bind anything that contains `$HOME` (it cannot mask `.ssh` / `.aws` the way bwrap does).

**Mode A, unattended one-shot (CPU, bwrap)**, the policy's primary target:

```bash
SBX=/groups/<lab>/home/<user>/agentic-sandbox/scripts
bsub -n 4 -W 4:00 -q local -J agent_<task> -o runs/logs/agent_%J.out -e runs/logs/agent_%J.err \
  "cd /groups/<lab>/project && $SBX/sandbox-run.sh --scratch --claude \
     --ro /groups/<lab>/project/data --rw /groups/<lab>/project/output \
     --allow api.anthropic.com --allow claude.ai --allow platform.claude.com -- \
     claude -p 'Analyse data/ and write results to output/. Do not touch anything else.' --model sonnet"
```

**Mode A with GPU (podman)**: same shape, swap the wrapper and request a GPU queue.

```bash
bsub -n 8 -gpu "num=1" -q gpu_l4 -W 2:00 -J agent_<task> -o runs/logs/agent_%J.out -e runs/logs/agent_%J.err \
  "cd /groups/<lab>/project && $SBX/podman-run.sh --gpu --scratch --claude \
     --rw /groups/<lab>/project/output \
     --allow api.anthropic.com --allow claude.ai --allow platform.claude.com -- \
     claude -p '...' --model sonnet"
```

**Mode B, loop in one sandbox**: wrap `/bin/bash -c 'for x in ...; do claude -p "..."; done'` as the command. For stronger isolation submit one Mode A job per item (array job).

**Mode C, interactive**: `bsub -n 4 -W 4:00 -Is /bin/bash`, `cd` to the project, then `$SBX/sandbox-run.sh --scratch --claude --allow ... -- claude`. The REPL behaves normally; context persists; auto-update fails harmlessly because `downloads.claude.ai` is not allowed.

**Running agent-generated analysis code**: the same wrappers, with no `--claude` and no `--allow` at all when the code needs no model and no network: `sandbox-run.sh --scratch --ro <data> --rw <output> -- python3 generated_script.py`. This is the minimal sandbox and should be the default for executing anything the model wrote.

Rules specific to context 3:

- **LSF limits still apply inside the sandbox** (cgroups are untouched) and `bkill` kills the whole tree. Request slots, walltime and GPU exactly as in context 2; the sandbox adds nothing to the resource request.
- **Assume `bsub` does not work from inside the sandbox** (the network namespace is unshared; not verified). Submitting further jobs is a context-2 action done by the outer, unsandboxed layer. A sandboxed agent writes a plan or a dataset list; the outer submitter dispatches it.
- **`--allow` is exact-host, default-deny.** A hang with "Request timed out... Retrying" means a missing host, not a bug. Proxy log: `<socket>.log` in `/tmp` of the job.
- **API keys**: export `ANTHROPIC_API_KEY` in the environment for bwrap, or read it from a `chmod 600` file under `/scratch/$USER` for podman (no env passthrough). Never on the `bsub` line.
- **One opencode sandbox per account at a time** (sqlite state on NFS). Claude Code has no such limit.
- **Kimi K3 via LiteLLM** (`--opencode --allow litellm.int.janelia.org`, `--model litellm/kimi-k3`) sits behind a guardrail; a `GuardrailRaisedException` is a refusal, not an outage.
- **Check `cat /proc/sys/user/max_user_namespaces` > 0** on any queue you have not used before; bwrap needs unprivileged user namespaces.

## Pre-submit checklist

**All contexts**
- Analysis script conforms to `script-opinions`; `BACKEND` and `SSH_HOST` set for the intended context; `runs/logs/` exists; `--dry-run` output reviewed.
- Threading env vars set in the worker; no `CUDA_VISIBLE_DEVICES`, no `/tmp`, no secrets in argv.

**Context 2 and 3**
- Queue + slots + `-W` + `-gpu` are valid together and sized from the slot model (`references.md`).
- `-o`/`-e` point into `runs/logs/` with `%J` (and `%I` for arrays); no per-job e-mail.
- Scripts and data on `/groups` or `/nrs`, not on the laptop disk; scratch cleaned on exit.
- Monitoring plan uses the backoff schedule, not a tight loop.

**Context 3 only**
- Every agent process and every agent-written program is wrapped by `sandbox-run.sh` or `podman-run.sh`.
- Binds are the minimum: specific `--ro` data, specific `--rw` output, `--scratch`; never `$HOME`.
- `--allow` lists only the model endpoint(s) the harness needs; none for pure compute.
- `claude auth login` done beforehand (or key file in place); podman storage config and subuid range in place for GPU runs.

## References

- `references.md`: Janelia LSF cheat sheet (queues, slot model, GPU table, bsub templates, env vars, storage tiers, polling schedule, Slurm-to-LSF translation, sandbox command reference). Consult before filling in `LSF` or any `bsub` line.
- Live cluster docs: <https://hpc.int.janelia.org/docs/ai-agent-hints> (authoritative; internal network / VPN).
- Sandbox wrappers and their tested behaviour: <https://github.com/JaneliaScientificComputingSystems/agentic-sandbox> (`README.md` short form, `ADMIN-NOTES.md` evidence).
