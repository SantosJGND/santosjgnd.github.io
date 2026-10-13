---
layout: post
title: "A real SLURM cluster in Docker Compose, and the bugs it hides"
date: 2026-01-05 17:32:33 +0000
categories: docker slurm devops
---

INSaFLU's development environment runs a real SLURM scheduler, not a fake queue. The database, controller,
compute nodes, and web app are separate containers started by one Compose file. This post describes the
layout, how a job gets from the web app to a compute node, and the problems I found while reading it.

One caveat on scope: this describes the `ubuntu-slurm` branch at commit `3bfd5eb0`. Some fixes exist on another
branch and are not in this one. I note where that matters.

### The services

| Service | Role | Hostname |
|---|---|---|
| `mysql` | MariaDB 12.3.2, SLURM accounting database | `mysql` |
| `slurmdbd` | SLURM accounting daemon | `slurmdbd` |
| `slurmctld` | SLURM controller | `slurmctld` |
| `c1`, `c2` | Compute nodes (`slurmd`) | `c1`, `c2` |
| `insaflu-ubuntu` | Django web app, and the job submitter | `insaflu` |

Each container finds the others by its service name on a private bridge network. The compute nodes also join
the web app's network, because they mount the same data volumes.

### Dependencies

Startup order is enforced with health checks, not sleeps:

```yaml
slurmdbd:
  depends_on:
    mysql:
      condition: service_healthy
slurmctld:
  depends_on:
    slurmdbd:
      condition: service_healthy
```

The chain is `mysql`, then `slurmdbd`, then `slurmctld`, then `c1` and `c2`, then the web app. Each entrypoint
also waits on the upstream TCP port before starting its daemon, which covers the gap between "container
started" and "service listening".

### The cluster config

The partition and node sizes are in `slurm.conf`:

```conf
SlurmctldHost=slurmctld
AuthType=auth/munge
AccountingStorageType=accounting_storage/slurmdbd
NodeName=c[1-2] CPUs=8 RealMemory=8192 State=UNKNOWN
PartitionName=normal Default=yes Nodes=c[1-2] Priority=50 Shared=NO MaxTime=5-00:00:00 State=UP
```

Two 8-CPU, 8 GB nodes in one default partition. Authentication is munge, and accounting goes through
`slurmdbd` into MariaDB.

The build compiles SLURM from source at a pinned tag (`SLURM_TAG=slurm-25.05`). The two-stage Dockerfile
produces a `slurm-runtime` image, and the node and web images are built on top of it, so the client and server
binaries come from one build.

### How a job runs

The web app submits work with `sbatch` from a shared `/data` volume. Status is read with `sinfo`, `squeue`,
and `scontrol`. A typical startup job looks like this:

```bash
#SBATCH --export=ALL
#SBATCH --job-name=update_files
#SBATCH --ntasks=1
#SBATCH --output=/data/tmp/%x_%j.out
```

Jobs live on the `slurm_jobdir` volume, which the controller and both nodes mount. The compute nodes read the
conda environments from a shared `/software` volume, so they do not install software themselves.

### UIDs and permissions

Shared volumes only work if the same user IDs exist in every container. The Dockerfile pins them: `slurm` is
UID 990, `munge` is UID 102, and the app user `flu_user` is UID 1000, all in the `slurm` group. Each entrypoint
then `chown`s its shared directories. That is the part most likely to break when someone changes a base image.

### Problems I found

1. **Three copies of the SLURM config.** The web app, the controller, and the nodes each have their own
   `configs/slurm` directory, and they already differ. The node copy of `cgroup.conf` adds a plugin the other
   copies do not have.
2. **A syntax error stopped the compute nodes.** The node entrypoint has a stray quote after its first `exec`.
   On the fix branch, that is listed as the reason `c1` and `c2` never started.
3. **Shared munge socket.** The nodes and the web app mount the controller's `/run/munge` and do not run their
   own `munged`. The fix branch notes that restarting `slurmctld` broke authentication for the whole cluster.
4. **Secrets in git.** The munge key is committed (128 bytes in each component), and the SLURM accounting
   password is `password` in `slurmdbd.conf`. Both are fine for a local dev cluster, and neither should be used
   anywhere else.
5. **Stale scripts.** `stop.sh` names a fixed list of containers and misses `officer`, `mlflow`, and `ml_app`.
   `up.sh` prints instructions for scaling scripts that no longer exist.
6. **Naming carry-over.** The app still uses `RUN_SGE` and `SGE_ROOT`, while the scheduler is SLURM. The
   migration from SGE is the same story as [the hash-dependency post]({% post_url 2025-07-31-the-hash-dependency-that-broke-our-slurm-jobs %}).

### What I would change

Keep one SLURM config directory, mounted everywhere. Run one `munged` per container, or document the shared
socket as a hard dependency on the controller. Move the accounting password into an environment secret. Remove
the stale scripts and the SGE names once nothing reads them.

Compose is a good place to test scheduler behaviour, since it is cheap to tear down and rebuild. The cost is
that the config drifts silently when three copies exist, and that drift is the main thing to fix here.
