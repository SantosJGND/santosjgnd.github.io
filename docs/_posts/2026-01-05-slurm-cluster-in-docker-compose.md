---
layout: post
title: "A real SLURM cluster in Docker Compose, and the bugs it hides"
date: 2026-01-05 17:32:33 +0000
categories: docker slurm devops
---

For development, INSaFLU runs a real SLURM scheduler, not a fake queue. I wanted that because the job logic is only
useful if it runs against the same scheduler we use on the cluster. The database, the controller, two compute nodes and
the web app are separate containers, started by one Compose file. This post describes how it fits together, how a job
gets from the web app to a compute node, and the problems I found while reading the configuration.

One caveat: this describes the `ubuntu-slurm` branch at commit `3bfd5eb0`. Some fixes exist on another branch and are not
in this one, and I say where that matters.

### The containers

There are five services that make up the cluster. MariaDB holds SLURM's accounting data. `slurmdbd` is the accounting
daemon that talks to it. `slurmctld` is the controller, and `c1` and `c2` are compute nodes. The web app is the fifth
container, and it is also where jobs are submitted from. Each container finds the others by its service name on a private
network, and the compute nodes also join the web app's network, because they mount the same data volumes.

Startup order is enforced with health checks rather than sleeps. The accounting daemon waits for the database to be
healthy, and the controller waits for the accounting daemon. Each entrypoint also waits for the upstream port to open
before it starts its own daemon, since "container started" does not mean "service listening".

### The cluster configuration

The partition and node sizes are in `slurm.conf`:

```conf
SlurmctldHost=slurmctld
AuthType=auth/munge
AccountingStorageType=accounting_storage/slurmdbd
NodeName=c[1-2] CPUs=8 RealMemory=8192 State=UNKNOWN
PartitionName=normal Default=yes Nodes=c[1-2] Priority=50 Shared=NO MaxTime=5-00:00:00 State=UP
```

That is two nodes with eight CPUs and 8 GB each, in one default partition. Authentication uses munge, and accounting
goes through `slurmdbd` into MariaDB. SLURM itself is compiled from source at a pinned tag, in a two-stage Dockerfile. The
first stage builds it, and the result becomes a base image. The compute node and web app images are built on top of that
base, so the client and server programs come from the same build.

### How a job gets to a node

The web app submits work with `sbatch`, from a directory on a volume that the controller and both compute nodes also
mount. The job scripts carry their options as `#SBATCH` lines, like this startup job:

```bash
#SBATCH --export=ALL
#SBATCH --job-name=update_files
#SBATCH --ntasks=1
#SBATCH --output=/data/tmp/%x_%j.out
```

Status comes from `sinfo`, `squeue` and `scontrol`, which the cluster status script calls from inside the web container.
The compute nodes read conda environments from a shared software volume, so they do not install anything themselves.

### Users and permissions

Shared volumes only work if the same user IDs exist in every container. The Dockerfile pins them: `slurm` is UID 990,
`munge` is UID 102, and the application user is UID 1000, all in the `slurm` group. Each entrypoint then changes ownership
of the shared directories. I think that is the part most likely to break, because nothing checks it. If someone changes a
base image and the IDs move, the failure shows up as permission errors far away from the cause.

### What I found

The three services each have their own copy of the SLURM configuration, and the copies have already drifted. The compute
node's `cgroup.conf` adds a plugin the others do not have. Keeping three copies in sync by hand is how that happens.

The compute node entrypoint has a stray quote after its first `exec`. On the fix branch that is listed as the reason the
nodes never started, which is a good example of how much a single character can matter in a shell script.

The nodes and the web app mount the controller's munge socket instead of running their own daemon. That means restarting
the controller can break authentication for the whole cluster. The fix branch splits this up, and I agree with the change.

There are two secrets in the repository. The munge key is committed in each component, and the accounting password is the
word `password` in the daemon configuration. That is fine for a local development cluster, and it should not be used
anywhere else. It is still worth saying out loud, because it is easy to copy a dev setup into something real.

Some scripts are stale. The stop script names a fixed list of containers and misses three of the newer ones. The start script
prints instructions for scaling scripts that were deleted. And the app still uses SGE names for its settings, even though the
scheduler is SLURM. That is the same migration story as [the hash-dependency post]({% post_url 2025-07-31-the-hash-dependency-that-broke-our-slurm-jobs %}).

### What I would change

Keep one SLURM configuration directory and mount it everywhere. Give each container its own munge daemon, or document the shared
socket as a hard dependency on the controller. Move the accounting password into an environment secret. Remove the stale scripts
and the SGE names once nothing reads them. Compose is a good place to test scheduler behaviour, because it is cheap to tear down
and rebuild. The thing to fix first is the drift between the three copies of the configuration.
