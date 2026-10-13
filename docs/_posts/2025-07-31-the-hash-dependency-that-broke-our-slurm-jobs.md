---
layout: post
title: "The #$ that silently broke our SLURM job dependencies"
date: 2025-07-31 16:25:13 +0000
categories: python hpc slurm debugging
---

This is a short post about a single character. A migration from SGE to SLURM left one line of job-script
generation using SGE's comment syntax, and because SLURM treats that syntax as an ordinary comment, our job
dependencies silently stopped working. Nothing errored. Jobs just ran before the things they depended on.

### Background

INSaFLU runs its heavy analysis as batch jobs. A `ProcessSched` class builds a bash script for each job,
submits it, and — when a job must wait for others — writes a dependency line into the script that tells the
scheduler not to start until the prerequisites have finished. The class still carries its SGE name
(`process_SGE.py`) long after the migration to SLURM.

### The generated script

Here is the script builder. Most lines are correct `#SBATCH` directives:

```python
def set_script_run_slurm(self, out_dir, queue_name, vect_cmd, job_name,
                         b_remove_out_dir=False, job_name_wait=[],
                         alternative_temp_dir=None, cpus=1, memory="4G"):
    file_name_out = os.path.join(out_dir, ProcessSched.FILE_NAME_SCRIPT_SLURM)

    with open(file_name_out, "w") as handleSLURM:
        handleSLURM.write("#!/bin/bash\n")
        handleSLURM.write("#SBATCH --export=ALL\n")
        handleSLURM.write("#SBATCH --cpus-per-task={}\n".format(cpus))
        handleSLURM.write("#SBATCH --mem={}\n".format(memory))

        if isinstance(job_name_wait, str):
            job_name_wait = [job_name_wait]
        if len(job_name_wait) > 0:
            job_wait_ids = self.jobwait_ids(job_name_wait)
            if len(job_wait_ids) > 0:
                handleSLURM.write(
                    "#$ --dependency=afterok:{}\n".format(",".join(job_wait_ids))
                )

        handleSLURM.write("#SBATCH -J {}\n".format(job_name))
        handleSLURM.write("#SBATCH --ntasks=1\n")
        handleSLURM.write("#SBATCH --output={}/slurm_%j.out\n".format(out_dir))
        handleSLURM.write("#SBATCH --begin=now\n")
        handleSLURM.write("\n")
        for cline in vect_cmd:
            handleSLURM.write("\n" + cline)

    return file_name_out
```

Look at the dependency line: `#$ --dependency=afterok:...`. In SGE, `#$` is how you pass options to
`qsub`. In SLURM, every directive must start with `#SBATCH`. `#$` is not a directive — it is just a
comment. SLURM parses the script, sees a comment where a dependency would have been, and happily schedules
the job immediately.

The correct line is:

```python
handleSLURM.write(
    "#SBATCH --dependency=afterok:{}\n".format(",".join(job_wait_ids))
)
```

### Why this was so hard to notice

A broken dependency does not crash anything. The dependent job starts, finds its inputs missing or partial,
and either fails with a confusing error or — worse — succeeds on stale data. And because the rest of the
script is correct `#SBATCH`, the generated job file *looks* right at a glance. The tell is easy to miss
precisely because it blends into the ocean of commented-out SGE lines that surround it:

```python
# handleSLURM.write("#$ -S /bin/bash\n")       # SGE shell selection
# handleSLURM.write("#$ -j y\n")               # SGE stderr merge
```

Those commented lines are harmless leftovers. The live `#$ --dependency` line is not commented *by the
author* — it is commented *by SLURM*, silently, at parse time.

### The other migration smells

Once I found one, I went looking for the rest. The same file still parses SGE-format status output. The
wait-for-quorum function reads `squeue` but skips a header containing `job-ID`:

```python
if line.find("job-ID") != -1 or len(line) < 3 or line.find("---") == 0:
    continue
```

`squeue`'s header is `JOBID PARTITION NAME USER ST ...`, not `job-ID`, so that guard never fires. And
`_get_prefix_in_wait_queue` still looks for SGE `qstat` markers that SLURM never emits:

```python
if line.find("=========") == 0:
    job_candicate = None
elif line.find("job_number:") == 0:
    job_candicate = line.split(":")[1].strip()
elif line.find("scheduling info:") == 0 and job_candicate is not None:
    vect_job_ids.append(job_candicate)
```

This function can never return a result against SLURM.

There is also a subtler design issue with the dependency mechanism itself: `jobwait_ids` resolves
prerequisite *job names to ids* by querying `squeue` at the moment the script is written. If a predecessor
has not yet appeared in the queue, its id is not found and it silently drops out of the dependency list.
Resolving dependencies by name against a live queue is racy; SLURM's native job-id chaining would be
robust.

### What I took away

- When migrating schedulers, grep for the old comment marker. `#$` and `#SBATCH` look like brothers; only
  one of them does anything.
- A dependency that fails open is more dangerous than one that fails closed: without it, jobs run early
  and corrupt downstream results rather than refusing to start.
- Leftover naming (`process_SGE.py`, `#$`-style commented options, stale header checks) is a map of where
  the migration is incomplete. When you rename a system, the old name keeps pointing at the parts you have
  not finished yet.
