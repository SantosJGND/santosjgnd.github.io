---
layout: post
title: "One wrapper for every bioinformatics tool, and what it does not check"
date: 2022-08-16 08:42:47 +0100
categories: python subprocess bioinformatics
---

INSaFLU calls dozens of command-line tools: aligners, classifiers, assemblers, format converters. The obvious way to
run them from Python is `subprocess` everywhere, and that is roughly what we started with. Instead, the project has
one class, `RunCMD`, that every pipeline step uses. I want to explain what it does well, and also the places where
it quietly does less than it appears to.

### Setting up a runner

Each `RunCMD` knows the directory its tools live in, and it has its own log file, named after the prefix and the
task:

```python
class RunCMD:
    def __init__(self, bin, logdir: str = "", prefix: str = "run", task: str = "NONE"):
        if bin:
            if bin[-1] != "/":
                bin += "/"
        self.bin = bin
        self.logger = logging.getLogger(f"{prefix}_{task}")
        ...
        self.logger.addHandler(
            logging.FileHandler(os.path.join(logdir, f"{prefix}_{task}.log"))
        )
```

A call site looks like this, and the classification step alone calls `run` around forty times:

```python
self.cmd = RunCMD(bin, logdir=log_dir, prefix=prefix, task="classification")
self.cmd.run(cmd)
```

Having one object per step means every tool's output lands in a predictable place, which is most of the value when
something goes wrong at three in the morning.

### Running a command

Under the hood, `run` joins the command with the binary directory and calls `Popen` with `shell=True`, then reads
stdout and stderr. Both are written to the log. The shell is convenient for pipes and redirects, but it means file
names with spaces need care, and we have had to fix quoting more than once.

### The failure check

This is where I need to be honest. The only thing that marks a run as failed is a string in stderr:

```python
def flag_error(self, subprocess_errorlog, cmd: str = ""):
    if subprocess_errorlog:
        subprocess_errorlog = subprocess_errorlog.decode("utf-8")
        if "[error]" in subprocess_errorlog.lower():
            return True
    return False
```

If a tool prints `[error]`, we raise. If a tool exits with a non-zero status and prints nothing that matches, we do
not raise. The exit code is never read. I went through the code twice to make sure I was not missing it, and it is
not there. Some tools do print `[error]` on success, which causes false failures, and others fail silently. The
wrapper cannot tell the difference, because it is not looking at the one thing that tells it.

### The script runners

There is a second group of methods that run a command as a temporary bash script, wait for a flag file, and time
out after a limit. These are worse. Their error state is a string, either the log text or the word "Timeout". When
the failure check runs on that string, it tries to decode it, which fails, and the exception is caught and the check
returns false. So a timed-out script gets logged and then forgotten. The caller carries on as if it worked.

### What I would change

Read the exit code in the single-command path and raise on anything non-zero, in addition to checking stderr. Make
the script runners raise on timeout and on failure instead of logging. Pass the command as a list with `shell=False`
where possible, which avoids most of the quoting problems. Keep the per-task log files, because they are the first
thing I open when a run fails.

The same problem shows up in the SLURM post, [the hash-dependency one]({% post_url 2025-07-31-the-hash-dependency-that-broke-our-slurm-jobs %}).
There too, the job ran, nothing reported an error, and the output was wrong. A wrapper that cannot see failure is the
same bug in a different place.
