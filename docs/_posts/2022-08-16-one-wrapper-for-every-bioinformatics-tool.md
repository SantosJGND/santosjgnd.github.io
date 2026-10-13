---
layout: post
title: "One wrapper for every bioinformatics tool, and what it does not check"
date: 2022-08-16 08:42:47 +0100
categories: python subprocess bioinformatics
---

INSaFLU calls dozens of command-line tools: aligners, classifiers, assemblers, and format converters. The
simplest way to run them is `subprocess` everywhere. Instead, the project has one class, `RunCMD`, that every
pipeline step uses. It builds the command, runs it through a shell, writes a per-task log, and decides
whether the run failed. This post explains what it does and, more importantly, what it does not check.

### Construction

```python
class RunCMD:
    def __init__(self, bin, logdir: str = "", prefix: str = "run", task: str = "NONE"):
        if bin:
            if bin[-1] != "/":
                bin += "/"
        self.bin = bin
        self.task = task
        self.prefix = prefix
        self.logger = logging.getLogger(f"{prefix}_{task}")
        self.logger.setLevel(logging.ERROR)
        for handler in self.logger.handlers[:]:
            self.logger.removeHandler(handler)
        self.logger.addHandler(
            logging.FileHandler(os.path.join(logdir, f"{prefix}_{task}.log"))
        )
```

Each `RunCMD` has a binary directory, and every command is that directory plus the tool name. Each one has its
own log file, named by prefix and task. A call site looks like this:

```python
self.cmd = RunCMD(bin, logdir=log_dir, prefix=prefix, task="classification")
self.cmd.run(cmd)
```

`classification_class.py` alone calls `run` about 40 times.

### Execution

Running a command goes through `system_deploy`:

```python
process = subprocess.Popen(cmd, shell=True, stdout=PIPE, stderr=PIPE)
out, err = process.communicate()
```

The command is a string run with `shell=True`. Stdout and stderr are captured and written to the log.

### How a failure is detected

The only failure check is a substring match on stderr:

```python
def flag_error(self, subprocess_errorlog, cmd: str = ""):
    if subprocess_errorlog:
        subprocess_errorlog = subprocess_errorlog.decode("utf-8")
        if "[error]" in subprocess_errorlog.lower():
            return True
    return False

def dispose_output_carefully(self, cmd, out, err, exec_time):
    if self.flag_error(err, cmd):
        self.logger.error(f"errror in command: {self.bin}{cmd}")
        raise Exception(err.decode("utf-8"))
    self.output_disposal(cmd, err, out, exec_time, self.bin)
```

**The exit code is never read.** `system_deploy` does not check `returncode`. A tool that exits non-zero
without printing `[error]` looks like a success. A tool that prints `[error]` on a successful run causes a
failure.

### The script runners swallow errors

`run_script` and `run_bash_script` write a temporary bash script, run it in the background, and wait for a
flag file. Their error state is a string (the log text, or `"Timeout"`). Then `flag_error` calls `.decode()`
on a string, which raises, and that exception is caught and returns `False`. So a timed-out script is logged
and never raised. A caller cannot tell that it failed.

### What I would change

1. Check `returncode` in `system_deploy`, and raise on non-zero, in addition to the `[error]` string.
2. Make the script runners raise on timeout and on non-zero exit, not log.
3. Stop building commands as one shell string. A list with `shell=False` avoids quoting bugs for file names
   with spaces.
4. Keep the log writing, since the per-task log files are useful in practice.

The problem of a pipeline silently continuing after a bad step is the same one as in
[the SLURM dependency post]({% post_url 2025-07-31-the-hash-dependency-that-broke-our-slurm-jobs %}): the
job runs, nothing errors, and the output is wrong.
